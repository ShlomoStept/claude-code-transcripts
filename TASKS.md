# TASKS.md - claude-code-transcripts Upgrade Roadmap

## Overview

This document tracks the implementation status of UX/UI improvements for claude-code-transcripts.

---

## Phase 1: Foundation (COMPLETED)

### B.5 ANSI Escape Code Sanitization ✅
**Status:** Completed
**Priority:** Critical
**Files Modified:** `src/claude_code_transcripts/__init__.py`

- Added `ANSI_ESCAPE_PATTERN` regex
- Added `strip_ansi()` function
- Applied to tool_result content before rendering

### B.4 Content-Block Array Rendering ✅
**Status:** Completed
**Priority:** Critical
**Files Modified:** `src/claude_code_transcripts/__init__.py`

- Added `is_content_block_array()` detection function
- Added `render_content_block_array()` for proper rendering
- Tool results with JSON content-block arrays now render as formatted text

### A.1 Copy Buttons (Simplified) ✅
**Status:** Completed
**Priority:** High
**Files Modified:** `src/claude_code_transcripts/__init__.py` (CSS + JS)

- Added `.copy-btn` CSS with hover-reveal animation
- Added JavaScript for dynamic copy button injection
- Supports clipboard API with "Copied!" feedback

### B.2 Syntax Highlighting ✅
**Status:** Completed
**Priority:** Medium
**Files Modified:**
- `pyproject.toml` (added pygments)
- `src/claude_code_transcripts/__init__.py`
- `src/claude_code_transcripts/templates/macros.html`

- Added Pygments integration with `highlight_code()` function
- Added Monokai-inspired dark theme CSS
- Applied to Write and Edit tool content

---

## Phase 2: Structure (PENDING)

### A.2 Metadata Subsection
**Status:** Not Started
**Priority:** High

Add collapsible metadata section to each message containing:
- Timestamp, session ID, working directory, git branch
- Character count, token estimate, tool call counts
- Optional user notes and tags

### A.3 Cell Subsections (Input/Output Split)
**Status:** Not Started
**Priority:** High

Split messages into collapsible subsections:
- Thinking block
- Response text
- Tool calls (grouped)

### B.3 Tool Call Headers with Type
**Status:** Not Started
**Priority:** High

Enhanced tool headers showing:
- Tool type icon and name
- Input/output toggle
- Parsed/raw view toggle

---

## Phase 3: Advanced (PENDING)

### A.4 Recursive Nesting Support
**Status:** Not Started
**Priority:** Medium

Enable collapsible cells to contain other collapsible cells for:
- Nested tool calls
- Subagent tasks

### A.5 Layout Toggle
**Status:** Not Started
**Priority:** Medium

Add layout modes:
- Stacked (default)
- Side-by-side
- Raw/Parsed toggle

### C.1 Subagent Detection and Grouping
**Status:** Not Started
**Priority:** High

From Issue #12:
- Detect subagent JSONL files (agent-*.jsonl)
- Link subagent sessions to main transcript
- Optional inline expansion

---

## Phase 4: Polish (PENDING)

### C.2 Multi-View Mode
**Status:** Not Started
**Priority:** Medium

Add thread view tabs:
- Chronological (all messages)
- Per-agent view

### C.3 Tool Call/Response Pairing
**Status:** Not Started
**Priority:** Medium

Group tool_use with corresponding tool_result:
- Collapsible wrapper
- Nested call/result sections

### D.1 Table of Contents
**Status:** Not Started
**Priority:** Medium

Add navigation features:
- Per-page TOC
- Per-message anchors
- Copy link to message

### D.2 Persistent UI State
**Status:** Not Started
**Priority:** Low

Save/restore:
- Collapse state per cell
- Scroll position per page
- Layout preferences

---

## Open Issues Reference

| Issue | Title | Phase |
|-------|-------|-------|
| #26 | Pagination links broken on gistpreview | Bug |
| #25 | Lists not separated from intro sentences | Bug |
| #17 | --full option for single HTML page | P4 |
| #12 | Support subagents and search | P3 |

## Open PRs Reference

| PR | Title | Status |
|----|-------|--------|
| #24 | Display custom title from /rename | Ready |
| #23 | Code view of transcripts | Draft |
| #22 | Gemini CLI support | Ready |
| #21 | Add copy buttons to result boxes | Superseded |
| #19 | Nice rendering for slash commands | Ready |
