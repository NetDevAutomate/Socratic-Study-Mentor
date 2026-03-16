# Implementation Patterns Research

**Date:** 2026-03-15
**Context:** studyctl TUI Phase 9 polish — SM-2 retry semantics, SQLite query correctness, error handling, progress bars

---

## 1. SM-2 Retry Semantics: Should Retry Rounds Record to SM-2?

### The Original SM-2 Specification (Wozniak 1990)

Step 7 of the SM-2 algorithm states:

> "After each repetition session of a given day repeat again all items that scored below four in the quality assessment."

This is an **in-session drill loop** — a same-day re-presentation of failed items. The key question: does the retry response update the E-Factor and scheduling?

**Answer from the original algorithm: No, only the first presentation counts for scheduling.**

Steps 3-6 govern interval scheduling and E-Factor updates. Step 7 is a separate mechanism — it ensures the learner leaves the session having seen the correct answer for every card, but it does not feed back into the scheduling formula. The E-Factor was already modified in Step 5, and the interval was already reset in Step 6 (for grades < 3).

### How Anki Handles It

Anki separates cards into **learning/relearning** and **review** queues:

- When you press **Again** on a review card, it enters the **relearning queue** with short-term steps (e.g., 10 minutes).
- The card's **ease factor is reduced once** at the moment of lapse (by 20% by default via "New Interval" setting).
- Subsequent Again presses during relearning **do not further reduce ease** — this is Anki's explicit fix for "ease hell" in SM-2.
- The card must pass through all relearning steps before graduating back to review.

**Source:** Anki FAQ: "Successive failures while cards are in learning do not result in further decreases to the card's ease."

### How FSRS Handles It

FSRS v6 has an explicit **same-day review formula**:

```
S'(S,G) = S * e^(w17 * (G - 3 + w18)) * S^(-w19)
```

Same-day reviews DO update stability, but with a **separate, attenuated formula** that converges (cannot grow unboundedly). This acknowledges that same-day repetition has diminishing returns for long-term memory.

**Source:** FSRS-6 wiki, open-spaced-repetition/fsrs4anki

### Recommendation for studyctl

**Do NOT record retry-round answers to SM-2 scheduling.** Rationale:

1. **Original SM-2** treats Step 7 as a drill, not a scheduling event.
2. **Anki** explicitly prevents relearning failures from compounding ease penalties.
3. **FSRS** uses a separate attenuated formula for same-day reviews — acknowledging they are fundamentally different from spaced reviews.
4. **Practical:** Recording retries as full reviews would artificially inflate review counts and potentially create "ease hell" for difficult cards.

**Implementation pattern:**

```python
def _record_answer(self, correct: bool, *, is_retry: bool = False) -> None:
    """Record answer. Retry-round answers update session stats but NOT SM-2."""
    card = self._cards[self.current_index]
    elapsed_ms = int((time.monotonic() - self._card_start_time) * 1000)

    if correct:
        self._result.correct += 1
    else:
        self._result.incorrect += 1
        if not is_retry:
            self._result.wrong_hashes.append(card.card_hash)

    # Only record to SM-2 on first presentation — retries are drill-only
    if not is_retry:
        card_type = "flashcard" if isinstance(card, Flashcard) else "quiz"
        with _safe_db_write():
            record_card_review(
                course=self._course,
                card_type=card_type,
                card_hash=card.card_hash,
                correct=correct,
                response_time_ms=elapsed_ms,
            )
```

The `ReviewResult` tracks `wrong_hashes` from the **first pass only**. The retry round lets the user drill those cards but does not call `record_card_review()`.

---

## 2. SQLite ROW_NUMBER() Window Function for Most-Recent Review

### The Bug

The current `get_due_cards()` uses this pattern:

```sql
GROUP BY card_hash
HAVING reviewed_at = MAX(reviewed_at)
```

**This is non-deterministic in SQLite.** When using `GROUP BY`, SQLite picks values for non-aggregated columns from an **arbitrary row** in the group (per SQLite docs on "bare columns"). The `HAVING` clause filters *after* grouping, but `reviewed_at` in the `HAVING` refers to the arbitrary row's value, not the row with the actual MAX.

In practice this works *sometimes* because SQLite's query planner often picks the last-inserted row, but it is **not guaranteed** and will break silently with index changes or query plan changes.

### The Correct Pattern: ROW_NUMBER()

**Source:** SQLite official docs, "Built-in Window Functions":

> `row_number()` — The number of the row within the current partition. Rows are numbered starting from 1 in the order defined by the ORDER BY clause in the window definition.

```sql
SELECT card_hash, correct, ease_factor, interval_days, next_review, review_count
FROM (
    SELECT
        cr.card_hash,
        cr.correct,
        cr.ease_factor,
        cr.interval_days,
        cr.next_review,
        COUNT(*) OVER (PARTITION BY cr.card_hash) AS review_count,
        ROW_NUMBER() OVER (
            PARTITION BY cr.card_hash
            ORDER BY cr.reviewed_at DESC
        ) AS rn
    FROM card_reviews cr
    WHERE cr.course = ?
)
WHERE rn = 1
  AND next_review <= ?
ORDER BY next_review ASC
```

### Why This Is Correct

1. `ROW_NUMBER() OVER (PARTITION BY card_hash ORDER BY reviewed_at DESC)` assigns `rn=1` to the **most recent review** for each card — deterministically.
2. The outer `WHERE rn = 1` picks exactly one row per card.
3. `COUNT(*) OVER (PARTITION BY cr.card_hash)` gives total review count without a separate subquery.
4. All column values come from the **actual most-recent row**, not an arbitrary group member.

### Performance on Small Datasets (<10K rows)

Window functions in SQLite use an **in-memory sort** per partition. For <10K rows:

- The sort is O(n log n) on the full result set, which for 10K rows completes in <1ms.
- The existing index `idx_card_reviews_hash` on `(card_hash)` helps the WHERE filter.
- Adding a composite index `(course, card_hash, reviewed_at DESC)` would make this a covering index but is unnecessary at <10K rows.
- **Bottom line:** No measurable performance difference from the GROUP BY approach at this scale. Correctness is the reason to switch, not performance.

### The `get_course_stats` Correlated Subquery (Also Affected)

The existing mastered count query has the same bug:

```sql
-- BUGGY: same bare-column issue
WHERE reviewed_at = (SELECT MAX(reviewed_at) FROM card_reviews cr2 WHERE cr2.card_hash = cr1.card_hash)
```

Replace with the same ROW_NUMBER() CTE pattern:

```sql
WITH latest AS (
    SELECT card_hash, interval_days,
           ROW_NUMBER() OVER (PARTITION BY card_hash ORDER BY reviewed_at DESC) AS rn
    FROM card_reviews
    WHERE course = ?
)
SELECT COUNT(DISTINCT card_hash) FROM latest
WHERE rn = 1 AND interval_days > 30
```

---

## 3. contextlib.suppress Narrowing: Never Crash TUI, But Log Diagnostics

### Current Pattern (Problematic)

```python
with contextlib.suppress(Exception):
    record_card_review(...)
```

This swallows **everything** — including `TypeError` from bad arguments, `PermissionError` from locked DB, `sqlite3.DatabaseError` from corruption. You get zero diagnostic signal.

### Best Practice: Narrow + Log

**Source:** Python contextlib docs — `suppress(*exceptions)` accepts specific exception classes.

The pattern for "never crash the TUI but log diagnostics":

```python
import logging
from contextlib import suppress

logger = logging.getLogger(__name__)

def _safe_db_write() -> contextlib.suppress:
    """Suppress DB write failures in TUI — study flow must not break.

    Narrows to specific exceptions that can legitimately occur during
    normal operation (locked DB, missing file, disk full).
    """
    return suppress(OSError, sqlite3.Error)
```

Then wrap each call site with a **try/except that logs before suppressing**:

```python
def _record_answer(self, correct: bool) -> None:
    # ... stats tracking ...

    card_type = "flashcard" if isinstance(card, Flashcard) else "quiz"
    try:
        record_card_review(
            course=self._course,
            card_type=card_type,
            card_hash=card.card_hash,
            correct=correct,
            response_time_ms=elapsed_ms,
        )
    except (OSError, sqlite3.Error) as exc:
        logger.warning("Failed to record review for %s: %s", card.card_hash, exc)
    # TypeError, ValueError, etc. will still propagate — those are bugs, not runtime failures
```

### Exception Type Narrowing Guide

| Exception | Cause | Should Suppress? |
|-----------|-------|-----------------|
| `sqlite3.OperationalError` | DB locked, disk full, table missing | Yes — runtime |
| `sqlite3.DatabaseError` | Corruption, malformed DB | Yes — runtime |
| `sqlite3.IntegrityError` | Constraint violation (duplicate) | Yes — runtime |
| `OSError` | File not found, permission denied | Yes — runtime |
| `TypeError` | Wrong argument types (code bug) | **No** — bug |
| `ValueError` | Invalid data (code bug) | **No** — bug |
| `AttributeError` | Missing attribute (code bug) | **No** — bug |

**Rule of thumb:** Suppress exceptions that come from the **environment** (disk, DB, permissions). Let exceptions from **code bugs** propagate so they surface during development.

### For the Voice/TTS Path

The `except Exception: pass` in `_speak()` is similarly broad but has a different justification — TTS involves optional external dependencies (kokoro). The narrowing here:

```python
def _speak(self, text: str) -> None:
    try:
        cfg = self._voice_config()
        # ... kokoro setup ...
    except (ImportError, OSError, RuntimeError) as exc:
        logger.debug("Voice unavailable: %s", exc)
```

`ImportError` covers missing kokoro. `OSError` covers audio device issues. `RuntimeError` covers kokoro internal failures.

---

## 4. Rich Progress Bar: Per-Source Tracking Pattern

### The Problem

You want to show **per-iteration stats** (e.g., "source X: 3 correct, 1 failed") alongside **cumulative stats** (e.g., "Total: 15/20 exported") in a Rich progress bar.

### Pattern: task.fields for Per-Iteration Metadata

**Source:** Rich docs, "Columns" section:

> "Additional fields passed via keyword arguments to `update()` are stored in `task.fields`. You can add them to a format string with the following syntax: `{task.fields[extra]}`"

```python
from rich.progress import (
    BarColumn,
    MofNCompleteColumn,
    Progress,
    SpinnerColumn,
    TextColumn,
    TimeElapsedColumn,
)


def export_with_progress(sources: list[Source]) -> ExportStats:
    """Export sources with per-source and cumulative progress tracking."""
    stats = ExportStats()

    progress = Progress(
        SpinnerColumn(),
        TextColumn("[progress.description]{task.description}"),
        BarColumn(),
        MofNCompleteColumn(),
        TextColumn("[dim]{task.fields[status]}[/dim]"),
        TimeElapsedColumn(),
    )

    with progress:
        task = progress.add_task(
            "Exporting",
            total=len(sources),
            status="starting...",
        )

        for source in sources:
            # Update description to show current source
            progress.update(
                task,
                description=f"[cyan]{source.name}[/cyan]",
                status=f"ok:{stats.success} err:{stats.errors}",
            )

            try:
                export_one(source)
                stats.success += 1
            except ExportError as exc:
                stats.errors += 1
                logger.warning("Failed %s: %s", source.name, exc)

            progress.update(
                task,
                advance=1,
                status=f"ok:{stats.success} err:{stats.errors}",
            )

    return stats
```

### Pattern: Multiple Tasks for Parallel-Style Display

If you want each source as its own row (useful when sources have sub-steps):

```python
with Progress(
    SpinnerColumn(),
    TextColumn("[progress.description]{task.description}"),
    BarColumn(),
    MofNCompleteColumn(),
    TextColumn("{task.fields[detail]}"),
) as progress:
    # Overall task
    overall = progress.add_task("Overall", total=len(sources), detail="")

    for source in sources:
        # Per-source task (auto-removes when done via visible=False)
        sub = progress.add_task(
            f"  {source.name}",
            total=source.step_count,
            detail="loading...",
        )

        for step in source.steps:
            process(step)
            progress.update(sub, advance=1, detail=step.name)

        progress.update(sub, visible=False)  # Hide completed sub-task
        progress.update(overall, advance=1, detail=f"Last: {source.name}")
```

### Key API Points

- `progress.update(task_id, advance=N)` — increment completed by N
- `progress.update(task_id, completed=N)` — set completed to exact value
- `progress.update(task_id, description="...", **fields)` — update any field
- `task.fields[key]` — accessible in format strings via `{task.fields[key]}`
- `progress.add_task(..., visible=False)` — create hidden task (show later)
- `Progress.get_default_columns()` — returns defaults so you can extend, not replace

---

## Summary: Application to studyctl

| Pattern | Current Code | Fix |
|---------|-------------|-----|
| SM-2 retry | No retry loop exists yet | Add `is_retry` flag; retries skip `record_card_review()` |
| ROW_NUMBER | `GROUP BY ... HAVING reviewed_at = MAX(reviewed_at)` | CTE with `ROW_NUMBER() OVER (PARTITION BY card_hash ORDER BY reviewed_at DESC)` |
| suppress narrowing | `suppress(Exception)` (2 sites) + bare `except Exception: pass` (1 site) | `except (OSError, sqlite3.Error)` + `logger.warning()` |
| Progress bar | N/A (future export polish) | `task.fields[status]` for per-iteration stats in format string |

### Sources

- **Wozniak 1990** — SM-2 Algorithm, Steps 1-7: https://super-memory.com/english/ol/sm2.htm
- **Anki FAQ** — SM-2 modifications and ease hell prevention: https://faqs.ankiweb.net/what-spaced-repetition-algorithm.html
- **Anki Manual** — Deck Options, Lapses, Relearning Steps: https://docs.ankiweb.net/deck-options.html
- **FSRS-6 Wiki** — Same-day review formula: https://github.com/open-spaced-repetition/fsrs4anki/wiki/The-Algorithm
- **SQLite Docs** — Window Functions, ROW_NUMBER(): https://www.sqlite.org/windowfunctions.html
- **Python Docs** — contextlib.suppress: https://docs.python.org/3/library/contextlib.html
- **Rich Docs** — Progress Display, Columns, task.fields: https://rich.readthedocs.io/en/stable/progress.html
