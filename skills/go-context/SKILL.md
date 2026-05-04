---
name: go-context
description: Go context package covering root contexts (Background/TODO), derived contexts (WithCancel/Cause, WithDeadline/Cause, WithTimeout/Cause), value passing with typed keys, cancellation patterns, timeout handling for HTTP/DB/API calls, and utility functions (AfterFunc, WithoutCancel). Use when managing request lifecycles, implementing timeouts, propagating cancellation signals, or passing request-scoped data in Go applications.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: core
  tags: [go, context, timeout, cancellation]
---

# Go Context Package (v3)

This skill covers the `context` package — root contexts, derived contexts with cancellation/deadline/timeout, typed value passing, and practical patterns for HTTP servers, goroutines, and external calls.

## SOP: Step-by-Step Procedures

### SOP 1: Root Contexts

**Background() — Entry Point:**

```go
import "context"

func main() {
    // Background is the root context with no deadline or cancellation
    ctx := context.Background()

    // Use as parent for all derived contexts
    httpServer.ListenAndServe(ctx, handler)
}
```

**TODO() — Placeholder:**

```go
// Use when you don't know which Context to pass yet
func DoSomething() error {
    ctx := context.TODO() // Replace with proper context later
    return queryDatabase(ctx)
}
```

### SOP 2: Derived Contexts — Cancellation

**WithCancel (Basic):**

```go
import "context"

func processRequest(ctx context.Context) error {
    // Create derived context with cancel function
    ctx, cancel := context.WithCancel(ctx)
    defer cancel() // Always call to release resources

    // Use in goroutines
    go func() {
        select {
        case <-ctx.Done():
            log.Println("Request cancelled")
            return
        default:
            // Process data...
        }
    }()

    return doWork(ctx)
}
```

**WithCancelCause (Go 1.20+ — Recommended):**

```go
func processWithCause(parent context.Context) error {
    ctx, cancel := context.WithCancelCause(parent)
    defer func() {
        // Record cause on cancellation
        if err := someOperation(); err != nil {
            cancel(err) // Sets the cancellation cause
        } else {
            cancel(context.Canceled)
        }
    }()

    return doWork(ctx)
}

// Retrieve cause later
if ctx.Err() != nil {
    cause := context.Cause(ctx) // Returns the error passed to cancel()
    log.Printf("Cancelled because: %v", cause)
}
```

### SOP 3: Derived Contexts — Deadline & Timeout

**WithDeadline (Absolute Time):**

```go
func requestWithDeadline(parent context.Context) error {
    deadline := time.Now().Add(5 * time.Second)
    ctx, cancel := context.WithDeadline(parent, deadline)
    defer cancel()

    // Context auto-cancels when deadline passes
    return doWork(ctx)
}
```

**WithTimeout (Relative Duration):**

```go
func requestWithTimeout(parent context.Context) error {
    ctx, cancel := context.WithTimeout(parent, 30*time.Second)
    defer cancel()

    // Context auto-cancels after 30 seconds
    return doWork(ctx)
}
```

**WithDeadlineCause / WithTimeoutCause (Go 1.21+):**

```go
func requestWithTimeoutAndCause(parent context.Context) error {
    ctx, cancel := context.WithTimeoutCause(
        parent,
        30*time.Second,
        fmt.Errorf("request timeout exceeded"), // Cause recorded on timeout
    )
    defer cancel()

    err := doWork(ctx)
    if errors.Is(err, context.DeadlineExceeded) {
        cause := context.Cause(ctx) // Returns the custom error above
        log.Printf("Timeout caused by: %v", cause)
    }
    return err
}
```

### SOP 4: Typed Value Passing

**Define Key Type (Prevents Collisions):**

```go
package main

import "context"

// UserKey is an unexported type to prevent key collisions
type UserKey struct{}

func WithUser(ctx context.Context, user *User) context.Context {
    return context.WithValue(ctx, UserKey{}, user)
}

func GetUserFromContext(ctx context.Context) (*User, bool) {
    user, ok := ctx.Value(UserKey{}).(*User)
    return user, ok
}
```

**Usage in HTTP Handler:**

```go
func handleRequest(w http.ResponseWriter, r *http.Request) {
    // Extract from request (Go 1.23+ supports context parameter)
    ctx := r.Context()

    // Pass to downstream functions
    user := GetUserFromContext(ctx)
    if user == nil {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }

    result, err := processWithUser(ctx, user.ID)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Write(result)
}
```

**⚠️ Never use Value for optional parameters:**

```go
// BAD - Use function parameters instead
func WithDebugMode(ctx context.Context, enabled bool) context.Context {
    return context.WithValue(ctx, "debug", enabled) // Wrong!
}

// GOOD - Use explicit parameter
func processWithDebug(ctx context.Context, debug bool) error { ... }
```

### SOP 5: Cancellation & Cleanup Patterns

**Defer Cancel (Always):**

```go
func doWork(parent context.Context) error {
    ctx, cancel := context.WithTimeout(parent, 10*time.Second)
    defer cancel() // Releases resources when function returns

    return queryDatabase(ctx)
}
```

**Goroutine Cleanup:**

```go
func fetchMultiple(ctx context.Context) ([]Result, error) {
    results := make([]Result, 3)
    errs := make(chan error, 3)

    for i := 0; i < 3; i++ {
        go func(idx int) {
            select {
            case <-ctx.Done():
                errs <- ctx.Err() // Propagate cancellation
            default:
                results[idx], errs <- doWork(ctx)
            }
        }(i)
    }

    for i := 0; i < 3; i++ {
        if err := <-errs; err != nil {
            return nil, err
        }
    }

    return results, nil
}
```

**AfterFunc (Go 1.21+ — Cleanup on Cancellation):**

```go
func requestWithCleanup(ctx context.Context) error {
    // Run cleanup when context is cancelled
    stop := context.AfterFunc(ctx, func() {
        log.Println("Cleaning up resources...")
        closeConnectionPool()
    })
    defer stop() // Stop the association (doesn't wait for f to complete)

    return doWork(ctx)
}
```

### SOP 6: WithoutCancel (Decouple Cancellation from Deadline/Value)

**Use Case — Background Task with Request Context Values:**

```go
func handleRequest(parent context.Context) error {
    // Extract deadline and values, but don't propagate cancellation
    ctx := context.WithoutCancel(parent)

    // Start long-running task that should NOT be cancelled by request timeout
    go func() {
        user := GetUserFromContext(ctx) // Values still accessible
        sendNotification(user.ID)         // Runs even after request completes
    }()

    return doWork(parent) // Original context can still timeout
}
```

### SOP 7: Practical HTTP Server Pattern

**Complete Request Handler:**

```go
func handleAPIRequest(w http.ResponseWriter, r *http.Request) {
    // Create timeout context for the entire request
    ctx, cancel := context.WithTimeout(r.Context(), 30*time.Second)
    defer cancel()

    // Add request-scoped values
    ctx = context.WithValue(ctx, RequestIDKey{}, generateRequestID())

    // Handle with proper error response
    result, err := processRequest(ctx)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            http.Error(w, "Request cancelled", http.StatusServiceUnavailable)
            return
        }
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "Request timeout", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("X-Request-ID", ctx.Value(RequestIDKey{}).(string))
    json.NewEncoder(w).Encode(result)
}
```

## Tool Integration

| Task | Tool | Usage |
|------|------|-------|
| Find context usage | `search_files` | Search for `context.Context`, `r.Context()`, `WithTimeout` |
| Verify defer cancel | `read_file` + `search_files` | Check all `WithCancel/WithTimeout` have matching `defer cancel()` |
| Test timeout behavior | `run_command` | Run tests with `-timeout` flag or mock time.Sleep |

## Anti-Patterns & Guardrails

❌ **Never** store Context in struct — always pass as first parameter:
```go
// BAD - Context stored in struct
type Service struct { ctx context.Context }
func (s *Service) Do() error { ... }

// GOOD - Explicit parameter
func Do(ctx context.Context, s *Service) error { ... }
```

❌ **Never** pass nil Context — use `context.TODO()` if unsure:
```go
// BAD
var ctx context.Context = nil
doWork(ctx) // Will panic on ctx.Done()

// GOOD
ctx := context.TODO()
doWork(ctx)
```

❌ **Never** forget to call `defer cancel()` — causes resource leaks:
```go
// BAD - leak!
ctx, cancel := context.WithTimeout(parent, 10*time.Second)
// Missing defer cancel()!

// GOOD
ctx, cancel := context.WithTimeout(parent, 10*time.Second)
defer cancel() // Always call when done
```

❌ **Never** use `WithValue` for optional parameters or sensitive data:
```go
// BAD - Use function parameter instead
context.WithValue(ctx, "debug", true)

// BAD - Never store passwords/tokens in context values
context.WithValue(ctx, "password", user.Password)
```

⚠️ **Always** check `ctx.Err()` or select on `<-ctx.Done()` before starting expensive operations.

⚠️ **Always** use unexported key types with `WithValue` to prevent collisions:
```go
type userIDKey struct{} // Unexported type prevents external packages from using same key
context.WithValue(ctx, userIDKey{}, 123)
```

## Best Practices

1. Always pass Context as the first parameter: `func Do(ctx context.Context, ...)`
2. Use `defer cancel()` immediately after creating derived contexts
3. Prefer `WithCancelCause` (Go 1.20+) for meaningful cancellation reasons
4. Use unexported types as keys with `WithValue` to prevent collisions
5. Check `<-ctx.Done()` in goroutines to respond to cancellation promptly
6. Use `WithoutCancel` when starting background tasks that should outlive the request

## References

- [Go Context Documentation](https://pkg.go.dev/context)
- [Go Blog: Go 1.20 Context Changes](https://go.dev/blog/go1.20-context-changes)
- [Go Blog: Context and Structs](https://go.dev/blog/context-and-structs)

---

**Skill successfully created:** `skills/go-context/SKILL.md`

This skill is now ready. Please renew the Skill Index before using it.
