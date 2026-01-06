# Feature Proposals

This document outlines proposed features to enhance the usability, maintainability, and scalability of claude-code-transcripts.

---

## Feature: Session Timeline View

### Problem Statement

Currently, session transcripts display messages linearly with pagination, making it difficult to understand the temporal flow and structure of long coding sessions. Users cannot easily see:
- How much time was spent on different parts of the session
- The overall rhythm of human-AI interaction
- Clusters of tool activity versus discussion periods

### Proposed Solution

Add an interactive timeline visualization at the top of each session page that provides a bird's-eye view of the entire session:

1. **Horizontal timeline bar** showing the session duration
2. **Color-coded segments** representing different activity types:
   - Blue: User prompts
   - Green: AI thinking/responses
   - Orange: Tool operations
   - Purple: Subagent activity
3. **Clickable regions** that navigate to specific points in the session
4. **Hover tooltips** showing message previews and timestamps

### Implementation Plan

- **Files to modify:**
  - `src/claude_code_transcripts/templates/page.html` - Add timeline container
  - `src/claude_code_transcripts/templates/macros.html` - Add timeline macro
  - `src/claude_code_transcripts/__init__.py` - Add timeline data extraction function
  - CSS in `__init__.py` - Add timeline styles

- **Key code changes:**
  1. Create `extract_timeline_data(messages)` function that returns:
     - List of activity segments with start/end times
     - Activity type classification
     - Message references for navigation
  2. Add `timeline_bar` Jinja2 macro with SVG-based rendering
  3. Implement CSS for responsive timeline with hover states
  4. Add JavaScript for click-to-navigate and tooltip functionality

- **Estimated effort:** Medium (2-3 days)

### User Impact

- Quick session overview at a glance
- Faster navigation to areas of interest
- Better understanding of session dynamics
- Improved accessibility for reviewing long sessions

### Technical Considerations

- **Performance:** Timeline data should be computed once during HTML generation, not client-side
- **Responsiveness:** Timeline should scale appropriately on mobile devices
- **Accessibility:** Ensure keyboard navigation and screen reader support for timeline elements
- **Pagination:** Timeline should show position within the overall session, even on paginated views

---

## Feature: Diff View for Edit Operations

### Problem Statement

The Edit tool shows `old_string` and `new_string` as separate code blocks, making it difficult to understand exactly what changed. Users must manually compare the two blocks to identify:
- Which lines were added
- Which lines were removed
- Which lines were modified

This is particularly challenging for large edits or subtle changes (e.g., whitespace, punctuation).

### Proposed Solution

Implement a unified diff view for Edit tool operations that clearly highlights:
- Deleted text in red with strikethrough
- Added text in green with highlighting
- Context lines in gray
- Line numbers for both old and new versions

Provide a toggle between:
1. **Unified diff** (default) - Single column with +/- indicators
2. **Side-by-side diff** - Two columns for comparison
3. **Raw view** - Current format showing old/new separately

### Implementation Plan

- **Files to modify:**
  - `src/claude_code_transcripts/__init__.py` - Add diff generation in `render_edit_tool`
  - `src/claude_code_transcripts/templates/macros.html` - Add `edit_tool_diff` macro
  - CSS in `__init__.py` - Add diff styling classes

- **Key code changes:**
  1. Add `generate_unified_diff(old_string, new_string)` function using `difflib`
  2. Create `render_diff_html(diff_lines)` for HTML output with syntax highlighting
  3. Add view toggle buttons similar to Markdown/JSON toggle
  4. Implement side-by-side view with synchronized scrolling

- **Dependencies:**
  - Standard library `difflib` module (no new dependencies)

- **Estimated effort:** Medium (2-3 days)

### User Impact

- Instantly understand what changed in each edit
- Reduce cognitive load when reviewing code modifications
- Catch subtle bugs or unintended changes more easily
- Match the familiar diff experience from Git tools

### Technical Considerations

- **Syntax highlighting:** Diffs should preserve language-specific highlighting from the file extension
- **Large diffs:** Consider collapsing unchanged regions with expandable "N lines hidden" indicators
- **Performance:** Diff computation should handle files up to ~10,000 lines efficiently
- **Word-level diffs:** For single-line changes, show inline word-level differences

---

## Feature: Keyboard Navigation

### Problem Statement

Navigating session transcripts currently requires mouse interaction for:
- Expanding/collapsing cells (thinking, response, tools)
- Switching between Markdown/JSON views
- Moving between messages
- Copying content

This slows down power users and creates accessibility barriers for users who rely on keyboard navigation.

### Proposed Solution

Implement comprehensive keyboard shortcuts for session navigation:

| Shortcut | Action |
|----------|--------|
| `j` / `k` | Move to next/previous message |
| `h` / `l` | Navigate between pages |
| `t` | Toggle thinking cell |
| `r` | Toggle response cell |
| `o` | Toggle tools cell |
| `m` | Switch to Markdown view |
| `d` | Switch to JSON/data view |
| `c` | Copy current cell content |
| `e` | Expand/collapse all |
| `?` | Show keyboard shortcuts help |
| `/` | Focus search input |
| `Esc` | Close modal / unfocus |

### Implementation Plan

- **Files to modify:**
  - `src/claude_code_transcripts/__init__.py` - Add keyboard event handlers in JS
  - `src/claude_code_transcripts/templates/page.html` - Add help modal
  - CSS in `__init__.py` - Add current-focus indicator styles

- **Key code changes:**
  1. Add `initKeyboardNavigation()` JavaScript function
  2. Implement message focus tracking with visual indicator
  3. Create accessible shortcuts help dialog
  4. Add `aria-` attributes for screen reader support
  5. Handle conflicts with browser shortcuts (use modifier keys if needed)

- **Estimated effort:** Low-Medium (1-2 days)

### User Impact

- Faster navigation for keyboard-oriented users
- Improved accessibility compliance (WCAG 2.1)
- Better user experience for reviewing multiple sessions
- Reduced reliance on mouse for common actions

### Technical Considerations

- **Conflict avoidance:** Ensure shortcuts don't conflict with browser defaults or screen readers
- **Discoverability:** Show shortcuts in tooltips and provide a visible help indicator
- **State persistence:** Remember user's navigation position when returning to a page
- **Focus management:** Maintain proper focus order for accessibility
- **Mobile support:** Shortcuts naturally don't apply to touch devices, but UI should remain fully functional

---

## Summary

| Feature | Effort | Impact | Priority |
|---------|--------|--------|----------|
| Session Timeline View | Medium | High | P1 |
| Diff View for Edits | Medium | High | P1 |
| Keyboard Navigation | Low-Medium | Medium | P2 |

These features collectively improve:
- **Usability:** Faster navigation, clearer visualization
- **Maintainability:** Well-structured components, standard patterns
- **Scalability:** Handle large sessions efficiently
