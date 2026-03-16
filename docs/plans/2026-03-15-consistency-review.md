# Consistency Review: TUI Polish & Documentation Plan

Reviewed against source code on `main` branch (commit `c27d7ac`).

---

## 1. Execution Order vs Dependency Graph -- INCONSISTENCY FOUND

**Stated order:** A5 -> A1 -> A2 -> A6 -> A3 -> A4

**Mermaid diagram shows:** A5 -> A3 (only explicit code dependency). A1, A2, A6, A3, A4 all independently feed into screenshots/docs. No arrows exist between A1/A2/A6 or between A6 and A3.

**Problem:** The diagram shows A5 -> A3 as the sole code dependency, but the text places A6 *between* A2 and A3 with no justification. A6 (narrowing suppress) has zero dependency on A1 or A2, and A3 has zero dependency on A6. The linear ordering A1 -> A2 -> A6 -> A3 -> A4 implies false sequencing. The diagram is actually correct that A1, A2, A6 are independent peers. The text should say: "A5 first, then {A1, A2, A6} in parallel, then {A3, A4} in parallel."

**Verdict:** Minor inconsistency. The linear order is a suggested implementation sequence, not a dependency chain, but this is not stated clearly. The diagram and text could confuse an implementer into thinking A6 blocks A3.

---

## 2. "A4 does NOT depend on A3" -- CONFIRMED CORRECT (with caveat)

The plan claims A4 can merge before A3. Source code confirms this:

- `_launch_study()` in `app.py` line 225: `name, path = courses[0]` hardcodes first course
- `StudyCardsTab.__init__()` receives `course_name` as a parameter and stores it as `self._course`
- A4's retry mode reuses `self._course` already set at session start -- no dependency on CoursePickerScreen

**Caveat the plan does NOT mention:** If A4 merges first and adds a retry `r` binding that calls `_launch_study()` again (to reload cards), it would hit the `courses[0]` hardcoding. But the plan says retry uses "in-memory filtering" from `self._result.wrong_hashes`, not a re-call to `_launch_study()`. This is safe. However, the plan should explicitly state that retry creates a new `StudyCardsTab` with filtered cards passed directly, never going through `_launch_study()` again. This assumption is implicit, not documented.

**Verdict:** Correct but fragile. Add a note clarifying the retry card-loading mechanism.

---

## 3. Deferred Items: concepts.py Extraction -- CONTRADICTION ACKNOWLEDGED

The plan's deferred items table says: "3 concept functions in 950-line history.py. Extract at 4+ (architecture reviewer wants now, simplicity reviewer says wait)."

This is a documented disagreement, not an unacknowledged contradiction. The plan sides with the simplicity reviewer. The rationale (extract at 4+ functions) is a reasonable threshold.

**Verdict:** No action needed. The contradiction is flagged clearly in the table.

---

## 4. wrong_hashes set vs list -- REAL BUG IN PLAN

**Current source code** (`review_loader.py` line in `ReviewResult`):
```python
wrong_hashes: list[str] = field(default_factory=list)
```

**Current usage** (`study_cards.py` `_record_answer()`):
```python
self._result.wrong_hashes.append(card.card_hash)
```

**Plan says:** Change to `set[str]` with `field(default_factory=set)`, change `append` to `add`.

**Problem:** The plan mentions this change in A4's research insights section but does NOT list it in A4's file targets or acceptance criteria. The `ReviewResult` dataclass lives in `review_loader.py`, but A4's file target is only `study_cards.py`. The change to `ReviewResult` is a prerequisite for A4's `O(1)` membership test during card filtering, but it is not tracked as a deliverable.

Additionally, `_show_summary()` calls `len(self._result.wrong_hashes)` which works for both list and set, so no breakage there. But `_record_answer()` uses `.append()` which will fail on a set. This `.append()` -> `.add()` change MUST be in A4's scope.

**Verdict:** Real gap. A4 must explicitly include `review_loader.py` in its file list, and the acceptance criteria should include the `wrong_hashes: set[str]` migration.

---

## 5. A5 Scope Creep -- REASONABLE, KEEP TOGETHER

Original scope: fix `get_due_cards()` GROUP BY bug.
Expanded scope: also fix `get_course_stats()` same bug + add composite index.

**Assessment:**
- `get_course_stats()` has the identical pattern (correlated subquery picking arbitrary rows). Fixing one without the other leaves a known bug. Both functions are in the same file (`review_db.py`), same pattern, same fix (window functions). Splitting them would be artificial.
- The composite index `(card_hash, reviewed_at DESC)` is a one-line addition to `ensure_tables()`. It directly supports the window function's `ORDER BY` clause introduced in the same commit.

**Verdict:** Reasonable scope. All three changes are cohesive within a single "fix review_db correctness" commit.

---

## 6. A6 Scope Creep -- REASONABLE, KEEP TOGETHER

Original: 2 call sites. Expanded: 3 call sites with different exception sets.

**Verified call sites in `study_cards.py`:**
1. Line ~218: `with contextlib.suppress(Exception):` around `record_card_review()` -- catch `(sqlite3.Error, OSError)`
2. Line ~263: `with contextlib.suppress(Exception):` around `record_session()` -- catch `(sqlite3.Error, OSError)`
3. Line ~280: `except Exception: pass` around TTS import/execution -- catch `(ImportError, OSError, RuntimeError)`

**Assessment:** Three call sites in the same file, each a 3-line change. The different exception sets per site are correct (DB ops vs TTS ops have different failure modes). This is still a small, focused fix. The differentiated exception types actually make it MORE correct, not more complex.

**Verdict:** Reasonable. Not scope creep -- it is a complete fix rather than a partial one.

---

## 7. Mermaid Diagram vs Text -- MINOR MISMATCH

**Diagram arrows:**
- A5 -> A3 (code dependency)
- A1 -> DOCS, A2 -> DOCS (independent, feed docs)
- A3 -> SS, A4 -> SS, A6 -> SS (feed screenshot)
- SS -> DOCS

**Text says:** "A5 -> A1 -> A2 -> A6 -> A3 -> A4"

**Mismatch:** The diagram shows no arrow from A5 to A1, no arrow from A2 to A6, and no arrow from A6 to A3. The diagram correctly shows {A1, A2} and {A3, A4, A6} as independent groups. The text imposes a false linear order.

**Verdict:** The diagram is correct. The text should be rewritten as: "A5 first (correctness bug), then {A1, A2, A6} in any order, then {A3, A4} in any order, then B1-B4."

---

## Summary

| Check | Status | Severity |
|-------|--------|----------|
| 1. Execution order vs diagram | Mismatch -- text implies false sequencing | Low |
| 2. A4 independent of A3 | Correct but implicit assumption undocumented | Low |
| 3. Deferred concepts.py | Contradiction acknowledged in plan | None |
| 4. wrong_hashes set vs list | Missing file target and acceptance criteria | Medium |
| 5. A5 scope | Reasonable, cohesive | None |
| 6. A6 scope | Reasonable, complete fix | None |
| 7. Diagram vs text | Diagram correct, text overly linear | Low |

**One actionable fix required:** A4 must add `review_loader.py` to its file list and include `.append()` -> `.add()` + `list[str]` -> `set[str]` in acceptance criteria.

**One recommended clarification:** Restate execution order as partial order (A5 first, then parallel groups) to match the diagram.
