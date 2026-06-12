# Quiz Edit — Dynamic Question Links & Remove Quiz Category/Level

**Date:** 2026-06-12  
**Status:** Approved  
**Scope:** Two related changes to the quiz edit form: (1) remove `category` and `level` from the `Quiz` model (keep on `QuizQuestion` only); (2) replace the static `extra=3` question formset with a dynamic drag-and-drop question list, matching the contest problem-add UX.

---

## Goals

1. Simplify `Quiz` by removing fields that belong to questions, not quizzes.
2. Give editors a clean, dynamic "add/remove/reorder" question list instead of three pre-filled blank rows.
3. Match the familiar contest problem lookup UX: searchable Select2 (code + title), drag-to-reorder, per-row delete.

## Non-goals

- Changing question editor URLs or the question bank.
- Adding pagination or lazy loading to the question list within a quiz.
- Drag-and-drop ordering on the student-facing quiz take page.

---

## Part 1: Remove Quiz.category and Quiz.level

### Model (`quiz/models.py`)

Remove from `Quiz`:
- `category = models.ForeignKey(QuizCategory, ...)`
- `level = models.CharField(..., choices=QuizLevel.choices, ...)`

`QuizLevel` TextChoices and `QuizCategory` model are **kept** — both are still used by `QuizQuestion`.

### Migration

A single migration with two `RemoveField` operations drops both columns from `quiz_quiz`. No data migration needed (DROP COLUMN only).

### Cascade of changes

| File | Change |
|------|--------|
| `quiz/forms.py` | Remove `'category'` and `'level'` from `QuizForm.Meta.fields` |
| `quiz/views/student.py` | Remove `.select_related('category')`; remove category/level filter logic; remove `categories`, `levels`, `selected_category`, `selected_level` from context |
| `quiz/admin.py` | Remove `'category'` and `'level'` from `QuizAdmin.list_display`, `list_filter`, and fieldsets |
| `templates/quiz/list.html` | Remove category/level filter dropdowns; remove Category and Level columns; table becomes 2 columns: Quiz + Your best score (colspan drops 4 → 2) |
| `templates/quiz/detail.html` | Remove any category/level display if present |

---

## Part 2: Dynamic Question Links with Drag-and-Drop

### Formset backend (`quiz/forms.py`)

Change `extra=3` → `extra=0` in `QuizQuestionLinkFormSet`. No other backend change. The existing per-user queryset restriction in `QuizEdit.get_forms()` and `QuizEdit.post()` is unchanged.

### Template (`templates/quiz/quiz_form.html`)

Replace `{{ form.as_p() }}` with individually-rendered quiz metadata fields, giving control over layout and naturally excluding the removed `category`/`level` fields.

Replace the plain questions table with a styled "Questions" section:

```
┌─────────────────────────────────────────────────────────┐
│  Questions                                              │
├──┬──────────────────────────────────┬────────┬──────────┤
│⠿ │ [Select question…            ▾]  │ Pts    │    ✕    │
│⠿ │ [py101q1: Python loops       ▾]  │ 2.0    │    ✕    │
│⠿ │ [muttype1: Mutable types     ▾]  │ 1.0    │    ✕    │
├──┴──────────────────────────────────┴────────┴──────────┤
│  + Add question                                         │
└─────────────────────────────────────────────────────────┘
```

**Rows:** Each `<tr>` contains:
- Drag-handle cell (`.drag-handle`, cursor `grab`, text `⠿`)
- Select2 question picker — `HeavySelect2Widget(data_view='quiz_question_select2')`, searches by code + title, displays as `[TYPE] code: title`
- Points number input (`step=0.1`, `min=0`, default 1.0)
- Hidden `order` input (`.order-field`) — value managed by JS
- ✕ delete button + hidden `DELETE` checkbox (checked by JS, row hidden)

**Template row:** A `<tr class="sortable-template" style="display:none">` rendered from `formset.empty_form`. Django renders it with `__prefix__` in all field names and IDs. The Select2 element gets class `django-select2 django-select2-heavy` from `HeavySelect2Widget`, with `id` containing `__prefix__` — the on-load Select2 auto-init skips it because `django_select2.js` filters `[id*=__prefix__]`.

**Management form** hidden inputs are rendered before the `<tbody>`.

### JavaScript (inline in `{% block content_js_media %}`)

**1. Drag-to-reorder**
```javascript
$('#question-links-body').sortable({
    handle: '.drag-handle',
    items: 'tr:not(.sortable-template)',
    stop: updateOrders,
});
```
`updateOrders()` iterates visible rows in DOM order and writes their 1-based position into `.order-field`.

**2. Add question**
```javascript
$('#add-question-btn').on('click', function () {
    var total = parseInt($('#id_question_links-TOTAL_FORMS').val(), 10);
    var templateHtml = $('#question-links-body .sortable-template')[0].outerHTML
        .replace(/sortable-template/g, 'sortable-row')
        .replace(/__prefix__/g, String(total));
    var $newRow = $(templateHtml).show();
    $('#question-links-body .sortable-template').before($newRow);
    $('#id_question_links-TOTAL_FORMS').val(total + 1);
    $newRow.find('.django-select2').djangoSelect2({ dropdownAutoWidth: true });
    updateOrders();
});
```

**3. Delete**
```javascript
$(document).on('click', '.delete-question-btn', function () {
    var $row = $(this).closest('tr');
    $row.find('[name$="-DELETE"]').prop('checked', true);
    $row.hide();
    updateOrders();
});
```

**4. Initial order**  
Call `updateOrders()` once on page load so rows that already have an order value stay consistent with their DOM position.

**jQuery UI dependency:** Already loaded globally by the lcoj-site layout (used by problem editor, attachments, course editor) — no new asset required.

---

## Affected files

| File | Change |
|------|--------|
| `quiz/models.py` | Remove `Quiz.category` and `Quiz.level` |
| `quiz/migrations/0004_remove_quiz_category_level.py` | Drop two columns |
| `quiz/forms.py` | `QuizForm` loses `category`/`level`; `QuizQuestionLinkFormSet` `extra=0` |
| `quiz/views/student.py` | Remove category/level filter code |
| `quiz/admin.py` | Remove category/level from `QuizAdmin` |
| `templates/quiz/list.html` | Remove filters and columns |
| `templates/quiz/detail.html` | Remove category/level display if present |
| `templates/quiz/quiz_form.html` | Rewrite: individual field rendering + dynamic question section |

---

## Testing

- Migration: `quiz_quiz` table has no `category_id` or `level` column after running.
- Model: `Quiz()` no longer accepts `category` or `level` kwargs.
- Form: `QuizForm` renders without category/level fields; submitting with extra category/level POST data is silently ignored.
- Student list: page renders without category/level dropdowns or columns; no 500 error.
- Formset: creating a new quiz with 0 initial question rows saves correctly.
- Add question: clicking "+ Add question" inserts a new row; Select2 is functional (can search and select a question); saving the quiz creates the expected `QuizQuestionLink` rows.
- Delete question: clicking ✕ hides the row; after save, the link is removed from the DB.
- Drag reorder: dragging rows changes the `order` field values; after save, questions are returned in the new order.
- Existing quizzes: editing a quiz with existing question links shows correct rows in drag-sortable table.
