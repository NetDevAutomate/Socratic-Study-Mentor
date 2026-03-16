---
title: Native macOS + iOS Socratic Study Mentor App
date: 2026-03-15
status: brainstorm
participants: ataylor, claude
---

# Native macOS + iOS Socratic Study Mentor App

## The Problem

Non-technical users can't install the current system. It requires Python, uv, terminal commands, config file editing, and understanding of workspace packages. The feedback is clear: the study content and AI mentor are valuable, but the installation barrier locks out the target audience (self-learners, career changers, neurodivergent students).

## What We're Building

A universal SwiftUI app (macOS + iOS) that makes the Socratic Study Mentor accessible to anyone who can download an app and paste an API key.

### macOS App (Core)

The full-featured study workstation:

- **AI Socratic Tutor** — built-in chat with Claude/GPT that uses the Socratic mentor system prompt, adapts to energy levels, tracks progress
- **Flashcard & Quiz Review** — same SM-2 spaced repetition as the PWA, with native UI
- **Content Pipeline** — import from:
  - Local directories (existing JSON flashcards/quizzes)
  - Obsidian vault notes
  - PDFs and eBooks (built-in pdf-by-chapters capabilities)
  - AI-generated flashcards/quizzes from any imported content
- **NotebookLM Integration** — audio overviews, video, infographics
- **MCP Server** — expose study tools to Claude Code, Claude Desktop, Kiro (bidirectional: app calls agents AND agents call app)
- **Voice** — native AVSpeechSynthesizer (Siri voices, much better than Web Speech API)
- **Pomodoro Timer** — native notifications, menu bar integration
- **Accessibility** — OpenDyslexic font bundled, AuDHD-friendly design throughout

### iOS App (Companion)

Lighter-weight study-on-the-go experience:

- **Flashcard & Quiz Review** — same SM-2 data, synced from macOS
- **AI Study Mentor Chat** — Socratic questioning via Anthropic/OpenAI API
- **Body Doubling** — session presence, ambient study modes
- **NotebookLM Integration** — listen to audio overviews during commute
- **Voice** — native Siri voices (excellent quality on iOS)
- **No heavy lifting** — content generation, PDF processing, MCP all on macOS

### Shared (Both Platforms)

- SwiftUI codebase with platform-specific adaptations
- SM-2 spaced repetition engine (shared Swift code)
- SQLite database (same schema as current `sessions.db`)
- OpenDyslexic font (bundled)
- Tokyo Night dark theme (matching the PWA)
- Anthropic + OpenAI API support (user provides API key in onboarding)

## Why This Approach

1. **SwiftUI universal app** — single codebase targets macOS + iOS. Apple's recommended approach for 2026. Xcode handles platform differences.
2. **Direct API calls** — no Python backend needed. The app calls Anthropic/OpenAI APIs directly via Swift HTTP client. Non-technical users just need an API key.
3. **MCP for power users** — technical users get MCP integration so their agents can access the study data. This is additive, not required.
4. **Bonjour for sync** — macOS and iOS discover each other on local network. No cloud dependency. Future path to iCloud/CloudKit if the app proves valuable.

## Key Decisions

### 1. Two-tier UX
- **Beginner**: Download app → paste API key → import PDF or notes → study. No terminal, no config files.
- **Power user**: Enable MCP server in settings → agents can access study data. Configure local directories, Obsidian paths.

### 2. LLM Provider: Anthropic + OpenAI
- Settings screen to enter API key for each provider
- Model picker (Claude Sonnet/Opus, GPT-4o, etc.)
- System prompt is the existing Socratic mentor prompt, adapted per provider

### 3. macOS = Core, iOS = Companion
- Heavy tasks (PDF processing, content generation, MCP server) are macOS-only
- iOS focuses on review and study sessions
- Sync via Bonjour on LAN, with iCloud as future upgrade path

### 4. Same Data Layer
- SQLite with same schema as current `card_reviews`, `review_sessions`, `study_progress`, `concepts`
- Can coexist with the Python tools (same DB, same JSON format)
- Power users can use both the native app and CLI tools interchangeably

### 5. Content Sources (macOS)
- **Import JSON** — drag & drop or pick folder. Same format as pdf-by-chapters output.
- **Obsidian Vault** — point at vault, app reads markdown notes, generates flashcards via LLM
- **PDF/eBook** — built-in chapter splitting and content extraction (port pdf-by-chapters logic to Swift or use Python bridge)
- **AI Generation** — paste any text, LLM generates flashcards and quiz questions

### 6. Autoresearch-style Optimization (Future)
- After accumulating 50+ study sessions, introduce autonomous curriculum optimization
- Agent experiments with review intervals, question ordering, difficulty curves
- Pattern from Karpathy's autoresearch: modify strategy → run session → measure retention → keep/discard
- Phase 11+ feature, not v1

## Architecture

```
┌─────────────────────────────────────────────┐
│           macOS App (Core)                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐│
│  │ AI Tutor │ │ Review   │ │ Content      ││
│  │ (Chat)   │ │ (Cards)  │ │ Pipeline     ││
│  └────┬─────┘ └────┬─────┘ └──────┬───────┘│
│       │            │              │         │
│  ┌────┴────────────┴──────────────┴───────┐ │
│  │         Shared Swift Core              │ │
│  │  SM-2 Engine │ SQLite │ API Client     │ │
│  └────────────────┬───────────────────────┘ │
│                   │                         │
│  ┌────────────────┴───────────────────────┐ │
│  │    MCP Server (for Claude/Kiro)        │ │
│  └────────────────────────────────────────┘ │
│                   │ Bonjour                  │
└───────────────────┼─────────────────────────┘
                    │
┌───────────────────┼─────────────────────────┐
│           iOS App (Companion)                │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐│
│  │ AI Tutor │ │ Review   │ │ Body         ││
│  │ (Chat)   │ │ (Cards)  │ │ Doubling     ││
│  └────┬─────┘ └────┬─────┘ └──────────────┘│
│       │            │                        │
│  ┌────┴────────────┴──────────────────────┐ │
│  │         Shared Swift Core              │ │
│  │  SM-2 Engine │ SQLite │ API Client     │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

## Onboarding Flow (Non-Technical User)

1. Download app from website (or App Store later)
2. Welcome screen: "Get your API key" with links to Anthropic/OpenAI
3. Paste API key → app validates it with a test call
4. "How do you want to study?" → Import PDF / Paste notes / Try sample content
5. App generates flashcards/quizzes from imported content
6. Start first study session immediately

Total time to first study session: **< 5 minutes** (vs current ~30 minutes with CLI).

## Resolved Questions

1. **PDF processing in Swift** — Use Swift PDFKit for text extraction + send to LLM for intelligent chapter splitting and flashcard generation. No Python dependency. Simpler code, smarter results.

## Open Questions

1. **App distribution** — Direct download (.dmg) first, App Store later? App Store sandboxing limits filesystem access for importing content. TestFlight for beta?
2. **NotebookLM integration** — The current integration uses notebooklm-py (Python). Port to Swift HTTP calls or bridge?
3. **Pricing model** — Free app + user provides their own API key? Or bundle API access with a subscription?
4. **Offline mode** — Should flashcard/quiz review work fully offline? (Yes for review, AI tutor needs network)
5. **Shared Swift Package structure** — Single Xcode project with macOS + iOS targets, or separate projects with a shared SPM package?

## Existing Assets to Reuse

| Asset | Location | Reuse Strategy |
|-------|----------|---------------|
| SM-2 algorithm | `packages/studyctl/src/studyctl/review_db.py` | Port to Swift |
| JSON flashcard format | `packages/studyctl/src/studyctl/review_loader.py` | Same JSON, Swift Codable |
| Socratic mentor prompt | `agents/claude/socratic-mentor.md` | Embed in app |
| MCP study-speak server | `agents/mcp/study-speak-server.py` | Rewrite as Swift MCP server |
| PWA UI/UX patterns | `packages/studyctl/src/studyctl/web/static/` | Reference for SwiftUI design |
| pdf-by-chapters | `/Users/ataylor/code/personal/tools/notebooklm_pdf_by_chapters/` | Port or bridge |
| AKI profile | `/Users/ataylor/code/personal/tools/aki-socratic-mentor/` | Reference for agent config |
| DB schema | `packages/agent-session-tools/src/agent_session_tools/schema.sql` | Same schema in Swift |

## Success Criteria

- Non-technical user goes from download to first study session in < 5 minutes
- AI Socratic tutor adapts to energy level and tracks progress
- Flashcard/quiz SM-2 review works identically to PWA
- MCP tools accessible from Claude Code/Desktop/Kiro
- OpenDyslexic font and AuDHD-friendly design throughout
- macOS and iOS sync seamlessly on LAN
- Power users can use native app alongside existing CLI tools (shared DB)

## Target Directory

```
/Users/ataylor/code/personal/tools/apps/
├── iOS/
│   └── SocraticMentor/     ← iOS companion app
└── macOS/
    └── SocraticMentor/     ← macOS core app
```

Shared Swift package for core logic (SM-2, SQLite, API client, models) as a Swift Package referenced by both targets.
