# Quiz Question Code Field Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a unique `code` field to `QuizQuestion` (same `^[a-z0-9]+$` validator as `Problem.code`), unify the `Quiz.code` validator to the same pattern, and wire the code into the Select2 question picker, question bank search, and XLSX import/export.

**Architecture:** Four independent change groups — model+migration, editor form+views, templates, XLSX pipeline — each tested before moving on. All changes are in `dmoj/repo/` (volume-mounted at `/site/` in the container). Tests run via `docker compose exec site python3 manage.py test quiz -v 2` from `dmoj/`.

**Tech Stack:** Django 4.2, django-jinja templates, openpyxl, HeavySelect2Widget (judge.widgets)

---

## File Map

| File | Change |
|------|--------|
| `quiz/models.py` | Add `QuizQuestion.code` field; update `Quiz.code` validator; update `QuizQuestion.__str__` |
| `quiz/migrations/0003_quizquestion_code.py` | AddField (default='') → RunPython → AlterField (unique, no default) |
| `quiz/tests/util.py` | Add `code` param to `create_question` |
| `quiz/tests/test_models.py` | Add code field tests; update `test_str` |
| `quiz/forms.py` | Add `'code'` as first entry in `QuestionForm.Meta.fields` |
| `quiz/views/select2.py` | Add `code__icontains` filter; update `get_name`; add `'code'` to `.only()` |
| `quiz/views/editor.py` | Add `code__icontains` to `QuestionBank.get_queryset` search |
| `quiz/importers/base.py` | Add `code: str = ''` to `ParsedQuestion`; add code format validation in `validate_question` |
| `quiz/importers/xlsx_fmt.py` | Add `'code'` as first column in `HEADERS`/`HEADER_LABELS`; shift `COL_WIDTHS`; update helpers, `parse()`, `write()`, `template()` |
| `quiz/views/importer.py` | Pass `code=parsed.code` in confirm; add within-batch + DB duplicate checks; add `code=q.code` to `QuizExport` |
| `quiz/tests/test_importer.py` | New — XLSX parse/validate/import tests |
| `templates/quiz/question_bank.html` | Add Code column header and cell |
| `templates/quiz/question_form.html` | Add code field row before Type section |

---

## Task 1: Model field + migration + util

**Files:**
- Modify: `quiz/models.py`
- Create: `quiz/migrations/0003_quizquestion_code.py`
- Modify: `quiz/tests/util.py`
- Modify: `quiz/tests/test_models.py`

- [ ] **Step 1.1: Write failing model tests**

Add to the bottom of `quiz/tests/test_models.py`:

```python
from django.core.exceptions import ValidationError


class QuizQuestionCodeTest(TestCase):
    def _make(self, code, **kwargs):
        defaults = {
            'type': 'MC', 'title': code, 'content': 'body',
            'choices': ['a', 'b'], 'correct_answers': 0,
        }
        defaults.update(kwargs)
        return QuizQuestion(code=code, **defaults)

    def test_valid_code_accepted(self):
        q = self._make('py101q1')
        q.full_clean()  # should not raise

    def test_code_required(self):
        q = QuizQuestion(
            type='MC', title='t', content='b',
            choices=['a', 'b'], correct_answers=0)
        with self.assertRaises(ValidationError):
            q.full_clean()

    def test_invalid_code_rejected(self):
        for bad in ('has_underscore', 'Has-Upper', 'has space', ''):
            with self.subTest(code=bad):
                q = self._make(bad)
                with self.assertRaises(ValidationError):
                    q.full_clean()

    def test_duplicate_code_rejected(self):
        QuizQuestion.objects.create(
            code='dupcode', type='MC', title='dup',
            content='body', choices=['a', 'b'], correct_answers=0)
        q = self._make('dupcode', title='dup2')
        with self.assertRaises(Exception):
            q.save()

    def test_str_includes_code(self):
        q = self._make('mycode')
        q.title = 'My title'
        self.assertEqual(str(q), 'mycode: My title')


class QuizCodeValidatorTest(TestCase):
    def test_underscore_rejected_by_new_validator(self):
        from django.core.exceptions import ValidationError
        q = Quiz(
            code='has_underscore', name='test',
        )
        with self.assertRaises(ValidationError):
            q.full_clean()

    def test_lowercase_alphanumeric_accepted(self):
        q = Quiz(code='validcode123', name='test')
        q.full_clean()  # should not raise
```

- [ ] **Step 1.2: Run tests to confirm they fail**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_models.QuizQuestionCodeTest quiz.tests.test_models.QuizCodeValidatorTest -v 2
```

Expected: `ERROR` — `QuizQuestion has no field named 'code'` (or similar).

- [ ] **Step 1.3: Add `code` field to `QuizQuestion`, update `Quiz.code` validator, update `__str__`**

In `quiz/models.py`, replace the existing `QuizQuestion` class opening (before the `type` field) to add `code` as the **first** field after the class declaration:

```python
# At the top of the file, RegexValidator is already imported. Confirm this line exists:
from django.core.validators import MinValueValidator, RegexValidator
```

In `QuizQuestion`, add `code` as the first field (before `type`):

```python
class QuizQuestion(models.Model):
    code = models.CharField(
        max_length=32, unique=True, verbose_name=_('question code'),
        validators=[RegexValidator('^[a-z0-9]+$',
                                   _('Question code must be ^[a-z0-9]+$'))],
    )
    type = models.CharField(...)   # existing — unchanged
```

Update `__str__`:

```python
def __str__(self):
    return f'{self.code}: {self.title}'
```

For `Quiz.code`, change the existing `RegexValidator` regex from `'^[a-z0-9_]+$'` to `'^[a-z0-9]+$'` and update its message:

```python
code = models.CharField(
    max_length=32, verbose_name=_('quiz code'), unique=True,
    validators=[RegexValidator('^[a-z0-9]+$',
                               _('Quiz code must be ^[a-z0-9]+$'))])
```

- [ ] **Step 1.4: Update `create_question` in `quiz/tests/util.py`**

```python
import re as _re

def create_question(title='question', qtype='MC', code=None, **kwargs):
    if code is None:
        code = _re.sub('[^a-z0-9]', '', title.lower()) or 'q1'
    defaults = {
        'content': 'Body of %s' % title,
        'choices': ['a', 'b', 'c', 'd'],
        'correct_answers': 0,
    }
    defaults.update(kwargs)
    authors = defaults.pop('authors', ())
    curators = defaults.pop('curators', ())
    question = QuizQuestion.objects.create(
        code=code, title=title, type=qtype, **defaults)
    question.authors.set(authors)
    question.curators.set(curators)
    return question
```

- [ ] **Step 1.5: Fix `test_str` in `test_models.py` to match new `__str__`**

Find and update:

```python
# Old:
def test_str(self):
    self.assertEqual(str(self.question), 'loops')

# New:
def test_str(self):
    self.assertEqual(str(self.question), 'loops: loops')
```

(`create_question(title='loops')` auto-generates `code='loops'` from the title.)

- [ ] **Step 1.6: Write the migration file**

Create `quiz/migrations/0003_quizquestion_code.py`:

```python
from django.core.validators import RegexValidator
from django.db import migrations, models


def set_question_codes(apps, schema_editor):
    QuizQuestion = apps.get_model('quiz', 'QuizQuestion')
    for q in QuizQuestion.objects.order_by('id'):
        q.code = f'q{q.id}'
        q.save(update_fields=['code'])


class Migration(migrations.Migration):
    dependencies = [
        ('quiz', '0002_quizquestion_choice_explanations'),
    ]

    operations = [
        # Step A: add column with a temporary default so existing rows aren't NULL
        migrations.AddField(
            model_name='quizquestion',
            name='code',
            field=models.CharField(
                default='',
                max_length=32,
                validators=[RegexValidator(
                    '^[a-z0-9]+$', 'Question code must be ^[a-z0-9]+$')],
                verbose_name='question code',
            ),
            preserve_default=False,  # default is only for the migration, not the model
        ),
        # Step B: populate deterministic codes for all existing rows
        migrations.RunPython(set_question_codes, migrations.RunPython.noop),
        # Step C: add unique constraint and drop default
        migrations.AlterField(
            model_name='quizquestion',
            name='code',
            field=models.CharField(
                max_length=32,
                unique=True,
                validators=[RegexValidator(
                    '^[a-z0-9]+$', 'Question code must be ^[a-z0-9]+$')],
                verbose_name='question code',
            ),
        ),
        # Update Quiz.code validator (DB-level no-op; updates migration state)
        migrations.AlterField(
            model_name='quiz',
            name='code',
            field=models.CharField(
                max_length=32,
                unique=True,
                validators=[RegexValidator(
                    '^[a-z0-9]+$', 'Quiz code must be ^[a-z0-9]+$')],
                verbose_name='quiz code',
            ),
        ),
    ]
```

- [ ] **Step 1.7: Apply migration**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py migrate quiz
```

Expected: `Applying quiz.0003_quizquestion_code... OK`

- [ ] **Step 1.8: Run model tests**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_models quiz.tests.test_attempts -v 2
```

Expected: All pass.

- [ ] **Step 1.9: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add quiz/models.py quiz/migrations/0003_quizquestion_code.py quiz/tests/util.py quiz/tests/test_models.py
git commit -m "feat(quiz): add QuizQuestion.code field, unify code validator"
```

---

## Task 2: Editor form + Select2 view + question bank view

**Files:**
- Modify: `quiz/forms.py`
- Modify: `quiz/views/select2.py`
- Modify: `quiz/views/editor.py`
- Modify: `quiz/tests/test_models.py`

- [ ] **Step 2.1: Write failing tests for Select2 code search and bank search**

Add to `quiz/tests/test_models.py`:

```python
from django.urls import reverse


class Select2CodeSearchTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.editor = create_user(
            username='s2editor', user_permissions=('edit_own_quiz',))
        cls.question = create_question(
            title='Python loops', code='pyloops1',
            authors=(cls.editor.profile,))

    def test_search_by_code_prefix(self):
        self.client.force_login(self.editor)
        resp = self.client.get(
            reverse('quiz_question_select2'), {'term': 'pyloops'})
        self.assertEqual(resp.status_code, 200)
        data = resp.json()
        ids = [r['id'] for r in data['results']]
        self.assertIn(self.question.id, ids)

    def test_result_text_includes_code(self):
        self.client.force_login(self.editor)
        resp = self.client.get(
            reverse('quiz_question_select2'), {'term': 'pyloops1'})
        data = resp.json()
        result = next(r for r in data['results']
                      if r['id'] == self.question.id)
        self.assertIn('pyloops1', result['text'])


class QuestionBankCodeSearchTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.editor = create_user(
            username='bankcodeeditor', user_permissions=('edit_own_quiz',))
        cls.question = create_question(
            title='Mutable types', code='muttype1',
            authors=(cls.editor.profile,))

    def test_bank_search_by_code(self):
        self.client.force_login(self.editor)
        resp = self.client.get(
            reverse('quiz_question_bank'), {'search': 'muttype1'})
        self.assertEqual(resp.status_code, 200)
        self.assertIn(self.question, resp.context['questions'])
```

- [ ] **Step 2.2: Run tests to confirm failure**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_models.Select2CodeSearchTest quiz.tests.test_models.QuestionBankCodeSearchTest -v 2
```

Expected: FAIL — searches return empty or text doesn't contain code.

- [ ] **Step 2.3: Update `QuizQuestionLinkForm` and `QuestionForm` in `quiz/forms.py`**

In `QuestionForm.Meta.fields`, add `'code'` as the first item:

```python
class Meta:
    model = QuizQuestion
    fields = ('code', 'type', 'title', 'content', 'category', 'level',
              'explanation', 'shuffle_choices', 'ma_grading_strategy',
              'is_public')
```

- [ ] **Step 2.4: Update `QuizQuestionSelect2View` in `quiz/views/select2.py`**

```python
from django.db.models import Q
from django.http import Http404

from judge.views.select2 import Select2View
from quiz.models import QuizQuestion


class QuizQuestionSelect2View(Select2View):
    def get(self, request, *args, **kwargs):
        if not request.user.is_authenticated or not (
            request.user.has_perm('quiz.edit_own_quiz') or
            request.user.has_perm('quiz.edit_all_quiz')
        ):
            raise Http404()
        return super().get(request, *args, **kwargs)

    def get_queryset(self):
        return QuizQuestion.get_bank_questions(self.request.user).filter(
            Q(code__icontains=self.term) |
            Q(title__icontains=self.term) |
            Q(content__icontains=self.term)
        ).only('id', 'code', 'title', 'type')

    def get_name(self, obj):
        return f'[{obj.type}] {obj.code}: {obj.title}'
```

- [ ] **Step 2.5: Update `QuestionBank.get_queryset` in `quiz/views/editor.py`**

Find the search filter block in `QuestionBank.get_queryset` and add `code__icontains`:

```python
if self.request.GET.get('search'):
    search = self.request.GET['search']
    queryset = queryset.filter(
        Q(code__icontains=search) |
        Q(title__icontains=search) |
        Q(content__icontains=search))
```

- [ ] **Step 2.6: Run tests**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_models.Select2CodeSearchTest quiz.tests.test_models.QuestionBankCodeSearchTest -v 2
```

Expected: All pass.

- [ ] **Step 2.7: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add quiz/forms.py quiz/views/select2.py quiz/views/editor.py quiz/tests/test_models.py
git commit -m "feat(quiz): wire question code into form, Select2, and bank search"
```

---

## Task 3: Templates

**Files:**
- Modify: `templates/quiz/question_form.html`
- Modify: `templates/quiz/question_bank.html`

- [ ] **Step 3.1: Add code field to `question_form.html`**

In `templates/quiz/question_form.html`, after the `{% csrf_token %}` line and before the `{# ── 1. Type ── #}` comment, insert a new "Identity" section as the very first visible section:

```html
{# ── 0. Code ── #}
<div class="qf-section">
    <h3>{{ _('Question Code') }}</h3>
    <div class="field-row{% if form.code.errors %} has-error{% endif %}">
        <label for="{{ form.code.id_for_label }}">{{ _('Code') }} <span class="req">*</span></label>
        {{ form.code }}
        {% for e in form.code.errors %}<div class="field-error">{{ e }}</div>{% endfor %}
        <div class="field-hint">{{ _('Unique short identifier, e.g. py101q1 — lowercase letters and digits only') }}</div>
    </div>
</div>
```

Insert this block immediately before the `{# ── 1. Type ── #}` comment (around line 194).

- [ ] **Step 3.2: Add Code column to `question_bank.html`**

In `templates/quiz/question_bank.html`, update the `<thead>` row to add a Code column:

```html
<thead><tr>
    <th></th>
    <th>{{ _('Code') }}</th>
    <th>{{ _('Title') }}</th>
    <th>{{ _('Type') }}</th>
    <th>{{ _('Category') }}</th>
    <th>{{ _('Level') }}</th>
    <th>{{ _('Public') }}</th>
</tr></thead>
```

Update each `<tr>` in the `{% for question in questions %}` loop to add a code cell after the checkbox:

```html
<tr>
    <td><input type="checkbox" name="ids" value="{{ question.id }}"></td>
    <td><code>{{ question.code }}</code></td>
    <td><a href="{{ url('quiz_question_edit', question.id) }}">{{ question.title }}</a></td>
    <td>{{ question.get_type_display() }}</td>
    <td>{{ question.category.name if question.category else '' }}</td>
    <td>{{ question.get_level_display() }}</td>
    <td>{{ _('Yes') if question.is_public else '' }}</td>
</tr>
```

Also update the `colspan` in the empty-state row from `6` to `7`:

```html
<tr><td colspan="7">{{ _('No questions yet.') }}</td></tr>
```

- [ ] **Step 3.3: Run full quiz test suite and system check**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz -v 2
docker compose exec site python3 manage.py check
```

Expected: All tests pass, no system check errors.

- [ ] **Step 3.4: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add templates/quiz/question_form.html templates/quiz/question_bank.html
git commit -m "feat(quiz): show question code in bank table and question form"
```

---

## Task 4: XLSX import/export pipeline

**Files:**
- Modify: `quiz/importers/base.py`
- Modify: `quiz/importers/xlsx_fmt.py`
- Modify: `quiz/views/importer.py`
- Create: `quiz/tests/test_importer.py`

- [ ] **Step 4.1: Write failing importer tests**

Create `quiz/tests/test_importer.py`:

```python
import io

from django.test import TestCase

from quiz.importers import xlsx_fmt
from quiz.importers.base import ParsedQuestion
from quiz.tests.util import create_question, create_user


def _make_xlsx(rows):
    """Build a minimal XLSX file with the current HEADERS and given data rows."""
    from openpyxl import Workbook
    wb = Workbook()
    ws = wb.active
    ws.append(list(xlsx_fmt.HEADER_LABELS))
    for row in rows:
        ws.append(row)
    buf = io.BytesIO()
    wb.save(buf)
    buf.seek(0)
    return buf


def _row(code='mycode1', qtype='Multiple Choice', title='Q title',
         content='Q body', correct='1', points=1,
         category='', level='Easy', explanation='',
         shuffle='', ma_strategy=''):
    """Build a full XLSX row with 23 values matching current HEADERS after code is added."""
    choices = ['Choice A', '', 'Choice B', '', None, '', None, '', None, '', None, '']
    return [code, qtype, title, content] + choices + [
        correct, points, category, level, explanation, shuffle, ma_strategy]


class XlsxParseCodeTest(TestCase):
    def test_parse_sets_code_from_column(self):
        buf = _make_xlsx([_row(code='pyq001')])
        questions = xlsx_fmt.parse(buf)
        self.assertEqual(len(questions), 1)
        self.assertFalse(questions[0].errors, questions[0].errors)
        self.assertEqual(questions[0].code, 'pyq001')

    def test_invalid_code_format_adds_error(self):
        buf = _make_xlsx([_row(code='has_underscore')])
        questions = xlsx_fmt.parse(buf)
        self.assertTrue(any('code' in e.lower() for e in questions[0].errors))

    def test_missing_code_adds_error(self):
        buf = _make_xlsx([_row(code='')])
        questions = xlsx_fmt.parse(buf)
        self.assertTrue(any('code' in e.lower() for e in questions[0].errors))


class XlsxImportDuplicateCodeTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.editor = create_user(
            username='imp_editor', user_permissions=('edit_own_quiz',))
        cls.existing = create_question(
            title='existing', code='existing1',
            authors=(cls.editor.profile,))

    def test_duplicate_code_in_batch_blocked(self):
        from django.urls import reverse
        buf = _make_xlsx([
            _row(code='samecode', title='Q1'),
            _row(code='samecode', title='Q2'),
        ])
        self.client.force_login(self.editor)
        resp = self.client.post(reverse('quiz_import'), {'file': buf})
        # Both rows should have errors about duplicate codes
        questions = resp.context['preview']
        duplicate_errors = [q for q in questions
                            if any('duplicate' in e.lower() or 'samecode' in e.lower()
                                   for e in q.errors)]
        self.assertTrue(len(duplicate_errors) >= 1)

    def test_code_already_in_db_blocked(self):
        from django.urls import reverse
        buf = _make_xlsx([_row(code='existing1', title='Conflict')])
        self.client.force_login(self.editor)
        resp = self.client.post(reverse('quiz_import'), {'file': buf})
        questions = resp.context['preview']
        self.assertTrue(any('existing1' in e for q in questions for e in q.errors))
```

- [ ] **Step 4.2: Run tests to confirm failure**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_importer -v 2
```

Expected: FAIL — `ParsedQuestion` has no `code` attribute, or `xlsx_fmt` has no `code` column.

- [ ] **Step 4.3: Add `code` to `ParsedQuestion` and validate in `quiz/importers/base.py`**

In `base.py`, add `code` field to the dataclass (after `row`):

```python
@dataclasses.dataclass
class ParsedQuestion:
    row: int
    code: str = ''
    type: str = ''
    title: str = ''
    # ... rest unchanged
```

In `validate_question`, add code validation at the top of the function (after the type check):

```python
import re

_CODE_RE = re.compile('^[a-z0-9]+$')

def validate_question(question):
    """Append problems to question.errors. Call after type-specific parsing."""
    if not question.code or not _CODE_RE.match(question.code):
        question.errors.append(
            'Question code is required and must match ^[a-z0-9]+$')
    if question.type not in TYPES:
        question.errors.append('Unknown question type %r' % (question.type,))
        return
    # ... rest unchanged
```

Note: place the `import re` at the module level at the top of `base.py` (it's already used for SA regex compilation, but it's inside the function — move it to module level while you're here to avoid repeated imports).

- [ ] **Step 4.4: Update `HEADERS`, `HEADER_LABELS`, `COL_WIDTHS`, and helpers in `quiz/importers/xlsx_fmt.py`**

**`HEADERS`** — prepend `'code'`:

```python
HEADERS = (
    'code',
    'type', 'title', 'content',
    'choice_1', 'choice_explanation_1',
    'choice_2', 'choice_explanation_2',
    'choice_3', 'choice_explanation_3',
    'choice_4', 'choice_explanation_4',
    'choice_5', 'choice_explanation_5',
    'choice_6', 'choice_explanation_6',
    'correct',
    'points', 'category', 'level',
    'explanation', 'shuffle', 'ma_strategy',
)
```

**`HEADER_LABELS`** — prepend `'Code'`:

```python
HEADER_LABELS = (
    'Code',
    'Type', 'Title', 'Question',
    'Choice 1', 'Explanation 1',
    'Choice 2', 'Explanation 2',
    'Choice 3', 'Explanation 3',
    'Choice 4', 'Explanation 4',
    'Choice 5', 'Explanation 5',
    'Choice 6', 'Explanation 6',
    'Correct Answer',
    'Points', 'Category', 'Level',
    'Explanation', 'Shuffle Choices', 'MA Strategy',
)
```

**`HEADER_NOTES`** — add a note for `'Code'`:

```python
HEADER_NOTES = {
    'Code': (
        'Unique short identifier for this question — lowercase letters and '
        'digits only, no spaces or underscores. Examples: py101q1, tfindent3'
    ),
    'Type': ...,  # unchanged
    ...
}
```

**`COL_WIDTHS`** — shift all columns right by one letter, add `'A'` for code. Old A→B, B→C, etc.:

```python
# 23 columns: A through W
COL_WIDTHS = {
    'A': 16,                          # code
    'B': 18,  'C': 28,  'D': 46,     # type, title, content
    'E': 22,  'F': 32,               # choice_1, explanation_1
    'G': 22,  'H': 32,               # choice_2, explanation_2
    'I': 22,  'J': 32,               # choice_3, explanation_3
    'K': 22,  'L': 32,               # choice_4, explanation_4
    'M': 22,  'N': 32,               # choice_5, explanation_5
    'O': 22,  'P': 32,               # choice_6, explanation_6
    'Q': 32,                          # correct
    'R': 8,   'S': 18,  'T': 12,    # points, category, level
    'U': 40,  'V': 14,  'W': 18,    # explanation, shuffle, ma_strategy
}
```

Also update the comment on the line above `COL_WIDTHS` from `# 22 columns` to `# 23 columns`.

**Row helper functions** — add `code` as first parameter and first element:

```python
def _mc(code, title, content, choices_expls, correct_1based, points,
        category, level, explanation, shuffle=False):
    row = [code, 'Multiple Choice', title, content]
    for i in range(MAX_CHOICES):
        if i < len(choices_expls):
            row += list(choices_expls[i])
        else:
            row += [None, None]
    row += [str(correct_1based), points, category, level,
            explanation, 'Yes' if shuffle else None, None]
    return tuple(row)


def _ma(code, title, content, choices_expls, correct_csv, points,
        category, level, explanation, strategy='Partial credit'):
    row = [code, 'Multiple Answer', title, content]
    for i in range(MAX_CHOICES):
        if i < len(choices_expls):
            row += list(choices_expls[i])
        else:
            row += [None, None]
    row += [correct_csv, points, category, level, explanation, 'Yes', strategy]
    return tuple(row)


def _tf(code, title, content, correct_bool, points, category, level, explanation):
    row = [code, 'True/False', title, content] + [None] * (MAX_CHOICES * 2)
    row += ['True' if correct_bool else 'False', points, category,
            level, explanation, None, None]
    return tuple(row)


def _sa(code, title, content, patterns, points, category, level, explanation):
    row = [code, 'Short Answer', title, content] + [None] * (MAX_CHOICES * 2)
    row += [' | '.join(patterns), points, category, level,
            explanation, None, None]
    return tuple(row)
```

**`EXAMPLE_ROWS`** — each of the 14 helper calls in `EXAMPLE_ROWS` gains a code string as its new first argument. The existing arguments are unchanged. Here is the mapping (in order):

| Call | New first argument (code) |
|------|--------------------------|
| `_mc('list len()', ...)` | `'pylistlen'` |
| `_mc('Floor division', ...)` | `'pyfloordiv'` |
| `_mc('Function keyword', ...)` | `'pyfunckey'` |
| `_mc('type() output', ...)` | `'pytypeof'` |
| `_mc('String slicing', ...)` | `'pystrslice'` |
| `_ma('Mutable types', ...)` | `'pymutable'` |
| `_ma('Valid list methods', ...)` | `'pylistmeth'` |
| `_tf('Indentation blocks', ...)` | `'pyindent'` |
| `_tf('0 == False', ...)` | `'pyzerofalse'` |
| `_tf('String immutability', ...)` | `'pystrimm'` |
| `_sa('len() function name', ...)` | `'pylenname'` |
| `_sa('Integer division operator', ...)` | `'pyintdivop'` |
| `_sa('Output of upper()', ...)` | `'pyupper'` |
| `_sa('Range result', ...)` | `'pyrangeres'` |

For example, the first row changes from:
```python
_mc('list len()',
    'What does len([10, 20, 30]) return?',
    ...
```
to:
```python
_mc('pylistlen',
    'list len()',
    'What does len([10, 20, 30]) return?',
    ...
```
Apply this same prepend-code pattern to all 14 rows.

**`parse()` function** — extract `code` from `data`:

After the line `q = ParsedQuestion(row=row_num)`, add:

```python
q.code = str(data.get('code') or '').strip().lower()
```

Place this before `raw_type = str(data['type'] or '').strip()`.

**`write()` function** — prepend `q.code` to each row:

```python
ws.append([
    q.code,
    TYPE_DISPLAY.get(q.type, q.type), q.title, q.content,
    *interleaved,
    correct_text,
    q.points, q.category, LEVEL_DISPLAY.get(q.level, q.level),
    q.explanation, 'Yes' if q.shuffle else None,
    MA_STRATEGY_DISPLAY.get(q.ma_strategy, None) if q.type == grading.MA else None,
])
```

- [ ] **Step 4.5: Update `quiz/views/importer.py`**

**In `QuizImport.form_valid`**, after `questions = xlsx_fmt.parse(uploaded)` (or `json_fmt.parse()`), add duplicate-code checks before the `has_errors` calculation:

```python
# Check for code duplicates within the batch
seen_codes = {}
for q in questions:
    if q.code:
        if q.code in seen_codes:
            q.errors.append(
                f'Duplicate code {q.code!r} in this file '
                f'(first seen on row {seen_codes[q.code]})')
        else:
            seen_codes[q.code] = q.row

# Check for code conflicts with existing DB records
batch_codes = {q.code for q in questions if q.code and not q.errors}
if batch_codes:
    db_conflicts = set(
        QuizQuestion.objects.filter(code__in=batch_codes)
        .values_list('code', flat=True))
    for q in questions:
        if q.code in db_conflicts:
            q.errors.append(
                f'Code {q.code!r} already exists in the question bank')
```

**In `QuizImportConfirm.post`**, in the `QuizQuestion.objects.create(...)` call, add `code=parsed.code`:

```python
question = QuizQuestion.objects.create(
    code=parsed.code,
    type=parsed.type, title=parsed.title,
    content=parsed.content,
    choices=parsed.choices,
    correct_answers=parsed.correct,
    explanation=parsed.explanation,
    category=categories.get(parsed.category),
    level=parsed.level,
    shuffle_choices=parsed.shuffle,
    ma_grading_strategy=parsed.ma_strategy)
```

**In `QuizExport.get`**, add `code=q.code` to the `ParsedQuestion` constructor:

```python
parsed = [ParsedQuestion(
    row=0, code=q.code, type=q.type, title=q.title, content=q.content,
    choices=q.choices or [], correct=q.correct_answers,
    category=q.category.slug if q.category else '',
    level=q.level, explanation=q.explanation,
    shuffle=q.shuffle_choices,
    ma_strategy=q.ma_grading_strategy) for q in questions]
```

- [ ] **Step 4.6: Run importer tests**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_importer -v 2
```

Expected: All pass.

- [ ] **Step 4.7: Run full test suite**

```bash
docker compose exec site python3 manage.py test quiz -v 2
```

Expected: All tests pass.

- [ ] **Step 4.8: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add quiz/importers/base.py quiz/importers/xlsx_fmt.py \
        quiz/views/importer.py quiz/tests/test_importer.py
git commit -m "feat(quiz): add code column to XLSX import/export pipeline"
```
