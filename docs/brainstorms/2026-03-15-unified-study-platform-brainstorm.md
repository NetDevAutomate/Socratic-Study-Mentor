# Brainstorm: Unified Study Platform

**Date:** 2026-03-15
**Status:** Final
**Participants:** ataylor, Claude
**Supersedes:** 2026-03-15-native-app-repackaging-brainstorm.md (native Swift -- deferred to Phase 2)

---

## What We're Building

A unified, polished study platform that merges Socratic-Study-Mentor and notebooklm-pdf-by-chapters into a single installable tool. The platform provides:

1. **AI coding assistants as the study brain** -- Claude Code, Gemini CLI, Kiro CLI, etc. with Socratic mentor agent definitions
2. **Web UI as the study review hub** -- flashcards, quizzes, NotebookLM artefacts (audio, video, slides, infographics), chunked PDFs, progress tracking. Accessible from any device on LAN.
3. **TUI as the terminal study interface** -- same review features for terminal users, with Pomodoro timer, voice toggle, OpenDyslexic font
4. **Complete content pipeline** -- PDF ebook splitting, NotebookLM upload/generation, Obsidian markdown conversion, artefact download/management

### Core Value Proposition

**The only AuDHD-aware study tool on the market** that combines AI Socratic mentoring with spaced repetition, energy-level adaptation, and a complete content-to-study pipeline. Nothing like this exists.

---

## Why This Approach (Not Native App)

The native Swift app direction (documented in `2026-03-15-native-app-repackaging-brainstorm.md` with 6 research docs in `docs/research/`) was explored thoroughly. Decision to defer it because:

1. **Existing codebase is substantial** -- 696+ tests, working PWA, working TUI, working agent definitions, working session tooling. Rewriting takes 3-6 months to reach parity.
2. **Real users giving feedback NOW** is more valuable than a perfect native app later. Ship weeks, not months.
3. **The problems are solvable without a rewrite** -- fragmentation, install difficulty, and interface confusion are packaging/UX problems, not technology problems.
4. **PWA already works on iOS Safari** -- not App Store polished, but usable today.
5. **Research is preserved** for when user feedback and usage data justify the native investment.

---

## Problem Statement

1. **Installation barrier**: Non-technical users can't `git clone`, install `uv`, or configure YAML
2. **Fragmentation**: Two separate repos with no clear journey from "I have an ebook" to "I'm studying"
3. **Interface confusion**: TUI vs PWA vs CLI vs Agent -- no guidance on which to use when
4. **Incomplete pipeline**: Content generation (pdf-by-chapters) and study consumption (studyctl) are disconnected

---

## Key Decisions

### 1. Absorb pdf-by-chapters into studyctl

Port ALL 12 CLI commands directly into studyctl as new commands. Kill the separate repo. Full feature parity -- the syllabus autopilot is a key differentiator for ebook-based study.

**What gets absorbed (full scope):**

| pdf-by-chapters capability | New studyctl location |
|---|---|
| PDF splitting (PyMuPDF, TOC bookmarks) | `studyctl content split <pdf>` |
| NotebookLM upload/notebook creation | `studyctl content upload <pdf>` |
| Audio/video generation + polling | `studyctl content generate` |
| Artefact download | `studyctl content download` |
| Notebook listing | `studyctl content list` |
| Notebook deletion | `studyctl content delete` |
| Syllabus generation (LLM-driven) | `studyctl content syllabus` |
| Generate next episode (autopilot) | `studyctl content autopilot` |
| Generation status/polling | `studyctl content status` |
| Obsidian markdown -> PDF conversion | `studyctl content from-obsidian <vault-path>` |
| Process (split + upload combined) | `studyctl content process <pdf>` |
| Interactive terminal review | Already exists in studyctl (TUI/PWA) |

**New `studyctl content` command group** unifies the entire pipeline under one CLI.

**Config unification**: Single `~/.config/studyctl/config.yaml` with a new `content` section:
```yaml
content:
  base_path: ~/study-materials       # Single download base path for everything
  notebooklm:
    default_types: [audio]           # What to generate by default
    timeout: 900                     # Generation timeout (seconds)
    inter_episode_gap: 30            # Rate limit gap
```

### 2. FastAPI backend for the web UI

Replace the stdlib HTTP server with FastAPI. Single process serves:
- Static PWA files (HTML/JS/CSS)
- REST API for all study operations
- WebSocket for live features (Pomodoro timer sync, voice control)
- File serving for NotebookLM artefacts (audio, video, slides, infographics, PDFs)

**Key API endpoints:**

| Endpoint | Purpose |
|---|---|
| `GET /api/cards` | Flashcards with filtering (source, chapter, due date) |
| `GET /api/quizzes` | Quiz questions with filtering |
| `POST /api/review` | Submit card grade (SM-2 update) |
| `GET /api/artefacts` | List NotebookLM artefacts by course/chapter |
| `GET /api/artefacts/{id}/stream` | Stream audio/video file |
| `GET /api/sessions` | Session history, streaks, progress |
| `GET /api/content/status` | Content pipeline status (generation progress) |
| `POST /api/content/split` | Trigger PDF split |
| `POST /api/content/generate` | Trigger NotebookLM generation |
| `GET /api/config` | Current config (safe fields only) |
| `PUT /api/config` | Update config from UI |
| `POST /api/feedback` | Submit feedback -> GitHub Issues API |

### 3. Vanilla HTML/JS + HTMX frontend

No build step, no node_modules. FastAPI serves Jinja2 templates. HTMX handles dynamic page updates. Alpine.js for interactive widgets (card flip, timer, audio controls).

**Artefact display** uses native browser capabilities:
- Audio (.mp3): `<audio controls>`
- Video (.mp4): `<video controls>`
- Slides/PDFs: `<embed>` or pdf.js
- Infographics: `<img>`

### 4. Architecture: AI coding assistants remain the AI brain

**No LLM API calls in the web UI.** The AI experience happens through the user's chosen coding assistant (Claude Code, Gemini CLI, Kiro CLI, etc.) with the Socratic mentor agent definition.

**How they connect:**
- Agent CLI connects to studyctl via MCP (already working today)
- MCP tools expose study context: current card, progress, energy level, session history
- Agent CLI provides Socratic questioning, adaptive mentoring, content generation guidance
- Web UI and TUI handle the non-AI study experience (review, progress, artefacts)

**For users without a coding assistant:** The web UI still works fully for flashcard/quiz review, artefact viewing, and progress tracking. The AI mentoring is an optional enhancement, not a requirement.

### 5. LAN access with optional password protection

`studyctl web` serves on `0.0.0.0:8567` (LAN accessible).

**Optional authentication:**
- `studyctl web --password` prompts for a password at startup
- Simple HTTP Basic Auth or session-based auth
- Stored as bcrypt hash in config, never plaintext
- Config option: `web.require_auth: true` + `web.password_hash: <bcrypt>`

### 6. Feedback via GitHub Issues API

In-app "Submit Bug" and "Submit Feedback" buttons that:
- Collect: description, category (bug/feature/UX), optional screenshot, app version, OS
- POST to GitHub Issues API on the repo
- Pre-fill labels (bug, feedback, from-app)
- Requires GitHub authentication (OAuth token stored in config, or opens browser for auth)

### 7. Onboarding agent (AI-guided setup)

The AI coding assistant that the user will study with ALSO helps them set up. An agent skill/command (`/study-setup` or similar) provides a conversational setup experience:

1. Asks what the user wants to learn (courses, subjects, goals)
2. Asks what they already know (to build knowledge bridges -- existing `bridges` feature)
3. Asks where their study materials are (ebooks, Obsidian vaults, downloaded courses)
4. Walks through NotebookLM setup if they want audio/video artefacts (optional)
5. Creates and maintains `~/.config/studyctl/config.yaml` based on the conversation
6. Generates initial flashcards/quizzes from the first PDF chapter as a demo
7. Verifies everything works (`studyctl content split`, `studyctl web`)

This is an **agent definition + MCP tools**, not application code. The agent uses `studyctl` CLI commands as tools. Available for Claude Code, Gemini CLI, Kiro CLI, etc.

**For non-technical users:**
```
brew install studyctl
claude /study-setup     # AI walks through everything conversationally
```

**For technical users:**
- `uv tool install studyctl` (PyPI)
- Homebrew formula: `brew install studyctl`
- Config: `studyctl config init` (existing interactive CLI)

**Fallback (no AI coding assistant):**
- `studyctl setup` -- interactive CLI wizard (non-AI, asks the same questions via prompts)
- "Getting Started" page in the web UI (first-run experience)

### 8. Three-tier content generation (NotebookLM optional)

NotebookLM is an optional content source, not a dependency. Flashcards and quizzes -- the core study artefacts -- are generated by the AI coding assistant directly.

| Tier | What generates | How | Cost |
|---|---|---|---|
| **Agent-generated (core)** | Flashcard JSON, quiz JSON | Agent skill sends chapter text to LLM, receives structured JSON | Free (uses agent subscription) |
| **Local TTS (built-in)** | Study audio (.mp3) | Agent writes study summary, kokoro-onnx converts to speech | Free (local) |
| **NotebookLM (optional)** | Podcast audio, video, slides, infographics | `studyctl content generate` via notebooklm-py | Free (Google account), unofficial API |

**Agent-generated flashcards/quizzes** are better than NotebookLM for this because:
- No unofficial API dependency
- Customisable: Socratic style, AuDHD-aware difficulty, topic bridging
- Works offline with local LLMs
- Uses the same subscription the user already pays for

**notebooklm-py** remains as an optional dependency (`studyctl[notebooklm]`). Not forked -- upstream maintained by Stefano Amorelli. Accreditation in README and pyproject.toml.

### 9. Course-centric artefact storage

All artefacts stored under `content.base_path` in a course-centric structure:

```
~/study-materials/
  python-crash-course/
    chapters/
      chapter_01_getting_started.pdf
      chapter_02_variables.pdf
    audio/
      ep01_intro.mp3
      ep02_variables.mp3
    flashcards/
      ch01-flashcards.json
      ch02-flashcards.json
    quizzes/
      ch01-quiz.json
      ch02-quiz.json
    video/
      ep01_intro.mp4
    slides/
      ep01_intro.pdf
    metadata.json  (notebook ID, syllabus state, generation history)
  aws-saa-c03/
    ...
```

Each course is self-contained and browsable. The web UI and TUI read this structure. `metadata.json` tracks NotebookLM notebook IDs, syllabus state, and generation history per course.

### 10. Package name: `studyctl`

PyPI and Homebrew package name: **`studyctl`**. Short, memorable, matches the CLI command.
- `brew install studyctl`
- `uv tool install studyctl`
- `pip install studyctl`

### 11. TUI enhancements

Keep the existing Textual TUI, add:
- Pomodoro timer (countdown, audio chime, notification)
- Voice toggle (on/off, speak current card/hint/answer)
- OpenDyslexic font toggle
- Artefact browser (list and play audio, open PDFs)
- tmux-friendly: designed to run alongside an agent CLI in a split pane

---

## Architecture Overview

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
|  |            |  |           |  |                      | |
|  | - split    |  | - SM-2    |  | - Flashcards/quizzes | |
|  | - upload   |  | - SQLite  |  | - Artefact viewer    | |
|  | - generate |  | - history |  | - Progress dashboard | |
|  | - download |  | - streaks |  | - Config editor      | |
|  | - obsidian |  | - wins    |  | - Feedback           | |
|  | - syllabus |  |           |  | - LAN + auth         | |
|  +------------+  +-----------+  +----------------------+ |
|                                                          |
|  +------------+  +-----------+  +----------------------+ |
|  | Session    |  | TUI       |  | Agent Definitions    | |
|  | Tools      |  | (Textual) |  | (Socratic Mentor)    | |
|  |            |  |           |  |                      | |
|  | - export   |  | - cards   |  | - Claude Code        | |
|  | - query    |  | - timer   |  | - Gemini CLI         | |
|  | - sync     |  | - voice   |  | - Kiro CLI           | |
|  | - maint    |  | - artefacts| | - OpenCode          | |
|  +------------+  +-----------+  +----------------------+ |
+----------------------------------------------------------+
```

### User Journey: "I have an ebook, I want to study"

**Path A: With AI coding assistant (recommended)**
```
1. Install:     brew install studyctl

2. Setup:       claude /study-setup
                  Agent asks: What do you want to learn? Where's your PDF?
                  What do you already know? Want NotebookLM audio?
                  -> Creates config, splits PDF, generates first flashcards

3. Study:       studyctl web    (opens browser -> flashcards, quizzes, audio)
                   OR
                studyctl tui    (terminal dashboard)

4. AI Mentor:   claude          (Socratic mentor via MCP -- adapts to energy/mood)
                   OR
                gemini / kiro   (same agent definition, different CLI)

5. Generate:    Agent generates flashcards/quizzes from next chapters on demand
                   OR
                studyctl content autopilot  (NotebookLM audio/video, optional)

6. Track:       Web UI dashboard / studyctl progress / streaks / wins

7. Feedback:    "Submit Feedback" button in web UI -> GitHub Issue
```

**Path B: Without AI coding assistant**
```
1. Install:     brew install studyctl
2. Setup:       studyctl setup  (interactive CLI wizard)
3. Import:      studyctl content split "Python_Crash_Course.pdf"
4. Generate:    studyctl content autopilot --download  (NotebookLM, optional)
5. Study:       studyctl web / studyctl tui
6. Track:       Web UI dashboard / CLI commands
```

---

## What We're NOT Building (YAGNI)

- Native iOS/macOS/Android app (deferred -- research preserved in docs/research/)
- LLM API calls embedded in the web UI or backend (agent CLIs handle all AI interaction)
- User accounts / cloud backend / server-side authentication
- Collaborative/social features
- Custom LLM fine-tuning
- Forked/maintained copy of notebooklm-py (use upstream as optional dependency)
- Real-time AI chat in the web UI (the AI lives in the terminal agent)
- Voice input in the app (recommend system-level tools: Whispr Flow, macOS Dictation, Warp voice)

---

## Resolved Questions

All original open questions have been resolved:

| # | Question | Resolution |
|---|---|---|
| 1 | PyPI package name | **`studyctl`** -- short, matches CLI command (Decision 10) |
| 2 | NotebookLM auth | **Optional, not core.** Flashcards/quizzes generated by agent. NotebookLM only for audio/video. Cookie fragility accepted and documented. (Decision 8) |
| 3 | Artefact storage | **Course-centric** under `content.base_path`. Each course self-contained. (Decision 9) |
| 4 | Absorption scope | **Full -- all 12 commands** ported. Syllabus autopilot is a differentiator. (Decision 1) |

---

## Next Steps

1. **Run `/ce:plan`** to create the implementation plan (all questions resolved)
2. **Phase 1**: Absorb pdf-by-chapters into studyctl, course-centric storage, unified config
3. **Phase 2**: FastAPI backend + HTMX web UI (artefact viewer, progress dashboard, config editor)
4. **Phase 3**: Agent-generated flashcards/quizzes skill, onboarding agent (`/study-setup`)
5. **Phase 4**: Installation improvements (PyPI as `studyctl`, Homebrew formula, `studyctl setup` wizard)
6. **Phase 5**: TUI enhancements (Pomodoro, voice, artefacts, tmux integration)
7. **Phase 6**: Feedback mechanism (GitHub Issues API, in-app buttons)
8. **Phase 7**: LAN auth, polish, documentation for non-technical users
9. **Future**: Native macOS/iOS app (research preserved in docs/research/)

## Research Documents (preserved for future native app phase)

| Topic | Path |
|---|---|
| MCP in native apps | `docs/research/mcp-native-app-research.md` |
| ACP maturity | `docs/research/2026-03-15-acp-research.md` |
| Swift PoC feasibility | `docs/research/swift-poc-feasibility.md` |
| Monetisation models | `docs/research/monetisation-research.md` |
| Socratic Study Mentor repo analysis | `docs/research/repo-research-summary.md` |
| pdf-by-chapters repo analysis | `docs/research/notebooklm-pdf-by-chapters-analysis.md` |
