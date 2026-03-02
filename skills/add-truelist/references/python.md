# Python Patterns

## Installation

```bash
pip install truelist
```

## Basic Usage

```python
import os
from truelist import Truelist

client = Truelist(api_key=os.environ["TRUELIST_API_KEY"])
result = client.validate("user@example.com")

print(result.state)       # "ok" | "email_invalid" | "risky" | "unknown" | "accept_all"
print(result.sub_state)   # "email_ok" | "is_disposable" | "is_role" | ...
print(result.suggestion)  # None or "user@gmail.com"
print(result.is_valid)    # True if state == "ok"
```

## Django Form Validator

```python
# validators.py
from truelist import Truelist
from django.core.exceptions import ValidationError
from django.conf import settings

_client = Truelist(api_key=settings.TRUELIST_API_KEY)

def validate_email_deliverable(email):
    result = _client.validate(email)

    if result.state == "email_invalid":
        raise ValidationError(f"This email address is not deliverable.")

    if result.sub_state == "is_disposable":
        raise ValidationError("Disposable email addresses are not allowed.")

    if result.suggestion:
        raise ValidationError(f"Did you mean {result.suggestion}?")
```

```python
# models.py
from django.db import models
from .validators import validate_email_deliverable

class User(models.Model):
    email = models.EmailField(validators=[validate_email_deliverable])
```

## Django View Example

```python
# views.py
from truelist import Truelist
from django.http import JsonResponse
from django.conf import settings

client = Truelist(api_key=settings.TRUELIST_API_KEY)

def validate_email_view(request):
    email = request.POST.get("email")
    if not email:
        return JsonResponse({"error": "Email required"}, status=400)

    result = client.validate(email)

    return JsonResponse({
        "valid": result.is_valid,
        "state": result.state,
        "suggestion": result.suggestion,
    })
```

## Flask Example

```python
from flask import Flask, request, jsonify
from truelist import Truelist
import os

app = Flask(__name__)
client = Truelist(api_key=os.environ["TRUELIST_API_KEY"])

@app.post("/validate-email")
def validate_email():
    email = request.json.get("email")
    if not email:
        return jsonify(error="Email required"), 400

    result = client.validate(email)

    return jsonify(
        valid=result.is_valid,
        state=result.state,
        suggestion=result.suggestion,
    )
```

## Batch Validation

```python
import time

emails = ["user1@example.com", "user2@example.com", "user3@example.com"]

for email in emails:
    result = client.validate(email)
    print(f"{email}: {result.state}")
    time.sleep(0.1)  # respect rate limits
```
