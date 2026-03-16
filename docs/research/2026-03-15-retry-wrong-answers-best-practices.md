# Research: Retry Wrong Answers / Review Failed Cards in SRS Systems

**Date**: 2026-03-15
**Scope**: How established SRS apps handle retrying failed cards, UX patterns, database design, and Textual TUI implementation patterns.
**Context**: studyctl already has SM-2 scheduling, `card_reviews` + `review_sessions` tables, `get_wrong_hashes()`, and a Textual TUI with correct/incorrect/skip bindings.

---

## 1. How Popular SRS Apps Handle Failed Cards

### 1.1 SM-2 Algorithm (SuperMemo, Mnemosyne)

**Source**: Wozniak 1990, supermemo.com/english/ol/sm2.htm

SM-2 defines two distinct mechanisms for failed cards:

**Step 6 — Inter-session reset**: If quality response < 3 (i.e., incorrect), start repetitions from the beginning WITHOUT changing the E-Factor. The card's interval resets to I(1)=1, I(2)=6, etc., as if memorised anew. The E-Factor (difficulty) is still updated by the formula, but the interval ladder restarts.

**Step 7 — Intra-session retry (CRITICAL)**: "After each repetition session of a given day, repeat again all items that scored below four in the quality assessment." This is the canonical "retry wrong answers" behaviour:
- Retries happen at the END of the current day's session (not inline)
- The threshold is quality < 4 (not just failed — also "serious difficulty" responses)
- Items are repeated until they score >= 4
- This is explicitly a same-day, same-session loop

**Your current implementation**: You implement Step 6 (interval reset to 1, ease -= 0.2) but NOT Step 7 (end-of-session retry). This is the gap.

### 1.2 Anki (SM-2 variant / FSRS)

**Source**: docs.ankiweb.net/studying.html, docs.ankiweb.net/deck-options.html

Anki uses **inline re-presentation**, not end-of-session retry:

- Pressing "Again" moves the card into a **relearning queue** with configurable steps (default: 10 minutes)
- The card is re-shown within the same study session after the step delay
- Relearning steps work identically to initial learning steps
- Cards cycle through relearning steps until they "graduate" back to review status
- After graduating, the card gets a new interval (default: 1 day minimum, configurable "New Interval" as % of previous)

**Lapse tracking**: Each "Again" press increments a lapse counter. After a threshold (default: 8 lapses), the card becomes a "leech" — either suspended or tagged for reformulation.

**FSRS approach** (modern Anki): FSRS calculates a new post-lapse stability S'_f using a dedicated formula. FSRS recommends AGAINST multiple same-day repetitions: "Evidence shows that repeating a card multiple times in a single day does not significantly contribute to long-term memory." FSRS can control short-term scheduling directly, often giving "Again" an interval of 1+ days.

**Same-day reviews in FSRS-6**: Have a dedicated stability formula: `S'(S,G) = S * e^(w17 * (G - 3 + w18)) * S^(-w19)`. Stability increases faster when small, converges when large. This means same-day retries have diminishing returns mathematically.

### 1.3 Mnemosyne

**Source**: mnemosyne-proj.org/principles.php

Uses a modified SM-2 algorithm. Same fundamental approach: grade 0-2 resets the interval, with same-day retry of failed items. Adds randomness to intervals to prevent clustering.

### 1.4 Consensus Summary

| Aspect | SM-2 (Original) | Anki (Legacy) | Anki (FSRS) | Recommendation for studyctl |
|--------|-----------------|---------------|-------------|----------------------------|
| When retry happens | End of session | Inline (after delay) | Discouraged for same-day | **End of session** (SM-2 Step 7) |
| Retry threshold | Score < 4 | "Again" button | N/A | **All incorrect cards** |
| Effect on scheduling | None (retry is separate) | Relearning steps | Separate S'_f formula | **Don't double-penalise** |
| Retry until | Score >= 4 | Graduates from steps | N/A | **Single retry pass** |
| Long-term effect | E-Factor already adjusted | Ease already reduced | Stability already updated | **Already handled in record_card_review** |

---

## 2. UX Patterns for Retry Flows

### 2.1 When to Offer Retry

**Proven pattern (SM-2 Step 7)**: After ALL cards in the session are complete, present a summary, then offer to retry wrong answers.

**Why end-of-session, not inline**:
- Inline delays (Anki-style) require a timer/queue system — over-engineered for a CLI tool
- End-of-session retry provides spacing (you've seen other cards in between) which aids encoding
- Cleaner UX: complete flow, then decision point
- Better for ADHD: clear phases, not context-switching between "new card" and "retry card" mid-flow

**When NOT to retry**:
- If all cards were correct (obviously)
- If the user is low-energy (offer to save for next session instead)
- If there are too many wrong (>50%): the material needs re-study, not just retry

### 2.2 How to Present Retry

**Summary screen pattern** (used by Anki, Quizlet, Brainscape):

```
Session Complete!

  Score: 7/10 (70%)  |  Duration: 4m 32s
  Grade: Good

  3 cards answered incorrectly.

  [r] Retry wrong answers  [q] Quit  [d] Dashboard
```

**During retry**:
- Show "Retry 1/3" not "Card 8/10" — make it clear this is a retry pass
- Show the card normally but add a subtle indicator: `[retry]` badge or different border colour
- After retry pass, show a combined summary:
  ```
  Retry Complete!
    Original: 7/10 (70%)
    Retry:    2/3 corrected
    Final:    9/10 (90%)
  ```

### 2.3 Shuffle vs Original Order

**Shuffle the retry cards**. Rationale:
- Prevents pattern-based memorisation ("the third wrong one was about X")
- SM-2 Step 7 doesn't specify order — it just says "repeat again"
- Anki's relearning queue interleaves with other cards (effective shuffling)
- Your existing code already shuffles via `shuffle_items()` in review_loader.py

### 2.4 Should Retry Success Count Differently?

**Yes, but subtly**. Three approaches from the literature:

1. **SM-2 approach**: Retry reviews don't affect the E-Factor at all — the damage was already done in the first pass. Retry is purely for same-day reinforcement.

2. **Anki approach**: Relearning steps don't change ease factor. Only the initial "Again" press penalises. Graduating from relearning just sets the new (reduced) interval.

3. **Recommended for studyctl**: Do NOT call `record_card_review()` during retry. The first-pass incorrect already:
   - Reset interval to 1 day
   - Reduced ease by 0.2
   - Set next_review to tomorrow

   Recording a retry "correct" would overwrite these values and give false confidence. Instead, track retry results separately for display/analytics only.

---

## 3. Database Design for Tracking Retries

### 3.1 Option A: Flag on Existing Table (Recommended)

Add an `is_retry` column to `card_reviews`:

```sql
ALTER TABLE card_reviews ADD COLUMN is_retry BOOLEAN DEFAULT 0;
ALTER TABLE card_reviews ADD COLUMN parent_session_id INTEGER REFERENCES review_sessions(id);
```

**Pros**: Simple, no new tables, easy to query "all reviews including retries"
**Cons**: Need to filter out retries when calculating SM-2 scheduling

**Query: Cards I got wrong across all sessions (excluding retries)**:
```sql
SELECT card_hash, COUNT(*) as times_wrong,
       GROUP_CONCAT(DISTINCT course) as courses
FROM card_reviews
WHERE correct = 0 AND is_retry = 0
GROUP BY card_hash
ORDER BY times_wrong DESC;
```

**Query: Retry improvement rate**:
```sql
SELECT
    cr_orig.card_hash,
    cr_retry.correct as corrected_on_retry
FROM card_reviews cr_orig
JOIN card_reviews cr_retry
    ON cr_orig.card_hash = cr_retry.card_hash
    AND cr_retry.is_retry = 1
    AND cr_retry.parent_session_id = cr_orig.parent_session_id
WHERE cr_orig.correct = 0 AND cr_orig.is_retry = 0;
```

### 3.2 Option B: Separate Retry Session Row

Add `session_type` and `parent_session_id` to `review_sessions`:

```sql
ALTER TABLE review_sessions ADD COLUMN session_type TEXT DEFAULT 'primary';
  -- Values: 'primary', 'retry'
ALTER TABLE review_sessions ADD COLUMN parent_session_id INTEGER REFERENCES review_sessions(id);
```

**Pros**: Clean session-level analytics, easy to distinguish "real" sessions from retries
**Cons**: Slightly more complex to link individual card retries back

### 3.3 Recommended Hybrid Approach

Use BOTH flags — one on sessions, one on card reviews:

```sql
-- Migration: add retry tracking
ALTER TABLE review_sessions ADD COLUMN session_type TEXT DEFAULT 'primary';
ALTER TABLE review_sessions ADD COLUMN parent_session_id INTEGER
    REFERENCES review_sessions(id);

ALTER TABLE card_reviews ADD COLUMN is_retry BOOLEAN DEFAULT 0;
ALTER TABLE card_reviews ADD COLUMN session_id INTEGER
    REFERENCES review_sessions(id);
```

The `session_id` foreign key on `card_reviews` is independently valuable — your current code links cards to sessions by timestamp correlation, which is fragile. An explicit FK makes queries reliable.

### 3.4 Leech Detection (from Anki)

Track cards that are repeatedly wrong across sessions:

```sql
-- Cards wrong 3+ times across different primary sessions = leeches
SELECT card_hash, COUNT(DISTINCT session_id) as sessions_wrong
FROM card_reviews
WHERE correct = 0 AND is_retry = 0
GROUP BY card_hash
HAVING sessions_wrong >= 3
ORDER BY sessions_wrong DESC;
```

This maps to Anki's leech concept. These cards likely need reformulation rather than more repetition — surface them to the user.

---

## 4. Textual TUI Patterns for Post-Session Actions

### 4.1 Current State in studyctl

`_show_summary()` in `study_cards.py` currently:
1. Calculates duration and score percentage
2. Displays summary text with grade (Excellent/Good/Needs review)
3. Shows wrong count
4. Hides score buttons (correct/incorrect/skip)
5. Updates progress label to "Press q to return"
6. Records session to DB

**Missing**: No retry option, no post-session action bindings.

### 4.2 Recommended Pattern: Dynamic Binding Swap

Textual supports per-screen and per-widget `BINDINGS`. The cleanest pattern for post-session actions:

**Approach**: In `_show_summary()`, swap the active bindings by toggling widget visibility or using `action_*` methods that check state.

Key design decisions:
- Add a `Binding("r", "retry_wrong", "Retry Wrong")` that is always defined but only active post-session
- Use Textual's `action_retry_wrong` to filter `_cards` to `wrong_hashes`, reset `current_index`, and restart the card flow
- The retry pass should use the SAME `StudyCardsTab` widget instance (not a new one) to maintain session continuity

### 4.3 Footer Key Binding Updates

During the session:
```
Space: Flip  |  y: Correct  |  n: Incorrect  |  s: Skip  |  h: Hint  |  v: Voice
```

After session complete (with wrong answers):
```
r: Retry Wrong (3)  |  q: Return  |  d: Dashboard
```

After session complete (all correct):
```
q: Return  |  d: Dashboard
```

After retry complete:
```
q: Return  |  d: Dashboard
```

### 4.4 Screen Transition Pattern

Two valid approaches from Textual docs:

**A. In-place state machine** (recommended for studyctl):
- `StudyCardsTab` tracks state: `reviewing` | `summary` | `retrying` | `retry_summary`
- `_show_summary()` sets state to `summary` and shows retry option
- `action_retry_wrong()` filters cards, sets state to `retrying`, restarts flow
- Bindings check state before acting (e.g., `action_mark_correct` is no-op in `summary` state)
- Avoids screen push/pop complexity

**B. Modal screen** (Textual's `push_screen` with callback):
- Push a `SessionSummaryScreen` that returns a result via `dismiss()`
- Result can be `"retry"`, `"quit"`, or `"dashboard"`
- Callback handles the transition
- Cleaner separation but more boilerplate for a widget-in-tab scenario

Approach A fits better because `StudyCardsTab` is a Widget mounted inside a TabPane, not a Screen. Pushing a modal screen from within a tab widget works but feels awkward. The state machine approach keeps everything self-contained.

### 4.5 Retry Flow State Machine

```
                    +-----------+
                    | reviewing |
                    +-----+-----+
                          |
                    last card done
                          |
                    +-----v-----+
              +---->| summary   |
              |     +-----+-----+
              |           |
              |     user presses 'r'
              |           |
              |     +-----v------+
              |     | retrying   |
              |     +-----+------+
              |           |
              |     last retry card done
              |           |
              |     +-----v---------+
              |     | retry_summary |
              |     +-----+---------+
              |           |
              |     user presses 'r' (optional: allow multiple retry passes)
              |           |
              +-----------+
```

---

## 5. Specific Recommendations for studyctl

### 5.1 What to Build

1. **End-of-session retry prompt**: After `_show_summary()`, show "[r] Retry wrong (N)" if wrong_hashes is non-empty
2. **Single retry pass** (not infinite loop): One retry attempt, then show final combined score
3. **No SM-2 recording for retries**: Track retry results for display only. The original incorrect already penalised the card's schedule.
4. **Shuffled retry order**: Use existing `shuffle_items()` on the filtered wrong cards
5. **Leech surfacing**: After N sessions where the same card is wrong, notify the user that the card may need reformulation

### 5.2 What NOT to Build

- **Inline re-presentation with delays** (Anki-style): Over-engineered for a TUI tool, requires timer infrastructure
- **Multiple retry passes in one session**: Diminishing returns per FSRS research. One pass is sufficient.
- **Retry affecting scheduling**: Would create "ease inflation" — a retry correct is not the same as a first-attempt correct
- **Separate retry screens**: Keep it in the same widget, just filter the card list

### 5.3 Database Migration Plan

Priority order:
1. Add `session_id` FK to `card_reviews` (independently valuable, fixes timestamp-correlation fragility)
2. Add `session_type` + `parent_session_id` to `review_sessions`
3. Add `is_retry` to `card_reviews`
4. Add leech detection query (can be a `studyctl leeches` CLI command)

### 5.4 Migration Compatibility

Your existing migration infrastructure in `agent-session-tools` uses `migrations.py` with `migrate()`. The retry columns should follow the same pattern:
- `ALTER TABLE` with `DEFAULT` values so existing data is unaffected
- New columns are nullable or have sensible defaults
- No data migration needed — all existing reviews are `is_retry=0`, `session_type='primary'`

---

## Sources

| Source | Authority | Key Contribution |
|--------|-----------|-----------------|
| Wozniak SM-2 (1990) | Primary algorithm specification | Step 6 (reset) + Step 7 (same-day retry) |
| Anki Manual - Studying | Official documentation | Relearning queue, inline retry with steps |
| Anki Manual - Deck Options (Lapses) | Official documentation | Relearning steps, minimum interval, leeches |
| FSRS Algorithm Wiki | Algorithm specification | Same-day review formula, evidence against excessive same-day reps |
| Mnemosyne Principles | Official documentation | SM-2 variant, same fundamental retry approach |
| Textual Screens Guide | Official documentation | push_screen/dismiss pattern, dynamic bindings |
| studyctl source code | Project codebase | Current state: review_db.py, study_cards.py, review_loader.py |
