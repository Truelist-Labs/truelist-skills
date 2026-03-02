# React / Next.js Patterns

## Installation

```bash
npm install truelist
```

## Server Action for Email Validation

```ts
// app/actions/validate-email.ts
"use server";

import Truelist, { isValid, isDisposable } from "truelist";

const truelist = new Truelist(process.env.TRUELIST_API_KEY!);

type ValidationResponse = {
  valid: boolean;
  error?: string;
  suggestion?: string;
};

export async function validateEmail(email: string): Promise<ValidationResponse> {
  const result = await truelist.email.validate(email);

  if (isDisposable(result)) {
    return { valid: false, error: "Disposable email addresses are not allowed." };
  }

  if (!isValid(result) && result.state === "email_invalid") {
    const error = result.suggestion
      ? `This email is not valid. Did you mean ${result.suggestion}?`
      : "This email address is not valid.";
    return { valid: false, error, suggestion: result.suggestion ?? undefined };
  }

  if (result.suggestion) {
    return { valid: true, suggestion: result.suggestion };
  }

  return { valid: true };
}
```

## Client Component with Validation

```tsx
// components/email-input.tsx
"use client";

import { useState } from "react";
import { validateEmail } from "@/app/actions/validate-email";

export function EmailInput() {
  const [email, setEmail] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [suggestion, setSuggestion] = useState<string | null>(null);
  const [validating, setValidating] = useState(false);

  async function handleBlur() {
    if (!email) return;
    setValidating(true);
    setError(null);
    setSuggestion(null);

    const result = await validateEmail(email);

    if (!result.valid && result.error) {
      setError(result.error);
    }

    if (result.suggestion) {
      setSuggestion(result.suggestion);
    }

    setValidating(false);
  }

  function applySuggestion() {
    if (suggestion) {
      setEmail(suggestion);
      setSuggestion(null);
      setError(null);
    }
  }

  return (
    <div>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        onBlur={handleBlur}
        placeholder="you@example.com"
        className="border rounded px-3 py-2 w-full"
      />
      {validating && <p className="text-sm text-gray-500 mt-1">Validating...</p>}
      {error && <p className="text-sm text-red-600 mt-1">{error}</p>}
      {suggestion && (
        <button
          type="button"
          onClick={applySuggestion}
          className="text-sm text-blue-600 mt-1 underline"
        >
          Did you mean {suggestion}?
        </button>
      )}
    </div>
  );
}
```

## API Route (if not using Server Actions)

```ts
// app/api/validate-email/route.ts
import { NextResponse } from "next/server";
import Truelist, { isValid } from "truelist";

const truelist = new Truelist(process.env.TRUELIST_API_KEY!);

export async function POST(req: Request) {
  const { email } = await req.json();

  if (!email || typeof email !== "string") {
    return NextResponse.json({ error: "Email required" }, { status: 400 });
  }

  const result = await truelist.email.validate(email);

  return NextResponse.json({
    valid: isValid(result),
    state: result.state,
    suggestion: result.suggestion,
  });
}
```

## Signup Form with Validation

```tsx
// app/signup/page.tsx
"use client";

import { useState } from "react";
import { validateEmail } from "@/app/actions/validate-email";

export default function SignupPage() {
  const [email, setEmail] = useState("");
  const [emailError, setEmailError] = useState<string | null>(null);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();

    const validation = await validateEmail(email);
    if (!validation.valid) {
      setEmailError(validation.error ?? "Invalid email");
      return;
    }

    // Proceed with signup...
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={(e) => {
          setEmail(e.target.value);
          setEmailError(null);
        }}
        placeholder="you@example.com"
      />
      {emailError && <p className="text-red-600 text-sm">{emailError}</p>}
      <button type="submit">Sign up</button>
    </form>
  );
}
```
