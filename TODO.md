# TODO.md

Brutally honest backlog for closing the current gaps. Priorities are ordered
within each tier.

P0 - Core correctness (must fix before calling the feature complete)

- Render all content-block array types (image, tool_use, tool_result) using the
  same code path as top-level blocks, so arrays in tool_result output are not
  degraded to raw JSON or plain text.
- Fix ANSI sanitization so it strips CSI, OSC, and non-alphabetic terminators.
  Current regex misses common sequences and can leak control bytes into HTML.
- Add tests for image/tool_use blocks inside content-block arrays and for ANSI
  edge cases (OSC title, cursor control, malformed sequences).

P1 - UX + structural gaps (should follow immediately)

- Pair tool_use with tool_result by tool id, render them as a single grouped UI
  component, and add tests for ordering/missing pairs.
- Improve copy button accessibility: keyboard focus, ARIA labels, and fallback
  behavior when Clipboard API is unavailable.
- Add tests for invalid/mixed content-block arrays to prevent regressions.

P2 - Follow-on improvements

- Highlight Bash tool commands using the existing Pygments pipeline.
- Add metadata subsection and collapsible cell sections once core rendering is
  correct, to avoid building on broken output.

Definitions of done

- All content-block types render correctly both as top-level blocks and when
  embedded inside tool_result arrays.
- ANSI sanitization handles CSI, OSC, and edge sequences with tests proving it.
- Snapshot updates are intentional and backed by new tests.
