---
title: "feat: Unified Study Platform"
type: feat
status: active
date: 2026-03-15
origin: docs/brainstorms/2026-03-15-unified-study-platform-brainstorm.md
---

# feat: Unified Study Platform

## Enhancement Summary

**Deepened on:** 2026-03-15
**Research agents used:** FastAPI+HTMX, MCP Python Server, Click CLI Patterns, PyPI+Homebrew Packaging
**Review agents used:** Security Sentinel, Architecture Strategist, Code Simplicity Reviewer
**Research documents created:** 4 (in `docs/research/`)
**Review documents created:** 3 (in `docs/reviews/`)

### Key Improvements from Deepening

1. **Phases cut from 7 to 5** (Phase 0 pre-work + 4 implementation phases). Config editor, GitHub Issues API, LAN auth, TUI enhancements, WebSocket, migration command, and config versioning all cut as YAGNI.
2. **Phase 0 added**: Consolidate dual config modules, split cli.py into cli/ package, enable WAL mode, formalize JSON schemas -- critical pre-work before feature development.
3. **Service layer pattern**: Business logic in `services/` modules (no framework imports), shared by CLI, FastAPI, and MCP. Prevents duplication across three entry points.
4. **LazyGroup for CLI**: Click's lazy import pattern keeps startup fast despite 30+ commands.
5. **Single uvicorn worker**: SQLite's single-writer lock makes multiple workers pointless. `aiosqlite` for async, WAL mode + `busy_timeout=5000` for concurrent reads.
6. **Security hardening**: Default to `127.0.0.1` (not `0.0.0.0`), require `--lan` flag for LAN exposure. GitHub PAT via `STUDYCTL_GITHUB_TOKEN` env var. CSRF protection for HTMX. Path validation for artefact serving.
7. **MCP: FastMCP v1, stdio transport**: Universal client support, lifespan pattern for shared SQLite/config state, ToolError for expected failures.
8. **HTMX + Alpine.js split**: HTMX for server communication (HTML fragments), Alpine for client-side UI (card flip, timer). Dual routers: `/cards/*` (HTMX HTML) and `/api/cards/*` (JSON backward compat).
9. **PyPI: Trusted Publishing (OIDC)**: No tokens needed. `uv build --package studyctl --no-sources` from workspace. Personal Homebrew tap first (below 75-star threshold for homebrew-core).
10. **10 features cut** (~1,500 lines never written): Config editor UI, GitHub Issues API, LAN password auth, JSON Schema validation library, config schema versioning, migration command, storage.py abstractions, Homebrew core formula, TUI media playback, WebSocket Pomodoro.

### Features Cut (YAGNI -- add later if real demand appears)

| Feature | Why cut | Alternative |
|---------|---------|-------------|
| Config editor in web UI | One user, CLI + text editor works | `studyctl config show` + edit YAML |
| GitHub Issues API feedback | Over-engineering for early stage | Link to GitHub Issues page |
| LAN password auth | Default to localhost. Add `--lan` flag first, auth later | `--lan` flag exposes to network |
| TUI enhancements (Pomodoro, artefacts) | Pomodoro already works client-side. TUI can't play media. | Web UI for artefacts, existing TUI for cards |
| WebSocket for Pomodoro | Purely client-side timer, no server sync needed | Alpine.js countdown in browser |
| Config schema versioning | New fields get defaults on load, no migration needed | `dict.get(key, default)` pattern |
| Migration command | Few existing users, manual migration is fine | Document manual steps |
| JSON Schema validation library | `try/except` on load + clear error messages is simpler | Validate shape on write, graceful on read |
| Homebrew core formula | Below 75-star threshold | Personal tap: `brew tap NetDevAutomate/studyctl` |
| storage.py abstractions | Direct Path operations are simpler | Inline helper functions |

### New Considerations Discovered

- **Dual config modules** (`config.py` vs `settings.py`) must be consolidated before Phase 1 (architecture review)
- **cli.py at 1272 lines** must be split into `cli/` package before absorption (architecture review)
- **CSRF protection** required for HTMX forms -- use `starlette-csrf` middleware (security review)
- **Security headers** (`CSP`, `X-Frame-Options`, `X-Content-Type-Options`) needed on FastAPI (security review)
- **Service worker must NOT cache HTMX fragments** -- only static assets and full pages (FastAPI+HTMX research)
- **Alpine.js v3 auto-initializes** new `x-data` elements after HTMX swaps via MutationObserver -- no special wiring needed (FastAPI+HTMX research)
- **GitHub PAT in plaintext config.yaml is a vulnerability** -- use `STUDYCTL_GITHUB_TOKEN` env var instead (security review)

### Review Documents

| Review | Path |
|--------|------|
| Simplicity review | `docs/reviews/2026-03-15-plan-simplicity-review.md` |
| Architecture review | `docs/reviews/2026-03-15-architecture-review.md` |
| Security review | `docs/reviews/2026-03-15-security-review-unified-study-platform.md` |

### Research Documents (from deepening)

| Research | Path |
|----------|------|
| FastAPI + HTMX best practices | `docs/research/fastapi-htmx-best-practices.md` |
| MCP Python server patterns | `docs/research/mcp-python-server-patterns.md` |
| Click CLI large command groups | `docs/research/click-cli-patterns.md` |
| PyPI + Homebrew packaging | `docs/research/pypi-homebrew-packaging.md` |

---

## Overview

Merge Socratic-Study-Mentor and notebooklm-pdf-by-chapters into a single installable tool (`studyctl`) with a polished FastAPI web UI, complete content-to-study pipeline, agent-generated flashcards via MCP, and guided onboarding. The only AuDHD-aware study tool on the market.

## Problem Statement

1. **Installation barrier**: Non-technical users can't `git clone`, install `uv`, or configure YAML
2. **Fragmentation**: Two repos with no clear journey from "I have an ebook" to "I'm studying"
3. **Interface confusion**: TUI vs PWA vs CLI vs Agent -- no guidance on which to use
4. **Incomplete pipeline**: Content generation and study consumption are disconnected

(see brainstorm: `docs/brainstorms/2026-03-15-unified-study-platform-brainstorm.md`)

## Proposed Solution

Five implementation phases (revised from 7 after simplicity + architecture review):

0. **Pre-work** -- Consolidate config modules, split cli.py into package, enable WAL mode, formalize JSON schemas
1. **Content Absorption** -- Absorb pdf-by-chapters (all 12 commands), course-centric storage, service layer
2. **FastAPI Web UI** -- Replace stdlib, HTMX frontend, artefact viewer, progress dashboard
3. **MCP Agent Integration** -- MCP server, flashcard generation skill, onboarding agent
4. **Packaging & Docs** -- PyPI as `studyctl`, Homebrew tap, `studyctl setup` wizard, user documentation

## Technical Approach

### Architecture

```
                    +-------------------+
                    |  AI Coding        |
                    |  Assistants       |
                    |  (Claude Code,    |
                    |   Gemini CLI,     |
                    |   Kiro CLI, etc.) |
                    +--------+----------+
                             |
                         MCP (tools)
                             |
+----------------------------+-----------------------------+
|                      studyctl                            |
|                                                          |
|  +------------+  +-----------+  +----------------------+ |
|  | Content    |  | Study     |  | Web UI               | |
|  | Pipeline   |  | Engine    |  | (FastAPI + HTMX)     | |
|  | (absorbed  |  | (SM-2,    |  | (artefacts, cards,   | |
|  |  from pdf- |  |  SQLite,  |  |  progress, config,   | |
|  |  by-chaps) |  |  review)  |  |  feedback, auth)     | |
|  +------------+  +-----------+  +----------------------+ |
|                                                          |
|  +------------+  +-----------+  +----------------------+ |
|  | Session    |  | TUI       |  | Agent Definitions    | |
|  | Tools      |  | (Textual) |  | + MCP Server         | |
|  +------------+  +-----------+  +----------------------+ |
+----------------------------------------------------------+
```

### Key Technical Decisions

| Decision | Choice | Rationale | Reference |
|----------|--------|-----------|-----------|
| CLI framework for content commands | Click (rewrite from Typer) | Consistency with studyctl. Extract business logic into modules, thin Click wrappers. | Repo research: Click/Typer mismatch |
| Web backend | FastAPI + uvicorn | Auth middleware, WebSocket, async, Jinja2, validation. Justified by expanded scope. | Brainstorm Decision 2 |
| Frontend | Vanilla HTML/JS + HTMX + Alpine.js | No build step, no node_modules. Jinja2 templates. | Brainstorm Decision 3 |
| Artefact storage | Course-centric under `content.base_path` | Self-contained, browsable per course | Brainstorm Decision 9 |
| LLM integration | Agent CLIs via MCP (no LLM calls in app) | Uses existing subscriptions, zero API cost | Brainstorm Decision 4 |
| NotebookLM | Optional dependency (`studyctl[notebooklm]`) | Fragile unofficial API, not core | Brainstorm Decision 8 |
| Review code dedup | Delete pdf-by-chapters `review.py`, use `review_loader.py` | Identical JSON format, eliminate duplication | Repo research finding 4 |
| SQLite concurrency | Enable WAL mode on all connections | Required for multi-device LAN access | SpecFlow GAP-3.6 |
| Package name | `studyctl` on PyPI and Homebrew | Short, matches CLI command | Brainstorm Decision 10 |

### Resolved SpecFlow Questions

| # | Question | Resolution |
|---|----------|------------|
| Q1 | Course directory structure | Defined in brainstorm Decision 9: `~/study-materials/{course-slug}/chapters/`, `audio/`, `flashcards/`, `quizzes/`, `video/`, `slides/`, `metadata.json` |
| Q2 | FastAPI vs stdlib | FastAPI -- expanded scope demands it (auth, WebSocket, async, templates) |
| Q3 | `--password` auth | HTTP Basic Auth, bcrypt hash in config. CLI arg sets it. Warn about cleartext HTTP. |
| Q4 | Pipeline state | `metadata.json` per course directory (notebook IDs, syllabus state, generation progress) |
| Q5 | Directory traversal | Strict allowlist: resolve path, verify child of `content.base_path`, reject else |
| Q6 | MCP skill definition | Defined in Phase 4 below |
| Q7 | No TOC bookmarks | Fail with clear error + suggest `--ranges "1-30,31-60"` flag |
| Q8 | sync vs content overlap | Deprecate `sync` command with warning pointing to `content from-obsidian` |
| Q9 | Single vs multi user | Single user, multiple devices. No user_id. Document explicitly. |
| Q10 | Migration | `studyctl content migrate <dir>` copies artefacts into course-centric structure |
| Q11 | Cross-machine sync | Sync metadata + JSON only. Media files too large for SSH sync. |
| Q12 | Config editor | Safe fields only (review dirs, theme, base_path). File locking on write. |

### Implementation Phases

---

#### Phase 0: Pre-work (Critical Infrastructure)

**Goal:** Fix structural issues that will compound if left unfixed during feature work. ~2-3 days.

**Tasks:**

- [ ] **Consolidate config modules**: Merge `config.py` (old, TopicConfig namedtuple) and `settings.py` (new, dataclass hierarchy) into single `settings.py`. Update all imports across codebase. (Architecture review: "dual config modules create a maintenance trap")

- [ ] **Split cli.py into cli/ package**: Convert 1272-line `cli.py` into:
  ```
  cli/
    __init__.py      -- root group + lazy registration
    _lazy.py         -- LazyGroup class (40 lines, from Click docs)
    _shared.py       -- shared decorators, Rich console
    _review.py       -- review, win, streak, progress, teachback, bridge
    _config.py       -- config init, config show
    _state.py        -- state push/pull/status/init
    _schedule.py     -- schedule install/remove/list/add/delete
    _web.py          -- web, tui, docs commands
  ```
  Use LazyGroup for fast startup (Click research: "defers imports until command is actually invoked").

- [ ] **Enable WAL mode on all SQLite connections** (`review_db.py`):
  ```python
  conn = sqlite3.connect(db_path)
  conn.execute("PRAGMA journal_mode=WAL")
  conn.execute("PRAGMA busy_timeout=5000")
  ```
  (SpecFlow GAP-3.6, Architecture review: "SQLITE_BUSY under multi-process writes")

- [ ] **Formalize flashcard/quiz JSON contract** -- Document the expected JSON shape in a docstring or comment in `review_loader.py`. Add shape validation on load (`try/except` with clear error message, not JSON Schema library).

- [ ] **Fix known SQL bug** in `get_due_cards()` (flagged in code review items)

- [ ] **Extract service layer stubs** (`services/`):
  ```
  services/
    __init__.py
    review.py        -- get_cards(), record_review(), get_stats()
    content.py       -- (empty, populated in Phase 1)
  ```
  CLI and web server both call service functions. No framework imports in services.

**Success criteria:** All 696+ tests pass, cli.py split into package, config consolidated, WAL mode enabled.

---

#### Phase 1: Content Absorption (replaces original Phase 1)

**Goal:** Absorb all 12 pdf-by-chapters commands into studyctl, unify config, establish course-centric storage.

**Tasks:**

- [ ] Create `packages/studyctl/src/studyctl/content/` package:
  - [ ] `__init__.py` -- package init
  - [ ] `cli.py` -- Click command group (`studyctl content`) with all 12 commands
  - [ ] `splitter.py` -- Port from `pdf_by_chapters/splitter.py` (PyMuPDF TOC splitting)
  - [ ] `notebooklm_client.py` -- Port from `pdf_by_chapters/notebooklm.py` (async API wrapper)
  - [ ] `syllabus.py` -- Port from `pdf_by_chapters/syllabus.py` (state machine, episode tracking)
  - [ ] `markdown_converter.py` -- Port from `pdf_by_chapters/markdown_converter.py` (pandoc pipeline)
  - [ ] `models.py` -- Port from `pdf_by_chapters/models.py` (UploadResult, NotebookInfo, SourceInfo)
  - [ ] `storage.py` -- New: course-centric directory management (create, discover, validate paths)

- [ ] Wire content CLI into main CLI (`packages/studyctl/src/studyctl/cli.py`):
  ```python
  from studyctl.content.cli import content_group
  cli.add_command(content_group)
  ```

- [ ] Content command mapping (all Click, business logic in modules):

  | Command | Source | Target |
  |---------|--------|--------|
  | `studyctl content split <pdf>` | `pdf_by_chapters/cli.py:L68` | `content/cli.py` calls `splitter.split_by_toc()` |
  | `studyctl content process <pdf>` | `pdf_by_chapters/cli.py:L80` | `content/cli.py` calls split + upload |
  | `studyctl content upload` | `pdf_by_chapters/cli.py` (within process) | `content/cli.py` calls `notebooklm_client.upload()` |
  | `studyctl content list` | `pdf_by_chapters/cli.py:L144` | `content/cli.py` calls `notebooklm_client.list_notebooks()` |
  | `studyctl content generate` | `pdf_by_chapters/cli.py:L178` | `content/cli.py` calls `notebooklm_client.generate()` |
  | `studyctl content download` | `pdf_by_chapters/cli.py:L219` | `content/cli.py` calls `notebooklm_client.download()` |
  | `studyctl content delete` | `pdf_by_chapters/cli.py` | `content/cli.py` calls `notebooklm_client.delete()` |
  | `studyctl content syllabus` | `pdf_by_chapters/cli.py` (Syllabus panel) | `content/cli.py` calls `syllabus.generate()` |
  | `studyctl content autopilot` | `pdf_by_chapters/cli.py` (generate-next) | `content/cli.py` calls `syllabus.generate_next()` |
  | `studyctl content status` | `pdf_by_chapters/cli.py` (status) | `content/cli.py` calls `syllabus.status()` |
  | `studyctl content from-obsidian <path>` | `pdf_by_chapters/cli.py` (from-obsidian) | `content/cli.py` calls `markdown_converter` + upload |
  | `studyctl content migrate <dir>` | New | `content/cli.py` copies existing artefacts into course-centric layout |

- [ ] Extend `Settings` dataclass in `settings.py` (L65):
  ```python
  @dataclass
  class ContentConfig:
      base_path: Path = field(default_factory=lambda: Path.home() / "study-materials")
      notebooklm_timeout: int = 900
      inter_episode_gap: int = 30
      default_types: list[str] = field(default_factory=lambda: ["audio"])
      pandoc_path: str = "pandoc"

  # Add to Settings:
  content: ContentConfig = field(default_factory=ContentConfig)
  ```

- [ ] Update `generate_default_config()` (settings.py:L139) to include content section

- [ ] Course-centric storage structure (`content/storage.py`):
  ```python
  def get_course_dir(base_path: Path, slug: str) -> Path:
      """Return course directory, creating subdirs if needed."""
      course_dir = base_path / slug
      for subdir in ("chapters", "audio", "flashcards", "quizzes", "video", "slides"):
          (course_dir / subdir).mkdir(parents=True, exist_ok=True)
      return course_dir

  def slugify_book_title(title: str) -> str:
      """Convert book title to filesystem-safe slug."""

  def load_course_metadata(course_dir: Path) -> dict:
      """Load metadata.json (notebook IDs, syllabus state, generation history)."""

  def save_course_metadata(course_dir: Path, metadata: dict) -> None:
      """Save metadata.json atomically (write to .tmp, rename)."""
  ```

- [ ] Auto-register course directories in `review.directories` config when content is generated (so web UI and TUI discover them automatically)

- [ ] Add dependency check for external tools (`content/cli.py`):
  ```python
  def check_content_dependencies() -> list[str]:
      """Check pandoc, mmdc, typst availability. Return list of missing tools with install instructions."""
  ```

- [ ] Deprecate `studyctl sync` command with warning:
  ```
  Warning: 'studyctl sync' is deprecated. Use 'studyctl content from-obsidian' instead.
  ```

- [ ] Add `--ranges` flag to `content split` for PDFs without TOC:
  ```
  studyctl content split "Book.pdf" --ranges "1-30,31-60,61-90"
  ```

- [ ] New dependencies in `packages/studyctl/pyproject.toml`:
  ```toml
  [project.optional-dependencies]
  content = ["pymupdf>=1.25", "httpx"]
  notebooklm = ["notebooklm-py>=0.3.4"]
  ```

- [ ] Port tests from `notebooklm_pdf_by_chapters/tests/`:
  - [ ] `test_splitter.py` -> `packages/studyctl/tests/test_content_splitter.py`
  - [ ] `test_notebooklm.py` -> `packages/studyctl/tests/test_content_notebooklm.py`
  - [ ] `test_syllabus.py` -> `packages/studyctl/tests/test_content_syllabus.py`
  - [ ] `test_cli.py` (content commands) -> `packages/studyctl/tests/test_content_cli.py`
  - [ ] New: `test_content_storage.py` (course directory management)

**Research insights (Click CLI patterns):**
- Use LazyGroup (from Phase 0) so content commands don't slow startup
- Typer-to-Click conversion is mechanical: `Annotated[str, typer.Option()]` becomes `@click.option()`
- Replace `raise SystemExit(1)` with `click.ClickException` for consistent error formatting
- Business logic in `services/content.py`, thin Click wrappers in `cli/_content.py`

**Success criteria:** `studyctl content split "test.pdf"` works end-to-end, all ported tests pass, `review.directories` auto-discovers content.

---

#### Phase 2: FastAPI Web UI (merges original Phases 2+3)

**Goal:** Replace stdlib HTTP server with FastAPI. Migrate existing 11 routes, add new endpoints, enable async and middleware.

**Tasks:**

- [ ] Add web dependencies to `packages/studyctl/pyproject.toml`:
  ```toml
  [project.optional-dependencies]
  web = ["fastapi>=0.115", "uvicorn[standard]>=0.34", "jinja2>=3.1", "python-multipart>=0.0.12"]
  ```

- [ ] Create FastAPI app structure:
  ```
  packages/studyctl/src/studyctl/web/
    __init__.py
    app.py          -- FastAPI app factory, middleware, startup
    routes/
      __init__.py
      cards.py      -- /api/cards, /api/quizzes, /api/review, /api/session
      courses.py    -- /api/courses, /api/sources, /api/stats, /api/due, /api/wrong
      artefacts.py  -- /api/artefacts, /api/artefacts/{course}/{type}/{filename}
      history.py    -- /api/history, /api/sessions
      content.py    -- /api/content/status, /api/content/split, /api/content/generate
      config.py     -- /api/config (GET/PUT)
      feedback.py   -- /api/feedback (POST -> GitHub Issues)
    templates/       -- Jinja2 templates (HTMX fragments)
      base.html
      index.html
      cards.html
      artefacts.html
      dashboard.html
      settings.html
      feedback.html
      partials/      -- HTMX partial templates
        card.html
        quiz.html
        course_list.html
        artefact_list.html
        progress_chart.html
    static/          -- Existing PWA files (migrated)
      app.js, style.css, sw.js, manifest.json, icons
  ```

- [ ] Migrate existing routes from `server.py` to FastAPI:

  | Old (stdlib) | New (FastAPI) | File |
  |---|---|---|
  | `_handle_courses()` | `GET /api/courses` | `routes/courses.py` |
  | `_handle_cards()` | `GET /api/cards/{course}` | `routes/cards.py` |
  | `_handle_sources()` | `GET /api/sources/{course}` | `routes/courses.py` |
  | `_handle_stats()` | `GET /api/stats/{course}` | `routes/courses.py` |
  | `_handle_due()` | `GET /api/due/{course}` | `routes/courses.py` |
  | `_handle_wrong()` | `GET /api/wrong/{course}` | `routes/courses.py` |
  | `_handle_history()` | `GET /api/history` | `routes/history.py` |
  | `_handle_review()` | `POST /api/review` | `routes/cards.py` |
  | `_handle_session()` | `POST /api/session` | `routes/history.py` |
  | `_handle_audio()` | `GET /artefacts/{course}/{type}/{file}` | `routes/artefacts.py` |
  | Static fallback | `StaticFiles` mount | `app.py` |

- [ ] New endpoints:

  | Endpoint | Method | Purpose | File |
  |----------|--------|---------|------|
  | `/api/artefacts/{course}` | GET | List all artefacts for a course (audio, video, slides, PDFs) | `routes/artefacts.py` |
  | `/api/artefacts/{course}/{type}/{filename}` | GET | Serve artefact file with path validation | `routes/artefacts.py` |
  | `/api/content/status` | GET | Content pipeline status (generation progress) | `routes/content.py` |
  | `/api/config` | GET | Current config (safe fields only) | `routes/config.py` |
  | `/api/config` | PUT | Update safe config fields | `routes/config.py` |
  | `/api/feedback` | POST | Submit bug/feature -> GitHub Issues API | `routes/feedback.py` |
  | `/` | GET | Jinja2 rendered index (with HTMX) | `app.py` |

- [ ] Enable WAL mode on all SQLite connections (`review_db.py`):
  ```python
  conn = sqlite3.connect(db_path)
  conn.execute("PRAGMA journal_mode=WAL")
  ```

- [ ] Artefact file serving with path validation (`routes/artefacts.py`):
  ```python
  def validate_artefact_path(course: str, artefact_type: str, filename: str) -> Path:
      """Resolve path, verify it's a child of content.base_path. Raise 404 if not."""
      base = settings.content.base_path
      resolved = (base / course / artefact_type / filename).resolve()
      if not resolved.is_relative_to(base.resolve()):
          raise HTTPException(status_code=404)
      if not resolved.is_file():
          raise HTTPException(status_code=404)
      return resolved
  ```

- [ ] Update `studyctl web` command in `cli.py` to use FastAPI:
  ```python
  @cli.command("web")
  @click.option("--host", default="0.0.0.0")
  @click.option("--port", default=8567)
  @click.option("--password", default=None, help="Set password for LAN access")
  def web_command(host, port, password):
      from studyctl.web.app import create_app
      import uvicorn
      app = create_app(password=password)
      uvicorn.run(app, host=host, port=port)
  ```

- [ ] Update service worker: cache static assets only, NOT HTMX fragments (FastAPI+HTMX research: "fragments would break if served as full pages")

- [ ] **Default to `127.0.0.1`** (not `0.0.0.0`). Add `--lan` flag to expose to network:
  ```python
  @click.option("--lan", is_flag=True, help="Expose to LAN (default: localhost only)")
  def web_command(host, port, lan):
      bind_host = "0.0.0.0" if lan else "127.0.0.1"
  ```
  (Security review: "refuse 0.0.0.0 by default")

- [ ] Add security headers middleware:
  ```python
  @app.middleware("http")
  async def security_headers(request, call_next):
      response = await call_next(request)
      response.headers["X-Content-Type-Options"] = "nosniff"
      response.headers["X-Frame-Options"] = "DENY"
      return response
  ```

- [ ] **Single uvicorn worker** -- SQLite's single-writer lock makes multiple workers pointless (FastAPI+HTMX research). Use `workers=1` explicitly.

- [ ] **HTMX + Alpine.js integration**: HTMX for server fragments, Alpine for client-side state (card flip, Pomodoro timer countdown). Alpine v3 auto-initializes after HTMX swaps -- no special wiring needed.

- [ ] **Dual router pattern**: `/cards/{course}` returns HTMX HTML fragments, `/api/cards/{course}` returns JSON for backward compat and PWA. Detect HTMX via `HX-Request` header.

- [ ] Tests:
  - [ ] `test_web_app.py` -- FastAPI TestClient for all API endpoints
  - [ ] `test_web_artefacts.py` -- Path validation, directory traversal prevention
  - [ ] `test_web_auth.py` -- Password auth middleware

**Research insights (FastAPI+HTMX):**
- Templates split into `pages/` (full, extend base.html) and `partials/` (bare fragments for HTMX)
- Centralize all `Depends()` in `deps.py` with `Annotated` type aliases
- `aiosqlite` for async DB access, single connection on `app.state.db`
- Alpine.js Pomodoro timer is purely client-side (no WebSocket needed)

**Success criteria:** `studyctl web` launches FastAPI, all existing PWA features work, artefact viewer works, secure by default (localhost).

**Phase 2 also includes (merged from original Phase 3):**

- [ ] Jinja2 base template with navigation: Courses | Dashboard | Artefacts
- [ ] Artefact viewer: course selector -> type tabs (Audio/Video/Slides/Chapters) -> native HTML5 players
- [ ] Progress dashboard: heatmap, streaks, wins (carry forward from existing PWA)
- [ ] Carry forward all existing PWA features: flashcard/quiz review, voice, OpenDyslexic, dark/light theme, Pomodoro, keyboard shortcuts
- [ ] Keep `app.js` as the interactive layer, add HTMX for page navigation
- [ ] Feedback: simple "Report Issue" link to GitHub Issues page (not API integration -- YAGNI)

**Phase 2 explicitly does NOT include (cut per simplicity review):**
- Config editor web UI (use `studyctl config show` + text editor)
- GitHub Issues API feedback (just link to Issues page)
- LAN password authentication (default localhost, add `--lan` flag only)
- WebSocket (Pomodoro is client-side Alpine.js)

---

#### Phase 3: MCP Agent Integration (replaces original Phase 4)

**Goal:** Create MCP server for agent CLI integration, flashcard/quiz generation skill, and onboarding agent.

**Tasks:**

- [ ] Create MCP server (`packages/studyctl/src/studyctl/mcp/`):
  - [ ] `__init__.py`
  - [ ] `server.py` -- FastMCP v1 server with lifespan pattern for shared state
  - [ ] `tools.py` -- Tool implementations (use `@mcp.tool()` decorators)

- [ ] **Use FastMCP v1** (not v2, which is pre-alpha). Lifespan pattern for shared state:
  ```python
  from mcp.server.fastmcp import FastMCP
  from contextlib import asynccontextmanager

  @asynccontextmanager
  async def lifespan(server):
      db = sqlite3.connect(get_db_path())
      db.execute("PRAGMA journal_mode=WAL")
      settings = load_settings()
      yield {"db": db, "settings": settings}
      db.close()

  mcp = FastMCP("studyctl", lifespan=lifespan)
  ```
  (MCP research: "lifespan pattern with @asynccontextmanager, yielding a dataclass")

- [ ] MCP tool definitions:

  | Tool Name | Parameters | Returns | Purpose |
  |-----------|-----------|---------|---------|
  | `generate_flashcards` | `course: str, chapter: int, content: str` | `{path: str, count: int}` | Agent generates flashcard JSON from chapter text, saves to course dir |
  | `generate_quiz` | `course: str, chapter: int, content: str` | `{path: str, count: int}` | Agent generates quiz JSON from chapter text, saves to course dir |
  | `get_study_context` | `course: str` | `{due_cards: int, streak: int, last_session: str, weak_topics: list}` | Agent reads current study state for adaptive mentoring |
  | `record_study_progress` | `course: str, card_hash: str, correct: bool` | `{next_review: str}` | Agent records review result, returns next review date |
  | `list_courses` | (none) | `{courses: [{name, slug, card_count, due_count}]}` | Agent discovers available courses |
  | `get_chapter_text` | `course: str, chapter: int` | `{text: str, title: str}` | Extract text from chapter PDF for LLM processing |

- [ ] Flashcard JSON Schema validation (`studyctl/content/schemas.py`):
  ```python
  FLASHCARD_SCHEMA = {
      "type": "object",
      "required": ["title", "cards"],
      "properties": {
          "title": {"type": "string"},
          "cards": {
              "type": "array",
              "items": {
                  "type": "object",
                  "required": ["front", "back"],
                  "properties": {
                      "front": {"type": "string"},
                      "back": {"type": "string"}
                  }
              }
          }
      }
  }
  ```
  Validate on write (from MCP tool), graceful degradation on read.

- [ ] MCP server entry point (**stdio transport** -- universal client support):
  ```toml
  [project.scripts]
  studyctl-mcp = "studyctl.mcp.server:main"
  ```
  ```python
  def main():
      mcp.run(transport="stdio")
  ```
  Agent CLI registration: `claude mcp add studyctl-mcp` (uses stdio by default)
  Gemini CLI: add to `~/.gemini/settings.json` mcpServers section
  Kiro CLI: add to agent JSON config

- [ ] **Error handling**: Use `ToolError` from `mcp.server.fastmcp.exceptions` for expected failures (e.g., course not found, chapter not available). Unexpected exceptions are caught automatically by the SDK. (MCP research)

- [ ] Create onboarding agent skill (`agents/claude/study-setup.md`):
  - System prompt for conversational setup
  - Uses MCP tools: `list_courses`, `get_study_context`
  - Uses CLI commands: `studyctl config init`, `studyctl content split`
  - Asks: What to learn? Existing knowledge? Where are materials? NotebookLM?
  - Creates/updates config.yaml
  - Generates sample flashcards from first chapter as demo
  - Available for Claude Code, adaptable for Gemini CLI, Kiro CLI

- [ ] Create flashcard generation agent skill (`agents/claude/study-generate.md`):
  - System prompt for structured flashcard/quiz generation
  - Uses MCP tools: `get_chapter_text`, `generate_flashcards`, `generate_quiz`
  - Generates AuDHD-aware content: varied difficulty, Socratic style, topic bridging
  - Quality guidelines embedded in prompt

- [ ] Local TTS generation skill (`agents/claude/study-audio.md`):
  - Agent writes study summary script
  - Calls `study-speak` (kokoro-onnx) to generate .mp3
  - Saves to course `audio/` directory

- [ ] Tests:
  - [ ] `test_mcp_tools.py` -- All MCP tool implementations
  - [ ] `test_mcp_schema_validation.py` -- Flashcard/quiz JSON validation
  - [ ] `test_mcp_server.py` -- MCP server startup and tool registration

**Research insights (MCP Python Server):**
- FastMCP v1 recommended (v2 pre-alpha). `@mcp.tool()` with docstring as description, type hints as JSON Schema.
- Test tools directly as Python functions, or use `ClientSession` + `stdio_client` for integration tests, or `mcp dev` Inspector UI.
- Can mount MCP's `.streamable_http_app()` in FastAPI if HTTP transport needed later.

**Success criteria:** `claude mcp add studyctl-mcp` works, agent generates flashcards via MCP, onboarding agent creates valid config.

---

#### Phase 4: Packaging & Documentation (replaces original Phases 5+7)

**Goal:** Make studyctl installable via PyPI and Homebrew with a guided setup experience.

**Tasks:**

- [ ] Prepare for PyPI publication:
  - [ ] Verify `studyctl` name available on PyPI
  - [ ] Update `packages/studyctl/pyproject.toml`:
    ```toml
    [project]
    name = "studyctl"
    version = "2.0.0"
    description = "AuDHD-aware study tool with AI Socratic mentoring"
    license = "MIT"  # PEP 639 string format (2024+)
    ```
  - [ ] Add classifiers, project-urls, long-description from member README
  - [ ] Add self-referencing meta-extra: `all = ["studyctl[content,web,notebooklm,tui]"]`
  - [ ] Build: `uv build --package studyctl --no-sources` (critical `--no-sources` flag for workspace builds)
  - [ ] Test: `uv tool install ./dist/studyctl-2.0.0-py3-none-any.whl` installs globally

- [ ] **PyPI Trusted Publishing** (OIDC, no tokens):
  ```yaml
  # .github/workflows/publish.yml
  jobs:
    publish:
      permissions:
        id-token: write  # OIDC
      steps:
        - uses: pypa/gh-action-pypi-publish@release/v1
          # No API token needed -- uses OIDC
  ```
  (PyPI research: "generates PEP 740 attestations automatically")

- [ ] Create `studyctl setup` command (alias for enhanced `config init`):
  ```python
  @cli.command("setup")
  def setup_wizard():
      """Interactive setup wizard for new users."""
      # 1. Welcome + what studyctl does
      # 2. Where are your study materials? (path with validation)
      # 3. Do you have an AI coding assistant? (y/n, which)
      # 4. NotebookLM account? (optional)
      # 5. Write config.yaml
      # 6. Offer to launch web UI
  ```

- [ ] Create **personal Homebrew tap** (below 75-star threshold for homebrew-core):
  ```bash
  brew tap-new NetDevAutomate/studyctl  # auto-generates bottle CI
  ```
  Install: `brew install NetDevAutomate/studyctl/studyctl`
  Use `Language::Python::Virtualenv` mixin. `brew update-python-resources` for resource stanzas.
  Optional deps (pandoc, mmdc) in `caveats` block, not `depends_on`.

- [ ] Update `scripts/install.sh` to detect Homebrew and suggest tap install

- [ ] First-run detection in web UI: "Getting Started" page if no courses configured

- [ ] **Documentation** (merged from original Phase 7):
  - [ ] Rewrite README.md with "3 Steps to Start"
  - [ ] Create `docs/user-guide.md` -- non-technical guide
  - [ ] Create `docs/content-pipeline.md` -- ebook-to-study-materials
  - [ ] Update `docs/setup-guide.md` for unified platform
  - [ ] Add "Which interface?" guide (web for most, TUI for terminal, agent for AI mentoring)

- [ ] Tests:
  - [ ] `test_setup_wizard.py` -- CLI wizard flow with mocked input
  - [ ] Build verification: `uv build --package studyctl --no-sources && uv tool install` round-trip

**Research insights (PyPI+Homebrew):**
- `--no-sources` flag disables `tool.uv.sources` so sdist/wheel builds correctly outside workspace
- PEP 639: `license = "MIT"` as string is the new standard; `License::` classifiers deprecated
- Homebrew: resource stanzas auto-generated via `brew update-python-resources`
- Both `uv tool install` and `pipx install` work identically from PyPI

**Success criteria:** `pip install studyctl && studyctl setup && studyctl web` works end-to-end from a clean machine.

---

#### Deferred Features (add when real demand appears)

These features were in the original 7-phase plan but cut after simplicity review. They can be added as standalone PRs when user feedback justifies them:

- **LAN password auth**: Add `--password` flag + HTTP Basic Auth middleware. Only if users actually expose to LAN.
- **GitHub Issues API feedback**: Replace link with API integration. Only if link-based feedback is insufficient.
- **Config editor web UI**: Add settings page. Only if users complain about editing YAML.
- **TUI artefact browser**: Open audio/PDF via external players. Only if TUI users request it.
- **Config schema versioning**: Add `schema_version` field. Only if config format changes significantly.
- **Migration command**: `studyctl content migrate`. Only if pdf-by-chapters has many active users.

---

## Alternative Approaches Considered

| Approach | Why rejected | Reference |
|----------|------------|-----------|
| Native Swift app (macOS + iOS) | 3-6 months to reach parity with existing Python codebase. Ship now, native later. | Brainstorm: deferred to Phase 2 |
| Keep repos separate, add as dependency | Still two repos to maintain, confusing for users | Brainstorm Q&A |
| Electron/Tauri desktop wrapper | Heavy runtime, not native, still bundles Python | Brainstorm: Why Not table |
| LLM API calls in web UI | Agent CLIs handle AI. Users already pay for subscriptions. | Brainstorm Decision 4 |
| Fork notebooklm-py | Maintenance burden of tracking unofficial API. Use upstream. | Brainstorm Q2 resolution |

## System-Wide Impact

### Interaction Graph

- `studyctl content split` -> `splitter.split_by_toc()` -> writes to `content.base_path/{slug}/chapters/`
- `studyctl content autopilot` -> `syllabus.generate_next()` -> `notebooklm_client.generate()` -> polls -> `notebooklm_client.download()` -> writes to `content.base_path/{slug}/audio/` etc.
- MCP `generate_flashcards` tool -> agent LLM generates JSON -> validates against schema -> writes to `content.base_path/{slug}/flashcards/`
- `review_loader.discover_directories()` reads `review.directories` config -> finds flashcard/quiz JSON -> loads into web UI and TUI
- `review_db.record_card_review()` writes to sessions.db (WAL mode) -> `review_db.get_due_cards()` reads SM-2 schedule

### Error Propagation

- NotebookLM cookie expiry -> `notebooklm_client` raises `AuthenticationError` -> `content autopilot` catches, logs "Cookie expired. Run `notebooklm login` to re-authenticate.", saves state to `metadata.json` for resume
- PDF without TOC -> `splitter.split_by_toc()` returns empty list -> CLI shows "No TOC bookmarks found. Use --ranges to specify page ranges manually."
- Missing pandoc -> `markdown_converter` raises `DependencyError` -> CLI shows "pandoc not found. Install: brew install pandoc"
- Malformed flashcard JSON from LLM -> schema validation catches -> MCP tool retries once, then returns error to agent

### State Lifecycle Risks

- **Content pipeline interruption**: `metadata.json` per course tracks generation state. Resume reads state, skips completed chapters. Atomic writes (tmp + rename) prevent corruption.
- **SQLite concurrent writes**: WAL mode handles multi-device access. SM-2 updates are per-card, low contention. No transactions span multiple operations.
- **Config file editing**: File locking (fcntl) on write. Read is lock-free. Web UI config editor is the only writer besides CLI.

## Acceptance Criteria

### Functional Requirements

- [ ] `studyctl content split "Book.pdf"` splits by TOC into course-centric directory
- [ ] `studyctl content autopilot` generates and downloads artefacts (with NotebookLM optional extra)
- [ ] `studyctl content from-obsidian ~/vault/` converts and processes Obsidian notes
- [ ] `studyctl web` launches FastAPI with flashcard review, artefact viewer, and progress dashboard
- [ ] `studyctl web --password secret` requires authentication for LAN access
- [ ] `studyctl tui` includes Pomodoro timer, voice toggle, and artefact browser
- [ ] `studyctl-mcp` exposes MCP tools for agent CLI integration
- [ ] Agent generates flashcards/quizzes via MCP tools
- [ ] `/study-setup` agent skill creates valid config conversationally
- [ ] `studyctl setup` CLI wizard creates valid config interactively
- [ ] In-app feedback button creates GitHub Issues
- [ ] `pip install studyctl && studyctl setup && studyctl web` works from clean machine

### Non-Functional Requirements

- [ ] All ported tests pass (target: 800+ tests after absorption)
- [ ] Artefact serving is secure against directory traversal (path validation tests)
- [ ] SQLite uses WAL mode for concurrent access
- [ ] No ruff warnings, pyright clean
- [ ] Web UI responsive on mobile (existing CSS carries forward)
- [ ] PWA installable and works offline for review (service worker)

### Quality Gates

- [ ] Pre-commit: ruff lint + format
- [ ] Pre-push: full pytest suite
- [ ] CI: Python 3.11/3.12/3.13 matrix
- [ ] Each phase has its own test suite before proceeding to next

## Dependencies & Prerequisites

| Dependency | Phase | Type | Notes |
|-----------|-------|------|-------|
| `pymupdf>=1.25` | 1 | Python (content extra) | PDF splitting |
| `httpx` | 1 | Python (content extra) | Async HTTP for NotebookLM |
| `notebooklm-py>=0.3.4` | 1 | Python (notebooklm extra) | Optional NotebookLM integration |
| `fastapi>=0.115` | 2 | Python (web extra) | Web backend |
| `uvicorn[standard]>=0.34` | 2 | Python (web extra) | ASGI server |
| `jinja2>=3.1` | 2 | Python (web extra) | Templates |
| `python-multipart>=0.0.12` | 2 | Python (web extra) | Form handling |
| `mcp` | 4 | Python | MCP server SDK |
| `pandoc` | 1 | System | Markdown conversion (checked, not required) |
| `mmdc` (mermaid-cli) | 1 | System | Mermaid diagrams (checked, not required) |

## Risk Analysis & Mitigation

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| FastAPI migration breaks existing PWA | High | Medium | Keep JSON API backward-compatible. Migrate route-by-route with tests. |
| pdf-by-chapters absorption introduces bugs | Medium | Medium | Port tests first, run against ported code. |
| Click/Typer mismatch during CLI rewrite | Low | High | Extract business logic into framework-agnostic modules first. |
| NotebookLM API changes break content pipeline | Medium | High | NotebookLM is optional. Core pipeline (agent-generated) doesn't depend on it. |
| MCP SDK instability | Medium | Low | MCP Python SDK is mature. Pin version. |
| Scope creep across 7 phases | High | Medium | Each phase has clear success criteria. Ship incrementally. |

## Success Metrics

- **Installation**: Time from `brew install studyctl` to first study session < 10 minutes
- **Usage**: Web UI serves flashcards, quizzes, and artefacts without errors
- **Coverage**: Test count > 800, all passing
- **Feedback**: Users can submit feedback from the web UI
- **Agent integration**: At least one MCP tool works end-to-end with Claude Code

## Documentation Plan

| Document | Action | Phase |
|----------|--------|-------|
| `README.md` | Rewrite with "3 Steps to Start" | 4 |
| `docs/user-guide.md` | New: non-technical user guide | 4 |
| `docs/content-pipeline.md` | New: ebook-to-study-materials guide | 1 |
| `docs/setup-guide.md` | Update for unified platform | 4 |
| `docs/cli-reference.md` | Add content commands | 1 |
| `docs/agent-install.md` | Update with MCP setup | 3 |
| `CONTRIBUTING.md` | Update dev setup, new package structure | 0 |

## Sources & References

### Origin

- **Brainstorm document:** [docs/brainstorms/2026-03-15-unified-study-platform-brainstorm.md](docs/brainstorms/2026-03-15-unified-study-platform-brainstorm.md)
  - Key decisions carried forward: absorb pdf-by-chapters fully, FastAPI + HTMX, agent CLIs as AI brain, course-centric storage, NotebookLM optional, package name `studyctl`

### Internal References

- CLI structure: `packages/studyctl/src/studyctl/cli.py:L112` (Click groups pattern)
- Web server: `packages/studyctl/src/studyctl/web/server.py` (11 routes to migrate)
- Settings: `packages/studyctl/src/studyctl/settings.py:L65` (Settings dataclass to extend)
- Review loader: `packages/studyctl/src/studyctl/review_loader.py` (JSON format contract)
- Review DB: `packages/studyctl/src/studyctl/review_db.py` (SM-2 schema, needs WAL)
- pdf-by-chapters CLI: `/Users/ataylor/code/personal/tools/notebooklm_pdf_by_chapters/src/pdf_by_chapters/cli.py` (1300 lines, 12 commands)
- pdf-by-chapters splitter: `/Users/ataylor/code/personal/tools/notebooklm_pdf_by_chapters/src/pdf_by_chapters/splitter.py`

### Research Documents

| Topic | Path |
|---|---|
| Repo research (integration points) | `docs/research/unified-platform-repo-research.md` |
| SpecFlow analysis (edge cases, gaps) | `docs/research/unified-platform-specflow.md` |
| MCP in native apps | `docs/research/mcp-native-app-research.md` |
| ACP maturity | `docs/research/2026-03-15-acp-research.md` |
| Swift PoC feasibility | `docs/research/swift-poc-feasibility.md` |
| Monetisation models | `docs/research/monetisation-research.md` |
| SSM repo analysis | `docs/research/repo-research-summary.md` |
| pdf-by-chapters analysis | `docs/research/notebooklm-pdf-by-chapters-analysis.md` |
