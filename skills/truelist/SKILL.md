---
name: truelist
description: Verify and clean email lists with Truelist, end to end. Use when the user wants to "clean this list", "verify these emails", "check if these emails are valid/real", "verify emails before I import them to Kit / Mailchimp / HubSpot / Klaviyo / ActiveCampaign / Beehiiv / Substack", "remove bad emails from my CSV", "dedupe and validate contacts", "verify a cold outreach / lead list before sending", "why are my emails bouncing", "my bounce rate is too high", "my ESP warned me about bounces", "find invalid / disposable / catch-all emails", or "add email validation to my signup form". Takes a CSV in and returns verified CSVs plus a summary table (deliverability %, bounces avoided). Signs the user up and connects an API key from inside the agent if they don't have one.
---

# Truelist: verify and clean email lists

You help the user stop sending to addresses that bounce. Run whole workflows (not single API calls), teach the right method on the first run, and finish every run with numbers that show what it saved them.

## 0. Pick how to call Truelist

1. **MCP tools loaded?** If tools like `validate_email`, `validate_emails`, `create_batch`, `get_batch`, `get_credit_balance` from a Truelist server are available (usually `mcp__truelist__*`), use them. No key needed.
2. **Otherwise curl** with `TRUELIST_API_KEY`. Check for it without printing it:
   ```bash
   [ -n "$TRUELIST_API_KEY" ] && echo "key set" || { [ -s "${TMPDIR:-/tmp}/truelist-agent/key" ] && echo "session key" || echo "no key"; }
   ```
   Also check a project `.env` / `.env.local` for `TRUELIST_API_KEY=` (grep with `-q`, don't print the line).
3. **No key:** connect the user with agent sign-up (section 1). Don't ask them to go find a key in settings.

Endpoints, response shapes and states: [references/api.md](references/api.md).

## 1. Connect (agent sign-up)

When there's no key, run the device flow in [references/auth.md](references/auth.md):

1. `POST /api/v1/agent_auth/device` → get a link and a code.
2. Show the user the link and code. They approve in the browser, creating a free account if they need one (100 free validations, no credit card).
3. Poll `POST /api/v1/agent_auth/token` until approved, then you have the key.
4. Ask where to save it (shell profile, gitignored `.env`, their secret store, or session only). Write nothing until they choose.

Key rules, always: never print the full key (mask it: first 6 + last 4 chars), never commit it, never write it to a file without asking, and never print raw `GET /me` output (it contains a token).

If the device endpoint isn't available, use the fallback in auth.md (sign up + create a key at app.truelist.io/settings/api-keys) or the hosted MCP server.

## 2. First run: teach the method (STOP and do this once)

The first time you verify anything for this user, before showing results, explain briefly in your own words:

- **States:** `ok` = deliverable. `email_invalid` = will hard-bounce. `risky` = the mailbox may exist but can't be confirmed (catch-all domains, role inboxes like info@, disposable). `unknown` = the check couldn't finish.
- **The method:** verify → **segment** risky into its own group → **suppress** invalid → import the clean list. Don't bulk-delete risky: most are real people on company catch-all domains. Send to them separately and drop the ones that never engage.
- **Batch vs real-time:** batches clean the list you have; real-time validation on signup forms keeps it clean. Offer the second after the first.

Keep it to a few lines. Skip it on later runs.

## 3. Workflows

### a. Verify one or a few emails

Use MCP `validate_email` / `validate_emails`, or `verify_inline` (up to 3 per call):

```bash
curl -s -X POST https://api.truelist.io/api/v1/verify_inline \
  -H "Authorization: Bearer $TRUELIST_API_KEY" \
  -G --data-urlencode "email=jane@acme.com bob@gmial.com" \
  | jq '.emails[] | {address, email_state, email_sub_state, did_you_mean}'
```

Answer with a small table (address, verdict, why, what to do). If `did_you_mean` is set, suggest the correction. For more than ~50 addresses, switch to workflow b.

### b. Clean a list (CSV in → verified CSVs + summary)

Follow [references/clean-list.md](references/clean-list.md):

1. Inspect the CSV: email column, rows, duplicates. Never modify the original.
2. Check credits/plan; if the list won't fit, say so with exact numbers **before** uploading.
3. Upload as a batch (`POST /api/v1/batches` with the file, or MCP `create_batch`), poll `GET /api/v1/batches/<id>` until `completed`, download `annotated_csv_url`.
4. Write `<name>.clean.csv`, `<name>.risky.csv`, `<name>.invalid.csv`, keeping all original columns.
5. End with the summary table: counts and shares per state, **deliverability %**, and **bounces avoided** (invalid count, plus bounce rate before vs after).
6. Offer the next step: import into their ESP (confirm first), set up suppression, or add signup validation.

### c. Debug a bounce problem

Follow [references/bounce-debugging.md](references/bounce-debugging.md). Classify first: dead addresses (verification fixes it) vs authentication or reputation (it doesn't; say so). Verify a sample of bounced addresses to confirm, then run workflow b on the active list.

### d. Add validation to a signup form

Hand off to the `add-truelist` skill in this repo, which detects the stack and installs the SDK (Node, React/Next.js, Ruby, Python, PHP, Go). Recommend: block `email_invalid` and `is_disposable`, show `did_you_mean` as "Did you mean ...?", allow `risky`/`unknown` (don't lose real signups). For WordPress or n8n, point to https://truelist.io/docs/sdks/wordpress and https://truelist.io/docs/sdks/n8n. If the user connected through the device flow, the key they saved is the one the app should use.

## 4. Errors and limits

Follow [references/errors.md](references/errors.md). In short:

- If an error body has `instructions_for_agent`, **do what it says.**
- `quota_exceeded` / `payment_required`: stop, tell the user exactly where things stand (how many done, how many left, where partial results are), share `purchase_url`, offer to resume. **Do not retry.**
- `unauthorized`: offer to reconnect via the device flow.
- `rate_limited`: back off (2s, 4s, 8s) and retry.
- Batch `credits_exhausted`: it resumes on its own after credits are added; tell the user and stop polling.

## Output checklist

- [ ] Original file untouched; outputs saved next to it with clear names
- [ ] Summary table with deliverability % and bounces avoided
- [ ] Risky segmented, not deleted; invalid ready to suppress
- [ ] Full API key never shown
- [ ] One suggested next step
