# Brainstorm: `studyctl clean` Command — 2026-04-03

## What We're Building

A `studyctl clean` command that removes orphaned artifacts from crashed or stale study sessions. Three cleanup targets:

1. **Stale tmux sessions** — Kill `study-*` sessions with no child process (zombies) or where the agent has exited
2. **Session directories** — Remove `~/.config/studyctl/sessions/<name>/` dirs for ended sessions (contains `CLAUDE.md`, `studyctl` wrapper script)
3. **Stale state file** — Reset `session-state.json` when `mode=ended` and no matching tmux session exists

**Not in scope:** IPC files (`session-topics.md`, `session-parking.md`, `session-oneline.txt`) — left for potential debugging.

## Why This Approach

- These artifacts accumulate from crashed sessions, tmux-resurrect restoring killed sessions, or manual intervention
- The Q-quit and `_cleanup_session()` paths handle the happy path; `clean` handles the unhappy path
- Keeps the command simple — no interactive prompts, just action + report

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Behavior | Silent action + summary report | Clean is a maintenance tool, not a risky delete. `--dry-run` flag for preview |
| Session dir cleanup | Only ended sessions | Never delete dirs for active/paused sessions |
| State file | Delete only when mode=ended + no tmux session | Preserves resume capability for paused sessions |
| IPC files | Excluded | Small, useful for debugging, already cleared on next session start |

## Existing Code to Reuse

- `tmux.kill_all_study_sessions()` — already kills all `study-*` sessions
- `tmux.pane_has_child_process()` — zombie detection
- `session_state.read_session_state()` — read current state
- `session_state.clear_session_files()` — file cleanup
- `tmux.list_sessions()` or equivalent — enumerate sessions

## Open Questions

None — scope is well-defined. Proceed to implementation.
