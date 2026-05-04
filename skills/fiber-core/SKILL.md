---
name: fiber-core
description: Fiber v3 core concepts including app creation, configuration, routing methods, route groups, and server lifecycle management. Use when creating a new Fiber application, defining routes, configuring the server, or managing the application lifecycle in Go.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: core
  tags: [fiber, go, http-server, routing]
---

# Fiber Core (v3)

This skill covers Fiber v3 fundamentals: app creation, configuration, routing methods, route groups, and server lifecycle management.

## SOP: Step-by-Step Procedures

### SOP 1: Installation & Basic Server Setup

```bash
go get -u github.com/gofiber/fiber/v3
```

**Basic Server:**

```go
package main

import (
	"log"
	"github.com/gofiber/fiber/v3"
)

func main() {
	app := fiber.New()

	app.Get("/", func(c fiber.Ctx) error {
		return c.SendString("Hello, World!")
	})

	log.Fatal(app.Listen(":3000"))
}
```

### SOP 2: Configuration Options

**Custom Config:**

```go
app := fiber.New(fiber.Config{
	AppName:          "my-app-v1.0",       // App name in logs and Server header
	BodyLimit:        10 * 1024 * 1024,    // 10MB request body limit
	CaseSensitive:    false,                // /Foo and /foo treated same
	StrictRouting:    false,                // Trailing slash matters
	ServerHeader:     "Fiber",              // Server header value
	Concurrency:      256 * 1024,           // Max concurrent connections
	Immutable:        true,                 // All context values immutable
	ReadBufferSize:   4096,                 // HTTP read buffer size
	ReadTimeout:      5 * time.Second,      // Max duration for reading request
	WriteTimeout:     10 * time.Second,     // Max duration for writing response
	IdleTimeout:      30 * time.Second,     // Keep-alive idle timeout
	DisableKeepalive: false,                // Disable connection keep-alive
	GETOnly:          false,                // Reject non-GET requests
	DisableDefaultDate:    false,            // Omit Date header
	DisableDefaultContentType: false,        // Omit Content-Type header
	DisableHeaderNormalizing:  false,        // Don't normalize header names (e.g., conteNT-tYPE -> Content-Type)
	DisableHeadAutoRegister:   false,        // Auto-register HEAD for each GET
	ErrorHandler:          DefaultErrorHandler, // Custom error handler
	JSONEncoder:           json.Marshal,     // Custom JSON encoder
	JSONDecoder:           json.Unmarshal,   // Custom JSON decoder
})
```

### SOP 3: Routing Methods

**HTTP Method Routes:**

```go
// All methods
app.All("/api/resource", func(c fiber.Ctx) error {
	return c.SendString("Handles all HTTP methods")
})

// GET
app.Get("/users", func(c fiber.Ctx) error {
	return c.JSON(fiber.Map{
		"users": []string{"Alice", "Bob"},
	})
})

// POST
app.Post("/users", func(c fiber.Ctx) error {
	name := c.FormValue("name")
	return c.JSON(fiber.Map{"name": name})
})

// PUT, PATCH, DELETE
app.Put("/users/:id", func(c fiber.Ctx) error {
	id := c.Params("id")
	return c.SendString("Update user: " + id)
})

app.Delete("/users/:id", func(c fiber.Ctx) error {
	id := c.Params("id")
	return c.SendString("Delete user: " + id)
})

// HEAD (auto-registered for GET unless DisableHeadAutoRegister=true)
```

**Dynamic Parameters:**

```go
// Single parameter
app.Get("/users/:id", func(c fiber.Ctx) error {
	id := c.Params("id")
	return c.SendString("User ID: " + id)
})

// Multiple parameters
app.Get("/users/:userId/posts/:postId", func(c fiber.Ctx) error {
	userId := c.Params("userId")
	postId := c.Params("postId")
	return c.JSON(fiber.Map{
		"userId": userId,
		"postId": postId,
	})
})

// Optional parameter (matches /users and /users/123)
app.Get("/users/:id?", func(c fiber.Ctx) error {
	id := c.Params("id", "anonymous") // Default value if not provided
	return c.SendString("User: " + id)
})

// Wildcard/catch-all route
app.Get("/files/*", func(c fiber.Ctx) error {
	path := c.Params("*") // Full remaining path
	return c.SendString("File path: " + path)
})
```

**Regex Route:**

```go
app.Get("/user/:name([a-z]+)", func(c fiber.Ctx) error {
	name := c.Params("name")
	return c.SendString("Name: " + name) // Only matches lowercase letters
})
```

### SOP 4: Route Groups

**Basic Grouping:**

```go
api := app.Group("/api/v1")

api.Get("/users", func(c fiber.Ctx) error {
	return c.JSON(fiber.Map{"route": "/api/v1/users"})
})

api.Post("/users", func(c fiber.Ctx) error {
	return c.JSON(fiber.Map{"action": "create user"})
})

api.Put("/users/:id", func(c fiber.Ctx) error {
	return c.JSON(fiber.Map{"action": "update user"})
})

api.Delete("/users/:id", func(c fiber.Ctx) error {
	return c.JSON(fiber.Map{"action": "delete user"})
})
```

**Nested Groups:**

```go
v1 := app.Group("/api/v1")
auth := v1.Group("/auth")

auth.Post("/login", func(c fiber.Ctx) error {
	return c.JSON(fiber.Map{"action": "login"})
})

auth.Post("/register", func(c fiber.Ctx) error {
	return c.JSON(fiber.Map{"action": "register"})
})
// Routes: /api/v1/auth/login, /api/v1/auth/register
```

**Group with Middleware:**

```go
api := app.Group("/api", middleware.RateLimit())

api.Get("/users", func(c fiber.Ctx) error {
	return c.JSON(fiber.Map{"users": []string{}}})
})
// Rate limit applies to all /api/* routes
```

### SOP 5: Server Lifecycle Management

**Start/Stop Server:**

```go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gofiber/fiber/v3"
)

func main() {
	app := fiber.New()

	app.Get("/", func(c fiber.Ctx) error {
		return c.SendString("Hello, World!")
	})

	// Graceful shutdown
	go func() {
		sigChan := make(chan os.Signal, 1)
		signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
		<-sigChan

		log.Println("Shutting down server...")

		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()

		if err := app.ShutdownWithContext(ctx); err != nil {
			log.Fatalf("Server forced to shutdown: %v", err)
		}
	}()

	log.Fatal(app.Listen(":3000"))
}
```

**Custom Listener:**

```go
import (
	"net"
	"github.com/gofiber/fiber/v3"
)

listener, err := net.Listen("tcp", ":3000")
if err != nil {
	log.Fatal(err)
}

app.Listener(listener)
```

### SOP 6: Static Files & Templates

**Static Files:**

```go
// Serve files from ./public directory at / prefix
app.Use("/static", fiber.New(fiber.Static{
	Root:   http.Dir("./public"),
	PathPrefix: "/static",
}))

// Or simpler with built-in helper
app.Get("/favicon.ico", func(c fiber.Ctx) error {
	return c.SendFile("./public/favicon.ico")
})
```

**Template Engine:**

```go
import (
	"github.com/gofiber/template/html"
	"github.com/gofiber/fiber/v3"
)

engine := html.New("./views", ".html")
app := fiber.New(fiber.Config{
	ViewEngine: engine,
})

app.Get("/", func(c fiber.Ctx) error {
	return c.Render("index", fiber.Map{
		"title": "Home Page",
		"name":  "Fiber",
	})
})
```

## Tool Integration

| Task | Tool | Usage |
|------|------|-------|
| Initialize Go module | `run_command` | `go mod init github.com/user/project` |
| Install Fiber | `run_command` | `go get -u github.com/gofiber/fiber/v3` |
| Verify dependencies | `run_command` | `go list -m all \| grep fiber` |
| Create route handler | `write_file` → `edit_file` | Add new route in main.go or routes package |
| Test server locally | `run_command` | `go run main.go` then curl localhost:3000 |

## Anti-Patterns & Guardrails

❌ **Never** ignore error from handler — always return error:
```go
// BAD - panic on error
name := c.Params("id") // won't panic but ignoring errors is bad practice
// GOOD - handle and return error
if err := doSomething(); err != nil {
    return c.Status(fiber.StatusInternalServerError).SendString(err.Error())
}
return nil
```

❌ **Never** use string concatenation for JSON responses — use `fiber.Map`:
```go
// BAD
c.SendString(`{"id": "` + id + `"}`)
// GOOD
c.JSON(fiber.Map{"id": id})
```

❌ **Never** block the handler with long-running operations without goroutines:
```go
// BAD - blocks entire request
time.Sleep(10 * time.Second)
// GOOD - use goroutine for background work
go func() { /* heavy work */ }()
```

⚠️ **Always** set `BodyLimit` to prevent DoS attacks from large payloads.

⚠️ **Always** handle errors properly — Fiber does not automatically log handler errors.

## Best Practices

1. Use `fiber.Map` for JSON responses (cleaner than manual serialization)
2. Group related routes with `app.Group()` for cleaner organization
3. Set appropriate `BodyLimit`, `ReadTimeout`, `WriteTimeout` for production
4. Implement graceful shutdown with signal handling
5. Use parameter validation in handlers or middleware
6. Keep handlers small — extract logic to separate functions

## References

- [Fiber Documentation](https://docs.gofiber.io/)
- [GitHub Repository](https://github.com/gofiber/fiber)
- [API Reference - Fiber](https://docs.gofiber.io/api/fiber)
- [API Reference - App](https://docs.gofiber.io/api/app)
- [Routing Guide](https://docs.gofiber.io/guide/routing)

---

**Skill successfully created:** `skills/fiber-core/SKILL.md`

This skill is now ready. Please renew the Skill Index before using it.
