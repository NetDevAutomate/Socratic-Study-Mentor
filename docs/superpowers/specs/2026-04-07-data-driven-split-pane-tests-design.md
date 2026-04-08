# Data-Driven Split-Pane Test Infrastructure

## Problem

The split-pane layout (dashboard + terminal) in the web UI has visual bugs that only manifest in real browsers, not headless Playwright. The existing test file (`test_split_pane_layout.py`) has two critical flaws:

1. **Destroys real user state** — `_clean_ipc` deletes `~/.config/studyctl/session-state.json`
2. **Not machine-readable** — an agent can't consume pytest output to drive a fix/retest loop

## Design: Autoagent-Inspired Measurement + Verification

Adapted from the autoagent framework at `/Users/ataylor/code/tools/autoagent`:
- **Score = ground truth** — every requirement produces a binary 1.0/0.0 score
- **Measurement before assertion** — DOM metrics always captured, verifiers evaluate them
- **Machine-readable results** — `results.json` contract for the agent loop
- **Full isolation** — tests never touch real user state

### Architecture

```
session_state.py (STUDYCTL_SESSION_DIR env var)
        |
test_split_pane_layout.py
   |          |           |
_measure_all()  verify_R01..R16()  results.json hook
   |          |           |
  JS eval   pure functions   pytest session finish
```

### Component 1: Test Isolation (`session_state.py` change)

Add env var override to the module-level constants:

```python
SESSION_DIR = Path(os.environ.get("STUDYCTL_SESSION_DIR", Path.home() / ".config" / "studyctl"))
```

All derived paths (`STATE_FILE`, `TOPICS_FILE`, `PARKING_FILE`, `_LOCK_FILE`) use `SESSION_DIR` as base. The web server subprocess inherits the env var, so both test code and server use the temp dir.

### Component 2: Measurement Layer

A single `_measure_all(page) -> dict` function that executes one JavaScript `page.evaluate()` call and returns ALL DOM metrics needed for R1-R16:

```python
{
    # Container
    "container_height": 740,
    "container_top": 60,
    "container_bottom": 800,

    # Dashboard panel
    "dashboard_height": 296,
    "dashboard_top": 60,
    "dashboard_bottom": 356,

    # Gutter (Split.js drag handle)
    "gutter_height": 10,
    "gutter_top": 356,
    "gutter_bottom": 366,

    # Terminal panel
    "terminal_height": 434,
    "terminal_top": 366,
    "terminal_bottom": 800,
    "terminal_display": "block",

    # iframe state
    "iframe_src": "/terminal/",
    "iframe_height": 400,
    "iframe_visible": True,

    # Visibility flags
    "split_container_visible": True,
    "start_picker_visible": False,
    "terminal_unavailable_visible": False,

    # Flex direction (for swap detection)
    "flex_direction": "column",

    # Error (if no visible split container found)
    "error": None,
}
```

Captured before AND after every action (drag, swap, lifecycle change). The delta between before/after measurements is how the agent diagnoses what went wrong.

### Component 3: Verifier Functions

16 pure functions, one per requirement. Signature:

```python
def verify_R01(m: dict) -> tuple[bool, str]:
    """R1: Two panes fill 100% of vertical space."""
    if m.get("error"):
        return False, f"No visible container: {m['error']}"
    ok = (m["container_height"] > 0
          and m["dashboard_height"] > 0
          and m["terminal_height"] > 0)
    detail = (f"container={m['container_height']}, "
              f"dashboard={m['dashboard_height']}, "
              f"terminal={m['terminal_height']}")
    return ok, detail
```

Returns `(passed, diagnostic_detail)`. The detail string always includes the raw measurements, regardless of pass/fail.

**Requirements mapping:**

| Req | Verifier | Key assertion |
|-----|----------|---------------|
| R01 | `verify_R01` | container, dashboard, terminal all > 0 |
| R02 | `verify_R02` | dashboard_top == container_top (within 2px) |
| R03 | `verify_R03` | terminal_bottom == container_bottom (within 2px) |
| R04 | `verify_R04` | gutter_height > 0, gutter_top == dashboard_bottom, gutter_bottom == terminal_top |
| R05 | `verify_R05` | abs(dashboard + gutter + terminal - container) < 2 |
| R06 | `verify_R06` | after drag down: dashboard_after > dashboard_before + delta*0.8 |
| R07 | `verify_R07` | after drag down: terminal_after < terminal_before - delta*0.8 |
| R08 | `verify_R08` | after drag: R05 still holds |
| R09 | `verify_R09` | after swap: terminal_top < dashboard_top |
| R10 | `verify_R10` | after swap + drag: R06-R08 still hold |
| R11 | `verify_R11` | iframe_src contains '/terminal/' AND iframe_height > 0 |
| R12 | `verify_R12` | tmux capture-pane contains expected marker |
| R13 | `verify_R13` | terminal_unavailable_visible when ttyd dead |
| R14 | `verify_R14` | split_container_visible when mode != 'ended' |
| R15 | `verify_R15` | start_picker_visible when mode == 'ended' |
| R16 | `verify_R16` | start_picker_visible when no state file |

### Component 4: Results JSON

Written by a pytest session-finish hook. Structure:

```json
{
    "timestamp": "2026-04-07T22:00:00Z",
    "total": 16,
    "passed": 14,
    "failed": 2,
    "scores": {
        "R01": 1.0, "R02": 1.0, "R03": 0.0, "R04": 1.0,
        "R05": 0.0, "R06": 1.0, "R07": 1.0, "R08": 1.0,
        "R09": 1.0, "R10": 1.0, "R11": 1.0, "R12": 1.0,
        "R13": 1.0, "R14": 1.0, "R15": 1.0, "R16": 1.0
    },
    "failures": {
        "R03": "terminal_bottom (795) != container_bottom (800), gap=5px",
        "R05": "sum=738, container=740, diff=2px"
    },
    "measurements": {
        "R03": {"terminal_bottom": 795, "container_bottom": 800},
        "R05": {"dashboard": 296, "gutter": 10, "terminal": 432, "sum": 738, "container": 740}
    }
}
```

Path: `packages/studyctl/tests/results/split_pane_results.json`

### Component 5: Test Structure

```python
class TestLayoutRequirements:
    """R01-R05: Static layout with terminal visible."""

    def test_R01_panels_fill_vertical_space(self, web_page):
        _setup_active_session(web_page, with_terminal=True)
        m = _measure_all(web_page)
        ok, detail = verify_R01(m)
        _record_result("R01", ok, detail, m)
        assert ok, f"R01 FAIL: {detail}"

class TestResizeRequirements:
    """R06-R08: Drag resize."""

    def test_R06_drag_down_dashboard_grows(self, web_page):
        _setup_active_session(web_page, with_terminal=True)
        m_before = _measure_all(web_page)
        _drag_gutter(web_page, delta_y=100)
        m_after = _measure_all(web_page)
        ok, detail = verify_R06(m_before, m_after, drag_delta=100)
        _record_result("R06", ok, detail, {"before": m_before, "after": m_after})
        assert ok, f"R06 FAIL: {detail}"
```

### Agentic Loop Contract

The agent (Claude Code) runs this cycle with no human:

1. `uv run pytest tests/test_split_pane_layout.py -v`
2. Read `tests/results/split_pane_results.json`
3. If all passed: done
4. If failures: read `failures` dict for root cause data
5. Modify CSS/JS to fix the highest-priority failure
6. Go to step 1
7. If passed count improved: keep change
8. If passed count same or worse: `git checkout` the change, try different fix

### What This Does NOT Cover

- The actual CSS/JS fixes for the layout bugs — those come from the agent loop
- Browser headed vs headless selection — tests run headless by default, `--headed` flag for debugging
- Real ttyd tests (R12) — these require tmux+ttyd fixtures, included but may be skipped in CI
