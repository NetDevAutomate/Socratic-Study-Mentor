# Architecture Review: TUI Study App Plan (A1-A6)

**Date**: 2026-03-15
**Reviewer**: Architecture Strategist Agent
**Scope**: Plan items A1 through A6

---

## 1. Architecture Overview

Monorepo with two packages sharing `~/.config/studyctl/sessions.db`:

- **agent-session-tools** owns: `sessions`, `messages`, `session_tags`, `session_notes`, `*_embeddings`
- **studyctl** owns: `card_reviews`, `review_sessions` (via `review_db.py`), plus planned `study_progress`/`concepts`

TUI is an optional extra (`studyctl[tui]`), excluded from pyright, guarded by try/except ImportError. This boundary is clean and correct.

---

## 2. Per-Item Assessment

### A1 (export fix) -- CLEAN
Package boundary respected: change is entirely within agent-session-tools. A 4-line display fix is low risk. The `source_stats` dual-type issue (dict vs ExportStats) is a legitimate smell but acceptable to defer as noted.

### A2 (list_concepts in history.py) -- CONCERN: God Module
history.py is already 950 lines with 25 functions spanning unrelated domains: medication windows, teachback recording, bridge graphs, concept seeding, streak tracking, spaced repetition. Adding 3 more concept functions deepens an existing Single Responsibility violation.

**Verdict**: The plan should extract concepts now, not defer it. The file already has natural clusters:

| Cluster | Functions | Target Module |
|---------|-----------|---------------|
| Study analytics | topic_frequency, struggle_topics, last_studied, get_wins | `analytics.py` |
| Teachback/bridges | record_teachback, get_teachback_history, record_bridge, get_bridges, update_bridge_usage, migrate_bridges_to_graph | `teachback.py` |
| Concepts | seed_concepts_from_config + new list/get/search | `concepts.py` |
| Session tracking | start_study_session, end_study_session, get_study_session_stats | stays in history.py |
| Spaced repetition | spaced_repetition_due, get_progress_for_map, get_progress_summary, record_progress | `progress.py` |

Recommendation: At minimum, extract `concepts.py` before adding to it. The full decomposition can be a separate ticket but the concepts cluster is self-contained and new -- no excuse to add it to the god module.

### A3 (course picker) -- DESIGN RISK: Course Name Consistency
The plan uses directory basename as course name. `record_card_review()` takes `course: str` and stores it directly. This means the course identity is just a raw directory name with no normalisation.

**Risk**: Renaming a directory silently orphans all historical review data. There is no `courses` table, no stable identifier.

**Recommendations**:
1. Normalise the course name (e.g., `slugify(basename)`) at the point of discovery in `review_loader.discover_directories()`.
2. Document that course identity = normalised directory basename.
3. Consider a future `courses` table with stable IDs (can be deferred, but the normalisation cannot).

Placing picker logic in `_launch_study()` on the parent App rather than in StudyCardsTab is architecturally correct -- the parent owns navigation and lifecycle; the child widget owns study interaction. This follows the Textual convention of screens/apps managing widget creation.

### A4 (retry mode) -- ACCEPTABLE COMPLEXITY
Current study_cards.py is ~342 lines with a straightforward linear state machine: show card -> reveal -> score -> next -> summary. Adding retry mode introduces:

- `_is_retry` boolean flag
- `_wrong_hashes` list (already tracked via `ReviewResult.wrong_hashes`)
- 2 extra states (retry prompt, retry loop)
- SM-2 skip during retry (correct -- retry reviews are not true spaced repetition events)
- No nested retries (good constraint)

This is manageable within the existing widget. The in-memory approach (no DB migration) is the right call -- retry is a UI-session concept, not a persistence concept.

**One issue**: The `_is_retry` boolean plus the implicit state (current_index position, summary shown) creates a mini state machine that is not explicitly modelled. Consider an enum:

```python
class StudyPhase(Enum):
    STUDYING = "studying"
    SUMMARY = "summary"
    RETRY_PROMPT = "retry_prompt"
    RETRYING = "retrying"
```

This would make the state transitions explicit and testable without adding significant complexity.

### A5 (get_due_cards SQL fix) -- CRITICAL, CORRECT
The current query is buggy:

```sql
GROUP BY card_hash
HAVING reviewed_at = MAX(reviewed_at)
```

This is non-deterministic in SQLite -- `reviewed_at` in the HAVING clause references an arbitrary row from the group, not the MAX. The ROW_NUMBER() window function fix is the correct approach:

```sql
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY card_hash ORDER BY reviewed_at DESC) AS rn
    FROM card_reviews
    WHERE course = ? AND next_review <= ?
)
SELECT ... FROM ranked WHERE rn = 1
```

This should be prioritised -- the current query can return stale review data, causing cards to appear due when they are not (or vice versa).

### A6 (suppress narrowing) -- CORRECT
Two `suppress(Exception)` calls at lines 218 and 263 in study_cards.py silently swallow all errors from `record_card_review()` and `record_session()`. This includes:

- `sqlite3.OperationalError` (DB locked, schema mismatch)
- `TypeError` (API contract violations)
- `ValueError` (bad data)

Replacing with `try/except sqlite3.Error` + `logging.warning()` is correct. The TUI should not crash on DB write failures, but it absolutely must not swallow programming errors silently.

**Additional**: The `except Exception: pass` in `_speak()` (line ~280) should also be narrowed to specific kokoro/audio exceptions.

---

## 3. Deferred Items Assessment

| Deferred Item | Verdict |
|---------------|---------|
| session_id FK linking review_sessions to agent-session-tools sessions | Correct to defer -- crosses package boundary, needs migration coordination |
| concepts.py extraction from history.py | **Should NOT be deferred** -- extract before adding new functions (see A2) |
| ExportStats normalisation | Acceptable to defer -- contained within agent-session-tools, no cross-boundary impact |

---

## 4. Risk Summary

| Risk | Severity | Item | Mitigation |
|------|----------|------|------------|
| history.py god module growing unchecked | Medium | A2 | Extract concepts.py now |
| Course name instability | Medium | A3 | Add slugify normalisation |
| Buggy get_due_cards SQL | High | A5 | Prioritise this fix |
| Silent exception swallowing | Medium | A6 | Already planned, include _speak() too |
| Implicit state machine in study_cards.py | Low | A4 | Add StudyPhase enum |

---

## 5. Recommended Execution Order

1. **A5** -- SQL correctness fix (data integrity, no dependencies)
2. **A6** -- suppress narrowing (quick win, improves debuggability for all subsequent work)
3. **A1** -- export fix (isolated to agent-session-tools, independent)
4. **A2** -- list_concepts, but extract to `concepts.py` first
5. **A3** -- course picker (depends on A2 for concepts tab, benefits from A5 fix)
6. **A4** -- retry mode (depends on A3 for course selection, A5 for correct due-card logic)
