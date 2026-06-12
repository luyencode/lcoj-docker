# Quiz Explanations — Design Spec

> **Date:** 2026-06-13
> **Extends:** `docs/superpowers/specs/2026-06-11-quiz-system-design.md`
> **Scope:** Per-question and per-choice explanations; result-page feedback configuration.

## 1. Goal

Allow quiz authors to attach explanatory text at two levels — one for the overall question and one per choice — and give each quiz a configurable feedback policy that controls how much students see on the result page.

## 2. Data model changes

### 2.1 QuizQuestion.choices — merge with choice explanations

Change `choices` from `list[str]` to `list[{text: str, explanation: str}]`.
Drop the `choice_explanations` field entirely.

```python
# Before
choices            = ["Paris", "London", "Berlin"]
choice_explanations = ["Capital of France.", "", ""]

# After
choices = [
    {"text": "Paris",  "explanation": "Capital of France."},
    {"text": "London", "explanation": ""},
    {"text": "Berlin", "explanation": ""},
]
```

**Migration:** zip `choices` (strings) with `choice_explanations` (strings) into the merged format. Where `choice_explanations` is shorter than `choices`, pad with `""`. Where it is longer (shouldn't happen but handle defensively), truncate to `len(choices)`.

`QuizQuestion.explanation` (question-level markdown textarea) is unchanged.

`grading.py` uses only `len(choices)` for MA strategy calculations and never reads choice text. No logic changes needed, but note the `choices` parameter type changes from `list[str]` to `list[dict]` — callers must pass the new format. `len([{"text":..., "explanation":...}])` behaves identically to `len(["..."])` so all calculations remain correct.

### 2.2 Quiz.result_feedback — replace two booleans with one enum

Remove `show_correctness` and `show_answers`. Add:

```python
class ResultFeedback(models.TextChoices):
    SCORE_ONLY  = 'score_only',  _('Score only')
    CORRECTNESS = 'correctness', _('Show correctness (no answer key)')
    FULL        = 'full',        _('Show correct answers and explanations')

result_feedback = models.CharField(
    max_length=11,
    choices=ResultFeedback.choices,
    default=ResultFeedback.FULL,
    verbose_name=_('result feedback'),
    help_text=_(
        'Controls what students see on the result page after submitting. '
        '"Score only" — total score, no per-question detail. '
        '"Show correctness" — green/red per question, answer key hidden. '
        '"Show correct answers and explanations" — full feedback.'
    ),
)
```

**Migration from old booleans:**

| `show_correctness` | `show_answers` | `result_feedback` |
|---|---|---|
| False | False | `score_only` |
| True | False | `correctness` |
| True | True | `full` |
| False | True | `full` |

**Default:** `full` (same effective default as before — both booleans defaulted to True).

Editors (`quiz.is_editable_by(user)`) always receive full feedback regardless of `result_feedback`.

**Note:** `result_feedback` takes effect immediately for all attempts, including already-submitted ones. If a teacher changes this field after students have submitted, their result pages update at once. This is intentional — the field controls the quiz's disclosure policy, not a per-attempt snapshot.

## 3. Result page behaviour

The result page renders three distinct modes:

### `score_only`
Score banner + question list showing question text and the student's answers only. No green/red marking. No correct answers. No explanations. The question text is visible so the student can review what they answered; only correctness information is withheld.

### `correctness`
Score banner + per-question green/red marking. Student's answer highlighted as correct or incorrect. Correct answer **not** revealed. No explanations shown.

### `full`
Everything in `correctness`, plus:
- Correct answer(s) highlighted
- Question-level `explanation` rendered as markdown, shown inline below the question body (if non-empty)
- Each choice that has a non-empty `explanation` shows a **"Why?"** toggle button; clicking it expands a callout beneath that choice

The "Why?" toggle is client-side only — all explanation text is rendered in the page HTML and toggled via CSS/JS. No extra requests.

**Shuffle compatibility:** choices on the result page must be rendered in the same order the student saw during the attempt, using the shuffled indices stored in `QuizAttempt.choice_orders[str(question_id)]`. Choice explanations travel with their choice objects, so they automatically appear next to the correct choice regardless of shuffle order.

## 4. Question editor UI

Each choice row in the dynamic editor has two stacked inputs:

```
[ Choice text (required)               ] [×]
[ Explanation (optional, single line)  ]
```

- Choice explanation is a single-line `<input type="text">` (not markdown — short "why" notes only)
- Leaving explanation blank means no "Why?" button appears for that choice on the result page
- The JS that adds/removes choice rows manages both sub-fields together; they travel as one object in the `choices` JSON array
- SA questions: no choice inputs (unchanged)
- TF questions: two fixed choice rows (True / False), both with explanation inputs

The question-level `explanation` remains a full markdown textarea (unchanged).

## 5. Import / export

### 5.1 XLSX template

Add `choice_explanation_1 … choice_explanation_6` columns interleaved with `choice_1 … choice_6`:

```
type | title | content
  | choice_1 | choice_explanation_1
  | choice_2 | choice_explanation_2
  | …
  | choice_6 | choice_explanation_6
  | correct | points | category | level | explanation | shuffle | ma_strategy
```

The prefix `choice_` disambiguates from the `explanation` column (question-level). Interleaving is more author-friendly than a separate block of 6 columns at the end.

At parse time, `choice_N` + `choice_explanation_N` pairs are zipped into `{"text": ..., "explanation": ...}` objects. Blank cells produce `""`. All `choice_explanation_N` columns are optional — authors can omit them entirely for questions without per-choice explanations.

The existing `explanation` column remains for the question-level explanation.

### 5.2 JSON schema

`choices` becomes an array of objects:

```json
"choices": [
  {"text": "Paris",  "explanation": "Capital of France."},
  {"text": "London", "explanation": ""},
  {"text": "Berlin", "explanation": ""}
]
```

The `explanation` key is optional; the parser defaults to `""` if absent (backwards-compatible with JSON files generated before this change).

### 5.3 Export

Reads `.text` and `.explanation` from each choice object and writes them to the interleaved XLSX columns. Round-trips cleanly.

## 6. Affected files (summary)

| File | Change |
|---|---|
| `quiz/models.py` | Merge `choices`/`choice_explanations`; replace `show_correctness`+`show_answers` with `result_feedback` |
| `quiz/migrations/` | New migration for both model changes |
| `quiz/grading.py` | No changes needed |
| `quiz/views/student.py` | `QuizResult`: replace `show_correctness`/`show_answers` context vars with `result_feedback` |
| `quiz/views/editor.py` | Question form: render choice rows with explanation sub-field |
| `quiz/importers/xlsx_fmt.py` | Parse/write interleaved `choice_explanation_N` columns |
| `quiz/importers/json_fmt.py` | Accept `choices` as objects; default `explanation` to `""` |
| `quiz/admin.py` | Replace two checkboxes with `result_feedback` dropdown |
| Templates `quiz/result.html` | Implement three feedback modes; "Why?" toggle |
| Templates `quiz/question_form.html` | Explanation sub-field per choice row |
| `resources/quiz.js` | "Why?" toggle; choice row add/remove handles both sub-fields |
| `quiz/tests/` | Update any test that constructs `choices` as `["A", "B", ...]` to use the new `[{"text": "A", "explanation": ""}, ...]` format |
