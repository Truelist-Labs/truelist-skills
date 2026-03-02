# Node.js / TypeScript Patterns

## Installation

```bash
npm install truelist
```

## Basic Usage

```ts
import Truelist from "truelist";

const truelist = new Truelist(process.env.TRUELIST_API_KEY!);
const result = await truelist.email.validate("user@example.com");

console.log(result.state);      // "ok" | "email_invalid" | "risky" | "unknown" | "accept_all"
console.log(result.subState);   // "email_ok" | "is_disposable" | "is_role" | ...
console.log(result.suggestion); // null or "user@gmail.com"
console.log(result.domain);     // "example.com"
console.log(result.verifiedAt); // "2026-02-21T10:00:00.000Z"
```

## Convenience Methods

```ts
import { isValid, isInvalid, isDisposable, isRole } from "truelist";

const result = await truelist.email.validate("user@example.com");

isValid(result);      // true if state === "ok"
isInvalid(result);    // true if state === "email_invalid"
isDisposable(result); // true if subState === "is_disposable"
isRole(result);       // true if subState === "is_role"
```

## Error Handling

```ts
import Truelist, {
  AuthenticationError,
  RateLimitError,
  ApiError,
  TruelistError,
} from "truelist";

try {
  const result = await truelist.email.validate("user@example.com");
} catch (error) {
  if (error instanceof AuthenticationError) {
    // Invalid API key (401)
  } else if (error instanceof RateLimitError) {
    // Rate limited (429) — retries exhausted
    console.error("Retry after:", error.retryAfter);
  } else if (error instanceof ApiError) {
    // Other API error (4xx/5xx)
    console.error("Status:", error.status, "Body:", error.body);
  } else if (error instanceof TruelistError) {
    // Timeout, network error, or cancelled request
  }
}
```

## TypeScript Types

```ts
import type {
  ValidationResult,
  ValidationState,
  ValidationSubState,
  AccountInfo,
  TruelistOptions,
  ValidateOptions,
} from "truelist";
```

## Configuration Options

```ts
const truelist = new Truelist(process.env.TRUELIST_API_KEY!, {
  baseUrl: "https://api.truelist.io", // default
  timeout: 30_000,                     // default: 30s
  maxRetries: 2,                       // default: 2
});
```

## Express Middleware Example

```ts
import Truelist, { isValid } from "truelist";
import { Request, Response, NextFunction } from "express";

const truelist = new Truelist(process.env.TRUELIST_API_KEY!);

async function validateEmail(req: Request, res: Response, next: NextFunction) {
  const { email } = req.body;
  if (!email) return res.status(400).json({ error: "Email required" });

  const result = await truelist.email.validate(email);

  if (!isValid(result)) {
    return res.status(422).json({
      error: "Invalid email",
      state: result.state,
      suggestion: result.suggestion,
    });
  }

  next();
}
```

## Abort / Cancel a Request

```ts
const controller = new AbortController();
setTimeout(() => controller.abort(), 5000);

const result = await truelist.email.validate("user@example.com", {
  signal: controller.signal,
});
```

## Check Account Info

```ts
const account = await truelist.account.get();
console.log(account.email);               // "you@company.com"
console.log(account.account.paymentPlan); // "pro"
```
