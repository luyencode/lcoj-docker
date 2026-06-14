# Quiz Scheduling — Design Spec

> **Date:** 2026-06-14
> **Target codebase:** `dmoj/repo` (lcoj-site, DMOJ fork, Django 4.2)
> **Pattern reference:** `judge/models/contest.py` and `judge/views/contests.py`

## 1. Goal

Allow quiz editors to set an optional `start_time` and/or `end_time` on a quiz, creating an exam window. Students can see upcoming quizzes (with countdown) but cannot start until the window opens. After the window closes, no new attempts are accepted; in-progress attempts without a personal time limit are auto-submitted.

## 2. Scope

**In scope:**
- `start_time` / `end_time` nullable fields on `Quiz`
- Status properties (`can_start`, `ended`, `time_before_start`, `time_before_end`) mirroring the Contest pattern
- QuizList split into upcoming / open / past sections
- QuizDetail status banner with countdown
- QuizStart enforcement (gate by `can_start` and `ended`)
- Auto-finalization at `end_time` for no-time-limit attempts
- QuizForm datetime inputs with validation
- One migration

**Out of scope:**
- Delayed result reveal (results remain immediate as today)
- Recurring or multi-window schedules
- Email/push notifications when a quiz opens or closes
- Contest integration

## 3. Behavior

### 3.1 Window states

| Condition | State | Student experience |
|---|---|---|
| `start_time` is None AND `end_time` is None | Always open | No change from today |
| `start_time` is set, now < start_time | **Upcoming** | Detail page visible, countdown shown, Start disabled |
| `start_time` passed (or None), `end_time` not passed (or None) | **Open** | Normal — Start button available |
| `end_time` is set, now >= end_time | **Ended** | Detail/results still visible, Start blocked |

### 3.2 In-progress attempts at end_time

- Attempt **with** a personal `time_limit`: runs to its own deadline (`started_at + time_limit + GRACE`). `end_time` only blocks new starts.
- Attempt **without** a `time_limit`: `end_time` is treated as the hard deadline. `has_expired()` returns True as soon as `now > end_time`. Finalized lazily on the next request that touches the attempt (same pattern as the existing timer).

### 3.3 Visibility

Scheduling does not affect visibility. A quiz appears in the list based on `is_public` / org membership as today. `start_time` controls whether the Start button is active, not whether the quiz is visible.

## 4. Data model

### 4.1 New fields on `Quiz`

```python
start_time = models.DateTimeField(
    verbose_name=_('start time'), null=True, blank=True, db_index=True,
    help_text=_('Leave blank to make the quiz available immediately.'))
end_time = models.DateTimeField(
    verbose_name=_('end time'), null=True, blank=True, db_index=True,
    help_text=_('Leave blank for no closing time.'))
```

### 4.2 New properties on `Quiz`

```python
@cached_property
def _now(self):
    return timezone.now()

@cached_property
def can_start(self):
    return self.start_time is None or self.start_time <= self._now

@cached_property
def ended(self):
    return self.end_time is not None and self.end_time < self._now

@property
def time_before_start(self):
    if self.start_time and self._now <= self.start_time:
        return self.start_time - self._now
    return None

@property
def time_before_end(self):
    if self.end_time and self._now <= self.end_time:
        return self.end_time - self._now
    return None
```

### 4.3 Updated `QuizAttempt.has_expired()`

```python
def has_expired(self, now=None):
    now = now or timezone.now()
    # Personal time limit takes priority
    if self.deadline is not None and now > self.deadline + self.GRACE:
        return True
    # No personal time limit: quiz end_time is the hard deadline
    if self.deadline is None and self.quiz.end_time is not None:
        return now > self.quiz.end_time
    return False
```

### 4.4 Migration

One migration: `AddField` for `start_time` and `end_time` on `quiz_quiz`. Both nullable with `db_index=True`.

## 5. Views

### 5.1 `QuizStart`

Add before attempt creation:

```python
if not self.quiz.can_start:
    messages.error(request, _('This quiz has not started yet.'))
    return redirect('quiz_detail', quiz=self.quiz.code)
if self.quiz.ended:
    messages.error(request, _('This quiz is no longer accepting submissions.'))
    return redirect('quiz_detail', quiz=self.quiz.code)
```

### 5.2 `QuizDetail.get_context_data`

```python
context['can_start'] = self.quiz.can_start
context['ended'] = self.quiz.ended
context['time_before_start'] = self.quiz.time_before_start
context['time_before_end'] = self.quiz.time_before_end
context['now'] = self.quiz._now
```

### 5.3 `QuizList.get_context_data`

Split page quizzes into three buckets (evaluated after pagination):

```python
upcoming, open_, past = [], [], []
for quiz in context['quizzes']:
    if not quiz.can_start:
        upcoming.append(quiz)
    elif quiz.ended:
        past.append(quiz)
    else:
        open_.append(quiz)
context['upcoming_quizzes'] = upcoming
context['open_quizzes'] = open_
context['past_quizzes'] = past
```

`AttemptMixin.dispatch` needs no changes — it already calls `attempt.finalize()` when `has_expired()` returns True.

## 6. Templates

### 6.1 Quiz list (`quiz/list.html`)

Replace the single flat list with three sections:

- **Upcoming** — badge + `Starting in {{ as_countdown(quiz.time_before_start) }}`
- **Open** — `Ends in {{ as_countdown(quiz.time_before_end) }}` when `end_time` set; nothing otherwise
- **Past** — closed date label, no countdown, no Start button

Reuses existing `as_countdown()` Jinja2 function and `count_down()` JS — no new frontend code. When countdown reaches zero the page auto-reloads (existing JS behavior), transitioning the UI to the next state.

### 6.2 Quiz detail (`quiz/detail.html`)

Status banner above the Start button:

| State | Banner |
|---|---|
| Upcoming | "Starting in `<countdown>`" — Start button hidden |
| Open, no end_time | No banner |
| Open, has end_time | "Ends in `<countdown>`" — Start button shown |
| Ended | "This quiz closed on `<date>`" — Start button hidden |

### 6.3 Quiz editor form (`quiz/quiz_form.html`)

New "Scheduling" fieldset with `start_time` and `end_time` inputs. Helper text: "Leave blank for no restriction."

## 7. Forms

### 7.1 `QuizForm`

- Add `start_time`, `end_time` to `fields`
- Widget: `DateTimeInput(attrs={'type': 'datetime-local'})`
- Clean validation:

```python
def clean(self):
    cleaned = super().clean()
    start = cleaned.get('start_time')
    end = cleaned.get('end_time')
    if start and end and end <= start:
        raise ValidationError(_('End time must be after start time.'))
    return cleaned
```

## 8. Admin

Add `start_time` and `end_time` to `QuizAdmin` fieldsets under a "Scheduling" section.

## 9. Tests

Four new test cases (add to `test_scheduling.py` or `test_attempts.py`):

1. **Upcoming gate** — `QuizStart` POST when `start_time` is in the future → redirects to detail with error message
2. **Ended gate** — `QuizStart` POST when `end_time` has passed → redirects to detail with error message
3. **Auto-finalize no time_limit** — in-progress attempt with no `time_limit`, `end_time` passed → `has_expired()` returns True → `finalize()` called in `AttemptMixin`
4. **Personal timer wins** — in-progress attempt with `time_limit` still remaining, `end_time` passed → `has_expired()` returns False

All are `TestCase`-level; no browser automation required.
