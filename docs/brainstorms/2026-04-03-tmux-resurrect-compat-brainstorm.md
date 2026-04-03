# Brainstorm: tmux-resurrect Compatibility — 2026-04-03

## What We're Building

Belt-and-suspenders approach to prevent tmux-resurrect from creating zombie study sessions:

1. **Auto-clean on startup** — Run zombie cleanup logic at top of `studyctl study` before creating new session. Uses existing `plan_clean()` / `_clean_logic.py`. Zero user action required.

2. **Documented restore hook** — Ship a `@resurrect-restore-hook` config snippet in `setup-guide.md` that users paste into `~/.tmux.conf`. Kills `study-*` sessions immediately after resurrect restore.

3. **Doctor check** — Add a `studyctl doctor` check that detects tmux-resurrect plugin and warns if the restore hook is not configured.

## Why This Approach

- Auto-clean handles the common case with zero user friction
- The hook prevents zombies at the source for users who configure it
- Doctor check educates users about the issue and nudges toward the hook
- No approach can fully automate resurrect exclusion without touching `~/.tmux.conf`

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Primary defence | Auto-clean on startup | Zero user action, uses existing FCIS logic |
| Secondary defence | Restore hook snippet | Prevents saving, one-time user setup |
| Detection | Doctor check | Non-blocking education |
| Resurrect save filtering | Not pursuing | Brittle (depends on resurrect file format internals) |

## Scope

- Modify `_study.py:_handle_start()` to call cleanup before session creation
- Update `docs/setup-guide.md` with restore hook snippet
- Add resurrect detection + hook check to `_doctor.py`
- Tests for auto-clean-on-startup path
