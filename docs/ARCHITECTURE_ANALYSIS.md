# Claude Code Transcripts - Comprehensive Architecture Analysis

**Date:** 2026-01-01  
**Version:** 0.4  
**Analyzed By:** Repository Analysis Agent

---

## Table of Contents

1. [Core Flows Executed in Local Claude Code Session Processes](#1-core-flows-executed-in-local-claude-code-session-processes)
2. [Connecting the Main Agent to Sub-Agent Activity](#2-connecting-the-main-agent-to-sub-agent-activity)
3. [Object Schemas (Complete Specification)](#3-object-schemas-complete-specification)
4. [Proposed Componentization Plan](#4-proposed-componentization-plan)

---

## 1. Core Flows Executed in Local Claude Code Session Processes

### 1.1 Session Discovery Flow

**Purpose:** Locate and enumerate available Claude Code session files on the local filesystem.

**Entry Points:**
- `find_local_sessions(folder, limit=10)` - Line 347 in `__init__.py`
- `local_cmd()` CLI command - Line 2268

**Step-by-Step Process:**

1. **Initialize Search**
   - Default folder: `~/.claude/projects`
   - Input: folder path, limit (default 10)
   - Output: List of (Path, summary) tuples

2. **Recursive File Discovery**
   ```python
   for f in folder.glob("**/*.jsonl"):
   ```
   - Recursively scans all subdirectories
   - Filters for `.jsonl` files only
   - Excludes files starting with `agent-` (agent session files)

3. **Session Filtering**
   - Calls `get_session_summary(f)` for each file
   - Skips sessions with:
     - Summary text `"warmup"` (case-insensitive)
     - Summary text `"(no summary)"`
   - Purpose: Exclude empty/test sessions

4. **Summary Extraction** (`get_session_summary()` - Line 272)
   - For JSONL files: Calls `_get_jsonl_summary()`
   - For JSON files: Extracts first user message
   - Priority order for JSONL:
     1. Look for `type: "summary"` entry with `summary` field
     2. Look for first non-meta user message with content
   - Truncates to max_length (default 200 chars)

5. **Sorting and Limiting**
   ```python
   results.sort(key=lambda x: x[0].stat().st_mtime, reverse=True)
   return results[:limit]
   ```
   - Sorts by modification time (most recent first)
   - Returns top N results based on limit

**Data Flow:**
```
User CLI Input
    ↓
~/.claude/projects folder
    ↓
Glob **/*.jsonl files
    ↓
Filter out agent-* files
    ↓
Extract summaries (skip boring)
    ↓
Sort by mtime (descending)
    ↓
Return top N sessions
```

---

### 1.2 Session Parsing Flow

**Purpose:** Read session files (JSON or JSONL) and normalize them into a standard internal format.

**Entry Points:**
- `parse_session_file(filepath)` - Line 637
- `generate_html(json_path, output_dir, github_repo=None)` - Line 2019

**Step-by-Step Process:**

1. **Format Detection**
   ```python
   if filepath.suffix == ".jsonl":
       return _parse_jsonl_file(filepath)
   else:
       return json.load(f)
   ```
   - Determines format based on file extension
   - `.jsonl` → JSONL format (one JSON object per line)
   - `.json` or other → Standard JSON format

2. **JSON Format Parsing** (Direct Load)
   - Loads entire file as single JSON object
   - Expected structure:
     ```json
     {
       "loglines": [
         {"type": "user|assistant", "timestamp": "...", "message": {...}},
         ...
       ]
     }
     ```
   - No transformation needed - already in standard format

3. **JSONL Format Parsing** (`_parse_jsonl_file()` - Line 653)
   
   **Line-by-Line Processing:**
   ```python
   for line in f:
       line = line.strip()
       if not line:
           continue
       obj = json.loads(line)
   ```
   
   **Entry Filtering:**
   - Only processes entries where `type` is `"user"` or `"assistant"`
   - Skips entries like:
     - `type: "summary"` (metadata)
     - `type: "meta"` (system messages)
     - Any other non-message types
   
   **Normalization:**
   ```python
   entry = {
       "type": entry_type,           # "user" or "assistant"
       "timestamp": obj.get("timestamp", ""),
       "message": obj.get("message", {}),
   }
   if obj.get("isCompactSummary"):
       entry["isCompactSummary"] = True
   ```
   
   **Output Structure:**
   ```python
   return {"loglines": loglines}
   ```

4. **Return Normalized Data**
   - Both formats return dict with `"loglines"` key
   - Each logline contains:
     - `type`: "user" or "assistant"
     - `timestamp`: ISO 8601 string
     - `message`: Message object (see schemas section)
     - `isCompactSummary`: Optional boolean flag

**Data Flow:**
```
Session File (JSON/JSONL)
    ↓
Format Detection (.jsonl vs .json)
    ↓
├─ JSON: Direct load
└─ JSONL: Parse line-by-line
    ↓
    Filter (keep user/assistant only)
    ↓
    Normalize to standard structure
    ↓
Standard Format: {loglines: [...]}
```

---

### 1.3 Message Assembly and Ordering Flow

**Purpose:** Take parsed loglines and assemble them into a structured conversation with proper ordering, tool pairing, and pagination.

**Entry Points:**
- `generate_html()` - Line 2019 (main orchestrator)

**Step-by-Step Process:**

#### Phase 1: Conversation Grouping (Lines 2042-2073)

**Purpose:** Group messages into conversations based on user prompts.

```python
conversations = []
current_conv = None

for entry in loglines:
    log_type = entry.get("type")
    timestamp = entry.get("timestamp", "")
    is_compact_summary = entry.get("isCompactSummary", False)
    message_data = entry.get("message", {})
```

**Logic:**
1. **Detect User Prompts:**
   - Check if `log_type == "user"`
   - Extract text from content using `extract_text_from_content(content)`
   - If text exists → This is a conversation start

2. **Start New Conversation:**
   ```python
   if is_user_prompt:
       if current_conv:
           conversations.append(current_conv)
       current_conv = {
           "user_text": user_text,
           "timestamp": timestamp,
           "messages": [(log_type, message_json, timestamp)],
           "is_continuation": bool(is_compact_summary),
       }
   ```

3. **Append to Current Conversation:**
   ```python
   elif current_conv:
       current_conv["messages"].append((log_type, message_json, timestamp))
   ```

**Conversation Structure:**
```python
{
    "user_text": "Original user prompt",
    "timestamp": "ISO timestamp of prompt",
    "messages": [
        (log_type, message_json, timestamp),
        ...
    ],
    "is_continuation": bool  # From isCompactSummary
}
```

#### Phase 2: Tool Pairing (Lines 2092-2105)

**Purpose:** Link `tool_use` blocks with corresponding `tool_result` blocks.

```python
tool_result_lookup = {}
for log_type, message_data, _ in parsed_messages:
    content = message_data.get("content", [])
    for block in content:
        if block.get("type") == "tool_result" and block.get("tool_use_id"):
            tool_id = block.get("tool_use_id")
            if tool_id not in tool_result_lookup:
                tool_result_lookup[tool_id] = block
```

**Key Mechanism:**
- Builds a dictionary: `{tool_use_id: tool_result_block}`
- Used during rendering to pair tool calls with their results
- Allows removal of duplicate tool results from user messages

#### Phase 3: Message Rendering with Tool Pairing (Lines 2107-2120)

**Purpose:** Render each message with proper tool call/result association.

```python
paired_tool_ids = set()
for log_type, message_data, timestamp in parsed_messages:
    msg_html = render_message_with_tool_pairs(
        log_type,
        message_data,
        timestamp,
        tool_result_lookup,
        paired_tool_ids,
    )
```

**Rendering Logic:**

1. **Assistant Messages** (`render_assistant_message_with_tool_pairs` - Line 1126):
   - Groups content blocks by type: thinking, text, tools
   - For each `tool_use` block:
     - Looks up matching `tool_result` in `tool_result_lookup`
     - If found: Renders as paired unit, adds to `paired_tool_ids`
     - If not found: Renders tool_use alone

2. **User Messages** (`render_user_message_content_with_tool_pairs` - Line 1090):
   - Filters out `tool_result` blocks already in `paired_tool_ids`
   - Prevents duplicate rendering of tool results

3. **Content Block Rendering** (`render_content_block` - Line 937):
   - Dispatches to specialized renderers based on block type:
     - `text` → Markdown rendering
     - `thinking` → Styled thinking block
     - `tool_use` → Tool-specific renderer (Write, Edit, Bash, etc.)
     - `tool_result` → Result display with JSON/Markdown toggle
     - `image` → Base64 image display

#### Phase 4: Pagination (Lines 2078-2134)

**Purpose:** Split conversations into pages.

```python
PROMPTS_PER_PAGE = 5  # Constant at line 50
total_pages = (total_convs + PROMPTS_PER_PAGE - 1) // PROMPTS_PER_PAGE

for page_num in range(1, total_pages + 1):
    start_idx = (page_num - 1) * PROMPTS_PER_PAGE
    end_idx = min(start_idx + PROMPTS_PER_PAGE, total_convs)
    page_convs = conversations[start_idx:end_idx]
```

**Output:**
- `page-001.html`, `page-002.html`, etc.
- Each page contains up to 5 conversations
- Pagination links generated via `generate_pagination_html()`

**Data Flow:**
```
Parsed Loglines
    ↓
Group by User Prompts
    ↓
Conversations List
    ↓
For Each Conversation:
    ├─ Build Tool Result Lookup
    ├─ Track Paired Tool IDs
    └─ Render Messages with Pairing
    ↓
Paginate (5 convs per page)
    ↓
Generate HTML Files
```

---

### 1.4 Complete Ordered Message History Derivation

**How the System Determines Complete Message History:**

1. **Temporal Ordering:**
   - All entries have `timestamp` field (ISO 8601)
   - Loglines array maintains chronological order from file
   - No explicit sorting needed - trust file order

2. **Message Continuity:**
   - Turn-based model: user → assistant → user → assistant
   - Tool results appear as user messages
   - Conversation boundaries marked by user text prompts

3. **Tool Call Sequencing:**
   - Tool calls identified by unique IDs (e.g., `"toolu_001"`)
   - Tool results reference the call via `tool_use_id` field
   - Pairing happens post-hoc during rendering using ID lookup

4. **Session Continuations:**
   - Marked by `isCompactSummary: true` flag
   - Indicates session was resumed/continued
   - Rendered as collapsible summary in UI

**Key Data Structures:**

```python
# Raw logline from file
{
    "type": "assistant",
    "timestamp": "2025-12-24T10:00:05.000Z",
    "message": {
        "role": "assistant",
        "content": [
            {"type": "text", "text": "..."},
            {"type": "tool_use", "id": "toolu_001", "name": "Write", "input": {...}}
        ]
    }
}

# Grouped conversation
{
    "user_text": "Create a hello world function",
    "timestamp": "2025-12-24T10:00:00.000Z",
    "messages": [
        ("user", message_json_1, timestamp_1),
        ("assistant", message_json_2, timestamp_2),
        ("user", message_json_3, timestamp_3),
        ...
    ],
    "is_continuation": False
}
```

---

## 2. Connecting the Main Agent to Sub-Agent Activity

### 2.1 Current State: No Sub-Agent Tracking

**Important Finding:** The current codebase does **not** have explicit sub-agent tracking or hierarchical agent relationships.

**Evidence:**
1. Session filtering explicitly excludes agent files:
   ```python
   # Line 359 in find_local_sessions()
   if f.name.startswith("agent-"):
       continue
   ```

2. No agent ID or parent-child relationships in message schemas

3. All messages treated as single-agent conversation

### 2.2 Tool Call and Tool Response Linking

**Mechanism:** ID-based pairing system

#### Tool Call Structure
```python
{
    "type": "tool_use",
    "id": "toolu_write_001",      # Unique identifier
    "name": "Write",               # Tool name
    "input": {                     # Tool-specific parameters
        "file_path": "/path/to/file",
        "content": "..."
    }
}
```

#### Tool Response Structure
```python
{
    "type": "tool_result",
    "tool_use_id": "toolu_write_001",  # References tool call ID
    "content": "File written successfully",
    "is_error": false
}
```

#### Pairing Algorithm (Line 2092-2105)

**Step 1: Build Lookup Table**
```python
tool_result_lookup = {}
for log_type, message_data, _ in parsed_messages:
    content = message_data.get("content", [])
    for block in content:
        if block.get("type") == "tool_result" and block.get("tool_use_id"):
            tool_id = block.get("tool_use_id")
            tool_result_lookup[tool_id] = block
```

**Step 2: Pair During Rendering**
```python
paired_tool_ids = set()
for block in groups["tools"]:
    if block.get("type") == "tool_use":
        tool_id = block.get("id", "")
        tool_result = tool_result_lookup.get(tool_id)
        if tool_result:
            paired_tool_ids.add(tool_id)
            # Render as paired unit
            tool_parts.append(_macros.tool_pair(tool_use_html, tool_result_html))
```

**Step 3: Filter User Messages**
```python
def filter_tool_result_blocks(content, paired_tool_ids):
    filtered = []
    for block in content:
        if (block.get("type") == "tool_result" 
            and block.get("tool_use_id") in paired_tool_ids):
            continue  # Skip already-paired results
        filtered.append(block)
    return filtered
```

### 2.3 Message Role Attribution

**How Messages Are Attributed to Agent/User:**

1. **Top-Level Type Field:**
   ```python
   log_type = entry.get("type")  # "user" or "assistant"
   ```
   - Directly from logline entry
   - No ambiguity - explicit in data

2. **Message Role Field (Redundant):**
   ```python
   message_data.get("role")  # Also "user" or "assistant"
   ```
   - Inside message object
   - Consistent with top-level type

3. **Rendering Classification:**
   ```python
   if log_type == "user":
       if is_tool_result_message(message_data):
           role_class, role_label = "tool-reply", "Tool reply"
       else:
           role_class, role_label = "user", "User"
   elif log_type == "assistant":
       role_class, role_label = "assistant", "Assistant"
   ```
   - Special handling for tool-result-only messages
   - Displayed as "Tool reply" instead of "User"

### 2.4 Event Sequencing

**Temporal Ordering:**
- All events have ISO 8601 timestamps
- File order preserves chronological sequence
- No re-ordering or sorting performed

**Turn-Based Flow:**
```
User Prompt
    ↓
Assistant Response (with tool calls)
    ↓
Tool Results (as user messages)
    ↓
Assistant Response (processing results)
    ↓
Repeat...
```

### 2.5 Hypothetical Sub-Agent Support

**If sub-agents were to be added, the system would need:**

1. **Agent Identifier Field:**
   ```json
   {
       "type": "assistant",
       "agentId": "main|sub-agent-123",
       "parentAgentId": "main",  // Optional
       "message": {...}
   }
   ```

2. **Agent Hierarchy Tracking:**
   ```python
   agent_hierarchy = {
       "main": {
           "children": ["sub-agent-123", "sub-agent-456"],
           "messages": [...]
       }
   }
   ```

3. **Visual Distinction in UI:**
   - Different colors for sub-agents
   - Indentation for nested agents
   - Agent name labels

**Current Code Impact:**
- Tool pairing system would work unchanged
- Rendering would need agent-aware styling
- Session discovery would need to handle agent files

---

## 3. Object Schemas (Complete Specification)

### 3.1 Schema Organization

Schemas are organized by:
1. **Source Format:** JSON vs JSONL
2. **Object Type:** Session, LogLine, Message, ContentBlock
3. **Content Block Variants:** text, thinking, tool_use, tool_result, image

### 3.2 Top-Level Session Schema

#### JSON Format
```typescript
interface JSONSession {
    loglines: LogLine[];
}
```

**Source:** Direct from `.json` files  
**File:** `__init__.py` line 649  
**Example:** `tests/sample_session.json`

#### JSONL Format (Raw)
```typescript
// Multiple JSON objects, one per line
// Summary line (metadata)
interface SummaryLine {
    type: "summary";
    summary: string;
    leafUuid?: string;
}

// Message lines
interface MessageLine {
    type: "user" | "assistant";
    timestamp: string;  // ISO 8601
    sessionId?: string;
    cwd?: string;
    gitBranch?: string;
    message: Message;
    uuid?: string;
    isMeta?: boolean;
    isCompactSummary?: boolean;
}
```

**Source:** One JSON object per line in `.jsonl` files  
**File:** `__init__.py` line 653-685  
**Example:** `tests/sample_session.jsonl`

#### JSONL Format (Normalized)
```typescript
interface NormalizedJSONLSession {
    loglines: LogLine[];
}
```

**Transformation:** Lines 671-681 in `_parse_jsonl_file()`  
**Purpose:** Convert JSONL to same structure as JSON for uniform processing

---

### 3.3 LogLine Schema

```typescript
interface LogLine {
    type: "user" | "assistant";
    timestamp: string;  // ISO 8601 format, e.g., "2025-12-24T10:00:00.000Z"
    message: Message;
    isCompactSummary?: boolean;  // Optional, indicates session continuation
}
```

**Source:** Standardized format after parsing  
**Used By:** All rendering functions  
**File:** `__init__.py` lines 2044-2073

**Field Details:**

- **`type`**: Role of the message sender
  - Values: `"user"` | `"assistant"`
  - Determines rendering style and icon

- **`timestamp`**: When the message was created
  - Format: ISO 8601 string
  - Used for: Sorting, display, message IDs
  - Example: `"2025-12-24T10:00:00.000Z"`

- **`message`**: The actual message content (see Message schema)

- **`isCompactSummary`**: Indicates session continuation/resume
  - Type: boolean (optional)
  - When true: Rendered as collapsible summary
  - Default: false (omitted)

---

### 3.4 Message Schema

```typescript
interface Message {
    role: "user" | "assistant";  // Redundant with LogLine.type
    content: string | ContentBlock[];
}
```

**Source:** Inside LogLine.message field  
**Format Variants:** String (legacy) vs Array (current)

#### Variant 1: String Content (Legacy)
```json
{
    "role": "user",
    "content": "Create a simple function"
}
```

**Handling:** Lines 1047-1051 in `render_user_message_content()`

#### Variant 2: Array Content (Current)
```json
{
    "role": "assistant",
    "content": [
        {"type": "text", "text": "I'll create that for you."},
        {"type": "tool_use", "id": "toolu_001", "name": "Write", "input": {...}}
    ]
}
```

**Handling:** Lines 1056-1062 in `render_user_message_content()`

---

### 3.5 ContentBlock Schemas

All content blocks share a base structure:
```typescript
interface BaseContentBlock {
    type: string;  // Discriminator field
}
```

#### 3.5.1 Text Block
```typescript
interface TextBlock extends BaseContentBlock {
    type: "text";
    text: string;  // Markdown-formatted text
}
```

**Example:**
```json
{
    "type": "text",
    "text": "I'll create a simple Python function for you."
}
```

**Rendering:** Lines 949-951  
**Renderer:** `_macros.assistant_text(content_html)`  
**Processing:** Markdown → HTML via `render_markdown_text()`

---

#### 3.5.2 Thinking Block
```typescript
interface ThinkingBlock extends BaseContentBlock {
    type: "thinking";
    thinking: string;  // Markdown-formatted internal reasoning
}
```

**Example:**
```json
{
    "type": "thinking",
    "thinking": "The user wants a simple addition function. I should:\n1. Create the function\n2. Add a basic test"
}
```

**Rendering:** Lines 946-948  
**Renderer:** `_macros.thinking(content_html)`  
**Styling:** Closed by default, yellow background

---

#### 3.5.3 Tool Use Block
```typescript
interface ToolUseBlock extends BaseContentBlock {
    type: "tool_use";
    id: string;           // Unique identifier, e.g., "toolu_write_001"
    name: string;         // Tool name, e.g., "Write", "Bash", "Edit"
    input: ToolInput;     // Tool-specific input object
}
```

**Example:**
```json
{
    "type": "tool_use",
    "id": "toolu_write_001",
    "name": "Write",
    "input": {
        "file_path": "/project/hello.py",
        "content": "def hello():\n    return 'Hello, World!'\n"
    }
}
```

**Rendering:** Lines 952-977  
**Dispatch:** Tool-specific renderers (Write, Edit, Bash, etc.)

---

#### 3.5.4 Tool Result Block
```typescript
interface ToolResultBlock extends BaseContentBlock {
    type: "tool_result";
    tool_use_id: string;  // References ToolUseBlock.id
    content: string | ContentBlock[];  // Result content
    is_error: boolean;    // Whether the tool execution failed
}
```

**Example Success:**
```json
{
    "type": "tool_result",
    "tool_use_id": "toolu_write_001",
    "content": "File written successfully",
    "is_error": false
}
```

**Example Error:**
```json
{
    "type": "tool_result",
    "tool_use_id": "toolu_bash_005",
    "content": "Command failed: Permission denied",
    "is_error": true
}
```

**Example Nested Content:**
```json
{
    "type": "tool_result",
    "tool_use_id": "toolu_003",
    "content": [
        {"type": "text", "text": "Multiple items found:"},
        {"type": "text", "text": "- Item 1\n- Item 2"}
    ],
    "is_error": false
}
```

**Rendering:** Lines 978-1042  
**Features:**
- ANSI escape code stripping
- Commit detection and card rendering
- JSON/Markdown dual view toggle
- Error styling

---

#### 3.5.5 Image Block
```typescript
interface ImageBlock extends BaseContentBlock {
    type: "image";
    source: {
        type: "base64";
        media_type: string;  // e.g., "image/png", "image/jpeg"
        data: string;        // Base64-encoded image data
    };
}
```

**Example:**
```json
{
    "type": "image",
    "source": {
        "type": "base64",
        "media_type": "image/png",
        "data": "iVBORw0KGgoAAAANSUhEUgAAAAUA..."
    }
}
```

**Rendering:** Lines 941-945  
**Renderer:** `_macros.image_block(media_type, data)`  
**Output:** `<img>` tag with data URI

---

### 3.6 Tool Input Schemas

Each tool has a unique input schema.

#### 3.6.1 Write Tool Input
```typescript
interface WriteToolInput {
    file_path: string;  // Absolute or relative path
    content: string;    // Full file content
}
```

**Example:**
```json
{
    "file_path": "/project/math_utils.py",
    "content": "def add(a: int, b: int) -> int:\n    return a + b\n"
}
```

**Rendering:** Lines 898-905  
**Features:**
- Syntax highlighting based on file extension
- Truncatable long content
- JSON view toggle

---

#### 3.6.2 Edit Tool Input
```typescript
interface EditToolInput {
    file_path: string;      // File to edit
    old_string: string;     // Text to replace
    new_string: string;     // Replacement text
    replace_all?: boolean;  // Optional: replace all occurrences
}
```

**Example:**
```json
{
    "file_path": "/project/math_utils.py",
    "old_string": "def add(a, b):",
    "new_string": "def add(a: int, b: int) -> int:",
    "replace_all": false
}
```

**Rendering:** Lines 908-925  
**Features:**
- Diff-style old/new display
- Syntax highlighting
- Replace all indicator

---

#### 3.6.3 Bash Tool Input
```typescript
interface BashToolInput {
    command: string;      // Shell command to execute
    description?: string; // Optional: human-readable description
    mode?: "sync" | "async" | "detached";  // Execution mode
    initial_wait?: number;  // Seconds to wait for output
}
```

**Example:**
```json
{
    "command": "pytest tests/ -v",
    "description": "Run tests with verbose output"
}
```

**Rendering:** Lines 928-934  
**Features:**
- Command as plain text (not highlighted)
- Description as Markdown

---

#### 3.6.4 TodoWrite Tool Input
```typescript
interface TodoWriteToolInput {
    todos: TodoItem[];
}

interface TodoItem {
    content: string;           // Todo text
    status: "pending" | "in_progress" | "completed";
    activeForm?: string;       // Optional: present progressive description
}
```

**Example:**
```json
{
    "todos": [
        {
            "content": "Create add function",
            "status": "completed",
            "activeForm": "Creating add function"
        },
        {
            "content": "Write tests",
            "status": "in_progress",
            "activeForm": "Writing tests"
        },
        {
            "content": "Push to remote",
            "status": "pending"
        }
    ]
}
```

**Rendering:** Lines 890-895  
**Renderer:** `_macros.todo_list(todos, input_json_html, tool_id)`  
**Features:**
- Status icons (✓, ○, ◐)
- Color-coded by status
- Strikethrough for completed

---

#### 3.6.5 Generic Tool Input
```typescript
interface GenericToolInput {
    description?: string;  // Optional description field
    [key: string]: any;    // Any other tool-specific fields
}
```

**Used For:** Tools without specialized renderers:
- Glob
- Grep  
- Read
- WebFetch
- WebSearch
- Agent
- Skill
- Task

**Rendering:** Lines 964-977  
**Features:**
- Description extracted and rendered as Markdown
- Remaining input rendered as JSON with Markdown in string values
- Dual JSON/Markdown view

---

### 3.7 Schema Inference Evidence

**Where Schemas Come From:**

1. **Not Explicitly Defined:**
   - No TypeScript or JSON Schema files
   - No Pydantic models or dataclasses
   - Schemas are implicit in parsing/rendering code

2. **Inferred From:**
   - **Parsing Code:** Lines 637-685 (`parse_session_file`, `_parse_jsonl_file`)
   - **Rendering Code:** Lines 937-1042 (`render_content_block`)
   - **Test Fixtures:** `tests/sample_session.json`, `tests/sample_session.jsonl`

3. **Duck Typing Approach:**
   ```python
   block_type = block.get("type", "")
   if block_type == "text":
       # Handle text block
   elif block_type == "thinking":
       # Handle thinking block
   ```
   - No validation
   - No schema enforcement
   - Relies on correct input format

---

### 3.8 Optional Fields and Edge Cases

#### Optional Fields Handling

**Pattern Throughout Code:**
```python
tool_input.get("description", "")  # Empty string default
obj.get("isCompactSummary")        # None default
```

**Common Optional Fields:**
- `LogLine.isCompactSummary` - Defaults to false
- `ToolInput.description` - Defaults to empty string
- `EditToolInput.replace_all` - Defaults to false
- `ContentBlock fields` - Missing fields return empty/None

#### Edge Cases

1. **Empty Content:**
   ```python
   if not content_html.strip():
       return ""  # Skip empty messages
   ```

2. **Malformed JSON in tool_result:**
   ```python
   try:
       parsed_blocks = json.loads(content)
   except (json.JSONDecodeError, TypeError):
       content_markdown_html = format_json(content)
   ```

3. **Missing Tool Results:**
   - Tool calls rendered alone if no matching result
   - No error thrown

4. **ANSI in Content:**
   - Stripped automatically in tool results (line 984)

5. **Long Content:**
   - Truncated with expand button
   - Threshold: 200px height

---

## 4. Proposed Componentization Plan

### 4.1 Current Architecture Issues

**Problems with Monolithic Design:**

1. **Single 2994-line file** (`__init__.py`)
   - Hard to navigate
   - Difficult to test specific functions
   - Merge conflicts likely with multiple contributors

2. **Mixed concerns in one module:**
   - Session discovery
   - File parsing
   - HTML rendering
   - API interaction
   - CLI commands
   - CSS/JS constants

3. **No clear module boundaries:**
   - Functions call each other across concerns
   - Global variables (`_github_repo`, `_jinja_env`)
   - Hard to reuse components

4. **Testing challenges:**
   - Must test through high-level functions
   - Hard to mock dependencies
   - Slow test execution

---

### 4.2 Proposed Module Structure

```
src/claude_code_transcripts/
├── __init__.py                    # Package exports only
├── __main__.py                    # CLI entry point
│
├── discovery/
│   ├── __init__.py
│   ├── local.py                   # Local session discovery
│   ├── web.py                     # API-based session fetching
│   └── filters.py                 # Session filtering logic
│
├── parsing/
│   ├── __init__.py
│   ├── session.py                 # Session file parsing
│   ├── normalizer.py              # Format normalization
│   └── schemas.py                 # Schema definitions (optional)
│
├── processing/
│   ├── __init__.py
│   ├── grouping.py                # Conversation grouping
│   ├── tool_pairing.py            # Tool call/result linking
│   └── analysis.py                # Stats, commits, metadata
│
├── rendering/
│   ├── __init__.py
│   ├── message.py                 # Message rendering
│   ├── content_blocks.py          # Content block renderers
│   ├── tools.py                   # Tool-specific renderers
│   └── formatters.py              # Code highlighting, markdown
│
├── templates/
│   ├── macros.html                # (existing)
│   ├── page.html                  # (existing)
│   ├── index.html                 # (existing)
│   └── base.html                  # (existing)
│
├── output/
│   ├── __init__.py
│   ├── html_generator.py          # HTML page generation
│   ├── pagination.py              # Pagination logic
│   └── gist.py                    # Gist upload functionality
│
├── cli/
│   ├── __init__.py
│   ├── commands.py                # CLI command definitions
│   ├── local_cmd.py               # Local command handler
│   ├── web_cmd.py                 # Web command handler
│   ├── json_cmd.py                # JSON command handler
│   └── all_cmd.py                 # All command handler
│
├── utils/
│   ├── __init__.py
│   ├── text.py                    # Text utilities (ANSI stripping, etc.)
│   ├── git.py                     # Git repo detection
│   └── credentials.py             # API credential handling
│
└── assets/
    ├── __init__.py
    ├── styles.py                  # CSS constants
    └── scripts.py                 # JS constants
```

---

### 4.3 Module Specifications

#### 4.3.1 Session Discovery Module

**File:** `src/claude_code_transcripts/discovery/local.py`

**Purpose:** Find and enumerate local Claude Code sessions.

**Public API:**
```python
def find_local_sessions(
    folder: Path,
    limit: int = 10,
    include_agents: bool = False,
) -> list[tuple[Path, str]]:
    """Find recent session files in folder.
    
    Returns:
        List of (filepath, summary) tuples, sorted by mtime.
    """

def get_session_summary(filepath: Path, max_length: int = 200) -> str:
    """Extract summary from session file."""

def get_project_display_name(folder_name: str) -> str:
    """Convert encoded folder name to readable name."""
```

**Dependencies:**
- Standard library only (pathlib, datetime)
- No rendering dependencies

**Why Separate:**
- Can be tested without parsing or rendering
- Reusable in other contexts (list sessions without converting)
- Clear single responsibility

---

#### 4.3.2 Session Parsing Module

**File:** `src/claude_code_transcripts/parsing/session.py`

**Purpose:** Read and normalize session files.

**Public API:**
```python
def parse_session_file(filepath: Path) -> dict:
    """Parse JSON or JSONL session file.
    
    Returns:
        Normalized dict with 'loglines' key.
    """

def _parse_json_file(filepath: Path) -> dict:
    """Parse standard JSON format."""

def _parse_jsonl_file(filepath: Path) -> dict:
    """Parse JSONL format and normalize."""
```

**File:** `src/claude_code_transcripts/parsing/schemas.py` (Optional)

**Purpose:** Define schemas using Pydantic or dataclasses.

```python
from dataclasses import dataclass
from typing import Literal, Union

@dataclass
class TextBlock:
    type: Literal["text"]
    text: str

@dataclass
class ToolUseBlock:
    type: Literal["tool_use"]
    id: str
    name: str
    input: dict

ContentBlock = Union[TextBlock, ToolUseBlock, ...]
```

**Why Separate:**
- Parsing is independent of rendering
- Can validate schemas separately
- Easier to add new format support
- Testable in isolation

---

#### 4.3.3 Processing Module

**File:** `src/claude_code_transcripts/processing/grouping.py`

**Purpose:** Group messages into conversations.

**Public API:**
```python
def group_loglines_into_conversations(
    loglines: list[dict],
) -> list[Conversation]:
    """Group messages by user prompts.
    
    Returns:
        List of Conversation objects.
    """

@dataclass
class Conversation:
    user_text: str
    timestamp: str
    messages: list[tuple[str, str, str]]  # (type, json, timestamp)
    is_continuation: bool
```

**File:** `src/claude_code_transcripts/processing/tool_pairing.py`

**Purpose:** Link tool calls with results.

**Public API:**
```python
def build_tool_result_lookup(
    messages: list[tuple],
) -> dict[str, dict]:
    """Build tool_use_id → tool_result mapping."""

def pair_tools_in_conversation(
    messages: list[tuple],
) -> tuple[dict, set]:
    """Return (lookup, paired_ids) for conversation."""
```

**File:** `src/claude_code_transcripts/processing/analysis.py`

**Purpose:** Extract stats and metadata.

**Public API:**
```python
def analyze_conversation(
    messages: list[tuple],
) -> ConversationStats:
    """Analyze messages for stats.
    
    Returns:
        Stats object with tool_counts, commits, long_texts.
    """
```

**Why Separate:**
- Business logic independent of I/O
- Highly testable (pure functions)
- Can optimize algorithms separately
- Clear data flow

---

#### 4.3.4 Rendering Module

**File:** `src/claude_code_transcripts/rendering/message.py`

**Purpose:** High-level message rendering.

**Public API:**
```python
def render_message_with_tool_pairs(
    log_type: str,
    message_data: dict,
    timestamp: str,
    tool_result_lookup: dict,
    paired_tool_ids: set,
) -> str:
    """Render a complete message as HTML."""
```

**File:** `src/claude_code_transcripts/rendering/content_blocks.py`

**Purpose:** Content block rendering dispatch.

**Public API:**
```python
def render_content_block(block: dict) -> str:
    """Dispatch to appropriate renderer based on block type."""

def render_content_block_array(blocks: list[dict]) -> str:
    """Render array of content blocks."""
```

**File:** `src/claude_code_transcripts/rendering/tools.py`

**Purpose:** Tool-specific renderers.

**Public API:**
```python
def render_write_tool(tool_input: dict, tool_id: str) -> str:
def render_edit_tool(tool_input: dict, tool_id: str) -> str:
def render_bash_tool(tool_input: dict, tool_id: str) -> str:
def render_todo_write(tool_input: dict, tool_id: str) -> str:
```

**File:** `src/claude_code_transcripts/rendering/formatters.py`

**Purpose:** Formatting utilities.

**Public API:**
```python
def highlight_code(
    code: str,
    filename: str = None,
    language: str = None,
) -> str:
    """Apply syntax highlighting."""

def render_markdown_text(text: str) -> str:
    """Convert Markdown to HTML."""

def format_json(obj: any) -> str:
    """Format JSON with HTML."""
```

**Why Separate:**
- Each renderer is independently testable
- Easy to add new tool renderers
- Clear separation of concerns
- Easier to optimize rendering performance

---

#### 4.3.5 Output Module

**File:** `src/claude_code_transcripts/output/html_generator.py`

**Purpose:** Generate HTML files from conversations.

**Public API:**
```python
def generate_html(
    json_path: Path,
    output_dir: Path,
    github_repo: str = None,
) -> None:
    """Main HTML generation orchestrator."""

def generate_page_html(
    page_num: int,
    conversations: list,
    total_pages: int,
    output_dir: Path,
) -> None:
    """Generate single page HTML."""

def generate_index_html(
    conversations: list,
    total_pages: int,
    output_dir: Path,
    github_repo: str,
) -> None:
    """Generate index page HTML."""
```

**File:** `src/claude_code_transcripts/output/pagination.py`

**Purpose:** Pagination logic and HTML generation.

**Public API:**
```python
PROMPTS_PER_PAGE = 5

def calculate_pagination(total_convs: int) -> int:
    """Calculate total pages needed."""

def get_page_conversations(
    conversations: list,
    page_num: int,
) -> list:
    """Get conversations for specific page."""

def generate_pagination_html(
    current_page: int,
    total_pages: int,
) -> str:
    """Generate pagination HTML."""
```

**File:** `src/claude_code_transcripts/output/gist.py`

**Purpose:** GitHub Gist upload.

**Public API:**
```python
def create_gist(
    output_dir: Path,
    public: bool = False,
) -> tuple[str, str]:
    """Create gist and return (gist_id, gist_url)."""

def inject_gist_preview_js(output_dir: Path) -> None:
    """Inject JS to fix gistpreview.github.io URLs."""
```

**Why Separate:**
- Output generation is distinct from rendering
- Gist functionality is optional
- Easier to add other output formats (PDF, etc.)
- Clear I/O boundary

---

#### 4.3.6 CLI Module

**File:** `src/claude_code_transcripts/cli/commands.py`

**Purpose:** CLI command definitions.

**Public API:**
```python
@click.group(cls=DefaultGroup, default="local")
def cli():
    """Main CLI group."""

def create_cli() -> click.Group:
    """Factory function for CLI creation."""
```

**File:** `src/claude_code_transcripts/cli/local_cmd.py`

**Purpose:** Local session command.

**Public API:**
```python
@click.command()
@click.option(...)
def local_cmd(...):
    """Select and convert local session."""
```

**Similar Files:**
- `web_cmd.py` - Web API session import
- `json_cmd.py` - Direct file conversion
- `all_cmd.py` - Batch conversion

**Why Separate:**
- Each command is independently testable
- Easier to add new commands
- Clear entry points
- Reduces CLI complexity

---

#### 4.3.7 Utilities Module

**File:** `src/claude_code_transcripts/utils/text.py`

**Purpose:** Text processing utilities.

**Public API:**
```python
def strip_ansi(text: str) -> str:
    """Strip ANSI escape sequences."""

def extract_text_from_content(content: str | list) -> str:
    """Extract plain text from message content."""

def is_content_block_array(text: str) -> bool:
    """Check if string is JSON array of content blocks."""
```

**File:** `src/claude_code_transcripts/utils/git.py`

**Purpose:** Git-related utilities.

**Public API:**
```python
def detect_github_repo(loglines: list[dict]) -> str | None:
    """Detect GitHub repo from git push output."""

COMMIT_PATTERN: re.Pattern  # Regex for commit detection
GITHUB_REPO_PATTERN: re.Pattern  # Regex for repo detection
```

**File:** `src/claude_code_transcripts/utils/credentials.py`

**Purpose:** API credential management.

**Public API:**
```python
def get_access_token_from_keychain() -> str | None:
    """Get token from macOS keychain."""

def get_org_uuid_from_config() -> str | None:
    """Get org UUID from ~/.claude.json."""

def get_api_headers(token: str, org_uuid: str) -> dict:
    """Build API request headers."""
```

**Why Separate:**
- Utility functions are highly reusable
- Easy to test in isolation
- Clear for adding new utilities
- No dependencies on main business logic

---

#### 4.3.8 Assets Module

**File:** `src/claude_code_transcripts/assets/styles.py`

**Purpose:** CSS constants.

**Public API:**
```python
CSS: str  # Complete CSS stylesheet
```

**File:** `src/claude_code_transcripts/assets/scripts.py`

**Purpose:** JavaScript constants.

**Public API:**
```python
JS: str  # Main JavaScript
GIST_PREVIEW_JS: str  # Gist preview fix script
```

**Why Separate:**
- Keeps main modules clean
- Easier to update styles/scripts
- Could be replaced with external files later
- Clear asset management

---

### 4.4 Migration Strategy

**Phase 1: Create Module Structure** (Low Risk)
1. Create new directory structure
2. Create empty `__init__.py` files
3. No code movement yet

**Phase 2: Extract Utilities** (Low Risk)
1. Move pure functions to utils/
   - `strip_ansi()` → `utils/text.py`
   - `detect_github_repo()` → `utils/git.py`
2. Update imports in `__init__.py`
3. Run full test suite

**Phase 3: Extract Assets** (Low Risk)
1. Move CSS constant → `assets/styles.py`
2. Move JS constants → `assets/scripts.py`
3. Update imports
4. Run tests

**Phase 4: Extract Discovery** (Medium Risk)
1. Move `find_local_sessions()` → `discovery/local.py`
2. Move `get_session_summary()` → `discovery/local.py`
3. Create tests for discovery module
4. Update imports

**Phase 5: Extract Parsing** (Medium Risk)
1. Move `parse_session_file()` → `parsing/session.py`
2. Move `_parse_jsonl_file()` → `parsing/session.py`
3. Create tests for parsing module
4. Update imports

**Phase 6: Extract Rendering** (High Risk)
1. Move rendering functions → `rendering/`
2. Split by responsibility (message, content_blocks, tools)
3. Create comprehensive tests
4. Update imports

**Phase 7: Extract Processing** (High Risk)
1. Move grouping logic → `processing/grouping.py`
2. Move tool pairing → `processing/tool_pairing.py`
3. Move analysis → `processing/analysis.py`
4. Create tests
5. Update imports

**Phase 8: Extract Output** (Medium Risk)
1. Move `generate_html()` → `output/html_generator.py`
2. Move pagination logic → `output/pagination.py`
3. Move gist functions → `output/gist.py`
4. Create tests
5. Update imports

**Phase 9: Extract CLI** (Low Risk)
1. Move CLI commands → `cli/`
2. Create command files
3. Update `__main__.py` entry point
4. Run CLI tests

**Phase 10: Clean Up** (Low Risk)
1. Update `__init__.py` to only export public API
2. Update documentation
3. Run full test suite
4. Performance testing

---

### 4.5 Benefits of Componentization

**For Development:**
- **Faster navigation:** Find code by module name
- **Easier testing:** Test small units independently
- **Better IDE support:** Type hints more effective
- **Clearer ownership:** Each module has defined purpose

**For Maintenance:**
- **Isolated changes:** Modify one component without affecting others
- **Easier debugging:** Smaller scope to search
- **Simpler refactoring:** Refactor one module at a time
- **Better code review:** Smaller, focused PRs

**For Extension:**
- **Plugin architecture:** Easy to add new renderers
- **Format support:** Add new session formats easily
- **Output formats:** Add PDF, Markdown, etc.
- **Custom tools:** Add tool renderers without core changes

**For Testing:**
- **Unit tests:** Test each function independently
- **Integration tests:** Test module interactions
- **Mocking:** Mock dependencies easily
- **Coverage:** Measure per-module coverage

**For Performance:**
- **Profiling:** Identify slow modules
- **Optimization:** Optimize one component
- **Lazy loading:** Import only what's needed
- **Caching:** Add caching at module boundaries

---

### 4.6 Additional Recommended Modules

#### 4.6.1 Caching Module

**File:** `src/claude_code_transcripts/caching/summary_cache.py`

**Purpose:** Cache session summaries for faster listing.

**Why:**
- Reading 100+ JSONL files is slow
- Summaries rarely change
- Improves UX for `local` command

**API:**
```python
def get_cached_summary(filepath: Path) -> str | None:
    """Get cached summary if fresh."""

def cache_summary(filepath: Path, summary: str) -> None:
    """Cache summary with mtime."""

def clear_cache() -> None:
    """Clear all cached summaries."""
```

---

#### 4.6.2 Validation Module

**File:** `src/claude_code_transcripts/validation/session_validator.py`

**Purpose:** Validate session file structure.

**Why:**
- Catch malformed files early
- Provide helpful error messages
- Support schema evolution

**API:**
```python
def validate_session(data: dict) -> ValidationResult:
    """Validate session structure."""

def validate_logline(logline: dict) -> ValidationResult:
    """Validate single logline."""

@dataclass
class ValidationResult:
    valid: bool
    errors: list[str]
    warnings: list[str]
```

---

#### 4.6.3 Export Module

**File:** `src/claude_code_transcripts/export/markdown.py`

**Purpose:** Export sessions to Markdown format.

**Why:**
- Many users prefer Markdown
- Easier to version control
- Better for documentation

**API:**
```python
def export_to_markdown(
    json_path: Path,
    output_path: Path,
) -> None:
    """Export session to Markdown."""
```

**Similar:**
- `export/pdf.py` - PDF export
- `export/json.py` - Cleaned JSON export

---

#### 4.6.4 Search Module

**File:** `src/claude_code_transcripts/search/indexer.py`

**Purpose:** Index sessions for full-text search.

**Why:**
- Find sessions by content
- Better than filename search
- Useful for large archives

**API:**
```python
def index_sessions(sessions_dir: Path) -> SearchIndex:
    """Build search index."""

def search_sessions(
    index: SearchIndex,
    query: str,
) -> list[SearchResult]:
    """Search indexed sessions."""
```

---

## 5. Conclusion

### Summary of Findings

1. **Core Flows:**
   - Session discovery is file-based, sorted by mtime
   - Parsing normalizes JSON/JSONL into uniform format
   - Message assembly groups by user prompts, pairs tool calls
   - Ordering relies on file order and timestamps

2. **Sub-Agent Connection:**
   - No current sub-agent support
   - Tool linking uses unique IDs
   - Message attribution via type/role fields
   - Turn-based conversation model

3. **Object Schemas:**
   - Schemas are implicit, not formally defined
   - Multiple format variants (string vs array content)
   - Rich content block types with specialized renderers
   - Tool-specific input schemas

4. **Componentization:**
   - Current monolith has 2994 lines in single file
   - Proposed 8-module structure with clear boundaries
   - Migration strategy in 10 phases
   - Benefits: testability, maintainability, extensibility

### Recommendations

**Immediate Actions:**
1. Extract utilities (low risk, high value)
2. Add session validation module
3. Create caching for summaries

**Short Term:**
1. Split discovery and parsing modules
2. Add comprehensive tests for each module
3. Document APIs with type hints

**Long Term:**
1. Complete full componentization
2. Add export formats (Markdown, PDF)
3. Implement search/indexing
4. Consider sub-agent support

### Files Referenced

**Main Implementation:**
- `src/claude_code_transcripts/__init__.py` - Lines 1-2994

**Tests:**
- `tests/test_generate_html.py`
- `tests/test_all.py`
- `tests/sample_session.json`
- `tests/sample_session.jsonl`

**Templates:**
- `src/claude_code_transcripts/templates/macros.html`
- `src/claude_code_transcripts/templates/page.html`
- `src/claude_code_transcripts/templates/index.html`

**Documentation:**
- `README.md`
- `TASKS.md`
- `AGENTS.md`

---

**Document End**
