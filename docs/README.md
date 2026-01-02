# Documentation

This directory contains comprehensive documentation for the claude-code-transcripts project.

## Documents

### 📊 [ARCHITECTURE_ANALYSIS.md](./ARCHITECTURE_ANALYSIS.md)
**1,890 lines | Complete Technical Specification**

Comprehensive analysis covering:
1. **Core Flows** - Session discovery, parsing, message assembly
2. **Sub-Agent Connections** - Tool linking, message attribution
3. **Object Schemas** - Complete specification of all data structures
4. **Componentization Plan** - Proposed module structure with migration strategy

### 📋 [ARCHITECTURE_SUMMARY.md](./ARCHITECTURE_SUMMARY.md)
**Quick Reference | Executive Summary**

Condensed reference document with:
- Quick reference for core flows
- Tool linking mechanism overview
- Key object schemas
- Module boundaries (current and proposed)
- Key findings and next steps

### 🔍 [SCHEMA_ANALYSIS_REPORT.md](./SCHEMA_ANALYSIS_REPORT.md)
**Schema Consolidation | Developer Response Analysis**

Comprehensive schema analysis from Claude Code artifact investigation:
- Cross-comparison of 5 developer responses
- Final de-duplicated list of 19 unique object types
- Conflicts identified and resolved with existing documentation
- Flow and functionality validation
- Recommendations for schema formalization and sub-agent support

### 🎨 [ARCHITECTURE_DIAGRAMS.md](./ARCHITECTURE_DIAGRAMS.md)
**Visual Documentation | Flow Charts**

Visual representations including:
- Current system architecture diagram
- Data flow: Session to HTML
- Tool pairing mechanism
- Proposed module structure
- Content block rendering pipeline
- Session discovery flow chart

### 📝 [IMPLEMENTATION_PLAN.md](./IMPLEMENTATION_PLAN.md)
**Implementation Roadmap**

Detailed plan for implementing the proposed architecture changes.

## Quick Navigation

### Understanding the System

**New to the project?** Start here:
1. Read [ARCHITECTURE_SUMMARY.md](./ARCHITECTURE_SUMMARY.md) for overview
2. Look at [ARCHITECTURE_DIAGRAMS.md](./ARCHITECTURE_DIAGRAMS.md) for visual understanding
3. Dive into [ARCHITECTURE_ANALYSIS.md](./ARCHITECTURE_ANALYSIS.md) for details

**Need specific information?**
- **Session Discovery:** ARCHITECTURE_ANALYSIS.md § 1.1
- **Session Parsing:** ARCHITECTURE_ANALYSIS.md § 1.2
- **Message Assembly:** ARCHITECTURE_ANALYSIS.md § 1.3
- **Tool Linking:** ARCHITECTURE_ANALYSIS.md § 2.2
- **Object Schemas:** ARCHITECTURE_ANALYSIS.md § 3
- **Componentization:** ARCHITECTURE_ANALYSIS.md § 4

### Implementing Changes

**Planning a refactor?**
1. Review [ARCHITECTURE_ANALYSIS.md § 4.4](./ARCHITECTURE_ANALYSIS.md#44-migration-strategy) for migration strategy
2. Check [IMPLEMENTATION_PLAN.md](./IMPLEMENTATION_PLAN.md) for roadmap
3. Follow the 10-phase migration plan

**Adding a new feature?**
1. Determine which module it belongs to (see § 4.3 in ARCHITECTURE_ANALYSIS.md)
2. Check if the module exists or needs to be created
3. Follow the proposed API patterns

## Key Statistics

- **Current Codebase:** 2,994 lines in single file
- **Content Block Types:** 5 (text, thinking, tool_use, tool_result, image)
- **Tool Renderers:** 4 specialized + 1 generic
- **Tool Input Types:** 9 documented (Write, Edit, Bash, TodoWrite, Task, TaskOutput, Read, Glob, Grep)
- **Entry Types:** 4 (user, assistant, queue-operation, file-history-snapshot)
- **Session Formats:** 2 (JSON, JSONL)
- **CLI Commands:** 4 (local, web, json, all)
- **Proposed Modules:** 8 core modules
- **Total Unique Types:** 19 (documented in SCHEMA_ANALYSIS_REPORT.md)

## Analysis Highlights

### Core Flows Identified

1. **Session Discovery** (347 lines of code)
   - File-based scanning with filtering
   - Summary extraction and caching
   - Sorting by modification time

2. **Session Parsing** (49 lines of code)
   - Format detection and normalization
   - JSONL → Standard format conversion
   - Type filtering (user/assistant only)

3. **Message Assembly** (225+ lines of code)
   - Conversation grouping by user prompts
   - Tool call/result pairing via IDs
   - Pagination (5 conversations per page)

### Key Findings

✅ **What Works Well:**
- Tool pairing system is robust and reliable
- Rendering is flexible and extensible
- Template system is well-organized

⚠️ **Areas for Improvement:**
- Monolithic structure (2,994 lines in one file)
- Schema validation not implemented
- Sub-agent discovery disabled (format is supported)
- Limited caching

🎯 **Recommended Actions:**
1. Extract utilities (low risk, high value)
2. Add session validation with Pydantic models
3. Implement summary caching
4. Enable optional sub-agent file discovery
5. Add specialized Task tool renderer

## Contributing

When modifying the codebase:

1. **Understand the Context:**
   - Read relevant sections in ARCHITECTURE_ANALYSIS.md
   - Review the data flow diagrams
   - Check for related functions

2. **Follow Patterns:**
   - Use existing rendering patterns for new tools
   - Follow the tool pairing mechanism for new features
   - Maintain consistency with current code style

3. **Update Documentation:**
   - Update ARCHITECTURE_ANALYSIS.md if changing core flows
   - Update diagrams if modifying architecture
   - Add new schemas to § 3 when introducing new types

4. **Test Thoroughly:**
   - Write unit tests for new functions
   - Run full test suite: `uv run pytest`
   - Test with sample session files

## Related Files

- **[../README.md](../README.md)** - User-facing documentation
- **[../AGENTS.md](../AGENTS.md)** - Development guide
- **[../TASKS.md](../TASKS.md)** - Implementation roadmap
- **[../src/claude_code_transcripts/__init__.py](../src/claude_code_transcripts/__init__.py)** - Main implementation

## Document Maintenance

These documents were generated through comprehensive code analysis on 2026-01-01.

**To update these documents:**
1. Re-analyze the codebase after major changes
2. Update schemas when new content blocks are added
3. Revise flow diagrams if the pipeline changes
4. Update statistics and line numbers

**Version History:**
- v1.1 (2026-01-02) - Schema consolidation update
  - Added SCHEMA_ANALYSIS_REPORT.md with complete schema definitions
  - Updated LogLine schema with uuid, parentUuid, sessionId, agentId fields
  - Added signature field to ThinkingBlock
  - Added Task, TaskOutput, Read, Glob, Grep tool schemas
  - Corrected sub-agent support documentation (format supported, discovery disabled)
  - Expanded entry types to include queue-operation and file-history-snapshot
- v1.0 (2026-01-01) - Initial comprehensive analysis
  - Full core flows documentation
  - Complete schema specification
  - Componentization proposal

---

**Questions or issues?** Open an issue on GitHub or refer to the main [README.md](../README.md).
