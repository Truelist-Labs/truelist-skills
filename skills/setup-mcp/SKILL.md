---
name: setup-mcp
description: Configure the Truelist MCP server so your AI assistant can validate emails directly as a tool. Supports Claude Desktop, Claude Code, Cursor, VS Code, and Windsurf.
---

# Set Up Truelist MCP Server

Configure the Truelist MCP server so your AI coding assistant can validate emails directly.

## Prerequisites

- Node.js 18+
- A Truelist API key (see below)

## Get an API Key

**No key yet? Don't send the user to a settings page.** Use agent sign-up: the `truelist` skill's [device flow](../truelist/references/auth.md) gives the user a link and a code, they approve it in the browser (creating a free account if needed), and you receive a new key. Never print the full key, and ask before writing it to a shell profile or `.env`.

Fallback (if the device flow isn't available): sign up at [app.truelist.io/signup](https://app.truelist.io/signup) and create a key at [app.truelist.io/settings/api-keys](https://app.truelist.io/settings/api-keys).

## Install

```bash
npm install -g truelist-mcp
```

Or run directly with `npx` (no install needed):

```bash
npx truelist-mcp
```

## Configure Your Assistant

### Claude Desktop

Edit your config file:

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

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

Restart Claude Desktop after saving.

### Claude Code

Run this command:

```bash
claude mcp add truelist -- npx truelist-mcp
```

Then set the environment variable:

```bash
export TRUELIST_API_KEY=your-api-key
```

Or add it to your shell profile (`~/.zshrc`, `~/.bashrc`):

```bash
echo 'export TRUELIST_API_KEY=your-api-key' >> ~/.zshrc
```

### Cursor

Add to `.cursor/mcp.json` in your project root (or global Cursor settings):

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

### VS Code (GitHub Copilot)

Add to `.vscode/mcp.json` in your project:

```json
{
  "servers": {
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

### Windsurf

Add to your Windsurf MCP configuration:

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

## Available Tools

Once configured, your assistant gets these tools:

### `validate_email`

Validate a single email address.

**Input:** `{ "email": "user@example.com" }`

**Output:**
```json
{
  "email": "user@example.com",
  "state": "ok",
  "sub_state": "email_ok",
  "suggestion": null,
  "domain": "example.com",
  "canonical": "user@example.com",
  "mx_record": "mx.example.com",
  "is_valid": true,
  "is_deliverable": true
}
```

### `validate_emails`

Validate up to 50 emails in one call.

**Input:** `{ "emails": ["user1@example.com", "user2@example.com"] }`

**Output:** Array of `{ email, state, sub_state, is_valid }` objects.

### `check_account`

Check your Truelist account info and plan.

**Input:** none

**Output:** `{ email, name, uuid, timeZone, isAdminRole, paymentPlan }`

## Verify It Works

After configuring, ask your assistant:

> "Validate the email test@example.com using Truelist"

It should call the `validate_email` tool and return the validation result.

## Troubleshooting

### "TRUELIST_API_KEY environment variable is required"

The API key is missing from your MCP config's `env` block or not exported in your shell.

### Tools not showing up

1. Restart your assistant after changing the config
2. Verify `npx truelist-mcp` runs without errors when `TRUELIST_API_KEY` is set
3. Check that the config file path is correct for your platform

### Authentication errors

The key is invalid or was revoked. Run the agent sign-up flow again to get a new one, or check your keys at [app.truelist.io/settings/api-keys](https://app.truelist.io/settings/api-keys).
