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
    loglines: SessionLogEntry[]
}

interface SessionLogEntry {
    // Core Identifiers
    uuid: string;                // Unique event identifier
    parentUuid: string | null;   // Linked-list parent pointer for ordering
    sessionId: string;           // Conversation/session grouping ID
    
    // Entry Type (expanded from user/assistant)
    type: "user" | "assistant" | "queue-operation" | "file-history-snapshot";
    
    timestamp: string;           // ISO 8601
    
    // Context Fields (optional)
    cwd?: string;                // Current working directory
    gitBranch?: string;          // Active git branch
    userType?: "external";       // User type indicator
    isSidechain?: boolean;       // Parallel/sidechain conversation flag
    agentId?: string;            // Sub-agent identifier (e.g., "a85b54c")
    slug?: string;               // Human-readable session identifier
    
    // Message Payload (for user/assistant types)
    message?: {
        role: "user" | "assistant";
        content: ContentBlock[];
        model?: string;          // e.g., "claude-opus-4-5-20251101"
        id?: string;             // Message-specific ID
    };
    
    // Snapshot Payload (for file-history-snapshot type)
    snapshot?: {
        messageId: string;
        trackedFileBackups: Record<string, any>;
        timestamp: string;
    };
    
    // Queue Payload (for queue-operation type)
    operation?: "enqueue" | "remove";
    content?: string;            // Queue item content
    
    isCompactSummary?: boolean;  // Session continuation indicator
}
```

**Content Blocks:**
- `text`: `{ type: "text", text: string }` - Markdown text
- `thinking`: `{ type: "thinking", thinking: string, signature: string }` - Internal reasoning with verification signature
- `tool_use`: `{ type: "tool_use", id: string, name: string, input: object }` - Tool invocation
- `tool_result`: `{ type: "tool_result", tool_use_id: string, content: string | ContentBlock[], is_error?: boolean }` - Tool output
- `image`: `{ type: "image", source: { type: "base64", media_type: string, data: string } }` - Base64 image

**Tool Inputs (Complete List):**
- `Write`: `{ file_path: string, content: string }`
- `Edit`: `{ file_path: string, old_string: string, new_string: string, replace_all?: boolean }`
- `Bash`: `{ command: string, description?: string }`
- `TodoWrite`: `{ todos: [{ content: string, status: "pending" | "in_progress" | "completed", activeForm?: string }] }`
- `Task` (Sub-Agent): `{ subagent_type: string, prompt: string, description: string, resume?: string, model?: string, run_in_background?: boolean }`
- `TaskOutput`: `{ task_id: string, block: boolean, timeout: number }`
- `Read`: `{ file_path: string, offset?: number, limit?: number }`
- `Glob`: `{ pattern: string, path?: string }`
- `Grep`: `{ pattern: string, path: string, output_mode?: string, "-n"?: boolean }`

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
- **Tool Input Types:** 9 documented (Write, Edit, Bash, TodoWrite, Task, TaskOutput, Read, Glob, Grep)
- **Entry Types:** 4 (user, assistant, queue-operation, file-history-snapshot)
- **Session Formats:** 2 (JSON, JSONL)
- **CLI Commands:** 4 (local, web, json, all)

---

## Key Findings

1. **Sub-Agent Format Supported:** Artifacts include `agentId` field and `Task` tool for sub-agent tracking. Discovery code excludes `agent-*` files from listing, but the format fully supports sub-agents.
2. **Schemas Now Documented:** Complete specifications including SessionLogEntry, Content Blocks, and 9 Tool Input types (see [SCHEMA_ANALYSIS_REPORT.md](./SCHEMA_ANALYSIS_REPORT.md))
3. **Tool Pairing Works:** Reliable ID-based system (`tool_use.id` ↔ `tool_result.tool_use_id`)
4. **Componentization Needed:** Monolithic structure limits maintainability

## Next Steps

1. ✅ **Analysis Complete** - See [ARCHITECTURE_ANALYSIS.md](./ARCHITECTURE_ANALYSIS.md)
2. **Short Term:** Extract utilities, add validation
3. **Medium Term:** Split discovery and parsing modules
4. **Long Term:** Full componentization, add export formats

---

**For detailed analysis, see:** [ARCHITECTURE_ANALYSIS.md](./ARCHITECTURE_ANALYSIS.md) (1890 lines)
