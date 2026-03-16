# YAGNI Review: Deepening Pass on TUI Polish Plan

**Reviewed:** 2026-03-15
**Plan:** `docs/plans/2026-03-15-feat-tui-polish-documentation-plan.md`
**Question:** Did the 8-agent deepening pass reintroduce complexity the simplification pass removed?

## Verdict: 3 items should be reverted or deferred, 3 are fine

---

### 1. A5 Composite Index -- DEFER (premature optimisation)

**Finding:** The performance oracle recommended adding a composite index on
`(card_hash, reviewed_at DESC)` to `ensure_tables()`. The oracle itself said
"not urgent" at <10K rows, then contradicted itself with "add this index."

**Evidence:** No `srs.db` or `studyctl_srs.db` file exists on disk yet. The
database has zero rows. The "100K rows (years of study)" scenario is literally
years away. Adding DDL to `ensure_tables()` for a table with zero rows is
textbook premature optimisation.

**Recommendation:** Remove the composite index from A5 acceptance criteria.
Add it to the Deferred Items table with rationale: "Add when row count exceeds
10K or query latency is measurable." The window function fix is the real
deliverable -- it is a correctness fix, not a performance fix.

**LOC saved:** ~3 lines (CREATE INDEX + acceptance criterion)

---

### 2. A5 get_course_stats() fix -- DEFER (scope creep)

**Finding:** The original scope was fixing `get_due_cards()`. The deepening
pass discovered `get_course_stats()` has a similar (but less broken -- the
correlated subquery IS deterministic) pattern and added it to A5.

**The difference matters:** `get_due_cards()` uses `HAVING reviewed_at =
MAX(reviewed_at)` with bare columns in GROUP BY -- genuinely non-deterministic.
`get_course_stats()` uses `WHERE reviewed_at = (SELECT MAX(reviewed_at)...)` --
deterministic, just slower. One is a correctness bug. The other is a style
preference.

**Recommendation:** Remove `get_course_stats()` from A5 scope. Add to Deferred
Items: "Refactor get_course_stats() to use window function for consistency --
not a correctness bug, just slower." Keep A5 focused on the actual bug.

**LOC saved:** ~15 lines (window function rewrite of get_course_stats + test)

---

### 3. A3 count_flashcards() function -- DROP (imaginary problem)

**Finding:** The performance oracle flagged `len(load_flashcards(fc_dir))` as
"200-500ms on cold filesystem with ~50 files." There are currently ZERO
flashcard JSON files on disk (confirmed via `find`). The "~50 files" number
was fabricated by the oracle.

**Even if files existed:** JSON deserialization of 50 small flashcard files is
not 200-500ms. That estimate assumes cold filesystem on spinning disk. This is
macOS with SSD and unified memory. The actual cost would be <20ms.

**Recommendation:** Remove the `count_flashcards()` suggestion entirely. Remove
the "len(load_flashcards()) slow on cold FS" risk from the Dependencies table.
If this ever becomes measurable, profile first, then optimise.

**LOC saved:** ~10 lines (function that would have been written for nothing)

---

### 4. A3 ModalScreen class -- KEEP (framework-idiomatic)

**Finding:** The previous simplicity review suggested an inline approach. The
deepening pass introduced a `CoursePickerScreen(ModalScreen)` class.

**Assessment:** The simplicity reviewer was wrong here. Textual's ModalScreen
pattern with typed return values (`ModalScreen[tuple[str, Path] | None]`) is
the framework-idiomatic way to handle this. An inline approach would fight
the framework. The class is ~10 lines and cleanly separates picker UI from
app logic. This earns its keep.

---

### 5. A6 Different exception sets per site -- KEEP (correct granularity)

**Finding:** DB sites catch `(sqlite3.Error, OSError)`, TTS catches
`(ImportError, OSError, RuntimeError)`. The question was whether maintaining
different sets is overengineered.

**Assessment:** No. The sets are different because the failure modes are
genuinely different. DB operations cannot raise `ImportError` (sqlite3 is
stdlib). TTS operations cannot raise `sqlite3.Error`. A unified superset
`(sqlite3.Error, ImportError, OSError, RuntimeError)` at every site would
catch exceptions that structurally cannot occur, defeating the purpose of
narrowing from `suppress(Exception)`. The differentiation is the whole point
of A6. Keep as-is.

---

### 6. Line count went UP (106 to 116) -- PARTIALLY JUSTIFIED

**Finding:** The estimate increased by 10 lines. Two sources:
- A5 scope expansion (get_course_stats): ~8 lines -- NOT justified (see item 2)
- A6 third call site (_speak): ~2 lines -- justified (real bug found)

**With recommendations applied:** Removing the index (3 lines) and
get_course_stats rewrite (15 lines) brings the estimate back to ~98 lines,
which is BELOW the original 106. The deepening pass found one legitimate
new call site (A6 _speak) and should have stopped there.

---

## Summary of Recommended Changes to Plan

| Item | Action | Reason |
|------|--------|--------|
| A5 composite index | Move to Deferred Items | Zero rows in DB, premature optimisation |
| A5 get_course_stats() | Move to Deferred Items | Deterministic query, not a correctness bug |
| A3 count_flashcards() | Remove entirely | Zero flashcard files exist, fabricated perf numbers |
| A3 "slow on cold FS" risk row | Remove from Dependencies table | Not a real risk |
| A3 ModalScreen class | Keep | Framework-idiomatic, clean separation |
| A6 different exception sets | Keep | Correct granularity, whole point of the fix |
| A6 third call site (_speak) | Keep | Legitimate bug found by deepening |

**Net effect:** Estimated LOC drops from 116 to ~98. Scope stays focused.
The deepening pass contributed one genuine finding (A6 _speak site) and
introduced three items that don't earn their keep.
