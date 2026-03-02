---
name: validate-email
description: Validate email addresses using the Truelist API. Check deliverability, detect disposable/role addresses, get typo suggestions, and verify emails in bulk.
---

# Validate Email

Validate email addresses for deliverability using the Truelist API.

## Prerequisites

Set the `TRUELIST_API_KEY` environment variable:

```bash
export TRUELIST_API_KEY=your-api-key
```

Get an API key at [truelist.io](https://truelist.io).

## Validate a Single Email

```bash
curl -s -X POST "https://api.truelist.io/api/v1/verify_inline?email=user@example.com" \
  -H "Authorization: Bearer $TRUELIST_API_KEY" | jq
```

**Response:**

```json
{
  "emails": [
    {
      "address": "user@example.com",
      "domain": "example.com",
      "canonical": "user@example.com",
      "mx_record": "mx.example.com",
      "first_name": null,
      "last_name": null,
      "email_state": "ok",
      "email_sub_state": "email_ok",
      "verified_at": "2026-02-21T10:39:12.570Z",
      "did_you_mean": null
    }
  ]
}
```

## Interpreting Results

### States

| State | Meaning | Action |
|-------|---------|--------|
| `ok` | Deliverable | Safe to send |
| `email_invalid` | Not deliverable | Do not send |
| `risky` | May be deliverable but carries risk | Send with caution |
| `accept_all` | Mail server accepts all addresses | Deliverability uncertain |
| `unknown` | Could not determine | Retry later or skip |

### Sub-states

| Sub-state | Meaning |
|-----------|---------|
| `email_ok` | Passed all checks |
| `is_disposable` | Temporary/disposable email provider |
| `is_role` | Role-based address (info@, admin@, support@) |
| `failed_smtp_check` | SMTP verification failed |
| `failed_mx_check` | Domain has no mail server |
| `failed_spam_trap` | Known spam trap address |
| `failed_no_mailbox` | Mailbox does not exist |
| `failed_greylisted` | Server temporarily rejected |
| `failed_syntax_check` | Invalid email syntax |
| `unknown_error` | Could not determine sub-state |

## Typo Suggestions

When the API detects a likely typo, `did_you_mean` contains a suggestion:

```bash
curl -s -X POST "https://api.truelist.io/api/v1/verify_inline?email=user@gmial.com" \
  -H "Authorization: Bearer $TRUELIST_API_KEY" | jq '.emails[0].did_you_mean'
# "user@gmail.com"
```

Always check `did_you_mean` and prompt the user if non-null.

## Batch Validation

Validate multiple emails by posting a JSON array:

```bash
curl -s -X POST "https://api.truelist.io/api/v1/verify" \
  -H "Authorization: Bearer $TRUELIST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"emails": ["user1@example.com", "user2@example.com"]}' | jq
```

This endpoint processes emails in the background. Use inline validation (`/api/v1/verify_inline`) for real-time, synchronous results.

## Check Account and Credits

```bash
curl -s "https://api.truelist.io/me" \
  -H "Authorization: Bearer $TRUELIST_API_KEY" | jq
```

**Response:**

```json
{
  "email": "team@company.com",
  "name": "Team Lead",
  "uuid": "a3828d19-1234-5678-9abc-def012345678",
  "time_zone": "America/New_York",
  "is_admin_role": true,
  "account": {
    "name": "Company Inc",
    "payment_plan": "pro"
  }
}
```

## Rate Limits

All endpoints are limited to **10 requests per second**. The API returns `429 Too Many Requests` when exceeded. Wait and retry with backoff.

## Decision Flowchart

When validating an email, follow this logic:

1. Call `/api/v1/verify_inline` with the email
2. Check `email_state`:
   - `ok` → Accept the email
   - `email_invalid` → Reject the email, show error
   - `risky` → Check `email_sub_state`:
     - `is_disposable` → Reject or warn (disposable address)
     - `is_role` → Accept with caution (role-based address)
   - `accept_all` → Accept with caution (server accepts everything)
   - `unknown` → Accept with caution or retry later
3. Check `did_you_mean`:
   - If non-null → Suggest the correction to the user

## Common Patterns

### Quick check in a script

```bash
STATE=$(curl -s -X POST "https://api.truelist.io/api/v1/verify_inline?email=$EMAIL" \
  -H "Authorization: Bearer $TRUELIST_API_KEY" | jq -r '.emails[0].email_state')

if [ "$STATE" = "ok" ]; then
  echo "Valid"
else
  echo "Invalid or risky: $STATE"
fi
```

### Validate a list from a file

```bash
while IFS= read -r email; do
  result=$(curl -s -X POST "https://api.truelist.io/api/v1/verify_inline?email=$email" \
    -H "Authorization: Bearer $TRUELIST_API_KEY")
  state=$(echo "$result" | jq -r '.emails[0].email_state')
  echo "$email: $state"
  sleep 0.1  # respect rate limits
done < emails.txt
```
