# Connecting to Truelist

The user should never have to visit a settings page or paste an API key. There are two zero-copy paths:

| Situation | Path |
|-----------|------|
| The client supports remote MCP servers (Claude Code, Claude Desktop, Cursor, VS Code) and the user is happy to use tools | **Hosted MCP**: no key at all, OAuth in the browser |
| You're calling the API with curl / a CLI, the user's app needs a key, or the client has no MCP | **Device sign-up**: the user approves a link and you receive an API key |

## Hosted MCP (no key)

`https://api.truelist.io/mcp` is a streamable-HTTP MCP server with OAuth 2.1 (dynamic client registration + PKCE). The first tool call opens the browser, where the user signs up or logs in.

```bash
claude mcp add --transport http truelist https://api.truelist.io/mcp
```

Other clients: see the `setup-mcp` skill. Ask before changing the user's MCP config. A newly added server usually shows up only in a new session; if the user wants results now, also run device sign-up for this session.

## Device sign-up

You start a request, the user approves it in their browser (signing up for a free account on the way if they don't have one), and you collect a fresh API key.

Base URL: `https://api.truelist.io`. Both endpoints below are unauthenticated.

### Secret handling rules

- Never print the full API key or the `device_code` — not in chat, not in command output. The scripts below write them to files with `umask 077` and only print masked values.
- Never commit the key. Never write it to a file without asking the user first (the temp file below is a short-lived handoff, not storage).
- When you need to refer to the key, show it masked: first 6 and last 4 characters.

### Step 1: Start the request

```bash
umask 077
TL_DIR="${TMPDIR:-/tmp}/truelist-agent"; mkdir -p "$TL_DIR"
curl -s -X POST https://api.truelist.io/api/v1/agent_auth/device \
  -H "Content-Type: application/json" \
  -d '{"client_name": "Claude Code"}' > "$TL_DIR/device.json"
jq '{user_code, verification_uri_complete, expires_in, interval, error}' "$TL_DIR/device.json"
```

Set `client_name` to the agent you are (`Claude Code`, `Cursor`, `Codex`, ...). It becomes the key's name in the user's account, e.g. "Claude Code (agent)".

If the response has no `user_code` (for example a 404 because the endpoint isn't live yet), use the hosted MCP server or skip to [Fallback](#fallback-copy-a-key-manually).

### Step 2: Show the user the link

Send this to the user **before** you start polling, in your own words:

> To connect Truelist, open this link and approve the request:
> **<verification_uri_complete>**
> Confirm the code on the page matches **<user_code>**. If you don't have a Truelist account, you can create a free one there (no credit card). I'll wait here.

If you can open URLs for the user (e.g. `open` on macOS), offer to, but still show the link. The code check matters: it's what stops a phishing link from attaching someone else's agent to their account.

### Step 3: Poll for the key

Run this right after showing the link. It waits up to the request's `expires_in` (15 minutes), so give the command a long timeout (or run it in the background and check back). It never prints the key.

```bash
umask 077
TL_DIR="${TMPDIR:-/tmp}/truelist-agent"
DEVICE_CODE=$(jq -r .device_code "$TL_DIR/device.json")
INTERVAL=$(jq -r '.interval // 5' "$TL_DIR/device.json")
DEADLINE=$(( $(date +%s) + $(jq -r '.expires_in // 900' "$TL_DIR/device.json") ))
STATUS=timeout
while [ "$(date +%s)" -lt "$DEADLINE" ]; do
  sleep "$INTERVAL"
  RESP=$(curl -s -X POST https://api.truelist.io/api/v1/agent_auth/token \
    -H "Content-Type: application/json" \
    -d "{\"device_code\": \"$DEVICE_CODE\"}")
  if [ -n "$(echo "$RESP" | jq -r '.api_key // empty' 2>/dev/null)" ]; then
    echo "$RESP" | jq -r .api_key > "$TL_DIR/key"
    KEY=$(cat "$TL_DIR/key")
    echo "approved: $(echo "$RESP" | jq -r .email) key=${KEY:0:6}...${KEY: -4}"
    STATUS=approved; break
  fi
  ERR=$(echo "$RESP" | jq -r '.error // empty' 2>/dev/null)
  case "$ERR" in
    authorization_pending) ;;
    slow_down) INTERVAL=$((INTERVAL + 5)) ;;
    *) echo "$RESP" | jq '{error, instructions_for_agent}' 2>/dev/null || echo "$RESP"; STATUS=failed; break ;;
  esac
done
rm -f "$TL_DIR/device.json"
echo "status=$STATUS"
```

| Result | What to do |
|--------|------------|
| `approved` | Key is in `$TMPDIR/truelist-agent/key`. Go to Step 4. |
| `access_denied` | The user clicked Deny. Stop and ask what they'd like to do. Don't restart unprompted. |
| `expired_token` or `timeout` | 15 minutes passed (or the key was already collected). Ask the user if they want a new link, then start over at Step 1. |
| `invalid_grant` | The device code is unknown. Start over at Step 1. |
| Anything else | Follow `instructions_for_agent` if present; otherwise use the fallback. |

The key is returned exactly once. If you lose it, the user can revoke it in Settings → API keys and you run the flow again.

### Step 4: Store the key (ask first)

For the rest of this session, read the key from the temp file in every command (shell state doesn't persist between commands in most agents):

```bash
TRUELIST_API_KEY="${TRUELIST_API_KEY:-$(cat "${TMPDIR:-/tmp}/truelist-agent/key" 2>/dev/null)}"
```

Then ask the user where to keep it for next time. Offer these, and write nothing until they choose:

1. **Shell profile** (works in every project): append `export TRUELIST_API_KEY=...` to `~/.zshrc` or `~/.bashrc`, whichever their `$SHELL` uses.
   ```bash
   printf '\nexport TRUELIST_API_KEY=%s\n' "$(cat "${TMPDIR:-/tmp}/truelist-agent/key")" >> ~/.zshrc
   ```
2. **Project `.env`** (only this project, e.g. when the app itself will call Truelist): first confirm `.env` is gitignored (`git check-ignore -q .env`). If it isn't, ask before adding it to `.gitignore`. Use `.env.local` instead if that's the project's convention.
   ```bash
   git check-ignore -q .env && printf 'TRUELIST_API_KEY=%s\n' "$(cat "${TMPDIR:-/tmp}/truelist-agent/key")" >> .env
   ```
3. **Their own secret store** (1Password, Doppler, Vercel env, ...). Tell them the key is in the temp file for this session and they can view it in Settings → API keys.
4. **Don't save it.** Use it for this session only.

When the session's work is done (and the key is saved somewhere, or the user chose option 4), delete the temp file: `rm -f "${TMPDIR:-/tmp}/truelist-agent/key"`.

### Check the connection

```bash
curl -s https://api.truelist.io/me -H "Authorization: Bearer $TRUELIST_API_KEY" \
  | jq '{email, plan: .account.payment_plan}'
```

Don't print the whole `/me` response: it includes a `token` field.

### Fallback: copy a key manually

Use this only if the device flow isn't available or the user prefers it:

1. Sign up or log in at https://app.truelist.io/signup (free plan, no credit card).
2. Create a key at https://app.truelist.io/settings/api-keys.
3. Ask the user to set it themselves (`export TRUELIST_API_KEY=...` in their terminal or profile) rather than pasting it into the chat. If they do paste it, don't repeat it back.
