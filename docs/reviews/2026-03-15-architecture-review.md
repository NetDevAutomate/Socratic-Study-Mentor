# Architecture Review: Unified Study Platform Plan

**Reviewer**: Architecture Strategist Agent
**Date**: 2026-03-15
**Plan**: `docs/plans/2026-03-15-feat-unified-study-platform-plan.md`
**Verdict**: CONDITIONALLY APPROVED -- 3 blocking issues, 6 significant recommendations

---

## 1. Architecture Overview

The plan merges `notebooklm-pdf-by-chapters` (Typer CLI, 1300-line cli.py, 12 commands) into the `studyctl` workspace member, adds FastAPI to replace the stdlib HTTP server, introduces an MCP server for agent integration, and extends the existing dataclass-based Settings. The target is a single `pip install studyctl` with optional extras.

Current studyctl has 20 source files, a 1272-line cli.py, dual config modules (config.py + settings.py), and an stdlib web server with 11 routes. agent-session-tools is a separate workspace member with its own MCP speak tool.

---

## 2. Blocking Issues

### BLOCK-1: SQLite Concurrency Model Is Underspecified

The plan states "WAL mode handles multi-device access" and "SM-2 updates are per-card, low contention." This is dangerously optimistic.

**The actual risk**: FastAPI runs async with uvicorn workers. The TUI runs in a separate process. The MCP server is yet another process. CLI commands are ad-hoc. All hit the same SQLite file. WAL mode permits concurrent reads with a single writer, but:

- FastAPI with multiple workers means multiple processes writing. WAL does NOT prevent `SQLITE_BUSY` under concurrent writes from separate processes.
- There is no connection pooling strategy defined.
- `review_db.py` creates and destroys connections per function call (no context managers, as the Kieran review notes). Under FastAPI async, this is a connection leak waiting to happen.
- The plan says "No transactions span multiple operations" -- but `record_card_review()` does a SELECT then INSERT that should be atomic.

**Required resolution**: Define a connection management strategy before Phase 2. Specifically:
1. Wrap all `review_db.py` functions with context managers (`with sqlite3.connect(...) as conn`).
2. FastAPI must use a single connection per request via dependency injection, not per-function connections.
3. Set `PRAGMA busy_timeout = 5000` on every connection (not just WAL mode).
4. Document that uvicorn must run single-worker for SQLite safety, or switch to a connection broker pattern.
5. The MCP server and CLI are separate processes -- accept `SQLITE_BUSY` retries with exponential backoff.

### BLOCK-2: Dual Config Modules Create a Maintenance Trap

The plan extends `Settings` in `settings.py` with `ContentConfig` but does not address the existing `config.py` module. Right now:

- `cli.py` imports `Topic` and `get_topics()` from `config.py`
- `settings.py` has `TopicConfig` dataclass with the same data
- `review_db.py` uses `config_path.py` for DB path
- The `web` command reads `config.yaml` directly via `yaml.safe_load` (bypassing both modules)

Adding a third config path (`ContentConfig`) on top of this creates three parallel ways to read the same YAML file. Phase 1 must consolidate config.py into settings.py first, or the content absorption will entrench the duplication.

**Required resolution**: Add a Phase 0 or make it the first task of Phase 1:
1. Migrate `config.py` Topic/get_topics into `settings.py` as TopicConfig (already exists there).
2. Update all cli.py imports to use `settings.load_settings()`.
3. Remove `config.py`.
4. Make the `web` command use `Settings` instead of raw yaml.safe_load.
5. Then add ContentConfig to the unified Settings.

### BLOCK-3: cli.py Is Already 1272 Lines -- Absorption Will Make It Unmanageable

The plan proposes adding `content/cli.py` as a separate module (good), but does not address the existing 1272-line cli.py. After absorption, content commands will be in a separate file, but the existing commands (review, schedule, bridge, config, docs, web, tui, teachback, progress, wins, streaks, auto-resume) remain in one monolithic file.

**Required resolution**: Phase 1 must split cli.py into submodules before adding content commands:
```
studyctl/
  cli/
    __init__.py      -- cli group, shared imports
    review.py        -- review, schedule-blocks
    schedule.py      -- schedule group
    bridge.py        -- bridge group
    config.py        -- config group
    content.py       -- NEW: content group (from absorption)
    web.py           -- web command
    tui.py           -- tui command
```
This is not optional polish -- without it, the file will exceed 1500 lines and become the primary source of merge conflicts across all 7 phases.

---

## 3. Significant Recommendations

### REC-1: MCP Server Must Be a Separate Process, Not Embedded in FastAPI

The plan correctly chooses a separate `studyctl-mcp` entry point. This is the right call. However, the plan does not discuss why. Document the rationale explicitly:

- MCP servers communicate over stdio (stdin/stdout). FastAPI communicates over HTTP/WebSocket. They are fundamentally different transport models.
- Embedding MCP in FastAPI would require a WebSocket-to-stdio bridge, adding complexity with no benefit.
- Agent CLIs (Claude Code, Gemini CLI) expect `stdio` transport. A separate binary is the only clean option.
- The MCP server and FastAPI share business logic modules (`review_db`, `review_loader`, `content/`) but not transport. This is correct layering.

**Action**: Add an ADR documenting this decision and the shared-module pattern.

### REC-2: Extract a Service Layer Between CLI/Web/MCP and Data Access

The plan says "Extract business logic into framework-agnostic modules first" for the Click rewrite. This is correct but should be formalized as a pattern. Right now:

- CLI commands directly call `review_db.record_card_review()` and `review_loader.load_flashcards()`
- Web handlers do the same
- MCP tools will do the same

This works today because operations are simple. But the plan adds: course auto-registration, content generation state tracking, artefact discovery, and feedback submission. These are multi-step operations that should not be duplicated across three entry points.

**Action**: Create a `studyctl/services/` layer:
```python
# studyctl/services/review.py
class ReviewService:
    def __init__(self, db_path: Path, content_base: Path): ...
    def get_due_cards(self, course: str) -> list[CardProgress]: ...
    def record_review(self, course: str, card_hash: str, correct: bool) -> None: ...
    def get_course_stats(self, course: str) -> CourseStats: ...

# studyctl/services/content.py
class ContentService:
    def __init__(self, settings: Settings): ...
    def split_pdf(self, path: Path, ...) -> CourseMeta: ...
    def discover_courses(self) -> list[Course]: ...
```

CLI, FastAPI routes, and MCP tools all depend on service classes. Service classes depend on data access modules. This is textbook Dependency Inversion and will prevent the "three places to fix every bug" problem.

### REC-3: JSON-on-Disk Contract Needs a Schema File

The plan acknowledges GAP-2.2 (no JSON Schema validation) and proposes validation on write. Good. But the contract is more fragile than stated:

- `review_loader.py` reads `*flashcards.json` and `*quiz.json` by glob pattern
- The content pipeline writes these files
- MCP agents write these files
- There is no versioning on the format

**Action**:
1. Create `studyctl/schemas/flashcard.schema.json` and `studyctl/schemas/quiz.schema.json` as formal JSON Schema files.
2. Validate on write (MCP tools, content pipeline). Reject malformed data before it reaches disk.
3. Validate on read with graceful degradation (log warning, skip bad file, continue).
4. Include a `"schema_version": 1` field so you can evolve the format without breaking existing files.

### REC-4: Phase Ordering Has a Hidden Dependency

Phase 4 (MCP + Agent Skills) depends on Phase 1 (content pipeline) for course directory structure and Phase 2 (FastAPI) for nothing. But Phase 4 also writes flashcard/quiz JSON that Phase 2's web UI serves. The data contract must be stable before both Phase 2 and Phase 4 can proceed independently.

**Action**: Formalize the JSON schema in Phase 1 (not Phase 4). Phase 1 defines the contract. Phase 2 reads it. Phase 4 writes to it. This is already implicit but should be an explicit deliverable of Phase 1 with tests.

### REC-5: The `content` Extra Should Include `httpx` Only If NotebookLM Needs It

The plan puts `httpx` in the `content` extra alongside `pymupdf`. But `httpx` is only needed for NotebookLM API calls. PDF splitting does not need HTTP. Users who want `studyctl content split` should not pull in `httpx`.

**Action**: Move `httpx` to the `notebooklm` extra:
```toml
[project.optional-dependencies]
content = ["pymupdf>=1.25"]
notebooklm = ["notebooklm-py>=0.3.4", "httpx"]
web = ["fastapi>=0.115", "uvicorn[standard]>=0.34", "jinja2>=3.1", "python-multipart>=0.0.12"]
tui = ["textual>=0.80"]
all = ["studyctl[content,notebooklm,web,tui]"]
```

Also add an `all` extra for convenience. Users who want everything should be able to `pip install studyctl[all]`.

### REC-6: Test Count Claim Needs Verification

The plan claims 696+ tests and targets 800+. The actual count from `grep -c 'def test_'` across all test files is 445 test functions. The 696 figure likely comes from pytest parametrize expanding to 696 collected tests. This is fine, but the plan should be precise: "696 collected test items (445 test functions with parametrize expansion)." The target of 800+ collected items after absorbing pdf-by-chapters tests is reasonable.

**Action**: Clarify the metric. Use `pytest --collect-only -q | tail -1` as the definitive count. Ensure CI fails if the count drops below a floor (add `--tb=short -q` output parsing or a pytest plugin).

---

## 4. Minor Observations

| # | Item | Severity | Note |
|---|------|----------|------|
| M1 | `mcp` package is listed without version pin | Low | Pin to `>=1.0,<2.0` to avoid SDK breaking changes |
| M2 | `hatchling` build backend in studyctl, `setuptools` at workspace root | Low | Inconsistency. Not a problem today but confusing for contributors |
| M3 | `pyright exclude` for `src/studyctl/tui` will need extending to `mcp/` and `content/` | Low | Anything behind an optional import needs pyright exclusion or TYPE_CHECKING guards |
| M4 | Phase 5 mentions Homebrew formula | Low | Formula needs testing on both Intel and Apple Silicon. Add to acceptance criteria |
| M5 | The Kieran review (code-review-plan-items.md) identifies a real SQL bug in get_due_cards | Medium | Fix this BEFORE Phase 2 migrates the web server, not during. It is a pre-existing defect |
| M6 | No mention of database migrations | Medium | Adding WAL mode, potentially new tables for content metadata -- need a migration strategy (review_db already has ensure_tables but no versioning) |

---

## 5. Recommended Phase Ordering (Revised)

```
Phase 0 (pre-work):
  - Consolidate config.py into settings.py
  - Split cli.py into cli/ package
  - Fix get_due_cards SQL bug
  - Add context managers to review_db.py
  - Define JSON schemas for flashcard/quiz format

Phase 1: Content Pipeline Absorption (as planned, minus config/CLI debt)

Phase 2: FastAPI Backend (as planned, with connection management strategy)

Phase 3: Web UI Polish (as planned)

Phase 4: MCP + Agent Integration (as planned, JSON schema already defined in Phase 0)

Phase 5: Packaging (as planned)

Phase 6: TUI Enhancements (as planned)

Phase 7: Feedback & Polish (as planned)
```

The Phase 0 work is 1-2 sessions and eliminates all three blocking issues. Without it, every subsequent phase inherits technical debt that compounds.

---

## 6. What the Plan Gets Right

- Absorbing pdf-by-chapters rather than maintaining two repos is correct. The JSON format contract between content generation and review consumption is already established and working.
- Click over Typer for the absorbed commands is correct -- consistency with the existing CLI matters more than Typer's marginal ergonomics.
- NotebookLM as an optional extra is correct -- it is an unstable unofficial API and should not be a hard dependency.
- Separate `studyctl-mcp` entry point is correct -- stdio transport is incompatible with HTTP embedding.
- Course-centric directory structure (`~/study-materials/{slug}/`) is correct -- self-contained, discoverable, and the right level of convention over configuration.
- The 7-phase incremental approach with per-phase success criteria is well-structured for managing scope.
