# Agent-Native Architecture Review: TUI Polish Plan

**Date:** 2026-03-15
**Plan:** `docs/plans/2026-03-15-feat-tui-polish-documentation-plan.md`
**Reviewer:** Claude (agent-native architecture)

## Summary

The plan has a significant agent-accessibility gap. Four capabilities are being added or fixed -- `list_concepts()`, course picker, retry wrong answers, and `get_due_cards()` -- but **none of them get a CLI command**. The Python library functions in `review_db.py` and `history.py` are importable, so an agent running Python can call them directly. However, an agent using `studyctl` as a CLI tool (the standard interface for Claude Code, Kiro, etc.) has **no way to invoke three of the four features**, and the fourth (`get_due_cards`) is conflated with a different concept in the existing `studyctl review` command.

## Capability Map

| Feature | Library Function | CLI Command | Agent-Accessible | Status |
|---------|-----------------|-------------|-------------------|--------|
| List concepts | `history.list_concepts()` (planned A2) | None | Import only | GAP |
| Get due cards (SM-2) | `review_db.get_due_cards(course)` | None (`studyctl review` is topic-level, not card-level) | Import only | GAP |
| Record card review | `review_db.record_card_review(...)` | None | Import only | GAP |
| Record session | `review_db.record_session(...)` | None | Import only | GAP |
| Get course stats | `review_db.get_course_stats(course)` | None | Import only | GAP |
| Get wrong hashes | `review_db.get_wrong_hashes(course)` | None | Import only | GAP |
| Course picker | TUI `CoursePickerScreen` (planned A3) | None | Not accessible | CRITICAL GAP |
| Retry wrong answers | TUI `r` key binding (planned A4) | Excluded (item 6, external repo) | Not accessible | CRITICAL GAP |
| Topic-level review check | `history.spaced_repetition_due()` | `studyctl review` | Yes | OK |
| Record progress | `history.record_progress()` | `studyctl progress` | Yes | OK |
| Progress map | `history.get_progress_for_map()` | `studyctl progress-map` | Yes | OK |
| Wins | `history.get_wins()` | `studyctl wins` | Yes | OK |
| Teach-back | `history.record_teachback()` | `studyctl teachback` | Yes | OK |
| Bridges | `history.record_bridge()` | `studyctl bridge add` | Yes | OK |

**Score: 8/14 capabilities are agent-accessible via CLI. The 6 gaps are all in the SM-2 card-review subsystem.**

## Critical Issues (Must Fix)

### 1. No CLI for `list_concepts()`

- **Location:** Plan item A2 adds `list_concepts()` to `history.py`, wires it to TUI Concepts tab
- **Impact:** An agent cannot list what concepts exist. This is Context Starvation -- the agent cannot see what the user sees in the TUI Concepts tab.
- **Fix:** Add `studyctl concepts` CLI command that calls `list_concepts()` and prints a table. ~15 lines in `cli.py`.

### 2. No CLI for `get_due_cards()`

- **Location:** `review_db.py:156` -- function exists, no CLI wrapper
- **Impact:** The existing `studyctl review` command calls `spaced_repetition_due()` from `history.py`, which is **topic-level** review scheduling (session-based). The SM-2 **card-level** `get_due_cards(course)` from `review_db.py` is a completely different system. An agent cannot query which specific cards are due.
- **Fix:** Add `studyctl cards due <course>` CLI command. Output card hashes, ease factor, interval, next review date.

### 3. No CLI for card review recording

- **Location:** `review_db.py:74` -- `record_card_review()` exists, no CLI wrapper
- **Impact:** An agent cannot conduct a flashcard review session without the TUI. This is an Orphan Feature -- the entire SM-2 card review loop is TUI-only.
- **Fix:** Add `studyctl cards record <course> <card_hash> --correct/--wrong` CLI command. This is the primitive. An agent can then build its own review loop by combining `cards due` + `cards record`.

### 4. Retry flow is TUI-only with no equivalent

- **Location:** Plan item A4 adds `r` key binding, uses in-memory `wrong_hashes`
- **Impact:** The plan explicitly excludes item 6 (`--retry-wrong` flag for `pdf-by-chapters review`) as "external repo, out of scope." This means the retry flow exists **only** as a TUI keybinding. An agent that wants to re-drill wrong cards has no path.
- **Fix:** `get_wrong_hashes(course)` already exists in `review_db.py:197`. Expose it: `studyctl cards wrong <course>`. The agent can then call `cards due` filtered to those hashes. No workflow tool needed -- just the primitive.

## Warnings (Should Fix)

### 5. No CLI for `get_course_stats()`

- **Location:** `review_db.py:225`
- **Recommendation:** Add `studyctl cards stats <course>` to expose total reviews, unique cards, due today, mastered count. An agent needs this to report study progress without launching the TUI dashboard.

### 6. Course discovery is not exposed

- **Location:** Plan A3 adds `CoursePickerScreen` using `discover_directories()` from config
- **Recommendation:** Add `studyctl cards courses` to list available courses with card counts. Without this, an agent must parse the YAML config to figure out valid course names for the `cards` subcommands.

### 7. `studyctl review` vs SM-2 cards naming confusion

- **Location:** `cli.py:553` (`studyctl review`) vs `review_db.py` card system
- **Recommendation:** The existing `review` command shows topic-level spaced repetition. The card-level SM-2 system uses the same vocabulary ("review", "due") but is a completely different subsystem. Use `studyctl cards` as the subcommand group to avoid confusion. Document the distinction in the system prompt / CLI help.

## Observations

### 8. Library API is well-designed for agents

The `review_db.py` functions are already good primitives: `record_card_review()`, `get_due_cards()`, `get_wrong_hashes()`, `get_course_stats()`. They accept `db_path` overrides, return typed data, and have no UI coupling. The gap is purely the missing CLI wrappers.

### 9. Plan item 6 exclusion is the right call but needs a follow-up

Excluding `--retry-wrong` from `pdf-by-chapters` (external repo) is correct scoping. But the plan should note that CLI parity for retry is needed in `studyctl` itself via the primitives described above.

## Recommended CLI Additions (Prioritised)

```
studyctl cards due <course>        -- list cards due for review (wraps get_due_cards)
studyctl cards record <course> <hash> --correct/--wrong  -- record a review (wraps record_card_review)
studyctl cards wrong <course>      -- list wrong hashes from last session (wraps get_wrong_hashes)
studyctl cards stats <course>      -- show course statistics (wraps get_course_stats)
studyctl cards courses             -- list available courses (wraps discover_directories)
studyctl concepts                  -- list tracked concepts (wraps list_concepts)
```

These are all primitives (not workflows). An agent composes them:
- "What is due?" -> `cards due <course>`
- "Drill me on wrong answers" -> `cards wrong <course>` then iterate with `cards record`
- "How am I doing?" -> `cards stats <course>` + `concepts`

## What Is Working Well

- `review_db.py` functions are clean primitives with no UI coupling
- `db_path` parameter on every function enables testing and agent flexibility
- Existing CLI commands (`progress`, `wins`, `teachback`, `bridge`) follow good patterns
- The plan's A4 design (in-memory retry, no SM-2 writes) is correct SM-2 behaviour
- Agent installation script (`install-agents.sh`) shows agent-native awareness

## Verdict

**NEEDS WORK** -- 8/14 capabilities accessible, 6 gaps all in the card-review subsystem. The library layer is solid; the gap is CLI exposure. Estimated effort: ~80 lines of Click commands in `cli.py` to reach full parity.
