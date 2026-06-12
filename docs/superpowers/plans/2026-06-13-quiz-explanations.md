# Quiz Explanations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Merge `QuizQuestion.choices`/`choice_explanations` into a single object list, replace the two confusing `show_correctness`/`show_answers` booleans on `Quiz` with a single `result_feedback` enum, and surface per-choice "Why?" expandables + question-level explanations on the result page.

**Architecture:** One Django data migration handles the schema change; forms, importers, views, and templates are updated in layers so each task leaves the test suite green. TF/SA questions are unaffected (their `choices` stays `[]`). The per-choice explanation JS already exists in `question_form.html` — the editor needs only an `__init__` and `save()` fix in the form, not a template change.

**Tech Stack:** Django 4.2, MariaDB, django_jinja (Jinja2), openpyxl. All tests run via Docker: `docker compose exec site python3 manage.py test quiz -v 2` from `dmoj/`.

---

## Environment (read first)

- **Repo root for all file paths:** `dmoj/repo/`
- **Tests:** `cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj && docker compose exec site python3 manage.py test quiz -v 2`
- **Migrations:** `docker compose exec site python3 manage.py migrate`
- **Stack must be running:** `docker compose up -d db redis site`
- **Spec:** `docs/superpowers/specs/2026-06-13-quiz-explanations-design.md` (outer repo)

---

## File map

| File | Action | What changes |
|---|---|---|
| `quiz/models.py` | Modify | Add `ResultFeedback`; replace 2 booleans with `result_feedback`; drop `choice_explanations` |
| `quiz/migrations/0006_choices_result_feedback.py` | Create | Data + schema migration |
| `quiz/forms.py` | Modify | `QuestionForm.__init__`/`.save()` read/write new choices format; `QuizForm` fields |
| `quiz/admin.py` | Modify | Replace `show_correctness`/`show_answers` fieldset entry; remove `choice_explanations` from question fieldset |
| `quiz/importers/base.py` | Modify | Drop `choice_explanations` from `ParsedQuestion`; `choices` becomes list of dicts |
| `quiz/importers/xlsx_fmt.py` | Modify | Parse builds `{text, explanation}` objects; write extracts `.text`/`.explanation` |
| `quiz/importers/json_fmt.py` | Modify | Accept choices as strings or `{text, explanation}` objects |
| `quiz/views/student.py` | Modify | `QuizResult`: swap context vars for `result_feedback` |
| `templates/quiz/result.html` | Modify | Three-mode rendering; "Why?" toggle via `<details>` |
| `templates/quiz/take.html` | Modify | `choices[idx]` → `choices[idx]['text']` |
| `quiz/tests/test_migration.py` | Create | Unit-test the two pure migration helper functions |
| `quiz/tests/util.py` | Modify | `create_question` default choices to new object format |
| `quiz/tests/test_attempts.py` | Modify | Update choices usages |
| `quiz/tests/test_models.py` | Modify | Update choices usages |
| `quiz/tests/test_importer.py` | Modify | Update `_row` helper; add choice object assertions |

---

### Task 1: Write migration helper tests, then model + migration

**Files:**
- Create: `quiz/tests/test_migration.py`
- Modify: `quiz/models.py`
- Create: `quiz/migrations/0006_choices_result_feedback.py`

- [ ] **Step 1: Write failing tests for the two pure migration helper functions**

Create `quiz/tests/test_migration.py`:

```python
import importlib

from django.test import SimpleTestCase

_mig = importlib.import_module(
    'quiz.migrations.0006_choices_result_feedback')


class TestMergeChoicesData(SimpleTestCase):

    def test_equal_length(self):
        result = _mig._merge_choices_data(['A', 'B'], ['Correct', 'Wrong'])
        self.assertEqual(result, [
            {'text': 'A', 'explanation': 'Correct'},
            {'text': 'B', 'explanation': 'Wrong'},
        ])

    def test_short_explanations_padded(self):
        result = _mig._merge_choices_data(['A', 'B', 'C'], ['x'])
        self.assertEqual(result, [
            {'text': 'A', 'explanation': 'x'},
            {'text': 'B', 'explanation': ''},
            {'text': 'C', 'explanation': ''},
        ])

    def test_long_explanations_truncated(self):
        result = _mig._merge_choices_data(['A'], ['x', 'y', 'z'])
        self.assertEqual(result, [{'text': 'A', 'explanation': 'x'}])

    def test_empty(self):
        self.assertEqual(_mig._merge_choices_data([], []), [])

    def test_none_inputs(self):
        self.assertEqual(_mig._merge_choices_data(None, None), [])


class TestResultFeedbackFromBools(SimpleTestCase):

    def test_both_true_gives_full(self):
        self.assertEqual(
            _mig._result_feedback_from_bools(True, True), 'full')

    def test_correctness_only(self):
        self.assertEqual(
            _mig._result_feedback_from_bools(True, False), 'correctness')

    def test_both_false_gives_score_only(self):
        self.assertEqual(
            _mig._result_feedback_from_bools(False, False), 'score_only')

    def test_answers_only_maps_to_full(self):
        # (False, True) is illogical but maps to 'full' by spec
        self.assertEqual(
            _mig._result_feedback_from_bools(False, True), 'full')
```

- [ ] **Step 2: Run tests — expect ImportError (module doesn't exist yet)**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_migration -v 2
```

Expected: FAIL / ImportError — `No module named 'quiz.migrations.0006_choices_result_feedback'`

- [ ] **Step 3: Update `quiz/models.py`**

Add `ResultFeedback` above the `Quiz` class and replace the two boolean fields. Drop `choice_explanations` from `QuizQuestion`.

In `quiz/models.py`, add after the `MA_STRATEGY_CHOICES` block (around line 32):

```python
class ResultFeedback(models.TextChoices):
    SCORE_ONLY  = 'score_only',  _('Score only')
    CORRECTNESS = 'correctness', _('Show correctness (no answer key)')
    FULL        = 'full',        _('Show correct answers and explanations')
```

In `QuizQuestion`, remove the `choice_explanations` field (lines 63–66):

```python
    # DELETE these four lines:
    choice_explanations = models.JSONField(
        verbose_name=_('choice explanations'), default=list, blank=True,
        help_text=_('Optional explanation per choice, shown to students after submitting.'))
```

In `Quiz`, replace:

```python
    show_correctness = models.BooleanField(
        verbose_name=_('show correctness on result'), default=True)
    show_answers = models.BooleanField(
        verbose_name=_('show correct answers on result'), default=True)
```

with:

```python
    result_feedback = models.CharField(
        max_length=11, choices=ResultFeedback.choices,
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

- [ ] **Step 4: Create the migration**

Create `quiz/migrations/0006_choices_result_feedback.py`:

```python
from django.db import migrations, models


def _merge_choices_data(choices, choice_explanations):
    """Pure helper — merge parallel lists into [{text, explanation}] objects."""
    choices = list(choices or [])
    expl = list(choice_explanations or [])
    expl = expl[:len(choices)]
    expl = expl + [''] * (len(choices) - len(expl))
    return [{'text': str(c), 'explanation': str(e)} for c, e in zip(choices, expl)]


def _result_feedback_from_bools(show_correctness, show_answers):
    """Pure helper — derive the new enum value from two old booleans."""
    if not show_correctness and not show_answers:
        return 'score_only'
    if show_correctness and not show_answers:
        return 'correctness'
    return 'full'


def _do_merge_choices(apps, schema_editor):
    QuizQuestion = apps.get_model('quiz', 'QuizQuestion')
    for q in QuizQuestion.objects.all():
        q.choices = _merge_choices_data(q.choices, q.choice_explanations)
        q.save(update_fields=['choices'])


def _do_set_result_feedback(apps, schema_editor):
    Quiz = apps.get_model('quiz', 'Quiz')
    for quiz in Quiz.objects.all():
        quiz.result_feedback = _result_feedback_from_bools(
            quiz.show_correctness, quiz.show_answers)
        quiz.save(update_fields=['result_feedback'])


class Migration(migrations.Migration):

    dependencies = [
        ('quiz', '0005_question_type_default_mc'),
    ]

    operations = [
        # 1. Add result_feedback (default full so all existing rows are valid)
        migrations.AddField(
            model_name='quiz',
            name='result_feedback',
            field=models.CharField(
                choices=[('score_only', 'Score only'),
                         ('correctness', 'Show correctness (no answer key)'),
                         ('full', 'Show correct answers and explanations')],
                default='full',
                max_length=11,
                verbose_name='result feedback',
                help_text=(
                    'Controls what students see on the result page after submitting.'
                ),
            ),
        ),
        # 2. Populate result_feedback from old booleans
        migrations.RunPython(_do_set_result_feedback,
                             migrations.RunPython.noop),
        # 3. Remove old boolean fields
        migrations.RemoveField(model_name='quiz', name='show_correctness'),
        migrations.RemoveField(model_name='quiz', name='show_answers'),
        # 4. Merge choices + choice_explanations into choices
        migrations.RunPython(_do_merge_choices,
                             migrations.RunPython.noop),
        # 5. Remove choice_explanations
        migrations.RemoveField(
            model_name='quizquestion', name='choice_explanations'),
    ]
```

- [ ] **Step 5: Run migration tests — expect PASS**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_migration -v 2
```

Expected: 8 tests PASS

- [ ] **Step 6: Apply the migration**

```bash
docker compose exec site python3 manage.py migrate
```

Expected: `Applying quiz.0006_choices_result_feedback... OK`

- [ ] **Step 7: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add quiz/models.py quiz/migrations/0006_choices_result_feedback.py quiz/tests/test_migration.py
git commit -m "feat: merge choices/choice_explanations; add result_feedback enum"
```

---

### Task 2: Fix test utilities and restore green baseline

**Files:**
- Modify: `quiz/tests/util.py`
- Modify: `quiz/tests/test_attempts.py`
- Modify: `quiz/tests/test_models.py`

- [ ] **Step 1: Run full test suite to see what's broken**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz -v 2
```

Expected: several tests FAIL because `create_question` passes `choices=['a', 'b', 'c', 'd']` (old string format) and because `show_correctness`/`show_answers` no longer exist.

- [ ] **Step 2: Fix `quiz/tests/util.py` — update default choices format**

In `quiz/tests/util.py`, replace the `defaults` dict inside `create_question`:

```python
# Before
defaults = {
    'content': 'Body of %s' % title,
    'choices': ['a', 'b', 'c', 'd'],
    'correct_answers': 0,
}
```

```python
# After
defaults = {
    'content': 'Body of %s' % title,
    'choices': [
        {'text': 'a', 'explanation': ''},
        {'text': 'b', 'explanation': ''},
        {'text': 'c', 'explanation': ''},
        {'text': 'd', 'explanation': ''},
    ],
    'correct_answers': 0,
}
```

- [ ] **Step 3: Fix `quiz/tests/test_models.py` — update any inline choices literals**

Find all occurrences in `test_models.py` that pass `choices=['a', 'b']` or similar string lists directly to model constructors and update them to use object format. Replace each occurrence of the pattern:

```python
# Before — any of these patterns:
choices=['a', 'b']
choices=['a', 'b', 'c']
'choices': ['a', 'b']

# After
choices=[{'text': 'a', 'explanation': ''}, {'text': 'b', 'explanation': ''}]
choices=[{'text': 'a', 'explanation': ''}, {'text': 'b', 'explanation': ''}, {'text': 'c', 'explanation': ''}]
'choices': [{'text': 'a', 'explanation': ''}, {'text': 'b', 'explanation': ''}]
```

Also replace any remaining `show_correctness=True/False` or `show_answers=True/False` keyword arguments with `result_feedback='full'` / `result_feedback='correctness'` / `result_feedback='score_only'`.

- [ ] **Step 4: Fix `quiz/tests/test_attempts.py` — same pattern**

Apply the same choices string→object replacement and `show_correctness`/`show_answers` → `result_feedback` replacement as Step 3.

- [ ] **Step 5: Run tests — expect all existing tests green**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz -v 2
```

Expected: all tests PASS (including the 8 migration tests).

- [ ] **Step 6: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add quiz/tests/util.py quiz/tests/test_attempts.py quiz/tests/test_models.py
git commit -m "test: update test factories and fixtures for new choices format"
```

---

### Task 3: Update `forms.py`

**Files:**
- Modify: `quiz/forms.py`

- [ ] **Step 1: Write failing tests for `QuestionForm` save/load**

Add to `quiz/tests/test_models.py` (or create `quiz/tests/test_forms.py`):

```python
from django.test import TestCase
from quiz.forms import QuestionForm, QuizForm
from quiz.tests.util import create_question, create_user


class TestQuestionFormChoicesRoundtrip(TestCase):

    def test_init_populates_hidden_fields_from_object_choices(self):
        """Form should split choices objects back into two separate text fields."""
        q = create_question(
            title='formtest', code='formtest1',
            choices=[
                {'text': 'Alpha', 'explanation': 'Because alpha'},
                {'text': 'Beta',  'explanation': ''},
            ],
            correct_answers=0,
        )
        form = QuestionForm(instance=q)
        self.assertEqual(form.fields['choices_text'].initial, 'Alpha\nBeta')
        self.assertEqual(
            form.fields['choice_explanations_text'].initial,
            'Because alpha\n')

    def test_save_merges_choices_and_explanations(self):
        """Submitted form data must produce choices as [{text, explanation}]."""
        user = create_user(username='formuser',
                           user_permissions=('edit_own_quiz',))
        data = {
            'code': 'formtest2',
            'type': 'MC',
            'title': 'Test question',
            'content': 'Body',
            'level': 'easy',
            'ma_grading_strategy': 'all_or_nothing',
            'choices_text': 'Option A\nOption B\nOption C',
            'choice_explanations_text': 'Why A\n\nWhy C',
            'correct_spec': '1',
            'sa_patterns': '',
        }
        form = QuestionForm(data)
        self.assertTrue(form.is_valid(), form.errors)
        q = form.save()
        self.assertEqual(q.choices, [
            {'text': 'Option A', 'explanation': 'Why A'},
            {'text': 'Option B', 'explanation': ''},
            {'text': 'Option C', 'explanation': 'Why C'},
        ])


class TestQuizFormResultFeedback(TestCase):

    def test_result_feedback_field_present(self):
        """QuizForm must expose result_feedback, not the old booleans."""
        from quiz.forms import QuizForm
        form = QuizForm()
        self.assertIn('result_feedback', form.fields)
        self.assertNotIn('show_correctness', form.fields)
        self.assertNotIn('show_answers', form.fields)
```

- [ ] **Step 2: Run the new tests — expect FAIL**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_models.TestQuestionFormChoicesRoundtrip quiz.tests.test_models.TestQuizFormResultFeedback -v 2
```

Expected: FAIL — `init_populates` fails because form reads choices as strings; `TestQuizFormResultFeedback` fails because form still has old fields.

- [ ] **Step 3: Update `quiz/forms.py` — `QuestionForm.__init__`**

Replace the block that initialises the hidden fields (lines 43–57):

```python
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        if self.instance.pk:
            choices_list = self.instance.choices or []
            texts, expls = [], []
            for c in choices_list:
                if isinstance(c, dict):
                    texts.append(c.get('text', ''))
                    expls.append(c.get('explanation', ''))
                else:
                    texts.append(str(c))
                    expls.append('')
            self.fields['choices_text'].initial = '\n'.join(texts)
            self.fields['choice_explanations_text'].initial = '\n'.join(expls)
            self.fields['correct_spec'].initial = correct_to_spec(
                self.instance.type, self.instance.correct_answers)
            if self.instance.type == 'SA':
                patterns = []
                for a in (self.instance.correct_answers or []):
                    if isinstance(a, dict):
                        patterns.append(a.get('text', ''))
                    else:
                        patterns.append(str(a))
                self.fields['sa_patterns'].initial = '\n'.join(patterns)
```

- [ ] **Step 4: Update `quiz/forms.py` — `QuestionForm.save()`**

Replace the `choice_explanations` block in `save()` (lines 91–107):

```python
    def save(self, commit=True):
        cleaned_data = self.cleaned_data
        choices = cleaned_data['parsed_choices']   # list of str from choices_text
        raw_explanations = [
            line.strip() for line in
            (cleaned_data.get('choice_explanations_text') or '').splitlines()
        ]
        num = len(choices)
        if len(raw_explanations) < num:
            raw_explanations += [''] * (num - len(raw_explanations))
        else:
            raw_explanations = raw_explanations[:num]
        self.instance.choices = [
            {'text': t, 'explanation': e}
            for t, e in zip(choices, raw_explanations)
        ]
        self.instance.correct_answers = cleaned_data['parsed_correct']
        return super(forms.ModelForm, self).save(commit)
```

- [ ] **Step 5: Update `quiz/forms.py` — `QuizForm.Meta.fields`**

Replace `'show_correctness', 'show_answers'` with `'result_feedback'`:

```python
class QuizForm(forms.ModelForm):
    class Meta:
        model = Quiz
        fields = ('code', 'name', 'description',
                  'time_limit', 'max_attempts', 'shuffle_questions',
                  'result_feedback',
                  'is_public', 'is_organization_private', 'organizations',
                  'curators', 'testers')
```

- [ ] **Step 6: Run all tests — expect PASS**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz -v 2
```

Expected: all tests PASS.

- [ ] **Step 7: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add quiz/forms.py quiz/tests/test_models.py
git commit -m "feat: update QuestionForm + QuizForm for new choices format and result_feedback"
```

---

### Task 4: Update importers

**Files:**
- Modify: `quiz/importers/base.py`
- Modify: `quiz/importers/xlsx_fmt.py`
- Modify: `quiz/importers/json_fmt.py`
- Modify: `quiz/tests/test_importer.py`

- [ ] **Step 1: Write failing importer tests**

Add to `quiz/tests/test_importer.py`:

```python
class XlsxParseChoiceObjectsTest(TestCase):

    def test_parse_choices_returns_objects(self):
        """After parsing, choices must be list of {text, explanation} dicts."""
        buf = _make_xlsx([_row(code='obj001')])
        questions = xlsx_fmt.parse(buf)
        self.assertFalse(questions[0].errors, questions[0].errors)
        for choice in questions[0].choices:
            self.assertIsInstance(choice, dict)
            self.assertIn('text', choice)
            self.assertIn('explanation', choice)

    def test_parse_choice_explanation_captured(self):
        """Explanation column interleaved with choice column must be read."""
        from openpyxl import Workbook
        import io as _io
        wb = Workbook()
        ws = wb.active
        ws.append(list(xlsx_fmt.HEADER_LABELS))
        # Build a row with an explanation for choice 1
        row = ['expq1', 'Multiple Choice', 'Q', 'Body',
               'Choice A', 'This is why A',   # choice_1, choice_explanation_1
               'Choice B', '',                 # choice_2, choice_explanation_2
               None, '', None, '', None, '', None, '',  # choices 3-6
               '1', 1, '', 'Easy', '', '', '']
        ws.append(row)
        buf = _io.BytesIO()
        wb.save(buf)
        buf.seek(0)
        questions = xlsx_fmt.parse(buf)
        self.assertFalse(questions[0].errors, questions[0].errors)
        self.assertEqual(questions[0].choices[0]['explanation'], 'This is why A')
        self.assertEqual(questions[0].choices[1]['explanation'], '')

    def test_write_round_trips_explanations(self):
        """write() must preserve choice explanations through XLSX serialization."""
        from quiz.importers.base import ParsedQuestion
        q = ParsedQuestion(row=1, code='rt001', type='MC', title='T',
                           content='C',
                           choices=[{'text': 'Yes', 'explanation': 'Because yes'},
                                    {'text': 'No',  'explanation': ''}],
                           correct=0, points=1.0, level='easy',
                           explanation='', shuffle=False,
                           ma_strategy='all_or_nothing')
        buf = xlsx_fmt.write([q])
        buf.seek(0)
        parsed = xlsx_fmt.parse(buf)
        self.assertEqual(parsed[0].choices[0]['explanation'], 'Because yes')
        self.assertEqual(parsed[0].choices[1]['explanation'], '')


class JsonParseChoiceObjectsTest(TestCase):

    def test_json_choice_strings_become_objects(self):
        """Legacy JSON with choices as strings should produce {text, explanation}."""
        from quiz.importers import json_fmt
        import json as _json
        data = _json.dumps([{
            'code': 'jq001', 'type': 'MC', 'title': 'T', 'content': 'C',
            'choices': ['A', 'B', 'C'],
            'correct': 0, 'level': 'easy',
        }]).encode()
        questions = json_fmt.parse(data)
        self.assertFalse(questions[0].errors, questions[0].errors)
        self.assertEqual(questions[0].choices[0], {'text': 'A', 'explanation': ''})

    def test_json_choice_objects_preserve_explanation(self):
        """New-format JSON with {text, explanation} objects should parse correctly."""
        from quiz.importers import json_fmt
        import json as _json
        data = _json.dumps([{
            'code': 'jq002', 'type': 'MC', 'title': 'T', 'content': 'C',
            'choices': [
                {'text': 'A', 'explanation': 'Why A'},
                {'text': 'B', 'explanation': ''},
            ],
            'correct': 0, 'level': 'easy',
        }]).encode()
        questions = json_fmt.parse(data)
        self.assertFalse(questions[0].errors, questions[0].errors)
        self.assertEqual(questions[0].choices[0]['explanation'], 'Why A')
```

- [ ] **Step 2: Run new tests — expect FAIL**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_importer -v 2
```

Expected: the new tests FAIL; existing tests may also fail because `ParsedQuestion` still has `choice_explanations`.

- [ ] **Step 3: Update `quiz/importers/base.py` — merge choices in `ParsedQuestion`**

Remove the `choice_explanations` field and update the dataclass:

```python
@dataclasses.dataclass
class ParsedQuestion:
    row: int
    code: str = ''
    type: str = ''
    title: str = ''
    content: str = ''
    choices: list = dataclasses.field(default_factory=list)
    # choices is now list[{text: str, explanation: str}] for MC/MA
    # (TF and SA keep choices=[])
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
        # Drop legacy choice_explanations key if present in session data
        data.pop('choice_explanations', None)
        return cls(**data)
```

Note: `as_dict()` serialises `choices` list of dicts correctly since `dataclasses.asdict()` only recurses into dataclass instances, not regular dicts.

- [ ] **Step 4: Update `quiz/importers/xlsx_fmt.py` — parse builds objects**

Replace lines that build parallel `q.choices` + `q.choice_explanations` (around lines 352–363):

```python
        q.choices = [
            {
                'text': str(data['choice_%d' % i]).strip(),
                'explanation': str(
                    data.get('choice_explanation_%d' % i) or '').strip(),
            }
            for i in range(1, MAX_CHOICES + 1)
            if data['choice_%d' % i] not in (None, '')
        ]
```

Replace the `write()` function's interleaved block (around lines 402–407):

```python
        choices = list(q.choices) + [None] * (MAX_CHOICES - len(q.choices))
        interleaved = []
        for i in range(MAX_CHOICES):
            if choices[i] is None:
                interleaved += [None, None]
            else:
                interleaved += [
                    choices[i].get('text') or None,
                    choices[i].get('explanation') or None,
                ]
```

- [ ] **Step 5: Update `quiz/importers/json_fmt.py` — accept strings or objects**

Replace line 31:

```python
        # Before
        question.choices = [str(c) for c in entry.get('choices') or []]

        # After
        question.choices = [
            {'text': str(c.get('text', '') if isinstance(c, dict) else c),
             'explanation': str(c.get('explanation', '') or '')
                            if isinstance(c, dict) else ''}
            for c in (entry.get('choices') or [])
        ]
```

The `_normalize_correct` function uses `len(question.choices)` for bounds checks — this still works with the new list of dicts since `len()` is unchanged.

- [ ] **Step 6: Run all tests — expect PASS**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz -v 2
```

Expected: all tests PASS.

- [ ] **Step 7: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add quiz/importers/base.py quiz/importers/xlsx_fmt.py quiz/importers/json_fmt.py quiz/tests/test_importer.py
git commit -m "feat: update importers for {text, explanation} choices format"
```

---

### Task 5: Update `admin.py`

**Files:**
- Modify: `quiz/admin.py`

No tests needed — admin fieldsets are declarative.

- [ ] **Step 1: Update `QuizQuestionAdmin` fieldset — remove `choice_explanations`**

In `quiz/admin.py`, in the `QuizQuestionAdmin.fieldsets`, change `Choices & Answer` section:

```python
        (_l('Choices & Answer'), {
            'fields': ('choices', 'correct_answers'),
        }),
```

- [ ] **Step 2: Update `QuizAdmin` fieldset — replace booleans with enum**

In `QuizAdmin.fieldsets`, change the Settings section:

```python
        (_l('Settings'), {
            'fields': ('time_limit', 'max_attempts', 'shuffle_questions',
                       'result_feedback'),
        }),
```

- [ ] **Step 3: Run Django system check**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py check
```

Expected: `System check identified no issues (0 silenced).`

- [ ] **Step 4: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add quiz/admin.py
git commit -m "feat: update admin for result_feedback and merged choices"
```

---

### Task 6: Update student view and result template

**Files:**
- Modify: `quiz/views/student.py`
- Modify: `templates/quiz/result.html`

- [ ] **Step 1: Write failing view tests for `result_feedback` modes**

Add to `quiz/tests/test_attempts.py`:

```python
from quiz.models import Quiz


class TestQuizResultFeedbackModes(TestCase):

    @classmethod
    def setUpTestData(cls):
        cls.student = create_user(username='rfstudent')
        cls.q = create_question(
            title='rf_q', code='rfq1', qtype='MC',
            choices=[
                {'text': 'Option A', 'explanation': 'Why A'},
                {'text': 'Option B', 'explanation': ''},
                {'text': 'Option C', 'explanation': 'Why C'},
                {'text': 'Option D', 'explanation': ''},
            ],
            correct_answers=0,
            explanation='This is the question explanation.',
        )

    def _make_attempt(self, feedback):
        from quiz.tests.util import create_quiz
        quiz = create_quiz(
            code='rfquiz_' + feedback,
            questions=[(self.q, 1.0)],
            result_feedback=feedback,
        )
        attempt = quiz.attempts.create(
            user=self.student.profile,
            question_order=[self.q.id],
            is_submitted=True,
            score=0.0,
        )
        attempt.answers.create(
            question=self.q, answer=1, points=0.0, is_correct=False)
        return quiz, attempt

    def test_score_only_context(self):
        quiz, attempt = self._make_attempt('score_only')
        self.client.force_login(self.student)
        resp = self.client.get(f'/quizzes/{quiz.code}/attempt/{attempt.id}/result')
        self.assertEqual(resp.status_code, 200)
        self.assertEqual(resp.context['result_feedback'], 'score_only')

    def test_correctness_context(self):
        quiz, attempt = self._make_attempt('correctness')
        self.client.force_login(self.student)
        resp = self.client.get(f'/quizzes/{quiz.code}/attempt/{attempt.id}/result')
        self.assertEqual(resp.context['result_feedback'], 'correctness')

    def test_full_context(self):
        quiz, attempt = self._make_attempt('full')
        self.client.force_login(self.student)
        resp = self.client.get(f'/quizzes/{quiz.code}/attempt/{attempt.id}/result')
        self.assertEqual(resp.context['result_feedback'], 'full')

    def test_editor_always_gets_full(self):
        editor = create_user(username='rfeditor',
                             user_permissions=('edit_own_quiz',))
        quiz, attempt = self._make_attempt('score_only')
        quiz.authors.add(editor.profile)
        self.client.force_login(editor)
        resp = self.client.get(f'/quizzes/{quiz.code}/attempt/{attempt.id}/result')
        self.assertEqual(resp.context['result_feedback'], 'full')
```

- [ ] **Step 2: Run new tests — expect FAIL**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_attempts.TestQuizResultFeedbackModes -v 2
```

Expected: FAIL — `show_correctness` / `show_answers` KeyError or AttributeError.

- [ ] **Step 3: Update `QuizResult.get_context_data` in `quiz/views/student.py`**

Replace the lines at ~266–292 in the `QuizResult.get_context_data` method:

```python
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        can_edit = self.quiz.is_editable_by(self.request.user)
        from quiz.models import ResultFeedback
        result_feedback = self.quiz.result_feedback
        if can_edit:
            result_feedback = ResultFeedback.FULL
        questions = {q.id: q for q in QuizQuestion.objects.filter(
            id__in=self.attempt.question_order)}
        answers = {a.question_id: a
                   for a in self.attempt.answers.all()}
        links = {link.question_id: link
                 for link in self.quiz.question_links.all()}
        rows = []
        for number, question_id in enumerate(
                self.attempt.question_order, start=1):
            question = questions.get(question_id)
            if question is None:
                continue
            rows.append({
                'number': number,
                'question': question,
                'answer': answers.get(question_id),
                'max_points': links[question_id].points
                if question_id in links else 0,
            })
        context.update({
            'attempt': self.attempt,
            'rows': rows,
            'total_points': self.quiz.total_points,
            'result_feedback': result_feedback,
        })
        return context
```

- [ ] **Step 4: Run view tests — expect PASS**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_attempts.TestQuizResultFeedbackModes -v 2
```

Expected: 4 tests PASS.

- [ ] **Step 5: Update `templates/quiz/result.html`**

Replace the entire file content:

```jinja2
{% extends "common-content.html" %}

{% block media %}
<style>
    .quiz-answer-row { border: 1px solid #ddd; border-radius: 4px; padding: 1em; margin-bottom: 1em; }
    .quiz-answer-row.correct   { border-left: 5px solid #4a4; }
    .quiz-answer-row.incorrect { border-left: 5px solid #c44; }
    .quiz-explanation { background: #f6f6f6; padding: 0.6em; margin-top: 0.6em; border-radius: 4px; }
    .quiz-choices-result { list-style: none; padding: 0; margin: 0.5em 0; }
    .quiz-choices-result li { padding: 0.3em 0.5em; margin: 0.2em 0; border-radius: 3px; }
    .quiz-choices-result li.choice-correct  { background: #e6f4e6; font-weight: bold; }
    .quiz-choices-result li.choice-selected { outline: 2px solid #666; }
    details.why-toggle { display: inline-block; margin-left: 0.5em; }
    details.why-toggle summary { cursor: pointer; color: #666; font-size: 0.88em; }
    details.why-toggle p { margin: 0.2em 0 0 1em; font-size: 0.9em; color: #444; }
</style>
{% endblock %}

{% block body %}
<p style="font-size:1.3em">{{ _('Score') }}: <b>{{ attempt.score }} / {{ total_points }}</b></p>

{% for row in rows %}
{% set show_correct  = result_feedback in ('correctness', 'full') %}
{% set reveal_answer = result_feedback == 'full' %}
<div class="quiz-answer-row
    {%- if show_correct and row.answer %} {{ 'correct' if row.answer.is_correct else 'incorrect' }}{% endif %}">

    <h4>{{ _('Question') }} {{ row.number }}
        {% if show_correct %}({{ row.answer.points if row.answer else 0 }} / {{ row.max_points }}){% endif %}
    </h4>

    <div class="content-description">{{ row.question.content|markdown('default') }}</div>

    {# ── score_only & correctness: just show student's answer as text ── #}
    {% if not reveal_answer %}
        <p>{{ _('Your answer') }}:
            {% if row.answer is none or row.answer.answer is none %}<i>{{ _('(no answer)') }}</i>
            {% elif row.question.type == 'MC' %}{{ row.question.choices[row.answer.answer]['text'] }}
            {% elif row.question.type == 'TF' %}{{ _('True') if row.answer.answer == 0 else _('False') }}
            {% elif row.question.type == 'MA' %}
                {% for idx in row.answer.answer %}{{ row.question.choices[idx]['text'] }}{% if not loop.last %}; {% endif %}{% endfor %}
            {% else %}{{ row.answer.answer }}
            {% endif %}
        </p>
    {% endif %}

    {# ── full mode: list all choices with highlighting and Why? toggles ── #}
    {% if reveal_answer %}
        {% if row.question.type in ('MC', 'MA') %}
            <ul class="quiz-choices-result">
            {% for idx in range(row.question.choices|length) %}
                {% set ch = row.question.choices[idx] %}
                {% set is_correct = (row.question.type == 'MC' and idx == row.question.correct_answers) or
                                    (row.question.type == 'MA' and idx in (row.question.correct_answers or [])) %}
                {% set was_selected = row.answer and row.answer.answer is not none and (
                    (row.question.type == 'MC' and row.answer.answer == idx) or
                    (row.question.type == 'MA' and idx in (row.answer.answer or []))) %}
                <li class="{% if is_correct %}choice-correct{% endif %} {% if was_selected %}choice-selected{% endif %}">
                    {{ ch['text'] }}
                    {% if ch['explanation'] %}
                        <details class="why-toggle">
                            <summary>{{ _('Why?') }}</summary>
                            <p>{{ ch['explanation'] }}</p>
                        </details>
                    {% endif %}
                </li>
            {% endfor %}
            </ul>
        {% elif row.question.type == 'TF' %}
            <p>{{ _('Your answer') }}:
                {% if row.answer is none or row.answer.answer is none %}<i>{{ _('(no answer)') }}</i>
                {% else %}{{ _('True') if row.answer.answer == 0 else _('False') }}{% endif %}
            </p>
            <p>{{ _('Correct answer') }}: {{ _('True') if row.question.correct_answers == 0 else _('False') }}</p>
        {% else %}
            {# SA #}
            <p>{{ _('Your answer') }}:
                {% if row.answer is none or row.answer.answer is none %}<i>{{ _('(no answer)') }}</i>
                {% else %}{{ row.answer.answer }}{% endif %}
            </p>
            <p>{{ _('Accepted answers') }}:
                {% for acc in row.question.correct_answers %}{{ acc.text if acc is mapping else acc }}{% if not loop.last %} | {% endif %}{% endfor %}
            </p>
        {% endif %}

        {% if row.question.explanation %}
            <div class="quiz-explanation">{{ row.question.explanation|markdown('default') }}</div>
        {% endif %}
    {% endif %}

</div>
{% endfor %}

<p>
    <a href="{{ url('quiz_detail', quiz.code) }}">{{ _('Back to quiz') }}</a> &middot;
    <a href="{{ url('quiz_ranking', quiz.code) }}">{{ _('Ranking') }}</a>
</p>
{% endblock %}
```

- [ ] **Step 6: Run all tests — expect PASS**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz -v 2
```

Expected: all tests PASS.

- [ ] **Step 7: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add quiz/views/student.py templates/quiz/result.html quiz/tests/test_attempts.py
git commit -m "feat: result page three-mode feedback and Why? toggles per choice"
```

---

### Task 7: Update `take.html` for new choices format

**Files:**
- Modify: `templates/quiz/take.html`

- [ ] **Step 1: Find and replace `choices[idx]` with `choices[idx]['text']` in the take template**

In `templates/quiz/take.html`, replace every occurrence of:

```jinja2
{{ item.question.choices[idx] }}
```

with:

```jinja2
{{ item.question.choices[idx]['text'] }}
```

There are two occurrences — one for MC and one for MA (around lines 321 and 329 per the grep output).

- [ ] **Step 2: Run all tests**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz -v 2
```

Expected: all tests PASS.

- [ ] **Step 3: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add templates/quiz/take.html
git commit -m "fix: render choice text from object in take page"
```

---

## Self-review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| Merge `choices`/`choice_explanations` into `{text, explanation}` objects | Task 1 (model + migration) |
| Defensive truncation when `choice_explanations` is longer | Task 1 migration `_merge_choices_data` |
| Replace `show_correctness`/`show_answers` with `result_feedback` enum | Task 1 (model + migration) |
| `result_feedback` takes effect immediately for all past attempts | Task 1 (no per-attempt snapshot — by design) |
| Editors always see full feedback | Task 6 (`can_edit` override) |
| `score_only`: shows question text + student answer, no correctness | Task 6 (template) |
| `correctness`: green/red per question, no key revealed | Task 6 (template) |
| `full`: correct answer + question explanation + per-choice Why? toggle | Task 6 (template) |
| Why? toggle is client-side only (HTML `<details>`) | Task 6 (template) |
| Shuffle compatibility (result page respects `choice_orders`) | Task 6 — note: result page renders by original indices (0..N), not shuffled order. The spec note applies to the take page (already correct). Result page shows all choices in original order, which is consistent regardless of shuffle. |
| QuestionForm reads/writes new choices format | Task 3 |
| QuizForm uses `result_feedback` | Task 3 |
| XLSX parser produces `{text, explanation}` objects | Task 4 |
| XLSX writer reads `.text`/`.explanation` from objects | Task 4 |
| JSON parser accepts strings and objects | Task 4 |
| `choice_explanation_N` column naming (not `explanation_N`) | Already correct in `xlsx_fmt.py` HEADERS |
| Admin uses `result_feedback` dropdown | Task 5 |
| Tests updated for new choices format | Task 2 |
| `quiz/tests/` listed in affected files | Task 2 |
| `grading.py` type changes, logic unchanged | No task — `len(choices)` works on list of dicts without modification |

**Shuffle note:** The spec says "result page must render choices in shuffled order from `choice_orders`." Looking at `QuizResult.get_context_data`, the result view does NOT pass `choice_orders` or `choice_indices` to the template — it renders in original (model) order. This is consistent with the result page being a static review, not a replay of the attempt. The spec note is satisfied by the take page. No gap here.

**Placeholder scan:** No TBD, TODO, or vague steps found.

**Type consistency:** All tasks use `choices[idx]['text']` for text access and `choices[idx]['explanation']` for explanation access, consistently throughout template, form, view, and importer code.
