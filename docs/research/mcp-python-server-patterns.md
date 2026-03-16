# MCP Python Server Patterns -- Research (2026-03-15)

Best practices for building an MCP (Model Context Protocol) server in Python,
targeting Claude Code, Gemini CLI, and Kiro CLI as consumers.

**Primary source**: MCP Python SDK (`mcp` on PyPI) -- official repo at
`github.com/modelcontextprotocol/python-sdk`.

---

## 1. SDK Version and Installation

| Detail | Value |
|--------|-------|
| PyPI package | `mcp` |
| Current stable | **v1.26.0** (v1.x branch) |
| Python requirement | **>=3.10** |
| v2 status | Pre-alpha on `main`, stable release anticipated Q1 2026. **v1.x recommended for production.** |
| v2 class rename | `FastMCP` renamed to `MCPServer` (import from `mcp.server.mcpserver`) |

**Install with uv** (recommended):

```bash
uv add "mcp[cli]"          # adds to pyproject.toml
```

The `[cli]` extra includes the `mcp` CLI for `mcp dev`, `mcp run`, `mcp install`.

**Recommendation for studyctl**: Target v1.x (`FastMCP`) today. The `@mcp.tool()` decorator
API is identical in v2 (`MCPServer`), so migration is a one-line import change. Both versions
shown below; v1 is primary, v2 noted where different.

---

## 2. Defining MCP Tools with @tool Decorators

### v1 (FastMCP) -- current stable

```python
from mcp.server.fastmcp import FastMCP, Context

mcp = FastMCP("studyctl")

@mcp.tool()
def search_sessions(query: str, limit: int = 10) -> str:
    """Search study sessions by topic.

    Args:
        query: Natural language search query
        limit: Maximum number of results to return
    """
    # The docstring becomes the tool description shown to the LLM.
    # Parameter types are inferred from type hints -> JSON Schema.
    # Default values become optional parameters.
    results = do_search(query, limit)
    return json.dumps(results)
```

### v2 (MCPServer) -- pre-alpha, same decorator API

```python
from mcp.server.mcpserver import MCPServer, Context

mcp = MCPServer("studyctl")

@mcp.tool()
def search_sessions(query: str, limit: int = 10) -> str:
    """Search study sessions by topic."""
    ...
```

### Key rules

- **Docstring = tool description**. First line is the summary shown to the LLM.
- **Type hints = JSON Schema**. `str`, `int`, `float`, `bool`, `list[str]`, `dict[str, Any]` all work.
- **Default values** make parameters optional.
- **Async tools** are fully supported: `async def my_tool(...) -> str`.
- **Context injection**: Add a `ctx: Context` parameter -- it is auto-injected, not exposed to the LLM.

### Decorator options (v1)

```python
@mcp.tool(
    name="custom_name",           # override function name
    title="Human-Readable Title", # UI display name
    description="Override desc",  # override docstring
    structured_output=True,       # enable structured JSON output (default: auto-detect)
)
```

---

## 3. Structured Output (Return Types)

Tools return structured JSON when the return type annotation is compatible:

```python
from pydantic import BaseModel

class SessionResult(BaseModel):
    session_id: str
    topic: str
    score: float
    timestamp: str

@mcp.tool()
def get_session(session_id: str) -> SessionResult:
    """Retrieve a study session by ID."""
    row = db.fetch(session_id)
    return SessionResult(**row)
```

**Supported structured return types**:
- Pydantic `BaseModel` subclasses (recommended)
- `TypedDict`
- `dataclass`
- `dict[str, T]` where T is JSON-serializable
- Primitives (`str`, `int`, `float`, `bool`) -- wrapped in `{"result": value}`

To suppress auto-structured output: `@mcp.tool(structured_output=False)`.

### Advanced: Direct CallToolResult

For full control over responses:

```python
from mcp.types import CallToolResult, TextContent

@mcp.tool()
def advanced_query(query: str) -> CallToolResult:
    """Run an advanced query with metadata."""
    result = run_query(query)
    return CallToolResult(
        content=[TextContent(type="text", text=json.dumps(result))],
        _meta={"execution_time_ms": 42},  # hidden from model, visible to client
    )
```

---

## 4. Running as a Standalone CLI Entry Point (`studyctl-mcp`)

### Direct execution pattern

```python
# src/studyctl_mcp/server.py
from mcp.server.fastmcp import FastMCP  # v1
# from mcp.server.mcpserver import MCPServer as FastMCP  # v2

mcp = FastMCP("studyctl")

@mcp.tool()
def search_sessions(query: str) -> str:
    """Search study sessions."""
    ...

def main():
    """Entry point for the studyctl-mcp CLI."""
    mcp.run()  # defaults to stdio transport

if __name__ == "__main__":
    main()
```

### pyproject.toml entry point

```toml
[project.scripts]
studyctl-mcp = "studyctl_mcp.server:main"
```

After `uv sync`, the `studyctl-mcp` command is available and runs the MCP
server over stdio. This is the pattern AI coding assistants expect.

### Alternative: use `mcp run`

```bash
uv run mcp run src/studyctl_mcp/server.py
```

The `mcp run` CLI discovers the `mcp` object in the module and calls `.run()`.
The entry-point approach (`studyctl-mcp`) is preferred for distribution.

---

## 5. stdio vs HTTP Transport

### stdio (default, recommended for local CLI tools)

```python
mcp.run()                              # defaults to stdio
mcp.run(transport="stdio")             # explicit
```

- Communication over stdin/stdout.
- Client spawns the server process and communicates via pipes.
- **This is what Claude Code, Gemini CLI, and Kiro CLI expect for local tools.**
- Zero configuration, zero network ports.
- One server process per client session.

### Streamable HTTP (recommended for production/remote deployments)

```python
mcp.run(transport="streamable-http")
# With production options:
mcp.run(
    transport="streamable-http",
    stateless_http=True,      # no session persistence (scalable)
    json_response=True,       # JSON instead of SSE streaming
)
```

- Server listens on a network port.
- Multiple clients can connect.
- Supports load balancing.
- Required for remote/cloud deployments.

### Recommendation for studyctl-mcp

**Use stdio as default** -- it is the universal transport for local CLI tools.
All three target clients (Claude Code, Gemini CLI, Kiro CLI) support stdio.
Optionally accept a `--transport` flag for HTTP mode.

```python
import sys

def main():
    transport = "streamable-http" if "--http" in sys.argv else "stdio"
    mcp.run(transport=transport)
```

---

## 6. Client Registration

### Claude Code

```bash
# Method 1: CLI registration (recommended)
claude mcp add studyctl -- studyctl-mcp

# Method 2: With uv (if not installed as tool)
claude mcp add studyctl -- uv run --directory /path/to/project studyctl-mcp

# Method 3: JSON configuration
claude mcp add-json studyctl '{
  "type": "stdio",
  "command": "studyctl-mcp",
  "args": [],
  "env": {"STUDYCTL_DB": "/path/to/sessions.db"}
}'

# Verify
claude mcp list
claude mcp get studyctl
```

Claude Code stores config in `.claude/mcp.json` (project) or `~/.claude/mcp.json` (global).

### Gemini CLI

Add to `~/.gemini/settings.json`:

```json
{
  "mcpServers": {
    "studyctl": {
      "command": "studyctl-mcp",
      "args": [],
      "env": {
        "STUDYCTL_DB": "/path/to/sessions.db"
      }
    }
  }
}
```

Gemini CLI supports three transport types: stdio (spawns subprocess), SSE, and
Streamable HTTP. For local Python tools, stdio is standard.

### Kiro CLI

Kiro uses agent config files with MCP server references. In this project,
agents are defined in `agents/kiro/` with JSON configs:

```json
{
  "mcpServers": {
    "studyctl": {
      "command": "uvx",
      "args": ["--from", "mcp[cli]", "mcp", "run", "/path/to/server.py"]
    }
  }
}
```

Or with an installed entry point:

```json
{
  "mcpServers": {
    "studyctl": {
      "command": "studyctl-mcp"
    }
  }
}
```

---

## 7. Concrete Example: File Processing Tool

```python
"""MCP server that reads flashcard files and returns structured data."""

import json
import sqlite3
from pathlib import Path

from pydantic import BaseModel

from mcp.server.fastmcp import Context, FastMCP

mcp = FastMCP("studyctl")


class FlashcardResult(BaseModel):
    """Structured result for flashcard queries."""
    cards: list[dict[str, str]]
    total: int
    source_file: str


@mcp.tool()
def read_flashcards(file_path: str, topic: str | None = None) -> FlashcardResult:
    """Read flashcards from a markdown file, optionally filtering by topic.

    Args:
        file_path: Path to the markdown flashcard file
        topic: Optional topic filter (case-insensitive substring match)
    """
    path = Path(file_path).expanduser()
    if not path.exists():
        # Return error as structured data -- the LLM can handle it
        return FlashcardResult(cards=[], total=0, source_file=str(path))

    content = path.read_text()
    cards = _parse_markdown_cards(content)

    if topic:
        cards = [c for c in cards if topic.lower() in c.get("topic", "").lower()]

    return FlashcardResult(
        cards=cards,
        total=len(cards),
        source_file=str(path),
    )


@mcp.tool()
async def search_sessions(
    query: str,
    limit: int = 10,
    ctx: Context = None,  # auto-injected, not exposed to LLM
) -> str:
    """Search past study sessions by topic or content.

    Args:
        query: Natural language search query
        limit: Maximum results (1-50)
    """
    await ctx.info(f"Searching for: {query}")
    await ctx.report_progress(0.0, 1.0, "Querying database...")

    db_path = _get_db_path()
    conn = sqlite3.connect(db_path)
    try:
        rows = conn.execute(
            "SELECT session_id, topic, summary, timestamp "
            "FROM sessions WHERE topic LIKE ? OR summary LIKE ? "
            "ORDER BY timestamp DESC LIMIT ?",
            (f"%{query}%", f"%{query}%", min(limit, 50)),
        ).fetchall()
    finally:
        conn.close()

    await ctx.report_progress(1.0, 1.0, "Done")

    results = [
        {"session_id": r[0], "topic": r[1], "summary": r[2], "timestamp": r[3]}
        for r in rows
    ]
    return json.dumps(results, indent=2)


def _parse_markdown_cards(content: str) -> list[dict[str, str]]:
    """Parse Q&A pairs from markdown content."""
    cards = []
    lines = content.strip().split("\n")
    i = 0
    while i < len(lines):
        line = lines[i].strip()
        if line.startswith("Q:"):
            question = line[2:].strip()
            answer = ""
            if i + 1 < len(lines) and lines[i + 1].strip().startswith("A:"):
                answer = lines[i + 1].strip()[2:].strip()
                i += 1
            cards.append({"question": question, "answer": answer})
        i += 1
    return cards


def _get_db_path() -> str:
    """Resolve the sessions database path."""
    return str(Path("~/.config/studyctl/sessions.db").expanduser())


def main():
    mcp.run()


if __name__ == "__main__":
    main()
```

---

## 8. Error Handling in MCP Tools

### How errors propagate

When a tool function raises an exception, the MCP SDK catches it and returns a
`CallToolResult` with `isError=True` to the client. The LLM sees the error
message and can decide what to do.

### ToolError (explicit error signaling)

```python
from mcp.server.fastmcp.exceptions import ToolError

@mcp.tool()
def get_session(session_id: str) -> str:
    """Retrieve a study session."""
    if not session_id:
        raise ToolError("session_id is required")

    conn = sqlite3.connect(_get_db_path())
    try:
        row = conn.execute(
            "SELECT * FROM sessions WHERE session_id = ?", (session_id,)
        ).fetchone()
    except sqlite3.Error as e:
        raise ToolError(f"Database error: {e}") from e
    finally:
        conn.close()

    if row is None:
        raise ToolError(f"Session {session_id!r} not found")

    return json.dumps(dict(row))
```

### Exception hierarchy

```
Exception
  +-- McpError                          # Protocol-level errors
  |     +-- UrlElicitationRequiredError # OAuth flow needed
  +-- FastMCPError                      # Application-level errors
        +-- ToolError                   # Tool execution errors -> isError=True
        +-- ResourceError              # Resource access errors
        +-- ValidationError            # Parameter validation errors
```

### Best practices

1. **Raise `ToolError` for expected failures** (not found, invalid input, DB errors).
   The SDK converts these to `isError=True` responses that the LLM can reason about.
2. **Let unexpected exceptions propagate** -- the SDK catches them and returns a
   generic error. This prevents silent failures.
3. **Never return error strings as successful responses** -- use `ToolError` so the
   client knows the tool failed.
4. **Use `ctx.error()` for logging** alongside raising, if you need server-side logs.

---

## 9. Sharing State Between Tools (Lifespan Pattern)

The lifespan pattern initialises shared resources (database connections, config)
at server startup and cleans them up at shutdown. Tools access them via the
`Context` object.

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from dataclasses import dataclass
import sqlite3

from mcp.server.fastmcp import Context, FastMCP


class Database:
    """Wrapper around SQLite with connection pooling."""

    def __init__(self, path: str):
        self.path = path
        self._conn: sqlite3.Connection | None = None

    def connect(self) -> None:
        self._conn = sqlite3.connect(self.path)
        self._conn.row_factory = sqlite3.Row
        self._conn.execute("PRAGMA journal_mode=WAL")

    def close(self) -> None:
        if self._conn:
            self._conn.close()

    def query(self, sql: str, params: tuple = ()) -> list[dict]:
        cursor = self._conn.execute(sql, params)
        return [dict(row) for row in cursor.fetchall()]


@dataclass
class AppContext:
    """Shared state available to all tools via ctx.request_context.lifespan_context."""
    db: Database
    config: dict


@asynccontextmanager
async def app_lifespan(server: FastMCP) -> AsyncIterator[AppContext]:
    """Initialise shared resources at startup, clean up at shutdown."""
    import yaml
    config_path = Path("~/.config/studyctl/config.yaml").expanduser()
    config = yaml.safe_load(config_path.read_text()) if config_path.exists() else {}

    db = Database(str(Path("~/.config/studyctl/sessions.db").expanduser()))
    db.connect()
    try:
        yield AppContext(db=db, config=config)
    finally:
        db.close()


# Pass lifespan to the server
mcp = FastMCP("studyctl", lifespan=app_lifespan)


@mcp.tool()
def search_sessions(query: str, ctx: Context) -> str:
    """Search study sessions."""
    # Access shared database through lifespan context
    app: AppContext = ctx.request_context.lifespan_context
    rows = app.db.query(
        "SELECT session_id, topic, summary FROM sessions WHERE topic LIKE ?",
        (f"%{query}%",),
    )
    return json.dumps(rows)


@mcp.tool()
def get_config(key: str, ctx: Context) -> str:
    """Get a configuration value."""
    app: AppContext = ctx.request_context.lifespan_context
    value = app.config.get(key, f"Key {key!r} not found")
    return str(value)
```

### v2 typing improvement

In v2, `Context` accepts type parameters for stronger typing:

```python
from mcp.server.mcpserver import Context, MCPServer
from mcp.server.session import ServerSession  # v1 only

# v1: Context[ServerSession, AppContext]
# v2: Context[AppContext]  (simplified)
@mcp.tool()
def my_tool(ctx: Context[AppContext]) -> str:
    app = ctx.request_context.lifespan_context
    ...
```

---

## 10. Testing MCP Tools

### Strategy 1: Test tool functions directly (recommended)

Since `@mcp.tool()` decorated functions are regular Python functions, test the
underlying logic directly:

```python
# tests/test_tools.py
import pytest
from studyctl_mcp.server import _parse_markdown_cards, read_flashcards

def test_parse_markdown_cards():
    content = "Q: What is MCP?\nA: Model Context Protocol\n\nQ: Transport?\nA: stdio or HTTP"
    cards = _parse_markdown_cards(content)
    assert len(cards) == 2
    assert cards[0]["question"] == "What is MCP?"
    assert cards[0]["answer"] == "Model Context Protocol"

def test_read_flashcards_missing_file(tmp_path):
    result = read_flashcards(str(tmp_path / "nonexistent.md"))
    assert result.total == 0
    assert result.cards == []

def test_read_flashcards_with_topic_filter(tmp_path):
    md = tmp_path / "cards.md"
    md.write_text("Q: Python Q1\nA: Answer 1\nQ: Rust Q1\nA: Answer 2\n")
    result = read_flashcards(str(md))
    assert result.total == 2
```

### Strategy 2: Test via MCP client (integration test)

```python
# tests/test_mcp_integration.py
import pytest
from mcp import ClientSession
from mcp.client.stdio import stdio_client

@pytest.mark.asyncio
async def test_mcp_server_tools():
    """Integration test: spawn server, list tools, call one."""
    async with stdio_client(
        command="studyctl-mcp",  # or "uv", args=["run", "studyctl-mcp"]
    ) as (read_stream, write_stream):
        async with ClientSession(read_stream, write_stream) as session:
            await session.initialize()

            # List tools
            tools = await session.list_tools()
            tool_names = [t.name for t in tools.tools]
            assert "search_sessions" in tool_names

            # Call a tool
            result = await session.call_tool(
                "search_sessions",
                arguments={"query": "python", "limit": 5},
            )
            assert not result.isError
```

### Strategy 3: MCP Inspector (manual/exploratory)

```bash
# Launch the interactive inspector UI
uv run mcp dev src/studyctl_mcp/server.py

# With editable install for live reloading
uv run mcp dev src/studyctl_mcp/server.py --with-editable .
```

The Inspector provides a web UI to list tools, call them interactively, and
inspect request/response payloads.

---

## 11. Coexistence with FastAPI

An MCP server can share a process with FastAPI using ASGI mounting. This is
useful if `studyctl` has a web API and you want MCP on the same process.

### Using Streamable HTTP transport

```python
import contextlib

from fastapi import FastAPI
from starlette.routing import Mount
from mcp.server.fastmcp import FastMCP  # v1

# Create MCP server
mcp = FastMCP("studyctl")

@mcp.tool()
def search_sessions(query: str) -> str:
    """Search study sessions."""
    ...

# Create FastAPI app
api = FastAPI(title="StudyCtl API")

@api.get("/health")
def health():
    return {"status": "ok"}

# Mount MCP under /mcp path
@contextlib.asynccontextmanager
async def lifespan(app):
    async with mcp.session_manager.run():
        yield

app = FastAPI(lifespan=lifespan)
app.mount("/api", api)
app.mount("/mcp", mcp.streamable_http_app())

# Run with: uvicorn studyctl_mcp.app:app --reload
```

Clients connect to `http://localhost:8000/mcp` for MCP, `http://localhost:8000/api` for REST.

### Important notes

- ASGI mounting only works with **Streamable HTTP** transport, not stdio.
- For stdio (the normal case for CLI tools), MCP must be the only thing on stdin/stdout.
- If you need both stdio MCP and a web server, run them as **separate processes**.

---

## 12. Architecture Decision: v1 vs v2

| Factor | v1 (`FastMCP`) | v2 (`MCPServer`) |
|--------|---------------|-----------------|
| Stability | Stable, production-ready | Pre-alpha |
| Import | `mcp.server.fastmcp.FastMCP` | `mcp.server.mcpserver.MCPServer` |
| Tool API | `@mcp.tool()` | `@mcp.tool()` (identical) |
| Context | `Context[ServerSession, AppContext]` | `Context[AppContext]` |
| CLI support | `mcp run`, `mcp dev`, `mcp install` | Same |
| ASGI mounting | `.sse_app()`, `.streamable_http_app()` | `.streamable_http_app()` |

**Recommendation**: Start with v1 (`FastMCP`). The decorator API is identical.
When v2 ships stable, migration is:

```python
# Before (v1):
from mcp.server.fastmcp import FastMCP, Context
mcp = FastMCP("studyctl")

# After (v2):
from mcp.server.mcpserver import MCPServer, Context
mcp = MCPServer("studyctl")
```

---

## 13. Project-Specific Recommendations for studyctl-mcp

### Proposed tool surface

| Tool | Description | Returns |
|------|-------------|---------|
| `search_sessions` | Full-text search over study sessions | JSON array of sessions |
| `get_session` | Retrieve a single session by ID | Session JSON |
| `list_topics` | List all studied topics with counts | JSON array |
| `get_progress` | Learning progress for a topic | Progress JSON with scores |
| `read_flashcards` | Read flashcards from a file | Structured card data |
| `review_card` | Record a flashcard review result | Confirmation |
| `get_concept_graph` | Concept relationships for a topic | Graph JSON |

### Proposed file structure

```
packages/studyctl-mcp/
  pyproject.toml
  src/
    studyctl_mcp/
      __init__.py
      server.py          # FastMCP instance + tool definitions
      tools/
        __init__.py
        sessions.py      # search_sessions, get_session
        progress.py      # get_progress, list_topics
        flashcards.py    # read_flashcards, review_card
        concepts.py      # get_concept_graph
      state.py           # AppContext dataclass + lifespan
```

### pyproject.toml

```toml
[project]
name = "studyctl-mcp"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "mcp[cli]>=1.20",
    "pyyaml>=6.0",
]

[project.scripts]
studyctl-mcp = "studyctl_mcp.server:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

---

## Sources

| Source | Authority | URL |
|--------|-----------|-----|
| MCP Python SDK README (v1) | Official | github.com/modelcontextprotocol/python-sdk (v1.x branch) |
| MCP Python SDK README.v2.md | Official (pre-alpha) | github.com/modelcontextprotocol/python-sdk (main branch) |
| MCP official quickstart | Official | modelcontextprotocol.io/quickstart/server |
| MCP transports spec | Official | modelcontextprotocol.io/specification/2025-06-18/basic/transports |
| Claude Code MCP docs | Official | docs.claude.com/en/docs/claude-code/mcp |
| Gemini CLI MCP docs | Official | ai.google.dev/gemini-api/docs/gemini-cli/mcp-servers |
| PyPI `mcp` package | Official | pypi.org/project/mcp/ |
