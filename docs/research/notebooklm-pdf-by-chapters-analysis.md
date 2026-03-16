# Repository Research: notebooklm-pdf-by-chapters

**Date:** 2026-03-15
**Path:** `/Users/ataylor/code/personal/tools/notebooklm_pdf_by_chapters`

---

## 1. What It Does

Splits ebook PDFs by chapter using TOC bookmarks (PyMuPDF), uploads chapter PDFs to Google NotebookLM, and generates AI audio/video overviews per chapter range. Also supports an Obsidian markdown pipeline that converts `.md` notes to PDFs, uploads them, and generates flashcards/quizzes/audio. Includes an interactive terminal review mode for studying.

## 2. Project Structure

```
src/pdf_by_chapters/
  __init__.py              # Package init
  cli.py                   # Typer CLI (~1300 lines) - all commands
  splitter.py              # PDF chapter splitting via PyMuPDF get_toc()
  notebooklm.py            # NotebookLM API integration (upload, generate, download, delete)
  syllabus.py              # LLM-driven syllabus generation, state machine, episode tracking
  models.py                # Shared dataclasses (UploadResult, NotebookInfo, SourceInfo)
  markdown_converter.py    # Obsidian markdown -> PDF via pandoc + mermaid-filter + typst
  review.py                # Interactive flashcard/quiz review (load JSON, present in terminal)

tests/
  conftest.py              # Shared fixtures
  unit/                    # test_cli.py, test_notebooklm.py, test_splitter.py, test_syllabus.py
  integration/             # test_split_roundtrip.py

docs/
  brainstorms/             # from-obsidian brainstorm, chunked generation brainstorm
  plans/                   # Refactor plan, chunked gen plan, generate-all plan, study review plan
  guide-*.md               # User guides (split, upload, generate, study workflow)
  codemap.md               # Module dependency map with mermaid diagrams
  use-cases.md             # UC1-UC10 documented
```

## 3. CLI Commands (Entry Point: `pdf-by-chapters`)

Installed via `[project.scripts]` as `pdf-by-chapters = "pdf_by_chapters.cli:app"` (Typer).

### Core Commands (no panel)
| Command | Purpose |
|---------|---------|
| `split` | Extract chapter PDFs from ebook using TOC bookmarks |
| `process` | Split + upload chapters to NotebookLM (creates notebook) |
| `list` | View notebooks and their sources |
| `generate` | Create audio/video overviews for a chapter range |
| `download` | Fetch generated audio/video artifacts |
| `delete` | Remove a notebook |

### Syllabus Panel
| Command | Purpose |
|---------|---------|
| `syllabus` | LLM-generated podcast plan grouping chapters into episodes |
| `generate-next` | Generate next pending episode (or `--all` for full autopilot) |
| `status` | Check generation progress (`--poll`, `--tail`) |

### Obsidian Panel
| Command | Purpose |
|---------|---------|
| `from-obsidian` | Convert .md files to PDF via pandoc, upload, generate audio/flashcards/quizzes |

### Review Panel
| Command | Purpose |
|---------|---------|
| `review` | Interactive terminal flashcard + quiz review from generated JSON files |

## 4. Installation

- **Requires:** Python 3.11+
- **Install:** `uv tool install .` or `uv tool install git+https://github.com/NetDevAutomate/notebooklm-pdf-by-chapters.git`
- **Build system:** Hatchling (`src/pdf_by_chapters` package)
- **Dependencies:** typer, pymupdf, notebooklm-py, rich
- **Dev deps:** pyright, ruff, pytest, pytest-asyncio, pytest-cov, pytest-mock
- **NotebookLM auth:** `pip install notebooklm-py[browser]` then `notebooklm login` (cookie-based)
- **from-obsidian prerequisites:** pandoc, mmdc (@mermaid-js/mermaid-cli), typst

## 5. User Journeys

### Journey A: PDF Ebook -> Audio Podcast Series (Autopilot)
1. `pdf-by-chapters process "Book.pdf"` -- splits by TOC, uploads chapters to NotebookLM
2. `export NOTEBOOK_ID=<id>`
3. `pdf-by-chapters generate-next -n $NOTEBOOK_ID -o ./chapters --all --download --no-video`
   - Auto-creates syllabus, generates each episode sequentially, downloads audio to `./chapters/downloads/`

### Journey B: PDF Ebook -> Manual Chapter Ranges
1. `pdf-by-chapters split "Book.pdf"` -- local splitting only
2. `pdf-by-chapters process "Book.pdf"` -- split + upload
3. `pdf-by-chapters generate -n $NOTEBOOK_ID -c 1-3` -- generate for chapters 1-3
4. `pdf-by-chapters download -n $NOTEBOOK_ID -o ./overviews -c 1-3`

### Journey C: Obsidian Notes -> Study Materials
1. `pdf-by-chapters from-obsidian ~/vault/course-notes/` -- converts .md to PDF, uploads, generates audio + flashcards + quizzes
2. Output: `downloads/` (audio .mp3), `flashcards/` (*-flashcards.json), `quizzes/` (*-quiz.json)
3. `pdf-by-chapters review ~/vault/course-notes/downloads` -- interactive terminal review

## 6. Output Formats

| Output | Format | Location |
|--------|--------|----------|
| Chapter PDFs | `{book}_chapter_{nn}_{title}.pdf` | `./chapters/` |
| Audio overviews | `.mp3` (NotebookLM deep-dive format) | `./chapters/downloads/` or `./overviews/` |
| Video overviews | `.mp4` (NotebookLM whiteboard style) | `./overviews/` |
| Syllabus state | `syllabus_state.json` (episodes, chunks, artifact tracking) | `./chapters/` |
| Flashcards | `*-flashcards.json` (`{"title", "cards": [{"front", "back"}]}`) | `downloads/flashcards/` |
| Quizzes | `*-quiz.json` (`{"title", "questions": [{"question", "answerOptions"}]}`) | `downloads/quizzes/` |
| Converted PDFs | `nn-slug.pdf` (from markdown) | `pdfs/` subdirectory |

## 7. Integration Points

| Integration | How |
|-------------|-----|
| **notebooklm-py** | Core dependency -- all NotebookLM API calls (create, upload, generate, poll, download) |
| **notebooklm-repo-artefacts** | Separate tool that generated artefacts for THIS repo (audio/video/slides/infographic hosted on artefact-store) |
| **Socratic-Study-Mentor (studyctl)** | Study review plan (2026-03-13) describes adding StudyCards tab to `studyctl tui` that loads the same flashcard/quiz JSON format |
| **Obsidian** | `from-obsidian` command ingests Obsidian vault markdown files |
| **Pandoc + mermaid-cli + typst** | External tools for markdown-to-PDF conversion in from-obsidian pipeline |
| **artefact-store** | GitHub Pages hosting at `artefacts.netdevautomate.dev` for generated overview media |

## 8. Key Architectural Decisions

- **Typer CLI** with `rich_help_panel` grouping commands into Core / Syllabus / Obsidian / Review
- **Async throughout** -- notebooklm.py uses `async with NotebookLMClient.from_storage()`, CLI wraps in `asyncio.run()`
- **Syllabus state machine** -- JSON-persisted state tracking episodes, chunk status (pending/generating/completed/failed), artifact task IDs
- **LLM-driven grouping** -- syllabus prompt sent to NotebookLM chat to group chapters into logical episodes
- **Retry with backoff** -- MAX_RETRIES=3 for generation, 30s inter-episode gap for rate limits
- **Chapter-aware generation** -- selects specific NotebookLM source IDs by chapter range (unlike repo-artefacts which generates for whole repo)

## 9. Testing

- pytest with `asyncio_mode = "auto"`
- Markers: `unit`, `integration`, `pre_deploy`, `post_deploy`
- Coverage target: 80% (`fail_under = 80`)
- CI: GitHub Actions on Python 3.11/3.12/3.13, runs pre-commit + pytest + build
- Quality: ruff linting, pyright type checking (basic mode)

## 10. Planned but Not Yet Implemented

From `docs/plans/2026-03-13-feat-study-review-interfaces-plan.md`:
- **Phase 1:** Textual TUI StudyCards tab in studyctl with voice, spaced repetition, wrong-answer review
- **Phase 2:** Local PWA web app (`studyctl serve`) with offline support, progress dashboard
- **Phase 3:** Native iOS/macOS apps + AWS cloud sync (future)
