# Go Patterns

## Installation

```bash
go get github.com/Truelist-io-Email-Validation/truelist-go
```

## Basic Usage

```go
package main

import (
	"fmt"
	"os"

	"github.com/Truelist-io-Email-Validation/truelist-go"
)

func main() {
	client := truelist.NewClient(os.Getenv("TRUELIST_API_KEY"))

	result, err := client.Validate("user@example.com")
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}

	fmt.Println("State:", result.State)         // "ok" | "email_invalid" | "risky" | ...
	fmt.Println("SubState:", result.SubState)    // "email_ok" | "is_disposable" | ...
	fmt.Println("Suggestion:", result.Suggestion) // "" or "user@gmail.com"
	fmt.Println("Valid:", result.IsValid())
}
```

## HTTP Handler Example

```go
func validateEmailHandler(w http.ResponseWriter, r *http.Request) {
	var req struct {
		Email string `json:"email"`
	}
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, `{"error":"invalid request"}`, http.StatusBadRequest)
		return
	}

	client := truelist.NewClient(os.Getenv("TRUELIST_API_KEY"))
	result, err := client.Validate(req.Email)
	if err != nil {
		http.Error(w, `{"error":"validation failed"}`, http.StatusInternalServerError)
		return
	}

	json.NewEncoder(w).Encode(map[string]any{
		"valid":      result.IsValid(),
		"state":      result.State,
		"suggestion": result.Suggestion,
	})
}
```

## Error Handling

```go
result, err := client.Validate("user@example.com")
if err != nil {
	var authErr *truelist.AuthenticationError
	var rateErr *truelist.RateLimitError

	switch {
	case errors.As(err, &authErr):
		// Invalid API key
	case errors.As(err, &rateErr):
		// Rate limited — check rateErr.RetryAfter
	default:
		// Other error
	}
}
```
