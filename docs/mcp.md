# MCP Integrations

Optional MCP (Model Context Protocol) servers that enhance the study mentor experience with calendar time-blocking and reminders.

---

## Apple Calendar & Reminders (macOS)

```bash
npx -y @nicepkg/gkd@latest install FradSer/mcp-server-apple-reminders
```

Or add manually to your MCP client config:

```json
{
  "mcpServers": {
    "apple-reminders": {
      "command": "npx",
      "args": ["-y", "mcp-server-apple-reminders"]
    }
  }
}
```

**Config locations:**

- Claude Desktop: `~/Library/Application Support/Claude/claude_desktop_config.json`
- kiro-cli: `~/.kiro/settings.json` (mcpServers section)

**Enables:** native macOS notifications for study reminders, calendar time-blocking, break reminders during sessions.

---

## Google Calendar (cross-platform)

### Claude Desktop (built-in)

No MCP needed — use Settings → Extensions → Google Calendar.

### kiro-cli / Claude Code (MCP server)

```json
{
  "mcpServers": {
    "google-calendar": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-google-calendar"],
      "env": {
        "GOOGLE_CLIENT_ID": "<your-client-id>",
        "GOOGLE_CLIENT_SECRET": "<your-client-secret>"
      }
    }
  }
}
```

Requires a Google Cloud project with Calendar API enabled.

---

## Suggested Workflow

1. Morning: scheduled task runs `studyctl review`
2. Agent creates calendar time blocks for due topics
3. Reminders fire notification: "Time to study Python decorators"
4. Open kiro-cli or Claude Code with study-mentor agent
5. Agent checks energy level, adapts session
6. After session: agent records progress via `studyctl progress`
7. Sessions exported to DB automatically via scheduled `session-export`
