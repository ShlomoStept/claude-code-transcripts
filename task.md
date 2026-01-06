# Technical Debt Fixes - Task Tracker

## PHASE 2: Critical Issue Fixes

### 2.1 Duplicate Test Method
- [/] Fix duplicate `test_tool_result_with_ansi_codes` (lines 512 and 545)
- [ ] Rename second test to be more specific (uses snapshot_html)
- [ ] Verify both tests run

### 2.2 Global Variable Thread Safety
- [ ] Refactor `_github_repo` global variable for thread safety
- [ ] Use context-based approach instead of module-level state
- [ ] Update all usages

### 2.3 View-Toggle Pattern Duplication
- [ ] Create shared `view_toggle` macro in macros.html
- [ ] Replace 8 duplicated instances with macro call
- [ ] Verify output unchanged

### 2.4 Clipboard API Fallback
- [ ] Add fallback using `document.execCommand('copy')`
- [ ] Test in both copy button implementations
- [ ] Add user-facing error notification

## Verification
- [ ] Run `uv run pytest`
- [ ] Run `uv run black . --check`
- [ ] Verify HTML output unchanged
