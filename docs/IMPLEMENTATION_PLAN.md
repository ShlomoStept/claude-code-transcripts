# Implementation Plan: Claude Code Transcripts UI Overhaul

## Executive Summary

This document outlines a comprehensive plan to address six core UI/UX issues identified in the claude-code-transcripts HTML output, plus integration of a Craft.do-inspired visual style. The issues range from text overflow problems to inconsistent toggle mechanisms and duplicate copy buttons.

### Issues at a Glance

| Issue | Severity | Summary |
|-------|----------|---------|
| #1 User Message Layout | Medium | Text overflow, not wrapped in collapsible cell |
| #2 Markdown/JSON Toggle UX | Medium | Small button, poor discoverability |
| #3 Collapsibility Inconsistencies | High | Two mechanisms (`<details>` vs truncatable), no sticky cascade |
| #4 Missing Toggles on Specialized Tools | Medium | Write/Edit/Bash/TodoWrite lack JSON toggle |
| #5 Duplicate Copy Buttons | Low | Cell + JavaScript both add copy buttons |
| #6 Copy Behavior Clarity | Medium | No format indicator, doesn't respect view mode |

### Target Aesthetic: Craft.do Style

- Warm off-white/cream backgrounds (`#FAF9F7`, `#F5F3EF`)
- Deep charcoal text (`#2D2D2D`), warm grays (`#6B6B6B`)
- Soft purple gradients, sky blue accents, muted pastels
- Card-based layouts with soft shadows, 12-16px border-radius
- Generous whitespace, frosted glass effects on sticky elements

---

## Phase 1: Foundation (CSS Variables & Base Styling)

### Objective
Establish CSS custom properties and base styles that will be used throughout all subsequent phases.

### Tasks

#### 1.1 Create CSS Variable System
**File:** `src/claude_code_transcripts/__init__.py` (CSS_STYLES constant)

Add at the top of `:root`:
```css
:root {
  /* Craft.do Color Palette */
  --color-bg-primary: #FAF9F7;
  --color-bg-secondary: #F5F3EF;
  --color-bg-card: #FFFFFF;
  --color-bg-code: #F8F6F3;

  --color-text-primary: #2D2D2D;
  --color-text-secondary: #6B6B6B;
  --color-text-muted: #9B9B9B;

  --color-accent-purple: #7C5CFF;
  --color-accent-purple-light: #EDE9FF;
  --color-accent-blue: #5BA4E6;
  --color-accent-blue-light: #E6F3FC;

  --color-border: #E8E6E3;
  --color-border-light: #F0EEEB;

  /* Shadows */
  --shadow-soft: 0 2px 8px rgba(0, 0, 0, 0.04);
  --shadow-medium: 0 4px 16px rgba(0, 0, 0, 0.08);
  --shadow-sticky: 0 4px 12px rgba(0, 0, 0, 0.1);

  /* Border Radius */
  --radius-sm: 6px;
  --radius-md: 12px;
  --radius-lg: 16px;

  /* Spacing */
  --space-xs: 4px;
  --space-sm: 8px;
  --space-md: 16px;
  --space-lg: 24px;
  --space-xl: 32px;

  /* Frosted Glass */
  --glass-bg: rgba(255, 255, 255, 0.85);
  --glass-backdrop: blur(12px);

  /* Transitions */
  --transition-fast: 150ms ease;
  --transition-medium: 250ms ease;
}
```

#### 1.2 Update Base Body Styles
```css
body {
  background: var(--color-bg-primary);
  color: var(--color-text-primary);
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}
```

#### 1.3 Update Card/Message Styles
Apply new variables to existing `.message`, `.cell`, `.tool-result` classes.

### Acceptance Criteria
- [ ] All color values use CSS variables
- [ ] Dark mode variables defined (optional, for future)
- [ ] No visual regression in existing output

---

## Phase 2: User Messages

### Objective
Fix text overflow in user messages and wrap them in collapsible cell structure for consistency.

### Tasks

#### 2.1 Add Text Wrapping CSS
**File:** `src/claude_code_transcripts/__init__.py` (CSS_STYLES)

```css
.user-content {
  overflow-wrap: break-word;
  word-break: break-word;
  white-space: pre-wrap;
}
```

#### 2.2 Wrap User Content in Cell Macro
**File:** `src/claude_code_transcripts/templates/macros.html`

**Current** (around line 15-25):
```jinja2
{% macro user_content(text) %}
<div class="user-content">{{ text }}</div>
{% endmacro %}
```

**New:**
```jinja2
{% macro user_content(text) %}
{% call cell("user-message", "User Message", icon="user") %}
<div class="user-content">{{ text }}</div>
{% endcall %}
{% endmacro %}
```

#### 2.3 Add User Icon
Add SVG icon for user messages to the icon system.

### Acceptance Criteria
- [ ] Long URLs in user messages wrap correctly
- [ ] User messages have collapsible structure matching assistant messages
- [ ] Copy button works on user messages

---

## Phase 3: Toggle System (Tab-Style)

### Objective
Replace small toggle buttons with shadcn-style tab toggles for Markdown/JSON views.

### Tasks

#### 3.1 Define Tab Toggle CSS
**File:** `src/claude_code_transcripts/__init__.py` (CSS_STYLES)

```css
.view-toggle-tabs {
  display: inline-flex;
  background: var(--color-bg-secondary);
  border-radius: var(--radius-sm);
  padding: 2px;
  gap: 2px;
}

.view-toggle-tabs button {
  padding: 4px 12px;
  border: none;
  background: transparent;
  border-radius: calc(var(--radius-sm) - 2px);
  font-size: 0.8rem;
  color: var(--color-text-secondary);
  cursor: pointer;
  transition: var(--transition-fast);
}

.view-toggle-tabs button:hover {
  color: var(--color-text-primary);
}

.view-toggle-tabs button.active {
  background: var(--color-bg-card);
  color: var(--color-text-primary);
  box-shadow: var(--shadow-soft);
}
```

#### 3.2 Update Toggle Button HTML
**File:** `src/claude_code_transcripts/templates/macros.html`

**Current** (search for `view-toggle-btn`):
```html
<button class="view-toggle-btn" onclick="toggleView(this)">JSON</button>
```

**New:**
```html
<div class="view-toggle-tabs">
  <button class="active" data-view="markdown" onclick="setView(this, 'markdown')">Markdown</button>
  <button data-view="json" onclick="setView(this, 'json')">JSON</button>
</div>
```

#### 3.3 Update JavaScript Toggle Logic
**File:** `src/claude_code_transcripts/__init__.py` (JS_CODE)

Replace `toggleView()` function:
```javascript
function setView(btn, view) {
  const container = btn.closest('.cell-content, .tool-content');
  const tabs = btn.closest('.view-toggle-tabs');

  // Update tab state
  tabs.querySelectorAll('button').forEach(b => b.classList.remove('active'));
  btn.classList.add('active');

  // Update view visibility
  const mdView = container.querySelector('.markdown-view');
  const jsonView = container.querySelector('.json-view');

  if (view === 'markdown') {
    mdView.style.display = 'block';
    jsonView.style.display = 'none';
  } else {
    mdView.style.display = 'none';
    jsonView.style.display = 'block';
  }
}
```

### Acceptance Criteria
- [ ] Tabs clearly show which view is active
- [ ] Transition between views is smooth
- [ ] Both views preserve scroll position

---

## Phase 4: Collapsibility Unification

### Objective
Replace all `truncatable + Show more/less` patterns with `<details>` elements and implement sticky cascade.

### Tasks

#### 4.1 Identify All Truncatable Patterns
**Grep pattern:** `truncatable|show-more|show-less`

Current locations:
- Tool results (file contents, bash output)
- Thinking blocks
- Long text responses

#### 4.2 Create Unified Collapsible Macro
**File:** `src/claude_code_transcripts/templates/macros.html`

```jinja2
{% macro collapsible(summary, id=none, default_open=true, sticky=false) %}
<details class="collapsible{% if sticky %} sticky-header{% endif %}"{% if default_open %} open{% endif %}{% if id %} id="{{ id }}"{% endif %}>
  <summary class="collapsible-summary">
    <span class="collapsible-icon">▶</span>
    {{ summary }}
  </summary>
  <div class="collapsible-content">
    {{ caller() }}
  </div>
</details>
{% endmacro %}
```

#### 4.3 Define Sticky Cascade CSS
```css
/* Sticky Cascade: Message → Cell → Subcell */
.message-header.sticky {
  position: sticky;
  top: 0;
  z-index: 30;
  background: var(--glass-bg);
  backdrop-filter: var(--glass-backdrop);
}

.cell > summary.sticky {
  position: sticky;
  top: 40px; /* Height of message header */
  z-index: 20;
  background: var(--glass-bg);
  backdrop-filter: var(--glass-backdrop);
}

.subcell > summary.sticky {
  position: sticky;
  top: 80px; /* Height of message + cell headers */
  z-index: 10;
  background: var(--glass-bg);
  backdrop-filter: var(--glass-backdrop);
}
```

#### 4.4 Remove Truncatable JavaScript
**File:** `src/claude_code_transcripts/__init__.py` (JS_CODE)

Remove or comment out:
- `initTruncation()` function
- Related event listeners
- CSS for `.truncatable`, `.show-more`, `.show-less`

#### 4.5 Add Thinking Block Collapsibility
Wrap thinking blocks in collapsible macro with default closed state.

#### 4.6 Add Response Block Truncation
For very long responses (>50 lines), wrap in collapsible with summary showing first 3 lines.

### Acceptance Criteria
- [ ] All expandable content uses `<details>` elements
- [ ] Sticky headers follow cascade (message → cell → subcell)
- [ ] Frosted glass effect on sticky elements
- [ ] No JS-based truncation remains
- [ ] Smooth open/close animations

---

## Phase 5: Copy System

### Objective
Remove duplicate copy buttons, add format indicators, and make copy respect current view mode.

### Tasks

#### 5.1 Prevent JavaScript Copy Button Injection in Cells
**File:** `src/claude_code_transcripts/__init__.py` (JS_CODE)

**Current** (search for copy button injection):
```javascript
document.querySelectorAll('.tool-result .truncatable-content').forEach(el => {
  // Add copy button
});
```

**New:**
```javascript
document.querySelectorAll('.tool-result .truncatable-content').forEach(el => {
  // Skip if already inside a cell (which has its own copy button)
  if (el.closest('.cell-content')) return;
  // Add copy button
});
```

#### 5.2 Add Format Indicator to Copy Button
**File:** `src/claude_code_transcripts/templates/macros.html`

```html
<button class="cell-copy-btn" onclick="copyCell(this)" data-format="markdown">
  <span class="copy-icon">📋</span>
  <span class="copy-label">Copy MD</span>
</button>
```

#### 5.3 Update Copy Logic for View Mode
**File:** `src/claude_code_transcripts/__init__.py` (JS_CODE)

```javascript
function copyCell(btn) {
  const cell = btn.closest('.cell');
  const activeView = cell.querySelector('.view-toggle-tabs button.active');
  const format = activeView ? activeView.dataset.view : 'markdown';

  let content;
  if (format === 'json') {
    content = cell.querySelector('.json-view pre')?.textContent;
  } else {
    content = cell.querySelector('.markdown-view')?.textContent;
  }

  navigator.clipboard.writeText(content).then(() => {
    btn.querySelector('.copy-label').textContent = 'Copied!';
    setTimeout(() => {
      btn.querySelector('.copy-label').textContent = `Copy ${format === 'json' ? 'JSON' : 'MD'}`;
    }, 2000);
  });
}
```

#### 5.4 Add Cell Header Master Toggle
Add a toggle in the cell header that shows/hides all content in that cell.

### Acceptance Criteria
- [ ] No duplicate copy buttons appear
- [ ] Copy button shows format (MD or JSON)
- [ ] Copied content matches visible view
- [ ] "Copied!" feedback appears briefly

---

## Phase 6: Specialized Tools

### Objective
Add Markdown/JSON toggle to Write, Edit, Bash, and TodoWrite macros.

### Tasks

#### 6.1 Update Write Macro
**File:** `src/claude_code_transcripts/templates/macros.html`

**Location:** Search for `{% macro write_tool`

Add toggle tabs and JSON view alongside existing content display.

#### 6.2 Update Edit Macro
**File:** `src/claude_code_transcripts/templates/macros.html`

**Location:** Search for `{% macro edit_tool`

Add toggle tabs showing:
- Markdown view: Diff display (current)
- JSON view: Raw edit parameters

#### 6.3 Update Bash Macro
**File:** `src/claude_code_transcripts/templates/macros.html`

**Location:** Search for `{% macro bash_tool`

Add toggle tabs showing:
- Markdown view: Formatted command + output
- JSON view: Raw input/result JSON

#### 6.4 Update TodoWrite Macro
**File:** `src/claude_code_transcripts/templates/macros.html`

**Location:** Search for `{% macro todo`

Add toggle tabs showing:
- Markdown view: Task list with checkboxes
- JSON view: Raw todo data

#### 6.5 Create Shared Toggle Helper
To reduce duplication, create a helper macro:

```jinja2
{% macro tool_with_toggle(tool_type, raw_input, raw_result) %}
<div class="tool-container">
  <div class="view-toggle-tabs">
    <button class="active" data-view="markdown" onclick="setView(this, 'markdown')">Markdown</button>
    <button data-view="json" onclick="setView(this, 'json')">JSON</button>
  </div>
  <div class="markdown-view">
    {{ caller() }}
  </div>
  <div class="json-view" style="display: none;">
    <pre><code>Input: {{ raw_input | tojson(indent=2) }}
Result: {{ raw_result | tojson(indent=2) }}</code></pre>
  </div>
</div>
{% endmacro %}
```

### Acceptance Criteria
- [ ] All specialized tools have toggle buttons
- [ ] JSON view shows complete raw data
- [ ] Toggle state persists within session
- [ ] Visual consistency with other toggles

---

## Parallel Execution Matrix

| Phase | Dependencies | Can Run In Parallel With |
|-------|--------------|-------------------------|
| Phase 1 | None | - |
| Phase 2 | Phase 1 (variables) | Phase 3, Phase 6 |
| Phase 3 | Phase 1 (variables) | Phase 2, Phase 6 |
| Phase 4 | Phase 1, Phase 3 (toggle macro) | Phase 5 (after 4.4) |
| Phase 5 | Phase 3 (toggle logic) | Phase 4 (after 4.4) |
| Phase 6 | Phase 3 (toggle helper) | Phase 2 |

### Recommended Execution Order

```
Phase 1 (Foundation)
    │
    ├──────────┬──────────┐
    ▼          ▼          ▼
Phase 2    Phase 3    Phase 6
(User)     (Toggle)   (Spec Tools)
    │          │          │
    │          ├──────────┘
    │          ▼
    │      Phase 5
    │      (Copy)
    │          │
    └────┬─────┘
         ▼
     Phase 4
     (Collapse)
```

---

## Files to Modify

### Primary Files

| File | Phases | Changes |
|------|--------|---------|
| `src/claude_code_transcripts/__init__.py` | 1,3,4,5 | CSS variables, JS functions, remove truncation |
| `src/claude_code_transcripts/templates/macros.html` | 2,3,4,6 | User cell, toggle tabs, collapsible macro, tool toggles |

### Secondary Files

| File | Phases | Changes |
|------|--------|---------|
| `src/claude_code_transcripts/templates/page.html` | 4 | Sticky message headers |
| `src/claude_code_transcripts/templates/base.html` | 1 | CSS variable includes |

### Line Number References

> Note: Line numbers are approximate and should be verified before implementation.

**`__init__.py`:**
- CSS_STYLES starts: ~line 50
- JS_CODE starts: ~line 350
- `toggleView` function: ~line 400
- `initTruncation` function: ~line 450
- Copy button injection: ~line 500

**`macros.html`:**
- `user_content` macro: ~line 15-25
- `cell` macro: ~line 50-80
- `view-toggle-btn`: ~line 100
- `write_tool` macro: ~line 150
- `edit_tool` macro: ~line 200
- `bash_tool` macro: ~line 250
- `todo` macro: ~line 300

---

## Test Plan

### Unit Tests

#### Phase 1 Tests
```python
def test_css_variables_defined():
    """Verify all CSS variables are present in output."""

def test_craft_colors_applied():
    """Verify Craft.do palette colors are used."""
```

#### Phase 2 Tests
```python
def test_user_message_wrapped_in_cell():
    """Verify user content uses cell macro."""

def test_long_url_wraps():
    """Verify URLs break correctly in user messages."""
```

#### Phase 3 Tests
```python
def test_toggle_tabs_present():
    """Verify tab-style toggle exists."""

def test_toggle_active_state():
    """Verify active tab styling."""
```

#### Phase 4 Tests
```python
def test_no_truncatable_class():
    """Verify truncatable classes removed."""

def test_details_elements_used():
    """Verify all collapsible content uses details."""

def test_sticky_cascade():
    """Verify z-index hierarchy for sticky elements."""
```

#### Phase 5 Tests
```python
def test_no_duplicate_copy_buttons():
    """Verify only one copy button per cell."""

def test_copy_respects_view_mode():
    """Verify copy captures active view content."""
```

#### Phase 6 Tests
```python
def test_write_tool_has_toggle():
    """Verify Write tool has markdown/json toggle."""

def test_edit_tool_has_toggle():
    """Verify Edit tool has markdown/json toggle."""

def test_bash_tool_has_toggle():
    """Verify Bash tool has markdown/json toggle."""

def test_todo_tool_has_toggle():
    """Verify TodoWrite tool has markdown/json toggle."""
```

### Integration Tests

```python
def test_full_transcript_renders():
    """End-to-end test with sample session."""

def test_keyboard_navigation():
    """Verify accessibility with keyboard only."""

def test_mobile_responsive():
    """Verify layout at mobile viewport widths."""
```

### Manual Testing Checklist

- [ ] Open sample transcript in browser
- [ ] Toggle all Markdown/JSON switches
- [ ] Expand/collapse all cells
- [ ] Verify sticky headers scroll correctly
- [ ] Copy content in both Markdown and JSON modes
- [ ] Test on Chrome, Firefox, Safari
- [ ] Test on mobile viewport (375px)
- [ ] Verify no console errors

---

## Known Risks

### Risk 1: Snapshot Test Failures
**Likelihood:** High
**Impact:** Medium
**Mitigation:** Run `uv run pytest --snapshot-update` after each phase. Review diffs carefully.

### Risk 2: JavaScript Conflicts
**Likelihood:** Medium
**Impact:** High
**Mitigation:** Test toggle and copy functions in isolation. Use unique function names.

### Risk 3: CSS Specificity Wars
**Likelihood:** Medium
**Impact:** Medium
**Mitigation:** Use consistent naming (BEM-style). Avoid `!important`.

### Risk 4: Sticky Header Jank on Scroll
**Likelihood:** Medium
**Impact:** Low
**Mitigation:** Use `will-change: transform` on sticky elements. Test on low-end devices.

### Risk 5: Breaking Existing Macros
**Likelihood:** Low
**Impact:** High
**Mitigation:** Keep backward compatibility. Don't remove macro parameters, only add new ones.

### Risk 6: Mobile Overflow Issues
**Likelihood:** Medium
**Impact:** Medium
**Mitigation:** Test early with `max-width: 100vw; overflow-x: hidden;` on body.

---

## Appendix: Craft.do Reference Tokens

For detailed style reference, here are the exact tokens used in Craft.do's design system:

```css
/* Typography Scale */
--font-size-xs: 0.75rem;    /* 12px */
--font-size-sm: 0.875rem;   /* 14px */
--font-size-base: 1rem;     /* 16px */
--font-size-lg: 1.125rem;   /* 18px */
--font-size-xl: 1.25rem;    /* 20px */

/* Font Weights */
--font-weight-normal: 400;
--font-weight-medium: 500;
--font-weight-semibold: 600;

/* Line Heights */
--line-height-tight: 1.25;
--line-height-normal: 1.5;
--line-height-relaxed: 1.75;

/* Animation Easings */
--ease-out-expo: cubic-bezier(0.16, 1, 0.3, 1);
--ease-in-out: cubic-bezier(0.4, 0, 0.2, 1);
```

---

*Last Updated: 2026-01-01*
*Plan Version: 1.0*
