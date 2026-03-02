---
name: add-truelist
description: Add Truelist email validation to an existing project. Detects the tech stack and installs the right SDK with idiomatic integration code.
---

# Add Truelist Email Validation

Add email validation to an existing project using the Truelist SDK.

## Step 1: Detect the Tech Stack

Check for these files to determine the project's stack:

| File | Stack | SDK |
|------|-------|-----|
| `package.json` | Node.js / TypeScript | `truelist` (npm) |
| `Gemfile` | Ruby / Rails | `truelist` (gem) |
| `requirements.txt` or `pyproject.toml` | Python | `truelist` (pip) |
| `go.mod` | Go | `github.com/Truelist-io-Email-Validation/truelist-go` |
| `composer.json` | PHP | `truelist/truelist` (composer) |

If the project uses React/Next.js (check for `next` in `package.json` dependencies), use the React patterns in `references/react.md`.

## Step 2: Install the SDK

### Node.js / TypeScript

```bash
npm install truelist
```

### Ruby

```bash
bundle add truelist
```

### Python

```bash
pip install truelist
```

### Go

```bash
go get github.com/Truelist-io-Email-Validation/truelist-go
```

### PHP

```bash
composer require truelist/truelist
```

## Step 3: Set the API Key

Add `TRUELIST_API_KEY` to the project's environment:

```bash
# .env or .env.local
TRUELIST_API_KEY=your-api-key
```

Get an API key at [truelist.io](https://truelist.io).

## Step 4: Integrate

### Node.js / TypeScript (Quick Start)

```ts
import Truelist, { isValid, isDisposable } from "truelist";

const truelist = new Truelist(process.env.TRUELIST_API_KEY!);

const result = await truelist.email.validate("user@example.com");

if (isValid(result)) {
  // Email is deliverable
}

if (isDisposable(result)) {
  // Reject disposable emails
}

if (result.suggestion) {
  // Suggest correction: result.suggestion
}
```

See `references/node.md` for full patterns including error handling and TypeScript types.

### Ruby / Rails (Quick Start)

```ruby
require "truelist"

client = Truelist::Client.new(api_key: ENV["TRUELIST_API_KEY"])
result = client.validate("user@example.com")

if result.valid?
  # Email is deliverable
end
```

See `references/ruby.md` for Rails model validation and controller patterns.

### Python (Quick Start)

```python
from truelist import Truelist

client = Truelist(api_key=os.environ["TRUELIST_API_KEY"])
result = client.validate("user@example.com")

if result.is_valid:
    # Email is deliverable
```

See `references/python.md` for Django/Flask integration patterns.

### React / Next.js

For form validation with server actions and client components, see `references/react.md`.

### Go / PHP

See `references/go.md` and `references/php.md` for integration patterns.

## Validation States Reference

Every validation returns a `state` and `subState`:

| State | Meaning | Action |
|-------|---------|--------|
| `ok` | Deliverable | Accept |
| `email_invalid` | Not deliverable | Reject |
| `risky` | May be deliverable | Accept with caution |
| `accept_all` | Server accepts everything | Accept with caution |
| `unknown` | Could not determine | Retry or accept |

Key sub-states to handle:
- `is_disposable` — Temporary/throwaway email. Block on signup forms.
- `is_role` — Role address (info@, admin@). May want to allow or flag.
- `failed_smtp_check` — SMTP check failed. Treat as invalid.

## Where to Validate

- **Signup forms** — Validate before creating the account. Block `email_invalid` and `is_disposable`.
- **Contact forms** — Validate to reduce spam. Show `did_you_mean` suggestions.
- **Import/upload** — Validate in bulk before processing. Use the batch endpoint or loop with rate limiting.
- **API endpoints** — Validate incoming emails in your request handler.
