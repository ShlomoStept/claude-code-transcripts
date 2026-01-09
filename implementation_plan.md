# Implementation Plan: Fix Tool Result Markdown Rendering

## Goal

Fix the markdown rendering issue in tool results when content is a Python list of content blocks (not just a JSON string).

## Background

The current implementation correctly handles tool result content when it's a JSON string like:
```json
'[{"type": "text", "text": "## Report\n\n- Item 1"}]'
```

However, it fails to render markdown when the content is already a Python list:
```python
[{"type": "text", "text": "## Report\n\n- Item 1"}]
```

This causes markdown headers, lists, and code blocks to appear as raw text instead of rendered HTML.

## Root Cause Analysis

In `src/claude_code_transcripts/__init__.py`, the `render_content_block()` function has this logic for `tool_result` blocks (lines 1020-1084):

1. **Lines 1033-1044**: When `content` is a string, it checks if it's a content block array using `is_content_block_array()`, parses it, and calls `render_content_block_array()` to render markdown.

2. **Lines 1080-1081**: When `content` is a list, it immediately calls `format_json(content)` WITHOUT checking if it's a content block array that should be rendered as markdown.

```python
# BUG: Line 1080 - doesn't check for content block array
elif isinstance(content, list) or is_json_like(content):
    content_markdown_html = format_json(content)  # Should render as markdown!
```

## Proposed Changes

### File: [src/claude_code_transcripts/__init__.py](file:///Users/sys/Documents/_TEMP_project_to_redesign_new_computer_file_system/testing_projects/testing_claude_code_transcripts/claude-code-transcripts/src/claude_code_transcripts/__init__.py)

#### Change 1: Add `is_content_block_list()` helper function [NEW]

Add a new helper function after `is_content_block_array()` (around line 138) to detect when a Python list is a content block array:

```python
def is_content_block_list(content):
    """Check if content is a Python list of content blocks.

    Args:
        content: Content to check.

    Returns:
        True if content is a list containing dict items with 'type' keys.
    """
    if not isinstance(content, list):
        return False
    if not content:
        return False
    # Check if items look like content blocks
    for item in content:
        if isinstance(item, dict) and "type" in item:
            return True
    return False
```

#### Change 2: Fix tool_result handling for Python list content [MODIFY]

Modify lines 1080-1083 to check for content block lists and render them properly:

**Before:**
```python
elif isinstance(content, list) or is_json_like(content):
    content_markdown_html = format_json(content)
else:
    content_markdown_html = format_json(content)
```

**After:**
```python
elif isinstance(content, list):
    # Check if it's a content block array that should be rendered as markdown
    if is_content_block_list(content):
        rendered = render_content_block_array(content)
        if rendered:
            content_markdown_html = rendered
        else:
            content_markdown_html = format_json(content)
    else:
        content_markdown_html = format_json(content)
else:
    content_markdown_html = format_json(content)
```

### File: [tests/test_generate_html.py](file:///Users/sys/Documents/_TEMP_project_to_redesign_new_computer_file_system/testing_projects/testing_claude_code_transcripts/claude-code-transcripts/tests/test_generate_html.py)

#### Change 3: Add test for Python list content [NEW]

Add a new test after `test_tool_result_content_block_array_with_tool_use`:

```python
def test_tool_result_content_block_list_renders_markdown(self, snapshot_html):
    """Test that tool_result with content as Python list renders markdown properly."""
    block = {
        "type": "tool_result",
        "content": [
            {"type": "text", "text": "## Report\n\n- Item 1\n- Item 2\n\n```python\ncode\n```"}
        ],
        "is_error": False,
    }
    result = render_content_block(block)
    # Should render as HTML, not raw JSON
    assert "<h2>Report</h2>" in result or "<h2>" in result
    assert "<li>Item 1</li>" in result or "<li>" in result
    # Should not show raw JSON structure
    assert '"type": "text"' not in result
    assert result == snapshot_html

def test_tool_result_content_block_list_with_multiple_blocks(self, snapshot_html):
    """Test that tool_result with multiple text blocks renders all as markdown."""
    block = {
        "type": "tool_result",
        "content": [
            {"type": "text", "text": "## First Section\n\n- Point A"},
            {"type": "text", "text": "## Second Section\n\n- Point B"}
        ],
        "is_error": False,
    }
    result = render_content_block(block)
    # Both sections should be rendered
    assert "First Section" in result
    assert "Second Section" in result
    assert "Point A" in result
    assert "Point B" in result
    # Should not show raw JSON
    assert '"type": "text"' not in result
    assert result == snapshot_html
```

#### Change 4: Add test for `is_content_block_list` function [NEW]

Add tests for the new helper function:

```python
class TestIsContentBlockList:
    """Tests for is_content_block_list helper function."""

    def test_empty_list(self):
        assert is_content_block_list([]) == False

    def test_list_with_text_block(self):
        assert is_content_block_list([{"type": "text", "text": "hello"}]) == True

    def test_list_with_image_block(self):
        assert is_content_block_list([{"type": "image", "source": {}}]) == True

    def test_list_with_mixed_blocks(self):
        content = [
            {"type": "text", "text": "hello"},
            {"type": "image", "source": {}}
        ]
        assert is_content_block_list(content) == True

    def test_list_without_type(self):
        assert is_content_block_list([{"key": "value"}]) == False

    def test_not_a_list(self):
        assert is_content_block_list("string") == False
        assert is_content_block_list({"type": "text"}) == False
        assert is_content_block_list(None) == False
```

## Verification Plan

### Automated Tests

1. Run existing tests to ensure no regressions:
   ```bash
   uv run pytest tests/test_generate_html.py -v
   ```

2. Run new tests for the fix:
   ```bash
   uv run pytest tests/test_generate_html.py::TestRenderContentBlock::test_tool_result_content_block_list_renders_markdown -v
   uv run pytest tests/test_generate_html.py::TestIsContentBlockList -v
   ```

3. Update snapshots if needed:
   ```bash
   uv run pytest --snapshot-update
   ```

### Manual Verification

1. Create a test session JSON with tool result content as Python list
2. Generate HTML using `uv run claude-code-transcripts`
3. Verify markdown is rendered (headings, lists, code blocks display correctly)

## Items Requiring Review

> [!IMPORTANT]
> The fix adds a new helper function `is_content_block_list()` which is similar to `is_content_block_array()` but for Python lists instead of JSON strings. Consider if these could be unified.

## Summary of Files to Change

| File | Action | Description |
|------|--------|-------------|
| `src/claude_code_transcripts/__init__.py` | MODIFY | Add `is_content_block_list()` function and fix tool_result handling |
| `tests/test_generate_html.py` | MODIFY | Add tests for new functionality |
| `tests/__snapshots__/*.ambr` | UPDATE | Snapshot updates from new tests |
