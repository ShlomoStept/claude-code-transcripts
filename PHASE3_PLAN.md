# Phase 3 Implementation Plan

## Overview

Phase 3 focuses on advanced features that build upon the foundation established in Phase 1 (core rendering) and Phase 2 (structural improvements). The primary goals are:

1. **A.4 Recursive Nesting** - Enable nested collapsible structures
2. **C.1 Subagent Detection** - Identify and visualize subagent conversations
3. **Architecture Improvements** - Module splitting for maintainability
4. **CSS/JS Externalization** - Build step for static assets

---

## A.4 Recursive Nesting Support

### Goal
Enable collapsible cells to contain other collapsible cells, supporting nested tool calls and subagent task visualization.

### Current State
- Cells are single-level collapsible `<details>` elements
- Thinking, Response, and Tools cells are siblings within a message
- No support for nesting cells within cells

### Implementation Design

#### 1. Data Model Changes
```python
# New recursive content structure
class NestedContent:
    """Represents content that may contain nested collapsible sections."""
    type: str  # 'text', 'thinking', 'tool_call', 'subagent'
    content: str | list["NestedContent"]
    metadata: dict  # Optional metadata like timestamps
```

#### 2. Template Changes (macros.html)

**New Macro: `nested_cell`**
```jinja
{# Recursive cell that can contain other cells #}
{% macro nested_cell(cell_type, label, content, children=[], depth=0, open_by_default=false) %}
<details class="cell {{ cell_type }}-cell nested-depth-{{ depth }}"{% if open_by_default %} open{% endif %}>
<summary class="cell-header">
    <span class="cell-label">{{ label }}</span>
    {% if children|length > 0 %}
    <span class="nested-count">({{ children|length }} sub-items)</span>
    {% endif %}
</summary>
<div class="cell-content">
    {{ content|safe }}
    {% for child in children %}
    {{ nested_cell(child.type, child.label, child.content, child.children, depth + 1) }}
    {% endfor %}
</div>
</details>
{%- endmacro %}
```

#### 3. CSS Additions
```css
/* Nested cell indentation */
.nested-depth-1 { margin-left: var(--spacing-md); }
.nested-depth-2 { margin-left: calc(var(--spacing-md) * 2); }
.nested-depth-3 { margin-left: calc(var(--spacing-md) * 3); }

/* Visual connector lines for nesting */
.cell.nested-depth-1::before,
.cell.nested-depth-2::before,
.cell.nested-depth-3::before {
    content: '';
    position: absolute;
    left: calc(-1 * var(--spacing-sm));
    top: 0;
    bottom: 0;
    width: 2px;
    background: var(--border-light);
}
```

#### 4. Use Cases
- Subagent tool calls that spawn their own tool calls
- Nested Task tool invocations
- Agent tool with child operations

### Files to Modify
| File | Changes |
|------|---------|
| `macros.html` | Add `nested_cell` macro |
| `__init__.py` | Add recursive content detection |
| `__init__.py` CSS | Add nesting styles |
| `tests/test_generate_html.py` | Add nesting tests |

### Estimated Effort
- Medium complexity
- 2-3 hours implementation
- Requires careful recursive template handling

---

## C.1 Subagent Detection and Grouping

### Goal
Detect subagent JSONL files (agent-*.jsonl), link them to main transcripts, and provide optional inline expansion.

### Current State
- Main session files processed independently
- No awareness of related subagent sessions
- Subagent files exist as separate `agent-*.jsonl` files

### Implementation Design

#### 1. Subagent Detection Function
```python
def find_related_agent_sessions(main_session_path: Path) -> list[dict]:
    """Find subagent JSONL files related to a main session.

    Subagent files follow the pattern: agent-{timestamp}.jsonl
    They exist in the same directory as the main session.

    Returns:
        List of dicts with keys: path, timestamp, summary
    """
    session_dir = main_session_path.parent
    agent_files = list(session_dir.glob("agent-*.jsonl"))

    related = []
    for agent_file in agent_files:
        # Parse timestamp from filename
        timestamp = extract_agent_timestamp(agent_file.name)
        summary = get_session_summary(agent_file)
        related.append({
            "path": agent_file,
            "timestamp": timestamp,
            "summary": summary,
        })

    return sorted(related, key=lambda x: x["timestamp"])
```

#### 2. Linking Strategy

**Option A: Reference Links**
- Add "View Subagent Session" links in tool result output
- Navigate to separate page for subagent

**Option B: Inline Expansion**
- Embed subagent content within collapsible cell
- Uses recursive nesting (A.4)
- Requires subagent HTML to be generated inline

**Recommended: Option A initially, Option B as enhancement**

#### 3. Template Changes
```jinja
{# Subagent reference card #}
{% macro subagent_card(agent_file, timestamp, summary) %}
<div class="subagent-card">
    <div class="subagent-header">
        <span class="subagent-icon">🤖</span>
        <span class="subagent-label">Subagent Session</span>
        <time datetime="{{ timestamp }}">{{ timestamp }}</time>
    </div>
    <div class="subagent-summary">{{ summary }}</div>
    <a href="{{ agent_file }}" class="subagent-link">View full session →</a>
</div>
{%- endmacro %}
```

#### 4. Detection Triggers
Subagent sessions are typically triggered by:
- `Agent` tool calls
- `Task` tool calls
- Some `Skill` tool calls

The main session shows a tool call, and the corresponding subagent file contains the subagent's work.

### Files to Modify
| File | Changes |
|------|---------|
| `__init__.py` | Add `find_related_agent_sessions()`, `extract_agent_timestamp()` |
| `macros.html` | Add `subagent_card` macro |
| `__init__.py` CSS | Add `.subagent-*` styles |
| `tests/test_generate_html.py` | Add subagent detection tests |

### Estimated Effort
- Medium-high complexity
- 3-4 hours implementation
- Requires understanding of subagent file naming conventions

---

## Architecture Improvements: Module Splitting

### Goal
Split the monolithic `__init__.py` (~3000 lines) into focused modules for better maintainability.

### Current Structure
```
src/claude_code_transcripts/
├── __init__.py          # Everything: CLI, render, utils, CSS, JS
└── templates/           # Jinja2 templates
```

### Proposed Structure
```
src/claude_code_transcripts/
├── __init__.py          # Package exports, version
├── cli.py               # Click commands, argument parsing
├── render.py            # HTML generation, template rendering
├── utils.py             # Helper functions (strip_ansi, highlight_code, etc.)
├── constants.py         # CSS, JS, TOOL_ICONS, patterns
├── api.py               # Anthropic API interaction
└── templates/           # Jinja2 templates (unchanged)
```

### Module Breakdown

#### `cli.py` (~500 lines)
- Click command group and subcommands
- `local`, `session`, `import`, `all` commands
- Browser opening logic
- Output directory handling

#### `render.py` (~800 lines)
- `generate_html()` and `generate_html_jsonl()`
- `render_message()`, `render_content_block()`
- Template rendering functions
- Pagination generation

#### `utils.py` (~300 lines)
- `strip_ansi()`
- `highlight_code()`
- `is_json_like()`, `format_json()`
- `render_markdown_text()`
- `get_session_summary()`
- Date/time formatting

#### `constants.py` (~1200 lines)
- `CSS` string constant
- `JS` string constant
- `TOOL_ICONS` dict
- `COMMIT_PATTERN`, `GITHUB_REPO_PATTERN`
- Configuration constants

#### `api.py` (~200 lines)
- Anthropic API interaction
- Model listing
- (Future) API-based transcript generation

### Migration Strategy

1. **Phase 1**: Create modules with imports from `__init__.py`
2. **Phase 2**: Move functions one module at a time, run tests
3. **Phase 3**: Update `__init__.py` to re-export public API
4. **Phase 4**: Remove old code, verify all tests pass

### Backward Compatibility
```python
# __init__.py
from .cli import cli
from .render import generate_html, generate_html_jsonl, render_content_block
from .utils import strip_ansi, highlight_code
from .constants import CSS, JS, TOOL_ICONS

# Ensure existing imports still work
__all__ = [
    "cli", "generate_html", "generate_html_jsonl",
    "render_content_block", "strip_ansi", ...
]
```

### Estimated Effort
- High complexity (refactoring risk)
- 4-6 hours implementation
- Requires comprehensive test coverage first

---

## CSS/JS Externalization

### Goal
Move CSS and JS from embedded string constants to external files, with a build step to embed them for distribution.

### Current State
- CSS is a ~800 line string constant in `__init__.py`
- JS is a ~200 line string constant in `__init__.py`
- Changes require editing Python strings

### Proposed Structure
```
src/claude_code_transcripts/
├── static/
│   ├── styles.css       # Development CSS
│   └── script.js        # Development JS
├── constants.py          # Contains embedded CSS/JS for distribution
└── build.py              # Script to embed static files
```

### Build Process

#### Development Mode
```python
# In render.py
def get_css():
    if os.environ.get("CLAUDE_TRANSCRIPTS_DEV"):
        css_path = Path(__file__).parent / "static" / "styles.css"
        return css_path.read_text()
    return CSS  # Embedded constant

def get_js():
    if os.environ.get("CLAUDE_TRANSCRIPTS_DEV"):
        js_path = Path(__file__).parent / "static" / "script.js"
        return js_path.read_text()
    return JS  # Embedded constant
```

#### Build Script
```python
# build.py
def embed_static_files():
    """Embed CSS/JS into constants.py for distribution."""
    css = Path("static/styles.css").read_text()
    js = Path("static/script.js").read_text()

    constants_path = Path("constants.py")
    content = constants_path.read_text()

    # Replace CSS constant
    content = re.sub(
        r'CSS = """.*?"""',
        f'CSS = """{css}"""',
        content,
        flags=re.DOTALL
    )

    # Replace JS constant
    content = re.sub(
        r'JS = """.*?"""',
        f'JS = """{js}"""',
        content,
        flags=re.DOTALL
    )

    constants_path.write_text(content)
```

### Benefits
1. **Syntax highlighting** in editors for CSS/JS
2. **Easier debugging** with proper source files
3. **Linting/formatting** with dedicated tools (Prettier, ESLint)
4. **Future optimization** (minification, bundling)

### Estimated Effort
- Low-medium complexity
- 2-3 hours implementation
- Optional: integrate with pre-commit hooks

---

## Implementation Priority

| Priority | Task | Effort | Dependencies |
|----------|------|--------|--------------|
| 1 | C.1 Subagent Detection | Medium-High | None |
| 2 | A.4 Recursive Nesting | Medium | None (enhances C.1) |
| 3 | Module Splitting | High | Test coverage first |
| 4 | CSS/JS Externalization | Low-Medium | After module splitting |

### Recommended Order
1. **C.1 first** - Addresses user-facing feature gap
2. **A.4 second** - Enables inline subagent expansion
3. **Module splitting third** - Improves maintainability for future work
4. **CSS/JS externalization last** - Nice-to-have, low user impact

---

## Success Criteria

### Phase 3 Complete When:
- [ ] Subagent files are detected and linked
- [ ] Recursive nesting works for at least 2 levels
- [ ] All tests pass after refactoring
- [ ] Code quality score remains above 80/100
- [ ] Documentation updated with new features

### Quality Gates:
- All tests pass: `uv run pytest`
- Code formatted: `uv run black .`
- Snapshot tests updated for intentional changes
- No regressions in existing functionality

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Breaking changes from module split | High | Comprehensive test coverage first |
| Subagent filename variations | Medium | Document expected patterns, add fallbacks |
| Recursive template performance | Low | Limit nesting depth to 5 levels |
| CSS/JS build step complexity | Low | Keep simple, use pre-commit hooks |

---

## Timeline Estimate

| Week | Focus |
|------|-------|
| Week 1 | C.1 Subagent Detection |
| Week 2 | A.4 Recursive Nesting |
| Week 3 | Module Splitting |
| Week 4 | CSS/JS Externalization + Polish |

**Total Estimated Effort**: 15-20 hours

---

## Appendix: Code Snippets

### A. Subagent Timestamp Extraction
```python
import re
from datetime import datetime

AGENT_FILE_PATTERN = re.compile(r"agent-(\d{4}-\d{2}-\d{2}T[\d:.-]+)\.jsonl")

def extract_agent_timestamp(filename: str) -> str | None:
    """Extract timestamp from agent filename."""
    match = AGENT_FILE_PATTERN.match(filename)
    if match:
        return match.group(1)
    return None
```

### B. Recursive Content Detection
```python
def has_nested_content(content_blocks: list) -> bool:
    """Check if content contains nested tool calls."""
    for block in content_blocks:
        if block.get("type") == "tool_use":
            tool_name = block.get("name", "")
            if tool_name in ("Agent", "Task"):
                return True
    return False
```

### C. CSS Variable Design Tokens
```css
:root {
    /* Nesting depth colors */
    --nest-color-1: var(--assistant-border);
    --nest-color-2: var(--user-border);
    --nest-color-3: var(--thinking-border);

    /* Nesting spacing */
    --nest-indent: var(--spacing-md);
    --nest-connector-width: 2px;
}
```
