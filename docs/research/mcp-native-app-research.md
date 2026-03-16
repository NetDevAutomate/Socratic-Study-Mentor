# MCP Server in a Native macOS SwiftUI Application

## Research Summary

**Date**: 2026-03-15
**Goal**: Determine how a native macOS .app bundle can run an MCP server that agent CLI tools (Claude Code, Gemini CLI, Kiro) can connect to.
**Verdict**: Fully viable. The official MCP Swift SDK (Tier 2) provides both HTTP server transports and stdio transport. The recommended PoC approach is **Streamable HTTP on localhost** using `StatefulHTTPServerTransport`, which all three major CLI agents already support.

---

## 1. MCP Transport Mechanisms

The MCP specification (2025-03-26 and later) defines two standard transports:

### 1.1 stdio (Standard I/O)

- Client launches the MCP server as a **subprocess** and communicates via stdin/stdout.
- Messages are newline-delimited JSON-RPC.
- The server MUST NOT write anything to stdout that is not a valid MCP message.
- The server MAY write to stderr for logging (clients MAY capture and forward this).
- Best for: local CLI tools, command-line servers.
- **Limitation for .app bundles**: The client must be able to spawn the server as a child process. A macOS .app bundle is not typically launched this way -- you would need a separate CLI binary inside the bundle (e.g., `MyApp.app/Contents/MacOS/mcp-server`) that the client spawns.

### 1.2 Streamable HTTP (Recommended for .app)

- Server runs as an **independent HTTP process** exposing a single endpoint (e.g., `http://127.0.0.1:PORT/mcp`).
- Uses HTTP POST for client-to-server messages and optionally SSE (Server-Sent Events) for streaming responses.
- GET requests can establish a standalone SSE stream for server-initiated messages.
- DELETE requests terminate the session.
- Session management via `Mcp-Session-Id` header (optional but recommended).
- **Security requirements**:
  - MUST validate `Origin` header to prevent DNS rebinding attacks.
  - When running locally, SHOULD bind only to `127.0.0.1` (not `0.0.0.0`).
  - SHOULD implement proper authentication for all connections.

### 1.3 Deprecated: SSE (Server-Sent Events)

- The old SSE-only transport is **deprecated** as of the 2025-03-26 spec.
- Claude Code still supports it (`--transport sse`) but recommends HTTP instead.
- Streamable HTTP subsumes SSE functionality.

### 1.4 Custom Transports

- The spec explicitly allows custom transports (e.g., Unix domain sockets, WebSocket).
- However, clients must support them -- stick to stdio or Streamable HTTP for broad compatibility.

---

## 2. How Agent CLIs Discover and Connect to MCP Servers

### 2.1 Claude Code

**Configuration locations (3 scopes)**:
- **Local scope**: `~/.claude.json` (per-project path, default)
- **Project scope**: `.mcp.json` at project root (version-controlled, shared with team)
- **User scope**: `~/.claude.json` (cross-project, private)

**Three transport options**:

```bash
# Option 1: Remote HTTP (Streamable HTTP) -- RECOMMENDED
claude mcp add --transport http my-app http://127.0.0.1:9400/mcp

# Option 2: SSE (deprecated)
claude mcp add --transport sse my-app http://127.0.0.1:9400/sse

# Option 3: Local stdio
claude mcp add --transport stdio my-app -- /path/to/binary --args
```

**JSON configuration format** (via `claude mcp add-json`):

```json
{
  "mcpServers": {
    "my-native-app": {
      "type": "http",
      "url": "http://127.0.0.1:9400/mcp"
    }
  }
}
```

Or for stdio:

```json
{
  "mcpServers": {
    "my-native-app": {
      "type": "stdio",
      "command": "/Applications/MyApp.app/Contents/MacOS/mcp-helper",
      "args": [],
      "env": {}
    }
  }
}
```

### 2.2 Gemini CLI

**Configuration location**: `settings.json` in `~/.gemini/` (user) or `.gemini/` (project).

```json
{
  "mcpServers": {
    "my-native-app": {
      "command": "path/to/server",
      "args": ["--arg1", "value1"],
      "env": { "API_KEY": "$MY_API_TOKEN" },
      "cwd": "./server-directory",
      "timeout": 30000,
      "trust": false
    }
  }
}
```

**Transport selection**: Gemini CLI supports stdio, SSE, and Streamable HTTP. For HTTP:

```bash
gemini mcp add -t http my-app http://127.0.0.1:9400/mcp
```

### 2.3 Kiro (AWS)

**Configuration location**: `.kiro/settings.json` in the project root or user-level config.

Kiro uses the same `mcpServers` format as the ecosystem standard. Supports stdio, SSE, and HTTP transports. Configuration is nearly identical to Gemini CLI and Claude Desktop formats.

### 2.4 Universal Compatibility Summary

| Client       | stdio | Streamable HTTP | SSE (deprecated) | Config File          |
|-------------|-------|-----------------|-------------------|----------------------|
| Claude Code | Yes   | Yes             | Yes               | `.mcp.json`, `~/.claude.json` |
| Gemini CLI  | Yes   | Yes             | Yes               | `.gemini/settings.json`       |
| Kiro        | Yes   | Yes             | Yes               | `.kiro/settings.json`         |
| Claude Desktop | Yes | Yes            | Yes               | `claude_desktop_config.json`  |

**All clients support Streamable HTTP**, making it the universal choice.

---

## 3. Can a macOS .app Bundle Run an MCP Server?

**Yes, absolutely.** There are two viable approaches:

### 3.1 Approach A: Embedded HTTP Server (Recommended for PoC)

The SwiftUI app launches a lightweight HTTP server on `127.0.0.1:PORT` at startup. Agent CLIs connect to it as a remote HTTP MCP server.

**Advantages**:
- App controls the server lifecycle naturally.
- No subprocess management needed on the client side.
- App can provide a UI showing connected clients, tool calls, etc.
- Server persists as long as the app is running.
- Works with ALL MCP clients without any special handling.

**Disadvantages**:
- Need to pick and manage a port (risk of conflicts).
- Requires the app to be running before the CLI connects.
- Slightly more complex than stdio for a minimal PoC.

### 3.2 Approach B: CLI Helper Binary + stdio

Bundle a small CLI executable inside the .app that speaks MCP over stdio. Clients spawn this binary as a subprocess.

**Advantages**:
- Simpler protocol (just stdin/stdout).
- No port management.
- Client manages the lifecycle.

**Disadvantages**:
- The CLI binary is a separate process from the main SwiftUI app. IPC needed between them.
- Cannot easily share state with the running GUI app (need XPC, Unix socket, or similar).
- Harder to show status in the GUI.

### 3.3 Approach C: Hybrid (Best of Both Worlds)

The .app runs the HTTP server. A thin CLI shim binary (also in the bundle) translates stdio to HTTP, allowing clients that prefer stdio to also connect.

```
Agent CLI --stdio--> shim binary --HTTP--> SwiftUI app (HTTP MCP server)
Agent CLI --HTTP--> SwiftUI app (HTTP MCP server) directly
```

---

## 4. Official Swift MCP SDK

The **official MCP Swift SDK** (`modelcontextprotocol/swift-sdk`) is Tier 2 and provides everything needed.

### 4.1 Installation

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/modelcontextprotocol/swift-sdk.git", from: "0.11.0")
]

// Target dependency
.product(name: "MCP", package: "swift-sdk")
```

### 4.2 Requirements

- Swift 6.0+ (Xcode 16+)
- macOS 13.0+ (Ventura or later)
- iOS 16.0+, watchOS 9.0+, tvOS 16.0+, visionOS 1.0+

### 4.3 Available Transports

| Transport | Type | Best For |
|-----------|------|----------|
| `StdioTransport` | Client + Server | Local subprocesses, CLI tools |
| `HTTPClientTransport` | Client only | Connecting to remote servers |
| `StatelessHTTPServerTransport` | Server | Simple request-response, serverless |
| `StatefulHTTPServerTransport` | Server | Full sessions, SSE streaming, resumability |

### 4.4 Server Setup (Basic)

```swift
import MCP

// Create server with capabilities
let server = Server(
    name: "MyMacApp",
    version: "1.0.0",
    capabilities: .init(
        tools: .init(listChanged: true)
    )
)

// Create transport and start
let transport = StdioTransport()  // or HTTP transport
try await server.start(transport: transport)
```

### 4.5 Registering Tools

```swift
// List available tools
await server.withMethodHandler(ListTools.self) { _ in
    let tools = [
        Tool(
            name: "get_flashcards",
            description: "Get flashcards for the current study topic",
            inputSchema: .object([
                "properties": .object([
                    "topic": .string("The study topic to get flashcards for"),
                    "count": .string("Number of flashcards to return")
                ])
            ])
        )
    ]
    return .init(tools: tools)
}

// Handle tool calls
await server.withMethodHandler(CallTool.self) { params in
    switch params.name {
    case "get_flashcards":
        let topic = params.arguments?["topic"]?.stringValue ?? "general"
        let flashcards = getFlashcards(topic: topic)  // Your implementation
        return .init(
            content: [.text(flashcards)],
            isError: false
        )
    default:
        return .init(
            content: [.text("Unknown tool: \(params.name)")],
            isError: true
        )
    }
}
```

### 4.6 HTTP Server Transport (Framework-Agnostic)

The Swift SDK provides framework-agnostic HTTP types. You integrate with any HTTP framework (SwiftNIO, Vapor, Hummingbird, or even Foundation's built-in server):

```swift
// HTTPRequest -- framework-agnostic input
public struct HTTPRequest: Sendable {
    public let method: String          // "GET", "POST", "DELETE"
    public let headers: [String: String]
    public let body: Data?
}

// HTTPResponse -- framework-agnostic output (enum)
public enum HTTPResponse: Sendable {
    case accepted(headers: [String: String])     // 202, no body
    case ok(headers: [String: String])           // 200, no body
    case data(Data, headers: [String: String])   // 200 with JSON body
    case stream(AsyncThrowingStream<Data, Error>, headers: [String: String])  // 200 SSE
    case error(statusCode: Int, MCPError, sessionID: String?)
}
```

**StatefulHTTPServerTransport** usage:

```swift
let transport = StatefulHTTPServerTransport()
try await server.start(transport: transport)

// In your HTTP framework handler:
let httpRequest = HTTPRequest(
    method: request.method.rawValue,
    headers: Dictionary(request.headers),
    body: request.body
)
let response = await transport.handleRequest(httpRequest)

// Convert response to your framework's response type
switch response {
case .stream(let sseStream, let headers):
    // Pipe the async stream to the HTTP response body
case .data(let data, let headers):
    // Return JSON response
case .accepted(let headers):
    // Return 202
// ... etc
}
```

### 4.7 Complete HTTP App Example (SwiftNIO)

The SDK includes a full reference implementation using SwiftNIO in `Sources/MCPConformance/Server/HTTPApp.swift`:

```swift
actor HTTPApp {
    struct Configuration: Sendable {
        var host: String = "127.0.0.1"
        var port: Int = 3000
        var endpoint: String = "/mcp"
        var sessionTimeout: TimeInterval = 3600
    }

    // Factory creates a new Server + Transport per session
    typealias ServerFactory = @Sendable (String, StatefulHTTPServerTransport) async throws -> Server

    // Sessions tracked by session ID
    struct SessionContext {
        let server: Server
        let transport: StatefulHTTPServerTransport
        let createdAt: Date
        var lastAccessedAt: Date
    }
}
```

This handles:
- Session creation on first request (no existing `Mcp-Session-Id`)
- Session routing for subsequent requests
- Session cleanup on DELETE or timeout
- SSE streaming for POST and GET requests

---

## 5. Simplest Viable PoC Architecture

### 5.1 Architecture Diagram

```
+-------------------------------------------+
|          macOS SwiftUI App (.app)          |
|                                           |
|  +----------------+  +----------------+  |
|  |  SwiftUI Views |  |  App State     |  |
|  |  (study cards, |  |  (courses,     |  |
|  |   progress)    |  |   flashcards)  |  |
|  +-------+--------+  +-------+--------+  |
|          |                    |            |
|  +-------v--------------------v--------+  |
|  |         MCP Server Layer            |  |
|  |  (Server + StatefulHTTPTransport)   |  |
|  |  Tools: get_flashcards,             |  |
|  |         quiz_me, get_progress,      |  |
|  |         list_courses                |  |
|  +-------+----------------------------+  |
|          |                                |
|  +-------v----------------------------+   |
|  |  HTTP Server (SwiftNIO/Foundation) |   |
|  |  127.0.0.1:9400/mcp               |   |
|  +------------------------------------+   |
+-------------------------------------------+
          ^            ^            ^
          |            |            |
     Claude Code  Gemini CLI    Kiro CLI
     (HTTP)       (HTTP)        (HTTP)
```

### 5.2 Implementation Steps

1. **Create a SwiftUI macOS app** (Xcode, macOS 13+ target).

2. **Add the MCP Swift SDK** via Swift Package Manager:
   ```swift
   .package(url: "https://github.com/modelcontextprotocol/swift-sdk.git", from: "0.11.0")
   ```

3. **Create an MCPServerManager actor** that:
   - Initializes the `Server` with tool capabilities.
   - Creates a `StatefulHTTPServerTransport`.
   - Starts listening on `127.0.0.1:9400/mcp`.
   - Registers tool handlers that read from the app's shared state.

4. **Wire the HTTP layer** using SwiftNIO (the SDK's HTTPApp example) or a simpler approach with Hummingbird/Vapor.

5. **Start the server** when the app launches (in the `App` struct's `init` or via a `.task` modifier).

6. **Register with agent CLIs**:
   ```bash
   # Claude Code
   claude mcp add --transport http study-mentor http://127.0.0.1:9400/mcp

   # Gemini CLI
   gemini mcp add -t http study-mentor http://127.0.0.1:9400/mcp

   # Or via .mcp.json in any project:
   {
     "mcpServers": {
       "study-mentor": {
         "type": "http",
         "url": "http://127.0.0.1:9400/mcp"
       }
     }
   }
   ```

### 5.3 Minimal PoC Code Sketch

```swift
import SwiftUI
import MCP

@main
struct StudyMentorApp: App {
    @State private var serverManager = MCPServerManager()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .task {
                    await serverManager.start()
                }
        }
    }
}

@Observable
class MCPServerManager {
    var isRunning = false
    var connectedClients = 0

    func start() async {
        let server = Server(
            name: "StudyMentor",
            version: "1.0.0",
            capabilities: .init(
                tools: .init(listChanged: true)
            )
        )

        // Register tools
        await server.withMethodHandler(ListTools.self) { _ in
            return .init(tools: [
                Tool(
                    name: "get_flashcards",
                    description: "Get flashcards for a study topic",
                    inputSchema: .object([
                        "properties": .object([
                            "topic": .string("Study topic"),
                            "count": .string("Number of cards (default 5)")
                        ])
                    ])
                ),
                Tool(
                    name: "quiz_me",
                    description: "Start an interactive quiz on a topic",
                    inputSchema: .object([
                        "properties": .object([
                            "topic": .string("Quiz topic"),
                            "difficulty": .string("easy, medium, or hard")
                        ])
                    ])
                )
            ])
        }

        await server.withMethodHandler(CallTool.self) { params in
            switch params.name {
            case "get_flashcards":
                let topic = params.arguments?["topic"]?.stringValue ?? "general"
                return .init(
                    content: [.text("Flashcards for \(topic): ...")],
                    isError: false
                )
            case "quiz_me":
                let topic = params.arguments?["topic"]?.stringValue ?? "general"
                return .init(
                    content: [.text("Quiz on \(topic): Q1: ...")],
                    isError: false
                )
            default:
                return .init(
                    content: [.text("Unknown tool")],
                    isError: true
                )
            }
        }

        // Start HTTP server on localhost:9400
        // (Use SwiftNIO HTTPApp pattern from the SDK, or Hummingbird/Vapor)
        let transport = StatefulHTTPServerTransport()
        try? await server.start(transport: transport)

        // ... HTTP listener wiring here (see Section 4.7)
        isRunning = true
    }
}
```

---

## 6. Gotchas and Considerations

### 6.1 macOS App Sandbox

- **Non-sandboxed apps** (distributed outside the App Store): No restrictions on binding to localhost ports. This is the simplest path for a PoC or developer tool.
- **Sandboxed apps** (App Store): Require the `com.apple.security.network.server` entitlement to listen on network ports, and `com.apple.security.network.client` to make outbound connections.

```xml
<!-- Entitlements for sandboxed app -->
<key>com.apple.security.network.server</key>
<true/>
<key>com.apple.security.network.client</key>
<true/>
```

- App Sandbox does NOT prevent binding to localhost. The entitlements just need to be declared.
- App Review: Apple allows localhost server functionality for legitimate purposes (developer tools, local integrations). The key is that the app's primary function is clear and the networking is justified.

### 6.2 Port Management

- **Fixed port**: Simplest for a PoC. Pick something unlikely to conflict (e.g., 9400).
- **Dynamic port**: Bind to port 0, get the assigned port, and write it to a well-known file (e.g., `~/.config/study-mentor/mcp-port`). CLI configs would need a wrapper script to read this.
- **Bonjour/mDNS**: Could advertise the service, but no MCP clients support discovery this way (yet).
- **Recommendation**: Use a fixed port for PoC. Consider a settings UI for the port number.

### 6.3 App Lifecycle

- The HTTP server should start when the app launches and stop when it terminates.
- Consider showing server status in the menu bar or a status indicator in the app.
- Handle the case where the port is already in use (show an error, offer to change port).
- Use `ServiceLifecycle` (from `swift-server/swift-service-lifecycle`) for graceful shutdown.

### 6.4 Security

- **Bind to 127.0.0.1 only** -- never 0.0.0.0 for a local tool.
- **Validate Origin header** on all requests (the SDK's default validation pipeline does this).
- The SDK's `StatefulHTTPServerTransport` includes sensible defaults:
  - `OriginValidator.localhost()` -- rejects non-localhost origins.
  - `AcceptHeaderValidator` -- validates content type negotiation.
  - `ContentTypeValidator` -- validates request content types.
  - `ProtocolVersionValidator` -- validates MCP protocol version.
- For a local-only PoC, no additional auth is needed. For production, consider a bearer token.

### 6.5 macOS App Store Review

If targeting the Mac App Store:
- The app must have a clear primary purpose beyond being an MCP server.
- In-app purchases cannot be required to use the MCP tools (they are not digital goods).
- The networking entitlements are standard and will not cause review issues.
- Notarization (required for non-App Store distribution) has no issues with localhost servers.

### 6.6 Firewall

- macOS may prompt the user to allow incoming connections when the app first binds.
- Binding to `127.0.0.1` (not `0.0.0.0`) typically avoids this prompt.
- If the prompt appears, the user must allow it for MCP clients to connect.

---

## 7. Alternative Approaches Considered

### 7.1 XPC Service
- macOS-specific IPC, very fast, sandboxed.
- No MCP client supports XPC as a transport. Would require a custom bridge.
- Not recommended for cross-tool compatibility.

### 7.2 Unix Domain Socket
- Faster than TCP for local communication.
- No MCP client supports this natively. Would require custom transport on both sides.
- Could be used as a custom transport with a stdio shim, but adds complexity.

### 7.3 WebSocket
- Not part of the MCP spec.
- Some community implementations exist but not standardized.
- Not recommended.

### 7.4 Embedding a Python/Node MCP Server
- The Swift app could spawn a Python or Node MCP server as a subprocess.
- Defeats the purpose of a native app. Adds runtime dependencies.
- Not recommended.

---

## 8. Recommendations

### For PoC (Immediate)

1. **Use Streamable HTTP transport** with `StatefulHTTPServerTransport` from the official Swift SDK.
2. **Bind to `127.0.0.1:9400/mcp`** as a fixed endpoint.
3. **Do not sandbox** the app initially (distribute via direct download or TestFlight).
4. **Use SwiftNIO** for the HTTP layer (the SDK includes a reference `HTTPApp` implementation).
5. **Register with Claude Code** via `claude mcp add --transport http study-mentor http://127.0.0.1:9400/mcp`.

### For Production (Future)

1. Add a **menu bar indicator** showing server status and connected clients.
2. Implement **configurable port** with auto-detection fallback.
3. Add **bearer token authentication** for non-localhost deployments.
4. Consider the **hybrid approach** (Section 3.3) with a stdio shim for maximum compatibility.
5. Add **Sandbox entitlements** if targeting the Mac App Store.
6. Use **ServiceLifecycle** for graceful shutdown.
7. Implement **MCP Resources** to expose study materials as browsable resources.
8. Implement **MCP Prompts** to provide pre-built study workflows.

---

## Sources

| Source | Authority | URL |
|--------|-----------|-----|
| MCP Specification (Transports) | Official Spec | https://modelcontextprotocol.io/specification/2025-03-26/basic/transports |
| MCP Architecture | Official Docs | https://modelcontextprotocol.io/docs/learn/architecture |
| MCP Swift SDK | Official SDK (Tier 2) | https://github.com/modelcontextprotocol/swift-sdk |
| Claude Code MCP Docs | Official Docs | https://docs.anthropic.com/en/docs/claude-code/mcp |
| Gemini CLI MCP Docs | Official Docs | https://geminicli.com/docs/tools/mcp-server |
| MCP Build Server Guide | Official Tutorial | https://modelcontextprotocol.io/docs/develop/build-server |
| MCP Server Concepts | Official Docs | https://modelcontextprotocol.io/docs/learn/server-concepts |
