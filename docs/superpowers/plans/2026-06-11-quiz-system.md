# Quiz System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a standalone quiz app for lcoj-site: question bank + quizzes with problem-style access control, auto-graded MC/MA/TF/SA attempts with lazy timer enforcement, XLSX/JSON import with preview, and a per-quiz leaderboard.

**Architecture:** New self-contained Django app `quiz/` at the repo root of `dmoj/repo` (sibling of `judge/`), depending on judge only for `Profile`/`Organization`. Pure-function grading engine; views split student/editor/importer; templates in `templates/quiz/`. Spec: `docs/superpowers/specs/2026-06-11-quiz-system-design.md` (outer repo).

**Tech Stack:** Django 4.2, django_jinja templates, MariaDB (via docker), openpyxl (new dep), vanilla JS.

---

## Environment & conventions (read first)

- **Code lives in** `/home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo` (its own git repo, branch `feature/quiz-system`). All `git` commands below run there. All file paths below are relative to that repo root unless absolute.
- **Tests run inside the docker dev container.** From `/home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj`:
  `docker compose exec site python3 manage.py test quiz -v 2`
  (Start the stack first if needed: `docker compose up -d db redis site`.) Repo is volume-mounted at `/site/`, so edits are live.
- **Conventions** (verified against the codebase):
  - `INSTALLED_APPS` tuple in `dmoj/settings.py` (~line 400, end of the third-party block).
  - URLs: nested `path()`/`include()` in `dmoj/urls.py`, global snake_case names (`quiz_list`, `quiz_detail`, …), no namespaces.
  - Models FK to `judge.models.profile.Profile` / `Organization` with `on_delete=models.CASCADE` (see `judge/models/problem.py:154-162` for authors/curators/testers M2M style).
  - Templates: django_jinja, extend `"common-content.html"` (see `templates/problem/list.html` for the block structure — verify block names there: `{% block media %}`, `{% block body %}`). Markdown filter: `{{ text|markdown('default') }}`.
  - Auth: `LoginRequiredMixin`; middleware sets `request.profile`.
  - Test factories: reuse `judge.models.tests.util.create_user` (creates Profile; accepts `user_permissions=('codename', ...)`) and `create_organization`.
  - JSON endpoints: plain `JsonResponse`, CSRF handled by middleware (send `X-CSRFToken` header from JS).
- **Spec refinements decided during planning** (flag to reviewer): (1) the AJAX save endpoint never returns points/correctness — feedback belongs to the result page; returning it live would let students brute-force answers before submitting. (2) Quiz-edit question ordering uses integer `order` inputs, not drag-and-drop, in v1.

---

### Task 0: Branch, app scaffold, settings

**Files:**
- Create: `quiz/__init__.py`, `quiz/apps.py`, `quiz/models.py` (empty for now), `quiz/tests/__init__.py`
- Modify: `dmoj/settings.py` (INSTALLED_APPS), `requirements.txt`

- [ ] **Step 1: Create branch in dmoj/repo**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git checkout -b feature/quiz-system
```

- [ ] **Step 2: Verify test infra works at all** (smoke test before writing anything)

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose up -d db redis site
docker compose exec site python3 manage.py test judge.tests.test_editorial -v 1
```
Expected: existing tests PASS (or are skipped). If this fails, stop and report — the whole plan depends on it.

- [ ] **Step 3: Scaffold the app**

`quiz/__init__.py`: empty file.

`quiz/apps.py`:
```python
from django.apps import AppConfig


class QuizConfig(AppConfig):
    default_auto_field = 'django.db.models.AutoField'
    name = 'quiz'
    verbose_name = 'Quiz'
```

`quiz/models.py`: empty file for now. `quiz/tests/__init__.py`: empty file.

- [ ] **Step 4: Register app and dependency**

In `dmoj/settings.py`, add `'quiz',` as the last entry of the main `INSTALLED_APPS +=` tuple (after `'django_cleanup.apps.CleanupConfig',`).

In `requirements.txt`, add a line: `openpyxl`.

Install into the running dev container (image rebuild picks it up permanently later):
```bash
docker compose exec site pip3 install openpyxl
```

- [ ] **Step 5: Verify Django sees the app**

```bash
docker compose exec site python3 manage.py check
```
Expected: `System check identified no issues`.

- [ ] **Step 6: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add quiz/ dmoj/settings.py requirements.txt
git commit -m "feat(quiz): scaffold standalone quiz app"
```

---

### Task 1: Grading engine — MC/TF

**Files:**
- Create: `quiz/grading.py`
- Test: `quiz/tests/test_grading.py`

- [ ] **Step 1: Write the failing tests**

`quiz/tests/test_grading.py`:
```python
from django.test import SimpleTestCase

from quiz import grading


class GradeSingleChoiceTestCase(SimpleTestCase):
    CHOICES = ['a', 'b', 'c', 'd']

    def grade(self, answer, correct=1, qtype=grading.MC):
        return grading.grade(qtype, self.CHOICES, correct, None, answer)

    def test_correct_choice(self):
        self.assertEqual(self.grade(1), (1.0, True))

    def test_wrong_choice(self):
        self.assertEqual(self.grade(0), (0.0, False))

    def test_empty_answer(self):
        self.assertEqual(self.grade(None), (0.0, False))

    def test_no_answer_key(self):
        self.assertEqual(self.grade(1, correct=None), (0.0, False))

    def test_true_false(self):
        # TF stores the correct index: 0 = True, 1 = False.
        self.assertEqual(self.grade(0, correct=0, qtype=grading.TF), (1.0, True))
        self.assertEqual(self.grade(1, correct=0, qtype=grading.TF), (0.0, False))

    def test_unknown_type_raises(self):
        with self.assertRaises(ValueError):
            grading.grade('ES', [], 0, None, 0)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_grading -v 2
```
Expected: ERROR — `ModuleNotFoundError: No module named 'quiz.grading'` (or AttributeError).

- [ ] **Step 3: Write minimal implementation**

`quiz/grading.py`:
```python
"""Pure grading functions for quiz answers.

No database or Django model imports - everything here is unit-testable
standalone and is re-run verbatim by regrading.

Answer formats by question type:
  MC/TF - int choice index (TF: 0 = True, 1 = False) or None
  MA    - list of int choice indices or None
  SA    - str or None
"""
import re

MC = 'MC'
MA = 'MA'
TF = 'TF'
SA = 'SA'

MA_ALL_OR_NOTHING = 'all_or_nothing'
MA_PARTIAL_CREDIT = 'partial_credit'
MA_RIGHT_MINUS_WRONG = 'right_minus_wrong'
MA_CORRECT_ONLY = 'correct_only'

MA_STRATEGIES = (MA_ALL_OR_NOTHING, MA_PARTIAL_CREDIT, MA_RIGHT_MINUS_WRONG, MA_CORRECT_ONLY)


def grade(question_type, choices, correct_answers, ma_strategy, answer):
    """Grade one answer. Returns (ratio, is_correct) with ratio in [0, 1]."""
    if question_type in (MC, TF):
        return _grade_single(correct_answers, answer)
    raise ValueError('unknown question type: %r' % (question_type,))


def _grade_single(correct_answers, answer):
    if answer is None or correct_answers is None:
        return 0.0, False
    if int(answer) == int(correct_answers):
        return 1.0, True
    return 0.0, False
```

- [ ] **Step 4: Run tests to verify they pass**

Same command. Expected: 6 tests, all PASS.

- [ ] **Step 5: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add quiz/grading.py quiz/tests/test_grading.py
git commit -m "feat(quiz): grading engine for MC/TF questions"
```

---

### Task 2: Grading engine — MA strategies

**Files:**
- Modify: `quiz/grading.py`
- Test: `quiz/tests/test_grading.py`

- [ ] **Step 1: Write the failing tests** (append to `quiz/tests/test_grading.py`)

```python
class GradeMultipleAnswerTestCase(SimpleTestCase):
    # 5 choices A-E; correct = {0, 2} so |C| = 2, |W| = 3.
    CHOICES = ['a', 'b', 'c', 'd', 'e']
    CORRECT = [0, 2]

    def grade(self, selected, strategy):
        return grading.grade(grading.MA, self.CHOICES, self.CORRECT, strategy, selected)

    def test_exact_match_full_credit_all_strategies(self):
        for strategy in grading.MA_STRATEGIES:
            self.assertEqual(self.grade([0, 2], strategy), (1.0, True), strategy)

    def test_empty_answer_zero_all_strategies(self):
        for strategy in grading.MA_STRATEGIES:
            for answer in (None, []):
                self.assertEqual(self.grade(answer, strategy), (0.0, False), strategy)

    # Worked example from spec section 5: selected {0, 1} -> cs=1, ws=1.
    def test_all_or_nothing_partial_selection(self):
        self.assertEqual(self.grade([0, 1], grading.MA_ALL_OR_NOTHING), (0.0, False))

    def test_partial_credit(self):
        ratio, ok = self.grade([0, 1], grading.MA_PARTIAL_CREDIT)
        self.assertAlmostEqual(ratio, 1 / 2 - 1 / 3)
        self.assertFalse(ok)

    def test_right_minus_wrong_floors_at_zero(self):
        self.assertEqual(self.grade([0, 1], grading.MA_RIGHT_MINUS_WRONG), (0.0, False))
        self.assertEqual(self.grade([1, 3, 4], grading.MA_RIGHT_MINUS_WRONG), (0.0, False))

    def test_correct_only_no_penalty(self):
        self.assertEqual(self.grade([0, 1], grading.MA_CORRECT_ONLY), (0.5, False))
        # Selecting everything yields full ratio under correct_only - by design.
        self.assertEqual(self.grade([0, 1, 2, 3, 4], grading.MA_CORRECT_ONLY), (1.0, True))

    def test_partial_credit_no_wrong_choices_no_penalty_term(self):
        # All choices correct -> |W| = 0; penalty term must be 0, not div-by-zero.
        ratio, ok = grading.grade(grading.MA, ['a', 'b'], [0, 1], grading.MA_PARTIAL_CREDIT, [0])
        self.assertEqual((ratio, ok), (0.5, False))

    def test_empty_answer_key_zero(self):
        self.assertEqual(
            grading.grade(grading.MA, self.CHOICES, [], grading.MA_ALL_OR_NOTHING, [0]),
            (0.0, False))

    def test_unknown_strategy_defaults_to_all_or_nothing(self):
        self.assertEqual(self.grade([0, 2], None), (1.0, True))
        self.assertEqual(self.grade([0], None), (0.0, False))
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_grading -v 2
```
Expected: new tests ERROR with `ValueError: unknown question type: 'MA'`.

- [ ] **Step 3: Implement** — in `quiz/grading.py`, insert into `grade()` before the `raise`:

```python
    if question_type == MA:
        return _grade_multiple(choices, correct_answers, ma_strategy, answer)
```

and add at module end:

```python
def _grade_multiple(choices, correct_answers, ma_strategy, answer):
    correct_set = {int(i) for i in (correct_answers or [])}
    selected = {int(i) for i in (answer or [])}
    if not correct_set or not selected:
        return 0.0, False
    wrong_set = set(range(len(choices or []))) - correct_set
    cs = len(selected & correct_set)
    ws = len(selected & wrong_set)
    if ma_strategy == MA_PARTIAL_CREDIT:
        penalty = ws / len(wrong_set) if wrong_set else 0.0
        ratio = max(0.0, cs / len(correct_set) - penalty)
    elif ma_strategy == MA_RIGHT_MINUS_WRONG:
        ratio = max(0.0, (cs - ws) / len(correct_set))
    elif ma_strategy == MA_CORRECT_ONLY:
        ratio = cs / len(correct_set)
    else:  # all_or_nothing, also the fallback for unknown values
        ratio = 1.0 if selected == correct_set else 0.0
    return ratio, ratio == 1.0
```

- [ ] **Step 4: Run tests to verify they pass**

Same command. Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add quiz/grading.py quiz/tests/test_grading.py
git commit -m "feat(quiz): four MA grading strategies"
```

---

### Task 3: Grading engine — SA

**Files:**
- Modify: `quiz/grading.py`
- Test: `quiz/tests/test_grading.py`

- [ ] **Step 1: Write the failing tests** (append to `quiz/tests/test_grading.py`)

```python
class GradeShortAnswerTestCase(SimpleTestCase):
    def grade(self, answer, accepted):
        return grading.grade(grading.SA, [], accepted, None, answer)

    @staticmethod
    def acc(text, case_sensitive=False, is_regex=False):
        return {'text': text, 'case_sensitive': case_sensitive, 'is_regex': is_regex}

    def test_case_insensitive_default(self):
        self.assertEqual(self.grade('Hanoi', [self.acc('hanoi')]), (1.0, True))

    def test_case_sensitive(self):
        accepted = [self.acc('Hanoi', case_sensitive=True)]
        self.assertEqual(self.grade('hanoi', accepted), (0.0, False))
        self.assertEqual(self.grade('Hanoi', accepted), (1.0, True))

    def test_whitespace_stripped(self):
        self.assertEqual(self.grade('  42 ', [self.acc('42')]), (1.0, True))

    def test_first_match_wins_across_multiple_accepted(self):
        accepted = [self.acc('42'), self.acc('forty two')]
        self.assertEqual(self.grade('forty TWO', accepted), (1.0, True))

    def test_regex_fullmatch(self):
        accepted = [self.acc(r'4[0-9]', is_regex=True)]
        self.assertEqual(self.grade('42', accepted), (1.0, True))
        self.assertEqual(self.grade('142', accepted), (0.0, False))  # fullmatch, not search

    def test_regex_case_sensitivity(self):
        self.assertEqual(self.grade('ABC', [self.acc('abc', is_regex=True)]), (1.0, True))
        self.assertEqual(
            self.grade('ABC', [self.acc('abc', case_sensitive=True, is_regex=True)]),
            (0.0, False))

    def test_invalid_stored_regex_is_no_match_not_crash(self):
        self.assertEqual(self.grade('x', [self.acc('(', is_regex=True)]), (0.0, False))

    def test_empty_answer(self):
        self.assertEqual(self.grade('', [self.acc('42')]), (0.0, False))
        self.assertEqual(self.grade(None, [self.acc('42')]), (0.0, False))

    def test_no_match_is_zero(self):
        # Spec section 13: no flag-for-review - a non-match is simply wrong.
        self.assertEqual(self.grade('43', [self.acc('42')]), (0.0, False))
```

- [ ] **Step 2: Run tests to verify they fail**

Expected: ERROR with `ValueError: unknown question type: 'SA'`.

- [ ] **Step 3: Implement** — in `grade()` insert before the `raise`:

```python
    if question_type == SA:
        return _grade_short_answer(correct_answers, answer)
```

and add at module end:

```python
def _grade_short_answer(correct_answers, answer):
    text = (answer or '').strip()
    if not text:
        return 0.0, False
    for accepted in correct_answers or []:
        expected = (accepted.get('text') or '').strip()
        if accepted.get('is_regex'):
            flags = 0 if accepted.get('case_sensitive') else re.IGNORECASE
            try:
                if re.fullmatch(expected, text, flags):
                    return 1.0, True
            except re.error:
                continue  # validated at import time; never crash at grade time
        elif accepted.get('case_sensitive'):
            if text == expected:
                return 1.0, True
        elif text.lower() == expected.lower():
            return 1.0, True
    return 0.0, False
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_grading -v 2
```
Expected: all tests PASS (the full grading matrix).

- [ ] **Step 5: Commit**

```bash
git add quiz/grading.py quiz/tests/test_grading.py
git commit -m "feat(quiz): SA grading with case/regex accepted answers"
```

---

### Task 4: Models — enums, QuizCategory, QuizQuestion

**Files:**
- Modify: `quiz/models.py`
- Create: `quiz/tests/util.py`, `quiz/migrations/` (generated)
- Test: `quiz/tests/test_models.py`

- [ ] **Step 1: Write the failing tests**

`quiz/tests/util.py`:
```python
from judge.models.tests.util import create_organization, create_user  # noqa: F401
from quiz.models import Quiz, QuizCategory, QuizQuestion, QuizQuestionLink


def create_question(title='question', qtype='MC', **kwargs):
    defaults = {
        'content': 'Body of %s' % title,
        'choices': ['a', 'b', 'c', 'd'],
        'correct_answers': 0,
    }
    defaults.update(kwargs)
    authors = defaults.pop('authors', ())
    curators = defaults.pop('curators', ())
    question = QuizQuestion.objects.create(title=title, type=qtype, **defaults)
    question.authors.set(authors)
    question.curators.set(curators)
    return question


def create_quiz(code='quiz1', questions=(), **kwargs):
    defaults = {'name': code, 'is_public': True}
    defaults.update(kwargs)
    for field in ('authors', 'curators', 'testers', 'organizations'):
        defaults.pop(field, None)
    quiz = Quiz.objects.create(code=code, **defaults)
    for field in ('authors', 'curators', 'testers', 'organizations'):
        if field in kwargs:
            getattr(quiz, field).set(kwargs[field])
    for order, (question, points) in enumerate(questions):
        QuizQuestionLink.objects.create(quiz=quiz, question=question, points=points, order=order)
    return quiz
```

`quiz/tests/test_models.py`:
```python
from django.contrib.auth.models import AnonymousUser
from django.test import TestCase

from quiz.models import QuizCategory
from quiz.tests.util import create_question, create_user


class QuizQuestionTestCase(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.staff = create_user(username='staff', user_permissions=('edit_all_quiz',))
        cls.teacher = create_user(username='teacher', user_permissions=('edit_own_quiz',))
        cls.other_teacher = create_user(username='other_teacher', user_permissions=('edit_own_quiz',))
        cls.student = create_user(username='student')
        cls.category = QuizCategory.objects.create(name='Python basics', slug='python-basics')
        cls.question = create_question(
            title='loops', category=cls.category, authors=(cls.teacher.profile,))

    def test_str(self):
        self.assertEqual(str(self.question), 'loops')

    def test_grade_delegates_to_grading_engine(self):
        self.assertEqual(self.question.grade(0), (1.0, True))
        self.assertEqual(self.question.grade(1), (0.0, False))

    def test_editable_by_author_and_staff_only(self):
        self.assertTrue(self.question.is_editable_by(self.teacher))
        self.assertTrue(self.question.is_editable_by(self.staff))
        self.assertFalse(self.question.is_editable_by(self.other_teacher))
        self.assertFalse(self.question.is_editable_by(self.student))
        self.assertFalse(self.question.is_editable_by(AnonymousUser()))

    def test_curator_can_edit(self):
        self.question.curators.add(self.other_teacher.profile)
        self.assertTrue(self.question.is_editable_by(self.other_teacher))

    def test_bank_visibility(self):
        # Private question: only editors of it (and staff) see it in the bank.
        self.assertTrue(self.question.is_visible_in_bank(self.teacher))
        self.assertTrue(self.question.is_visible_in_bank(self.staff))
        self.assertFalse(self.question.is_visible_in_bank(self.other_teacher))
        # Public question: any quiz editor sees it; students still don't.
        self.question.is_public = True
        self.assertTrue(self.question.is_visible_in_bank(self.other_teacher))
        self.assertFalse(self.question.is_visible_in_bank(self.student))
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_models -v 2
```
Expected: ImportError (models don't exist yet).

- [ ] **Step 3: Implement models**

`quiz/models.py`:
```python
from django.core.validators import MinValueValidator, RegexValidator
from django.db import models
from django.urls import reverse
from django.utils.translation import gettext_lazy as _

from judge.models.profile import Organization, Profile
from quiz import grading


class QuizLevel(models.TextChoices):
    EASY = 'easy', _('Easy')
    MEDIUM = 'medium', _('Medium')
    HARD = 'hard', _('Hard')


class QuestionType(models.TextChoices):
    MULTIPLE_CHOICE = grading.MC, _('Multiple Choice')
    MULTIPLE_ANSWER = grading.MA, _('Multiple Answer')
    TRUE_FALSE = grading.TF, _('True/False')
    SHORT_ANSWER = grading.SA, _('Short Answer')


MA_STRATEGY_CHOICES = [
    (grading.MA_ALL_OR_NOTHING, _('All or nothing')),
    (grading.MA_PARTIAL_CREDIT, _('Partial credit with penalty')),
    (grading.MA_RIGHT_MINUS_WRONG, _('Right minus wrong')),
    (grading.MA_CORRECT_ONLY, _('Correct only, no penalty')),
]


class QuizCategory(models.Model):
    name = models.CharField(max_length=100, verbose_name=_('name'), unique=True)
    slug = models.SlugField(max_length=100, verbose_name=_('slug'), unique=True)
    description = models.TextField(verbose_name=_('description'), blank=True)

    class Meta:
        ordering = ('name',)
        verbose_name = _('quiz category')
        verbose_name_plural = _('quiz categories')

    def __str__(self):
        return self.name


class QuizQuestion(models.Model):
    type = models.CharField(max_length=2, verbose_name=_('question type'), choices=QuestionType.choices)
    title = models.CharField(max_length=200, verbose_name=_('title'),
                             help_text=_('Short name to identify this question in the bank.'))
    content = models.TextField(verbose_name=_('question body'))
    choices = models.JSONField(verbose_name=_('choices'), default=list, blank=True,
                               help_text=_('List of choice texts (MC/MA only).'))
    correct_answers = models.JSONField(verbose_name=_('correct answers'), null=True, blank=True,
                                       help_text=_('MC/TF: choice index. MA: list of indices. '
                                                   'SA: list of {text, case_sensitive, is_regex}.'))
    explanation = models.TextField(verbose_name=_('explanation'), blank=True,
                                   help_text=_('Shown on the result page when the quiz allows it.'))
    category = models.ForeignKey(QuizCategory, verbose_name=_('category'), null=True, blank=True,
                                 on_delete=models.SET_NULL, related_name='questions')
    level = models.CharField(max_length=6, verbose_name=_('level'), choices=QuizLevel.choices,
                             default=QuizLevel.EASY)
    shuffle_choices = models.BooleanField(verbose_name=_('shuffle choices'), default=False)
    ma_grading_strategy = models.CharField(max_length=20, verbose_name=_('MA grading strategy'),
                                           choices=MA_STRATEGY_CHOICES,
                                           default=grading.MA_ALL_OR_NOTHING)
    is_public = models.BooleanField(verbose_name=_('publicly visible in bank'), default=False,
                                    help_text=_('Visible to every quiz editor, not only authors.'))
    authors = models.ManyToManyField(Profile, verbose_name=_('creators'), blank=True,
                                     related_name='authored_quiz_questions')
    curators = models.ManyToManyField(Profile, verbose_name=_('curators'), blank=True,
                                      related_name='curated_quiz_questions')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ('-id',)
        verbose_name = _('quiz question')
        verbose_name_plural = _('quiz questions')

    def __str__(self):
        return self.title

    def grade(self, answer):
        return grading.grade(self.type, self.choices, self.correct_answers,
                             self.ma_grading_strategy, answer)

    def is_editable_by(self, user):
        if not user.is_authenticated:
            return False
        if user.has_perm('quiz.edit_all_quiz'):
            return True
        if not user.has_perm('quiz.edit_own_quiz'):
            return False
        return self.authors.filter(id=user.profile.id).exists() or \
            self.curators.filter(id=user.profile.id).exists()

    def is_visible_in_bank(self, user):
        if self.is_editable_by(user):
            return True
        return self.is_public and user.is_authenticated and \
            user.has_perm('quiz.edit_own_quiz')
```

The app-level permissions live permanently on `QuizQuestion.Meta` (Django checks them as `quiz.edit_all_quiz` / `quiz.edit_own_quiz` regardless of which model declares them). Add to the `Meta` class above:

```python
        permissions = (
            ('edit_all_quiz', _('Edit all quizzes and questions')),
            ('edit_own_quiz', _('Edit own quizzes and questions')),
        )
```

(Do NOT redeclare these on `Quiz` in Task 5.)

- [ ] **Step 4: Generate migration and run tests**

```bash
docker compose exec site python3 manage.py makemigrations quiz
docker compose exec site python3 manage.py test quiz.tests.test_models -v 2
```
Expected: migration `quiz/migrations/0001_initial.py` created; all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add quiz/models.py quiz/migrations/ quiz/tests/
git commit -m "feat(quiz): QuizCategory and QuizQuestion models with bank permissions"
```

---

### Task 5: Models — Quiz, QuizQuestionLink, access control

**Files:**
- Modify: `quiz/models.py`, `quiz/tests/test_models.py`
- Create: migration (generated)

- [ ] **Step 1: Write the failing tests** (append to `quiz/tests/test_models.py`)

```python
from quiz.models import Quiz  # add to imports at top
from quiz.tests.util import create_organization, create_quiz  # add to imports at top


class QuizAccessTestCase(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.staff = create_user(username='qstaff', user_permissions=('edit_all_quiz',))
        cls.author = create_user(username='qauthor', user_permissions=('edit_own_quiz',))
        cls.tester = create_user(username='qtester')
        cls.student = create_user(username='qstudent')
        cls.org_member = create_user(username='qorguser')
        cls.org = create_organization(name='testorg', admins=())
        cls.org_member.profile.organizations.add(cls.org)

        cls.public = create_quiz(code='public')
        cls.private = create_quiz(code='private', is_public=False,
                                  authors=(cls.author.profile,), testers=(cls.tester.profile,))
        cls.org_quiz = create_quiz(code='orgquiz', is_organization_private=True,
                                   organizations=(cls.org,))

    def test_public_quiz_accessible_to_everyone(self):
        self.assertTrue(self.public.is_accessible_by(AnonymousUser()))
        self.assertTrue(self.public.is_accessible_by(self.student))

    def test_private_quiz_access(self):
        self.assertFalse(self.private.is_accessible_by(self.student))
        self.assertFalse(self.private.is_accessible_by(AnonymousUser()))
        self.assertTrue(self.private.is_accessible_by(self.author))
        self.assertTrue(self.private.is_accessible_by(self.tester))
        self.assertTrue(self.private.is_accessible_by(self.staff))

    def test_org_private_quiz_access(self):
        self.assertTrue(self.org_quiz.is_accessible_by(self.org_member))
        self.assertFalse(self.org_quiz.is_accessible_by(self.student))
        self.assertFalse(self.org_quiz.is_accessible_by(AnonymousUser()))

    def test_editable_by(self):
        self.assertTrue(self.private.is_editable_by(self.author))
        self.assertTrue(self.private.is_editable_by(self.staff))
        self.assertFalse(self.private.is_editable_by(self.tester))
        self.assertFalse(self.private.is_editable_by(self.student))

    def test_get_visible_quizzes_matrix(self):
        def codes(user):
            return set(Quiz.get_visible_quizzes(user).values_list('code', flat=True))

        self.assertEqual(codes(AnonymousUser()), {'public'})
        self.assertEqual(codes(self.student), {'public'})
        self.assertEqual(codes(self.org_member), {'public', 'orgquiz'})
        self.assertEqual(codes(self.tester), {'public', 'private'})
        self.assertEqual(codes(self.author), {'public', 'private'})
        self.assertEqual(codes(self.staff), {'public', 'private', 'orgquiz'})

    def test_total_points(self):
        question_a = create_question(title='qa')
        question_b = create_question(title='qb')
        quiz = create_quiz(code='points', questions=((question_a, 1.5), (question_b, 2.0)))
        self.assertEqual(quiz.total_points, 3.5)
        self.assertEqual(create_quiz(code='nopoints').total_points, 0.0)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_models -v 2
```
Expected: ImportError — `cannot import name 'Quiz'`.

- [ ] **Step 3: Implement** — append to `quiz/models.py`:

```python
class Quiz(models.Model):
    code = models.CharField(max_length=32, verbose_name=_('quiz code'), unique=True,
                            validators=[RegexValidator('^[a-z0-9_]+$',
                                                       _('Quiz code must be ^[a-z0-9_]+$'))])
    name = models.CharField(max_length=100, verbose_name=_('quiz name'))
    description = models.TextField(verbose_name=_('description'), blank=True)
    category = models.ForeignKey(QuizCategory, verbose_name=_('category'), null=True, blank=True,
                                 on_delete=models.SET_NULL, related_name='quizzes')
    level = models.CharField(max_length=6, verbose_name=_('level'), choices=QuizLevel.choices,
                             default=QuizLevel.EASY)
    time_limit = models.PositiveIntegerField(verbose_name=_('time limit (minutes)'),
                                             null=True, blank=True,
                                             help_text=_('Leave blank for unlimited time.'))
    max_attempts = models.PositiveIntegerField(verbose_name=_('maximum attempts'),
                                               null=True, blank=True,
                                               help_text=_('Leave blank for unlimited attempts.'))
    shuffle_questions = models.BooleanField(verbose_name=_('shuffle questions'), default=False)
    show_correctness = models.BooleanField(verbose_name=_('show correctness on result'), default=True)
    show_answers = models.BooleanField(verbose_name=_('show correct answers on result'), default=True)
    is_public = models.BooleanField(verbose_name=_('publicly visible'), default=False)
    is_organization_private = models.BooleanField(verbose_name=_('private to organizations'),
                                                  default=False)
    organizations = models.ManyToManyField(Organization, verbose_name=_('organizations'), blank=True,
                                           related_name='quizzes')
    authors = models.ManyToManyField(Profile, verbose_name=_('creators'), blank=True,
                                     related_name='authored_quizzes')
    curators = models.ManyToManyField(Profile, verbose_name=_('curators'), blank=True,
                                      related_name='curated_quizzes')
    testers = models.ManyToManyField(Profile, verbose_name=_('testers'), blank=True,
                                     related_name='tested_quizzes',
                                     help_text=_('These users can take the quiz while it is private.'))
    questions = models.ManyToManyField(QuizQuestion, through='QuizQuestionLink',
                                       related_name='quizzes')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ('-created_at',)
        verbose_name = _('quiz')
        verbose_name_plural = _('quizzes')

    def __str__(self):
        return self.name

    def get_absolute_url(self):
        return reverse('quiz_detail', args=(self.code,))

    @property
    def total_points(self):
        return self.question_links.aggregate(total=models.Sum('points'))['total'] or 0.0

    def is_editable_by(self, user):
        if not user.is_authenticated:
            return False
        if user.has_perm('quiz.edit_all_quiz'):
            return True
        if not user.has_perm('quiz.edit_own_quiz'):
            return False
        return self.authors.filter(id=user.profile.id).exists() or \
            self.curators.filter(id=user.profile.id).exists()

    def is_accessible_by(self, user):
        if self.is_public:
            if not self.is_organization_private:
                return True
            if user.is_authenticated and \
                    self.organizations.filter(id__in=user.profile.organizations.all()).exists():
                return True
        if not user.is_authenticated:
            return False
        if self.is_editable_by(user):
            return True
        return self.testers.filter(id=user.profile.id).exists()

    @classmethod
    def get_visible_quizzes(cls, user):
        if user.is_authenticated and user.has_perm('quiz.edit_all_quiz'):
            return cls.objects.all()
        filters = models.Q(is_public=True, is_organization_private=False)
        if user.is_authenticated:
            profile = user.profile
            filters |= models.Q(is_public=True, is_organization_private=True,
                                organizations__in=profile.organizations.all())
            filters |= models.Q(authors=profile) | models.Q(curators=profile) | \
                models.Q(testers=profile)
        return cls.objects.filter(filters).distinct()


class QuizQuestionLink(models.Model):
    quiz = models.ForeignKey(Quiz, on_delete=models.CASCADE, related_name='question_links')
    question = models.ForeignKey(QuizQuestion, on_delete=models.CASCADE, related_name='quiz_links')
    points = models.FloatField(verbose_name=_('points'), default=1.0,
                               validators=[MinValueValidator(0)])
    order = models.IntegerField(verbose_name=_('order'), default=0)

    class Meta:
        unique_together = ('quiz', 'question')
        ordering = ('order', 'id')
        verbose_name = _('quiz question link')
        verbose_name_plural = _('quiz question links')

    def __str__(self):
        return '%s in %s' % (self.question, self.quiz)
```

Note: `create_quiz` in `quiz/tests/util.py` was already written in Task 4 to support these fields.

- [ ] **Step 4: Generate migration and run tests**

```bash
docker compose exec site python3 manage.py makemigrations quiz
docker compose exec site python3 manage.py test quiz.tests.test_models -v 2
```
Expected: new migration; all tests PASS. Important: org-membership test relies on `Profile.organizations` M2M existing in judge (verify with `docker compose exec site python3 manage.py shell -c "from judge.models import Profile; Profile.organizations"`).

- [ ] **Step 5: Commit**

```bash
git add quiz/models.py quiz/migrations/ quiz/tests/
git commit -m "feat(quiz): Quiz model with problem-style access control"
```

---

### Task 6: Models — QuizAttempt, QuizAnswer, lifecycle

**Files:**
- Modify: `quiz/models.py`
- Test: `quiz/tests/test_attempts.py`
- Create: migration (generated)

- [ ] **Step 1: Write the failing tests**

`quiz/tests/test_attempts.py`:
```python
from datetime import timedelta

from django.test import TestCase
from django.utils import timezone

from quiz.models import QuizAttempt
from quiz.tests.util import create_question, create_quiz, create_user


class QuizAttemptTestCase(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.student = create_user(username='attempt_student')
        cls.q_mc = create_question(title='mc1')                       # correct: 0
        cls.q_sa = create_question(
            title='sa1', qtype='SA', choices=[],
            correct_answers=[{'text': '42', 'case_sensitive': False, 'is_regex': False}])
        cls.quiz = create_quiz(code='attemptquiz', time_limit=10,
                               questions=((cls.q_mc, 2.0), (cls.q_sa, 1.0)))

    def start(self):
        return QuizAttempt.start(self.quiz, self.student.profile)

    def test_start_freezes_question_order(self):
        attempt = self.start()
        self.assertEqual(attempt.question_order, [self.q_mc.id, self.q_sa.id])
        self.assertFalse(attempt.is_submitted)

    def test_start_shuffles_when_enabled(self):
        self.quiz.shuffle_questions = True
        attempt = self.start()
        self.assertEqual(sorted(attempt.question_order), sorted([self.q_mc.id, self.q_sa.id]))

    def test_choice_order_frozen_for_shuffled_questions(self):
        self.q_mc.shuffle_choices = True
        self.q_mc.save()
        attempt = self.start()
        frozen = attempt.choice_orders[str(self.q_mc.id)]
        self.assertEqual(sorted(frozen), [0, 1, 2, 3])

    def test_deadline_and_expiry(self):
        attempt = self.start()
        self.assertEqual(attempt.deadline, attempt.started_at + timedelta(minutes=10))
        self.assertFalse(attempt.has_expired())
        # 30s grace: just past deadline is not yet expired.
        just_past = attempt.deadline + timedelta(seconds=10)
        self.assertFalse(attempt.has_expired(now=just_past))
        self.assertTrue(attempt.has_expired(now=attempt.deadline + timedelta(seconds=31)))

    def test_no_time_limit_never_expires(self):
        quiz = create_quiz(code='untimed', questions=((self.q_mc, 1.0),))
        attempt = QuizAttempt.start(quiz, self.student.profile)
        self.assertIsNone(attempt.deadline)
        self.assertFalse(attempt.has_expired(now=timezone.now() + timedelta(days=365)))

    def test_save_answer_grades_immediately(self):
        attempt = self.start()
        answer = attempt.save_answer(self.q_mc, 0)
        self.assertEqual(answer.points, 2.0)
        self.assertTrue(answer.is_correct)
        # Upsert: changing the answer regrades, doesn't duplicate.
        answer = attempt.save_answer(self.q_mc, 1)
        self.assertEqual(answer.points, 0.0)
        self.assertEqual(attempt.answers.count(), 1)

    def test_finalize_scores_and_is_idempotent(self):
        attempt = self.start()
        attempt.save_answer(self.q_mc, 0)
        attempt.save_answer(self.q_sa, ' 42 ')
        attempt.finalize()
        self.assertTrue(attempt.is_submitted)
        self.assertEqual(attempt.score, 3.0)
        self.assertIsNotNone(attempt.submitted_at)
        first_submitted_at = attempt.submitted_at
        attempt.finalize()  # no-op
        self.assertEqual(attempt.submitted_at, first_submitted_at)

    def test_finalize_after_expiry_clamps_submitted_at_to_deadline(self):
        attempt = self.start()
        attempt.save_answer(self.q_mc, 0)
        late = attempt.deadline + timedelta(minutes=5)
        attempt.finalize(now=late)
        self.assertEqual(attempt.submitted_at, attempt.deadline)
        self.assertEqual(attempt.duration, timedelta(minutes=10))

    def test_regrade_after_answer_key_change(self):
        attempt = self.start()
        attempt.save_answer(self.q_mc, 1)  # wrong under current key
        attempt.finalize()
        self.assertEqual(attempt.score, 0.0)
        self.q_mc.correct_answers = 1      # teacher fixes the key
        self.q_mc.save()
        self.quiz.regrade_attempts()
        attempt.refresh_from_db()
        self.assertEqual(attempt.score, 2.0)

    def test_ranking_best_attempt_per_user_with_duration_tiebreak(self):
        fast = create_user(username='rank_fast')
        slow = create_user(username='rank_slow')
        low = create_user(username='rank_low')
        now = timezone.now()
        for user, score, minutes in ((fast, 3.0, 3), (slow, 3.0, 7), (low, 1.0, 2)):
            attempt = QuizAttempt.start(self.quiz, user.profile)
            attempt.started_at = now
            attempt.score = score
            attempt.is_submitted = True
            attempt.submitted_at = now + timedelta(minutes=minutes)
            attempt.save()
        # A second, worse attempt for 'fast' must not appear.
        worse = QuizAttempt.start(self.quiz, fast.profile)
        worse.started_at = now
        worse.score = 0.5
        worse.is_submitted = True
        worse.submitted_at = now + timedelta(minutes=1)
        worse.save()
        ranking = self.quiz.get_ranking()
        self.assertEqual([a.user_id for a in ranking],
                         [fast.profile.id, slow.profile.id, low.profile.id])
        self.assertEqual(ranking[0].score, 3.0)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_attempts -v 2
```
Expected: ImportError — `cannot import name 'QuizAttempt'`.

- [ ] **Step 3: Implement** — add `import random` and `from datetime import timedelta` + `from django.utils import timezone` to the imports of `quiz/models.py`, then append:

```python
class QuizAttempt(models.Model):
    GRACE = timedelta(seconds=30)

    quiz = models.ForeignKey(Quiz, on_delete=models.CASCADE, related_name='attempts')
    user = models.ForeignKey(Profile, on_delete=models.CASCADE, related_name='quiz_attempts')
    started_at = models.DateTimeField(verbose_name=_('started at'), default=timezone.now)
    submitted_at = models.DateTimeField(verbose_name=_('submitted at'), null=True, blank=True)
    is_submitted = models.BooleanField(default=False)
    score = models.FloatField(verbose_name=_('score'), default=0.0)
    question_order = models.JSONField(default=list)
    choice_orders = models.JSONField(default=dict)

    class Meta:
        indexes = (
            models.Index(fields=('quiz', 'user', 'is_submitted')),
            models.Index(fields=('quiz', 'is_submitted', 'score')),
        )
        verbose_name = _('quiz attempt')
        verbose_name_plural = _('quiz attempts')

    def __str__(self):
        return '%s on %s' % (self.user, self.quiz)

    @classmethod
    def start(cls, quiz, profile):
        """Create an attempt with question and choice order frozen for resume."""
        links = list(quiz.question_links.select_related('question'))
        order = [link.question_id for link in links]
        if quiz.shuffle_questions:
            random.shuffle(order)
        choice_orders = {}
        for link in links:
            question = link.question
            if question.shuffle_choices and question.choices:
                indices = list(range(len(question.choices)))
                random.shuffle(indices)
                choice_orders[str(question.id)] = indices
        return cls.objects.create(quiz=quiz, user=profile,
                                  question_order=order, choice_orders=choice_orders)

    @property
    def deadline(self):
        if self.quiz.time_limit is None:
            return None
        return self.started_at + timedelta(minutes=self.quiz.time_limit)

    def has_expired(self, now=None):
        deadline = self.deadline
        if deadline is None:
            return False
        return (now or timezone.now()) > deadline + self.GRACE

    def time_remaining_seconds(self, now=None):
        deadline = self.deadline
        if deadline is None:
            return None
        return max(0, int((deadline - (now or timezone.now())).total_seconds()))

    @property
    def duration(self):
        if self.submitted_at is None:
            return None
        return self.submitted_at - self.started_at

    def save_answer(self, question, answer):
        """Upsert and immediately grade one answer."""
        link = self.quiz.question_links.get(question=question)
        ratio, is_correct = question.grade(answer)
        obj, _created = QuizAnswer.objects.update_or_create(
            attempt=self, question=question,
            defaults={'answer': answer,
                      'points': round(ratio * link.points, 2),
                      'is_correct': is_correct})
        return obj

    def _grade_answers(self):
        points_by_question = {link.question_id: link.points
                              for link in self.quiz.question_links.all()}
        total = 0.0
        for answer in self.answers.select_related('question'):
            ratio, is_correct = answer.question.grade(answer.answer)
            answer.points = round(ratio * points_by_question.get(answer.question_id, 0.0), 2)
            answer.is_correct = is_correct
            answer.save(update_fields=('points', 'is_correct'))
            total += answer.points
        return round(total, 2)

    def finalize(self, now=None):
        """Grade everything and mark submitted. Idempotent; clamps to deadline."""
        if self.is_submitted:
            return
        now = now or timezone.now()
        self.score = self._grade_answers()
        self.is_submitted = True
        deadline = self.deadline
        self.submitted_at = min(now, deadline) if deadline is not None else now
        self.save(update_fields=('score', 'is_submitted', 'submitted_at'))

    def regrade(self):
        self.score = self._grade_answers()
        self.save(update_fields=('score',))


class QuizAnswer(models.Model):
    attempt = models.ForeignKey(QuizAttempt, on_delete=models.CASCADE, related_name='answers')
    question = models.ForeignKey(QuizQuestion, on_delete=models.CASCADE, related_name='+')
    answer = models.JSONField(null=True, blank=True)
    points = models.FloatField(default=0.0)
    is_correct = models.BooleanField(default=False)
    saved_at = models.DateTimeField(auto_now=True)

    class Meta:
        unique_together = ('attempt', 'question')
        verbose_name = _('quiz answer')
        verbose_name_plural = _('quiz answers')
```

And append these two methods to the `Quiz` class (after `get_visible_quizzes`):

```python
    def regrade_attempts(self):
        """Re-run grading over every submitted attempt (answer key fixed, etc.)."""
        count = 0
        for attempt in self.attempts.filter(is_submitted=True):
            attempt.regrade()
            count += 1
        return count

    def get_ranking(self):
        """Best submitted attempt per user; ties broken by shorter duration,
        then earlier submission. Python-side aggregation - attempt volumes are
        small and MySQL lacks DISTINCT ON."""
        best = {}
        for attempt in self.attempts.filter(is_submitted=True).select_related('user__user'):
            key = (-attempt.score, attempt.duration, attempt.submitted_at)
            current = best.get(attempt.user_id)
            if current is None or key < current[0]:
                best[attempt.user_id] = (key, attempt)
        return [attempt for _key, attempt in sorted(best.values(), key=lambda pair: pair[0])]
```

- [ ] **Step 4: Generate migration and run tests**

```bash
docker compose exec site python3 manage.py makemigrations quiz
docker compose exec site python3 manage.py test quiz -v 2
```
Expected: new migration; ALL quiz tests pass (grading + models + attempts).

- [ ] **Step 5: Commit**

```bash
git add quiz/models.py quiz/migrations/ quiz/tests/
git commit -m "feat(quiz): attempt lifecycle with lazy timer, regrade, ranking"
```

---

### Task 7: URLs, quiz list and detail pages

**Files:**
- Create: `quiz/views/__init__.py`, `quiz/views/student.py`, `quiz/urls.py`, `templates/quiz/list.html`, `templates/quiz/detail.html`
- Modify: `dmoj/urls.py`
- Test: `quiz/tests/test_views.py`

- [ ] **Step 1: Write the failing tests**

`quiz/tests/test_views.py`:
```python
from django.test import TestCase
from django.urls import reverse

from quiz.tests.util import create_quiz, create_user


class QuizListViewTestCase(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.student = create_user(username='view_student')
        cls.public = create_quiz(code='vpublic', name='Visible Quiz')
        cls.private = create_quiz(code='vprivate', is_public=False)

    def test_list_shows_only_visible(self):
        response = self.client.get(reverse('quiz_list'))
        self.assertContains(response, 'Visible Quiz')
        self.assertNotContains(response, 'vprivate')

    def test_list_filters_by_level(self):
        create_quiz(code='vhard', name='Hard One', level='hard')
        response = self.client.get(reverse('quiz_list'), {'level': 'hard'})
        self.assertContains(response, 'Hard One')
        self.assertNotContains(response, 'Visible Quiz')

    def test_detail_public_accessible_anonymously(self):
        response = self.client.get(reverse('quiz_detail', args=('vpublic',)))
        self.assertEqual(response.status_code, 200)

    def test_detail_private_404s_for_student(self):
        self.client.force_login(self.student)
        response = self.client.get(reverse('quiz_detail', args=('vprivate',)))
        self.assertEqual(response.status_code, 404)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_views -v 2
```
Expected: `NoReverseMatch: Reverse for 'quiz_list' not found`.

- [ ] **Step 3: Implement views**

`quiz/views/__init__.py`: empty file.

`quiz/views/student.py`:
```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.db.models import Max, Q
from django.http import Http404
from django.shortcuts import get_object_or_404
from django.views.generic import ListView, TemplateView

from quiz.models import Quiz, QuizAttempt, QuizCategory, QuizLevel


class QuizList(ListView):
    model = Quiz
    template_name = 'quiz/list.html'
    context_object_name = 'quizzes'
    paginate_by = 50

    def get_queryset(self):
        queryset = Quiz.get_visible_quizzes(self.request.user) \
            .select_related('category').order_by('-created_at')
        category = self.request.GET.get('category')
        if category:
            queryset = queryset.filter(category__slug=category)
        level = self.request.GET.get('level')
        if level in QuizLevel.values:
            queryset = queryset.filter(level=level)
        search = self.request.GET.get('search')
        if search:
            queryset = queryset.filter(Q(name__icontains=search) | Q(code__icontains=search))
        return queryset

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['categories'] = QuizCategory.objects.all()
        context['levels'] = QuizLevel.choices
        context['selected_category'] = self.request.GET.get('category', '')
        context['selected_level'] = self.request.GET.get('level', '')
        context['search'] = self.request.GET.get('search', '')
        best_scores = {}
        if self.request.user.is_authenticated:
            rows = QuizAttempt.objects.filter(
                user=self.request.profile, is_submitted=True,
                quiz__in=context['quizzes'],
            ).values('quiz_id').annotate(best=Max('score'))
            best_scores = {row['quiz_id']: row['best'] for row in rows}
        context['best_scores'] = best_scores
        return context


class QuizMixin:
    """Resolves self.quiz from the URL and 404s if not accessible."""

    def dispatch(self, request, *args, **kwargs):
        self.quiz = get_object_or_404(Quiz, code=kwargs['quiz'])
        if not self.quiz.is_accessible_by(request.user):
            raise Http404()
        return super().dispatch(request, *args, **kwargs)

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['quiz'] = self.quiz
        return context


class QuizDetail(QuizMixin, TemplateView):
    template_name = 'quiz/detail.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['question_count'] = self.quiz.question_links.count()
        context['total_points'] = self.quiz.total_points
        context['can_edit'] = self.quiz.is_editable_by(self.request.user)
        if self.request.user.is_authenticated:
            attempts = list(self.quiz.attempts.filter(user=self.request.profile)
                            .order_by('-started_at'))
            for attempt in attempts:
                if not attempt.is_submitted and attempt.has_expired():
                    attempt.finalize()
            context['attempts'] = attempts
            context['in_progress'] = next(
                (a for a in attempts if not a.is_submitted), None)
            if self.quiz.max_attempts is not None:
                used = sum(1 for a in attempts if a.is_submitted)
                context['attempts_left'] = max(0, self.quiz.max_attempts - used)
            else:
                context['attempts_left'] = None
        return context
```

In `dmoj/urls.py`, next to the `problems` block (after its `])),` line), add the single mount point:

```python
    path('quizzes', include('quiz.urls')),
```

`quiz/urls.py` (everything lives under `/quizzes`; leading-slash subpaths match the site convention seen in the `problems` block):

```python
from django.urls import include, path

from quiz.views import student

urlpatterns = [
    path('/', student.QuizList.as_view(), name='quiz_list'),
    path('/<str:quiz>', include([
        path('', student.QuizDetail.as_view(), name='quiz_detail'),
    ])),
]
```

- [ ] **Step 4: Create templates**

First verify block names: `head -30 templates/common-content.html` — adjust `body`/`media` block names below if they differ.

`templates/quiz/list.html`:
```jinja2
{% extends "common-content.html" %}

{% block body %}
<form method="get" style="margin-bottom: 1em">
    <input type="text" name="search" value="{{ search }}" placeholder="{{ _('Search quizzes...') }}">
    <select name="category">
        <option value="">{{ _('All categories') }}</option>
        {% for cat in categories %}
            <option value="{{ cat.slug }}" {% if cat.slug == selected_category %}selected{% endif %}>{{ cat.name }}</option>
        {% endfor %}
    </select>
    <select name="level">
        <option value="">{{ _('All levels') }}</option>
        {% for value, label in levels %}
            <option value="{{ value }}" {% if value == selected_level %}selected{% endif %}>{{ label }}</option>
        {% endfor %}
    </select>
    <button type="submit">{{ _('Filter') }}</button>
</form>
<table class="table">
    <thead>
        <tr>
            <th>{{ _('Quiz') }}</th>
            <th>{{ _('Category') }}</th>
            <th>{{ _('Level') }}</th>
            <th>{{ _('Your best') }}</th>
        </tr>
    </thead>
    <tbody>
        {% for quiz in quizzes %}
        <tr>
            <td><a href="{{ url('quiz_detail', quiz.code) }}">{{ quiz.name }}</a></td>
            <td>{{ quiz.category.name if quiz.category else '' }}</td>
            <td>{{ quiz.get_level_display() }}</td>
            <td>{{ best_scores.get(quiz.id, '') }}</td>
        </tr>
        {% else %}
        <tr><td colspan="4">{{ _('No quizzes found.') }}</td></tr>
        {% endfor %}
    </tbody>
</table>
{% endblock %}
```

`templates/quiz/detail.html`:
```jinja2
{% extends "common-content.html" %}

{% block body %}
<div class="quiz-info">
    <p>{{ quiz.description|markdown('default') }}</p>
    <ul>
        <li>{{ _('Questions') }}: {{ question_count }} ({{ total_points }} {{ _('points') }})</li>
        <li>{{ _('Time limit') }}: {% if quiz.time_limit %}{{ quiz.time_limit }} {{ _('minutes') }}{% else %}{{ _('unlimited') }}{% endif %}</li>
        <li>{{ _('Attempts') }}: {% if quiz.max_attempts %}{{ quiz.max_attempts }}{% else %}{{ _('unlimited') }}{% endif %}</li>
        <li><a href="{{ url('quiz_ranking', quiz.code) }}">{{ _('Ranking') }}</a></li>
    </ul>
    {% if can_edit %}<p><a href="{{ url('quiz_edit', quiz.code) }}">{{ _('Edit quiz') }}</a></p>{% endif %}
</div>
{% if request.user.is_authenticated %}
    <form action="{{ url('quiz_start', quiz.code) }}" method="post">
        {% csrf_token %}
        {% if in_progress %}
            <button type="submit">{{ _('Resume attempt') }}</button>
        {% elif attempts_left is none or attempts_left > 0 %}
            <button type="submit">{{ _('Start quiz') }}</button>
        {% else %}
            <p>{{ _('No attempts remaining.') }}</p>
        {% endif %}
    </form>
    {% if attempts %}
        <h3>{{ _('Your attempts') }}</h3>
        <table class="table">
            <tr><th>{{ _('Started') }}</th><th>{{ _('Score') }}</th><th></th></tr>
            {% for attempt in attempts %}
            <tr>
                <td>{{ attempt.started_at|date('DATETIME_FORMAT') }}</td>
                <td>{% if attempt.is_submitted %}{{ attempt.score }} / {{ total_points }}{% else %}{{ _('in progress') }}{% endif %}</td>
                <td><a href="{{ url('quiz_result', quiz.code, attempt.id) if attempt.is_submitted else url('quiz_take', quiz.code, attempt.id) }}">{{ _('view') }}</a></td>
            </tr>
            {% endfor %}
        </table>
    {% endif %}
{% else %}
    <p>{{ _('Log in to take this quiz.') }}</p>
{% endif %}
{% endblock %}
```

Note: `quiz_start`/`quiz_take`/`quiz_result`/`quiz_ranking`/`quiz_edit` URLs arrive in Tasks 8-11 and 15-16. For THIS task's tests to pass, temporarily guard those links — simplest is to add all URL names now as stubs in `quiz/urls.py` pointing at `QuizDetail.as_view()` and replace them in their tasks. Add inside the `<str:quiz>` include:

```python
        path('/start', student.QuizDetail.as_view(), name='quiz_start'),
        path('/ranking', student.QuizDetail.as_view(), name='quiz_ranking'),
        path('/attempt/<int:attempt>', include([
            path('', student.QuizDetail.as_view(), name='quiz_take'),
            path('/result', student.QuizDetail.as_view(), name='quiz_result'),
        ])),
```

and `quiz_edit` stub under the list block: `path('/edit/<str:quiz>', student.QuizDetail.as_view(), name='quiz_edit'),` — **replaced in Task 16** (final form is `/quiz/<code>/edit`; the stub only exists so `url()` resolves).

- [ ] **Step 5: Run tests to verify they pass**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_views -v 2
```
Expected: 4 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add quiz/ templates/quiz/ dmoj/urls.py
git commit -m "feat(quiz): quiz list and detail pages"
```

---

### Task 8: Start and take pages

**Files:**
- Modify: `quiz/views/student.py`, `quiz/urls.py`
- Create: `templates/quiz/take.html`, `resources/quiz.js`
- Test: `quiz/tests/test_views.py`

- [ ] **Step 1: Write the failing tests** (append to `quiz/tests/test_views.py`)

```python
from quiz.models import QuizAttempt  # add to imports
from quiz.tests.util import create_question  # add to imports


class QuizTakeFlowTestCase(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.student = create_user(username='flow_student')
        cls.question = create_question(title='flow_q')
        cls.quiz = create_quiz(code='flowquiz', max_attempts=2,
                               questions=((cls.question, 1.0),))

    def setUp(self):
        self.client.force_login(self.student)

    def start(self):
        return self.client.post(reverse('quiz_start', args=('flowquiz',)))

    def test_start_creates_attempt_and_redirects_to_take(self):
        response = self.start()
        attempt = QuizAttempt.objects.get(quiz=self.quiz, user=self.student.profile)
        self.assertRedirects(response, reverse('quiz_take', args=('flowquiz', attempt.id)))

    def test_start_resumes_existing_attempt(self):
        self.start()
        self.start()
        self.assertEqual(QuizAttempt.objects.filter(user=self.student.profile).count(), 1)

    def test_start_blocked_when_attempts_exhausted(self):
        for _i in range(2):
            self.start()
            attempt = QuizAttempt.objects.filter(is_submitted=False).get()
            attempt.finalize()
        self.start()
        self.assertEqual(QuizAttempt.objects.filter(quiz=self.quiz).count(), 2)

    def test_start_requires_login(self):
        self.client.logout()
        response = self.start()
        self.assertEqual(response.status_code, 302)
        self.assertIn('login', response['Location'])

    def test_take_page_renders_questions(self):
        self.start()
        attempt = QuizAttempt.objects.get(user=self.student.profile)
        response = self.client.get(reverse('quiz_take', args=('flowquiz', attempt.id)))
        self.assertContains(response, 'Body of flow_q')

    def test_take_page_of_other_user_404s(self):
        self.start()
        attempt = QuizAttempt.objects.get(user=self.student.profile)
        other = create_user(username='flow_other')
        self.client.force_login(other)
        response = self.client.get(reverse('quiz_take', args=('flowquiz', attempt.id)))
        self.assertEqual(response.status_code, 404)

    def test_submitted_attempt_take_redirects_to_result(self):
        self.start()
        attempt = QuizAttempt.objects.get(user=self.student.profile)
        attempt.finalize()
        response = self.client.get(reverse('quiz_take', args=('flowquiz', attempt.id)))
        self.assertRedirects(response, reverse('quiz_result', args=('flowquiz', attempt.id)),
                             target_status_code=200)
```

(The last assertion needs `quiz_result` to return 200 for a submitted attempt — Task 10 implements it; until then mark it `target_status_code=302` is NOT acceptable. Order note: implement Tasks 8-10 before expecting the full file to pass; within this task run only the tests listed below.)

- [ ] **Step 2: Run the new tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_views.QuizTakeFlowTestCase -v 2
```
Expected: failures — start URL currently renders QuizDetail (no POST/redirect behaviour).

- [ ] **Step 3: Implement** — append to `quiz/views/student.py`:

```python
from django.contrib import messages          # add to imports
from django.shortcuts import redirect        # add to imports
from django.utils.translation import gettext as _   # add to imports
from django.views.generic import View        # merge into the generic import
from quiz.models import QuizQuestion         # merge into the models import


class QuizStart(LoginRequiredMixin, QuizMixin, View):
    def post(self, request, *args, **kwargs):
        in_progress = self.quiz.attempts.filter(
            user=request.profile, is_submitted=False).first()
        if in_progress is not None:
            if in_progress.has_expired():
                in_progress.finalize()
            else:
                return redirect('quiz_take', quiz=self.quiz.code, attempt=in_progress.id)
        if self.quiz.max_attempts is not None:
            used = self.quiz.attempts.filter(
                user=request.profile, is_submitted=True).count()
            if used >= self.quiz.max_attempts:
                messages.error(request, _('You have no attempts remaining for this quiz.'))
                return redirect('quiz_detail', quiz=self.quiz.code)
        attempt = QuizAttempt.start(self.quiz, request.profile)
        return redirect('quiz_take', quiz=self.quiz.code, attempt=attempt.id)


class AttemptMixin(QuizMixin):
    """Resolves self.attempt; owner-only unless the user can edit the quiz.
    Lazily finalizes expired attempts before any handler runs."""
    allow_editor = False

    def dispatch(self, request, *args, **kwargs):
        self.quiz = get_object_or_404(Quiz, code=kwargs['quiz'])
        if not self.quiz.is_accessible_by(request.user):
            raise Http404()
        self.attempt = get_object_or_404(QuizAttempt, id=kwargs['attempt'], quiz=self.quiz)
        is_owner = request.user.is_authenticated and \
            self.attempt.user_id == request.profile.id
        if not is_owner and not (self.allow_editor and self.quiz.is_editable_by(request.user)):
            raise Http404()
        if not self.attempt.is_submitted and self.attempt.has_expired():
            self.attempt.finalize()
        return super(QuizMixin, self).dispatch(request, *args, **kwargs)


class QuizTake(LoginRequiredMixin, AttemptMixin, TemplateView):
    template_name = 'quiz/take.html'

    def get(self, request, *args, **kwargs):
        if self.attempt.is_submitted:
            return redirect('quiz_result', quiz=self.quiz.code, attempt=self.attempt.id)
        return super().get(request, *args, **kwargs)

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        questions = {q.id: q for q in QuizQuestion.objects.filter(
            id__in=self.attempt.question_order)}
        answers = {a.question_id: a.answer for a in self.attempt.answers.all()}
        items = []
        for number, question_id in enumerate(self.attempt.question_order, start=1):
            question = questions.get(question_id)
            if question is None:
                continue
            choice_indices = self.attempt.choice_orders.get(str(question_id)) or \
                list(range(len(question.choices or [])))
            items.append({
                'number': number,
                'question': question,
                'choice_indices': choice_indices,
                'answer': answers.get(question_id),
            })
        context['items'] = items
        context['attempt'] = self.attempt
        context['time_remaining'] = self.attempt.time_remaining_seconds()
        return context
```

In `quiz/urls.py`, replace the `quiz_start` and `quiz_take` stubs:
```python
        path('/start', student.QuizStart.as_view(), name='quiz_start'),
```
and inside the attempt include:
```python
            path('', student.QuizTake.as_view(), name='quiz_take'),
```

- [ ] **Step 4: Create the take template and JS**

`templates/quiz/take.html`:
```jinja2
{% extends "common-content.html" %}

{% block media %}
<style>
    .quiz-layout { display: flex; gap: 1.5em; align-items: flex-start; }
    .quiz-questions { flex: 1; }
    .quiz-sidebar { position: sticky; top: 60px; width: 180px; }
    .quiz-question { border: 1px solid #ddd; border-radius: 4px; padding: 1em; margin-bottom: 1em; }
    .quiz-map { display: flex; flex-wrap: wrap; gap: 4px; margin-top: 0.5em; }
    .quiz-map a { display: block; width: 28px; height: 28px; line-height: 28px;
                  text-align: center; border: 1px solid #ccc; border-radius: 3px;
                  text-decoration: none; }
    .quiz-map a.answered { background: #cce5cc; }
    #quiz-timer { font-size: 1.4em; font-weight: bold; }
    #quiz-timer.warning { color: #c90; }
    #quiz-timer.danger { color: #c00; }
    #quiz-save-status { font-size: 0.85em; color: #666; }
</style>
{% endblock %}

{% block body %}
<div class="quiz-layout">
    <div class="quiz-questions">
        {% for item in items %}
        <div class="quiz-question" id="question-{{ item.question.id }}"
             data-question="{{ item.question.id }}" data-type="{{ item.question.type }}">
            <h4>{{ _('Question') }} {{ item.number }}</h4>
            <div>{{ item.question.content|markdown('default') }}</div>
            {% if item.question.type == 'MC' %}
                {% for idx in item.choice_indices %}
                <label style="display:block">
                    <input type="radio" name="q{{ item.question.id }}" value="{{ idx }}"
                           {% if item.answer == idx %}checked{% endif %}>
                    {{ item.question.choices[idx] }}
                </label>
                {% endfor %}
            {% elif item.question.type == 'MA' %}
                {% for idx in item.choice_indices %}
                <label style="display:block">
                    <input type="checkbox" name="q{{ item.question.id }}" value="{{ idx }}"
                           {% if item.answer and idx in item.answer %}checked{% endif %}>
                    {{ item.question.choices[idx] }}
                </label>
                {% endfor %}
            {% elif item.question.type == 'TF' %}
                <label style="display:block"><input type="radio" name="q{{ item.question.id }}" value="0"
                    {% if item.answer == 0 %}checked{% endif %}> {{ _('True') }}</label>
                <label style="display:block"><input type="radio" name="q{{ item.question.id }}" value="1"
                    {% if item.answer == 1 %}checked{% endif %}> {{ _('False') }}</label>
            {% elif item.question.type == 'SA' %}
                <input type="text" name="q{{ item.question.id }}" size="50"
                       value="{{ item.answer or '' }}">
            {% endif %}
        </div>
        {% endfor %}
    </div>
    <div class="quiz-sidebar">
        {% if time_remaining is not none %}<div id="quiz-timer"></div>{% endif %}
        <div id="quiz-save-status"></div>
        <div class="quiz-map">
            {% for item in items %}
            <a href="#question-{{ item.question.id }}" id="map-{{ item.question.id }}"
               class="{% if item.answer is not none %}answered{% endif %}">{{ item.number }}</a>
            {% endfor %}
        </div>
        <form id="quiz-submit-form" action="{{ url('quiz_submit', quiz.code, attempt.id) }}" method="post"
              style="margin-top:1em">
            {% csrf_token %}
            <button type="submit">{{ _('Submit quiz') }}</button>
        </form>
    </div>
</div>
<script>
window.quizConfig = {
    saveUrl: '{{ url('quiz_save', quiz.code, attempt.id) }}',
    csrfToken: '{{ csrf_token }}',
    timeRemaining: {{ time_remaining if time_remaining is not none else 'null' }},
};
</script>
<script src="{{ static('quiz.js') }}"></script>
{% endblock %}
```

(`quiz_submit` and `quiz_save` URL stubs: add them now in `quiz/urls.py` inside the attempt include, pointing at `student.QuizTake.as_view()`; Tasks 9-10 replace them.)

`resources/quiz.js`:
```javascript
(function () {
    'use strict';
    var config = window.quizConfig;
    if (!config) return;

    var status = document.getElementById('quiz-save-status');
    var saveTimers = {};

    function collectAnswer(block) {
        var qid = block.dataset.question, type = block.dataset.type;
        if (type === 'MC' || type === 'TF') {
            var checked = block.querySelector('input:checked');
            return checked ? parseInt(checked.value, 10) : null;
        }
        if (type === 'MA') {
            var values = [];
            block.querySelectorAll('input:checked').forEach(function (el) {
                values.push(parseInt(el.value, 10));
            });
            return values.length ? values : null;
        }
        var text = block.querySelector('input[type=text]');
        return text && text.value.trim() !== '' ? text.value : null;
    }

    function save(block) {
        var qid = block.dataset.question;
        var answer = collectAnswer(block);
        fetch(config.saveUrl, {
            method: 'POST',
            headers: {'Content-Type': 'application/json', 'X-CSRFToken': config.csrfToken},
            body: JSON.stringify({question: parseInt(qid, 10), answer: answer}),
        }).then(function (response) {
            if (!response.ok) throw new Error('save failed');
            status.textContent = '';
            var map = document.getElementById('map-' + qid);
            if (map) map.classList.toggle('answered', answer !== null);
        }).catch(function () {
            status.textContent = 'Save failed — retrying…';
            clearTimeout(saveTimers[qid]);
            saveTimers[qid] = setTimeout(function () { save(block); }, 3000);
        });
    }

    document.querySelectorAll('.quiz-question').forEach(function (block) {
        block.addEventListener('change', function () { save(block); });
        var text = block.querySelector('input[type=text]');
        if (text) {
            text.addEventListener('input', function () {
                clearTimeout(saveTimers['t' + block.dataset.question]);
                saveTimers['t' + block.dataset.question] =
                    setTimeout(function () { save(block); }, 800);
            });
        }
    });

    if (config.timeRemaining !== null) {
        var timerEl = document.getElementById('quiz-timer');
        var deadline = Date.now() + config.timeRemaining * 1000;
        var tick = setInterval(function () {
            var left = Math.max(0, Math.round((deadline - Date.now()) / 1000));
            var minutes = Math.floor(left / 60), seconds = left % 60;
            timerEl.textContent = minutes + ':' + (seconds < 10 ? '0' : '') + seconds;
            timerEl.className = left < 60 ? 'danger' : (left < 300 ? 'warning' : '');
            timerEl.id = 'quiz-timer';
            if (left <= 0) {
                clearInterval(tick);
                document.getElementById('quiz-submit-form').submit();
            }
        }, 500);
    }
}());
```

- [ ] **Step 5: Run the tests** (all but the result-redirect one pass; that one passes after Task 10)

```bash
docker compose exec site python3 manage.py test quiz.tests.test_views.QuizTakeFlowTestCase -v 2
```
Expected: all PASS except `test_submitted_attempt_take_redirects_to_result` (404/render mismatch until Task 10 gives `quiz_result` a real view). If anything ELSE fails, fix before committing.

- [ ] **Step 6: Commit**

```bash
git add quiz/ templates/quiz/take.html resources/quiz.js
git commit -m "feat(quiz): start/resume and take pages with autosave JS"
```

---

### Task 9: AJAX answer save with lazy timer enforcement

**Files:**
- Modify: `quiz/views/student.py`, `quiz/urls.py`
- Test: `quiz/tests/test_views.py`

- [ ] **Step 1: Write the failing tests** (append to `quiz/tests/test_views.py`)

```python
import json  # add to imports at top
from datetime import timedelta  # add to imports at top

from django.utils import timezone  # add to imports at top

from quiz.models import QuizAnswer  # merge into quiz.models import


class QuizSaveAnswerTestCase(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.student = create_user(username='save_student')
        cls.question = create_question(title='save_q')
        cls.quiz = create_quiz(code='savequiz', time_limit=10,
                               questions=((cls.question, 2.0),))

    def setUp(self):
        self.client.force_login(self.student)
        self.client.post(reverse('quiz_start', args=('savequiz',)))
        self.attempt = QuizAttempt.objects.get(user=self.student.profile, quiz=self.quiz)
        self.url = reverse('quiz_save', args=('savequiz', self.attempt.id))

    def save(self, payload):
        return self.client.post(self.url, json.dumps(payload),
                                content_type='application/json')

    def test_save_grades_and_upserts(self):
        response = self.save({'question': self.question.id, 'answer': 0})
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json(), {'saved': True})
        answer = QuizAnswer.objects.get(attempt=self.attempt)
        self.assertEqual(answer.points, 2.0)
        # Response must NOT leak correctness (spec refinement 1).
        self.assertNotIn('points', response.json())
        self.save({'question': self.question.id, 'answer': 1})
        self.assertEqual(QuizAnswer.objects.filter(attempt=self.attempt).count(), 1)

    def test_save_rejects_invalid_payloads(self):
        for payload in ({'answer': 0}, {'question': 999999, 'answer': 0},
                        {'question': self.question.id, 'answer': 99}):
            self.assertEqual(self.save(payload).status_code, 400, payload)

    def test_save_after_expiry_rejected_and_finalized(self):
        QuizAttempt.objects.filter(id=self.attempt.id).update(
            started_at=timezone.now() - timedelta(minutes=11))
        response = self.save({'question': self.question.id, 'answer': 0})
        self.assertEqual(response.status_code, 400)
        self.assertEqual(response.json()['error'], 'attempt_closed')
        self.attempt.refresh_from_db()
        self.assertTrue(self.attempt.is_submitted)

    def test_save_within_grace_period_accepted(self):
        QuizAttempt.objects.filter(id=self.attempt.id).update(
            started_at=timezone.now() - timedelta(minutes=10, seconds=10))
        response = self.save({'question': self.question.id, 'answer': 0})
        self.assertEqual(response.status_code, 200)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_views.QuizSaveAnswerTestCase -v 2
```
Expected: failures (stub view renders a page, not JSON).

- [ ] **Step 3: Implement** — append to `quiz/views/student.py`:

```python
import json  # add to module imports

from django.http import HttpResponseBadRequest, JsonResponse  # add to imports


def _normalize_answer(question, answer):
    """Validate the client payload for the question type. Raises ValueError."""
    if answer is None:
        return None
    if question.type in ('MC', 'TF'):
        index = int(answer)
        limit = 2 if question.type == 'TF' else len(question.choices or [])
        if not 0 <= index < limit:
            raise ValueError('choice index out of range')
        return index
    if question.type == 'MA':
        if not isinstance(answer, list):
            raise ValueError('MA answer must be a list')
        indices = sorted({int(i) for i in answer})
        if indices and not (0 <= indices[0] and indices[-1] < len(question.choices or [])):
            raise ValueError('choice index out of range')
        return indices or None
    if not isinstance(answer, str):
        raise ValueError('SA answer must be a string')
    return answer[:10000]


class QuizSaveAnswer(LoginRequiredMixin, AttemptMixin, View):
    def post(self, request, *args, **kwargs):
        if self.attempt.is_submitted:
            return JsonResponse({'error': 'attempt_closed'}, status=400)
        try:
            body = json.loads(request.body)
            question_id = int(body['question'])
            raw_answer = body.get('answer')
        except (KeyError, TypeError, ValueError, json.JSONDecodeError):
            return HttpResponseBadRequest()
        if question_id not in self.attempt.question_order:
            return HttpResponseBadRequest()
        question = QuizQuestion.objects.get(id=question_id)
        try:
            answer = _normalize_answer(question, raw_answer)
        except (TypeError, ValueError):
            return HttpResponseBadRequest()
        self.attempt.save_answer(question, answer)
        return JsonResponse({'saved': True})
```

In `quiz/urls.py`, replace the `quiz_save` stub:
```python
            path('/save', student.QuizSaveAnswer.as_view(), name='quiz_save'),
```

- [ ] **Step 4: Run tests to verify they pass**

Expected: 4 tests PASS. Note the expiry test works because `AttemptMixin.dispatch` finalizes expired attempts BEFORE the handler runs, so the handler sees `is_submitted=True`.

- [ ] **Step 5: Commit**

```bash
git add quiz/views/student.py quiz/urls.py quiz/tests/test_views.py
git commit -m "feat(quiz): AJAX answer save with server-side deadline enforcement"
```

---

### Task 10: Submit and result pages

**Files:**
- Modify: `quiz/views/student.py`, `quiz/urls.py`
- Create: `templates/quiz/result.html`
- Test: `quiz/tests/test_views.py`

- [ ] **Step 1: Write the failing tests** (append to `quiz/tests/test_views.py`)

```python
class QuizSubmitResultTestCase(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.student = create_user(username='submit_student')
        cls.question = create_question(title='submit_q', explanation='Because reasons.')
        cls.quiz = create_quiz(code='submitquiz', questions=((cls.question, 1.0),))
        cls.hidden = create_quiz(code='hiddenquiz', show_correctness=False,
                                 show_answers=False, questions=((cls.question, 1.0),))

    def run_quiz(self, code, answer):
        self.client.force_login(self.student)
        self.client.post(reverse('quiz_start', args=(code,)))
        attempt = QuizAttempt.objects.get(user=self.student.profile, quiz__code=code)
        self.client.post(reverse('quiz_save', args=(code, attempt.id)),
                         json.dumps({'question': self.question.id, 'answer': answer}),
                         content_type='application/json')
        response = self.client.post(reverse('quiz_submit', args=(code, attempt.id)))
        attempt.refresh_from_db()
        return attempt, response

    def test_submit_finalizes_and_redirects(self):
        attempt, response = self.run_quiz('submitquiz', 0)
        self.assertTrue(attempt.is_submitted)
        self.assertEqual(attempt.score, 1.0)
        self.assertRedirects(response, reverse('quiz_result', args=('submitquiz', attempt.id)))

    def test_result_shows_feedback_when_enabled(self):
        attempt, _response = self.run_quiz('submitquiz', 1)
        response = self.client.get(reverse('quiz_result', args=('submitquiz', attempt.id)))
        self.assertContains(response, 'Because reasons.')   # explanation (show_answers)
        self.assertContains(response, 'incorrect')          # correctness marker css class

    def test_result_hides_feedback_when_disabled(self):
        attempt, _response = self.run_quiz('hiddenquiz', 1)
        response = self.client.get(reverse('quiz_result', args=('hiddenquiz', attempt.id)))
        self.assertNotContains(response, 'Because reasons.')
        self.assertNotContains(response, 'incorrect')
        self.assertContains(response, '0.0')                # score itself always shown

    def test_result_of_in_progress_attempt_redirects_to_take(self):
        self.client.force_login(self.student)
        self.client.post(reverse('quiz_start', args=('submitquiz',)))
        attempt = QuizAttempt.objects.get(user=self.student.profile, quiz=self.quiz)
        response = self.client.get(reverse('quiz_result', args=('submitquiz', attempt.id)))
        self.assertRedirects(response, reverse('quiz_take', args=('submitquiz', attempt.id)))

    def test_editor_can_view_student_result(self):
        attempt, _response = self.run_quiz('submitquiz', 0)
        editor = create_user(username='submit_editor', user_permissions=('edit_all_quiz',))
        self.client.force_login(editor)
        response = self.client.get(reverse('quiz_result', args=('submitquiz', attempt.id)))
        self.assertEqual(response.status_code, 200)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_views.QuizSubmitResultTestCase -v 2
```
Expected: failures (stub views).

- [ ] **Step 3: Implement** — append to `quiz/views/student.py`:

```python
class QuizSubmit(LoginRequiredMixin, AttemptMixin, View):
    def post(self, request, *args, **kwargs):
        if not self.attempt.is_submitted:
            self.attempt.finalize()
        return redirect('quiz_result', quiz=self.quiz.code, attempt=self.attempt.id)


class QuizResult(LoginRequiredMixin, AttemptMixin, TemplateView):
    template_name = 'quiz/result.html'
    allow_editor = True

    def get(self, request, *args, **kwargs):
        if not self.attempt.is_submitted:
            return redirect('quiz_take', quiz=self.quiz.code, attempt=self.attempt.id)
        return super().get(request, *args, **kwargs)

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        can_edit = self.quiz.is_editable_by(self.request.user)
        show_correctness = self.quiz.show_correctness or can_edit
        show_answers = self.quiz.show_answers or can_edit
        questions = {q.id: q for q in QuizQuestion.objects.filter(
            id__in=self.attempt.question_order)}
        answers = {a.question_id: a for a in self.attempt.answers.all()}
        links = {link.question_id: link for link in self.quiz.question_links.all()}
        rows = []
        for number, question_id in enumerate(self.attempt.question_order, start=1):
            question = questions.get(question_id)
            if question is None:
                continue
            rows.append({
                'number': number,
                'question': question,
                'answer': answers.get(question_id),
                'max_points': links[question_id].points if question_id in links else 0,
            })
        context.update({
            'attempt': self.attempt,
            'rows': rows,
            'total_points': self.quiz.total_points,
            'show_correctness': show_correctness,
            'show_answers': show_answers,
        })
        return context
```

In `quiz/urls.py`, replace the `quiz_submit` and `quiz_result` stubs:
```python
            path('/submit', student.QuizSubmit.as_view(), name='quiz_submit'),
            path('/result', student.QuizResult.as_view(), name='quiz_result'),
```

- [ ] **Step 4: Create the result template**

`templates/quiz/result.html`:
```jinja2
{% extends "common-content.html" %}

{% block media %}
<style>
    .quiz-answer-row { border: 1px solid #ddd; border-radius: 4px; padding: 1em; margin-bottom: 1em; }
    .quiz-answer-row.correct { border-left: 5px solid #4a4; }
    .quiz-answer-row.incorrect { border-left: 5px solid #c44; }
    .quiz-explanation { background: #f6f6f6; padding: 0.6em; margin-top: 0.6em; border-radius: 4px; }
</style>
{% endblock %}

{% block body %}
<h2>{{ quiz.name }} — {{ _('Result') }}</h2>
<p style="font-size: 1.3em">{{ _('Score') }}: <b>{{ attempt.score }} / {{ total_points }}</b></p>
{% for row in rows %}
<div class="quiz-answer-row
        {%- if show_correctness and row.answer %} {{ 'correct' if row.answer.is_correct else 'incorrect' }}{% endif %}">
    <h4>{{ _('Question') }} {{ row.number }}
        {% if show_correctness %}({{ row.answer.points if row.answer else 0 }} / {{ row.max_points }}){% endif %}</h4>
    <div>{{ row.question.content|markdown('default') }}</div>
    <p>{{ _('Your answer') }}:
        {% if row.answer is none or row.answer.answer is none %}<i>{{ _('(no answer)') }}</i>
        {% elif row.question.type in ('MC',) %}{{ row.question.choices[row.answer.answer] }}
        {% elif row.question.type == 'TF' %}{{ _('True') if row.answer.answer == 0 else _('False') }}
        {% elif row.question.type == 'MA' %}
            {% for idx in row.answer.answer %}{{ row.question.choices[idx] }}{% if not loop.last %}; {% endif %}{% endfor %}
        {% else %}{{ row.answer.answer }}{% endif %}
    </p>
    {% if show_answers %}
        <p>{{ _('Correct answer') }}:
            {% if row.question.type == 'MC' %}{{ row.question.choices[row.question.correct_answers] }}
            {% elif row.question.type == 'TF' %}{{ _('True') if row.question.correct_answers == 0 else _('False') }}
            {% elif row.question.type == 'MA' %}
                {% for idx in row.question.correct_answers %}{{ row.question.choices[idx] }}{% if not loop.last %}; {% endif %}{% endfor %}
            {% else %}
                {% for acc in row.question.correct_answers %}{{ acc.text }}{% if not loop.last %} | {% endif %}{% endfor %}
            {% endif %}
        </p>
        {% if row.question.explanation %}
            <div class="quiz-explanation">{{ row.question.explanation|markdown('default') }}</div>
        {% endif %}
    {% endif %}
</div>
{% endfor %}
<p><a href="{{ url('quiz_detail', quiz.code) }}">{{ _('Back to quiz') }}</a> ·
   <a href="{{ url('quiz_ranking', quiz.code) }}">{{ _('Ranking') }}</a></p>
{% endblock %}
```

- [ ] **Step 5: Run the whole view test file**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_views -v 2
```
Expected: ALL view tests pass now, including Task 8's deferred `test_submitted_attempt_take_redirects_to_result`.

- [ ] **Step 6: Commit**

```bash
git add quiz/ templates/quiz/result.html
git commit -m "feat(quiz): submit flow and result page with feedback gating"
```

---

### Task 11: Leaderboard page

**Files:**
- Modify: `quiz/views/student.py`, `quiz/urls.py`
- Create: `templates/quiz/ranking.html`
- Test: `quiz/tests/test_views.py`

- [ ] **Step 1: Write the failing tests** (append to `quiz/tests/test_views.py`)

```python
class QuizRankingViewTestCase(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.question = create_question(title='rank_q')
        cls.quiz = create_quiz(code='rankquiz', questions=((cls.question, 1.0),))
        cls.winner = create_user(username='rank_winner')
        now = timezone.now()
        attempt = QuizAttempt.start(cls.quiz, cls.winner.profile)
        attempt.score, attempt.is_submitted = 1.0, True
        attempt.started_at, attempt.submitted_at = now, now + timedelta(minutes=2)
        attempt.save()

    def test_ranking_lists_best_attempts(self):
        response = self.client.get(reverse('quiz_ranking', args=('rankquiz',)))
        self.assertContains(response, 'rank_winner')
        self.assertContains(response, '1.0')

    def test_ranking_of_private_quiz_404s(self):
        create_quiz(code='rankpriv', is_public=False)
        response = self.client.get(reverse('quiz_ranking', args=('rankpriv',)))
        self.assertEqual(response.status_code, 404)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_views.QuizRankingViewTestCase -v 2
```
Expected: first test fails (stub renders detail page without ranking).

- [ ] **Step 3: Implement** — append to `quiz/views/student.py`:

```python
class QuizRanking(QuizMixin, TemplateView):
    template_name = 'quiz/ranking.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['ranking'] = self.quiz.get_ranking()
        context['total_points'] = self.quiz.total_points
        context['my_profile_id'] = self.request.profile.id \
            if self.request.user.is_authenticated else None
        return context
```

Replace the `quiz_ranking` stub in `quiz/urls.py`:
```python
        path('/ranking', student.QuizRanking.as_view(), name='quiz_ranking'),
```

`templates/quiz/ranking.html`:
```jinja2
{% extends "common-content.html" %}

{% block media %}
<style>
    .quiz-ranking tr.me { background: #fdf2cc; }
</style>
{% endblock %}

{% block body %}
<h2>{{ quiz.name }} — {{ _('Ranking') }}</h2>
<table class="table quiz-ranking">
    <thead>
        <tr><th>#</th><th>{{ _('User') }}</th><th>{{ _('Score') }}</th><th>{{ _('Time') }}</th></tr>
    </thead>
    <tbody>
        {% for attempt in ranking %}
        <tr {% if attempt.user_id == my_profile_id %}class="me"{% endif %}>
            <td>{{ loop.index }}</td>
            <td>{{ link_user(attempt.user) }}</td>
            <td>{{ attempt.score }} / {{ total_points }}</td>
            <td>{{ attempt.duration }}</td>
        </tr>
        {% else %}
        <tr><td colspan="4">{{ _('No submitted attempts yet.') }}</td></tr>
        {% endfor %}
    </tbody>
</table>
<p><a href="{{ url('quiz_detail', quiz.code) }}">{{ _('Back to quiz') }}</a></p>
{% endblock %}
```

(`link_user` is a site-wide jinja global used across DMOJ templates — verify with `grep -r "link_user" templates/user/ | head -1`; fall back to `{{ attempt.user.user.username }}` if absent.)

- [ ] **Step 4: Run tests to verify they pass**

```bash
docker compose exec site python3 manage.py test quiz -v 2
```
Expected: entire quiz suite PASSES.

- [ ] **Step 5: Commit**

```bash
git add quiz/ templates/quiz/ranking.html
git commit -m "feat(quiz): per-quiz leaderboard"
```

---

### Task 12: Import core — spec parsing, validation, JSON format

**Files:**
- Create: `quiz/importers/__init__.py`, `quiz/importers/base.py`, `quiz/importers/json_fmt.py`
- Test: `quiz/tests/test_importers.py`

The `correct` spec mini-syntax (shared by the XLSX column and the question editor form):
MC `2` (1-based choice number) · MA `1,3` · TF `true`/`false` · SA `42 | re:4[0-9] | cs:Hanoi` (pipe-separated; `re:` regex, `cs:` case-sensitive, combinable).

- [ ] **Step 1: Write the failing tests**

`quiz/tests/test_importers.py`:
```python
import json

from django.test import SimpleTestCase

from quiz.importers import base, json_fmt


class ParseCorrectSpecTestCase(SimpleTestCase):
    def test_mc_one_based(self):
        self.assertEqual(base.parse_correct_spec('MC', '2', 4), (1, []))

    def test_mc_out_of_range(self):
        value, errors = base.parse_correct_spec('MC', '5', 4)
        self.assertTrue(errors)

    def test_mc_not_a_number(self):
        value, errors = base.parse_correct_spec('MC', 'abc', 4)
        self.assertTrue(errors)

    def test_ma_comma_list(self):
        self.assertEqual(base.parse_correct_spec('MA', '1, 3', 4), ([0, 2], []))

    def test_tf_values(self):
        self.assertEqual(base.parse_correct_spec('TF', 'true', 0), (0, []))
        self.assertEqual(base.parse_correct_spec('TF', 'False', 0), (1, []))
        self.assertTrue(base.parse_correct_spec('TF', 'yes', 0)[1])

    def test_sa_pipe_and_prefixes(self):
        value, errors = base.parse_correct_spec('SA', '42 | re:4[0-9] | cs:Hanoi | re:cs:A.B', 0)
        self.assertEqual(errors, [])
        self.assertEqual(value, [
            {'text': '42', 'case_sensitive': False, 'is_regex': False},
            {'text': '4[0-9]', 'case_sensitive': False, 'is_regex': True},
            {'text': 'Hanoi', 'case_sensitive': True, 'is_regex': False},
            {'text': 'A.B', 'case_sensitive': True, 'is_regex': True},
        ])

    def test_sa_invalid_regex_rejected(self):
        value, errors = base.parse_correct_spec('SA', 're:(', 0)
        self.assertTrue(errors)

    def test_sa_empty(self):
        self.assertTrue(base.parse_correct_spec('SA', '', 0)[1])


class ValidateQuestionTestCase(SimpleTestCase):
    @staticmethod
    def make(**kwargs):
        question = base.ParsedQuestion(row=1, type='MC', title='t', content='c',
                                       choices=['a', 'b'], correct=0)
        for key, value in kwargs.items():
            setattr(question, key, value)
        return question

    def test_valid_question_has_no_errors(self):
        question = self.make()
        base.validate_question(question)
        self.assertEqual(question.errors, [])

    def test_each_violation_reported(self):
        cases = [
            {'type': 'ES'},                       # unknown type
            {'title': ''},                        # missing title
            {'content': ''},                      # missing content
            {'choices': ['only']},                # MC needs >= 2 choices
            {'correct': None},                    # missing answer key
            {'level': 'extreme'},                 # bad level
            {'points': -1},                       # negative points
            {'ma_strategy': 'bogus'},             # bad strategy
        ]
        for overrides in cases:
            question = self.make(**overrides)
            base.validate_question(question)
            self.assertTrue(question.errors, overrides)


class JsonFormatTestCase(SimpleTestCase):
    def parse(self, payload):
        return json_fmt.parse(json.dumps(payload).encode())

    def test_round_trip_all_types(self):
        questions = self.parse([
            {'type': 'MC', 'title': 'm', 'content': 'c', 'choices': ['a', 'b'], 'correct': 1},
            {'type': 'MA', 'title': 'n', 'content': 'c', 'choices': ['a', 'b', 'c'],
             'correct': [0, 2], 'ma_strategy': 'partial_credit'},
            {'type': 'TF', 'title': 'o', 'content': 'c', 'correct': True},
            {'type': 'SA', 'title': 'p', 'content': 'c',
             'correct': ['42', {'text': 'x', 'case_sensitive': True, 'is_regex': False}],
             'category': 'python-basics', 'level': 'hard', 'points': 2.5},
        ])
        self.assertEqual([q.errors for q in questions], [[], [], [], []])
        self.assertEqual(questions[0].correct, 1)
        self.assertEqual(questions[1].correct, [0, 2])
        self.assertEqual(questions[2].correct, 0)   # True -> index 0
        self.assertEqual(questions[3].correct[0],
                         {'text': '42', 'case_sensitive': False, 'is_regex': False})
        self.assertEqual(questions[3].level, 'hard')
        self.assertEqual(questions[3].points, 2.5)

    def test_invalid_json_is_one_error(self):
        questions = json_fmt.parse(b'not json')
        self.assertEqual(len(questions), 1)
        self.assertTrue(questions[0].errors)

    def test_non_list_payload_is_error(self):
        questions = self.parse({'type': 'MC'})
        self.assertTrue(questions[0].errors)

    def test_row_level_errors_collected(self):
        questions = self.parse([{'type': 'MC', 'title': '', 'content': '', 'choices': [],
                                 'correct': None}])
        self.assertTrue(questions[0].errors)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_importers -v 2
```
Expected: ImportError on `quiz.importers`.

- [ ] **Step 3: Implement**

`quiz/importers/__init__.py`: empty file.

`quiz/importers/base.py`:
```python
"""Shared import machinery: the normalized question record, the 'correct'
spec mini-syntax, and validation. Pure functions - DB only at commit time."""
import dataclasses
import re

from quiz import grading

LEVELS = ('easy', 'medium', 'hard')
TYPES = (grading.MC, grading.MA, grading.TF, grading.SA)
TF_TRUE = ('true', '1', 'dung', 'đúng')
TF_FALSE = ('false', '0', 'sai')


@dataclasses.dataclass
class ParsedQuestion:
    row: int
    type: str = ''
    title: str = ''
    content: str = ''
    choices: list = dataclasses.field(default_factory=list)
    correct: object = None
    points: float = 1.0
    category: str = ''
    level: str = 'easy'
    explanation: str = ''
    shuffle: bool = False
    ma_strategy: str = grading.MA_ALL_OR_NOTHING
    errors: list = dataclasses.field(default_factory=list)

    def as_dict(self):
        return dataclasses.asdict(self)

    @classmethod
    def from_dict(cls, data):
        return cls(**data)


def parse_correct_spec(qtype, spec, num_choices):
    """Parse the human 'correct' syntax. Returns (normalized_value, errors)."""
    spec = ('' if spec is None else str(spec)).strip()
    if qtype == grading.MC:
        try:
            index = int(spec) - 1
        except ValueError:
            return None, ['MC correct answer must be a choice number (1-based)']
        if not 0 <= index < num_choices:
            return None, ['MC correct answer %s out of range 1-%d' % (spec, num_choices)]
        return index, []
    if qtype == grading.TF:
        low = spec.lower()
        if low in TF_TRUE:
            return 0, []
        if low in TF_FALSE:
            return 1, []
        return None, ['TF correct answer must be true or false']
    if qtype == grading.MA:
        try:
            indices = sorted({int(token) - 1 for token in spec.split(',') if token.strip()})
        except ValueError:
            return None, ['MA correct answers must be comma-separated choice numbers']
        if not indices:
            return None, ['MA needs at least one correct choice']
        if indices[0] < 0 or indices[-1] >= num_choices:
            return None, ['MA correct answer out of range 1-%d' % num_choices]
        return indices, []
    # SA
    tokens = [token.strip() for token in spec.split('|') if token.strip()]
    if not tokens:
        return None, ['SA needs at least one accepted answer']
    accepted, errors = [], []
    for token in tokens:
        is_regex = case_sensitive = False
        changed = True
        while changed:
            changed = False
            if token.startswith('re:'):
                is_regex, token, changed = True, token[3:], True
            elif token.startswith('cs:'):
                case_sensitive, token, changed = True, token[3:], True
        token = token.strip()
        if is_regex:
            try:
                re.compile(token)
            except re.error as exc:
                errors.append('Invalid regex %r: %s' % (token, exc))
                continue
        accepted.append({'text': token, 'case_sensitive': case_sensitive,
                         'is_regex': is_regex})
    if not accepted and not errors:
        errors.append('SA needs at least one accepted answer')
    return accepted, errors


def validate_question(question):
    """Append problems to question.errors. Call after type-specific parsing."""
    if question.type not in TYPES:
        question.errors.append('Unknown question type %r' % (question.type,))
        return  # everything else depends on the type
    if not question.title.strip():
        question.errors.append('Missing title')
    if not question.content.strip():
        question.errors.append('Missing question content')
    if question.type in (grading.MC, grading.MA) and len(question.choices) < 2:
        question.errors.append('%s questions need at least 2 choices' % question.type)
    if question.correct is None:
        question.errors.append('Missing correct answer')
    if question.level not in LEVELS:
        question.errors.append('Level must be one of %s' % (LEVELS,))
    try:
        question.points = float(question.points)
    except (TypeError, ValueError):
        question.errors.append('Points must be a number')
    else:
        if question.points < 0:
            question.errors.append('Points must not be negative')
    if question.type == grading.MA and question.ma_strategy not in grading.MA_STRATEGIES:
        question.errors.append('MA strategy must be one of %s' % (grading.MA_STRATEGIES,))
```

`quiz/importers/json_fmt.py`:
```python
"""JSON import format: a list of question objects with explicit fields.

This schema is the documented hand-off format for programmatic generation
(e.g. asking an LLM to convert a legacy quiz PDF into importable JSON).
  MC: "correct": <0-based index>     MA: "correct": [<0-based indices>]
  TF: "correct": true|false          SA: "correct": ["text", {"text", "case_sensitive", "is_regex"}]
"""
import json

from quiz import grading
from quiz.importers.base import ParsedQuestion, validate_question


def parse(data):
    """bytes -> list[ParsedQuestion]. File-level failures yield one errored row."""
    try:
        payload = json.loads(data.decode('utf-8'))
    except (UnicodeDecodeError, ValueError) as exc:
        broken = ParsedQuestion(row=0)
        broken.errors.append('Invalid JSON file: %s' % exc)
        return [broken]
    if not isinstance(payload, list):
        broken = ParsedQuestion(row=0)
        broken.errors.append('JSON root must be a list of question objects')
        return [broken]
    questions = []
    for index, entry in enumerate(payload, start=1):
        question = ParsedQuestion(row=index)
        questions.append(question)
        if not isinstance(entry, dict):
            question.errors.append('Entry must be an object')
            continue
        question.type = str(entry.get('type', '')).upper()
        question.title = str(entry.get('title', ''))
        question.content = str(entry.get('content', ''))
        question.choices = [str(c) for c in entry.get('choices') or []]
        question.points = entry.get('points', 1.0)
        question.category = str(entry.get('category', '') or '')
        question.level = str(entry.get('level', 'easy') or 'easy')
        question.explanation = str(entry.get('explanation', '') or '')
        question.shuffle = bool(entry.get('shuffle', False))
        question.ma_strategy = str(entry.get('ma_strategy',
                                             grading.MA_ALL_OR_NOTHING))
        question.correct = _normalize_correct(question, entry.get('correct'))
        validate_question(question)
    return questions


def _normalize_correct(question, correct):
    if correct is None:
        return None
    if question.type == grading.TF:
        if isinstance(correct, bool):
            return 0 if correct else 1
        question.errors.append('TF correct must be true or false')
        return None
    if question.type == grading.MC:
        if isinstance(correct, int) and 0 <= correct < len(question.choices):
            return correct
        question.errors.append('MC correct must be a 0-based choice index')
        return None
    if question.type == grading.MA:
        if isinstance(correct, list) and correct and \
                all(isinstance(i, int) and 0 <= i < len(question.choices) for i in correct):
            return sorted(set(correct))
        question.errors.append('MA correct must be a non-empty list of 0-based indices')
        return None
    # SA: strings get default flags; objects pass through with defaults filled.
    accepted = []
    for item in correct if isinstance(correct, list) else [correct]:
        if isinstance(item, str):
            accepted.append({'text': item, 'case_sensitive': False, 'is_regex': False})
        elif isinstance(item, dict) and 'text' in item:
            accepted.append({'text': str(item['text']),
                             'case_sensitive': bool(item.get('case_sensitive', False)),
                             'is_regex': bool(item.get('is_regex', False))})
        else:
            question.errors.append('SA accepted answers must be strings or {text, ...} objects')
            return None
    return accepted
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_importers -v 2
```
Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add quiz/importers/ quiz/tests/test_importers.py
git commit -m "feat(quiz): import core with correct-spec syntax and JSON format"
```

---

### Task 13: XLSX parser, writer, and template

**Files:**
- Create: `quiz/importers/xlsx_fmt.py`
- Test: `quiz/tests/test_importers.py`

- [ ] **Step 1: Write the failing tests** (append to `quiz/tests/test_importers.py`)

```python
import io  # add to imports

from openpyxl import Workbook, load_workbook  # add to imports

from quiz.importers import xlsx_fmt  # add to imports


def build_xlsx(rows):
    workbook = Workbook()
    sheet = workbook.active
    sheet.append(list(xlsx_fmt.HEADERS))
    for row in rows:
        sheet.append(row)
    buffer = io.BytesIO()
    workbook.save(buffer)
    buffer.seek(0)
    return buffer


class XlsxFormatTestCase(SimpleTestCase):
    MC_ROW = ['MC', 'loops', 'How many times?', '1', '2', '3', '4', None, None,
              '2', 1, 'python-basics', 'easy', 'Twice.', 'yes', None]

    def test_parse_mc_row(self):
        questions = xlsx_fmt.parse(build_xlsx([self.MC_ROW]))
        self.assertEqual(len(questions), 1)
        question = questions[0]
        self.assertEqual(question.errors, [])
        self.assertEqual(question.type, 'MC')
        self.assertEqual(question.choices, ['1', '2', '3', '4'])
        self.assertEqual(question.correct, 1)
        self.assertEqual(question.category, 'python-basics')
        self.assertTrue(question.shuffle)

    def test_parse_sa_row_with_prefixes(self):
        row = ['SA', 'answer', 'What is it?', None, None, None, None, None, None,
               '42 | re:4[0-9]', 2, '', 'medium', '', '', None]
        question = xlsx_fmt.parse(build_xlsx([row]))[0]
        self.assertEqual(question.errors, [])
        self.assertEqual(question.correct[1]['is_regex'], True)
        self.assertEqual(question.points, 2.0)

    def test_bad_row_collects_errors_with_row_number(self):
        row = ['MC', '', '', '1', '2', None, None, None, None, '9', 1, '', 'easy', '', '', None]
        question = xlsx_fmt.parse(build_xlsx([row]))[0]
        self.assertTrue(question.errors)
        self.assertEqual(question.row, 2)  # 1-based, row 1 is the header

    def test_empty_rows_skipped(self):
        questions = xlsx_fmt.parse(build_xlsx([[None] * 16, self.MC_ROW]))
        self.assertEqual(len(questions), 1)

    def test_invalid_file_is_one_error(self):
        questions = xlsx_fmt.parse(io.BytesIO(b'not an xlsx'))
        self.assertEqual(len(questions), 1)
        self.assertTrue(questions[0].errors)

    def test_writer_round_trips_through_parser(self):
        original = xlsx_fmt.parse(build_xlsx([self.MC_ROW]))
        buffer = xlsx_fmt.write(original)
        reparsed = xlsx_fmt.parse(buffer)
        self.assertEqual(reparsed[0].as_dict(), original[0].as_dict() | {'row': reparsed[0].row})

    def test_template_workbook_has_headers_and_examples(self):
        buffer = xlsx_fmt.template()
        sheet = load_workbook(buffer).active
        headers = [cell.value for cell in sheet[1]]
        self.assertEqual(headers, list(xlsx_fmt.HEADERS))
        self.assertGreaterEqual(sheet.max_row, 4)  # header + one example per type
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_importers.XlsxFormatTestCase -v 2
```
Expected: ImportError on `xlsx_fmt`.

- [ ] **Step 3: Implement**

`quiz/importers/xlsx_fmt.py`:
```python
"""XLSX import/export. One row per question, columns defined by HEADERS.
The 'correct' column uses the parse_correct_spec mini-syntax (1-based)."""
import io

from openpyxl import Workbook, load_workbook

from quiz import grading
from quiz.importers.base import (ParsedQuestion, parse_correct_spec,
                                 validate_question)

MAX_CHOICES = 6
HEADERS = ('type', 'title', 'content',
           'choice_1', 'choice_2', 'choice_3', 'choice_4', 'choice_5', 'choice_6',
           'correct', 'points', 'category', 'level', 'explanation', 'shuffle',
           'ma_strategy')
TRUTHY = ('yes', 'true', '1', 'x')

EXAMPLE_ROWS = (
    ('MC', 'mc-example', 'What is 1 + 1?', '1', '2', '3', '4', None, None,
     '2', 1, 'examples', 'easy', 'Basic arithmetic.', 'yes', None),
    ('MA', 'ma-example', 'Select the even numbers.', '1', '2', '3', '4', None, None,
     '2,4', 2, 'examples', 'medium', '', None, 'partial_credit'),
    ('TF', 'tf-example', 'The sky is blue.', None, None, None, None, None, None,
     'true', 1, 'examples', 'easy', '', None, None),
    ('SA', 'sa-example', 'The answer to everything?', None, None, None, None, None, None,
     '42 | re:forty[ -]two', 1, 'examples', 'hard', 'See Hitchhiker\'s Guide.', None, None),
)


def parse(file_obj):
    """file-like -> list[ParsedQuestion]. File-level failures yield one errored row."""
    try:
        sheet = load_workbook(file_obj, read_only=True, data_only=True).active
    except Exception as exc:  # openpyxl raises various types for bad files
        broken = ParsedQuestion(row=0)
        broken.errors.append('Cannot read XLSX file: %s' % exc)
        return [broken]
    questions = []
    for row_number, cells in enumerate(sheet.iter_rows(min_row=2, values_only=True), start=2):
        cells = list(cells) + [None] * (len(HEADERS) - len(cells))
        if all(cell in (None, '') for cell in cells):
            continue
        data = dict(zip(HEADERS, cells))
        question = ParsedQuestion(row=row_number)
        questions.append(question)
        question.type = str(data['type'] or '').strip().upper()
        question.title = str(data['title'] or '').strip()
        question.content = str(data['content'] or '')
        question.choices = [str(data['choice_%d' % i]).strip()
                            for i in range(1, MAX_CHOICES + 1)
                            if data['choice_%d' % i] not in (None, '')]
        question.points = data['points'] if data['points'] is not None else 1.0
        question.category = str(data['category'] or '').strip()
        question.level = str(data['level'] or 'easy').strip().lower()
        question.explanation = str(data['explanation'] or '')
        question.shuffle = str(data['shuffle'] or '').strip().lower() in TRUTHY
        question.ma_strategy = str(data['ma_strategy'] or '').strip().lower() or \
            grading.MA_ALL_OR_NOTHING
        if question.type in (grading.MC, grading.MA, grading.TF, grading.SA):
            question.correct, errors = parse_correct_spec(
                question.type, data['correct'], len(question.choices))
            question.errors.extend(errors)
        validate_question(question)
    return questions


def _correct_to_spec(question):
    if question.type == grading.MC:
        return str(question.correct + 1)
    if question.type == grading.TF:
        return 'true' if question.correct == 0 else 'false'
    if question.type == grading.MA:
        return ','.join(str(i + 1) for i in question.correct)
    parts = []
    for accepted in question.correct or []:
        prefix = ('re:' if accepted['is_regex'] else '') + \
            ('cs:' if accepted['case_sensitive'] else '')
        parts.append(prefix + accepted['text'])
    return ' | '.join(parts)


def write(questions):
    """list[ParsedQuestion] -> BytesIO of an importable workbook."""
    workbook = Workbook()
    sheet = workbook.active
    sheet.append(list(HEADERS))
    for question in questions:
        choices = list(question.choices) + [None] * (MAX_CHOICES - len(question.choices))
        sheet.append([question.type, question.title, question.content, *choices[:MAX_CHOICES],
                      _correct_to_spec(question), question.points, question.category,
                      question.level, question.explanation,
                      'yes' if question.shuffle else None,
                      question.ma_strategy if question.type == grading.MA else None])
    buffer = io.BytesIO()
    workbook.save(buffer)
    buffer.seek(0)
    return buffer


def template():
    """Downloadable starter workbook: headers + one example row per type."""
    workbook = Workbook()
    sheet = workbook.active
    sheet.append(list(HEADERS))
    for row in EXAMPLE_ROWS:
        sheet.append(list(row))
    buffer = io.BytesIO()
    workbook.save(buffer)
    buffer.seek(0)
    return buffer
```

Round-trip caveat the test accounts for: `write()` emits the parsed-normalized values, so `parse(write(parse(x)))` equals `parse(x)` apart from row numbers — and for MA questions the re-parsed `ma_strategy` survives because `write` emits it explicitly. The MC example writes `ma_strategy=None`, which re-parses to the default `all_or_nothing` — equal to the original default. (If the round-trip test fails on a field, fix `write`, not the test.)

- [ ] **Step 4: Run tests to verify they pass**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_importers -v 2
```
Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add quiz/importers/xlsx_fmt.py quiz/tests/test_importers.py
git commit -m "feat(quiz): XLSX parser, writer and starter template"
```

---

### Task 14: Editor foundation — question bank and question form

**Files:**
- Create: `quiz/views/editor.py`, `quiz/forms.py`, `templates/quiz/question_bank.html`, `templates/quiz/question_form.html`
- Modify: `quiz/models.py` (one classmethod), `quiz/importers/base.py` (move spec serializer here), `quiz/importers/xlsx_fmt.py` (use it), `quiz/urls.py`
- Test: `quiz/tests/test_editor.py`

- [ ] **Step 1: Write the failing tests**

`quiz/tests/test_editor.py`:
```python
from django.test import TestCase
from django.urls import reverse

from quiz.models import QuizQuestion
from quiz.tests.util import create_question, create_user


class QuestionBankTestCase(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.teacher = create_user(username='bank_teacher', user_permissions=('edit_own_quiz',))
        cls.other = create_user(username='bank_other', user_permissions=('edit_own_quiz',))
        cls.student = create_user(username='bank_student')
        cls.mine = create_question(title='bank_mine', authors=(cls.teacher.profile,))
        cls.shared = create_question(title='bank_shared', is_public=True,
                                     authors=(cls.other.profile,))
        cls.private = create_question(title='bank_private', authors=(cls.other.profile,))

    def test_bank_requires_editor_permission(self):
        self.client.force_login(self.student)
        self.assertEqual(self.client.get(reverse('quiz_question_bank')).status_code, 404)

    def test_bank_shows_own_and_public_questions(self):
        self.client.force_login(self.teacher)
        response = self.client.get(reverse('quiz_question_bank'))
        self.assertContains(response, 'bank_mine')
        self.assertContains(response, 'bank_shared')
        self.assertNotContains(response, 'bank_private')

    def test_create_question_via_form(self):
        self.client.force_login(self.teacher)
        response = self.client.post(reverse('quiz_question_create'), {
            'type': 'MC', 'title': 'created_via_form', 'content': 'Pick one.',
            'choices_text': 'first\nsecond\nthird', 'correct_spec': '2',
            'level': 'easy', 'ma_grading_strategy': 'all_or_nothing',
        })
        question = QuizQuestion.objects.get(title='created_via_form')
        self.assertRedirects(response, reverse('quiz_question_edit', args=(question.id,)))
        self.assertEqual(question.choices, ['first', 'second', 'third'])
        self.assertEqual(question.correct_answers, 1)
        self.assertTrue(question.authors.filter(id=self.teacher.profile.id).exists())

    def test_create_question_with_bad_spec_shows_error(self):
        self.client.force_login(self.teacher)
        response = self.client.post(reverse('quiz_question_create'), {
            'type': 'MC', 'title': 'bad', 'content': 'Pick.', 'choices_text': 'a\nb',
            'correct_spec': '9', 'level': 'easy', 'ma_grading_strategy': 'all_or_nothing',
        })
        self.assertEqual(response.status_code, 200)  # re-rendered with errors
        self.assertFalse(QuizQuestion.objects.filter(title='bad').exists())

    def test_edit_forbidden_for_non_author(self):
        self.client.force_login(self.other)
        response = self.client.get(reverse('quiz_question_edit', args=(self.mine.id,)))
        self.assertEqual(response.status_code, 404)

    def test_edit_form_prefills_spec_syntax(self):
        self.client.force_login(self.teacher)
        response = self.client.get(reverse('quiz_question_edit', args=(self.mine.id,)))
        self.assertContains(response, 'value="1"')  # correct_answers 0 -> spec '1'
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_editor -v 2
```
Expected: `NoReverseMatch` for `quiz_question_bank`.

- [ ] **Step 3: Implement**

First, move the spec serializer into base for reuse. In `quiz/importers/base.py`, add (module level):
```python
def correct_to_spec(qtype, correct):
    """Inverse of parse_correct_spec - renders the human syntax."""
    if qtype == grading.MC:
        return str(correct + 1)
    if qtype == grading.TF:
        return 'true' if correct == 0 else 'false'
    if qtype == grading.MA:
        return ','.join(str(i + 1) for i in correct)
    parts = []
    for accepted in correct or []:
        prefix = ('re:' if accepted['is_regex'] else '') + \
            ('cs:' if accepted['case_sensitive'] else '')
        parts.append(prefix + accepted['text'])
    return ' | '.join(parts)
```
In `quiz/importers/xlsx_fmt.py`, delete `_correct_to_spec` and replace its one call with `correct_to_spec(question.type, question.correct)` (add to the base import). Re-run `test_importers` to confirm no regression.

Add to `QuizQuestion` in `quiz/models.py` (no migration needed):
```python
    @classmethod
    def get_bank_questions(cls, user):
        """Questions this editor may see: their own + public bank questions."""
        if user.has_perm('quiz.edit_all_quiz'):
            return cls.objects.all()
        profile = user.profile
        return cls.objects.filter(
            models.Q(is_public=True) | models.Q(authors=profile) |
            models.Q(curators=profile)).distinct()
```

`quiz/forms.py`:
```python
from django import forms
from django.utils.translation import gettext_lazy as _

from quiz.importers.base import correct_to_spec, parse_correct_spec
from quiz.models import Quiz, QuizQuestion


class QuestionForm(forms.ModelForm):
    choices_text = forms.CharField(
        label=_('Choices'), required=False,
        widget=forms.Textarea(attrs={'rows': 6}),
        help_text=_('One choice per line (MC/MA only).'))
    correct_spec = forms.CharField(
        label=_('Correct answer'),
        help_text=_('MC: choice number (e.g. 2). MA: comma list (1,3). TF: true/false. '
                    'SA: answers separated by |, prefix re: for regex, cs: for case-sensitive.'))

    class Meta:
        model = QuizQuestion
        fields = ('type', 'title', 'content', 'category', 'level', 'explanation',
                  'shuffle_choices', 'ma_grading_strategy', 'is_public')

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        if self.instance.pk:
            self.fields['choices_text'].initial = '\n'.join(self.instance.choices or [])
            self.fields['correct_spec'].initial = correct_to_spec(
                self.instance.type, self.instance.correct_answers)

    def clean(self):
        cleaned = super().clean()
        qtype = cleaned.get('type')
        choices = [line.strip() for line in
                   (cleaned.get('choices_text') or '').splitlines() if line.strip()]
        if qtype in ('MC', 'MA') and len(choices) < 2:
            self.add_error('choices_text', _('Provide at least 2 choices.'))
        if qtype:
            correct, errors = parse_correct_spec(
                qtype, cleaned.get('correct_spec', ''), len(choices))
            for error in errors:
                self.add_error('correct_spec', error)
            cleaned['parsed_choices'] = choices if qtype in ('MC', 'MA') else []
            cleaned['parsed_correct'] = correct
        return cleaned

    def save(self, commit=True):
        self.instance.choices = self.cleaned_data['parsed_choices']
        self.instance.correct_answers = self.cleaned_data['parsed_correct']
        return super().save(commit)
```

`quiz/views/editor.py`:
```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.db.models import Q
from django.http import Http404
from django.shortcuts import get_object_or_404
from django.urls import reverse
from django.views.generic import CreateView, ListView, UpdateView

from quiz.forms import QuestionForm
from quiz.models import QuestionType, QuizCategory, QuizLevel, QuizQuestion


class EditorPermissionMixin(LoginRequiredMixin):
    """Quiz editors only: edit_own_quiz or edit_all_quiz. Others see 404."""

    def dispatch(self, request, *args, **kwargs):
        if request.user.is_authenticated and not (
                request.user.has_perm('quiz.edit_own_quiz') or
                request.user.has_perm('quiz.edit_all_quiz')):
            raise Http404()
        return super().dispatch(request, *args, **kwargs)


class QuestionBank(EditorPermissionMixin, ListView):
    template_name = 'quiz/question_bank.html'
    context_object_name = 'questions'
    paginate_by = 50

    def get_queryset(self):
        queryset = QuizQuestion.get_bank_questions(self.request.user) \
            .select_related('category').order_by('-id')
        if self.request.GET.get('category'):
            queryset = queryset.filter(category__slug=self.request.GET['category'])
        if self.request.GET.get('level') in QuizLevel.values:
            queryset = queryset.filter(level=self.request.GET['level'])
        if self.request.GET.get('type') in QuestionType.values:
            queryset = queryset.filter(type=self.request.GET['type'])
        if self.request.GET.get('search'):
            search = self.request.GET['search']
            queryset = queryset.filter(Q(title__icontains=search) |
                                       Q(content__icontains=search))
        return queryset

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['categories'] = QuizCategory.objects.all()
        context['levels'] = QuizLevel.choices
        context['types'] = QuestionType.choices
        return context


class QuestionCreate(EditorPermissionMixin, CreateView):
    model = QuizQuestion
    form_class = QuestionForm
    template_name = 'quiz/question_form.html'

    def form_valid(self, form):
        response = super().form_valid(form)
        self.object.authors.add(self.request.profile)
        return response

    def get_success_url(self):
        return reverse('quiz_question_edit', args=(self.object.id,))


class QuestionEdit(EditorPermissionMixin, UpdateView):
    model = QuizQuestion
    form_class = QuestionForm
    template_name = 'quiz/question_form.html'
    pk_url_kwarg = 'question'

    def get_object(self, queryset=None):
        question = get_object_or_404(QuizQuestion, id=self.kwargs['question'])
        if not question.is_editable_by(self.request.user):
            raise Http404()
        return question

    def get_success_url(self):
        return reverse('quiz_question_edit', args=(self.object.id,))
```

In `quiz/urls.py`, add ABOVE the `/<str:quiz>` include (literal prefixes must match first):
```python
    path('/questions', include([
        path('/', editor.QuestionBank.as_view(), name='quiz_question_bank'),
        path('/new', editor.QuestionCreate.as_view(), name='quiz_question_create'),
        path('/<int:question>/edit', editor.QuestionEdit.as_view(), name='quiz_question_edit'),
    ])),
```
with `from quiz.views import editor, student` replacing the existing import.

- [ ] **Step 4: Create templates**

`templates/quiz/question_bank.html`:
```jinja2
{% extends "common-content.html" %}

{% block body %}
<p>
    <a href="{{ url('quiz_question_create') }}">{{ _('New question') }}</a> ·
    <a href="{{ url('quiz_import') }}">{{ _('Import') }}</a> ·
    <a href="{{ url('quiz_import_template') }}">{{ _('Download XLSX template') }}</a>
</p>
<form method="get" style="margin-bottom: 1em">
    <input type="text" name="search" value="{{ request.GET.get('search', '') }}" placeholder="{{ _('Search...') }}">
    <select name="type"><option value="">{{ _('All types') }}</option>
        {% for value, label in types %}<option value="{{ value }}" {% if value == request.GET.get('type') %}selected{% endif %}>{{ label }}</option>{% endfor %}
    </select>
    <select name="category"><option value="">{{ _('All categories') }}</option>
        {% for cat in categories %}<option value="{{ cat.slug }}" {% if cat.slug == request.GET.get('category') %}selected{% endif %}>{{ cat.name }}</option>{% endfor %}
    </select>
    <select name="level"><option value="">{{ _('All levels') }}</option>
        {% for value, label in levels %}<option value="{{ value }}" {% if value == request.GET.get('level') %}selected{% endif %}>{{ label }}</option>{% endfor %}
    </select>
    <button type="submit">{{ _('Filter') }}</button>
</form>
<form method="get" action="{{ url('quiz_export') }}">
<table class="table">
    <thead><tr><th></th><th>{{ _('Title') }}</th><th>{{ _('Type') }}</th><th>{{ _('Category') }}</th><th>{{ _('Level') }}</th><th>{{ _('Public') }}</th></tr></thead>
    <tbody>
        {% for question in questions %}
        <tr>
            <td><input type="checkbox" name="ids" value="{{ question.id }}"></td>
            <td><a href="{{ url('quiz_question_edit', question.id) }}">{{ question.title }}</a></td>
            <td>{{ question.get_type_display() }}</td>
            <td>{{ question.category.name if question.category else '' }}</td>
            <td>{{ question.get_level_display() }}</td>
            <td>{{ _('Yes') if question.is_public else '' }}</td>
        </tr>
        {% else %}
        <tr><td colspan="6">{{ _('No questions yet.') }}</td></tr>
        {% endfor %}
    </tbody>
</table>
<button type="submit">{{ _('Export selected to XLSX') }}</button>
</form>
{% endblock %}
```

(`quiz_import`, `quiz_import_template`, `quiz_export` come in Task 15 — add URL stubs now pointing at `QuestionBank.as_view()`, replaced next task.)

`templates/quiz/question_form.html`:
```jinja2
{% extends "common-content.html" %}

{% block body %}
<form method="post">
    {% csrf_token %}
    {{ form.as_p() }}
    <button type="submit">{{ _('Save question') }}</button>
</form>
<p><a href="{{ url('quiz_question_bank') }}">{{ _('Back to question bank') }}</a></p>
{% endblock %}
```
(`form.as_p()` with parentheses — jinja2, not Django templates.)

- [ ] **Step 5: Run tests to verify they pass**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_editor quiz.tests.test_importers -v 2
```
Expected: all PASS (including the importer refactor regression check).

- [ ] **Step 6: Commit**

```bash
git add quiz/ templates/quiz/
git commit -m "feat(quiz): question bank and question editor form"
```

---

### Task 15: Import/export views with preview and atomic commit

**Files:**
- Create: `quiz/views/importer.py`, `templates/quiz/import.html`
- Modify: `quiz/forms.py`, `quiz/urls.py`
- Test: `quiz/tests/test_import_views.py`

- [ ] **Step 1: Write the failing tests**

`quiz/tests/test_import_views.py`:
```python
import io
import json

from django.core.files.uploadedfile import SimpleUploadedFile
from django.test import TestCase
from django.urls import reverse

from quiz.models import Quiz, QuizCategory, QuizQuestion
from quiz.tests.util import create_question, create_user

VALID_JSON = [
    {'type': 'MC', 'title': 'imported_mc', 'content': 'Pick.', 'choices': ['a', 'b'],
     'correct': 0, 'category': 'new-cat'},
    {'type': 'TF', 'title': 'imported_tf', 'content': 'True?', 'correct': True},
]


def upload(content, name='quiz.json'):
    return SimpleUploadedFile(name, content, content_type='application/octet-stream')


class ImportFlowTestCase(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.teacher = create_user(username='import_teacher',
                                  user_permissions=('edit_own_quiz',))
        cls.student = create_user(username='import_student')

    def setUp(self):
        self.client.force_login(self.teacher)

    def post_upload(self, payload=VALID_JSON, **extra):
        data = {'file': upload(json.dumps(payload).encode())}
        data.update(extra)
        return self.client.post(reverse('quiz_import'), data)

    def test_requires_editor_permission(self):
        self.client.force_login(self.student)
        self.assertEqual(self.client.get(reverse('quiz_import')).status_code, 404)

    def test_preview_lists_questions_and_new_categories(self):
        response = self.post_upload()
        self.assertContains(response, 'imported_mc')
        self.assertContains(response, 'new-cat')        # flagged as will-be-created

    def test_confirm_commits_questions_and_category(self):
        self.post_upload()
        response = self.client.post(reverse('quiz_import_confirm'))
        self.assertRedirects(response, reverse('quiz_question_bank'))
        self.assertTrue(QuizQuestion.objects.filter(title='imported_mc').exists())
        self.assertTrue(QuizCategory.objects.filter(slug='new-cat').exists())
        question = QuizQuestion.objects.get(title='imported_mc')
        self.assertTrue(question.authors.filter(id=self.teacher.profile.id).exists())

    def test_bad_row_blocks_confirm_entirely(self):
        bad = VALID_JSON + [{'type': 'MC', 'title': '', 'content': '', 'choices': [],
                             'correct': None}]
        response = self.post_upload(payload=bad)
        self.assertNotContains(response, 'quiz_import_confirm_button')
        response = self.client.post(reverse('quiz_import_confirm'))
        self.assertEqual(response.status_code, 302)  # bounced back, nothing created
        self.assertFalse(QuizQuestion.objects.filter(title='imported_mc').exists())

    def test_confirm_without_pending_import_redirects(self):
        response = self.client.post(reverse('quiz_import_confirm'))
        self.assertRedirects(response, reverse('quiz_import'))

    def test_create_quiz_during_import(self):
        self.post_upload(create_quiz='on', quiz_code='imported_quiz',
                         quiz_name='Imported Quiz')
        self.client.post(reverse('quiz_import_confirm'))
        quiz = Quiz.objects.get(code='imported_quiz')
        self.assertEqual(quiz.question_links.count(), 2)
        self.assertFalse(quiz.is_public)  # imported quizzes start private
        self.assertTrue(quiz.authors.filter(id=self.teacher.profile.id).exists())

    def test_xlsx_upload_parses(self):
        from quiz.importers import xlsx_fmt
        buffer = xlsx_fmt.template()  # example rows are themselves importable
        response = self.client.post(reverse('quiz_import'),
                                    {'file': upload(buffer.read(), name='quiz.xlsx')})
        self.assertContains(response, 'mc-example')

    def test_template_download(self):
        response = self.client.get(reverse('quiz_import_template'))
        self.assertEqual(response['Content-Disposition'],
                         'attachment; filename="quiz-template.xlsx"')

    def test_export_selected_questions(self):
        question = create_question(title='export_me', authors=(self.teacher.profile,))
        response = self.client.get(reverse('quiz_export'), {'ids': question.id})
        self.assertEqual(response.status_code, 200)
        self.assertIn('attachment', response['Content-Disposition'])

    def test_export_excludes_invisible_questions(self):
        other = create_user(username='import_other', user_permissions=('edit_own_quiz',))
        secret = create_question(title='secret_q', authors=(other.profile,))
        from openpyxl import load_workbook
        response = self.client.get(reverse('quiz_export'), {'ids': secret.id})
        sheet = load_workbook(io.BytesIO(response.content)).active
        self.assertEqual(sheet.max_row, 1)  # header only
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_import_views -v 2
```
Expected: NoReverseMatch / stub failures.

- [ ] **Step 3: Implement**

Append to `quiz/forms.py`:
```python
class QuizImportForm(forms.Form):
    file = forms.FileField(label=_('XLSX or JSON file'))
    create_quiz = forms.BooleanField(label=_('Also create a quiz from these questions'),
                                     required=False)
    quiz_code = forms.SlugField(label=_('Quiz code'), required=False)
    quiz_name = forms.CharField(label=_('Quiz name'), required=False, max_length=100)

    def clean(self):
        cleaned = super().clean()
        if cleaned.get('create_quiz'):
            if not cleaned.get('quiz_code') or not cleaned.get('quiz_name'):
                raise forms.ValidationError(
                    _('Quiz code and name are required when creating a quiz.'))
            if Quiz.objects.filter(code=cleaned['quiz_code']).exists():
                raise forms.ValidationError(_('A quiz with this code already exists.'))
        return cleaned
```

`quiz/views/importer.py`:
```python
from django.contrib import messages
from django.db import transaction
from django.http import HttpResponse
from django.shortcuts import redirect
from django.utils.translation import gettext as _
from django.views.generic import FormView, View

from quiz.forms import QuizImportForm
from quiz.importers import json_fmt, xlsx_fmt
from quiz.importers.base import ParsedQuestion
from quiz.models import Quiz, QuizCategory, QuizQuestion, QuizQuestionLink
from quiz.views.editor import EditorPermissionMixin

SESSION_KEY = 'quiz_import_pending'
XLSX_MIME = 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'


class QuizImport(EditorPermissionMixin, FormView):
    form_class = QuizImportForm
    template_name = 'quiz/import.html'

    def form_valid(self, form):
        uploaded = form.cleaned_data['file']
        if uploaded.name.lower().endswith('.json'):
            questions = json_fmt.parse(uploaded.read())
        else:
            questions = xlsx_fmt.parse(uploaded)
        existing = set(QuizCategory.objects.values_list('slug', flat=True))
        new_categories = sorted({q.category for q in questions
                                 if q.category and q.category not in existing})
        has_errors = any(q.errors for q in questions) or not questions
        if not has_errors:
            self.request.session[SESSION_KEY] = {
                'questions': [q.as_dict() for q in questions],
                'create_quiz': form.cleaned_data['create_quiz'],
                'quiz_code': form.cleaned_data['quiz_code'],
                'quiz_name': form.cleaned_data['quiz_name'],
            }
        return self.render_to_response(self.get_context_data(
            form=form, preview=questions, has_errors=has_errors,
            new_categories=new_categories))


class QuizImportConfirm(EditorPermissionMixin, View):
    def post(self, request, *args, **kwargs):
        payload = request.session.pop(SESSION_KEY, None)
        if payload is None:
            messages.error(request, _('No pending import - upload a file first.'))
            return redirect('quiz_import')
        questions = [ParsedQuestion.from_dict(data) for data in payload['questions']]
        if any(q.errors for q in questions):
            messages.error(request, _('Import file has errors.'))
            return redirect('quiz_import')
        with transaction.atomic():
            categories = {}
            for parsed in questions:
                if parsed.category and parsed.category not in categories:
                    categories[parsed.category], _created = \
                        QuizCategory.objects.get_or_create(
                            slug=parsed.category,
                            defaults={'name': parsed.category.replace('-', ' ').title()})
            created = []
            for parsed in questions:
                question = QuizQuestion.objects.create(
                    type=parsed.type, title=parsed.title, content=parsed.content,
                    choices=parsed.choices, correct_answers=parsed.correct,
                    explanation=parsed.explanation,
                    category=categories.get(parsed.category),
                    level=parsed.level, shuffle_choices=parsed.shuffle,
                    ma_grading_strategy=parsed.ma_strategy)
                question.authors.add(request.profile)
                created.append((question, parsed.points))
            if payload['create_quiz']:
                quiz = Quiz.objects.create(code=payload['quiz_code'],
                                           name=payload['quiz_name'])
                quiz.authors.add(request.profile)
                for order, (question, points) in enumerate(created):
                    QuizQuestionLink.objects.create(quiz=quiz, question=question,
                                                    points=points, order=order)
        messages.success(request, _('Imported %d questions.') % len(created))
        if payload['create_quiz']:
            return redirect('quiz_edit', quiz=payload['quiz_code'])
        return redirect('quiz_question_bank')


class QuizImportTemplate(EditorPermissionMixin, View):
    def get(self, request, *args, **kwargs):
        response = HttpResponse(xlsx_fmt.template().read(), content_type=XLSX_MIME)
        response['Content-Disposition'] = 'attachment; filename="quiz-template.xlsx"'
        return response


class QuizExport(EditorPermissionMixin, View):
    def get(self, request, *args, **kwargs):
        ids = [int(i) for i in request.GET.getlist('ids') if str(i).isdigit()]
        questions = QuizQuestion.get_bank_questions(request.user).filter(id__in=ids)
        parsed = [ParsedQuestion(
            row=0, type=q.type, title=q.title, content=q.content,
            choices=q.choices or [], correct=q.correct_answers,
            category=q.category.slug if q.category else '', level=q.level,
            explanation=q.explanation, shuffle=q.shuffle_choices,
            ma_strategy=q.ma_grading_strategy) for q in questions]
        response = HttpResponse(xlsx_fmt.write(parsed).read(), content_type=XLSX_MIME)
        response['Content-Disposition'] = 'attachment; filename="quiz-questions.xlsx"'
        return response
```

One template serves both stages — `templates/quiz/import.html` renders the upload form always, and the preview block when `preview` is defined in the context:

```jinja2
{% extends "common-content.html" %}

{% block body %}
{% if preview is defined %}
    <h3>{{ _('Preview') }}</h3>
    {% if new_categories %}
        <p>{{ _('These categories will be created') }}: <b>{{ new_categories|join(', ') }}</b></p>
    {% endif %}
    {% for question in preview %}
        <div style="border:1px solid {{ '#c44' if question.errors else '#ddd' }}; padding:0.8em; margin-bottom:0.8em">
            <b>{{ _('Row') }} {{ question.row }}: [{{ question.type }}] {{ question.title }}</b>
            {% if question.errors %}
                <ul style="color:#c00">{% for error in question.errors %}<li>{{ error }}</li>{% endfor %}</ul>
            {% else %}
                <div>{{ question.content|markdown('default') }}</div>
                {% if question.choices %}<ol>{% for choice in question.choices %}<li>{{ choice }}</li>{% endfor %}</ol>{% endif %}
            {% endif %}
        </div>
    {% endfor %}
    {% if not has_errors %}
        <form action="{{ url('quiz_import_confirm') }}" method="post" id="quiz_import_confirm_button">
            {% csrf_token %}
            <button type="submit">{{ _('Confirm import') }}</button>
        </form>
    {% else %}
        <p style="color:#c00">{{ _('Fix the errors above and re-upload. Nothing was imported.') }}</p>
    {% endif %}
    <hr>
{% endif %}
<h3>{{ _('Upload') }}</h3>
<form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    {{ form.as_p() }}
    <button type="submit">{{ _('Upload and preview') }}</button>
</form>
<p><a href="{{ url('quiz_import_template') }}">{{ _('Download the XLSX template') }}</a>
   — {{ _('or import JSON: a list of objects with type, title, content, choices, correct, category, level, points, explanation, shuffle, ma_strategy.') }}</p>
{% endblock %}
```
In `quiz/urls.py`, replace the Task 14 stubs (above the `/<str:quiz>` include):
```python
    path('/import', include([
        path('/', importer.QuizImport.as_view(), name='quiz_import'),
        path('/confirm', importer.QuizImportConfirm.as_view(), name='quiz_import_confirm'),
        path('/template', importer.QuizImportTemplate.as_view(), name='quiz_import_template'),
    ])),
    path('/export', importer.QuizExport.as_view(), name='quiz_export'),
```
and import `importer` in the views import line.

- [ ] **Step 4: Run tests to verify they pass**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_import_views -v 2
```
Expected: all PASS. (`test_create_quiz_during_import` exercises `quiz_edit`, still a stub — it only checks DB state and the redirect target name resolves, so it passes; the real view lands next task.)

- [ ] **Step 5: Commit**

```bash
git add quiz/ templates/quiz/import.html
git commit -m "feat(quiz): XLSX/JSON import with preview and atomic confirm, export"
```

---

### Task 16: Quiz editor, attempts dashboard, regrade

**Files:**
- Create: `templates/quiz/quiz_form.html`, `templates/quiz/attempts.html`
- Modify: `quiz/views/editor.py`, `quiz/forms.py`, `quiz/urls.py`
- Test: `quiz/tests/test_editor.py`

- [ ] **Step 1: Write the failing tests** (append to `quiz/tests/test_editor.py`)

```python
from quiz.models import Quiz, QuizAttempt  # merge into imports
from quiz.tests.util import create_quiz  # merge into imports


class QuizEditorTestCase(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.teacher = create_user(username='qe_teacher', user_permissions=('edit_own_quiz',))
        cls.student = create_user(username='qe_student')
        cls.question = create_question(title='qe_q', authors=(cls.teacher.profile,))
        cls.quiz = create_quiz(code='qe_quiz', authors=(cls.teacher.profile,),
                               questions=((cls.question, 1.0),))

    def form_data(self, **overrides):
        data = {
            'name': 'Edited name', 'description': '', 'level': 'easy',
            'show_correctness': 'on', 'show_answers': 'on',
            # inline formset management form:
            'question_links-TOTAL_FORMS': '1', 'question_links-INITIAL_FORMS': '1',
            'question_links-MIN_NUM_FORMS': '0', 'question_links-MAX_NUM_FORMS': '1000',
            'question_links-0-id': str(self.quiz.question_links.get().id),
            'question_links-0-quiz': str(self.quiz.id),
            'question_links-0-question': str(self.question.id),
            'question_links-0-points': '3.0', 'question_links-0-order': '0',
        }
        data.update(overrides)
        return data

    def test_create_quiz(self):
        self.client.force_login(self.teacher)
        response = self.client.post(reverse('quiz_create'), {
            'code': 'qe_created', 'name': 'Created', 'level': 'easy',
            'show_correctness': 'on', 'show_answers': 'on',
            'question_links-TOTAL_FORMS': '0', 'question_links-INITIAL_FORMS': '0',
            'question_links-MIN_NUM_FORMS': '0', 'question_links-MAX_NUM_FORMS': '1000',
        })
        quiz = Quiz.objects.get(code='qe_created')
        self.assertRedirects(response, reverse('quiz_edit', args=('qe_created',)))
        self.assertTrue(quiz.authors.filter(id=self.teacher.profile.id).exists())

    def test_edit_updates_link_points(self):
        self.client.force_login(self.teacher)
        self.client.post(reverse('quiz_edit', args=('qe_quiz',)), self.form_data())
        self.assertEqual(self.quiz.question_links.get().points, 3.0)

    def test_edit_404_for_non_author_editor(self):
        other = create_user(username='qe_other', user_permissions=('edit_own_quiz',))
        self.client.force_login(other)
        self.assertEqual(
            self.client.get(reverse('quiz_edit', args=('qe_quiz',))).status_code, 404)

    def test_attempts_page_lists_and_regrades(self):
        attempt = QuizAttempt.start(self.quiz, self.student.profile)
        attempt.save_answer(self.question, 1)  # wrong
        attempt.finalize()
        self.question.correct_answers = 1
        self.question.save()
        self.client.force_login(self.teacher)
        response = self.client.get(reverse('quiz_attempts', args=('qe_quiz',)))
        self.assertContains(response, 'qe_student')
        self.client.post(reverse('quiz_attempts', args=('qe_quiz',)))
        attempt.refresh_from_db()
        self.assertEqual(attempt.score, 1.0)

    def test_attempts_page_404_for_students(self):
        self.client.force_login(self.student)
        self.assertEqual(
            self.client.get(reverse('quiz_attempts', args=('qe_quiz',))).status_code, 404)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_editor -v 2
```
Expected: NoReverseMatch for `quiz_create` / stub behaviour for the rest.

- [ ] **Step 3: Implement**

Append to `quiz/forms.py`:
```python
from django.forms import inlineformset_factory  # add to imports
from quiz.models import QuizQuestionLink  # merge into imports


class QuizForm(forms.ModelForm):
    class Meta:
        model = Quiz
        fields = ('code', 'name', 'description', 'category', 'level', 'time_limit',
                  'max_attempts', 'shuffle_questions', 'show_correctness', 'show_answers',
                  'is_public', 'is_organization_private', 'organizations',
                  'curators', 'testers')

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        if self.instance.pk:
            self.fields['code'].disabled = True


QuizQuestionLinkFormSet = inlineformset_factory(
    Quiz, QuizQuestionLink, fields=('question', 'points', 'order'), extra=3,
    can_delete=True)
```

Append to `quiz/views/editor.py`:
```python
from django.contrib import messages
from django.shortcuts import redirect
from django.utils.translation import gettext as _
from django.views.generic import TemplateView, View

from quiz.forms import QuizForm, QuizQuestionLinkFormSet
from quiz.models import Quiz


class QuizEditorObjectMixin(EditorPermissionMixin):
    """Resolves self.quiz by code and requires object-level edit rights."""

    def dispatch(self, request, *args, **kwargs):
        if 'quiz' in kwargs:
            self.quiz = get_object_or_404(Quiz, code=kwargs['quiz'])
            if request.user.is_authenticated and not self.quiz.is_editable_by(request.user):
                raise Http404()
        else:
            self.quiz = None
        return super().dispatch(request, *args, **kwargs)


class QuizEdit(QuizEditorObjectMixin, TemplateView):
    template_name = 'quiz/quiz_form.html'

    def get_forms(self):
        kwargs = {'instance': self.quiz} if self.quiz else {}
        data = self.request.POST if self.request.method == 'POST' else None
        form = QuizForm(data, **kwargs)
        formset = QuizQuestionLinkFormSet(data, **kwargs) if self.quiz else \
            QuizQuestionLinkFormSet(data)
        for link_form in formset.forms:
            link_form.fields['question'].queryset = \
                QuizQuestion.get_bank_questions(self.request.user)
        return form, formset

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        if 'form' not in context:
            context['form'], context['formset'] = self.get_forms()
        context['editing'] = self.quiz is not None
        return context

    def post(self, request, *args, **kwargs):
        form, formset = self.get_forms()
        if form.is_valid():
            quiz = form.save()
            if self.quiz is None:
                quiz.authors.add(request.profile)
                # Re-bind the formset to the new instance before validating.
                formset = QuizQuestionLinkFormSet(request.POST, instance=quiz)
            if formset.is_valid():
                formset.save()
                messages.success(request, _('Quiz saved.'))
                return redirect('quiz_edit', quiz=quiz.code)
            self.quiz = quiz
        return self.render_to_response(self.get_context_data(form=form, formset=formset))


class QuizAttempts(QuizEditorObjectMixin, TemplateView):
    template_name = 'quiz/attempts.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['quiz'] = self.quiz
        context['attempts'] = self.quiz.attempts.select_related('user__user') \
            .order_by('-started_at')
        context['total_points'] = self.quiz.total_points
        return context

    def post(self, request, *args, **kwargs):
        count = self.quiz.regrade_attempts()
        messages.success(request, _('Regraded %d attempts.') % count)
        return redirect('quiz_attempts', quiz=self.quiz.code)
```

In `quiz/urls.py`:
- remove the Task 7 `quiz_edit` stub;
- above the `/<str:quiz>` include add: `path('/new', editor.QuizEdit.as_view(), name='quiz_create'),`
- inside the `/<str:quiz>` include add:
```python
        path('/edit', editor.QuizEdit.as_view(), name='quiz_edit'),
        path('/attempts', editor.QuizAttempts.as_view(), name='quiz_attempts'),
```

- [ ] **Step 4: Create templates**

`templates/quiz/quiz_form.html`:
```jinja2
{% extends "common-content.html" %}

{% block body %}
<form method="post">
    {% csrf_token %}
    {{ form.as_p() }}
    <h3>{{ _('Questions') }}</h3>
    {{ formset.management_form }}
    <table class="table">
        <tr><th>{{ _('Question') }}</th><th>{{ _('Points') }}</th><th>{{ _('Order') }}</th><th>{{ _('Delete') }}</th></tr>
        {% for link_form in formset %}
        <tr>
            {% for hidden in link_form.hidden_fields() %}{{ hidden }}{% endfor %}
            <td>{{ link_form.question }}</td>
            <td>{{ link_form.points }}</td>
            <td>{{ link_form.order }}</td>
            <td>{{ link_form.DELETE if link_form.instance.pk else '' }}</td>
        </tr>
        {% if link_form.errors %}<tr><td colspan="4" style="color:#c00">{{ link_form.errors }}</td></tr>{% endif %}
        {% endfor %}
    </table>
    <button type="submit">{{ _('Save quiz') }}</button>
</form>
{% if editing %}
<p>
    <a href="{{ url('quiz_detail', quiz.code) }}">{{ _('View quiz') }}</a> ·
    <a href="{{ url('quiz_attempts', quiz.code) }}">{{ _('Attempts & regrade') }}</a>
</p>
{% endif %}
{% endblock %}
```

`templates/quiz/attempts.html`:
```jinja2
{% extends "common-content.html" %}

{% block body %}
<h2>{{ quiz.name }} — {{ _('Attempts') }}</h2>
<form method="post" onsubmit="return confirm('{{ _('Regrade all submitted attempts?') }}')">
    {% csrf_token %}
    <button type="submit">{{ _('Regrade all attempts') }}</button>
</form>
<table class="table">
    <thead><tr><th>{{ _('User') }}</th><th>{{ _('Started') }}</th><th>{{ _('Status') }}</th><th>{{ _('Score') }}</th><th></th></tr></thead>
    <tbody>
        {% for attempt in attempts %}
        <tr>
            <td>{{ attempt.user.user.username }}</td>
            <td>{{ attempt.started_at|date('DATETIME_FORMAT') }}</td>
            <td>{{ _('submitted') if attempt.is_submitted else _('in progress') }}</td>
            <td>{% if attempt.is_submitted %}{{ attempt.score }} / {{ total_points }}{% endif %}</td>
            <td>{% if attempt.is_submitted %}<a href="{{ url('quiz_result', quiz.code, attempt.id) }}">{{ _('view') }}</a>{% endif %}</td>
        </tr>
        {% else %}
        <tr><td colspan="5">{{ _('No attempts yet.') }}</td></tr>
        {% endfor %}
    </tbody>
</table>
{% endblock %}
```

- [ ] **Step 5: Run the full quiz suite**

```bash
docker compose exec site python3 manage.py test quiz -v 2
```
Expected: ALL tests pass.

- [ ] **Step 6: Commit**

```bash
git add quiz/ templates/quiz/
git commit -m "feat(quiz): quiz editor with question links, attempts dashboard, regrade"
```

---

### Task 17: Django admin registrations

**Files:**
- Create: `quiz/admin.py`

- [ ] **Step 1: Implement** (no TDD — declarative registration; verified via smoke check)

`quiz/admin.py`:
```python
from django.contrib import admin

from quiz.models import (Quiz, QuizAnswer, QuizAttempt, QuizCategory,
                         QuizQuestion, QuizQuestionLink)


@admin.register(QuizCategory)
class QuizCategoryAdmin(admin.ModelAdmin):
    list_display = ('name', 'slug')
    prepopulated_fields = {'slug': ('name',)}


@admin.register(QuizQuestion)
class QuizQuestionAdmin(admin.ModelAdmin):
    list_display = ('title', 'type', 'category', 'level', 'is_public')
    list_filter = ('type', 'level', 'category', 'is_public')
    search_fields = ('title', 'content')


class QuizQuestionLinkInline(admin.TabularInline):
    model = QuizQuestionLink
    extra = 1


@admin.register(Quiz)
class QuizAdmin(admin.ModelAdmin):
    list_display = ('code', 'name', 'category', 'level', 'is_public')
    list_filter = ('level', 'category', 'is_public')
    search_fields = ('code', 'name')
    inlines = (QuizQuestionLinkInline,)


class QuizAnswerInline(admin.TabularInline):
    model = QuizAnswer
    extra = 0
    readonly_fields = ('question', 'answer', 'points', 'is_correct', 'saved_at')
    can_delete = False


@admin.register(QuizAttempt)
class QuizAttemptAdmin(admin.ModelAdmin):
    list_display = ('quiz', 'user', 'started_at', 'is_submitted', 'score')
    list_filter = ('is_submitted', 'quiz')
    readonly_fields = ('quiz', 'user', 'started_at', 'submitted_at', 'score',
                       'question_order', 'choice_orders')
    inlines = (QuizAnswerInline,)
```

- [ ] **Step 2: Smoke check**

```bash
docker compose exec site python3 manage.py check
docker compose exec site python3 manage.py test quiz.tests.test_models -v 1
```
Expected: no issues; tests still pass.

- [ ] **Step 3: Commit**

```bash
git add quiz/admin.py
git commit -m "feat(quiz): admin registrations"
```

---

### Task 18: Final verification and integration notes

**Files:**
- Create: `quiz/README.md`

- [ ] **Step 1: Write the app README** (the JSON schema doc promised by the spec)

`quiz/README.md`:
```markdown
# Quiz app

Standalone quiz system: question bank, quizzes with problem-style access
control, auto-graded attempts (MC/MA/TF/SA), XLSX/JSON import, per-quiz
leaderboard. Design spec lives in the deployment repo under
`docs/superpowers/specs/2026-06-11-quiz-system-design.md`.

## Permissions

- `quiz.edit_all_quiz` - staff: everything.
- `quiz.edit_own_quiz` - teachers: create quizzes/questions, edit own,
  import/export. Grant via group or user admin.

## JSON import format

A list of question objects:

    [{
      "type": "MC" | "MA" | "TF" | "SA",
      "title": "short bank name",
      "content": "markdown body",
      "choices": ["a", "b"],            // MC/MA only
      "correct": 0,                      // MC: 0-based index
                                         // MA: [0, 2]
                                         // TF: true | false
                                         // SA: ["42", {"text": "4[0-9]",
                                         //      "case_sensitive": false, "is_regex": true}]
      "points": 1.0,                     // used when also creating a quiz
      "category": "category-slug",       // auto-created if missing
      "level": "easy" | "medium" | "hard",
      "explanation": "markdown, shown on result page",
      "shuffle": false,                  // shuffle choices per attempt
      "ma_strategy": "all_or_nothing" | "partial_credit" |
                     "right_minus_wrong" | "correct_only"
    }]

Tip: hand this schema plus a legacy quiz PDF to an LLM and import the JSON
it produces. The XLSX template (downloadable from the import page) is the
spreadsheet equivalent.

## Timer model

No background jobs. `started_at + time_limit` is the deadline; saves past
deadline + 30s grace are rejected and the attempt is finalized lazily on
the next touch.
```

- [ ] **Step 2: Full suite + lint**

```bash
docker compose exec site python3 manage.py test quiz -v 1
docker compose exec site python3 manage.py check
docker compose exec site flake8 quiz/ || true   # match CI lint; fix any findings
```
Expected: all quiz tests pass, no check issues, no flake8 findings in quiz/ (the repo's `.flake8` config applies). If flake8 isn't installed in the container, run it on the host: `cd dmoj/repo && flake8 quiz/`.

- [ ] **Step 3: Manual smoke test in the browser** (dev stack)

1. `docker compose exec site python3 manage.py migrate`
2. In `/admin/auth/user/`, give a test teacher `quiz | quiz question | Can edit own quizzes`.
3. As the teacher: `/quizzes/questions/` → download template → import it → confirm → create a quiz from the bank at `/quizzes/new`, mark it public.
4. As a student: take it, watch autosave + timer, submit, check result + `/quizzes/<code>/ranking`.
5. **Navigation**: add a "Quizzes" entry pointing at `/quizzes/` via admin → Navigation bars (DMOJ nav is DB-driven; no template change needed).

- [ ] **Step 4: Final commit**

```bash
git add quiz/README.md
git commit -m "docs(quiz): app README with JSON format and permission notes"
git log --oneline master..HEAD   # review the branch history
```

Done. Hand back for code review (superpowers:requesting-code-review) and the deployment image rebuild (openpyxl is in requirements.txt; `docker compose build site`).
