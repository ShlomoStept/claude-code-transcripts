# Technical Debt Fixes - Task Tracker

## PHASE 2: Critical Issue Fixes

### 2.1 Duplicate Test Method
- [x] Fix duplicate `test_tool_result_with_ansi_codes` (lines 512 and 545)
- [x] Rename second test to `test_tool_result_with_ansi_codes_snapshot`
- [x] Verify both tests run

### 2.2 Global Variable Thread Safety
- [x] Refactor `_github_repo` global variable for thread safety
- [x] Use contextvars for thread-local storage
- [x] Add `get_github_repo()` and `set_github_repo()` functions
- [x] Update all usages

### 2.3 View-Toggle Pattern Duplication
- [ ] Create shared `view_toggle` macro in macros.html
- [ ] Replace 8 duplicated instances with macro call
- **Status**: Deferred to Phase 3 - risk of output changes

### 2.4 Clipboard API Fallback
- [x] Add `copyToClipboard()` helper function
- [x] Fallback using `document.execCommand('copy')`
- [x] Add user-facing error notification ("Failed" text)
- [x] Test in both copy button implementations

## Verification
- [x] Run `uv run pytest` - 141 tests pass
- [x] Run `uv run black . --check` - 4 files unchanged
- [x] Verify HTML output unchanged (snapshot tests pass)

## Final Status: COMPLETE

All critical issues addressed except view-toggle duplication (documented for Phase 3).
