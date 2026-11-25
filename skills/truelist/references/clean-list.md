# Clean a list: CSV in, verified CSVs out

Goal: the user ends up with files they can import or suppress right away, plus one summary table showing what verification saved them.

## 1. Inspect the file

- Find the email column (header like `email`, `Email Address`, `e-mail`; otherwise the column whose values contain `@`).
- Count rows, blank emails, and duplicates (case-insensitive). Tell the user the numbers before spending anything.
- Never modify the original file. Every output gets a new name next to it.

```bash
python3 - contacts.csv <<'EOF'
import csv, sys
rows = list(csv.DictReader(open(sys.argv[1], newline='', encoding='utf-8-sig')))
col = next((c for c in rows[0] if 'mail' in c.lower()), None) or next(c for c in rows[0] if '@' in (rows[0][c] or ''))
emails = [r[col].strip().lower() for r in rows if r[col].strip()]
print(f"column={col!r} rows={len(rows)} with_email={len(emails)} unique={len(set(emails))}")
EOF
```

## 2. Check there's room before you start

- With MCP: call `get_credit_balance` and compare `available_credits` with the unique count.
- With curl: `GET /me` shows the plan. Free plans have 100 validations and a small per-file row limit.

If the list is bigger than what's left, say so **before** uploading, with the exact numbers, and give the choice:

> This list has 2,552 unique addresses and your free plan has 100 validations left. I can verify the first 100 now so you can see the results, or you can add credits first (https://app.truelist.io/settings/billing) and I'll do all of them.

## 3. Verify

Pick by size:

| Unique addresses | Use |
|------------------|-----|
| up to ~50 | MCP `validate_emails`, or `verify_inline` 3 at a time (stay under 10 requests/second) |
| more | A batch: upload the CSV with `POST /api/v1/batches`, or MCP `create_batch` with the deduped email list |

Batch flow (see [api.md](api.md) for details):

```bash
BATCH_ID=$(curl -s -X POST https://api.truelist.io/api/v1/batches \
  -H "Authorization: Bearer $TRUELIST_API_KEY" -F "file=@contacts.csv" | tee /dev/stderr | jq -r '.id // empty')
[ -n "$BATCH_ID" ] || { echo "upload failed: see the error above"; exit 1; }

while :; do
  B=$(curl -s https://api.truelist.io/api/v1/batches/$BATCH_ID -H "Authorization: Bearer $TRUELIST_API_KEY")
  STATE=$(echo "$B" | jq -r .batch_state); URL=$(echo "$B" | jq -r '.annotated_csv_url // empty')
  echo "$STATE $(echo "$B" | jq -r '"\(.processed_count)/\(.email_count)"')"
  [ "$STATE" = completed ] && [ -n "$URL" ] && break
  [ "$STATE" = credits_exhausted ] && break
  sleep 10
done

curl -sL -H "Authorization: Bearer $TRUELIST_API_KEY" -o contacts.truelist.csv "$URL"
```

If the upload returns an error, handle it per [errors.md](errors.md) (409 → reuse `existing_batch.id`). If the batch hits `credits_exhausted`, stop and tell the user. It resumes by itself once they add credits.

Give the user a progress line for long lists (e.g. every minute: "1,200 / 2,552 checked").

## 4. Segment (don't just delete)

Split into these files from the annotated CSV (or from your inline results), keeping every original column:

| File | Rows | What to do with it |
|------|------|--------------------|
| `<name>.clean.csv` | `ok` (including role addresses, flagged) | Import and send |
| `<name>.risky.csv` | `risky` and `unknown` | Import into a separate segment/tag. Don't blast it; send to it gently or re-verify later. |
| `<name>.invalid.csv` | `email_invalid` | Add to the ESP's suppression list. Never import as subscribers. |

Inspect the annotated CSV header first (`head -1`) and use whatever result columns Truelist added (they contain the state and sub-state). Add a `truelist_state` / `truelist_sub_state` column to each output if the user's ESP should be able to filter on it.

Why not delete risky? Most risky addresses are real people on catch-all company domains or role inboxes. Deleting them throws away reach; blasting them raises bounces. A separate segment lets the user send to engaged ones and drop the rest after a campaign or two. Explain this the first time.

Disposable addresses (`is_disposable`) and spam traps (`failed_spam_trap`) are the exception: recommend suppressing them along with invalid.

## 5. Summary table (every run)

Always end with this, filled in with real numbers:

```
| Result                       | Count | Share  |
|------------------------------|------:|-------:|
| Deliverable (ok)             | 2,104 | 82.4%  |
| Risky (catch-all, role, ...) |   301 | 11.8%  |
| Unknown                      |    39 |  1.5%  |
| Invalid (will bounce)        |   108 |  4.2%  |
| Duplicates removed           |    17 |        |
| Total unique checked         | 2,552 |        |

Deliverability: 82.4% confirmed deliverable
Bounces avoided: ~108 hard bounces (4.2% bounce rate -> under 1% on the clean file)
```

How to compute:
- **Deliverability %** = ok / unique checked.
- **Bounces avoided** = invalid count. That's how many hard bounces sending the raw list would have caused.
- **Bounce rate before** = invalid / unique checked. Mailbox providers start filtering above ~2%, and most ESPs (Kit, Mailchimp, ...) review or suspend accounts around 5%. Mention this when the "before" number is over 2%.
- Never claim the clean file has a 0% bounce rate. Say "under 1%" for the ok-only file.

Then list the output files with what to do with each, and offer the next step (import into their ESP, set up a suppression list, or add real-time validation to their signup form so the list stays clean; see the `add-truelist` skill).

## Importing into an ESP

If an ESP's MCP tools or API are available (Kit, Mailchimp, HubSpot, ...), offer to do the import: clean file as subscribers, risky file as a tagged segment, invalid file as suppressions/unsubscribes. Always confirm with the user before writing to their ESP, and say how many contacts will be touched.
