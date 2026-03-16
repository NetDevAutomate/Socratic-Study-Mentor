# Plan Review: TUI Polish, Documentation & Phase 1 Cleanup

**Reviewer:** Kieran (Python reviewer agent)
**Date:** 2026-03-15
**Verdict:** APPROVE with issues to address before implementation

---

## 1. The `Concept` Dataclass Design (A2) -- APPROVE

Extracting a typed `Concept` dataclass with `id`, `name`, `domain`, `description`, `relation_count` is the right call. This is a clean value object that gives you type safety at call sites instead of passing around `dict` or `sqlite3.Row`.

**One concern:** The plan says "SQL includes LEFT JOIN to concept_relations for aggregation." Make sure `relation_count` is typed as `int` with a default of `0` (not `int | None`), since `COUNT(*)` with a `LEFT JOIN` and `GROUP BY` will always produce a number. This avoids every caller needing a `or 0` guard.

```python
# Good
@dataclass(frozen=True, slots=True)
class Concept:
    id: int
    name: str
    domain: str
    description: str
    relation_count: int = 0
```

Use `frozen=True` and `slots=True` -- these are read-only query results, not mutable state. This also makes them hashable for free, which is useful if you ever need to deduplicate.

---

## 2. `ExportStats` Normalisation (A1) -- APPROVE, but fix the root cause

The bug is real: line 182 displays `batch_stats.added` (cumulative) when it should show per-source deltas. The plan says "normalise exporter return types to ExportStats dataclass" but `ExportStats` already exists in `exporters/base.py` with an `__iadd__` method.

**The actual problem is the dual-type handling.** Lines 169-177 have `isinstance(source_stats, dict)` vs `getattr` branches. This means some exporters return `dict` and others return `ExportStats`. The fix is:

1. Enforce the `SessionExporter` protocol -- all exporters MUST return `ExportStats` from `export_all()`.
2. Kill the `isinstance(source_stats, dict)` branch entirely.
3. Track a `source_stats` separately from `batch_stats` for the progress bar display.

The `__iadd__` on `ExportStats` already supports clean accumulation. Use it:

```python
source_stats = exporter.export_all(conn, incremental)
batch_stats += source_stats
# Display source_stats (not batch_stats) in progress description
progress.update(task, description=f"{source.title()}: {source_stats.added} added, {source_stats.updated} updated")
```

Do NOT add a `__sub__` method to compute deltas. That is needless complexity. Just use the per-source stats directly.

---

## 3. State Machine Pattern for TUI (A4) -- CONDITIONAL APPROVE

The four states (`reviewing` / `summary` / `retrying` / `retry_summary`) are the right granularity. No nested retries is correct YAGNI. However, I have concerns about how this gets implemented:

**FAIL: Implicit state via booleans.** Do NOT add `self._is_retrying: bool` and `self._showing_summary: bool` as separate flags. That gives you 4 boolean combinations but only 4 valid states -- a classic state explosion bug.

**PASS: Use a `StrEnum` for the state.**

```python
class ReviewPhase(StrEnum):
    REVIEWING = "reviewing"
    SUMMARY = "summary"
    RETRYING = "retrying"
    RETRY_SUMMARY = "retry_summary"
```

Then `self._phase: ReviewPhase` is your single source of truth. Footer bindings change based on `self._phase`. The `watch_` pattern in Textual makes this clean -- you can use `watch__phase` to update bindings reactively.

**Critical: Dynamic footer bindings in Textual.** The plan says "dynamic footer bindings" but does not specify the mechanism. In Textual, you cannot just mutate `BINDINGS` at runtime. You need to override `_get_bindings()` or use `action_*` methods that check phase before executing. Make sure the implementation plan specifies which Textual pattern to use. Consult the Context7 docs for `Binding` and `action` patterns.

---

## 4. `session_id` FK Migration (A4 prerequisite) -- APPROVE with caveats

Nullable FK for backwards compatibility is the standard SQLite migration pattern. This is correct.

**Caveats:**

- **Migration must be idempotent.** SQLite's `ALTER TABLE ADD COLUMN` will fail if the column already exists. Use the `PRAGMA table_info()` check pattern you already use elsewhere, or wrap in a try/except on `sqlite3.OperationalError` (NOT a bare `Exception`).
- **Add the `is_retry` column to `review_sessions` in the same migration.** Do not split these into two migrations -- they are one logical change.
- **Index the FK.** `CREATE INDEX IF NOT EXISTS idx_card_reviews_session ON card_reviews(session_id)` -- foreign keys without indexes are a performance trap on any non-trivial dataset.
- **Do NOT enable `PRAGMA foreign_keys` enforcement retroactively** on the studyctl DB unless you are certain all existing rows satisfy the constraint. Nullable FKs with `ON DELETE SET NULL` is the safe approach.

---

## 5. `concepts.py` Extraction from `history.py` (A2) -- STRONG APPROVE

`history.py` is 953 lines. It contains study sessions, teach-back scoring, knowledge bridges, medication tracking, progress recording, and concept queries. This is well past the extraction signal threshold.

The plan to extract concept-related functions into `concepts.py` is exactly right. Looking at the grep output, the concept-related code touches: `record_progress`, `get_progress_for_map`, `get_wins`, `get_progress_summary`, `record_teachback_score`, `get_teachback_history`, and the bridge functions.

**Recommendation:** Extract in two phases:
1. `concepts.py` -- study progress and concept CRUD (record_progress, get_progress_for_map, get_wins, get_progress_summary)
2. Leave bridges in `history.py` for now OR extract to `bridges.py` -- do NOT mix concept CRUD and bridge logic in the same new module

The `_connect()` helper is duplicated across every function in `history.py`. When extracting, consider whether `concepts.py` should accept an optional `conn` parameter (see item 7 below) rather than duplicating the connect/close pattern yet again.

---

## 6. `suppress(sqlite3.Error)` Recommendation -- APPROVE, but scope it further

The plan correctly identifies that `suppress(Exception)` at lines 218 and 263 of `study_cards.py` is too broad. However, `sqlite3.Error` may still be too broad for line 218 (the `record_session` call) vs line 263 (the `record_card_review` call).

**What can actually go wrong here?**
- Database file missing or locked: `sqlite3.OperationalError`
- Schema mismatch (table not created): `sqlite3.OperationalError`
- Constraint violation: `sqlite3.IntegrityError`

All of these are subclasses of `sqlite3.DatabaseError` which is a subclass of `sqlite3.Error`. Using `sqlite3.OperationalError` is too narrow (misses integrity errors). Using `sqlite3.Error` is acceptable and captures the right failure domain without swallowing unrelated exceptions like `TypeError` from bad arguments.

**Final recommendation:** `suppress(sqlite3.Error)` is the right level. But add a logging call before the suppress so failures are not completely silent:

```python
try:
    record_session(...)
except sqlite3.Error:
    logger.debug("Failed to record review session to DB", exc_info=True)
```

This is better than `suppress` because you get diagnostics without disrupting the user. The `suppress` context manager is fine for truly fire-and-forget operations, but review tracking is important enough to log.

---

## 7. Optional `conn` Parameter on Hot-Path Functions -- CONDITIONAL APPROVE

The current `review_db.py` creates a new `sqlite3.connect()` call for every single function invocation. During a review session, `record_card_review` is called once per card, each time opening and closing a connection. This is wasteful.

**The pattern is sound in principle** -- accepting an optional `conn: sqlite3.Connection | None = None` and falling back to creating one internally. However:

**FAIL: Do not just add `conn` as a parameter and leave the caller to manage lifecycle.** This creates a confusing API where sometimes the function commits and sometimes it does not.

**PASS: Use a context manager pattern instead.**

```python
from contextlib import contextmanager

@contextmanager
def _db_connection(db_path: Path | None = None) -> Iterator[sqlite3.Connection]:
    path = db_path or _get_db()
    ensure_tables(path)
    conn = sqlite3.connect(path)
    try:
        yield conn
        conn.commit()
    except sqlite3.Error:
        conn.rollback()
        raise
    finally:
        conn.close()
```

Then the `StudyCardsTab` can hold a single connection for the session lifetime and pass it in, while standalone callers get the auto-managed path. The key contract is: **if you pass a conn, you own the commit/close lifecycle. If you do not, the function manages it.**

This must be documented in the function docstrings. An undocumented conn parameter is a maintenance trap.

**Alternative (simpler, recommended for now):** Just open one connection in `StudyCardsTab.__init__` and close it in `on_unmount`. Pass it through to all review_db calls. This avoids the complexity of a dual-path API and the TUI widget already has a clear lifecycle. The context manager pattern can come later if needed by other callers.

---

## Cross-Cutting Issues Found During Review

### Connection leaks in `review_db.py`

Every function in `review_db.py` does `conn = sqlite3.connect(path)` but several paths do not call `conn.close()`. For example, `get_due_cards` and `get_wrong_hashes` create connections but I could not confirm close calls in the truncated output. Use `with sqlite3.connect(path) as conn:` or explicit try/finally. This is a pre-existing bug that the plan should address.

### `_get_db()` swallows all exceptions

```python
def _get_db() -> Path:
    try:
        return get_db_path()
    except Exception:
        return Path.home() / ".config" / "studyctl" / "sessions.db"
```

This bare `except Exception` means a typo in `get_db_path()` (e.g., `AttributeError`) silently falls back to a hardcoded path. Narrow this to `(FileNotFoundError, KeyError)` or whatever `get_db_path()` actually raises on missing config.

### `ensure_tables` early return is a logic bug

```python
def ensure_tables(db_path: Path | None = None) -> None:
    path = db_path or _get_db()
    if not path.exists():
        return  # <-- Bug: should CREATE the db, not bail out
```

If the database file does not exist yet, `ensure_tables` returns without creating it. Then `record_card_review` calls `sqlite3.connect(path)` which creates an empty file with no tables. The `CREATE TABLE IF NOT EXISTS` statements never run. This is a latent bug that only works because `export_sessions` creates the DB first.

### `history.py` medication tracking

The plan does not mention this, but `history.py` contains medication/ADHD focus tracking (`dose_time`, `onset_minutes`, `peak_hours`). This is a completely separate concern from study concepts. If you are already extracting `concepts.py`, consider whether medication tracking should also be extracted. It is personal health data mixed with study analytics.

---

## Execution Order Assessment

The proposed order `A1 -> A2 -> A3 -> (migration) -> A4 -> B1-B4` is correct. A1 is a standalone bugfix. A2 is an extraction that does not change behavior. A3 (course picker) is a prerequisite for A4 (retry mode needs to know which course to filter wrong cards from). The migration must happen before A4 because A4 needs `session_id` and `is_retry` columns.

**One suggestion:** Do the `suppress(Exception)` narrowing and connection leak fixes as a preparatory cleanup commit before A1. These are low-risk, high-value fixes that reduce noise in later diffs.

---

## Summary

| Item | Verdict | Key Condition |
|------|---------|---------------|
| A1 ExportStats fix | APPROVE | Kill the dict branch, use per-source stats for display |
| A2 Concept dataclass | APPROVE | Use frozen=True, slots=True; relation_count defaults to 0 |
| A2 concepts.py extraction | STRONG APPROVE | Do not mix bridges and concept CRUD in the new module |
| A3 Course picker | APPROVE | (No concerns raised) |
| A4 State machine | CONDITIONAL | Must use StrEnum, not boolean flags; specify Textual binding mechanism |
| A4 session_id FK migration | APPROVE | Same migration for both columns; index the FK; idempotent |
| suppress narrowing | APPROVE | Use sqlite3.Error + logging, not bare suppress |
| Optional conn parameter | CONDITIONAL | Prefer single connection owned by TUI widget lifecycle over dual-path API |
