# Brainstorm: Study Backlog Phase 1 — 2026-04-03

## What We're Building

Cross-session study backlog surfaced via `studyctl topics list/add/resolve`. The backlog is a **query layer over existing data** — no new table. Uses `parked_topics` as the single store for all backlog items (parked, struggled, manually added).

### Commands

- `studyctl topics list` — show pending backlog items across all sessions
- `studyctl topics add "topic" --tech "Python" --note "..."` — manually add to backlog
- `studyctl topics resolve <id>` — mark as resolved
- Agent surfaces backlog count at session start

## Why This Approach

- `parked_topics` table (v14) already has status tracking, session refs, timestamps
- Creating a separate `study_backlog` table would duplicate data and add sync complexity
- Query layer is simpler: `SELECT * FROM parked_topics WHERE status='pending'`
- Struggled topics get persisted to the same table at session end — one code path

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Data model | Query layer over `parked_topics` | No new table, no data duplication, YAGNI |
| Struggled topics | Persist to `parked_topics` at session end | Scan `session-topics.md` for `status:struggling`, insert with `source='struggled'` |
| Manual add | NULL session refs | Migration v16 relaxes FK, `source='manual'` distinguishes them |
| Agent surfacing | Inject backlog count into agent persona | "You have 3 outstanding Python topics" at session start |

## Migration v16

- Make `study_session_id` and `session_id` NULLable on `parked_topics`
- Add `source` column: `CHECK('parked', 'struggled', 'manual')` DEFAULT 'parked'
- Add `tech_area` column (TEXT, nullable) for technology categorization

## Scope

### In scope (Phase 1)
- CLI commands: `topics list`, `topics add`, `topics resolve`
- Migration v16: nullable FKs + source + tech_area columns
- Auto-persist struggled topics at session end
- Agent persona injection at session start
- Tests: unit for query logic (FCIS), CLI tests

### Out of scope (Phase 2)
- AI-driven categorization
- Priority scoring from history
- `studyctl topics suggest` command
