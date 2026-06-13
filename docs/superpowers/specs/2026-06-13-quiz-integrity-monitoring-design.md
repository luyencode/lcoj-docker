# Quiz Integrity Monitoring — Design Spec

**Date:** 2026-06-13  
**Status:** Approved  
**Scope:** Anti-cheat monitoring for the quiz take page — detection, alerting, recording, and teacher review.

---

## Overview

When a quiz has integrity monitoring enabled, students see a pre-quiz warning, suspicious behaviors are detected in the browser and recorded as violations, and teachers can review flagged attempts. Violations never affect a student's score.

---

## 1. Data Model

### `Quiz` model — new field

```python
integrity_monitoring = models.BooleanField(
    default=True,
    verbose_name=_('integrity monitoring'),
    help_text=_('Track suspicious behavior during this quiz.'),
)
```

Exposed in the quiz edit form and admin.

### New model: `QuizViolation`

```python
class ViolationType(models.TextChoices):
    TAB_SWITCH    = 'tab_switch',    _('Tab switch')
    WINDOW_BLUR   = 'window_blur',   _('Window blur')
    DEVTOOLS      = 'devtools',      _('DevTools opened')
    PRINT_SCREEN  = 'print_screen',  _('PrintScreen key')
    COPY_ATTEMPT  = 'copy_attempt',  _('Copy attempt')

class QuizViolation(models.Model):
    attempt     = models.ForeignKey(
        QuizAttempt, on_delete=models.CASCADE, related_name='violations')
    type        = models.CharField(max_length=20, choices=ViolationType.choices)
    occurred_at = models.DateTimeField()
    extra_data  = models.JSONField(default=dict)

    class Meta:
        ordering = ['occurred_at']
        indexes = [models.Index(fields=['attempt'])]
```

`occurred_at` is set client-side and sent with the AJAX request so the log reflects real event time, not server-receive time.

---

## 2. Pre-quiz Warning Screen

**Trigger:** When `quiz.integrity_monitoring` is `True` and the student clicks "Start quiz" (not "Resume attempt" — resuming skips the modal since the student already saw it).

**Implementation:** A JavaScript modal intercepts the start form's submit event. No new URL or server round-trip is needed.

**Modal content (exact copy):**

```
⚠  Academic Integrity Notice

This quiz monitors for suspicious behavior. During the attempt:

  • Copying text is disabled
  • Tab switches will be recorded
  • DevTools open will be recorded
  • PrintScreen key will be recorded

Violations are visible to the teacher but do NOT affect your score.

[ Cancel ]   [ I understand, start → ]
```

**Visual design:** Amber/orange warning icon at top; bullet items with icons; backdrop blur overlay; "Cancel" as a ghost/outline button; "I understand, start →" as a solid primary button.

**Behavior:**
- "Cancel" — closes modal, student stays on detail page.
- "I understand, start →" — submits the existing `quiz_start` POST form, proceeding normally.

---

## 3. Client-side Detection

A JavaScript module (`quiz-integrity.js`) is injected into the take page only when `quiz.integrity_monitoring` is `True`.

### Signals detected

| Signal | Detection method | Action |
|--------|-----------------|--------|
| Tab switch | `document.addEventListener('visibilitychange', ...)` | Alert + log |
| Window blur | `window.addEventListener('blur', ...)` | Alert + log |
| DevTools open | Periodic check: `window.outerWidth − window.innerWidth > 160` (200ms interval) | Alert + log |
| PrintScreen key | `document.addEventListener('keydown', e => e.key === 'PrintScreen')` | Alert + log |
| Copy attempt | `document.addEventListener('copy', e => e.preventDefault())` | Block + alert + log |

**Copy blocking:** Only `copy` is blocked (`e.preventDefault()`). Paste remains fully functional so students can type into short-answer fields.

**Right-click:** The context menu is disabled (`contextmenu` event prevented) to close off the right-click → Copy path. This is part of copy enforcement and is not announced separately in the warning modal — it falls under "Copying text is disabled".

### In-quiz alert toast

When a violation fires, a non-blocking toast appears at the top of the take page:

```
⚠  Tab switch detected. This has been recorded.
```

- Auto-dismisses after 4 seconds.
- Does not pause or interrupt the quiz.
- One toast at a time (subsequent violations replace the current toast).

### Debounce

Each violation type is debounced client-side with a **5-second cooldown** per type. If the user switches tabs 10 times in 10 seconds, only 2 events are sent. This prevents log flooding and excessive DB writes.

### Watermark

A fixed `<div class="quiz-watermark">` is rendered over the take page containing the student's username and the attempt start timestamp, repeated in a diagonal tile pattern:

```css
.quiz-watermark {
    position: fixed;
    inset: 0;
    pointer-events: none;
    opacity: 0.05;
    z-index: 9999;
    /* tiled with background-image or repeated absolutely-positioned spans */
    transform: rotate(-25deg);
    font-size: 14px;
    color: #000;
    user-select: none;
}
```

The watermark does not block interaction (`pointer-events: none`). It is a traceability measure: if a screenshot leaks, the student can be identified. It does not constitute a violation by itself.

---

## 4. Violation Recording Endpoint

### `POST /quizzes/<code>/attempt/<id>/violation/`

**Auth:** Student must own the attempt; attempt must be in-progress (not submitted, not expired).  
**Request body:**
```json
{
  "type": "tab_switch",
  "occurred_at": "2026-06-13T14:02:31.000Z",
  "extra_data": {}
}
```
**Response:**
```json
{ "recorded": true }
```

Server-side validation:
- `type` must be a valid `ViolationType` value.
- `occurred_at` must be within the attempt window (`started_at` to now + 30s grace).
- Silently drops requests after the attempt is submitted (no error, no record).

---

## 5. Teacher Review UI

### Attempts list (`/quizzes/<code>/attempts/`)

Each attempt row gains a **violation badge** column:

| # | Student | Started | Score | Violations |
|---|---------|---------|-------|------------|
| 1 | alice | Jun 13 14:02 | 18 / 20 | ⚠ 3 |
| 2 | bob | Jun 13 14:05 | 12 / 20 | — |

- Badge is teacher/staff-only; students cannot see it.
- Clicking the badge opens a **modal** loaded via AJAX.

### Violation log modal

```
Violations — alice · Attempt #1

  14:02:31   Tab switch
  14:05:18   DevTools opened
  14:07:44   PrintScreen key

3 violations total          [ Close ]
```

Loaded from `GET /quizzes/<code>/attempt/<id>/violations/` which returns JSON:
```json
{
  "student": "alice",
  "attempt_id": 1,
  "violations": [
    { "type": "tab_switch",  "label": "Tab switch",      "occurred_at": "2026-06-13T14:02:31Z" },
    { "type": "devtools",    "label": "DevTools opened",  "occurred_at": "2026-06-13T14:05:18Z" },
    { "type": "print_screen","label": "PrintScreen key",  "occurred_at": "2026-06-13T14:07:44Z" }
  ]
}
```

This endpoint is restricted to users who can edit the quiz (`can_edit` check, same as the existing editor views).

---

## 6. Quiz Edit Form

The `integrity_monitoring` toggle appears in the quiz edit form (editor view) and in Django admin, alongside existing fields like `time_limit` and `max_attempts`. Label: **"Integrity monitoring"**. Help text: **"Track suspicious behavior during this quiz."**

---

## 7. Constraints & Non-Goals

- Violations **never affect score**. A student with 10 violations scores the same as one with 0.
- Screenshot detection via OS tools (Snipping Tool, Cmd+Shift+4) is **not possible** — the watermark is the only mitigation.
- No Celery tasks, no WebSocket, no server push — all detection is client-side JS with AJAX callbacks.
- No automatic quiz termination or lockout — teachers decide what action to take after reviewing.
- v1 does not support exporting the violation log.
