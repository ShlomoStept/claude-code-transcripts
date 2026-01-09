# Repository Cleanup and Upstream PR Preparation - Implementation Plan

## Goal

Clean up this fork of `simonw/claude-code-transcripts` and prepare a comprehensive PR to submit all improvements upstream.

---

## Phase 1 Analysis Summary

### Branches (11 local, 23 remote)

**Local Branches:**
| Branch | Status | Action |
|--------|--------|--------|
| `main` | Primary branch with all merged features | KEEP |
| `cleanup/prepare-upstream-pr` | Current work branch | KEEP (temporary) |
| `analysis/codebase-branch-review-2026-01-05` | Analysis only, no code | DELETE |
| `evaluation/markdown-json-rendering-fixes` | Open PR #13 | CLOSE PR, DELETE |
| `feature/phase2-collapsible-cells` | Merged to main | DELETE |
| `feature/phase3-planning` | Planning docs only | DELETE |
| `feature/phase3-subagent-detection` | Open PR #10, has FEATURE_PROPOSALS.md | EVALUATE |
| `feature/quality-improvements` | Stale | DELETE |
| `fix/markdown-json-rendering-v2` | Open PR #12 | CLOSE PR, DELETE |
| `fix/markdown-rendering-issues` | Open PR #11 | CLOSE PR, DELETE |
| `fix/technical-debt` | Merged to main | DELETE |

**Remote-only Branches to Delete:**
- `origin/claude/*` - Claude-generated feature branches (not merged)
- `origin/copilot/*` - Copilot analysis branches

### Open PRs (4 open, 6 merged, 3 closed)

| PR | Title | Branch | Action |
|----|-------|--------|--------|
| #13 | Fix content block array rendering | evaluation/markdown-json-rendering-fixes | CLOSE |
| #12 | Fix markdown/JSON rendering issues (v2) | fix/markdown-json-rendering-v2 | CLOSE |
| #11 | Fix Markdown/JSON rendering issues | fix/markdown-rendering-issues | CLOSE |
| #10 | Phase 3: C.1 Subagent Detection | feature/phase3-subagent-detection | EVALUATE - may have useful code |

### Large Files in Git History

5 PNG screenshots in `docs/snapshots_of_current_UI/` totaling ~9MB:
- Screenshot 2025-12-31 at 11.31.34 PM.png (1.75MB)
- Screenshot 2025-12-31 at 11.31.42 PM.png (1.87MB)
- Screenshot 2025-12-31 at 11.31.46 PM.png (1.82MB)
- Screenshot 2025-12-31 at 11.31.53 PM.png (2.05MB)
- Screenshot 2025-12-31 at 11.52.44 PM.png (1.34MB)

### Documents to Remove

**Internal Planning Documents (remove from upstream PR):**
- `PLAN.md` - Phase 2 regression fix planning
- `TASKS.md` - Internal task tracking
- `EVALUATION_REPORT.md` - Internal evaluation
- `TODO.md` - Internal todos
- `implementation_plan.md` - Current planning doc
- `task.md` - Current task tracking
- `walkthrough.md` - Internal walkthrough
- `2025-12-31-systemrules.txt` - System rules transcript
- `2026-01-01-systemrules.txt` - System rules transcript
- `2026-01-06-systemrules.txt` - System rules transcript
- `docs/IMPLEMENTATION_PLAN.md` - Internal planning
- `docs/snapshots_of_current_UI/` - Internal screenshots

**Documents to Keep:**
- `README.md` - Updated with improvements
- `AGENTS.md` - Development guide (useful for contributors)
- `CLAUDE.md` - Project instructions

---

## Phase 2: Document Cleanup

> [!IMPORTANT]
> This will remove internal planning documents that should not be in the upstream PR.

**Files to Remove:**
```
git rm PLAN.md TASKS.md EVALUATION_REPORT.md TODO.md
git rm implementation_plan.md task.md walkthrough.md
git rm 2025-12-31-systemrules.txt 2026-01-01-systemrules.txt 2026-01-06-systemrules.txt
git rm -r docs/
```

---

## Phase 3: Git History Cleanup

> [!WARNING]
> Purging files from git history is a destructive operation. This will rewrite history and require force push.

**Files to Purge from History:**
- `docs/snapshots_of_current_UI/**/*.png` (9MB of screenshots)
- `*-systemrules.txt` files

**Method:**
```bash
# Using git filter-repo (preferred)
pip install git-filter-repo
git filter-repo --path docs/snapshots_of_current_UI --invert-paths
git filter-repo --path-glob '*-systemrules.txt' --invert-paths
```

---

## Phase 4: Branch and PR Cleanup

**PRs to Close:**
- #13, #12, #11 - Superseded by merged work
- #10 - Evaluate if subagent detection code should be merged first

**Local Branches to Delete:**
- analysis/codebase-branch-review-2026-01-05
- feature/phase2-collapsible-cells
- feature/phase3-planning
- feature/quality-improvements
- fix/markdown-json-rendering-v2
- fix/markdown-rendering-issues
- fix/technical-debt

**Remote Branches to Delete:**
- All `origin/claude/*` branches
- All `origin/copilot/*` branches
- Branches for closed/merged PRs

---

## Phase 5: Merge Remaining Features

**Check PR #10 (Subagent Detection):**
- Review if the feature is complete and tested
- If valuable, merge to main before cleanup

---

## Phase 6: Upstream PR

**Commits Ahead of Upstream:** 32 commits

**Key Features to Highlight:**

1. **ANSI Escape Code Sanitization**
   - Strips terminal escape codes from tool output
   - Supports CSI, OSC, and C1 sequences

2. **Content Block Array Rendering**
   - Properly renders `[{"type": "text", ...}]` arrays as markdown
   - Supports text and thinking blocks

3. **Syntax Highlighting**
   - Pygments integration for code blocks
   - Supports 500+ languages

4. **Copy Buttons**
   - Per-code-block copy buttons
   - Clipboard API with fallback

5. **Tool Call/Result Pairing**
   - Groups tool_use with corresponding tool_result
   - Matched by tool ID

6. **Collapsible Cell Structure**
   - Thinking cell (closed by default)
   - Response cell (open by default)
   - Tools cell (closed by default, shows count)

7. **Per-Cell Copy Buttons**
   - Copy entire cell contents
   - ARIA labels for accessibility

8. **Markdown Rendering in Tools**
   - Tool descriptions render as markdown
   - JSON string values with inline markdown

9. **Message Metadata**
   - Character count
   - Token estimate
   - Tool call counts

10. **Tool Icons**
    - 14 tool-specific icons
    - Markdown/JSON view toggle

**Test Coverage:**
- 140+ tests passing
- Comprehensive snapshot tests

---

## Verification Plan

1. **Automated Tests:**
   ```bash
   uv run pytest
   uv run black . --check
   ```

2. **Manual Verification:**
   - Generate HTML from sample session
   - Verify all features render correctly
   - Test in multiple browsers

3. **Diff Review:**
   - Review final diff against upstream
   - Ensure no internal documents included
   - Verify git history is clean

---

## Proposed Changes Summary

| Component | Action |
|-----------|--------|
| Internal docs (10+ files) | DELETE |
| Screenshot images (5 files) | DELETE + PURGE from history |
| Open PRs (4) | CLOSE |
| Local branches (9) | DELETE |
| Remote branches (15+) | DELETE |
| Upstream PR | CREATE with comprehensive description |

---

> [!IMPORTANT]
> **User Approval Required**
>
> Please review this plan and confirm:
> 1. OK to close PRs #10, #11, #12, #13?
> 2. OK to delete internal planning documents?
> 3. OK to purge screenshots from git history (requires force push)?
> 4. Any documents you want to keep that are listed for removal?
