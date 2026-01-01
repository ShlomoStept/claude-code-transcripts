# TASKS.md - claude-code-transcripts Upgrade Roadmap

## Overview

This document tracks the implementation status of UX/UI improvements for claude-code-transcripts. It serves as a complete handoff document for new developers.

---

## Phase 1: Foundation (COMPLETED)

### Task Grading Summary

| Task | Score | Status |
|------|-------|--------|
| B.5 ANSI Escape Code Sanitization | 6.75/10 | Completed |
| B.4 Content-Block Array Rendering | 6.75/10 | Completed |
| A.1 Copy Buttons | 6.75/10 | Completed |
| B.2 Syntax Highlighting | 9.25/10 | Completed |

---

### B.5 ANSI Escape Code Sanitization

**Status:** Completed (Score: 6.75/10)
**Priority:** Critical
**Files Modified:** `src/claude_code_transcripts/__init__.py`

**Implementation:**
- `ANSI_ESCAPE_PATTERN` regex at line 56: `r"\x1b\[[0-9;]*[a-zA-Z]"`
- `strip_ansi()` function at lines 59-70
- Applied in `render_content_block()` at line 840

**Known Limitations:**
- Regex catches ~70% of ANSI sequences but misses:
  - Sequences with `?` parameter prefix: `\x1b[?25h`
  - Non-alphabetic final bytes: `\x1b[@`
  - OSC sequences: `\x1b]0;Title\x07`
- Recommended fix: Update regex to `r"\x1b\[[0-9;?]*[@-~]"`

**Test Coverage Gaps:**
- No unit tests for `strip_ansi()` itself
- Missing edge case tests (malformed sequences, empty strings)

---

### B.4 Content-Block Array Rendering

**Status:** Completed (Score: 6.75/10)
**Priority:** Critical
**Files Modified:** `src/claude_code_transcripts/__init__.py`

**Implementation:**
- `is_content_block_array()` function at lines 73-97
- `render_content_block_array()` function at lines 100-126
- Integration in `render_content_block()` at lines 828-837

**Known Limitations:**
- Only handles `text` and `thinking` blocks
- Does NOT render `image` blocks (falls back to JSON)
- Does NOT render `tool_use` blocks

**Test Coverage Gaps:**
- No tests for thinking blocks in arrays
- No tests for empty arrays or invalid JSON
- No tests for image/tool_use blocks

---

### A.1 Copy Buttons (Simplified)

**Status:** Completed (Score: 6.75/10)
**Priority:** High
**Files Modified:** `src/claude_code_transcripts/__init__.py` (CSS + JS)

**Implementation:**
- CSS at lines 1122-1125: `.copy-btn` with hover-reveal
- JavaScript at lines 1210-1236: Dynamic button injection
- Targets: `pre`, `.tool-result .truncatable-content`, `.bash-command`

**Known Limitations:**
- No fallback for browsers without Clipboard API
- No keyboard accessibility (not focusable)
- No ARIA labels for screen readers
- Silent failures (errors only logged to console)

**Test Coverage Gaps:**
- Tests only verify CSS/JS presence, not functionality
- No accessibility testing

---

### B.2 Syntax Highlighting

**Status:** Completed (Score: 9.25/10)
**Priority:** Medium
**Files Modified:**
- `pyproject.toml` (added pygments>=2.17.0)
- `src/claude_code_transcripts/__init__.py` (lines 20-23, 129-155)
- `src/claude_code_transcripts/templates/macros.html`

**Implementation:**
- Imports: `highlight`, `get_lexer_for_filename`, `get_lexer_by_name`, `TextLexer`, `HtmlFormatter`, `ClassNotFound`
- `highlight_code()` function at lines 129-155
- Applied in `render_write_tool()` and `render_edit_tool()`
- Monokai-inspired CSS theme at lines 1080-1110

**Strengths:**
- Excellent error handling with graceful fallback
- Supports 500+ languages via Pygments
- Clean integration with templates

**Minor Gaps:**
- Bash tool commands not syntax highlighted
- No line numbers support

---

## Technical Specifications

### File Architecture

```
src/claude_code_transcripts/
├── __init__.py          # Main implementation (~1300 lines)
│   ├── Lines 1-50       # Imports and constants
│   ├── Lines 51-160     # Utility functions (strip_ansi, highlight_code, etc.)
│   ├── Lines 700-850    # Render functions
│   ├── Lines 1000-1120  # CSS constant
│   └── Lines 1120-1270  # JS constant
└── templates/
    ├── macros.html      # Tool rendering macros
    ├── page.html        # Main page template
    ├── index.html       # Index page template
    ├── base.html        # Base template
    └── search.js        # Client-side search
```

### CSS Guidelines

- Follow existing naming: `.tool-*`, `.file-tool-*`, `.truncatable-*`
- CSS is embedded in `__init__.py` CSS constant
- Use CSS variables: `--user-bg`, `--user-border`, `--text-muted`, etc.

### JavaScript Guidelines

- JavaScript is embedded in `__init__.py` JS constant
- Use vanilla JS, no frameworks
- Use `querySelectorAll()` pattern for batch operations
- Event delegation for dynamic content

### Testing Requirements

Per AGENTS.md:
1. Write failing test first
2. Watch it fail
3. Implement feature
4. Watch test pass
5. Run `uv run black .` to format
6. Commit test + implementation + docs together

---

## Phase 2: Structure (PARTIALLY COMPLETED)

### Task Grading Summary - Phase 2

| Task | Score | Status |
|------|-------|--------|
| A.3 Cell Subsections | 9.0/10 | Completed |
| A.4 Per-Cell Copy Buttons | 7.5/10 | Completed |
| B.6 Tool Markdown Rendering | 8.5/10 | Completed |
| A.2 Metadata Subsection | - | Not Started |
| B.3 Tool Call Headers | - | Not Started |

---

### A.3 Cell Subsections (Input/Output Split)

**Status:** Completed (Score: 9.0/10)
**Priority:** High
**Dependencies:** None

**Implementation:**
- `group_blocks_by_type()` function groups content blocks into thinking/text/tools
- `render_assistant_message()` and `render_assistant_message_with_tool_pairs()` create cell structure
- `cell` macro in macros.html creates collapsible `<details>` wrapper
- CSS for `.cell`, `.thinking-cell`, `.response-cell`, `.tools-cell`
- Per-cell copy buttons with keyboard accessibility and ARIA labels

**Features:**
- Thinking cell: closed by default
- Response cell: open by default
- Tools cell: closed by default, shows count
- Expand/collapse indicator (▶ rotates on open)
- Copy button per cell with "Copied!" feedback

**Test Coverage:**
- `test_group_blocks_by_type`: Verifies block grouping
- `test_cell_structure_in_assistant_message`: Verifies cell HTML structure
- `test_thinking_cell_closed_by_default`: Verifies default state
- `test_response_cell_open_by_default`: Verifies default state
- `test_tools_cell_shows_count`: Verifies tool count display
- `test_cell_has_copy_button`: Verifies copy button presence
- `test_cell_copy_button_aria_label`: Verifies accessibility

---

### A.4 Per-Cell Copy Buttons

**Status:** Completed (Score: 7.5/10)
**Priority:** High
**Dependencies:** A.3

**Implementation:**
- Copy button added to each cell header in `cell` macro (macros.html)
- CSS for `.cell-copy-btn` with hover, focus, and copied states
- JavaScript handler using Clipboard API (`navigator.clipboard.writeText`)
- Keyboard accessibility (Enter/Space key support)
- ARIA labels for screen readers ("Copy Thinking", "Copy Response", "Copy Tool Calls")

**Features:**
- Visual feedback: button text changes to "Copied!" for 2 seconds
- Color feedback: green background on success
- Focus styling: outline for keyboard navigation
- Contextual labels based on cell type

**Known Limitations:**
- No user-facing error notification (errors only logged to console)
- No fallback for browsers without Clipboard API
- Limited functional test coverage

**Test Coverage:**
- `test_cell_has_copy_button`: Verifies button HTML presence
- `test_cell_copy_button_aria_label`: Verifies accessibility labels

**What would improve the score:**
- Add user-facing error notifications
- Add fallback copy method for older browsers
- Add functional tests for click/keyboard behavior

---

### B.6 Tool Markdown Rendering

**Status:** Completed (Score: 8.5/10)
**Priority:** High
**Dependencies:** None

**Implementation:**
- `render_json_with_markdown()` function renders JSON with Markdown in string values
- `render_bash_tool()` renders description as Markdown HTML
- Generic tool handler renders description as Markdown HTML
- Updated macros use `|safe` filter for pre-rendered HTML
- CSS classes for styled JSON: `.json-key`, `.json-string-value`, `.json-number`, etc.

**Features:**
- Tool descriptions render as Markdown (bold, italic, links, code)
- JSON string values render inline Markdown
- Syntax-highlighted JSON keys and types
- Preserves JSON structure visually

**Test Coverage:**
- `test_render_bash_tool_markdown_description`: Verifies Markdown in description
- `test_render_json_with_markdown_simple`: Verifies basic JSON rendering
- `test_render_json_with_markdown_nested`: Verifies nested structures
- `test_render_json_with_markdown_types`: Verifies type preservation

---

### A.2 Metadata Subsection

**Status:** Not Started
**Priority:** High
**Dependencies:** None

**Implementation Details:**
- Add collapsible metadata section to each message
- Use `<details>` tag with `.message-metadata` class

**Data to Include:**
- Timestamp (from message object)
- Working directory (from session context)
- Character count: `len(message_text)`
- Token estimate: `len(message_text) // 4`
- Tool call counts (use existing counting logic)

**Files to Modify:**
- `src/claude_code_transcripts/__init__.py`: Add `render_metadata()` function
- `src/claude_code_transcripts/templates/macros.html`: Add metadata macro

**CSS Classes:**
- `.message-metadata` - container
- `.metadata-item` - row
- `.metadata-label` - label
- `.metadata-value` - value

---

### B.3 Tool Call Headers with Type

**Status:** Not Started
**Priority:** High
**Dependencies:** None

**Implementation Details:**
- Enhanced tool headers showing tool type icon and name
- Add input/output toggle capability
- Add parsed/raw view toggle

**Files to Modify:**
- Tool macros in `macros.html`
- CSS for toggle buttons

---

## Phase 3: Advanced (PENDING)

### A.4 Recursive Nesting Support

**Status:** Not Started
**Priority:** Medium
**Dependencies:** A.3

**Implementation Details:**
- Enable collapsible cells to contain other collapsible cells
- For nested tool calls and subagent tasks
- Add CSS for nested styling with indentation

---

### A.5 Layout Toggle

**Status:** Not Started
**Priority:** Medium
**Dependencies:** A.3

**Implementation Details:**
- Add layout modes: Stacked, Side-by-side, Raw/Parsed toggle
- Use localStorage for persistence
- Add toggle buttons to cell headers

**localStorage Keys:**
- `layout-default`: Global default
- `layout-{cell-id}`: Per-cell override

---

### C.1 Subagent Detection and Grouping

**Status:** Not Started
**Priority:** High
**Dependencies:** None
**Related Issue:** #12 (upstream)

**Implementation Details:**
- Detect subagent JSONL files: `agent-*.jsonl`
- Link subagent sessions to main transcript
- Optional inline expansion

**Files to Modify:**
- Add `find_related_agent_sessions()` function
- Update `generate_html()` to handle subagents

---

## Phase 4: Polish (PENDING)

### C.2 Multi-View Mode

**Status:** Not Started
**Priority:** Medium
**Dependencies:** C.1

**Implementation Details:**
- Add thread view tabs: Chronological, Per-agent view
- Tab component for switching views

---

### C.3 Tool Call/Response Pairing

**Status:** Not Started
**Priority:** Medium
**Dependencies:** B.3

**Implementation Details:**
- Group tool_use with corresponding tool_result
- Match by tool ID
- Collapsible wrapper with nested call/result sections

---

### D.1 Table of Contents

**Status:** Not Started
**Priority:** Medium
**Dependencies:** None

**Implementation Details:**
- Per-page TOC in sidebar or header
- Per-message anchors
- Copy link to message functionality
- Scroll-spy highlighting

**Anchor Naming:** `#msg-{timestamp}` (existing pattern)

---

### D.2 Persistent UI State

**Status:** Not Started
**Priority:** Low
**Dependencies:** All collapsible features

**Implementation Details:**
- Save/restore collapse state per cell
- Save scroll position per page
- Save layout preferences

**localStorage Schema:**
```json
{
  "cellStates": {
    "cell-id": { "collapsed": true, "layout": "stacked" }
  },
  "scrollPosition": { "page-001": 1234 },
  "preferences": { "defaultLayout": "stacked" }
}
```

---

## Task Dependencies Graph

```
Phase 2:
  A.2 (Metadata) → A.3 (Cell Subsections) → B.3 (Tool Headers)

Phase 3:
  A.3 → A.4 (Recursive Nesting)
  C.1 (Subagent Detection) - independent
  A.3 → A.5 (Layout Toggle)

Phase 4:
  C.1 → C.2 (Multi-View)
  B.3 → C.3 (Tool Call/Response Pairing)
  D.1 (Table of Contents) - independent
  All collapsible features → D.2 (Persistent UI State)
```

**Recommended Implementation Order:**
1. Phase 2: A.2, A.3, B.3
2. Phase 3: A.4, C.1, A.5
3. Phase 4: C.2, C.3, D.1, D.2

---

## Critical Test Gaps to Address

### High Priority Tests to Add

| Test Name | What to Test | Why Important |
|-----------|--------------|---------------|
| `test_strip_ansi_edge_cases` | Empty strings, malformed sequences, Unicode | ANSI stripping applied to ALL tool results |
| `test_is_content_block_array_invalid_inputs` | Non-JSON, non-array, missing type fields | Guards parsing logic |
| `test_render_content_block_array_all_block_types` | Images, thinking, mixed arrays | Real sessions have complex arrays |
| `test_highlight_code_lexer_failures` | Unknown extensions, binary content | Must not crash on any input |
| `test_copy_button_accessibility` | ARIA labels, keyboard navigation | Accessibility compliance |

---

## Open Issues Reference

Issue numbers reference upstream: https://github.com/simonw/claude-code-transcripts/issues

| Issue | Title | Phase |
|-------|-------|-------|
| #26 | Pagination links broken on gistpreview | Bug |
| #25 | Lists not separated from intro sentences | Bug |
| #17 | --full option for single HTML page | P4 |
| #12 | Support subagents and search | P3 |

---

## Documentation Gaps Identified

### README.md Missing:
- Search feature documentation
- New features (ANSI sanitization, copy buttons, syntax highlighting)
- URL support for json command
- Black formatting instructions in Development section

### AGENTS.md Missing:
- Initial setup instructions (clone, uv sync)
- Project structure overview
- Specific test running examples
- Contribution guidelines

### pyproject.toml Issues:
- Missing version constraints on 6/7 dependencies
- CI/CD uses pip but project uses uv build backend
- No [tool.pytest] or [tool.black] configuration
