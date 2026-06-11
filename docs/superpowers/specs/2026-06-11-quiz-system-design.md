# Quiz System — Design Spec

> **Date:** 2026-06-11
> **Target codebase:** `dmoj/repo` (lcoj-site, DMOJ fork, Django 4.2)
> **Reference:** `quiz-system-methodology.md` (LQDOJ analysis), `lqdoj/` checkout (read-only reference)
> **Approach:** Clean-room rebuild. LQDOJ code is consulted as reference only; all code is written fresh and idiomatic to lcoj-site.

## 1. Goal

Replace the manual PDF quiz workflow (author PDF → send to students → grade by hand) with a website-native quiz feature: teachers build quizzes from a reusable question bank (imported via XLSX/JSON or edited in the UI), students take them online with instant auto-grading, feedback, attempt history, and a per-quiz leaderboard.

## 2. Scope decisions (settled)

| Decision | Choice |
|---|---|
| Where quizzes live | **Standalone** `/quiz/` section, like `/problem/`. No course system, no contest integration (deferrable later). |
| Question types | **Auto-graded only**: MC, MA, TF, SA. No Essay, no file upload, no manual-grading dashboard. |
| Import | **XLSX template + JSON schema**, two-phase with preview. No AI/LLM extraction. |
| Taxonomy | **Flat `QuizCategory` list + Easy/Medium/Hard level enum** on both questions and quizzes. |
| Ranking | **Per-quiz leaderboard** (best score per student, time tie-break). No profile-points integration. |
| Implementation | **Clean-room rebuild** using the methodology doc as spec. |

### Out of scope (deferred, not designed here)
Course/lesson system, contest integration (`ContestProblem.quiz` FK), Essay questions + manual grading, AI features, file uploads, `BestQuizAttempt` denormalized cache, profile-page quiz stats.

## 3. Architecture

A new self-contained Django app **`quiz/`** at the repo root of lcoj-site (sibling of `judge/`):

```
quiz/
  models.py        6 models (below)
  grading.py       pure functions, no model imports
  views/           student.py, editor.py, importer.py
  importers/       xlsx.py, json_schema.py, validator.py (shared)
  admin.py
  urls.py          mounted at /quiz/
  tests/
templates/quiz/    django_jinja templates extending base.html
resources/quiz.js  timer + autosave + sidebar map
resources/quiz.scss
```

Dependencies on `judge`: `Profile` and `Organization` FKs only. The app has its own migrations and permissions and can be removed without touching `judge`.

**New pip dependency:** `openpyxl` (XLSX parse/write). No other new dependencies; no Celery tasks.

## 4. Data models

### 4.1 QuizCategory
Flat admin-managed list (like `ProblemGroup`): `name`, `slug` (unique), `description` (optional). Used by both questions and quizzes.

### 4.2 Level (shared enum)
`TextChoices`: `easy` / `medium` / `hard`. Field on both `QuizQuestion` and `Quiz`.

### 4.3 QuizQuestion — the reusable bank unit
| Field | Type / notes |
|---|---|
| `type` | enum: `MC`, `MA`, `TF`, `SA` |
| `title` | short identifying name for the bank |
| `content` | markdown question body |
| `choices` | JSON list of choice texts (MC/MA; TF auto-renders True/False) |
| `correct_answers` | JSON — MC/TF: single index; MA: list of indices; SA: list of `{text, case_sensitive, is_regex}` |
| `explanation` | markdown, shown on result page when quiz allows |
| `category` | FK → QuizCategory (nullable) |
| `level` | Level enum |
| `shuffle_choices` | bool — randomize choice order per attempt |
| `ma_grading_strategy` | enum, MA only (see §5) |
| `is_public` | visible to all quiz editors in the bank when True |
| `authors`, `curators` | M2M → Profile |
| `created_at`, `updated_at` | timestamps |

### 4.4 Quiz — the published collection
| Field | Type / notes |
|---|---|
| `code` | unique URL slug |
| `name`, `description` | description is markdown |
| `category` | FK → QuizCategory (nullable) |
| `level` | Level enum |
| `time_limit` | minutes; null = unlimited |
| `max_attempts` | int; null = unlimited |
| `shuffle_questions` | bool |
| `show_correctness` | show green/red per question on result page |
| `show_answers` | show correct answers + explanations on result page |
| `is_public` | student visibility |
| `is_organization_private`, `organizations` | org-gating, mirrors `Problem` |
| `authors`, `curators`, `testers` | M2M → Profile, problem-style semantics |
| `created_at`, `updated_at` | timestamps |

### 4.5 QuizQuestionLink (junction)
`quiz` FK + `question` FK + `points` (float) + `order` (int). Unique on `(quiz, question)`. The same question may carry different points in different quizzes. Quiz total = Σ points.

### 4.6 QuizAttempt
| Field | Notes |
|---|---|
| `quiz`, `user` | FKs |
| `started_at` | set on Start |
| `submitted_at` | null while in progress |
| `is_submitted` | bool |
| `score` | float, final graded score |
| `question_order` | JSON list of question IDs — freezes shuffle per attempt, enables resume |
| `choice_orders` | JSON `{question_id: [choice indices]}` — freezes per-question choice shuffle |

Index: `(quiz, user, is_submitted)`, plus `(quiz, score)` for the leaderboard query.

**Lifecycle:** created on Start (not Submit) → answers autosave → Submit (or expiry) sets `is_submitted` + `submitted_at` + final `score`. At most one in-progress attempt per (quiz, user); Start resumes it if present.

**Timer enforcement is lazy server-side** (deviation from LQDOJ's Celery-beat auto-submit): deadline = `started_at + time_limit + grace` (grace 30 s for network latency). Any answer save past the deadline is rejected (HTTP 400 with a "time expired" code); any request touching an expired in-progress attempt finalizes it first using the answers already saved. Same guarantee as a scheduler, no background infrastructure.

### 4.7 QuizAnswer
`attempt` FK + `question` FK (unique together) + `answer` (JSON-encoded: index for MC/TF, list for MA, raw string for SA) + `points` (float) + `is_correct` (bool) + `saved_at`.

### 4.8 Leaderboard — a query, not a model
Best submitted attempt per user: `GROUP BY user, MAX(score)`, tie-break by smallest `submitted_at − started_at` duration, then earliest `submitted_at`. Revisit denormalization only if quiz attempt volume makes it slow.

## 5. Grading engine (`quiz/grading.py`)

Pure functions, no DB access:

```python
grade(question_type, choices, correct_answers, ma_strategy, raw_answer) -> (ratio: float, is_correct: bool)
```

`ratio` ∈ [0, 1]; answer points = `ratio × link.points`, rounded to 2 decimals. `is_correct` is True iff `ratio == 1.0`.

- **MC / TF:** selected index == correct index → 1.0, else 0.0. Empty answer → 0.0.
- **SA:** raw answer (stripped) tested against each accepted answer in order; exact compare (case-insensitive unless `case_sensitive`) or `re.fullmatch` when `is_regex`. First match → 1.0; no match → 0.0. *(Deviation from LQDOJ: no "flag for manual review" — there is no manual queue; the recovery path is regrade, §7.)* Invalid stored regexes are rejected at question-save/import time, never at grade time.
- **MA:** with `C` = correct choice set, `W` = incorrect choice set, `S` = selected set, `cs = |S∩C|`, `ws = |S∩W|`:

| Strategy | Formula (ratio) |
|---|---|
| `all_or_nothing` (default) | `1.0` if `S == C` else `0.0` |
| `partial_credit` | `max(0, cs/|C| − ws/|W|)` (penalty term 0 when `|W| = 0`) |
| `right_minus_wrong` | `max(0, (cs − ws)/|C|)` |
| `correct_only` | `cs/|C|` |

**Worked example** — choices A–E, correct = {A, C}; student selects {A, B} (`cs = 1`, `ws = 1`, `|C| = 2`, `|W| = 3`):
all_or_nothing → 0 · partial_credit → 1/2 − 1/3 ≈ 0.17 · right_minus_wrong → (1−1)/2 = 0 · correct_only → 1/2 = 0.5

**When grading runs:** on every AJAX answer save, and re-run in full on submit (submit result is authoritative).

## 6. Views & URLs

Mounted at `/quiz/`:

| URL | View |
|---|---|
| `/quiz/` | List — filter by category, level; text search; shows your best score per quiz |
| `/quiz/<code>/` | Detail — description, attempt history, attempts remaining, Start/Resume button |
| `/quiz/<code>/start` | POST — visibility + `max_attempts` check; resumes existing in-progress attempt if any |
| `/quiz/<code>/attempt/<id>/` | Take page (owner only, resume-safe) |
| `/quiz/<code>/attempt/<id>/save` | AJAX POST — upsert one answer, grade it, return `{saved, points}` (points withheld from response unless `show_correctness`) |
| `/quiz/<code>/attempt/<id>/submit` | POST — finalize + redirect to result |
| `/quiz/<code>/attempt/<id>/result` | Score + per-question feedback per `show_correctness` / `show_answers` |
| `/quiz/<code>/ranking/` | Leaderboard; viewer's row highlighted |

Editor URLs (permission-gated):

| URL | View |
|---|---|
| `/quiz/questions/` | Question bank — filter category/level/type, search |
| `/quiz/questions/new`, `/quiz/questions/<id>/edit` | Question CRUD with live preview |
| `/quiz/new`, `/quiz/<code>/edit` | Quiz CRUD — pick bank questions, drag order, per-question points |
| `/quiz/import/` | Upload XLSX/JSON → preview → confirm (§8) |
| `/quiz/export/` | Selected bank questions → XLSX in template format |
| `/quiz/<code>/attempts/` | All attempts for a quiz; per-quiz and per-question **regrade** actions |

**Take page:** all questions on one page (chosen for ≤ ~30-question quizzes) with sticky sidebar map (answered/unanswered markers, jump links) and countdown timer. Inputs autosave on change. On timer expiry the JS submits the form; the server independently enforces the deadline (§4.6).

## 7. Regrading

Editor action on a quiz (or a single question across all its quizzes): re-run §5 over every submitted attempt's stored answers, recompute attempt scores, leaderboard reflects automatically (it's a query). This is the recovery path for wrong answer keys discovered after submission. Runs synchronously in-request with a row-count guard; if a quiz ever exceeds the guard, chunked processing is the follow-up — not designed now.

## 8. Import / export

Bank-first: files import **questions** (with category/level); a checkbox optionally creates a quiz containing them in the same step.

**Pipeline:** parse (per-format) → shared validator → **preview page** (every question rendered, per-row errors with row numbers, list of categories that would be auto-created) → confirm → single atomic transaction. One invalid row blocks the whole commit; no partial imports, no silent skips.

### 8.1 XLSX template
Downloadable template with example rows; columns:

`type | title | content | choice_1 … choice_6 | correct | points | category | level | explanation | shuffle | ma_strategy`

- `correct` by type — MC: choice number (1-based); MA: comma list (`1,3`); TF: `true`/`false`; SA: pipe-separated accepted answers, with optional prefixes `re:` (regex) and `cs:` (case-sensitive), combinable as `re:cs:`.
- `category` is a slug; unknown slugs auto-create a category (listed on the preview page before commit).
- `points` used only when "also create quiz" is checked; defaults to 1.

### 8.2 JSON schema
Documented JSON array, same fields with explicit structure (per-SA-answer `{text, case_sensitive, is_regex}` objects instead of prefixes). Intended for programmatic generation — e.g. handing the schema plus an old quiz PDF to an LLM and importing the produced JSON; this replaces LQDOJ's AI import with zero API dependency. Validated with the same shared validator.

### 8.3 Export
Question-bank selection → XLSX in the §8.1 template format (lossy only where XLSX is less expressive; SA flags become prefixes). Enables round-tripping and sharing banks between teachers.

## 9. Permissions

App-level permissions (declared on quiz models):

- **`quiz.edit_all_quiz`** — staff: full access to every quiz/question/attempt, regrade anything.
- **`quiz.edit_own_quiz`** — teachers: create quizzes and questions, edit those they author/curate, use import/export.

Object-level, mirroring the problem system:

| Role | Rights |
|---|---|
| author / curator | edit quiz & its settings, view all attempts, regrade |
| tester | take the quiz while `is_public = False` |
| `is_public` | visible & takeable by logged-in students |
| `is_organization_private` + `organizations` | restrict visibility to org members |

Enforcement is queryset-level everywhere (list, detail, take, leaderboard); a quiz the user cannot see returns 404 (DMOJ convention). Question-bank visibility: authors/curators always see their own; `is_public = True` questions are visible to all editors. Anonymous users can browse public quiz lists/details but must log in to start an attempt.

## 10. Admin

Plain Django admin for all 6 models: `QuizCategory` management lives here; `QuizAdmin` gets a `QuizQuestionLink` inline; `QuizAttemptAdmin` is read-only diagnostics. The custom editor views (§6) are the daily tool; admin is the escape hatch.

## 11. Frontend

- `quiz.js` — countdown timer (green → yellow < 5 min → red < 1 min, auto-submit at zero), per-input AJAX autosave with debounce + retry-on-failure indicator, sidebar question map. Vanilla JS in the site's existing style; no new framework.
- `quiz.scss` — take page, result feedback (green/red), leaderboard table.
- Templates extend the site's `base.html` (django_jinja), reuse the site's markdown rendering for question content/explanations.

## 12. Testing strategy (TDD throughout)

| Area | Tests |
|---|---|
| `grading.py` | exhaustive matrix: every type × every MA strategy × edges (empty answer, regex SA, case sensitivity, floor-at-0, `|W| = 0` partial_credit) — pure unit tests, no DB |
| Attempt flow | start → autosave → resume → expiry (late save rejected, lazy finalize) → submit; `max_attempts`; single in-progress attempt; shuffle frozen per attempt |
| Regrade | answer-key change → scores and leaderboard recompute |
| Import | XLSX & JSON round-trip; every per-row validation error; atomicity (one bad row → zero rows committed); category auto-create |
| Access control | full matrix (anonymous / student / tester / curator / author / staff) × (public / private / org-private) × (list, detail, take, ranking) |
| Leaderboard | best-of-N attempts, duration tie-break |

## 13. Deviations from LQDOJ (deliberate)

| Area | LQDOJ | This design | Why |
|---|---|---|---|
| Timer expiry | Celery-beat auto-submit | Lazy server-side finalize | No scheduler infrastructure to deploy/debug |
| SA non-match | Flag for manual review | Incorrect (0) | No manual-grading queue in scope; regrade is the recovery path |
| Take page | Question-by-question navigator | All on one page + sidebar map | Simpler; fits ≤ ~30-question quizzes |
| Taxonomy | None on bank | Category + level on questions and quizzes | Core user requirement |
| Import | AI extraction from PDF | XLSX template + JSON schema | No LLM API dependency; JSON schema still enables LLM-assisted conversion manually |
| Leaderboard | None (course-grade only) | Per-quiz ranking query | Core user requirement |
| App location | Inside `judge` app | Separate `quiz` app | Isolation; removable without touching `judge` |
