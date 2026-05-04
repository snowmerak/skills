---
name: go-error-handling
description: Go error handling patterns including custom error structs, error interface implementation, error wrapping with fmt.Errorf, error checking with errors.Is/AsType (Go 1.26+), and best practices for error messages and propagation. Use when creating custom errors, propagating errors up the call stack, or implementing consistent error handling in Go applications.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: core
  tags: [go, error-handling, patterns]
---

# Go Error Handling Patterns

This skill covers Go's error handling idioms — custom error structs, `error` interface implementation, wrapping with `%w`, checking with `errors.Is/AsType` (Go 1.26+), and consistent message formatting.

## SOP: Step-by-Step Procedures

### SOP 1: Custom Error Struct Definition

**Basic Custom Error:**

```go
package main

import "fmt"

// UserNotFoundError represents a missing user in the database
type UserNotFoundError struct {
	UserID string
}

func (e *UserNotFoundError) Error() string {
	return fmt.Sprintf("user not found: %s", e.UserID)
}
```

**Error with Additional Context:**

```go
package main

import "fmt"

// ValidationError represents a validation failure with field details
type ValidationError struct {
	Field   string
	Message string
	Code    int
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("validation error on field '%s': %s (code: %d)", e.Field, e.Message, e.Code)
}

// Implement custom method for programmatic access
func (e *ValidationError) FieldName() string {
	return e.Field
}
```

**Error with HTTP Status Code:**

```go
package main

import "fmt"

type HTTPError struct {
	StatusCode int
	Message    string
}

func (e *HTTPError) Error() string {
	return fmt.Sprintf("HTTP %d: %s", e.StatusCode, e.Message)
}

func (e *HTTPError) StatusCode() int {
	return e.StatusCode
}
```

### SOP 2: Implementing the error Interface

**The `error` interface is simple:**

```go
type error interface {
	Error() string
}
```

**Any type with an `Error() string` method satisfies `error`:**

```go
// Struct-based (recommended for complex errors)
type NotFoundError struct {
	Resource string
	ID       string
}

func (e *NotFoundError) Error() string {
	return fmt.Sprintf("%s not found: %s", e.Resource, e.ID)
}

// Function-based (simple one-off errors)
var ErrInvalidInput = fmt.Errorf("invalid input provided")

// Sentinel error (package-level constant)
var ErrNotFound = errors.New("resource not found")
```

### SOP 3: Returning Errors with `error` Interface

**Returning Custom Errors:**

```go
func FindUser(id string) (*User, error) {
	user, err := db.QueryUser(id)
	if err != nil {
		return nil, &UserNotFoundError{UserID: id} // Return custom error struct
	}
	return user, nil
}
```

**Returning Sentinel Errors:**

```go
func GetUser(id string) (*User, error) {
	user := db.FindByID(id)
	if user == nil {
		return nil, ErrNotFound // Return package-level sentinel error
	}
	return user, nil
}
```

### SOP 4: Wrapping & Propagating Errors with `%w`

**Basic Error Wrapping:**

```go
func CreateUser(name string) (*User, error) {
	user := &User{Name: name}

	if err := validateName(user.Name); err != nil {
		// Wrap with context using %w (preserves errors.Is/As chain)
		return nil, fmt.Errorf("creating user: %w", err)
	}

	if err := db.Save(user); err != nil {
		return nil, fmt.Errorf("saving user '%s': %w", name, err)
	}

	return user, nil
}
```

**Multiple Levels of Wrapping:**

```go
func ProcessOrder(orderID string) error {
	order, err := fetchOrder(orderID)
	if err != nil {
		return fmt.Errorf("fetching order %s: %w", orderID, err) // Level 1
	}

	payment, err := processPayment(order)
	if err != nil {
		return fmt.Errorf("processing payment for order %s: %w", orderID, err) // Level 2
	}

	if err := notifyCustomer(order, payment); err != nil {
		return fmt.Errorf("notifying customer for order %s: %w", orderID, err) // Level 3
	}

	return nil
}
```

**Adding Extra Context Without Wrapping (use `%v`):**

```go
func LoadConfig(path string) (*Config, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		// %v does NOT preserve errors.Is/As chain — use when you don't need to unwrap
		return nil, fmt.Errorf("reading config file '%s': %v", path, err)
	}

	var cfg Config
	if err := json.Unmarshal(data, &cfg); err != nil {
		return nil, fmt.Errorf("parsing config: %w", err) // %w to preserve chain
	}

	return &cfg, nil
}
```

### SOP 5: Checking Errors with `errors.Is` and `errors.AsType`

**Go 1.26+ — Prefer `errors.AsType` over `errors.As`:**

```go
import "errors"

func handleUser(id string) {
	user, err := FindUser(id)
	if err != nil {
		// Check if error is (or wraps) a specific sentinel error
		if errors.Is(err, ErrNotFound) {
			log.Printf("User %s not found", id)
			return // Handle gracefully
		}

		// Go 1.26+: AsType returns the matched error directly (no pointer needed)
		if userErr := errors.AsType[*UserNotFoundError](err); userErr != nil {
			log.Printf("Specific user error: %s", userErr.UserID)
			return
		}

		// Fallback for unexpected errors
		log.Printf("Unexpected error: %v", err)
		return
	}

	// Process user...
}
```

**Why `AsType` is preferred over `As` (Go 1.26+):**

```go
// BAD with As — requires pointer, verbose
var userErr *UserNotFoundError
if errors.As(err, &userErr) {
    log.Printf("Error: %s", userErr.UserID)
}

// GOOD with AsType — returns value directly, cleaner
if userErr := errors.AsType[*UserNotFoundError](err); userErr != nil {
    log.Printf("Error: %s", userErr.UserID)
}
```

**Go < 1.26 fallback (using `errors.As`):**

```go
// For Go versions before 1.26, use errors.As with pointer
var userErr *UserNotFoundError
if errors.As(err, &userErr) {
    log.Printf("Error: %s", userErr.UserID)
}
```

**Extracting Multiple Error Types:**

```go
func handlePayment(amount float64) error {
	err := processPayment(amount)
	if err != nil {
		// Check custom validation error
		if valErr := errors.AsType[*ValidationError](err); valErr != nil {
			log.Printf("Validation failed on field: %s", valErr.FieldName())
			return fmt.Errorf("payment validation: %w", err)
		}

		// Check HTTP error
		if httpErr := errors.AsType[*HTTPError](err); httpErr != nil {
			log.Printf("HTTP error: %d - %s", httpErr.StatusCode(), httpErr.Message())
			return fmt.Errorf("payment HTTP failure: %w", err)
		}

		// Unexpected error
		return fmt.Errorf("unexpected payment error: %w", err)
	}
	return nil
}
```

**Multiple Error Checks with switch:**

```go
func handleRequest(id string) error {
	err := processRequest(id)
	if err != nil {
		switch {
		case errors.Is(err, ErrNotFound):
			return c.Status(404).JSON(...)
		case errors.Is(err, ErrUnauthorized):
			return c.Status(401).JSON(...)
		default:
			if valErr := errors.AsType[*ValidationError](err); valErr != nil {
				return c.Status(422).JSON(fiber.Map{"field": valErr.Field})
			}
			return c.Status(500).JSON(...)
		}
	}
	return nil
}
```

### SOP 6: Error Message Guidelines

**Good Error Messages:**

```go
// ✅ Clear, actionable, includes context
fmt.Errorf("failed to connect to database 'mydb' on host '%s': %w", host, err)

// ✅ Describes what operation failed
fmt.Errorf("invalid email format for user '%s': %v", username, input)

// ✅ Includes relevant identifiers
fmt.Errorf("file upload failed: size %d bytes exceeds limit of %d bytes", size, maxLimit)
```

**Bad Error Messages:**

```go
// ❌ Vague, no context
errors.New("error occurred")

// ❌ Redundant (already in wrapping message)
fmt.Errorf("database error: database connection failed: %w", err)

// ❌ Technical jargon without explanation
fmt.Errorf("errno 28: disk full")
```

## Tool Integration

| Task | Tool | Usage |
|------|------|-------|
| Find error definitions | `search_files` | Search for `errors.New(` or `type.*Error struct` |
| Verify wrapping chain | `read_file` + `search_files` | Check `%w` usage in error propagation paths |
| Test error handling | `run_command` | Run tests with `go test -v ./...` |

## Anti-Patterns & Guardrails

❌ **Never** use `%v` when you need to preserve the error chain for `errors.Is/AsType`:
```go
// BAD - breaks errors.Is() chain
return fmt.Errorf("context: %v", err)
// GOOD - preserves chain
return fmt.Errorf("context: %w", err)
```

❌ **Never** return raw string as error — always use `error` interface or custom struct:
```go
// BAD
func FindUser(id string) (*User, string) { ... }
// GOOD
func FindUser(id string) (*User, error) { ... }
```

❌ **Never** swallow errors with empty catch blocks:
```go
// BAD - silently ignores error
if err := doSomething(); err != nil {
    log.Println(err) // Just logging, not returning
}
// GOOD - propagate or handle explicitly
if err := doSomething(); err != nil {
    return fmt.Errorf("operation failed: %w", err)
}
```

❌ **Never** use `panic()` for expected error conditions — reserve for truly unrecoverable states:
```go
// BAD - user input validation should return error, not panic
if name == "" {
    panic("name is required") // Wrong!
}
// GOOD
if name == "" {
    return nil, &ValidationError{Field: "name", Message: "required"}
}
```

⚠️ **Always** use `errors.Is()` for sentinel errors and `errors.AsType[T]()` (Go 1.26+) or `errors.As()` for custom error structs. Prefer `AsType` — it returns the value directly without needing a pointer variable.

⚠️ **Always** include operation context in wrapped errors (e.g., `"saving user: %w"` not just `"%w"`).

## Best Practices

1. Define custom error structs with meaningful fields for programmatic access
2. Use `%w` when wrapping to preserve `errors.Is/AsType` chain; use `%v` only when unwrapping is unnecessary
3. Check errors at the boundary (HTTP handlers, CLI entry points) and handle appropriately
4. Keep error messages clear, actionable, and include relevant identifiers (IDs, names)
5. Use package-level sentinel errors (`var ErrNotFound = errors.New(...)`) for common cases
6. Never use `panic()` for expected errors — always return `error`

## References

- [Go Error Handling Best Practices](https://go.dev/blog/go1.13-errors)
- [errors Package Documentation (Go 1.26)](https://pkg.go.dev/errors)
- [fmt.Errorf Documentation](https://pkg.go.dev/fmt#Errorf)
- [A Tour of Go: Errors](https://go.dev/tour/basics/7)

---

**Skill successfully created:** `skills/go-error-handling/SKILL.md`

This skill is now ready. Please renew the Skill Index before using it.
