---
name: setup-mcp
description: Configure the Truelist MCP server so your AI assistant can validate emails directly as a tool. Uses the hosted server (no API key, browser login) for Claude Code, Claude Desktop, Cursor, and VS Code, with a local npm server as a fallback for clients without remote MCP support (e.g. Windsurf).
---

# Set Up Truelist MCP Server

Configure the Truelist MCP server so your AI coding assistant can validate emails directly.

Use the **hosted server** whenever the client supports remote (HTTP) MCP servers. It needs no API key: the first time a tool is used, the client opens the browser and the user signs up or logs in to Truelist (free plan, no credit card). Only fall back to the local npm server for clients that can't connect to a remote server.

## Hosted Server (recommended, no API key)

URL: `https://api.truelist.io/mcp` (streamable HTTP, OAuth 2.1 with dynamic client registration and PKCE).

### Claude Code

```bash
claude mcp add --transport http truelist https://api.truelist.io/mcp
```

Claude Code opens the browser to authorize the first time it calls a Truelist tool (or run `/mcp` and pick `truelist` to authorize right away).

### Claude Desktop

Edit your config file:

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "truelist": {
      "url": "https://api.truelist.io/mcp"
    }
  }
}
```

Restart Claude Desktop. On first use it opens a browser window asking you to authorize Truelist.

### Cursor

Add to `.cursor/mcp.json` (project-level) or your global Cursor MCP settings:

```json
{
  "mcpServers": {
    "truelist": {
      "url": "https://api.truelist.io/mcp"
    }
  }
}
```

### VS Code (GitHub Copilot)

Add to `.vscode/mcp.json`:

```json
{
  "servers": {
    "truelist": {
      "url": "https://api.truelist.io/mcp"
    }
  }
}
```

### Hosted tools

| Tool | What it does | Uses credits |
|------|--------------|--------------|
| `validate_email` | Validate one address (`email`, optional `validation_strategy`) | Yes |
| `validate_emails` | Validate up to 50 addresses; returns partial results plus `stopped_reason` if credits or rate limits run out | Yes |
| `create_batch` | Start a batch of up to 10,000 addresses (`name`, `emails`) | Yes |
| `get_batch` | Batch progress and CSV download URLs when complete | No |
| `list_batches` | List batches with progress counters | No |
| `list_email_addresses` | Search past results by batch or state | No |
| `get_credit_balance` | Credits available, auto top-up status, billing link | No |
| `check_account` | Connected account, plan and user | No |

To disconnect, revoke the connection in [app.truelist.io](https://app.truelist.io); the assistant will be asked to reconnect next time.

## Fallback: Local npm Server

For clients without remote MCP support. Requires Node.js 18+ and an API key, and exposes only `validate_email`, `validate_emails` (up to 50) and `check_account`.

### Get an API Key

**No key yet? Don't send the user to a settings page.** Use agent sign-up: the `truelist` skill's [device flow](../truelist/references/auth.md) gives the user a link and a code, they approve it in the browser (creating a free account if needed), and you receive a new key. Never print the full key, and ask before writing it to a shell profile or `.env`.

Fallback (if the device flow isn't available): sign up at [app.truelist.io/signup](https://app.truelist.io/signup) and create a key at [app.truelist.io/settings/api-keys](https://app.truelist.io/settings/api-keys).

### Install

```bash
npm install -g truelist-mcp
```

Or run directly with `npx` (no install needed):

```bash
npx truelist-mcp
```

### Windsurf (or any stdio-only client)

Add to your MCP configuration:

```json
{
  "mcpServers": {
    "truelist": {
      "command": "npx",
      "args": ["truelist-mcp"],
      "env": {
        "TRUELIST_API_KEY": "your-api-key"
      }
    }
  }
}
```

The same `command` / `args` / `env` block works in Claude Desktop, Cursor and VS Code (VS Code uses `"servers"` instead of `"mcpServers"`) if you can't use the hosted server. In Claude Code: `claude mcp add truelist -e TRUELIST_API_KEY=your-api-key -- npx truelist-mcp`.

## Verify It Works

After configuring, ask your assistant:

> "Validate the email test@example.com using Truelist"

It should call the `validate_email` tool and return the validation result.

## Troubleshooting

### Hosted: "Authentication required" or 401 on every request

The authorization expired or was revoked. Trigger any Truelist tool call (or `/mcp` in Claude Code) to re-authorize in the browser.

### Hosted: browser opens but doesn't redirect back

Log in with the same browser the assistant opened; the OAuth callback goes to a temporary local URL the assistant is listening on.

### Hosted: URL rejected

Use exactly `https://api.truelist.io/mcp` (HTTPS, no trailing slash).

### Local: "TRUELIST_API_KEY environment variable is required"

The API key is missing from your MCP config's `env` block or not exported in your shell.

### Local: authentication errors

The key is invalid or was revoked. Run the agent sign-up flow again to get a new one, or check your keys at [app.truelist.io/settings/api-keys](https://app.truelist.io/settings/api-keys).

### Tools not showing up

1. Restart your assistant after changing the config
2. Check that the config file is valid JSON and the path is correct for your platform
3. Local server only: verify `npx truelist-mcp` runs without errors when `TRUELIST_API_KEY` is set
