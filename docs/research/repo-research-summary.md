# Socratic Study Mentor -- Repository Research Summary

Generated: 2026-03-15

---

## Architecture & Structure

**Monorepo** using a `uv` workspace (`[tool.uv.workspace] members = ["packages/*"]`). The root `pyproject.toml` is not installable itself (`py-modules = []`) -- it holds shared dev tooling config (ruff, pyright, pytest).

### Two workspace packages

| Package | Version | Build Backend | Entry Point | Purpose |
|---------|---------|---------------|-------------|---------|
| `studyctl` | 1.0.0 | hatchling | `studyctl` (Click) | Study pipeline: sync, review, scheduling, web/TUI |
| `agent-session-tools` | 2.0.0 | hatchling | 6 CLI commands (Typer) | AI session export, query, sync, maintenance, TTS |

### Directory layout (key paths)

```
/
  pyproject.toml                          # Workspace root
  CONTRIBUTING.md
  scripts/
    install.sh                            # Full installer (interactive + flags)
    install-agents.sh                     # Agent-only installer
  packages/
    studyctl/
      pyproject.toml
      src/studyctl/
        cli.py                            # Click CLI -- all studyctl commands
        config.py, settings.py            # Topic config + YAML loader
        sync.py, state.py                 # NotebookLM sync + state tracking
        history.py                        # Session history + spaced repetition
        scheduler.py                      # launchd/cron scheduling
        shared.py                         # Cross-machine sync
        maintenance.py                    # Notebook deduplication
        calendar.py                       # .ics time-block generation
        review_db.py, review_loader.py    # Flashcard/quiz data
        pdf.py                            # Markdown to PDF export
        web/
          server.py                       # Built-in HTTP server (stdlib)
          static/
            index.html, app.js, style.css # PWA frontend
            manifest.json, sw.js          # Service worker for offline
        tui/
          app.py                          # Textual TUI dashboard
          study_cards.py                  # Card review widget
    agent-session-tools/
      pyproject.toml
      src/agent_session_tools/
        export_sessions.py                # session-export CLI
        query_sessions/                   # session-query CLI (package)
        sync.py                           # session-sync CLI
        maintenance.py                    # session-maint CLI
        tutor_checkpoint.py               # tutor-checkpoint CLI
        speak.py                          # study-speak CLI (TTS)
        mcp_speak.py                      # MCP server for voice
        exporters/                        # 8 source exporters
          base.py, claude.py, kiro.py, gemini.py,
          aider.py, opencode.py, litellm.py,
          repoprompt.py, bedrock.py
        embeddings.py                     # Vector embeddings
        semantic_search.py                # Hybrid FTS+vector
        classifier.py                     # Session classification
        deduplication.py                  # Duplicate detection
        migrations.py                     # Schema migrations
        config_loader.py                  # Shared config (~/.config/studyctl/config.yaml)
        integrations/
          git.py, vscode.py
  agents/
    claude/
      socratic-mentor.md                  # Claude Code agent definition
      mentor-reviewer.yaml
    kiro/
      study-mentor.json                   # Kiro CLI agent
      study-mentor/, skills/
  docs/
    setup-guide.md                        # Comprehensive installation + config guide
    agent-install.md                      # AI agent setup
    audhd-learning-philosophy.md          # Design philosophy
    voice-output.md                       # TTS guide
    roadmap.md                            # v1.0 through v1.5
    cli-reference.md, session-protocol.md, concept-graph.md, etc.
```

---

## All User-Facing Interfaces

### 1. `studyctl` CLI (Click-based)

Single entry point: `studyctl`. Key command groups/commands:

| Command | Purpose |
|---------|---------|
| `studyctl config init` | Interactive config setup (creates ~/.config/studyctl/config.yaml) |
| `studyctl config show` | Display current config |
| `studyctl review` | Show spaced repetition schedule (what is due) |
| `studyctl schedule-blocks` | Generate .ics calendar time blocks for due reviews |
| `studyctl wins` | Show mastered concepts (fight imposter syndrome) |
| `studyctl resume` | Auto-resume last session context |
| `studyctl streaks` | Show study streak stats |
| `studyctl progress` | Progress map across topics |
| `studyctl teachback` | Record/view teach-back scores |
| `studyctl bridges` | Knowledge bridges (network -> DE analogies) |
| `studyctl schedule install/remove/list/add/delete` | Manage launchd/cron jobs |
| `studyctl state push/pull/status` | Cross-machine state sync |
| `studyctl sync` | Sync Obsidian notes to NotebookLM |
| `studyctl web` | Launch Web PWA (port 8567, LAN accessible) |
| `studyctl tui` | Launch Textual TUI dashboard |
| `studyctl docs` | Serve MkDocs documentation site |
| `studyctl pdf` | Export markdown to PDF |

### 2. Web PWA (recommended for multi-device study)

- Launched via `studyctl web` -- no extra deps needed
- Built-in Python HTTP server serving static HTML/JS/CSS
- PWA installable (manifest.json + service worker for offline)
- Features: flashcard/quiz review with SM-2 spaced repetition, source/chapter filtering, card count limiter, session history with 90-day heatmap, Pomodoro timer, Web Speech API voice output, OpenDyslexic font toggle, dark/light theme, keyboard shortcuts

### 3. TUI Dashboard (terminal)

- Launched via `studyctl tui` -- requires `studyctl[tui]` extra (Textual)
- Tabs: Dashboard, Review, Concepts, Sessions, StudyCards
- Keyboard-driven: f=flashcards, z=quiz, space=flip, y/n=grade, v=voice, o=OpenDyslexic

### 4. agent-session-tools CLI Commands

| Command | Entry Point | Purpose |
|---------|-------------|---------|
| `session-export` | `export_sessions:main` | Export sessions from AI tools to SQLite |
| `session-query` | `query_sessions:main` | Search/query session database (FTS5 + semantic) |
| `session-maint` | `maintenance:main` | DB maintenance (vacuum, stats, cleanup) |
| `session-sync` | `sync:main` | Cross-machine DB sync via SSH |
| `tutor-checkpoint` | `tutor_checkpoint:main` | Record study progress checkpoints |
| `study-speak` | `speak:main` | TTS voice output (kokoro-onnx) |

### 5. AI Agent Definitions

- **Claude Code**: `agents/claude/socratic-mentor.md` -- Socratic mentoring with AuDHD-aware methodology
- **Kiro CLI**: `agents/kiro/study-mentor.json` + skills
- **Gemini CLI / OpenCode / Amp**: Shared agent framework installed by `install-agents.sh`

---

## Installation

### Automatic (recommended)

```bash
git clone https://github.com/NetDevAutomate/Socratic-Study-Mentor.git
cd Socratic-Study-Mentor
./scripts/install.sh
```

### install.sh behavior

The install script supports flags:
- `--non-interactive` -- No prompts (for Ansible/CI)
- `--tools-only` -- Just install CLI tools globally via `uv tool install`
- `--agents-only` -- Just install agent definitions

What it does:
1. Checks prerequisites: Python >= 3.10, uv
2. Installs packages via `uv tool install` (global CLI availability)
3. Installs agent definitions for detected AI tools (Claude Code, Kiro, Gemini, etc.)
4. Creates skeleton `~/.config/studyctl/config.yaml` if missing
5. Optionally downloads TTS voice model (~85MB kokoro-onnx)
6. Prints summary of installed tools and next steps

### install-agents.sh

Standalone agent installer with per-tool flags:
- `--kiro`, `--claude`, `--gemini`, `--opencode`, `--amp`
- `--uninstall` to remove agent definitions
- Creates symlinks from AI tool config dirs to repo agent definitions
- Installs shared skills and Claude Code status line integration

### Development setup

```bash
uv sync --all-packages --extra dev --extra test
uv run pre-commit install
```

Critical: `uv sync --all-packages` is required (not just `uv sync`) to install workspace members.

### Optional extras

| Extra | Package | What it adds |
|-------|---------|-------------|
| `studyctl[notebooklm]` | notebooklm-py | NotebookLM sync |
| `studyctl[tui]` | textual>=0.80 | TUI dashboard |
| `agent-session-tools[semantic]` | sentence-transformers | Vector search |
| `agent-session-tools[tokens]` | tiktoken | Token counting |
| `agent-session-tools[tts]` | kokoro-onnx | Voice output |
| `agent-session-tools[all]` | Combined | All optional features |

---

## User Journey

### From "I have course materials" to "I'm studying"

**Step 1: Install**
```bash
git clone ... && cd Socratic-Study-Mentor && ./scripts/install.sh
```

**Step 2: Configure**
```bash
studyctl config init    # Interactive -- sets up topics, directories, hosts
```
This creates `~/.config/studyctl/config.yaml` with:
- `review.directories` -- paths to course material directories containing markdown quiz/flashcard files
- `topics` -- named study topics with schedules
- `hosts` -- machines for cross-device sync (optional)
- `tui` -- theme and accessibility preferences

**Step 3: Populate session database** (if using AI session tracking)
```bash
session-export          # Exports sessions from Claude Code, Kiro, etc. into SQLite
```

**Step 4: Study -- choose your interface**

Option A -- Web PWA (recommended):
```bash
studyctl web            # Opens browser, accessible from phone/tablet too
```

Option B -- TUI:
```bash
studyctl tui            # Terminal dashboard with keyboard shortcuts
```

Option C -- CLI review:
```bash
studyctl review         # See what is due for spaced repetition
studyctl schedule-blocks # Generate calendar time blocks
```

Option D -- AI mentor session:
- Start a Claude Code / Kiro / Gemini session with the Socratic mentor agent
- Agent asks energy level, emotional state, sets session parameters
- Socratic questioning methodology (70% questions, 30% info drops)
- End-of-session: auto-records progress, suggests next review

**Step 5: Track progress**
```bash
studyctl wins           # See mastered concepts
studyctl streaks        # Study streak stats
studyctl progress       # Progress map
session-query <term>    # Search across all AI sessions
```

**Step 6: Sync across machines** (optional)
```bash
studyctl state push macmini       # Push study state
session-sync sync macbookpro      # Sync session database
```

### Content format

The review system reads markdown files from configured directories. Course materials are expected to be in a specific directory structure (e.g., `~/Desktop/ZTM-DE/downloads/`). The Web PWA and TUI both parse these for flashcard and quiz content, with source/chapter filtering.

---

## Existing Documentation

| Document | Path | Content |
|----------|------|---------|
| README.md | `/README.md` | Full overview, features, quick start, CLI reference |
| Setup Guide | `/docs/setup-guide.md` | Installation, config, Obsidian, NotebookLM, sync, WSL2, troubleshooting |
| Agent Install | `/docs/agent-install.md` | Per-tool AI agent setup |
| AuDHD Philosophy | `/docs/audhd-learning-philosophy.md` | Design rationale |
| Voice Output | `/docs/voice-output.md` | TTS configuration |
| Roadmap | `/docs/roadmap.md` | v1.0 (current) through v1.5 plans |
| Contributing | `/CONTRIBUTING.md` | Dev setup, code style, project structure, how-to guides |
| CLI Reference | `/docs/cli-reference.md` | Detailed command docs |
| Session Protocol | `/docs/session-protocol.md` | Agent session workflow |
| Concept Graph | `/docs/concept-graph.md` | Knowledge graph docs |
| MCP Integrations | `/agents/mcp/README.md` | Calendar/reminder MCP setup |

---

## Key Observations

1. **Mature project**: v1.0 with comprehensive docs, 696+ tests, CI, pre-commit hooks, multiple AI agent integrations.
2. **AuDHD-first design**: Every feature is designed around neurodivergent learning patterns -- energy adaptation, emotional regulation, sensory support, dopamine maintenance.
3. **Three study interfaces**: Web PWA (primary, cross-device), TUI (terminal), AI agents (Socratic methodology). All share the same config and data.
4. **Global CLI install**: Uses `uv tool install` for system-wide availability rather than venv activation.
5. **Cross-machine sync**: SSH-based state and DB sync built in, with hostname auto-detection.
6. **Roadmap**: v1.1 (AuDHD intelligence) mostly complete, v1.2 (community/polish) and v1.3-v1.5 planned. PyPI publishing not yet done.
