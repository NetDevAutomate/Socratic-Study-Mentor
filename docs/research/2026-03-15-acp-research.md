# Agent Client Protocol (ACP) Research

**Date**: 2026-03-15
**Sources**: agentclientprotocol.com, GitHub repo, PyPI, crates.io, npm registry

---

## 1. What Is ACP and How Does It Differ from MCP?

**ACP** (Agent Client Protocol) standardises communication between **code editors** and **coding agents**. Think of it as "LSP for AI agents" -- just as the Language Server Protocol decoupled language intelligence from editors, ACP decouples AI coding agents from the editors that host them.

**MCP** (Model Context Protocol) standardises how LLMs access **tools and context** (files, databases, APIs). It sits at a different layer.

### The Key Distinction

```
MCP:  LLM  <--tools/context-->  External Services (databases, APIs, file systems)
ACP:  Code Editor  <--agent UX-->  Coding Agent (which may itself use MCP tools)
```

ACP and MCP are **complementary, not competing**:

- ACP handles the **editor-to-agent UX layer**: streaming responses, displaying diffs, requesting tool-call permissions, managing sessions, rendering agent plans.
- MCP handles the **agent-to-world data layer**: the agent calls MCP servers to read files, query databases, hit APIs.
- ACP explicitly re-uses MCP JSON types where possible. The ACP schema includes `McpCapabilities` as a first-class field in `AgentCapabilities`.
- The ACP architecture page states: "Commonly the code editor will have user-configured MCP servers. When forwarding the prompt from the user, it passes configuration for the [MCP servers] to the agent."

### Protocol Mechanics

| Aspect | ACP | MCP |
|--------|-----|-----|
| Transport | JSON-RPC over stdio (local), HTTP/WebSocket (remote, WIP) | JSON-RPC over stdio, SSE, HTTP |
| Direction | Bidirectional (agent can request permissions from editor) | Primarily client-to-server |
| Session model | Multi-session per connection, concurrent trains of thought | Single request/response or streaming |
| Content format | Markdown-first, with structured diff types | Tool results, resource content |
| Primary concern | Agent UX in editors | Tool and context access |

---

## 2. Who Created ACP? Governance and Backing

- **Created by**: Zed Industries (makers of the Zed editor)
- **Co-steward**: JetBrains (both logos appear as primary sponsors on the ACP website footer)
- **GitHub org**: `agentclientprotocol` (Apache 2.0 license)
- **Repo created**: 2025-06-23
- **Current stats** (2026-03-15): 2,411 stars, 187 forks, 951 commits
- **Governance**: Open contribution under Apache 2.0; no CLA required. Structured contribution process "to ensure changes are well-considered." Community discussion via GitHub Discussions and Zulip (`agentclientprotocol.zulipchat.com`).

The backing is significant: Zed + JetBrains gives it access to two major editor ecosystems (Zed natively, JetBrains via IntelliJ platform + Junie). The wide agent adoption (see section 3) suggests strong industry momentum.

---

## 3. Which Agent/CLI Tools Support ACP Today?

### Agents (implement ACP server-side)

The ecosystem is already large (25+ agents listed on the official site):

| Agent | Notes |
|-------|-------|
| **Claude Agent** | Via Zed's SDK adapter (`zed-industries/claude-agent-acp`) |
| **Codex CLI** | Via Zed's adapter (`zed-industries/codex-acp`) |
| **Gemini CLI** | Native ACP support |
| **GitHub Copilot** | Public preview (announced 2026-01-28) |
| **Cursor** | Via CLI ACP mode |
| **Kiro CLI** | Native ACP support |
| **Kimi CLI** | Native ACP support |
| **Junie** | JetBrains' agent, native ACP |
| **Augment Code** | Native ACP support |
| **Cline** | Native ACP support |
| **Goose** (Block) | Native ACP support |
| **Docker cagent** | Native ACP support |
| **Blackbox AI** | Native ACP support |
| **Mistral Vibe** | Native ACP support |
| **fast-agent** | Native ACP support |
| **Roo Code** | Native ACP support |
| **Windsurf** | Native ACP support |
| AutoDev, Factory Droid, Minion Code, OpenClaw, AgentPool, fount | Various levels of support |

### Clients (implement ACP client-side -- editors/IDEs)

| Client | Status |
|--------|--------|
| **Zed** | First-class, native |
| **JetBrains IDEs** | Via Junie integration |
| **VS Code** | Community/third-party extensions |

### Agent Frameworks with ACP Integration

- **LangChain / LangGraph** -- via Deep Agents ACP
- **LlamaIndex** -- via `workflows-acp` adapter
- **Koog** (JetBrains) -- via `agents-features-acp`
- **AgentPool**, **LLMling-Agent**, **fast-agent** -- built-in ACP

### Connectors

- **Aptove Bridge** -- bridges stdio-based ACP agents to mobile over WebSocket
- **OpenClaw** -- bridges to OpenClaw Gateway

---

## 4. Is There a Swift/Native SDK?

**No official Swift SDK exists.** The official SDKs are:

| Language | Package | Latest Version |
|----------|---------|----------------|
| **Python** | `agent-client-protocol` (PyPI) | 0.8.1 |
| **Rust** | `agent-client-protocol` (crates.io) | 0.10.2 (880K downloads) |
| **TypeScript** | `@agentclientprotocol/sdk` (npm) | 0.16.1 |
| **Kotlin** | `acp-kotlin` (GitHub) | JVM target, others in progress |
| **Java** | `java-sdk` (GitHub) | Available |

The Community Libraries page exists but returned no content when fetched, suggesting it is either empty or rendered client-side only. **No Swift, Go, C#, or .NET SDKs were found** in either official or community listings.

For a Swift/native macOS implementation, the options would be:
1. Use the **Rust SDK** via Swift-Rust FFI (UniFFI or direct C bridging)
2. Implement against the **JSON schema** directly (`schema/schema.json` in the repo) -- ACP is JSON-RPC, which is straightforward to implement in any language
3. Wait for a community Swift SDK to emerge

---

## 5. How Mature Is the Spec?

### Version Assessment: Pre-1.0, Actively Evolving

**All SDK versions are 0.x** -- none have reached 1.0:
- Python SDK: 0.8.1
- Rust crate: 0.10.2
- TypeScript SDK: 0.16.1
- (The different version numbers across SDKs suggest they track the schema independently)

### Maturity Signals

**Positive signals (surprisingly mature for 0.x)**:
- 951 commits in ~9 months (very active development)
- 2.4K GitHub stars, 187 forks
- 25+ agents already support it, including major players (Claude, Copilot, Gemini, Cursor)
- Comprehensive protocol spec with 15+ sub-sections (initialization, sessions, content, tool calls, file system, terminals, agent plans, slash commands, etc.)
- 880K Rust crate downloads suggests real production usage
- Structured contribution process, RFDs (Request for Discussion) system
- ACP Registry for discovering/installing agents

**Caution signals**:
- Still 0.x -- breaking changes are expected and likely
- Remote agent support explicitly described as "work in progress"
- No formal versioning/compatibility guarantee visible in the spec
- The SDK version numbers are diverging (0.8 vs 0.10 vs 0.16), which suggests rapid, possibly inconsistent iteration
- The Python SDK README says "Releases track the upstream ACP schema" -- implying the schema itself is the source of truth and changes frequently
- `ACP_SCHEMA_VERSION=<ref> make gen-all` in the Python SDK confirms schema-driven code generation, meaning each schema bump can break things

### Honest Assessment

ACP is in a **"late alpha / early beta"** stage. It is far enough along that real products ship with it (Zed, JetBrains IDEs, GitHub Copilot preview), but the spec is still evolving and breaking changes should be expected before 1.0. The adoption momentum is strong enough that it is unlikely to be abandoned, but the API surface is not yet frozen.

---

## 6. Should You Invest in ACP Now or Focus on MCP First?

### Recommendation: MCP First, ACP When You Need Editor Integration

**They solve different problems**, so the answer depends on what you are building:

#### If you are building a **tool/service that AI agents consume** (database connector, API wrapper, file processor):
- **Focus on MCP only.** ACP is irrelevant to you. Your tool is consumed by agents, not by editors.

#### If you are building a **coding agent** that runs in editors:
- **Invest in ACP now.** The adoption is already broad enough that ACP support is becoming table-stakes for editor-integrated agents. Claude, Copilot, Gemini CLI, Cursor, and Kiro all support it. If your agent does not speak ACP, it cannot run in Zed or JetBrains IDEs without custom integration.

#### If you are building a **study/learning tool** like Socratic Study Mentor:
- **MCP first, ACP later.** Your primary interface is CLI/TUI/PWA, not a code editor. MCP lets agents call your tools. ACP would only matter if you wanted to expose your study agent as an editor panel -- possible but not the primary use case.
- However, the project already has agent support via `AGENTS.md`, Claude Code `/agent`, and kiro-cli -- these are proto-ACP patterns. When ACP stabilises, wrapping the study agent as a proper ACP agent would unlock native Zed/JetBrains integration with streaming, diff display, etc.

### Decision Matrix

| Building... | Invest in ACP? | Invest in MCP? |
|-------------|---------------|---------------|
| Tool/service for agents | No | **Yes** |
| Coding agent for editors | **Yes, now** | Yes (for tool access) |
| CLI/TUI application | Later | **Yes** |
| Agent framework/orchestrator | **Yes** | **Yes** |

---

## 7. Can ACP and MCP Coexist in the Same Application?

**Yes, and they are designed to.** This is not an either/or choice.

The ACP architecture explicitly describes the coexistence pattern:

1. The **code editor** (ACP client) has user-configured **MCP servers**.
2. When the editor sends a prompt to an **ACP agent**, it passes the MCP server configuration along.
3. The **ACP agent** connects to MCP servers to access tools and context.
4. The agent streams results back to the editor via ACP.

```
User in Editor
    |
    | (ACP: prompt, sessions, diffs, plans)
    v
Coding Agent
    |
    | (MCP: tool calls, resource access)
    v
MCP Servers (files, databases, APIs)
```

The `AgentCapabilities` schema includes `mcpCapabilities` with `http` and `sse` boolean fields, meaning an ACP agent explicitly advertises what MCP transports it can handle.

**In practice**: An application can implement an ACP agent (for editor integration) that internally uses MCP clients (for tool access). The protocols operate at different layers and do not conflict.

---

## Summary

| Question | Answer |
|----------|--------|
| What is ACP? | LSP-like protocol for editor-to-agent communication |
| Created by? | Zed Industries + JetBrains |
| Agent support? | 25+ agents including Claude, Copilot, Gemini, Cursor, Kiro |
| Swift SDK? | **No.** Rust, Python, TypeScript, Kotlin, Java only |
| Maturity? | **0.x pre-release.** Active, well-adopted, but expect breaking changes |
| Invest now? | Only if building editor-integrated agents. Otherwise MCP first |
| ACP + MCP coexist? | **Yes, by design.** They complement each other at different layers |

---

## Key Links

- **Spec site**: https://agentclientprotocol.com/
- **GitHub**: https://github.com/agentclientprotocol/agent-client-protocol
- **Python SDK**: https://github.com/agentclientprotocol/python-sdk (`pip install agent-client-protocol`)
- **Rust SDK**: https://crates.io/crates/agent-client-protocol
- **TypeScript SDK**: https://www.npmjs.com/package/@agentclientprotocol/sdk
- **Schema**: https://github.com/agentclientprotocol/agent-client-protocol/blob/main/schema/schema.json
- **Community chat**: https://agentclientprotocol.zulipchat.com/
