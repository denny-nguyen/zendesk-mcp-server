# Zendesk MCP Server — Team Setup Guide

This MCP server connects Claude Desktop to Zendesk, enabling direct ticket search, retrieval, and investigation from within Claude conversations.

**Built on:** [reminia/zendesk-mcp-server](https://github.com/reminia/zendesk-mcp-server) + Goodnotes additions (`search_tickets`, `list_ticket_fields`).

---

## Prerequisites

- Mac (macOS) — required for Keychain credential storage
- [Claude Desktop](https://claude.ai/download) installed
- Python 3.11 or 3.12
- `uv` package manager

---

## Step 1 — Install uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Verify:
```bash
which uv   # should print something like /Users/yourname/.local/bin/uv
```

Keep note of the full path — you'll need it in Step 4.

---

## Step 2 — Clone the repo

```bash
git clone https://github.com/goodnotes/zendesk-mcp-server ~/zendesk-mcp-server
cd ~/zendesk-mcp-server
uv sync
```

---

## Step 3 — Store credentials in macOS Keychain

Run each command, replacing the placeholder values with your own:

```bash
# Zendesk subdomain (just "goodnotes", not the full URL)
security add-generic-password -a $USER -s zendesk-subdomain -w "goodnotes"

# Your Zendesk agent email
security add-generic-password -a $USER -s zendesk-email -w "you@goodnotesapp.com"

# Your Zendesk API token
# Get one from: Zendesk Admin → Apps & Integrations → Zendesk API → API tokens → Add API token
security add-generic-password -a $USER -s zendesk-api-token -w "your_api_token_here"
```

To verify a value was stored correctly:
```bash
security find-generic-password -a $USER -s zendesk-subdomain -w
security find-generic-password -a $USER -s zendesk-email -w
```

> **Note:** Each team member uses their own API token. You can generate one from Zendesk Admin under
> *Apps and Integrations → Zendesk API → API tokens*. Tokens are tied to your agent account.

---

## Step 4 — Configure Claude Desktop

Open `~/Library/Application Support/Claude/claude_desktop_config.json` in a text editor.

Find your `uv` path first:
```bash
which uv
```

Add the `zendesk` block inside `mcpServers`. If `mcpServers` doesn't exist yet, create it:

```json
{
  "mcpServers": {
    "zendesk": {
      "command": "/bin/sh",
      "args": [
        "-c",
        "ZENDESK_SUBDOMAIN=$(security find-generic-password -a $USER -s zendesk-subdomain -w) ZENDESK_EMAIL=$(security find-generic-password -a $USER -s zendesk-email -w) ZENDESK_API_KEY=$(security find-generic-password -a $USER -s zendesk-api-token -w) /Users/YOURUSERNAME/.local/bin/uv --directory /Users/YOURUSERNAME/zendesk-mcp-server run zendesk"
      ]
    }
  }
}
```

**Replace both `YOURUSERNAME` placeholders** with your actual macOS username (output of `whoami`).

If you already have other MCP servers configured, just add the `"zendesk": { ... }` block alongside them — don't replace the entire file.

---

## Step 5 — Restart Claude Desktop

Fully quit and reopen Claude Desktop. On next launch, the Zendesk MCP server will connect automatically.

To verify it's working: open a new conversation and ask *"Search Zendesk for fountain pen complaints in the last 7 days"*.

---

## Available Tools

Once connected, Claude has access to these Zendesk tools:

| Tool | What it does |
|------|-------------|
| `search_tickets` | Full ZQL search — keywords, date ranges, status, tags, boolean operators |
| `get_ticket` | Fetch a single ticket by ID |
| `get_tickets` | List recent tickets with pagination |
| `get_ticket_comments` | Full conversation thread for a ticket |
| `get_ticket_attachment` | Fetch an image attachment from a ticket |
| `create_ticket` | Create a new ticket |
| `create_ticket_comment` | Post a comment on a ticket |
| `update_ticket` | Update ticket fields (status, priority, assignee, etc.) |
| `list_ticket_fields` | List all active custom field definitions |

---

## Example ZQL queries for Claude

```
"fountain pen" created>7days
"sync error" status:open created>=2026-03-01 created<=2026-03-07
"eraser" -"billing" type:ticket
"crash" OR "freeze" created>30days
```

---

## Troubleshooting

**"Server disconnected" in Claude Desktop logs**
- Check that `uv` path in the config matches `which uv` output exactly
- Run the shell command manually in Terminal to see the raw error:
  ```bash
  ZENDESK_SUBDOMAIN=$(security find-generic-password -a $USER -s zendesk-subdomain -w) \
  ZENDESK_EMAIL=$(security find-generic-password -a $USER -s zendesk-email -w) \
  ZENDESK_API_KEY=$(security find-generic-password -a $USER -s zendesk-api-token -w) \
  /Users/YOURUSERNAME/.local/bin/uv --directory ~/zendesk-mcp-server run zendesk
  ```

**401 Unauthorized**
- Verify your API token is active: Zendesk Admin → Apps & Integrations → Zendesk API → API tokens
- Check the stored token matches: `security find-generic-password -a $USER -s zendesk-api-token -w`

**`security: command not found`**
- This setup requires macOS. Windows is not supported with this credential approach.

**JSON parse error in config**
- Validate your config file: `python3 -m json.tool ~/Library/Application\ Support/Claude/claude_desktop_config.json`
