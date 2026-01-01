# Architecture Analysis - Executive Summary

**Full Documentation:** [ARCHITECTURE_ANALYSIS.md](./ARCHITECTURE_ANALYSIS.md)

---

## Quick Reference

### Core Flows

1. **Session Discovery** (`find_local_sessions()` - Line 347)
   - Scans `~/.claude/projects/**/*.jsonl`
   - Filters: Excludes `agent-*` files, empty sessions
   - Sorts: By modification time (newest first)
   - Returns: List of (Path, summary) tuples

2. **Session Parsing** (`parse_session_file()` - Line 637)
   - Detects format: `.jsonl` vs `.json`
   - Normalizes: Both formats → `{loglines: [...]}`
   - Filters: Keeps only `user` and `assistant` types
   - Output: Standard format for rendering

3. **Message Assembly** (`generate_html()` - Line 2019)
   - Groups: Messages by user prompts
   - Pairs: Tool calls with results via ID lookup
   - Renders: HTML with specialized tool renderers
   - Paginates: 5 conversations per page

### Tool Linking System

**Mechanism:** ID-based pairing
- Tool calls have unique `id` field (e.g., `"toolu_001"`)
- Tool results reference via `tool_use_id` field
- Lookup table built: `{tool_id: tool_result}`
- Pairing during render prevents duplicate display

**Example:**
```json
// Tool call (in assistant message)
{"type": "tool_use", "id": "toolu_001", "name": "Write", "input": {...}}

// Tool result (in user message)
{"type": "tool_result", "tool_use_id": "toolu_001", "content": "..."}
```

### Key Object Schemas

**Session (Normalized):**
```typescript
{
    loglines: [
        {
            type: "user" | "assistant",
            timestamp: string,  // ISO 8601
            message: {
                role: string,
                content: string | ContentBlock[]
            },
            isCompactSummary?: boolean
        }
    ]
}
```

**Content Blocks:**
- `text` - Markdown text
- `thinking` - Internal reasoning
- `tool_use` - Tool invocation
- `tool_result` - Tool output
- `image` - Base64 image

**Tool Inputs:**
- `Write`: `{file_path, content}`
- `Edit`: `{file_path, old_string, new_string, replace_all?}`
- `Bash`: `{command, description?}`
- `TodoWrite`: `{todos: [{content, status, activeForm?}]}`

### Module Boundaries (Current Monolith)

**Current:** Single 2994-line file mixing all concerns

**Proposed Structure:**
```
discovery/    - Session finding
parsing/      - File reading & normalization
processing/   - Grouping, pairing, analysis
rendering/    - HTML generation
output/       - File writing, pagination
cli/          - Command handlers
utils/        - Shared utilities
assets/       - CSS/JS
```

### Quick Stats

- **Lines of Code:** 2994 (single file)
- **Content Block Types:** 5 (text, thinking, tool_use, tool_result, image)
- **Tool Renderers:** 4 specialized (Write, Edit, Bash, TodoWrite) + 1 generic
- **Session Formats:** 2 (JSON, JSONL)
- **CLI Commands:** 4 (local, web, json, all)

---

## Key Findings

1. **No Sub-Agent Support:** Current code explicitly excludes agent files
2. **Schemas Implicit:** No formal definitions, inferred from code
3. **Tool Pairing Works:** Reliable ID-based system
4. **Componentization Needed:** Monolithic structure limits maintainability

## Next Steps

1. ✅ **Analysis Complete** - See [ARCHITECTURE_ANALYSIS.md](./ARCHITECTURE_ANALYSIS.md)
2. **Short Term:** Extract utilities, add validation
3. **Medium Term:** Split discovery and parsing modules
4. **Long Term:** Full componentization, add export formats

---

**For detailed analysis, see:** [ARCHITECTURE_ANALYSIS.md](./ARCHITECTURE_ANALYSIS.md) (1890 lines)
