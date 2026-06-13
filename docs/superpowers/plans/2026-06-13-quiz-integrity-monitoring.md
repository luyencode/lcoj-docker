# Quiz Integrity Monitoring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-quiz integrity monitoring — copy blocking, suspicious-behaviour detection (tab switch, window blur, DevTools, PrintScreen), client-side alerts, server-side violation recording, and a teacher review UI.

**Architecture:** New `QuizViolation` model stores timestamped events per attempt. A JS module (`quiz-integrity.js`) runs on the take page, fires debounced AJAX POSTs to a new endpoint, and shows non-blocking toasts. Teachers see violation counts on the attempts list and a timestamped log modal via a JSON endpoint.

**Tech Stack:** Django models/views, vanilla JS (no framework), Jinja2 templates, Django admin.

**Spec:** `docs/superpowers/specs/2026-06-13-quiz-integrity-monitoring-design.md`

---

## File map

| File | Change |
|------|--------|
| `quiz/models.py` | Add `Quiz.integrity_monitoring`; add `ViolationType` + `QuizViolation` model |
| `quiz/migrations/000X_...` | Auto-generated migration |
| `quiz/views/student.py` | Add `QuizRecordViolation` view; pass `integrity_monitoring` context to `QuizTake` |
| `quiz/views/editor.py` | Add `QuizViolationLog` view; annotate `QuizAttempts` queryset with violation counts |
| `quiz/urls.py` | Add `quiz_violation` and `quiz_violation_log` URL patterns |
| `quiz/forms.py` | Add `integrity_monitoring` to `QuizForm.Meta.fields` |
| `quiz/admin.py` | Add `integrity_monitoring` to `QuizAdmin` fieldsets; register `QuizViolation` |
| `templates/quiz/detail.html` | Add pre-quiz warning modal + JS |
| `templates/quiz/take.html` | Add watermark div, toast div, inject `quiz-integrity.js` |
| `templates/quiz/attempts.html` | Add violation badge column + modal |
| `resources/quiz-integrity.js` | NEW: client-side detection module |
| `quiz/tests/test_violations.py` | NEW: endpoint + model tests |

---

## Task 1: Model — `Quiz.integrity_monitoring` + `QuizViolation`

**Files:**
- Modify: `quiz/models.py`
- Create: `quiz/tests/test_violations.py`

- [ ] **Step 1: Write failing tests**

Create `quiz/tests/test_violations.py`:

```python
from django.test import TestCase

from quiz.models import Quiz, QuizAttempt, QuizViolation, ViolationType
from quiz.tests.util import create_question, create_quiz, create_user


class QuizViolationModelTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.student = create_user(username='vm_student')
        cls.q = create_question(title='vmq1')
        cls.quiz = create_quiz(code='vmquiz', questions=((cls.q, 1.0),))
        cls.attempt = QuizAttempt.start(cls.quiz, cls.student.profile)

    def test_integrity_monitoring_default_true(self):
        quiz = create_quiz(code='vmdefault')
        self.assertTrue(quiz.integrity_monitoring)

    def test_integrity_monitoring_can_be_disabled(self):
        quiz = create_quiz(code='vmdisabled', integrity_monitoring=False)
        self.assertFalse(quiz.integrity_monitoring)

    def test_violation_creation(self):
        from django.utils import timezone
        v = QuizViolation.objects.create(
            attempt=self.attempt,
            type=ViolationType.TAB_SWITCH,
            occurred_at=timezone.now(),
            extra_data={},
        )
        self.assertEqual(v.attempt, self.attempt)
        self.assertEqual(v.type, 'tab_switch')

    def test_violations_related_name(self):
        from django.utils import timezone
        QuizViolation.objects.create(
            attempt=self.attempt,
            type=ViolationType.DEVTOOLS,
            occurred_at=timezone.now(),
            extra_data={},
        )
        self.assertEqual(self.attempt.violations.count(), 1)

    def test_violation_ordering(self):
        from datetime import timedelta
        from django.utils import timezone
        now = timezone.now()
        QuizViolation.objects.create(
            attempt=self.attempt, type=ViolationType.TAB_SWITCH,
            occurred_at=now + timedelta(seconds=10), extra_data={})
        QuizViolation.objects.create(
            attempt=self.attempt, type=ViolationType.WINDOW_BLUR,
            occurred_at=now, extra_data={})
        types = list(self.attempt.violations.values_list('type', flat=True))
        self.assertEqual(types, ['window_blur', 'tab_switch'])
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker
docker compose exec site python3 manage.py test quiz.tests.test_violations -v 2
```

Expected: `ImportError` or `AttributeError` — `ViolationType` and `QuizViolation` don't exist yet.

- [ ] **Step 3: Add `ViolationType`, `QuizViolation`, and `Quiz.integrity_monitoring` to `quiz/models.py`**

After the `ResultFeedback` class (around line 39), add:

```python
class ViolationType(models.TextChoices):
    TAB_SWITCH   = 'tab_switch',   _('Tab switch')
    WINDOW_BLUR  = 'window_blur',  _('Window blur')
    DEVTOOLS     = 'devtools',     _('DevTools opened')
    PRINT_SCREEN = 'print_screen', _('PrintScreen key')
    COPY_ATTEMPT = 'copy_attempt', _('Copy attempt')
```

In the `Quiz` model, after the `updated_at` field (before `class Meta`), add:

```python
    integrity_monitoring = models.BooleanField(
        verbose_name=_('integrity monitoring'),
        default=True,
        help_text=_('Track suspicious behavior during this quiz.'),
    )
```

At the end of `quiz/models.py` (after `QuizAnswer`), add:

```python
class QuizViolation(models.Model):
    attempt = models.ForeignKey(
        QuizAttempt, on_delete=models.CASCADE, related_name='violations')
    type = models.CharField(
        max_length=20, choices=ViolationType.choices)
    occurred_at = models.DateTimeField()
    extra_data = models.JSONField(default=dict)

    class Meta:
        ordering = ['occurred_at']
        indexes = [models.Index(fields=['attempt'])]
        verbose_name = _('quiz violation')
        verbose_name_plural = _('quiz violations')

    def __str__(self):
        return '%s — %s' % (self.attempt, self.get_type_display())
```

- [ ] **Step 4: Generate and apply migration**

```bash
docker compose exec site python3 manage.py makemigrations quiz --name quiz_integrity_monitoring
docker compose exec site python3 manage.py migrate
```

Expected: new migration created, applied without errors.

- [ ] **Step 5: Run tests to verify they pass**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_violations -v 2
```

Expected: all 5 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add quiz/models.py quiz/migrations/ quiz/tests/test_violations.py
git commit -m "feat: add Quiz.integrity_monitoring field and QuizViolation model"
```

---

## Task 2: Violation recording endpoint (`POST .../violation/`)

**Files:**
- Modify: `quiz/views/student.py`
- Modify: `quiz/urls.py`
- Modify: `quiz/tests/test_violations.py`

- [ ] **Step 1: Add endpoint tests to `quiz/tests/test_violations.py`**

Append to the file:

```python
import json

from django.urls import reverse


class QuizRecordViolationTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.student = create_user(username='rv_student')
        cls.other = create_user(username='rv_other')
        cls.teacher = create_user(
            username='rv_teacher', user_permissions=('edit_own_quiz',))
        cls.q = create_question(title='rvq1')
        cls.quiz = create_quiz(code='rvquiz', questions=((cls.q, 1.0),))
        cls.quiz.authors.add(cls.teacher.profile)

    def setUp(self):
        self.attempt = QuizAttempt.start(self.quiz, self.student.profile)
        self.url = reverse('quiz_violation', kwargs={
            'quiz': 'rvquiz', 'attempt': self.attempt.id})

    def _post(self, data, user=None):
        self.client.force_login(user or self.student)
        return self.client.post(
            self.url,
            data=json.dumps(data),
            content_type='application/json',
        )

    def test_valid_violation_recorded(self):
        resp = self._post({'type': 'tab_switch',
                           'occurred_at': '2026-06-13T14:00:00Z'})
        self.assertEqual(resp.status_code, 200)
        self.assertEqual(json.loads(resp.content), {'recorded': True})
        self.assertEqual(self.attempt.violations.count(), 1)

    def test_invalid_type_returns_400(self):
        resp = self._post({'type': 'hacking',
                           'occurred_at': '2026-06-13T14:00:00Z'})
        self.assertEqual(resp.status_code, 400)
        self.assertEqual(self.attempt.violations.count(), 0)

    def test_non_owner_gets_404(self):
        resp = self._post({'type': 'tab_switch',
                           'occurred_at': '2026-06-13T14:00:00Z'},
                          user=self.other)
        self.assertEqual(resp.status_code, 404)

    def test_submitted_attempt_silently_ignored(self):
        self.attempt.finalize()
        resp = self._post({'type': 'tab_switch',
                           'occurred_at': '2026-06-13T14:00:00Z'})
        self.assertEqual(resp.status_code, 200)
        self.assertEqual(json.loads(resp.content), {'recorded': True})
        self.attempt.refresh_from_db()
        self.assertEqual(self.attempt.violations.count(), 0)

    def test_missing_body_returns_400(self):
        self.client.force_login(self.student)
        resp = self.client.post(self.url, data='not-json',
                                content_type='application/json')
        self.assertEqual(resp.status_code, 400)

    def test_extra_data_stored(self):
        resp = self._post({'type': 'devtools',
                           'occurred_at': '2026-06-13T14:00:00Z',
                           'extra_data': {'window_delta': 200}})
        self.assertEqual(resp.status_code, 200)
        v = self.attempt.violations.first()
        self.assertEqual(v.extra_data, {'window_delta': 200})
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_violations.QuizRecordViolationTest -v 2
```

Expected: `NoReverseMatch` — `quiz_violation` URL doesn't exist yet.

- [ ] **Step 3: Add `QuizRecordViolation` view to `quiz/views/student.py`**

Add imports at the top of `student.py` (after existing imports):

```python
from django.utils.dateparse import parse_datetime
from django.utils.timezone import is_aware, make_aware

from quiz.models import Quiz, QuizAttempt, QuizQuestion, QuizViolation, ViolationType
```

Replace the existing `from quiz.models import Quiz, QuizAttempt, QuizQuestion` line with the above.

Then append to `student.py` after `QuizSaveAnswer`:

```python
class QuizRecordViolation(LoginRequiredMixin, AttemptMixin, View):
    def post(self, request, *args, **kwargs):
        if self.attempt.is_submitted:
            return JsonResponse({'recorded': True})
        try:
            body = json.loads(request.body)
            vtype = body['type']
            occurred_at_str = body['occurred_at']
            extra_data = body.get('extra_data', {})
        except (KeyError, TypeError, ValueError, json.JSONDecodeError):
            return HttpResponseBadRequest()

        if vtype not in ViolationType.values:
            return JsonResponse({'error': 'invalid_type'}, status=400)

        occurred_at = parse_datetime(occurred_at_str)
        if occurred_at is None:
            return JsonResponse({'error': 'invalid_time'}, status=400)
        if not is_aware(occurred_at):
            occurred_at = make_aware(occurred_at)

        now = timezone.now()
        if occurred_at > now:
            occurred_at = now

        QuizViolation.objects.create(
            attempt=self.attempt,
            type=vtype,
            occurred_at=occurred_at,
            extra_data=extra_data if isinstance(extra_data, dict) else {},
        )
        return JsonResponse({'recorded': True})
```

- [ ] **Step 4: Add URL to `quiz/urls.py`**

Inside the `path('/attempt/<int:attempt>', include([...]))` block, add after `quiz_save`:

```python
path('/violation', student.QuizRecordViolation.as_view(), name='quiz_violation'),
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_violations.QuizRecordViolationTest -v 2
```

Expected: all 6 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add quiz/views/student.py quiz/urls.py quiz/tests/test_violations.py
git commit -m "feat: add violation recording endpoint POST quiz_violation"
```

---

## Task 3: Violation log endpoint + attempts list violation counts

**Files:**
- Modify: `quiz/views/editor.py`
- Modify: `quiz/urls.py`
- Modify: `quiz/tests/test_violations.py`

- [ ] **Step 1: Add violation log tests to `quiz/tests/test_violations.py`**

Append to the file:

```python
class QuizViolationLogTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        from django.utils import timezone
        cls.student = create_user(username='vl_student')
        cls.teacher = create_user(
            username='vl_teacher', user_permissions=('edit_own_quiz',))
        cls.other_teacher = create_user(
            username='vl_other', user_permissions=('edit_own_quiz',))
        cls.q = create_question(title='vlq1')
        cls.quiz = create_quiz(code='vlquiz', questions=((cls.q, 1.0),))
        cls.quiz.authors.add(cls.teacher.profile)
        cls.attempt = QuizAttempt.start(cls.quiz, cls.student.profile)
        cls.violation = QuizViolation.objects.create(
            attempt=cls.attempt,
            type=ViolationType.TAB_SWITCH,
            occurred_at=timezone.now(),
            extra_data={},
        )
        cls.log_url = reverse('quiz_violation_log', kwargs={
            'quiz': 'vlquiz', 'attempt': cls.attempt.id})

    def test_student_cannot_view_log(self):
        self.client.force_login(self.student)
        resp = self.client.get(self.log_url)
        self.assertEqual(resp.status_code, 404)

    def test_non_author_editor_cannot_view_log(self):
        self.client.force_login(self.other_teacher)
        resp = self.client.get(self.log_url)
        self.assertEqual(resp.status_code, 404)

    def test_author_can_view_log(self):
        self.client.force_login(self.teacher)
        resp = self.client.get(self.log_url)
        self.assertEqual(resp.status_code, 200)
        data = json.loads(resp.content)
        self.assertEqual(data['attempt_id'], self.attempt.id)
        self.assertEqual(data['student'], 'vl_student')
        self.assertEqual(len(data['violations']), 1)
        self.assertEqual(data['violations'][0]['type'], 'tab_switch')
        self.assertEqual(data['violations'][0]['label'], 'Tab switch')
        self.assertIn('occurred_at', data['violations'][0])

    def test_empty_log_returns_empty_list(self):
        empty_attempt = QuizAttempt.start(self.quiz, self.student.profile)
        url = reverse('quiz_violation_log', kwargs={
            'quiz': 'vlquiz', 'attempt': empty_attempt.id})
        self.client.force_login(self.teacher)
        resp = self.client.get(url)
        self.assertEqual(resp.status_code, 200)
        data = json.loads(resp.content)
        self.assertEqual(data['violations'], [])
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_violations.QuizViolationLogTest -v 2
```

Expected: `NoReverseMatch` — `quiz_violation_log` URL doesn't exist yet.

- [ ] **Step 3: Add `QuizViolationLog` view to `quiz/views/editor.py`**

Add import at the top of `editor.py` (after existing imports):

```python
from django.db.models import Count
from django.http import JsonResponse

from quiz.models import (QuestionType, Quiz, QuizAttempt, QuizCategory,
                         QuizLevel, QuizQuestion)
```

Replace the existing `from quiz.models import ...` import line with the above.

Append to `editor.py` after `QuizAttempts`:

```python
class QuizViolationLog(QuizEditorObjectMixin, View):
    def get(self, request, *args, **kwargs):
        attempt = get_object_or_404(
            QuizAttempt, id=kwargs['attempt'], quiz=self.quiz)
        violations = attempt.violations.all()
        return JsonResponse({
            'student': attempt.user.user.username,
            'attempt_id': attempt.id,
            'violations': [
                {
                    'type': v.type,
                    'label': v.get_type_display(),
                    'occurred_at': v.occurred_at.isoformat(),
                }
                for v in violations
            ],
        })
```

Also update `QuizAttempts.get_context_data` to annotate violation counts. Change:

```python
        context['attempts'] = self.quiz.attempts.select_related(
            'user__user').order_by('-started_at')
```

To:

```python
        context['attempts'] = self.quiz.attempts.select_related(
            'user__user').annotate(
            violation_count=Count('violations'),
        ).order_by('-started_at')
```

Also update the views generic import in `editor.py` — change:

```python
from django.views.generic import CreateView, ListView, TemplateView, UpdateView
```

To:

```python
from django.views.generic import CreateView, ListView, TemplateView, UpdateView, View
```

- [ ] **Step 4: Add `quiz_violation_log` URL to `quiz/urls.py`**

Inside the `path('/attempt/<int:attempt>', include([...]))` block, add after `quiz_violation`:

```python
path('/violations', editor.QuizViolationLog.as_view(), name='quiz_violation_log'),
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
docker compose exec site python3 manage.py test quiz.tests.test_violations -v 2
```

Expected: all tests PASS (model + endpoint + log tests).

- [ ] **Step 6: Commit**

```bash
git add quiz/views/editor.py quiz/urls.py quiz/tests/test_violations.py
git commit -m "feat: add violation log endpoint and annotate attempts with violation counts"
```

---

## Task 4: Form + admin

**Files:**
- Modify: `quiz/forms.py`
- Modify: `quiz/admin.py`

No new tests — these are form field inclusion and admin registration changes.

- [ ] **Step 1: Add `integrity_monitoring` to `QuizForm`**

In `quiz/forms.py`, in `QuizForm.Meta.fields`, add `'integrity_monitoring'` after `'result_feedback'`:

```python
        fields = ('code', 'name', 'description',
                  'time_limit', 'max_attempts', 'shuffle_questions',
                  'result_feedback', 'integrity_monitoring',
                  'is_public', 'is_organization_private', 'organizations',
                  'curators', 'testers')
```

- [ ] **Step 2: Add `integrity_monitoring` to `QuizAdmin` and register `QuizViolation`**

In `quiz/admin.py`, update the `QuizAdmin.fieldsets` Settings section:

```python
        (_l('Settings'), {
            'fields': ('time_limit', 'max_attempts', 'shuffle_questions',
                       'result_feedback', 'integrity_monitoring'),
        }),
```

At the top of `admin.py`, add `QuizViolation` and `ViolationType` to the models import:

```python
from quiz.models import (Quiz, QuizAnswer, QuizAttempt, QuizCategory,
                         QuizQuestion, QuizQuestionLink, QuizViolation)
```

At the bottom of `admin.py`, append:

```python
@admin.register(QuizViolation)
class QuizViolationAdmin(admin.ModelAdmin):
    list_display = ('attempt', 'type', 'occurred_at')
    list_filter = ('type',)
    readonly_fields = ('attempt', 'type', 'occurred_at', 'extra_data')
    can_delete = False

    def has_add_permission(self, request):
        return False
```

- [ ] **Step 3: Run system check**

```bash
docker compose exec site python3 manage.py check
```

Expected: `System check identified no issues`.

- [ ] **Step 4: Commit**

```bash
git add quiz/forms.py quiz/admin.py
git commit -m "feat: expose integrity_monitoring in quiz edit form and admin"
```

---

## Task 5: Pre-quiz warning modal (`detail.html`)

**Files:**
- Modify: `templates/quiz/detail.html`

- [ ] **Step 1: Add modal CSS to `{% block media %}`**

Inside the `<style>` block in `detail.html`, append before the closing `</style>`:

```css
    /* ── Integrity modal ────────────────────────────────────── */
    .integrity-overlay {
        display: none;
        position: fixed;
        inset: 0;
        background: rgba(0,0,0,0.55);
        backdrop-filter: blur(2px);
        z-index: 10000;
        align-items: center;
        justify-content: center;
    }
    .integrity-overlay.active {
        display: flex;
    }
    .integrity-modal {
        background: #fff;
        border-radius: 8px;
        box-shadow: 0 8px 32px rgba(0,0,0,0.18);
        max-width: 440px;
        width: 90%;
        padding: 2em 2em 1.5em;
        position: relative;
    }
    .integrity-modal-icon {
        text-align: center;
        font-size: 2.4em;
        margin-bottom: 0.4em;
        color: #d97706;
    }
    .integrity-modal h3 {
        text-align: center;
        font-size: 1.15em;
        margin: 0 0 0.8em;
        color: #1a1a1a;
    }
    .integrity-modal p {
        color: #444;
        font-size: 0.92em;
        margin: 0 0 0.8em;
    }
    .integrity-modal ul {
        margin: 0 0 1em 1.2em;
        padding: 0;
        color: #444;
        font-size: 0.92em;
    }
    .integrity-modal ul li {
        margin-bottom: 0.35em;
    }
    .integrity-modal .notice-footer {
        font-size: 0.85em;
        color: #666;
        margin-bottom: 1.4em;
    }
    .integrity-modal .modal-actions {
        display: flex;
        gap: 0.75em;
        justify-content: flex-end;
    }
    .btn-modal-cancel {
        padding: 0.55em 1.2em;
        border: 1px solid #999;
        border-radius: 4px;
        background: #fff;
        color: #444;
        cursor: pointer;
        font-size: 0.95em;
    }
    .btn-modal-confirm {
        padding: 0.55em 1.4em;
        border: none;
        border-radius: 4px;
        background: #1958c1;
        color: #fff;
        cursor: pointer;
        font-size: 0.95em;
        font-weight: bold;
    }
    .btn-modal-cancel:hover { background: #f4f4f4; }
    .btn-modal-confirm:hover { background: #0645ad; }
```

- [ ] **Step 2: Add modal HTML and JS to `{% block body %}`**

At the end of `{% block body %}`, before `{% endblock %}`, add:

```html
{% if quiz.integrity_monitoring and not in_progress and request.user.is_authenticated %}
<div class="integrity-overlay" id="integrity-overlay" role="dialog" aria-modal="true">
    <div class="integrity-modal">
        <div class="integrity-modal-icon">⚠</div>
        <h3>{{ _('Academic Integrity Notice') }}</h3>
        <p>{{ _('This quiz monitors for suspicious behavior. During the attempt:') }}</p>
        <ul>
            <li>{{ _('Copying text is disabled') }}</li>
            <li>{{ _('Tab switches will be recorded') }}</li>
            <li>{{ _('DevTools open will be recorded') }}</li>
            <li>{{ _('PrintScreen key will be recorded') }}</li>
        </ul>
        <p class="notice-footer">
            {{ _('Violations are visible to the teacher but do') }}
            <strong>{{ _('NOT') }}</strong> {{ _('affect your score.') }}
        </p>
        <div class="modal-actions">
            <button class="btn-modal-cancel" id="integrity-cancel">{{ _('Cancel') }}</button>
            <button class="btn-modal-confirm" id="integrity-confirm">{{ _('I understand, start →') }}</button>
        </div>
    </div>
</div>
<script>
(function () {
    var overlay = document.getElementById('integrity-overlay');
    var confirmBtn = document.getElementById('integrity-confirm');
    var cancelBtn = document.getElementById('integrity-cancel');
    var startForm = document.querySelector('form[action*="start"]');

    if (!startForm || !overlay) return;

    startForm.addEventListener('submit', function (e) {
        e.preventDefault();
        overlay.classList.add('active');
    });

    confirmBtn.addEventListener('click', function () {
        overlay.classList.remove('active');
        startForm.submit();
    });

    cancelBtn.addEventListener('click', function () {
        overlay.classList.remove('active');
    });

    overlay.addEventListener('click', function (e) {
        if (e.target === overlay) overlay.classList.remove('active');
    });
}());
</script>
{% endif %}
```

- [ ] **Step 3: Verify manually**

Start the services if not running:
```bash
cd /home/hieu/workspaces/luyencode/dev-lcoj-docker && docker compose up -d db redis site
```

Open a quiz with `integrity_monitoring=True` in a browser. Clicking "Start quiz" should show the modal. "Cancel" closes it. "I understand, start →" proceeds.

- [ ] **Step 4: Commit**

```bash
git add templates/quiz/detail.html
git commit -m "feat: add pre-quiz integrity warning modal"
```

---

## Task 6: Client-side detection (`quiz-integrity.js` + `take.html`)

**Files:**
- Create: `resources/quiz-integrity.js`
- Modify: `templates/quiz/take.html`

- [ ] **Step 1: Create `resources/quiz-integrity.js`**

```javascript
/* Quiz integrity monitoring — runs only when window.quizIntegrityConfig is set. */
(function () {
    'use strict';

    var cfg = window.quizIntegrityConfig;
    if (!cfg) return;

    var DEBOUNCE_MS = 5000;
    var lastFired = {};

    function canFire(type) {
        var now = Date.now();
        if (lastFired[type] && now - lastFired[type] < DEBOUNCE_MS) return false;
        lastFired[type] = now;
        return true;
    }

    var toastEl = document.getElementById('quiz-integrity-toast');
    var toastTimer = null;

    function showToast(msg) {
        if (!toastEl) return;
        toastEl.textContent = msg;
        toastEl.style.display = 'block';
        if (toastTimer) clearTimeout(toastTimer);
        toastTimer = setTimeout(function () {
            toastEl.style.display = 'none';
        }, 4000);
    }

    function record(type, msg) {
        if (!canFire(type)) return;
        showToast(msg);
        fetch(cfg.violationUrl, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRFToken': cfg.csrfToken,
            },
            body: JSON.stringify({
                type: type,
                occurred_at: new Date().toISOString(),
                extra_data: {},
            }),
        }).catch(function () {});
    }

    /* Tab switch */
    document.addEventListener('visibilitychange', function () {
        if (document.hidden) {
            record('tab_switch', '⚠ Tab switch detected. This has been recorded.');
        }
    });

    /* Window blur */
    window.addEventListener('blur', function () {
        record('window_blur', '⚠ Window blur detected. This has been recorded.');
    });

    /* PrintScreen key */
    document.addEventListener('keydown', function (e) {
        if (e.key === 'PrintScreen') {
            record('print_screen', '⚠ Screenshot key detected. This has been recorded.');
        }
    });

    /* Copy block */
    document.addEventListener('copy', function (e) {
        e.preventDefault();
        record('copy_attempt', '⚠ Copying is not allowed during this quiz.');
    });

    /* Right-click disable (part of copy enforcement) */
    document.addEventListener('contextmenu', function (e) {
        e.preventDefault();
    });

    /* DevTools detection — periodic window size heuristic */
    var devtoolsOpen = false;
    setInterval(function () {
        var widthDelta = window.outerWidth - window.innerWidth;
        var heightDelta = window.outerHeight - window.innerHeight;
        var detected = widthDelta > 160 || heightDelta > 160;
        if (detected && !devtoolsOpen) {
            devtoolsOpen = true;
            record('devtools', '⚠ DevTools detected. This has been recorded.');
        } else if (!detected) {
            devtoolsOpen = false;
        }
    }, 1000);

}());
```

- [ ] **Step 2: Add watermark, toast, and integrity config to `take.html`**

In `take.html`, inside `{% block media %}`, append to the `<style>` block before `</style>`:

```css
    /* ── Integrity watermark ─────────────────────────────────── */
    .quiz-watermark {
        position: fixed;
        inset: 0;
        pointer-events: none;
        z-index: 9999;
        overflow: hidden;
        display: flex;
        flex-wrap: wrap;
        align-content: flex-start;
        gap: 80px 60px;
        padding: 60px 40px;
        transform: rotate(-25deg);
        transform-origin: center center;
        opacity: 0.05;
        user-select: none;
    }
    .quiz-watermark span {
        font-size: 13px;
        color: #000;
        white-space: nowrap;
    }
    /* ── Integrity toast ─────────────────────────────────────── */
    #quiz-integrity-toast {
        display: none;
        position: fixed;
        top: 14px;
        left: 50%;
        transform: translateX(-50%);
        z-index: 10001;
        background: #7c3e00;
        color: #fff;
        padding: 0.6em 1.4em;
        border-radius: 4px;
        font-size: 0.92em;
        box-shadow: 0 2px 8px rgba(0,0,0,0.25);
        white-space: nowrap;
    }
```

Inside `{% block body %}`, just before the closing `</div>` of the outer `.quiz-take-wrap` (or just before `{% endblock %}`), add:

```html
{% if attempt.quiz.integrity_monitoring %}
<div class="quiz-watermark" aria-hidden="true">
    {% for i in range(30) %}
    <span>{{ request.user.username }} · {{ attempt.started_at|date('Y-m-d H:i') }}</span>
    {% endfor %}
</div>
<div id="quiz-integrity-toast" role="alert" aria-live="assertive"></div>
<script>
window.quizIntegrityConfig = {
    violationUrl: '{{ url('quiz_violation', quiz.code, attempt.id) }}',
    csrfToken: '{{ csrf_token }}',
};
</script>
<script src="{{ static('quiz-integrity.js') }}"></script>
{% endif %}
```

- [ ] **Step 3: Verify manually**

Open a quiz take page with `integrity_monitoring=True`. Verify:
- Watermark appears (very faint, rotated text with username + timestamp)
- Switching tabs shows toast on return
- Pressing PrintScreen shows toast
- Ctrl+C / copy is blocked
- Right-click is disabled
- DevTools opens → toast within 1-2 seconds

- [ ] **Step 4: Commit**

```bash
git add resources/quiz-integrity.js templates/quiz/take.html
git commit -m "feat: add quiz-integrity.js detection module and take page watermark/toast"
```

---

## Task 7: Teacher violation badge in `attempts.html`

**Files:**
- Modify: `templates/quiz/attempts.html`

- [ ] **Step 1: Rewrite `attempts.html` with violation badge and modal**

Replace the entire file content with:

```html
{% extends "common-content.html" %}

{% block media %}
<style>
    .violation-badge {
        display: inline-block;
        background: #fef3c7;
        border: 1px solid #d97706;
        color: #92400e;
        border-radius: 99px;
        padding: 0.15em 0.6em;
        font-size: 0.82em;
        font-weight: bold;
        cursor: pointer;
        white-space: nowrap;
    }
    .violation-badge:hover {
        background: #fde68a;
    }
    /* Violation log modal */
    .vlog-overlay {
        display: none;
        position: fixed;
        inset: 0;
        background: rgba(0,0,0,0.5);
        z-index: 10000;
        align-items: center;
        justify-content: center;
    }
    .vlog-overlay.active { display: flex; }
    .vlog-modal {
        background: #fff;
        border-radius: 8px;
        box-shadow: 0 8px 32px rgba(0,0,0,0.18);
        max-width: 480px;
        width: 90%;
        padding: 1.5em 1.5em 1em;
    }
    .vlog-modal h3 {
        margin: 0 0 1em;
        font-size: 1.05em;
        border-bottom: 1px solid #eee;
        padding-bottom: 0.5em;
    }
    .vlog-table {
        width: 100%;
        border-collapse: collapse;
        font-size: 0.9em;
    }
    .vlog-table td {
        padding: 0.35em 0.5em;
        vertical-align: top;
    }
    .vlog-table td:first-child {
        color: #555;
        white-space: nowrap;
        width: 110px;
    }
    .vlog-footer {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-top: 1em;
        padding-top: 0.75em;
        border-top: 1px solid #eee;
        font-size: 0.88em;
        color: #666;
    }
    .btn-vlog-close {
        padding: 0.4em 1.2em;
        border: 1px solid #ccc;
        border-radius: 4px;
        background: #fff;
        cursor: pointer;
    }
    .btn-vlog-close:hover { background: #f4f4f4; }
    .vlog-loading { color: #888; font-size: 0.9em; }
    .vlog-error { color: #c00; font-size: 0.9em; }
</style>
{% endblock %}

{% block body %}
<form method="post" onsubmit="return confirm('{{ _('Regrade all submitted attempts?') }}')">
    {% csrf_token %}
    <button type="submit">{{ _('Regrade all attempts') }}</button>
</form>

<table class="table">
    <thead>
        <tr>
            <th>{{ _('User') }}</th>
            <th>{{ _('Started') }}</th>
            <th>{{ _('Status') }}</th>
            <th>{{ _('Score') }}</th>
            <th>{{ _('Violations') }}</th>
            <th></th>
        </tr>
    </thead>
    <tbody>
        {% for attempt in attempts %}
        <tr>
            <td>{{ attempt.user.user.username }}</td>
            <td>{{ attempt.started_at|date('DATETIME_FORMAT') }}</td>
            <td>{{ _('submitted') if attempt.is_submitted else _('in progress') }}</td>
            <td>{% if attempt.is_submitted %}{{ attempt.score }} / {{ total_points }}{% endif %}</td>
            <td>
                {% if attempt.violation_count %}
                <span class="violation-badge"
                      data-url="{{ url('quiz_violation_log', quiz.code, attempt.id) }}"
                      role="button"
                      tabindex="0"
                      title="{{ _('View violation log') }}">
                    ⚠ {{ attempt.violation_count }}
                </span>
                {% else %}
                —
                {% endif %}
            </td>
            <td>
                {% if attempt.is_submitted %}
                <a href="{{ url('quiz_result', quiz.code, attempt.id) }}">{{ _('view') }}</a>
                {% endif %}
            </td>
        </tr>
        {% else %}
        <tr><td colspan="6">{{ _('No attempts yet.') }}</td></tr>
        {% endfor %}
    </tbody>
</table>

<!-- Violation log modal -->
<div class="vlog-overlay" id="vlog-overlay" role="dialog" aria-modal="true">
    <div class="vlog-modal">
        <h3 id="vlog-title">{{ _('Violations') }}</h3>
        <div id="vlog-body"><span class="vlog-loading">{{ _('Loading…') }}</span></div>
        <div class="vlog-footer">
            <span id="vlog-count"></span>
            <button class="btn-vlog-close" id="vlog-close">{{ _('Close') }}</button>
        </div>
    </div>
</div>

<script>
(function () {
    var overlay = document.getElementById('vlog-overlay');
    var titleEl = document.getElementById('vlog-title');
    var bodyEl  = document.getElementById('vlog-body');
    var countEl = document.getElementById('vlog-count');
    var closeBtn = document.getElementById('vlog-close');

    function openLog(url) {
        bodyEl.innerHTML = '<span class="vlog-loading">{{ _('Loading…') }}</span>';
        countEl.textContent = '';
        titleEl.textContent = '{{ _('Violations') }}';
        overlay.classList.add('active');

        fetch(url, {headers: {'X-Requested-With': 'XMLHttpRequest'}})
            .then(function (r) { return r.json(); })
            .then(function (data) {
                titleEl.textContent = '{{ _('Violations') }} — ' + data.student;
                if (!data.violations.length) {
                    bodyEl.innerHTML = '<em>{{ _('No violations recorded.') }}</em>';
                    countEl.textContent = '';
                    return;
                }
                var rows = data.violations.map(function (v) {
                    var t = new Date(v.occurred_at);
                    var time = t.toLocaleTimeString([], {hour: '2-digit', minute: '2-digit', second: '2-digit'});
                    return '<tr><td>' + time + '</td><td>' + v.label + '</td></tr>';
                }).join('');
                bodyEl.innerHTML = '<table class="vlog-table"><tbody>' + rows + '</tbody></table>';
                countEl.textContent = data.violations.length + ' {{ _('violation(s) total') }}';
            })
            .catch(function () {
                bodyEl.innerHTML = '<span class="vlog-error">{{ _('Failed to load.') }}</span>';
            });
    }

    document.querySelectorAll('.violation-badge').forEach(function (badge) {
        badge.addEventListener('click', function () {
            openLog(this.dataset.url);
        });
        badge.addEventListener('keydown', function (e) {
            if (e.key === 'Enter' || e.key === ' ') openLog(this.dataset.url);
        });
    });

    closeBtn.addEventListener('click', function () {
        overlay.classList.remove('active');
    });
    overlay.addEventListener('click', function (e) {
        if (e.target === overlay) overlay.classList.remove('active');
    });
}());
</script>
{% endblock %}
```

- [ ] **Step 2: Run full quiz test suite to check for regressions**

```bash
docker compose exec site python3 manage.py test quiz -v 2
```

Expected: all tests pass.

- [ ] **Step 3: Verify manually**

Open the attempts list for a quiz where some attempts have violations. Verify:
- Attempts with violations show the `⚠ N` amber badge
- Attempts without violations show `—`
- Clicking a badge opens the modal with the timestamped log
- Close button and backdrop click dismiss the modal

- [ ] **Step 4: Commit**

```bash
git add templates/quiz/attempts.html
git commit -m "feat: add violation badge and log modal to teacher attempts list"
```

---

## Task 8: Final end-to-end verification

- [ ] **Step 1: Run the full quiz test suite**

```bash
docker compose exec site python3 manage.py test quiz -v 2
```

Expected: all tests pass, no regressions.

- [ ] **Step 2: Django system check**

```bash
docker compose exec site python3 manage.py check
```

Expected: `System check identified no issues`.

- [ ] **Step 3: Manual smoke test**

1. Log in as a teacher, create/edit a quiz — confirm **Integrity monitoring** checkbox appears in the form (checked by default).
2. Log in as a student, go to quiz detail — confirm warning modal appears before start (not before resume).
3. Start the quiz — confirm:
   - Watermark visible (faint diagonal text with username + date).
   - Switching tabs shows toast on return.
   - Ctrl+C blocked; right-click disabled.
   - PrintScreen toast appears.
4. Submit the quiz.
5. As teacher, open attempts list — confirm `⚠ N` badge appears.
6. Click badge — confirm timestamped log modal opens with correct events.
