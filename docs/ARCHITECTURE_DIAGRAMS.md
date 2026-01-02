# Architecture Diagrams

## Current System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLI Entry Point                             │
│  (local, web, json, all commands)                               │
└───────────────────┬─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                  __init__.py (2994 lines)                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐   │
│  │ Session Discovery                                       │   │
│  │ • find_local_sessions()                                │   │
│  │ • get_session_summary()                                │   │
│  │ • get_project_display_name()                           │   │
│  └────────────────────────────────────────────────────────┘   │
│                         │                                       │
│                         ▼                                       │
│  ┌────────────────────────────────────────────────────────┐   │
│  │ Session Parsing                                         │   │
│  │ • parse_session_file()                                 │   │
│  │ • _parse_jsonl_file()                                  │   │
│  └────────────────────────────────────────────────────────┘   │
│                         │                                       │
│                         ▼                                       │
│  ┌────────────────────────────────────────────────────────┐   │
│  │ Message Processing                                      │   │
│  │ • Group by conversations                               │   │
│  │ • Build tool_result_lookup                             │   │
│  │ • Track paired_tool_ids                                │   │
│  │ • analyze_conversation()                               │   │
│  └────────────────────────────────────────────────────────┘   │
│                         │                                       │
│                         ▼                                       │
│  ┌────────────────────────────────────────────────────────┐   │
│  │ Rendering                                               │   │
│  │ • render_message_with_tool_pairs()                     │   │
│  │ • render_content_block()                               │   │
│  │ • Tool-specific renderers                              │   │
│  │ • highlight_code()                                     │   │
│  │ • render_markdown_text()                               │   │
│  └────────────────────────────────────────────────────────┘   │
│                         │                                       │
│                         ▼                                       │
│  ┌────────────────────────────────────────────────────────┐   │
│  │ Output Generation                                       │   │
│  │ • generate_html()                                      │   │
│  │ • Pagination (5 per page)                              │   │
│  │ • create_gist()                                        │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Constants: CSS (330 lines), JS (150 lines)                    │
└─────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Jinja2 Templates                             │
│  • macros.html  • page.html  • index.html  • base.html         │
└─────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                   HTML Output Files                             │
│  • index.html  • page-001.html  • page-002.html  • ...         │
└─────────────────────────────────────────────────────────────────┘
```

## Data Flow: Session to HTML

```
Session File (.json/.jsonl)
    │
    ├─ JSON: Direct load
    │      {"loglines": [...]}
    │
    └─ JSONL: Parse line-by-line
           {"type": "user", ...}
           {"type": "assistant", ...}
           {"type": "user", ...}
    │
    ▼
Normalized Format
    {"loglines": [
        {"type": "user", "timestamp": "...", "message": {...}},
        {"type": "assistant", "timestamp": "...", "message": {...}},
        ...
    ]}
    │
    ▼
Conversation Grouping
    [
        {
            "user_text": "Prompt 1",
            "messages": [(type, json, ts), ...],
            "is_continuation": false
        },
        ...
    ]
    │
    ▼
Tool Pairing
    tool_result_lookup = {
        "toolu_001": {...},
        "toolu_002": {...}
    }
    paired_tool_ids = {"toolu_001", "toolu_002"}
    │
    ▼
Message Rendering
    For each message:
        ├─ Parse JSON
        ├─ Lookup tool results
        ├─ Render content blocks
        │   ├─ text → Markdown
        │   ├─ thinking → Styled block
        │   ├─ tool_use → Tool renderer
        │   └─ tool_result → Paired display
        └─ Generate HTML
    │
    ▼
Pagination
    Split into pages (5 conversations each)
    │
    ▼
HTML Generation
    • page-001.html (Conversations 1-5)
    • page-002.html (Conversations 6-10)
    • ...
    • index.html (Timeline + stats)
```

## Tool Pairing Mechanism

```
Assistant Message                    User Message
┌──────────────────────┐            ┌──────────────────────┐
│ content: [           │            │ content: [           │
│   {                  │            │   {                  │
│     type: "tool_use",│────────────│     type: "tool_result",
│     id: "toolu_001", │   Links    │     tool_use_id: "toolu_001",
│     name: "Write",   │    via     │     content: "Success"
│     input: {...}     │   ID ref   │   }                  │
│   }                  │            │ ]                    │
│ ]                    │            │                      │
└──────────────────────┘            └──────────────────────┘
         │                                      │
         └──────────────┬───────────────────────┘
                        │
                        ▼
              Paired Rendering
         ┌──────────────────────────┐
         │ ┌──────────────────────┐ │
         │ │ Tool Call: Write     │ │
         │ │ file_path: foo.py    │ │
         │ └──────────────────────┘ │
         │ ┌──────────────────────┐ │
         │ │ Tool Result          │ │
         │ │ Success              │ │
         │ └──────────────────────┘ │
         └──────────────────────────┘
```

## Proposed Module Structure

```
claude-code-transcripts/
├── src/claude_code_transcripts/
│   ├── __init__.py              # Public API exports only
│   ├── __main__.py              # CLI entry point
│   │
│   ├── discovery/               # Session finding
│   │   ├── __init__.py
│   │   ├── local.py            # find_local_sessions()
│   │   ├── web.py              # API-based fetching
│   │   └── filters.py          # Session filtering
│   │
│   ├── parsing/                 # File reading
│   │   ├── __init__.py
│   │   ├── session.py          # parse_session_file()
│   │   ├── normalizer.py       # Format conversion
│   │   └── schemas.py          # Optional: Pydantic/dataclass
│   │
│   ├── processing/              # Data transformation
│   │   ├── __init__.py
│   │   ├── grouping.py         # Conversation grouping
│   │   ├── tool_pairing.py     # Tool call/result linking
│   │   └── analysis.py         # Stats extraction
│   │
│   ├── rendering/               # HTML generation
│   │   ├── __init__.py
│   │   ├── message.py          # Message rendering
│   │   ├── content_blocks.py   # Block renderers
│   │   ├── tools.py            # Tool-specific renderers
│   │   └── formatters.py       # Code/Markdown formatting
│   │
│   ├── output/                  # File writing
│   │   ├── __init__.py
│   │   ├── html_generator.py   # Main orchestrator
│   │   ├── pagination.py       # Page splitting
│   │   └── gist.py             # GitHub Gist upload
│   │
│   ├── cli/                     # CLI commands
│   │   ├── __init__.py
│   │   ├── commands.py         # CLI group definition
│   │   ├── local_cmd.py        # Local command
│   │   ├── web_cmd.py          # Web command
│   │   ├── json_cmd.py         # JSON command
│   │   └── all_cmd.py          # All command
│   │
│   ├── utils/                   # Shared utilities
│   │   ├── __init__.py
│   │   ├── text.py             # ANSI stripping, text extraction
│   │   ├── git.py              # Repo detection
│   │   └── credentials.py      # API credentials
│   │
│   ├── assets/                  # Static resources
│   │   ├── __init__.py
│   │   ├── styles.py           # CSS constants
│   │   └── scripts.py          # JS constants
│   │
│   └── templates/               # Jinja2 templates (existing)
│       ├── macros.html
│       ├── page.html
│       ├── index.html
│       └── base.html
│
└── tests/                       # Test suite
    ├── test_discovery/
    ├── test_parsing/
    ├── test_processing/
    ├── test_rendering/
    └── ...
```

## Content Block Rendering Pipeline

```
ContentBlock
    │
    ├─ type: "text"
    │   └─> render_markdown_text()
    │       └─> Markdown → HTML
    │
    ├─ type: "thinking"
    │   └─> render_markdown_text()
    │       └─> _macros.thinking()
    │           └─> Styled yellow block
    │
    ├─ type: "tool_use"
    │   ├─ name: "Write"
    │   │   └─> render_write_tool()
    │   │       └─> highlight_code()
    │   │           └─> _macros.write_tool()
    │   │
    │   ├─ name: "Edit"
    │   │   └─> render_edit_tool()
    │   │       └─> highlight_code() (old & new)
    │   │           └─> _macros.edit_tool()
    │   │
    │   ├─ name: "Bash"
    │   │   └─> render_bash_tool()
    │   │       └─> _macros.bash_tool()
    │   │
    │   ├─ name: "TodoWrite"
    │   │   └─> render_todo_write()
    │   │       └─> _macros.todo_list()
    │   │
    │   └─ name: [Other]
    │       └─> render_json_with_markdown()
    │           └─> _macros.tool_use()
    │
    ├─ type: "tool_result"
    │   ├─> strip_ansi()
    │   ├─> Detect commits
    │   ├─> format_json() (JSON view)
    │   ├─> render_markdown_text() (Markdown view)
    │   └─> _macros.tool_result()
    │
    └─ type: "image"
        └─> _macros.image_block()
            └─> <img data:base64,...>
```

## Session Discovery Flow Chart

```
CLI: claude-code-transcripts local
    │
    ▼
find_local_sessions(~/.claude/projects, limit=10)
    │
    ├─> Glob: **/*.jsonl
    │   └─> [file1.jsonl, file2.jsonl, ...]
    │
    ├─> Filter: Skip if name.startswith("agent-")
    │   └─> [file1.jsonl, file2.jsonl]
    │
    ├─> For each file:
    │   ├─> get_session_summary(file)
    │   │   ├─> JSONL: _get_jsonl_summary()
    │   │   │   ├─> Priority 1: type="summary" entry
    │   │   │   └─> Priority 2: First user message
    │   │   └─> JSON: First user message
    │   │
    │   ├─> Skip if summary == "warmup"
    │   └─> Skip if summary == "(no summary)"
    │
    ├─> Sort: By mtime (newest first)
    │   └─> [(file1, mtime1), (file2, mtime2), ...]
    │
    └─> Return: Top N results
        └─> [(file1, summary1), (file2, summary2), ...]
            │
            ▼
        questionary.select()
            │
            ▼
        Selected session file
            │
            ▼
        generate_html()
```

---

**Full Documentation:** [ARCHITECTURE_ANALYSIS.md](./ARCHITECTURE_ANALYSIS.md)
