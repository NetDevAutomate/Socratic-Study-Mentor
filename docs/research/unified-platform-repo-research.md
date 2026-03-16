# Unified Platform Repository Research

> Generated: 2026-03-15
> Purpose: Research findings for absorbing notebooklm-pdf-by-chapters into studyctl, replacing stdlib HTTP with FastAPI, adding HTMX/Jinja2, and related features.

---

## 1. Click CLI Structure (studyctl)

**File**: `/Users/ataylor/code/personal/tools/Socratic-Study-Mentor/packages/studyctl/src/studyctl/cli.py`
**Lines**: 1272

### Entry Point

```
[project.scripts]
studyctl = "studyctl.cli:cli"
```

### Top-Level Group

```python
@click.group()
def cli() -> None:
    """studyctl -- sync, plan, and schedule study sessions."""
```

### Current Command Tree (with line numbers)

| Line | Type | Name | Description |
|------|------|------|-------------|
| L112 | `@click.group()` | `cli` | Root group |
| L116 | `@cli.command()` | `sync` | Sync sources to NotebookLM |
| L153 | `@cli.command()` | `status` | Show sync status |
| L186 | `@cli.command()` | `audio` | Generate audio |
| L211 | `@cli.command()` | `topics` | List topics |
| L221 | `@cli.command()` | `dedup` | Deduplicate notebook sources |
| L260 | `@cli.group("state")` | `state_group` | Remote state management |
| L267 | subcommand | `state push` | Push state to remote |
| L284 | subcommand | `state pull` | Pull state from remote |
| L299 | subcommand | `state status` | Show sync status |
| L313 | subcommand | `state init` | Init remote state |
| L323 | `@cli.group("config")` | `config_group` | Configuration management |
| L334 | subcommand | `config init` | Interactive setup |
| L349 | subcommand | `config show` | Display current config |
| L445 | `@cli.group("schedule")` | `schedule_group` | Scheduled jobs (launchd/cron) |
| L452 | subcommand | `schedule install` | Install all jobs |
| L462 | subcommand | `schedule remove` | Remove all jobs |
| L470 | subcommand | `schedule list` | List active jobs |
| L486 | subcommand | `schedule add` | Add custom job |
| L500 | subcommand | `schedule delete` | Delete a job |
| L510 | `@cli.command()` | `schedule-blocks` | Generate calendar blocks |
| L553 | `@cli.command()` | `review` | Launch spaced repetition review |
| L580 | `@cli.command()` | `win` | Record a study win |
| L612 | `@cli.command()` | `resume` | Resume last session |
| L640 | `@cli.command()` | `streak` | Show study streak |
| L672 | `@cli.command()` | `progress-map` | Visual progress map |
| L720 | `@cli.command()` | `teachback` | Record teach-back score |
| L780 | `@cli.command()` | `bridge` | Record knowledge bridge |
| L830 | `@cli.group("docs")` | `docs_group` | Documentation commands |
| L860 | `@cli.command()` | `serve` | Start web PWA server |
| L900 | `@cli.command()` | `tui` | Launch Textual TUI |

### How to Add a New `content` Group

Pattern follows existing groups (`state`, `config`, `schedule`, `docs`):

```python
@cli.group(name="content")
def content_group() -> None:
    """Manage course content -- split, convert, generate."""

@content_group.command(name="split")
def content_split(...) -> None:
    ...

@content_group.command(name="from-obsidian")
def content_from_obsidian(...) -> None:
    ...
```

**Integration point**: Add `from .content import content_group` near the top of cli.py, then `cli.add_command(content_group)` or use the decorator pattern. Given the file is already 1272 lines, a separate `content_cli.py` module is strongly recommended, with lazy import in cli.py.

---

## 2. Current Web Server Implementation

**File**: `/Users/ataylor/code/personal/tools/Socratic-Study-Mentor/packages/studyctl/src/studyctl/web/server.py`

### Architecture

- Uses **stdlib `http.server.SimpleHTTPRequestHandler`** -- zero external dependencies
- Single `StudyHandler` class with manual URL dispatch in `do_GET`/`do_POST`
- Static files served from `web/static/` directory (7 files: index.html, app.js, style.css, sw.js, manifest.json, icon-192.svg, icon-512.svg)
- All frontend logic is in a single `app.js` file (vanilla JS, no framework)

### Current API Routes

| Method | Path | Handler | Purpose |
|--------|------|---------|---------|
| GET | `/api/courses` | `_handle_courses()` | List all discovered courses |
| GET | `/api/cards/<course>?mode=` | `_handle_cards()` | Load flashcards or quizzes |
| GET | `/api/sources/<course>` | `_handle_sources()` | List content sources for a course |
| GET | `/api/stats/<course>` | `_handle_stats()` | Get review stats |
| GET | `/api/due/<course>` | `_handle_due()` | Get due cards (spaced repetition) |
| GET | `/api/wrong/<course>` | `_handle_wrong()` | Get wrong answer hashes |
| GET | `/api/history` | `_handle_history()` | Session history |
| POST | `/api/review` | `_handle_review()` | Record individual card review |
| POST | `/api/session` | `_handle_session()` | Record session results |
| GET | `/audio/<path>` | `_handle_audio()` | Serve audio files from course dirs |
| * | (everything else) | `SimpleHTTPRequestHandler` | Static file fallback |

### serve() Function Signature

```python
def serve(
    host: str = "localhost",
    port: int = 8567,
    study_dirs: list[str] | None = None,
) -> None:
```

### What Needs to Change for FastAPI

1. **Replace**: `HTTPServer` + `SimpleHTTPRequestHandler` with `FastAPI` + `uvicorn`
2. **Routes**: Convert manual URL dispatch to `@app.get()`/`@app.post()` decorators
3. **Static files**: Use `fastapi.staticfiles.StaticFiles` mount
4. **Templates**: Add Jinja2 `templates/` directory alongside `static/`
5. **HTMX**: Return HTML fragments from endpoints instead of JSON for UI updates
6. **JSON API**: Keep `/api/*` endpoints returning JSON for backward compat
7. **Audio serving**: Use `FileResponse` from starlette
8. **Dependencies**: Add `fastapi`, `uvicorn[standard]`, `jinja2` to studyctl pyproject.toml
9. **LAN access**: Change default `host` from `"localhost"` to `"0.0.0.0"` (with auth middleware)

### Static Files (current PWA)

```
web/static/
  index.html       -- Single-page app shell
  app.js           -- All frontend logic (vanilla JS)
  style.css        -- Tokyo Night theme, all component styles
  sw.js            -- Service worker for offline
  manifest.json    -- PWA manifest
  icon-192.svg     -- App icon
  icon-512.svg     -- App icon
```

---

## 3. Configuration Structure

### settings.py

**File**: `/Users/ataylor/code/personal/tools/Socratic-Study-Mentor/packages/studyctl/src/studyctl/settings.py`
**Lines**: 208

### Dataclass Hierarchy

```
Settings (L65)
  obsidian_base: Path          -- ~/Obsidian
  session_db: Path             -- ~/.config/studyctl/sessions.db
  state_dir: Path              -- ~/.local/share/studyctl
  topics: list[TopicConfig]
  sync_remote: str
  sync_user: str
  knowledge_domains: KnowledgeDomainsConfig
  notebooklm: NotebookLMConfig

TopicConfig (L30)
  name: str
  slug: str
  obsidian_path: Path
  notebook_id: str
  tags: list[str]

KnowledgeDomainsConfig (L49)
  primary: str
  anchors: list[dict]
  secondary: list[KnowledgeDomain]

KnowledgeDomain (L41)
  domain: str
  anchors: list[str]

NotebookLMConfig (L58)
  enabled: bool
```

### Key Functions

- `load_settings()` (L80) -- Loads from `~/.config/studyctl/config.yaml` with defaults
- `generate_default_config()` (L139) -- Generates default YAML config

### config.py (Legacy/Minimal)

**File**: `/Users/ataylor/code/personal/tools/Socratic-Study-Mentor/packages/studyctl/src/studyctl/config.py`

Provides `Topic` namedtuple and `get_topics()` -- reads the same config.yaml. This is the older config module used by cli.py imports. `settings.py` is the newer centralized loader.

### How to Extend for Content Pipeline

Add new dataclasses to settings.py:

```python
@dataclass
class ContentConfig:
    """Configuration for the content generation pipeline."""
    base_path: Path = field(default_factory=lambda: Path.home() / "StudyCourses")
    pandoc_path: str = "pandoc"
    mermaid_filter: str = "mermaid-filter"
    default_toc_level: int = 1
    notebooklm_timeout: int = 900

# Add to Settings:
    content: ContentConfig = field(default_factory=ContentConfig)
```

### Config YAML Structure (current live config)

```yaml
obsidian_base: ~/Obsidian/Personal
session_db: ~/.config/studyctl/sessions.db
state_dir: ~/.local/share/studyctl
topics:
  - name: "Course Name"
    slug: course-slug
    obsidian_path: ~/Obsidian/path/to/course
    notebook_id: "uuid"
    tags: [tag1, tag2]
review:
  directories:
    - ~/Desktop/ZTM-DE/downloads
    - ~/Obsidian/Personal/2-Areas/Courses
  default_mode: flashcards
  shuffle: true
  voice_enabled: false
tui:
  theme: ""
  dyslexic_friendly: false
hosts:
  hostname1:
    address: ip-or-hostname
    user: username
```

**Note**: The `review.directories` list is used by both TUI and web server to discover flashcard/quiz JSON files. The content pipeline should write its output into directories that appear here.

---

## 4. Review System (Flashcards/Quizzes)

### review_loader.py

**File**: `/Users/ataylor/code/personal/tools/Socratic-Study-Mentor/packages/studyctl/src/studyctl/review_loader.py`

### Data Format Contract (JSON)

**Flashcard JSON** (`*flashcards.json`):
```json
{
  "title": "Section Name",
  "cards": [
    {"front": "Question text", "back": "Answer text"}
  ]
}
```

**Quiz JSON** (`*quizzes.json`):
```json
{
  "title": "Section Name",
  "questions": [
    {
      "question": "Question text",
      "answerOptions": [
        {"text": "Option A", "isCorrect": false},
        {"text": "Option B", "isCorrect": true}
      ]
    }
  ]
}
```

### Dataclasses

```python
@dataclass
class Flashcard:
    front: str
    back: str
    source: str = ""
    # card_hash property: sha256 of front+back for SM-2 tracking

@dataclass
class QuizQuestion:
    question: str
    options: list[str]
    correct_index: int
    source: str = ""
    # card_hash property

@dataclass
class CardProgress:
    card_hash: str
    correct: int = 0
    incorrect: int = 0
    last_correct: bool = False
    reviewed_hashes: set[str] = field(default_factory=set)
    # score_pct property
```

### Key Functions

| Function | Purpose |
|----------|---------|
| `load_flashcards(directory: Path) -> list[Flashcard]` | Glob `*flashcards.json`, parse all cards |
| `load_quizzes(directory: Path) -> list[QuizQuestion]` | Glob `*quizzes.json`, parse all questions |
| `discover_directories(dirs: list[str]) -> dict[str, Path]` | Map course names to content directories |
| `find_content_dirs(base: Path) -> list[Path]` | Find subdirs containing flashcard/quiz files |

### Directory Discovery Logic

Scans `review.directories` from config. For each directory, looks for `flashcards/` and `quizzes/` subdirectories. Course name is derived from parent directory name (with special handling: if dir name is "downloads", uses grandparent name).

### review_db.py

**File**: `/Users/ataylor/code/personal/tools/Socratic-Study-Mentor/packages/studyctl/src/studyctl/review_db.py`

### Schema (2 tables, studyctl-owned)

```sql
CREATE TABLE IF NOT EXISTS card_reviews (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    course TEXT NOT NULL,
    card_type TEXT NOT NULL,       -- "flashcard" or "quiz"
    card_hash TEXT NOT NULL,       -- sha256 from review_loader
    correct BOOLEAN NOT NULL,
    reviewed_at TEXT NOT NULL,     -- ISO 8601
    ease_factor REAL DEFAULT 2.5,
    interval_days INTEGER DEFAULT 1,
    next_review TEXT,              -- ISO 8601 date
    response_time_ms INTEGER
);

CREATE TABLE IF NOT EXISTS review_sessions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    course TEXT NOT NULL,
    mode TEXT NOT NULL,
    total INTEGER NOT NULL,
    correct INTEGER NOT NULL,
    duration_seconds INTEGER,
    started_at TEXT NOT NULL,
    finished_at TEXT
);
```

### SM-2 Algorithm

Simplified SM-2: correct doubles interval, wrong resets to 1. Ease factor adjusts (min 1.3, default 2.5).

### Key Functions

| Function | Purpose |
|----------|---------|
| `ensure_tables(db_path)` | Create tables if not exist |
| `record_card_review(course, card_type, card_hash, correct, response_time_ms)` | Record + update SM-2 state |
| `record_session(course, mode, total, correct, duration_seconds)` | Record session summary |
| `get_course_stats(course)` | Total reviews, unique cards, due today |
| `get_due_cards(course)` | Cards due for review (SM-2 schedule) |
| `get_wrong_hashes(course)` | Hashes of incorrectly answered cards |

### Ownership Boundary (Critical)

- `card_reviews` + `review_sessions` -- owned by **studyctl** (`review_db.ensure_tables()`)
- `sessions` + `messages` + 10 other tables -- owned by **agent-session-tools** (`migrations.py` v1-v12)
- Both share the same `sessions.db` file but have separate migration domains
- Do NOT mix migration systems

---

## 5. MCP Tools / Agent Infrastructure

### agents/ Directory

**Path**: `/Users/ataylor/code/personal/tools/Socratic-Study-Mentor/agents/`

Contains agent system prompts (markdown files), NOT MCP server code:
- `agents/study-mentor.md` -- Socratic study mentor persona
- `agents/CLAUDE.md` -- Project-level Claude instructions
- `scripts/study-statusline.sh` -- Claude Code status bar script

### agent-session-tools Package

**Path**: `/Users/ataylor/code/personal/tools/Socratic-Study-Mentor/packages/agent-session-tools/`

Source files:
```
src/agent_session_tools/
  __init__.py
  cli.py
  config_loader.py
  db.py
  exporters/           -- Session exporters (claude, cursor, etc.)
  maintenance.py
  migrations.py
  query_sessions/      -- Query CLI (cli.py + resolver.py)
  schema.py
  schema.sql
  session_sync.py
  speak.py             -- TTS via kokoro
  tutor_checkpoint.py
```

### No MCP Server Currently

There is **no `mcp_server.py`** in either package. The project does not currently expose MCP tools. This is a greenfield opportunity for the planned agent-generated flashcards/quizzes feature.

### MCP Integration Points for New Features

1. Create `packages/agent-session-tools/src/agent_session_tools/mcp_server.py`
2. Expose tools: `generate_flashcards(topic, content)`, `generate_quiz(topic, content)`, `record_study_progress(topic, score)`
3. Register in pyproject.toml: `[project.entry-points."mcp.tools"]`
4. Or alternatively, add MCP tools in a new `packages/studyctl/src/studyctl/mcp/` module

---

## 6. pdf-by-chapters Source Code Structure

**Repo**: `/Users/ataylor/code/personal/tools/notebooklm_pdf_by_chapters/`

### Package Structure

```
src/pdf_by_chapters/
  __init__.py
  cli.py                 -- 1300+ lines, Typer CLI
  splitter.py            -- PDF chapter splitting via PyMuPDF
  notebooklm.py          -- NotebookLM API (upload, generate, download, delete)
  syllabus.py            -- LLM syllabus generation, episode tracking
  markdown_converter.py  -- Obsidian markdown -> PDF via pandoc + mermaid
  review.py              -- Interactive flashcard/quiz review (terminal)
  models.py              -- Shared dataclasses
```

### cli.py -- Typer Commands (with line numbers)

| Line | Command | Help Panel | Purpose |
|------|---------|------------|---------|
| L68 | `split` | (default) | Split PDF by TOC bookmarks |
| L80 | `process` | (default) | Split + upload (autopilot) |
| L144 | `list` | (default) | List notebook sources |
| L178 | `generate` | (default) | Generate audio/video for chapters |
| L219 | `download` | (default) | Download generated artifacts |
| -- | `syllabus` | Syllabus | Generate episode syllabus |
| -- | `generate-next` | Syllabus | Generate next syllabus episode |
| -- | `status` | Syllabus | Check generation status |
| -- | `from-obsidian` | Obsidian | Full pipeline: md->pdf->upload->generate |
| -- | `review` | Review | Interactive flashcard/quiz review |

### Key Module Details

#### splitter.py
- `split_by_toc(source: Path, output_dir: Path, level: int)` -- Uses PyMuPDF `get_toc()` for bookmark extraction
- Sanitizes filenames (lowercase, underscores, max 80 chars)
- Handles single files or directories of PDFs

#### notebooklm.py
- Async API integration with `notebooklm-py` library
- Functions: upload, generate (audio/video), download, delete, list sources
- Timeout handling (900s+ for slides/video generation)
- Rate limiting (30s gap between artifact types)

#### syllabus.py
- LLM-driven syllabus generation with state machine
- Episode tracking, status polling
- Chunked generation support

#### markdown_converter.py
- `obsidian_to_pdf(source_dir, output_dir, subdir)` -- Obsidian .md files to PDF
- Uses pandoc with mermaid-filter and typst backend
- Handles Mermaid diagrams, code blocks, images

#### review.py
- Terminal-based flashcard and quiz review
- Loads same JSON format as studyctl's `review_loader.py`
- Separate implementation (no cross-package imports)
- This code is effectively DUPLICATED in studyctl already

#### models.py
```python
@dataclass
class UploadResult:
    id: str; title: str; chapters: int

@dataclass
class NotebookInfo:
    id: str; title: str; sources_count: int

@dataclass
class SourceInfo:
    id: str; title: str
```

### Dependencies (from pyproject.toml)

```
typer>=0.12
pymupdf>=1.25
httpx
rich
notebooklm-py>=0.3.4
anthropic        # For syllabus generation
pandoc           # System dependency for markdown conversion
```

### CLI Framework Mismatch

pdf-by-chapters uses **Typer** while studyctl uses **Click**. Since Typer is built on Click, a Typer app can be mounted as a Click group. However, the cleanest approach is to rewrite the commands as Click commands during absorption, extracting the business logic into separate modules.

---

## 7. Test Structure

### Socratic-Study-Mentor Tests

```
packages/studyctl/tests/
  test_calendar.py
  test_cli.py
  test_review_db.py
  test_review_loader.py

packages/agent-session-tools/tests/
  conftest.py              -- temp_db, migrated_db fixtures
  test_checkpoint.py
  test_config_loader.py
  test_db.py
  test_exporters.py
  test_maintenance.py
  test_migrations.py
  test_query_sessions.py
  test_schema.py
  test_session_sync.py
  test_speak.py
```

Test suite status (2026-03-14): **696 passed, 0 failures, 8 skipped**

Key patterns:
- `pytest` with `uv sync --all-packages` required
- `conftest.py` provides `temp_db` (base) and `migrated_db` (all migrations) fixtures
- `pytest.importorskip()` for optional dependencies
- Pre-push hook runs pytest; pre-commit runs ruff

### notebooklm-pdf-by-chapters Tests

```
tests/
  conftest.py
  unit/
    test_cli.py
    test_notebooklm.py
    test_splitter.py
    test_syllabus.py
  integration/
    test_split_roundtrip.py
```

Pattern: unit tests mock external APIs (NotebookLM, Anthropic), integration tests use real PDFs.

---

## 8. Project Conventions

### No CLAUDE.md at Project Root

Neither repo has a CLAUDE.md file. The agents/ directory contains agent persona prompts but no project-level build/test instructions.

### Code Quality

- **Linter/formatter**: ruff (configured in pyproject.toml)
- **Type hints**: Required on all functions
- **Package manager**: uv (never pip, never poetry)
- **Git**: Never auto-commit

### Pyproject.toml (Root Workspace)

**File**: `/Users/ataylor/code/personal/tools/Socratic-Study-Mentor/pyproject.toml`

```toml
[tool.uv.workspace]
members = ["packages/*"]
```

Root has `py-modules = []` (not installable itself). Each package is independently installable.

### Studyctl Dependencies (current)

```toml
dependencies = [
    "click>=8.1",
    "rich>=13.0",
    "pyyaml>=6.0",
    "icalendar>=5.0",
]
```

### Contributing Pattern

New session exporters follow protocol pattern in `exporters/base.py`. New CLI commands follow Click group pattern. New features get brainstorm docs in `docs/brainstorms/`.

---

## Key Integration Decisions & Risks

### 1. Click vs Typer
pdf-by-chapters uses Typer; studyctl uses Click. Options:
- **(a)** Rewrite pdf-by-chapters commands as Click (cleanest, consistent)
- **(b)** Mount Typer app as Click subgroup via `typer.main.get_command()` (quick but messy)
- **Recommendation**: Option (a) -- extract business logic from cli.py into modules, write thin Click wrappers

### 2. Review Code Duplication
`pdf_by_chapters/review.py` and `studyctl/review_loader.py` both parse the same JSON format. After absorption, delete `review.py` and use `review_loader.py` as the single implementation.

### 3. Dependency Additions for FastAPI
New deps needed: `fastapi`, `uvicorn[standard]`, `jinja2`, `python-multipart` (for form handling), `htmx` (client-side only, no Python dep). These are significant additions to a currently-lightweight package.

### 4. Config Extension
The `Settings` dataclass is simple to extend. Add a `ContentConfig` sub-dataclass. The `review.directories` config already supports multiple paths, so content pipeline output dirs just need to be added there.

### 5. Database Ownership
Two separate migration domains share sessions.db. New tables for content pipeline should follow studyctl's pattern (`ensure_tables()` with column checks), NOT agent-session-tools' migration versioning.

### 6. MCP Tools -- Greenfield
No existing MCP infrastructure. The `mcp` Python package provides `@tool` decorators. Consider whether MCP tools belong in agent-session-tools (session-focused) or studyctl (content-focused), or a new package.

### 7. File Sizes
`cli.py` in studyctl is already 1272 lines. The content commands should live in a separate module (`content_cli.py` or `content/cli.py`). Similarly, pdf-by-chapters' cli.py is 1300+ lines and should be decomposed during absorption.
