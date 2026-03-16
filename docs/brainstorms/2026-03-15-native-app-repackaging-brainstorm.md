# Brainstorm: Native App Repackaging

**Date:** 2026-03-15
**Status:** Draft
**Participants:** ataylor, Claude

---

## What We're Building

A native macOS SwiftUI app (with iOS to follow) that unifies the currently fragmented Socratic Study Mentor and notebooklm-pdf-by-chapters into a single, installable study product. The app replaces the current "clone two repos, install uv, configure YAML" experience with a downloadable app that non-technical learners can use immediately, while retaining power-user depth via MCP/ACP integration with agent CLI tools.

### Core Value Proposition

**Live AI-powered Socratic tutoring** that adapts to your energy level, learning style, and neurodivergent needs (AuDHD-aware). Not just flashcard review -- an AI mentor that questions, probes, and guides understanding in real-time.

---

## Why This Approach

### Problem Statement

1. **Installation barrier**: Non-technical users can't install Python, uv, clone repos, or configure YAML
2. **Fragmentation**: Two separate repos (study consumption vs content generation) with no clear user journey connecting them
3. **Interface confusion**: TUI vs PWA vs CLI vs Agent -- users don't know which to use
4. **NotebookLM dependency**: Unofficial cookie-based API is fragile, and the roundabout pipeline (markdown -> PDF -> upload -> generate -> download) is unnecessary for flashcard/quiz generation

### Why Native SwiftUI

- Single downloadable app (.app bundle) -- no Python, no terminal, no configuration
- PDFKit is built into macOS/iOS -- native PDF chapter splitting
- Shared SwiftUI codebase for macOS + iOS
- App Store distribution enables monetisation
- Homebrew cask for power-user installation
- MCP server can run natively alongside the app

### Why Not (Alternatives Considered)

| Alternative | Why not |
|---|---|
| **Improve Python packaging (Homebrew formula + PyPI)** | Solves install friction but not the fragmentation or interface confusion. Still requires terminal comfort. |
| **Desktop wrapper (Electron/Tauri)** | Heavy runtime, not native feel, still bundles Python or needs backend |
| **PWA-only** | Already built and works, but limited iOS capabilities, can't run MCP server, no App Store monetisation path |
| **Native shell + Python engine** | PythonKit doesn't work on iOS. Embedding Python runtime in iOS app is painful and App Store review risky. Creates throwaway bridge code. |

---

## Key Decisions

### 1. Clean break from Python codebase

The Python repos (Socratic-Study-Mentor, notebooklm-pdf-by-chapters) served as prototypes. The new Swift project starts fresh, porting:
- **Algorithms**: SM-2 spaced repetition (~50 lines), flashcard/quiz JSON format
- **Data formats**: Flashcard JSON schema, quiz JSON schema, SQLite review database schema
- **UX patterns**: Energy-level adaptation, AuDHD-aware session flow, Socratic questioning methodology
- **Agent definitions**: Socratic mentor personality, session protocol (these are markdown, language-agnostic)

The Python repos remain archived as reference. No gradual migration -- clean cut.

### 2. Replace NotebookLM with official LLM APIs

| Component | Old (NotebookLM) | New (Native) |
|---|---|---|
| Flashcard generation | Upload PDF -> NotebookLM generates | Send chapter text to Claude/Gemini API -> structured JSON output |
| Quiz generation | Same roundabout pipeline | Same direct API approach |
| PDF splitting | PyMuPDF (Python) | PDFKit (Apple native framework) |
| Markdown import | pandoc + mermaid-cli + typst -> PDF -> upload | Read .md files directly, send text to LLM |
| Audio podcasts | NotebookLM deep-dive (unique) | Dropped from MVP. Optional NotebookLM integration later if needed. |

### 3. Two-mode architecture

| Mode | LLM Required | Description |
|---|---|---|
| **Study mode** (daily use) | Yes -- live AI | Socratic mentoring, adaptive questioning, energy-level awareness. This IS the product differentiator. |
| **Offline review** (fallback) | No | Review previously-studied cards, spaced repetition scheduling. Works without LLM connection. |
| **Generation mode** (occasional) | Yes | Import new PDF/Obsidian content, generate flashcards/quizzes |

### 4. Pluggable LLM backend (cost-conscious)

**macOS primary path -- agent CLI via MCP (free with existing subscriptions):**
- App runs an MCP server exposing tools: generate-flashcards, next-review-card, grade-response, socratic-question, etc.
- User's existing agent CLI (Claude Code, Gemini CLI, Kiro CLI, Kimi CLI, etc.) connects via MCP
- LLM intelligence comes from the agent's subscription -- no extra API cost
- ACP (Agent Client Protocol) support planned alongside MCP for broader agent compatibility

**macOS/iOS fallback -- direct API calls:**
- User provides API key (Claude, Gemini, OpenAI, OpenRouter, Z.AI, Moonshot, etc.)
- OpenAI-compatible API format as the common interface (most providers support it)
- Pay-per-use cost, but works without an agent CLI

**iOS primary path:**
- Direct API calls (no agent CLI available on iOS)
- Same multi-provider support as fallback path above

### 5. Content input: PDF + Obsidian first

**MVP**: Open PDF (PDFKit splits by TOC), import Obsidian .md files directly
**Future**: EPUB, plain text, web URLs (universal document import)

### 6. PoC-first approach (validated by research)

Before committing to the full native app, build a minimal PoC (~2.5-4 weeks with AI pair programming) that validates:
1. SwiftUI app opens a PDF and lists chapters via **PDFKit** `PDFOutline`
2. Sends chapter text to Claude API via **SwiftAnthropic** -> receives flashcards as structured JSON
3. Displays flashcards with flip + grade (SM-2 basics) in SwiftUI, stored in **GRDB.swift** SQLite
4. Exposes one MCP tool via **MCP Swift SDK** + SwiftNIO HTTP on `127.0.0.1:9400/mcp`

**Kill criteria**: If any step requires >1 week of fighting the framework, reassess. Fallback: hybrid Python backend + SwiftUI frontend (1.5 week validation).

---

## Architecture Overview

```
+------------------------------------------+
|           SwiftUI App (macOS/iOS)         |
|                                          |
|  +----------+  +----------+  +--------+  |
|  | PDF      |  | Study    |  | LLM    |  |
|  | Import   |  | Engine   |  | Router |  |
|  | (PDFKit) |  | (SM-2,   |  |        |  |
|  |          |  |  SQLite) |  |        |  |
|  +----------+  +----------+  +---+----+  |
|                                  |        |
+----------------------------------+--------+
                                   |
                    +--------------+--------------+
                    |              |               |
              +-----+----+  +-----+-----+  +------+------+
              | MCP      |  | Direct    |  | Local LLM   |
              | Server   |  | API       |  | (future)    |
              | (agent   |  | (Claude,  |  | (MLX/Ollama)|
              | CLI)     |  | Gemini,   |  |             |
              +----------+  | OpenRouter|  +-------------+
                            | etc.)     |
                            +-----------+
```

### Shared Swift Package

A Swift package containing the core logic, shared between macOS and iOS targets:
- `StudyEngine`: SM-2 algorithm, card scheduling, progress tracking
- `ContentImport`: PDFKit chapter splitting, Obsidian markdown parsing
- `LLMRouter`: Protocol-based LLM abstraction (MCP, direct API, local)
- `DataStore`: SQLite via GRDB.swift, flashcard/quiz JSON Codable models
- `SocraticMentor`: Session protocol, energy adaptation, questioning methodology

---

## Resolved Questions

### 1. MCP server in a macOS app -- VIABLE

**Research**: `docs/research/mcp-native-app-research.md`

- **Official Swift MCP SDK exists** (`modelcontextprotocol/swift-sdk`, v0.11.0+, Tier 2)
- **Recommended transport**: Streamable HTTP on `127.0.0.1:9400/mcp` using `StatefulHTTPServerTransport` + SwiftNIO
- **Agent CLI registration**: `claude mcp add --transport http study-mentor http://127.0.0.1:9400/mcp` (similar for Gemini CLI, Kiro CLI)
- **All major agent CLIs support HTTP transport** (Claude Code, Gemini CLI, Kiro CLI, Kimi CLI)
- **Key gotchas**: Bind to `127.0.0.1` (not `0.0.0.0`) to avoid firewall prompts; sandboxed apps need `com.apple.security.network.server` entitlement; use fixed port for PoC
- **Verdict**: This is straightforward. The Swift MCP SDK includes a complete SwiftNIO reference implementation.

### 2. ACP -- DEFER (MCP first)

**Research**: `docs/research/2026-03-15-acp-research.md`

- **ACP is a different layer**: MCP = agent-to-tools; ACP = editor-to-agent. They coexist by design.
- **Created by Zed Industries, co-stewarded by JetBrains**. 2.4K stars, 951 commits, Apache 2.0.
- **Massive agent adoption**: Claude Agent, GitHub Copilot, Gemini CLI, Cursor, Kiro, Kimi -- 25+ agents.
- **No Swift SDK**. Official SDKs: Python, Rust, TypeScript, Kotlin, Java. Would need Rust FFI or raw JSON-RPC.
- **Pre-1.0** (all SDKs 0.x) but not going away given adoption.
- **Verdict**: ACP would only matter for native Zed/JetBrains panel integration -- a future nice-to-have. MCP covers the agent CLI use case completely. Defer ACP to post-MVP.

### 3. Monetisation -- Freemium + BYOK + One-Time Premium

**Research**: `docs/research/monetisation-research.md`

- **Recommended model**: Free app with BYOK for AI features + optional one-time $29.99 premium purchase for advanced non-AI features (PDF batch import, advanced analytics, multi-device sync)
- **Precedent**: BoltAI, TypingMind -- one-time purchase BYOK apps on Mac App Store. Apple accepts BYOK as "user configuration", not content purchase.
- **Apple Small Business Program**: 15% commission (not 30%) under $1M revenue
- **Distribution**: Homebrew cask (primary for power users) + direct download + Mac App Store. All can coexist.
- **Direct API calls, never proxy user keys** -- avoids GDPR processor obligations and ToS risk
- **Provider note**: OpenRouter is the most BYOK-friendly provider. Google Gemini has "not for consumer use" clause and 18+ restriction -- risk for a study app. Prioritise Claude, OpenAI, OpenRouter.
- **No backend needed** for MVP -- all client-side. Backend only if adding user accounts/sync later.

### 4. Swift/SwiftUI learning curve -- REALISTIC WITH AI PAIRING

**Research**: `docs/research/swift-poc-feasibility.md`

- **Timeline**: 2.5-4 weeks with AI pair programming (Claude Code), 5-8 weeks without
- **Hardest paradigm shifts**: Strict type system, value vs reference semantics (SwiftUI views are structs), reactive state management (`@State`, `@Observable`, `@Binding`). Week 1 will feel slow.
- **No official Anthropic Swift SDK** -- use community `SwiftAnthropic` (232 stars, supports Messages API, streaming, function calling)
- **Key packages**: PDFKit (built-in), SwiftAnthropic (Claude API), GRDB.swift (SQLite, 8.3K stars), Vapor or SwiftNIO (HTTP server), MCP Swift SDK
- **PDFKit**: Reliably reads TOC bookmarks via `PDFOutline` -- same gotchas as PyMuPDF (missing TOCs in some PDFs)
- **Alternative flagged**: Hybrid approach (Python backend on localhost + thin SwiftUI frontend) could validate the UX in ~1.5 weeks instead of 3-4. Worth considering as a stepping stone.
- **Verdict**: Doable but challenging. AI pair programming is essential. The hybrid option is a pragmatic fallback if pure Swift feels too slow.

### 5. Data migration from Python -- CLEAN START

Given this targets a new (non-technical) user base, no migration needed. The Python tool's existing users are technical enough to export/re-import if needed. The SQLite schema and JSON formats will be documented for manual migration.

---

## Open Questions (Remaining)

### 1. Hybrid vs Pure Swift for PoC?
The Swift PoC research flagged that a Python backend + SwiftUI frontend hybrid could validate the UX in 1.5 weeks vs 3-4 weeks for pure Swift. However, the hybrid reintroduces the Python dependency we're trying to eliminate. Is the hybrid worth it as a time-boxed stepping stone, or should we commit to pure Swift from day one?

### 2. Provider-specific API differences
OpenAI-compatible format covers most providers, but Claude uses a different API format. SwiftAnthropic handles Claude; for OpenRouter/others we'd need a separate OpenAI-compatible client. How much multi-provider work goes into the PoC vs deferred?

### 3. iOS timeline
macOS app first is clear. But when does iOS become a priority? After macOS MVP ships? After monetisation is validated? This affects whether we invest in the shared Swift package architecture upfront.

---

## What We're NOT Building (YAGNI)

- Cloud backend / user accounts (at least not for MVP)
- Android app (Apple-first)
- Collaborative/social features
- NotebookLM podcast generation (dropped from core, optional future plugin)
- TUI (terminal UI was a proof of concept)
- Custom LLM fine-tuning
- Video/infographic generation

---

## Next Steps

1. **Resolve remaining open questions** (hybrid vs pure Swift, multi-provider scope for PoC, iOS timeline)
2. **Build PoC** (~2.5-4 weeks with Claude Code AI pairing) -- SwiftUI + PDFKit + SwiftAnthropic + GRDB + MCP Swift SDK
3. **If PoC succeeds**: `/ce:plan` for full MVP with monetisation (freemium + BYOK + $29.99 premium)
4. **If PoC struggles at week 2**: Fall back to hybrid (Python backend + SwiftUI frontend) for UX validation
5. **If hybrid also struggles**: Fall back to improved Python packaging (merge repos, Homebrew formula, better PWA)

## Research Documents

| Topic | Path |
|---|---|
| MCP in native apps | `docs/research/mcp-native-app-research.md` |
| ACP maturity | `docs/research/2026-03-15-acp-research.md` |
| Swift PoC feasibility | `docs/research/swift-poc-feasibility.md` |
| Monetisation models | `docs/research/monetisation-research.md` |
| Socratic Study Mentor repo analysis | `docs/research/repo-research-summary.md` |
| pdf-by-chapters repo analysis | `docs/research/notebooklm-pdf-by-chapters-analysis.md` |
