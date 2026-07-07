# Problem Copy Block — Design Spec

**Date:** 2026-07-07
**Status:** Approved

## Goal

Deter students from copying problem descriptions into ChatGPT (and similar tools) without thinking. This is a **soft deterrent**, not an unbreakable lock — determined students can still use DevTools or View Source.

## Approach

CSS-based `user-select: none` on the problem body + a `beforeunload` confirmation dialog when navigating away, both conditioned on a per-problem boolean flag. No JS event blocking on copy itself, no violation recording. Copy buttons on `<pre><code>` test-data blocks continue to work because clipboard.js reads from `data-clipboard-text`, not the user's selection.

## Design

### 1. Model — `Problem.block_copy`

**File:** `judge/models/problem.py` (~line 120, `class Problem`)

- `block_copy = BooleanField(default=False, help_text="Block text selection and show a warning on the problem description page.")`

A migration is auto-generated via `makemigrations`.

### 2. Template condition

**File:** `templates/problem/problem.html` (the main problem page, which includes `problem-detail.html`)

When **all** of the following are true:

- `problem.block_copy` is `True`
- `request.user.is_staff` is `False`
- `can_edit_problem` is `False` (covers the problem author/editor)

…the page renders:

1. **Warning banner** above the problem body:
   > ⚠ Copying is disabled for this problem. Please solve it on your own.

2. **CSS rule** on the problem description container:
   ```html
   <div style="user-select: none;">
       {% include "problem/problem-detail.html" %}
   </div>
   ```

When any exemption applies (staff, editor), the page renders normally with no banner and no `user-select` wrapper.

3. **Before-unload confirmation dialog** — when a blocked user tries to navigate away from the problem page, prompt with:
   > Are you sure you want to leave? Copying problem content to external tools is not allowed. Please solve the problem on your own.

   Implementation: attach a `beforeunload` event listener on `window` (conditioned on the same three checks). The browser's native confirm dialog will appear. Remove the listener on form submission (the "Submit solution" button) to avoid false positives when the student is legitimately submitting their code.

### 3. Copy buttons on code blocks remain functional

No change needed. The copy buttons in `common-content.html` use clipboard.js with `data-clipboard-text` — they write that text to the clipboard via `document.execCommand('copy')`, which does not depend on the user's text selection. `user-select: none` does not interfere.

### 4. PDF problems

`problem-detail.html` renders an `<object>` PDF embed **outside** the markdown body area. The `user-select` wrapper goes only around the markdown content, so PDF viewing is unaffected.

## What this does NOT block

- **DevTools** — students can still inspect elements and copy the inner text
- **View Source** — students can view the raw HTML
- **Screenshot + OCR** — students can screenshot the problem
- **Disabling CSS** — students can disable `user-select` with browser tools

These are acceptable gaps. The goal is to raise the friction for casual copy-paste cheating, not to implement DRM. Students who invest the effort to bypass this are the same students who would bypass any JS-level blocking.

## Admin UI

The `block_copy` field appears in:

- The problem edit form (Django admin and/or the site's problem edit page)
- Default: unchecked (disabled)

## Testing

- `user-select: none` is applied when `block_copy=True` for a regular student
- Warning banner appears
- `beforeunload` dialog fires when navigating away (not on "Submit solution" click)
- Staff users see normal page (no banner, no `user-select` block, no beforeunload)
- Problem author (`can_edit_problem`) sees normal page
- Copy button on `<pre><code>` still works regardless of `block_copy`
- PDF embed unaffected
