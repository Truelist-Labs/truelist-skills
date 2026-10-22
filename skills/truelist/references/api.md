# Truelist API reference (for the workflows)

Base URL `https://api.truelist.io`. Auth: `Authorization: Bearer $TRUELIST_API_KEY`. Rate limit: 10 requests/second per endpoint (429 when exceeded).

## MCP tools vs curl

If Truelist MCP tools are loaded (names end in `validate_email`, `validate_emails`, `create_batch`, `get_batch`, `get_credit_balance`, ... and usually start with `mcp__truelist__`), prefer them. They handle auth for you. Otherwise use curl with the key.

| Job | Hosted MCP tool (`https://api.truelist.io/mcp`) | curl |
|-----|-----------------|------|
| One address | `validate_email` | `POST /api/v1/verify_inline` |
| A few addresses | `validate_emails` (up to 50) | `POST /api/v1/verify_inline` (up to 3 per call) |
| A list / CSV | `create_batch` (up to 10,000, JSON emails), then `get_batch` | `POST /api/v1/batches` (file upload), then `GET /api/v1/batches/<id>` |
| Credits left | `get_credit_balance` | none; `GET /me` gives the plan only |
| Account / plan | `check_account` | `GET /me` |
| Past results | `list_batches`, `list_email_addresses` | `GET /api/v1/batches` |

The older npm package (`npx truelist-mcp`) exposes only `validate_email`, `validate_emails` and `check_account`; use curl for lists in that case. MCP tools return `state` / `sub_state` / `suggestion` where the REST API returns `email_state` / `email_sub_state` / `did_you_mean`.

## Verify inline: `POST /api/v1/verify_inline`

Synchronous. Takes `email` as a query parameter; up to **3 addresses per call**, space-separated. Optional `validation_strategy` (`accurate` default, `fast`, `thorough`, `enhanced`) and `checks[]` (`syntax`, `mx`, `disposable`, `role`, `smtp`).

```bash
curl -s -X POST "https://api.truelist.io/api/v1/verify_inline" \
  -H "Authorization: Bearer $TRUELIST_API_KEY" \
  -G --data-urlencode "email=jane@acme.com bob@gmial.com" \
  | jq '.emails[] | {address, email_state, email_sub_state, did_you_mean}'
```

Response:

```json
{
  "emails": [
    {
      "address": "jane@acme.com",
      "domain": "acme.com",
      "canonical": "jane",
      "mx_record": "aspmx.l.google.com",
      "first_name": null,
      "last_name": null,
      "email_state": "ok",
      "email_sub_state": "email_ok",
      "verified_at": "2026-10-22T10:39:12.570Z",
      "did_you_mean": null
    }
  ]
}
```

A 422 means one of the addresses failed syntax (`"Invalid email '...'"`); nothing was verified, so drop or fix that address and retry the rest.

## Batches (lists)

### Upload: `POST /api/v1/batches`

From a CSV file (the email column is detected automatically; other columns are kept in the annotated output):

```bash
curl -s -X POST https://api.truelist.io/api/v1/batches \
  -H "Authorization: Bearer $TRUELIST_API_KEY" \
  -F "file=@contacts.csv" | jq '{id, batch_state, email_count, error}'
```

From a JSON list (when you don't have a file): `data` is an array of one-element arrays.

```bash
curl -s -X POST https://api.truelist.io/api/v1/batches \
  -H "Authorization: Bearer $TRUELIST_API_KEY" \
  -F 'data=[["a@example.com"],["b@example.com"]]' -F "filename=from-agent.csv"
```

Optional fields: `validation_strategy`, `webhook_url`.

Errors: 422 for a bad CSV or more rows than the plan's per-file limit (the message says the limit); 413 when the file is too large (split it); 409 when the same list was already uploaded (`existing_batch.id` is the batch to use).

### Poll: `GET /api/v1/batches/<id>`

```bash
curl -s https://api.truelist.io/api/v1/batches/$BATCH_ID \
  -H "Authorization: Bearer $TRUELIST_API_KEY" \
  | jq '{batch_state, email_count, processed_count}'
```

`batch_state` goes `pending` → `pending_processing` → `processing` → `finalizing` → `completed`. `credits_exhausted` means it paused for lack of credits (see [errors.md](errors.md)). Poll every 5–10 seconds; slow down to 30 seconds for lists over ~10k.

Fields on a completed batch: `email_count`, `ok_count`, `ok_for_all_count` (accept-all), `disposable_count`, `role_count`, `failed_syntax_check_count`, `failed_mx_check_count`, `failed_no_mailbox_count`, `duplicateCount`, and four download URLs:

| Field | Contents |
|-------|----------|
| `annotated_csv_url` | The full original list with Truelist result columns added. Use this for segmenting. |
| `safest_bet_csv_url` | Only addresses confirmed deliverable. |
| `highest_reach_csv_url` | Deliverable plus some risky addresses (larger list, slightly higher bounce risk). Build your own segments from the annotated CSV when the user needs to know exactly what's in each. |
| `only_invalid_csv_url` | Only the undeliverable addresses (your suppression list). |

The URLs can be `null` for a few seconds after `completed` while the CSVs are generated; keep polling until `annotated_csv_url` is set.

### Download

```bash
curl -sL -H "Authorization: Bearer $TRUELIST_API_KEY" -o contacts.truelist.csv "$ANNOTATED_URL"
head -1 contacts.truelist.csv   # inspect the result column names before segmenting
```

### Other

- `GET /api/v1/batches` lists batches (filters: `batch_state`, `search`).
- `DELETE /api/v1/batches/<id>` deletes a batch.

## Account: `GET /me`

Returns the user and `account.payment_plan`. The response also contains a `token` and `api_keys`, so always filter it with jq and never print it raw.

## States

| `email_state` | Meaning |
|---------------|---------|
| `ok` | Deliverable |
| `email_invalid` | Will bounce. Suppress. |
| `risky` | Mailbox may exist but carries risk (see sub-state) |
| `unknown` | Couldn't be determined (timeouts, greylisting) |

Some responses and docs show `accept_all` as a state; treat it the same as `risky` + `accept_all`.

| `email_sub_state` | Usually under | Meaning |
|-------------------|---------------|---------|
| `email_ok` | ok | Passed every check |
| `accept_all` | risky | Catch-all domain: accepts every address, so the mailbox can't be confirmed |
| `is_disposable` | risky | Throwaway provider (mailinator etc.) |
| `is_role` | risky / ok | Role address (info@, sales@) |
| `failed_no_mailbox` | email_invalid | Mailbox doesn't exist |
| `failed_mx_check` | email_invalid | Domain can't receive mail |
| `failed_syntax_check` | email_invalid | Not a valid address |
| `failed_smtp_check` | email_invalid | Server rejected it |
| `failed_spam_trap` | email_invalid | Known spam trap. Never send. |
| `failed_greylisted` | unknown | Server deferred the check |
| `timed_out`, `unknown_error` | unknown | Couldn't finish the check |

## Credits

Standard validation costs 1 credit; `unknown` and `accept_all` results are free; `enhanced` costs 3 only when it resolves an otherwise uncertain result. The free plan includes 100 validations plus 10 enhanced credits and a smaller per-file row limit. Unlimited plans don't spend credits on standard validations.
