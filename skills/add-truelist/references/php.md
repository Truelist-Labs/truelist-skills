# PHP Patterns

## Installation

```bash
composer require truelist/truelist
```

## Basic Usage

```php
<?php

use Truelist\Client;

$client = new Client(getenv('TRUELIST_API_KEY'));
$result = $client->validate('user@example.com');

echo $result->state;       // "ok" | "email_invalid" | "risky" | ...
echo $result->subState;    // "email_ok" | "is_disposable" | ...
echo $result->suggestion;  // null or "user@gmail.com"
echo $result->isValid();   // true if state === "ok"
```

## Laravel Form Request

```php
<?php

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;
use Truelist\Client;

class DeliverableEmail implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $client = new Client(config('services.truelist.api_key'));
        $result = $client->validate($value);

        if ($result->state === 'email_invalid') {
            $fail('This email address is not deliverable.');
        }

        if ($result->subState === 'is_disposable') {
            $fail('Disposable email addresses are not allowed.');
        }

        if ($result->suggestion) {
            $fail("Did you mean {$result->suggestion}?");
        }
    }
}
```

```php
// In a Form Request or controller:
$request->validate([
    'email' => ['required', 'email', new DeliverableEmail],
]);
```

## Laravel Controller Example

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Truelist\Client;

class EmailValidationController extends Controller
{
    public function validate(Request $request)
    {
        $request->validate(['email' => 'required|email']);

        $client = new Client(config('services.truelist.api_key'));
        $result = $client->validate($request->email);

        return response()->json([
            'valid' => $result->isValid(),
            'state' => $result->state,
            'suggestion' => $result->suggestion,
        ]);
    }
}
```

## Environment Setup

```php
// config/services.php
'truelist' => [
    'api_key' => env('TRUELIST_API_KEY'),
],
```

```bash
# .env
TRUELIST_API_KEY=your-api-key
```
