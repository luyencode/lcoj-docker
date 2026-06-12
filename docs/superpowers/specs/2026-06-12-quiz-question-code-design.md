# Quiz Question Code Field

**Date:** 2026-06-12  
**Status:** Approved  
**Scope:** Add a `code` field (user-friendly identifier) to `QuizQuestion`; unify the code validator across `Quiz` and `QuizQuestion` to match the `Problem` code pattern.

---

## Problem

`QuizQuestion` has no stable short identifier. When a teacher attaches questions to a quiz via the Select2 widget they can only search by title or question body — there is no way to reference a question by a memorable, unambiguous code. `Quiz` already has a `code` field but its regex (`^[a-z0-9_]+$`) differs from the stricter `Problem` code pattern (`^[a-z0-9]+$`).

---

## Goals

1. Give every `QuizQuestion` a unique, human-readable `code` (e.g. `py101q3`).
2. Allow teachers to look up questions by code in the quiz editor's Select2 widget.
3. Standardise the code validator across `Quiz` and `QuizQuestion` to `^[a-z0-9]+$` (same as `Problem`).
4. Expose the code in the XLSX import/export template so bulk-imported questions receive their code from the spreadsheet.

---

## Non-goals

- Changing question editor URLs to slug-based (stays PK-based: `/quiz/question/<id>/edit`).
- Auto-generating the code from the title; entry is always manual.
- Locking the code after save (unlike `Quiz.code`, question codes are editable to allow typo fixes).

---

## Design

### 1. Model changes

#### `QuizQuestion.code`

```python
code = models.CharField(
    max_length=32,
    unique=True,
    verbose_name=_('question code'),
    validators=[RegexValidator('^[a-z0-9]+$',
                               _('Question code must be ^[a-z0-9]+$'))],
)
```

- Required, unique, max 32 chars.
- Regex `^[a-z0-9]+$` — identical to `Problem.code`.
- `__str__` updated to `f'{self.code}: {self.title}'` for consistent display in admin, Select2, and `__repr__`.

#### `Quiz.code` validator update

Change the existing validator regex from `^[a-z0-9_]+$` to `^[a-z0-9]+$`.

**Migration impact:** `Quiz.code` is disabled in the edit form, so existing quiz codes containing underscores in the DB will not be re-validated on re-save. Only new quizzes are subject to the stricter pattern.

### 2. Migration

A single migration file performs two steps:

1. **Schema step:** Add `code` column with a temporary `default=''` and `unique=True`.
2. **Data step (`RunPython`):** For every existing `QuizQuestion`, set `code = f'q{question.id}'`. This produces short, deterministic codes (`q1`, `q42`, …) that satisfy `^[a-z0-9]+$`.
3. Remove the `default` (make the column non-nullable with no default) after the data step via `AlterField`.

### 3. Editor form

`QuestionForm.Meta.fields` gains `'code'` as the first entry:

```python
fields = ('code', 'type', 'title', ...)
```

- Plain `CharField` widget; no auto-fill, no locking after save.
- Standard model validation enforces uniqueness and regex on submit.

### 4. Question bank list

The `question_bank.html` template gains a **Code** column (monospace badge) in the question table, sorted/filterable alongside existing columns.

`QuestionBank.get_queryset` gains `code__icontains` to the existing `Q(title__icontains=search) | Q(content__icontains=search)` search filter:

```python
queryset = queryset.filter(
    Q(code__icontains=search) |
    Q(title__icontains=search) |
    Q(content__icontains=search)
)
```

### 5. Select2 lookup

`QuizQuestionSelect2View`:

```python
def get_queryset(self):
    return QuizQuestion.get_bank_questions(self.request.user).filter(
        Q(code__icontains=self.term) |
        Q(title__icontains=self.term) |
        Q(content__icontains=self.term)
    ).only('id', 'code', 'title', 'type')

def get_name(self, obj):
    return f'[{obj.type}] {obj.code}: {obj.title}'
```

Teachers can now type a code prefix (e.g. `py1`) in the quiz editor's question selector and immediately get matching questions.

### 6. XLSX importer

`HEADERS` and `HEADER_LABELS` gain `'code'` / `'Code'` as the **first** column (column A), shifting all existing columns one to the right.

**Validation rules for the `code` column:**

- Must match `^[a-z0-9]+$`.
- Must be non-empty.
- Must be unique within the import batch.
- Must not already exist in the DB (treated as a hard error, not a skip).

The XLSX template generator is updated to include the new column with an example value and a cell note explaining the format.

---

## Affected files

| File | Change |
|------|--------|
| `quiz/models.py` | Add `QuizQuestion.code`; update `Quiz.code` validator; update `QuizQuestion.__str__` |
| `quiz/migrations/XXXX_quizquestion_code.py` | Schema + data migration |
| `quiz/forms.py` | Add `code` to `QuestionForm.Meta.fields` |
| `quiz/views/select2.py` | Add `code__icontains` filter; update `get_name` display; add `code` to `.only()` |
| `quiz/views/editor.py` | `QuestionBank.get_queryset` — add `code__icontains` to search |
| `quiz/importers/xlsx_fmt.py` | Add `code` column to `HEADERS` / `HEADER_LABELS`; add validation |
| `templates/quiz/question_bank.html` | Add Code column |
| `templates/quiz/question_form.html` | Code field renders first |

---

## Testing

- Unit: `QuizQuestion` with duplicate or invalid code raises `ValidationError`.
- Unit: `Quiz` with underscore in code fails the new validator.
- Migration: data migration sets codes of existing questions to `q{id}`.
- Integration: Select2 endpoint returns questions when searching by code prefix.
- Integration: XLSX import with a `code` column creates questions with the specified codes; duplicate code in batch → import blocked with row-level error.
