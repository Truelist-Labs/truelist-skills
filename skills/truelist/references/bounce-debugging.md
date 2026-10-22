# Why are my emails bouncing?

Work out which kind of bounce problem this is before recommending anything. List verification fixes hard bounces from bad addresses; it doesn't fix authentication or reputation problems, and saying it does will lose the user's trust.

## 1. Ask for (or find) the evidence

- Bounce rate and over what period, and which ESP / sending tool.
- Hard vs soft bounces, if the ESP splits them.
- A sample of bounce messages (SMTP codes and text), or an export of bounced contacts.
- Where the list came from: signup form, purchased/scraped, old import, cold outreach.

## 2. Classify

| Signal | Likely cause | Fix |
|--------|--------------|-----|
| `550 5.1.1` "user unknown", "mailbox not found", "no such user" | Dead or mistyped addresses | Verify the list, suppress invalid ([clean-list.md](clean-list.md)) |
| Bounces concentrated on one old import or a purchased/scraped list | List decay or bad source | Verify that segment first |
| Typos like `gmial.com`, `hotmal.com` | Signup form takes anything | Real-time validation on the form (`add-truelist` skill), and fix the existing ones using `did_you_mean` |
| Lots of mailinator-style domains | Disposable signups | Block disposable at signup |
| `5.7.x`, "DMARC", "SPF", "DKIM", "not authenticated" | Authentication | Fix DNS records (SPF/DKIM/DMARC); verification won't help |
| "blocked", "spam", "reputation", blocklist names (Spamhaus, ...) | Sender reputation | Slow down, warm up, clean the list, check blocklists |
| `4.x.x`, "mailbox full", "try again later", greylisting | Soft bounces | Usually temporary; suppress only if they repeat across several sends |

If the user has the bounced addresses, verify a sample (20–50) inline. If most come back `email_invalid` / `failed_no_mailbox`, the list is the problem: verify the whole active list next.

## 3. Report

Give a short diagnosis with the numbers:

> Your 6.1% bounce rate is almost all hard bounces ("550 user unknown") from the March webinar import. I checked 40 of the bounced addresses and 37 don't exist. Your SPF and DKIM look fine in the bounce headers.
> Next: verify the full list (about 8,400 contacts) and suppress the invalid ones before your next send. Want me to start?

Then run the clean-list workflow. When it's done, compare: "bounce rate on the raw list would have been X%, the clean file should be under 1%."

If the cause is authentication or reputation, say so plainly and point to the fix, and offer verification as a secondary step only.

## 4. Prevent it next time

- Validate at signup (`add-truelist` skill: Node, React/Next.js, Ruby, Python, PHP, Go SDKs; WordPress and n8n integrations also exist).
- Re-verify the list before big campaigns and every few months (lists decay roughly 2% a month).
- Verify cold-outreach lists right before each sequence, not when they were built.
