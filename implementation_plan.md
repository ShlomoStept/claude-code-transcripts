# Implementation Plan: Fix Markdown/JSON Rendering Issues

## Goal Description

Fix rendering issues in the claude-code-transcripts project related to how tool calls display markdown and JSON content. The issues include:
1. Content duplication (markdown and JSON shown simultaneously)
2. Hidden/non-scrollable long content
3. JSON not wrapped in code blocks in Markdown mode
4. No visual difference between modes for tool results
5. Tool call/result sections not collapsible
6. Potential subagent content truncation

## Background Context

The project converts Claude Code session JSON files to HTML transcripts. Tool calls have a view toggle allowing users to switch between "Markdown" and "JSON" views. The current implementation has several rendering bugs where content visibility and formatting don't behave correctly.

### Current Architecture

- **CSS**: Uses `.view-json { display: none; }` and `.view-markdown { display: block; }` with `.show-json` class to toggle visibility
- **JS**: Tab click handlers toggle the `.show-json` class on containers
- **Templates**: `macros.html` defines the HTML structure for tool rendering

## Items Requiring User Review

> [!IMPORTANT]
> **Breaking Change**: The fix for Issue 2.1 (duplication) requires restructuring how `view-markdown` and `view-json` divs are nested within the truncatable wrapper. This may affect any custom CSS users have applied.

> [!IMPORTANT]
> **Design Decision**: For Issue 2.5 (collapsible tool sections), I propose using HTML `<details>` elements similar to the existing cell collapsible behavior. Alternative: custom JS-based accordion.

## Root Cause Analysis

### Issue 2.1: Content Duplication
In `macros.html` line 153 (tool_use macro), both `view-markdown` and `view-json` are inside the same `truncatable-content` div:
```html
<div class="truncatable"><div class="truncatable-content">
  <div class="view-markdown tool-input-rendered">{{ input_markdown_html|safe }}</div>
  <div class="view-json tool-input-rendered">{{ input_json_html|safe }}</div>
</div>
```

The CSS `.view-json { display: none; }` should hide JSON, but the nesting within truncatable causes both to render due to truncatable's overflow handling.

### Issue 2.2: Hidden Long Content
The truncatable CSS sets `max-height: 200px; overflow: hidden;` but the view containers may not properly inherit scrolling when expanded.

### Issue 2.3: JSON Not in Code Block
The `format_json()` function wraps output in `<pre class="json">`, but the markdown view uses `render_json_with_markdown()` which does NOT wrap in a code block.

### Issue 2.4: Tool Result Mode Differentiation
Tool results use the same pre-rendered content for both views. Markdown view should show rendered content, JSON view should show raw JSON.

### Issue 2.5: Non-Collapsible Sections
Tool pairs are wrapped in `<div class="tool-pair">` but not in `<details>` elements.

## Proposed Changes

### [MODIFY] `src/claude_code_transcripts/templates/macros.html`

1. **Fix tool_use macro (line 148-154)**: Move view-markdown and view-json outside truncatable to prevent display conflicts
2. **Fix subagent_tool macro (line 157-187)**: Same restructuring as tool_use
3. **Fix tool_result macro (line 189-193)**: Ensure proper view separation with distinct content
4. **Add collapsible tool_pair macro**: Wrap with `<details>` element for collapse/expand

### [MODIFY] `src/claude_code_transcripts/__init__.py`

1. **Fix `render_json_with_markdown()` function**: Wrap output in `<pre class="json-markdown">` for proper code block styling
2. **Update CSS (line 1784-1787)**: Ensure view toggle CSS properly hides/shows content
3. **Add CSS for `.json-markdown`**: Style JSON in markdown mode with code block appearance
4. **Add CSS for collapsible tool pairs**: Style new `<details>` elements for tool pairs
5. **Review JS view toggle handlers**: Ensure they work with new structure

## Verification Plan

### Automated Tests
- [ ] Run `uv run pytest` - all tests must pass
- [ ] Run `uv run black .` - code must be formatted
- [ ] Snapshot tests may need updating if output changes

### Manual Verification
- [ ] Generate HTML from a sample session with tool calls
- [ ] Verify only one view (markdown OR JSON) displays at a time
- [ ] Test view toggle switches content correctly
- [ ] Verify long content is scrollable when expanded
- [ ] Confirm JSON has code block styling regardless of mode
- [ ] Test tool result shows visual difference between modes
- [ ] Verify tool call/result sections are collapsible
- [ ] Check subagent content is not truncated

## File Change Summary

| File | Action | Description |
|------|--------|-------------|
| `src/claude_code_transcripts/templates/macros.html` | MODIFY | Fix view toggle HTML structure, add collapsible tool pairs |
| `src/claude_code_transcripts/__init__.py` | MODIFY | Fix CSS for view toggles, add JSON-in-markdown styling |
