# Quiz Edit — Dynamic Question Links & Remove Quiz Category/Level Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove `category` and `level` from the `Quiz` model, then replace the static 3-row question formset with a dynamic drag-and-drop question list that matches the contest problem-add UX.

**Architecture:** Three sequential tasks — (1) model/migration foundation, (2) backend + template cascade cleanup, (3) quiz_form.html rewrite with jQuery UI Sortable and dynamic formset JS. Task 2 depends on Task 1; Task 3 depends on Task 2.

**Tech Stack:** Django 4.2, django-jinja templates, jQuery (globally loaded), jQuery UI Sortable (globally loaded), HeavySelect2Widget / django_select2.js

---

## File Map

| File | Change |
|------|--------|
| `quiz/models.py` | Remove `Quiz.category` FK and `Quiz.level` CharField |
| `quiz/migrations/0004_remove_quiz_category_level.py` | Two `RemoveField` operations |
| `quiz/forms.py` | Drop `category`/`level` from `QuizForm`; `extra=3→0`; add `HiddenInput` for `order` in `QuizQuestionLinkForm` |
| `quiz/views/student.py` | Remove category/level filter logic and context keys |
| `quiz/admin.py` | Remove category/level from `QuizAdmin.list_display`, `list_filter`, fieldsets |
| `templates/quiz/list.html` | Remove filter dropdowns and table columns |
| `templates/quiz/quiz_form.html` | Full rewrite: `form.as_p()` for metadata + dynamic question section with drag-and-drop |
| `quiz/tests/test_models.py` | Add model regression tests and quiz-edit integration tests |

---

## Task 1: Remove Quiz.category and Quiz.level from model

**Files:**
- Modify: `quiz/models.py`
- Create: `quiz/migrations/0004_remove_quiz_category_level.py`
- Modify: `quiz/tests/test_models.py`

- [ ] **Step 1.1: Write failing tests**

Add to `quiz/tests/test_models.py` (inside a new test class at the bottom of the file):

```python
class QuizNoCategoryLevelTest(TestCase):
    def test_quiz_has_no_category_field(self):
        self.assertFalse(hasattr(Quiz, 'category'))

    def test_quiz_has_no_level_field(self):
        self.assertFalse(hasattr(Quiz, 'level'))

    def test_create_quiz_without_category_level(self):
        quiz = Quiz.objects.create(code='testnocatlvl', name='Test')
        self.assertEqual(quiz.name, 'Test')
```

- [ ] **Step 1.2: Run tests to confirm they fail**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_models.QuizNoCategoryLevelTest -v 2
```

Expected: FAIL — `Quiz` still has `category` and `level` attributes.

- [ ] **Step 1.3: Remove `category` and `level` from `Quiz` in `quiz/models.py`**

Remove these two fields from the `Quiz` class (lines 145–150 in the current file):

```python
# DELETE these two fields from Quiz:
category = models.ForeignKey(
    QuizCategory, verbose_name=_('category'), null=True, blank=True,
    on_delete=models.SET_NULL, related_name='quizzes')
level = models.CharField(
    max_length=6, verbose_name=_('level'), choices=QuizLevel.choices,
    default=QuizLevel.EASY)
```

`QuizLevel` and `QuizCategory` stay in the file — they are still used by `QuizQuestion`.

- [ ] **Step 1.4: Create migration `quiz/migrations/0004_remove_quiz_category_level.py`**

```python
from django.db import migrations


class Migration(migrations.Migration):
    dependencies = [
        ('quiz', '0003_quizquestion_code'),
    ]

    operations = [
        migrations.RemoveField(model_name='quiz', name='category'),
        migrations.RemoveField(model_name='quiz', name='level'),
    ]
```

- [ ] **Step 1.5: Apply migration**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py migrate quiz
```

Expected: `Applying quiz.0004_remove_quiz_category_level... OK`

- [ ] **Step 1.6: Run model tests**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_models quiz.tests.test_attempts -v 2
```

Expected: All pass (including the 3 new tests).

- [ ] **Step 1.7: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add quiz/models.py quiz/migrations/0004_remove_quiz_category_level.py quiz/tests/test_models.py
git commit -m "feat(quiz): remove category and level from Quiz model"
```

---

## Task 2: Backend + template cascade cleanup

**Files:**
- Modify: `quiz/forms.py`
- Modify: `quiz/views/student.py`
- Modify: `quiz/admin.py`
- Modify: `templates/quiz/list.html`
- Modify: `quiz/tests/test_models.py`

- [ ] **Step 2.1: Write failing test for quiz list view**

Add to `quiz/tests/test_models.py`:

```python
from django.urls import reverse


class QuizListNoCategoryLevelTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.quiz = create_quiz(code='listtest1')

    def test_list_renders_without_category_level_filters(self):
        resp = self.client.get(reverse('quiz_list'))
        self.assertEqual(resp.status_code, 200)
        # Category and level filter dropdowns must be gone
        self.assertNotContains(resp, 'name="category"')
        self.assertNotContains(resp, 'name="level"')
        # Quiz name still appears
        self.assertContains(resp, 'listtest1')
```

- [ ] **Step 2.2: Run test to confirm failure**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_models.QuizListNoCategoryLevelTest -v 2
```

Expected: FAIL — page still renders category/level dropdowns (server error or assertion fail).

- [ ] **Step 2.3: Update `QuizForm` in `quiz/forms.py`**

Remove `'category'` and `'level'` from `QuizForm.Meta.fields`:

```python
class QuizForm(forms.ModelForm):
    class Meta:
        model = Quiz
        fields = ('code', 'name', 'description',
                  'time_limit', 'max_attempts', 'shuffle_questions',
                  'show_correctness', 'show_answers',
                  'is_public', 'is_organization_private', 'organizations',
                  'curators', 'testers')

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        if self.instance.pk:
            self.fields['code'].disabled = True
```

- [ ] **Step 2.4: Update `QuizQuestionLinkForm` and `QuizQuestionLinkFormSet` in `quiz/forms.py`**

Add `HiddenInput` for `order` in the form widgets, and change `extra=3` to `extra=0` on the formset:

```python
class QuizQuestionLinkForm(forms.ModelForm):
    class Meta:
        model = QuizQuestionLink
        fields = ('question', 'points', 'order')
        widgets = {
            'question': HeavySelect2Widget(
                data_view='quiz_question_select2',
                attrs={'style': 'width: 100%'},
            ),
            'order': forms.HiddenInput(attrs={'class': 'order-field'}),
        }


QuizQuestionLinkFormSet = inlineformset_factory(
    Quiz, QuizQuestionLink, form=QuizQuestionLinkForm,
    extra=0, can_delete=True)
```

- [ ] **Step 2.5: Update `QuizList` in `quiz/views/student.py`**

Replace the entire `QuizList` class with this (keep all other classes below it unchanged):

```python
class QuizList(ListView):
    model = Quiz
    template_name = 'quiz/list.html'
    context_object_name = 'quizzes'
    paginate_by = 50

    def get_queryset(self):
        queryset = Quiz.get_visible_quizzes(self.request.user) \
            .order_by('-created_at')
        search = self.request.GET.get('search')
        if search:
            queryset = queryset.filter(
                Q(name__icontains=search) | Q(code__icontains=search))
        return queryset

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['search'] = self.request.GET.get('search', '')
        best_scores = {}
        if self.request.user.is_authenticated:
            quiz_ids = [q.pk for q in context['quizzes']]
            rows = QuizAttempt.objects.filter(
                user=self.request.profile, is_submitted=True,
                quiz_id__in=quiz_ids,
            ).values('quiz_id').annotate(best=Max('score'))
            best_scores = {row['quiz_id']: row['best'] for row in rows}
        context['best_scores'] = best_scores
        return context
```

Also update the import at the top of `student.py` — remove `QuizCategory` and `QuizLevel` since they are no longer used there:

```python
from quiz.models import Quiz, QuizAttempt, QuizQuestion
```

- [ ] **Step 2.6: Update `QuizAdmin` in `quiz/admin.py`**

Change `QuizAdmin` list settings and fieldsets (the `QuizQuestionAdmin` is unchanged):

```python
@admin.register(Quiz)
class QuizAdmin(admin.ModelAdmin):
    form = QuizAdminForm
    list_display = ('code', 'name', 'time_limit', 'is_public')
    list_filter = ('is_public', 'is_organization_private')
    search_fields = ('code', 'name')
    inlines = (QuizQuestionLinkInline,)
    fieldsets = (
        (None, {
            'fields': ('code', 'name', 'description'),
        }),
        (_l('Settings'), {
            'fields': ('time_limit', 'max_attempts', 'shuffle_questions',
                       'show_correctness', 'show_answers'),
        }),
        (_l('Access'), {
            'fields': ('is_public', 'is_organization_private',
                       'organizations', 'authors', 'curators', 'testers'),
        }),
    )
```

- [ ] **Step 2.7: Update `templates/quiz/list.html`**

Replace the entire file with:

```html
{% extends "common-content.html" %}

{% block body %}
<form method="get" style="margin-bottom: 1em">
    <input type="text" name="search" value="{{ search }}" placeholder="{{ _('Search quizzes...') }}">
    <button type="submit">{{ _('Filter') }}</button>
</form>
<table class="table">
    <thead>
        <tr>
            <th>{{ _('Quiz') }}</th>
            <th>{{ _('Your best') }}</th>
        </tr>
    </thead>
    <tbody>
        {% for quiz in quizzes %}
        <tr>
            <td><a href="{{ url('quiz_detail', quiz.code) }}">{{ quiz.name }}</a></td>
            <td>{{ best_scores.get(quiz.id, '') }}</td>
        </tr>
        {% else %}
        <tr><td colspan="2">{{ _('No quizzes found.') }}</td></tr>
        {% endfor %}
    </tbody>
</table>
{% endblock %}
```

- [ ] **Step 2.8: Run tests**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz -v 2
docker compose exec site python3 manage.py check
```

Expected: All tests pass, system check clean.

- [ ] **Step 2.9: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add quiz/forms.py quiz/views/student.py quiz/admin.py \
        templates/quiz/list.html quiz/tests/test_models.py
git commit -m "feat(quiz): remove category/level cascade, dynamic formset backend"
```

---

## Task 3: Rewrite `quiz_form.html` with dynamic question section

**Files:**
- Modify: `templates/quiz/quiz_form.html`
- Modify: `quiz/tests/test_models.py`

- [ ] **Step 3.1: Write failing integration tests**

Add to `quiz/tests/test_models.py`:

```python
class QuizEditDynamicFormsetTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.editor = create_user(
            username='dyneditor', user_permissions=('edit_own_quiz',))
        cls.q1 = create_question(title='dyn q1', code='dynq1',
                                  authors=(cls.editor.profile,))
        cls.q2 = create_question(title='dyn q2', code='dynq2',
                                  authors=(cls.editor.profile,))
        cls.quiz = create_quiz(code='dynquiz1',
                               questions=((cls.q1, 2.0), (cls.q2, 1.0)))
        cls.quiz.authors.add(cls.editor.profile)

    def test_create_page_has_empty_formset_and_template_row(self):
        self.client.force_login(self.editor)
        resp = self.client.get(reverse('quiz_create'))
        self.assertEqual(resp.status_code, 200)
        # No pre-filled rows (extra=0)
        self.assertNotContains(resp, 'question_links-0-question')
        # Management form present
        self.assertContains(resp, 'id_question_links-TOTAL_FORMS')
        # Template row for JS cloning
        self.assertContains(resp, '__prefix__')
        # Add button
        self.assertContains(resp, 'add-question-btn')

    def test_create_quiz_with_questions_via_post(self):
        self.client.force_login(self.editor)
        resp = self.client.post(reverse('quiz_create'), {
            'code': 'postquiz1',
            'name': 'Post Quiz',
            'question_links-TOTAL_FORMS': '2',
            'question_links-INITIAL_FORMS': '0',
            'question_links-MIN_NUM_FORMS': '0',
            'question_links-MAX_NUM_FORMS': '1000',
            'question_links-0-question': self.q1.id,
            'question_links-0-points': '2.0',
            'question_links-0-order': '1',
            'question_links-1-question': self.q2.id,
            'question_links-1-points': '1.0',
            'question_links-1-order': '2',
        })
        self.assertEqual(resp.status_code, 302)
        quiz = Quiz.objects.get(code='postquiz1')
        self.assertEqual(quiz.question_links.count(), 2)
        self.assertEqual(
            list(quiz.question_links.order_by('order')
                 .values_list('question_id', flat=True)),
            [self.q1.id, self.q2.id])

    def test_edit_page_shows_existing_question_rows(self):
        self.client.force_login(self.editor)
        resp = self.client.get(reverse('quiz_edit', kwargs={'quiz': 'dynquiz1'}))
        self.assertEqual(resp.status_code, 200)
        # Existing rows rendered
        self.assertContains(resp, 'question_links-0-question')
        self.assertContains(resp, 'question_links-1-question')
        # Template row still present for adding new questions
        self.assertContains(resp, '__prefix__')
```

- [ ] **Step 3.2: Run tests to confirm failure**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz.tests.test_models.QuizEditDynamicFormsetTest -v 2
```

Expected: Some tests pass (POST test should already work), `test_create_page_has_empty_formset_and_template_row` and `test_edit_page_shows_existing_question_rows` may pass already for management form but fail on `__prefix__` and `add-question-btn` (those are in the template we're rewriting).

- [ ] **Step 3.3: Rewrite `templates/quiz/quiz_form.html`**

Replace the entire file with:

```html
{% extends "common-content.html" %}

{% block media %}
    {{ formset.media.css }}
{% endblock %}

{% block js_media %}
    {{ formset.media.js }}
{% endblock %}

{% block body %}
<form method="post">
    {% csrf_token %}
    {{ form.as_p() }}

    <h3>{{ _('Questions') }}</h3>
    {{ formset.management_form }}
    <table class="table" id="question-links-table">
        <thead>
            <tr>
                <th style="width:32px;"></th>
                <th>{{ _('Question') }}</th>
                <th style="width:90px;">{{ _('Points') }}</th>
                <th style="width:36px;"></th>
            </tr>
        </thead>
        <tbody id="question-links-body">
            {% for link_form in formset %}
            <tr class="sortable-row">
                {% for hidden in link_form.hidden_fields() %}{{ hidden }}{% endfor %}
                <td class="drag-handle" title="{{ _('Drag to reorder') }}">⠿</td>
                <td>{{ link_form.question }}</td>
                <td>{{ link_form.points }}</td>
                <td>
                    <span style="display:none;">{{ link_form.DELETE }}</span>
                    <button type="button" class="delete-question-btn"
                            title="{{ _('Remove') }}">✕</button>
                </td>
            </tr>
            {% if link_form.errors %}
            <tr><td colspan="4" style="color:#c00;">{{ link_form.errors }}</td></tr>
            {% endif %}
            {% endfor %}
            {# Template row — JS clones this for new rows; Select2 skips it (id contains __prefix__) #}
            <tr class="sortable-template" style="display:none;">
                {% for hidden in formset.empty_form.hidden_fields() %}{{ hidden }}{% endfor %}
                <td class="drag-handle">⠿</td>
                <td>{{ formset.empty_form.question }}</td>
                <td>{{ formset.empty_form.points }}</td>
                <td>
                    <span style="display:none;">{{ formset.empty_form.DELETE }}</span>
                    <button type="button" class="delete-question-btn">✕</button>
                </td>
            </tr>
        </tbody>
    </table>
    <button type="button" id="add-question-btn" style="margin:8px 0;">
        + {{ _('Add question') }}
    </button>

    <br><br>
    <button type="submit">{{ _('Save quiz') }}</button>
</form>
{% if editing %}
<p>
    <a href="{{ url('quiz_detail', quiz.code) }}">{{ _('View quiz') }}</a> &middot;
    <a href="{{ url('quiz_attempts', quiz.code) }}">{{ _('Attempts &amp; regrade') }}</a>
</p>
{% endif %}

<style>
.drag-handle {
    cursor: grab;
    user-select: none;
    color: #aaa;
    text-align: center;
    font-size: 18px;
    padding: 4px 8px;
}
.drag-handle:active { cursor: grabbing; }
.ui-sortable-helper {
    background: #fff;
    box-shadow: 0 2px 8px rgba(0,0,0,.15);
    display: table;
}
.delete-question-btn {
    background: none;
    border: none;
    cursor: pointer;
    color: #c00;
    font-size: 16px;
    padding: 2px 6px;
    border-radius: 3px;
}
.delete-question-btn:hover { background: #fff0f0; }
.md-editor { border: 1px solid #ccc; border-radius: 4px; overflow: hidden; margin-top: 4px; }
.md-tabs { display: flex; background: #f5f5f5; border-bottom: 1px solid #ccc; }
.md-tab {
    background: none; border: none; padding: 5px 14px;
    font-size: 12px; font-weight: 600; color: #666; cursor: pointer;
    border-right: 1px solid #e0e0e0;
}
.md-tab.active { background: #fff; color: #417690; border-bottom: 2px solid #417690; margin-bottom: -1px; }
.md-tab:hover:not(.active) { background: #eee; }
.md-pane-write textarea { border: none !important; border-radius: 0 !important; display: block; margin: 0; width: 100%; box-sizing: border-box; }
.md-pane-preview { min-height: 60px; padding: 10px 12px; font-size: 14px; line-height: 1.6; background: #fff; }
.md-preview-loading, .md-preview-empty { color: #aaa; font-style: italic; font-size: 13px; }
</style>

<script>
(function () {
'use strict';

/* ── Order management ── */
function updateOrders() {
    $('#question-links-body tr.sortable-row:visible').each(function (i) {
        $(this).find('.order-field').val(i + 1);
    });
}

/* ── Drag-to-reorder ── */
$('#question-links-body').sortable({
    handle: '.drag-handle',
    items: 'tr.sortable-row',
    stop: updateOrders,
});

/* ── Add question ── */
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

/* ── Delete question ── */
$(document).on('click', '.delete-question-btn', function () {
    var $row = $(this).closest('tr');
    $row.find('[name$="-DELETE"]').prop('checked', true);
    $row.addClass('sortable-deleted').hide();
    updateOrders();
});

/* ── Set initial orders on page load ── */
updateOrders();

})();

/* ── Markdown preview for description field ── */
(function () {
'use strict';

var PREVIEW_URL = {{ url('quiz_preview')|tojson }};
var CSRF = document.querySelector('[name=csrfmiddlewaretoken]').value;

var textarea = document.getElementById('id_description');
if (!textarea) return;

var wrapper = document.createElement('div');
wrapper.className = 'md-editor';
wrapper.innerHTML =
    '<div class="md-tabs">' +
    '<button type="button" class="md-tab active" data-pane="write">{{ _("Write") }}</button>' +
    '<button type="button" class="md-tab" data-pane="preview">{{ _("Preview") }}</button>' +
    '</div>' +
    '<div class="md-pane-write"></div>' +
    '<div class="md-pane-preview" style="display:none;"></div>';

textarea.parentNode.insertBefore(wrapper, textarea);
wrapper.querySelector('.md-pane-write').appendChild(textarea);

var previewPane = wrapper.querySelector('.md-pane-preview');

wrapper.querySelectorAll('.md-tab').forEach(function (tab) {
    tab.addEventListener('click', function () {
        wrapper.querySelectorAll('.md-tab').forEach(function (t) { t.classList.remove('active'); });
        tab.classList.add('active');

        if (tab.dataset.pane === 'write') {
            textarea.parentNode.style.display = '';
            previewPane.style.display = 'none';
        } else {
            textarea.parentNode.style.display = 'none';
            previewPane.style.display = '';
            var content = textarea.value.trim();
            if (!content) {
                previewPane.innerHTML = '<span class="md-preview-empty">{{ _("Nothing to preview.") }}</span>';
                return;
            }
            previewPane.innerHTML = '<span class="md-preview-loading">{{ _("Loading…") }}</span>';
            fetch(PREVIEW_URL, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded',
                    'X-CSRFToken': CSRF,
                },
                body: 'content=' + encodeURIComponent(content),
            })
            .then(function (r) { return r.text(); })
            .then(function (html) { previewPane.innerHTML = html; })
            .catch(function () {
                previewPane.innerHTML = '<span class="md-preview-empty">{{ _("Preview failed.") }}</span>';
            });
        }
    });
});

})();
</script>
{% endblock %}
```

- [ ] **Step 3.4: Run tests and system check**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj
docker compose exec site python3 manage.py test quiz -v 2
docker compose exec site python3 manage.py check
```

Expected: All tests pass, system check clean.

- [ ] **Step 3.5: Commit**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker/dmoj/repo
git add templates/quiz/quiz_form.html quiz/tests/test_models.py
git commit -m "feat(quiz): dynamic drag-and-drop question list in quiz edit form"
```
