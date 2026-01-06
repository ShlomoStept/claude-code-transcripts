# Markdown/JSON Rendering Fixes - Task Tracker

## Issue 2.1: Content Duplication
- [x] Restructure tool_use macro to separate view-markdown and view-json into independent truncatable containers
- [x] Restructure subagent_tool macro similarly
- [x] Restructure tool_result macro similarly
- **Root cause**: Both views were inside same truncatable-content, causing CSS display conflicts

## Issue 2.2: Hidden/Non-Scrollable Long Content
- [x] Each view now has its own truncatable wrapper
- [x] Show more/less button works independently per view
- **Fix**: Separating views ensures correct height calculation per view

## Issue 2.3: JSON Not in Code Block in Markdown Mode
- [x] Wrap `render_json_with_markdown()` output in `<pre class="json-markdown">`
- [x] Add CSS styling for `.json-markdown` class
- **Fix**: JSON now has consistent dark code block appearance in both modes

## Issue 2.4: Tool Result Mode Differentiation
- [x] Markdown view shows rendered content
- [x] JSON view shows raw JSON in code block
- [x] Both views have distinct visual styling
- **Fix**: Proper separation of views with independent content rendering

## Issue 2.5: Make Tool Call/Result Sections Collapsible
- [x] Convert tool_pair macro to use `<details>` element
- [x] Add tool name to summary label
- [x] Add CSS styling for collapsible tool pairs
- [x] Update Python code to pass tool_name to macro
- [x] Update test to check for new class name

## Issue 1: Subagent Content Truncation (Investigated)
- [x] Confirmed: prompt_preview truncates to 200 chars (by design for preview)
- [x] Confirmed: Full content available in markdown/JSON views via display_input
- **Status**: Working as intended - preview is truncated, full content accessible

## Verification
- [x] Run `uv run pytest` - 157 tests pass
- [x] Run `uv run black .` - code formatted
- [x] Snapshot tests updated to reflect structural changes

## Final Status: COMPLETE

All rendering issues addressed with structural HTML changes.
