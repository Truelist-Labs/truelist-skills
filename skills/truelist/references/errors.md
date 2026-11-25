# Handling Truelist API errors

Every error body has a human-readable `error`. Quota, plan, auth and rate-limit errors also carry fields written for agents:

```json
{
  "error": "You have used all of your free validations. ...",
  "error_code": "quota_exceeded",
  "purchase_url": "https://app.truelist.io/settings/billing",
  "instructions_for_agent": "Tell the user their free validations are used up and share purchase_url. Do not retry this request."
}
```

## The rule

1. If the body has `instructions_for_agent`, **follow it**. It takes priority over the defaults below.
2. Otherwise, handle by `error_code` (or HTTP status when `error_code` is absent):

| `error_code` | Status | What to tell the user | Retry? |
|--------------|--------|------------------------|--------|
| `quota_exceeded` | 403 | Their free validations (or credits) are used up. Quote `error` for the exact numbers. Share `purchase_url`. | **No** |
| `payment_required` | 402 | Their account has a payment issue. Share `purchase_url` so they can update the payment method. | **No** |
| `unauthorized` | 401 | The key is missing, invalid, or revoked. Offer to reconnect with the device flow ([auth.md](auth.md)). | After reconnecting |
| `rate_limited` | 429 | Nothing, unless it keeps happening. | Yes, with backoff: wait 2s, 4s, 8s; give up after 3 tries |
| (none) | 408 | That address timed out. Mark it `unknown` in your results and move on. | Once, later |
| (none) | 409 on `POST /api/v1/batches` | The same list was already uploaded. Use `existing_batch.id` instead of re-uploading. | No |
| (none) | 422 | The input was rejected (bad email syntax, bad CSV, too many rows for the plan). Show `error` and fix the input. | After fixing |
| (none) | 5xx | Truelist had a problem. | Once, after a short wait |

## Quota and payment errors: what good looks like

These are the moments the user decides whether to pay, so be exact and calm. Don't apologize at length, don't hide the link, don't retry in a loop.

> Your free Truelist validations are used up, so I stopped after 212 of 2,552 addresses. The 212 results so far are saved in `contacts.partial.csv`.
> To finish the remaining 2,340, add credits here: https://app.truelist.io/settings/billing
> Once that's done, tell me and I'll pick up where I left off.

Always:
- Say how far you got and where partial results are.
- Show the exact remaining count when you know it.
- Share `purchase_url` (fall back to https://app.truelist.io/settings/billing if it's missing).
- Offer to resume, and resume only the unprocessed rows.

## Batches that run out of credits

A batch doesn't fail when credits run out: `batch_state` becomes `credits_exhausted` and it waits. Stop polling, tell the user (same pattern as above), and explain that the batch **resumes automatically** after they buy credits. When they say they're done, poll `GET /api/v1/batches/<id>` again.

## Before the agent-native fields ship

Older API responses have only `error` (no `error_code`, `purchase_url` or `instructions_for_agent`). Fall back to the HTTP status column above and use https://app.truelist.io/settings/billing as the purchase link. The free-quota error is a 403 whose `error` starts with "You have used all of your free validations".
