---
name: fiber-middleware
description: Fiber v3 middleware ecosystem including built-in middleware (logger, recover, cors, gzip, static files), custom middleware creation, and popular addons like jwt, rate-limit, and csrf. Use when adding request processing pipelines, security headers, logging, or authentication to Fiber applications.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: core
  tags: [fiber, go, middleware, security]
---

# Fiber Middleware (v3)

This skill covers Fiber v3 middleware ecosystem — built-in middleware, custom middleware creation, and popular addons for logging, security, authentication, and more.

## SOP: Step-by-Step Procedures

### SOP 1: Built-in Middleware Installation & Usage

```bash
go get -u github.com/gofiber/fiber/v3
# Popular addons
go get -u github.com/gofiber/jwt/v3
go get -u github.com/gofiber/cache/v3
go get -u github.com/gofiber/fiber/v3/middleware/recover
```

**Logger Middleware:**

```go
import (
	"github.com/gofiber/fiber/v3"
	"github.com/gofiber/fiber/v3/middleware/logger"
)

app := fiber.New()
app.Use(logger.New()) // Logs all requests to stdout
// Output: [logger] 2024/01/01 - 12:00:00 | 200 | 2.5ms | GET /
```

**Custom Logger Configuration:**

```go
app.Use(logger.New(logger.Config{
	Format:     "[${time}] ${status} - ${latency} | ${method} ${uri}\n",
	TimeZone:   "Asia/Seoul",
	TimeFormat: "2006-01-02 15:04:05",
}))
```

**Recover Middleware:**

```go
import (
	"github.com/gofiber/fiber/v3"
	"github.com/gofiber/fiber/v3/middleware/recover"
)

app := fiber.New()
app.Use(recover.New()) // Catches panics and returns 500 response
// Prevents server crash on unexpected errors
```

**CORS Middleware:**

```go
import (
	"github.com/gofiber/fiber/v3"
	"github.com/gofiber/fiber/v3/middleware/cors"
)

app := fiber.New()
app.Use(cors.New()) // Default CORS config (allows all origins)

// Custom CORS config
app.Use(cors.New(cors.Config{
	AllowOrigins:     "https://example.com, https://app.example.com",
	AllowMethods:     "GET, POST, PUT, DELETE, OPTIONS",
	AllowHeaders:     "Origin, Content-Type, Accept, Authorization",
	AllowCredentials: true,
	MaxAge:           86400, // 24 hours
}))
```

**Gzip Compression:**

```go
import (
	"github.com/gofiber/fiber/v3"
	"github.com/gofiber/fiber/v3/middleware/gzip"
)

app := fiber.New()
app.Use(gzip.New()) // Compress responses with gzip

// Custom config
app.Use(gzip.New(gzip.Config{
	Level: gzip.LevelBestCompression, // 1-9
	Threshold: 1024,                  // Only compress if response > 1KB
}))
```

**Static Files:**

```go
import (
	"github.com/gofiber/fiber/v3"
	"github.com/gofiber/fiber/v3/middleware/static"
)

app := fiber.New()
app.Use(static.New("./public")) // Serve static files from ./public at /

// Custom config
app.Use(static.New("./public", static.Config{
	Index: "index.html",       // Default index file
	Browse: false,             // Disable directory browsing
	Compress: true,            // Enable gzip compression
	MaxAge: 3600,              // Cache time in seconds
}))
```

### SOP 2: Custom Middleware Creation

**Basic Custom Middleware:**

```go
package main

import (
	"time"
	"github.com/gofiber/fiber/v3"
)

// RequestID middleware - adds unique ID to each request
func RequestID() fiber.Handler {
	return func(c fiber.Ctx) error {
		requestID := c.Get("X-Request-ID")
		if requestID == "" {
			requestID = generateUUID() // Your UUID generation function
		}

		c.Set("X-Request-ID", requestID)
		c.Locals("request_id", requestID)

		return c.Next() // Pass to next handler
	}
}

func main() {
	app := fiber.New()
	app.Use(RequestID())

	app.Get("/", func(c fiber.Ctx) error {
		requestID := c.Locals("request_id").(string)
		return c.JSON(fiber.Map{
			"requestId": requestID,
		})
	})
}
```

**Authentication Middleware:**

```go
func AuthMiddleware() fiber.Handler {
	return func(c fiber.Ctx) error {
		token := c.Get("Authorization")
		if token == "" {
			return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
				"error": "Missing authorization header",
			})
		}

		// Validate token (replace with actual JWT validation)
		claims, err := validateJWT(token)
		if err != nil {
			return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
				"error": "Invalid or expired token",
			})
		}

		// Store user info in context
		c.Locals("user_id", claims.UserID)
		c.Locals("role", claims.Role)

		return c.Next()
	}
}

// Usage - protect specific routes
app.Get("/profile", AuthMiddleware(), func(c fiber.Ctx) error {
	userID := c.Locals("user_id").(string)
	return c.JSON(fiber.Map{"userId": userID})
})
```

**Rate Limiting Middleware:**

```go
import (
	"sync"
	"time"
	"github.com/gofiber/fiber/v3"
)

type RateLimiter struct {
	requests map[string][]time.Time
	mu       sync.Mutex
	limit    int
	window   time.Duration
}

func NewRateLimiter(limit int, window time.Duration) *RateLimiter {
	return &RateLimiter{
		requests: make(map[string][]time.Time),
		limit:    limit,
		window:   window,
	}
}

func (rl *RateLimiter) Limiter() fiber.Handler {
	return func(c fiber.Ctx) error {
		ip := c.IP()

		rl.mu.Lock()
		now := time.Now()
		timestamps := rl.requests[ip]

		// Remove old timestamps outside the window
		validTimestamps := make([]time.Time, 0)
		for _, ts := range timestamps {
			if now.Sub(ts) <= rl.window {
				validTimestamps = append(validTimestamps, ts)
			}
		}

		if len(validTimestamps) >= rl.limit {
			rl.mu.Unlock()
			return c.Status(fiber.StatusTooManyRequests).JSON(fiber.Map{
				"error": "Rate limit exceeded",
			})
		}

		validTimestamps = append(validTimestamps, now)
		rl.requests[ip] = validTimestamps
		rl.mu.Unlock()

		c.Set("X-RateLimit-Limit", strconv.Itoa(rl.limit))
		c.Set("X-RateLimit-Remaining", strconv.Itoa(rl.limit-len(validTimestamps)))

		return c.Next()
	}
}

// Usage
rateLimiter := NewRateLimiter(100, time.Minute) // 100 requests per minute
app.Use(rateLimiter.Limiter())
```

### SOP 3: Middleware Chaining & Order

**Middleware Execution Order:**

```go
app := fiber.New()

// These execute in order for every request
app.Use(RequestID())       // 1. Add request ID
app.Use(Logger())          // 2. Log the request
app.Use(CORS())            // 3. Handle CORS preflight
app.Use(Gzip())            // 4. Compress response

// Auth only on protected routes
api := app.Group("/api")
api.Use(AuthMiddleware())  // Only applies to /api/* routes

api.Get("/users", func(c fiber.Ctx) error {
	return c.JSON(fiber.Map{"users": []string{}}})
})

// Execution order: RequestID → Logger → CORS → Gzip → Auth (for /api/*)
```

**Conditional Middleware:**

```go
// Only apply rate limiting in production
if os.Getenv("ENV") == "production" {
	app.Use(rateLimiter.Limiter())
}

// Apply to specific path prefix only
app.Use("/api", func(c fiber.Ctx) error {
	// Custom logic for /api/* routes only
	return c.Next()
})
```

### SOP 4: Popular Addons

**JWT Authentication:**

```go
import (
	"github.com/gofiber/jwt/v3"
)

app := fiber.New()

jwtMiddleware, err := jwt.New(jwt.Config{
	SigningKey: []byte("secret-key"),
	TokenLookup: "header:Authorization",
	AuthScheme: "Bearer",
})
if err != nil {
	log.Fatal(err)
}

app.Use(jwtMiddleware)

app.Get("/protected", func(c fiber.Ctx) error {
	user := c.Locals("user").(*jwt.Token)
	return c.JSON(fiber.Map{
		"userId": user.Claims.UserID,
	})
})
```

**CSRF Protection:**

```go
import (
	"github.com/gofiber/csrf/v3"
)

app := fiber.New()

csrfMiddleware, err := csrf.New(csrf.Config{
	KeyLookup:        "header:X-CSRF-Token",
	CookieName:       "csrf_token",
	CookieSameSite:   "Strict",
	Expiration:       24 * time.Hour,
})
if err != nil {
	log.Fatal(err)
}

app.Use(csrfMiddleware)
```

**Request ID (built-in):**

```go
import (
	"github.com/gofiber/fiber/v3/middleware/requestid"
)

app := fiber.New()
app.Use(requestid.New()) // Automatically generates UUID for each request
```

## Tool Integration

| Task | Tool | Usage |
|------|------|-------|
| Install middleware package | `run_command` | `go get -u github.com/gofiber/fiber/v3/middleware/logger` |
| List installed packages | `run_command` | `go list -m all \| grep fiber` |
| Create custom middleware | `write_file` → `edit_file` | Add new file in middleware/ package, register with app.Use() |
| Test middleware order | `run_command` + `search_files` | Verify middleware registration order in main.go |

## Anti-Patterns & Guardrails

❌ **Never** put heavy logic in middleware — keep it fast:
```go
// BAD - database query in middleware
func AuthMiddleware() fiber.Handler {
    db.Query("SELECT * FROM users WHERE token = ?", token) // Slow!
    return func(c fiber.Ctx) error { return c.Next() }
}
// GOOD - use JWT validation (fast, no DB call)
```

❌ **Never** forget to call `c.Next()` in middleware — it breaks the chain:
```go
// BAD - request never reaches handler
func MyMiddleware() fiber.Handler {
    return func(c fiber.Ctx) error {
        c.SendString("blocked")
        // Missing c.Next()!
    }
}
// GOOD - always call Next() unless blocking
func MyMiddleware() fiber.Handler {
    return func(c fiber.Ctx) error {
        if blocked {
            return c.SendString("blocked")
        }
        return c.Next()
    }
}
```

❌ **Never** use middleware for route-specific logic — use route groups instead:
```go
// BAD - middleware applies to all routes
app.Use(AuthMiddleware()) // Protects everything including /health

// GOOD - apply only to protected routes
api := app.Group("/api")
api.Use(AuthMiddleware())
```

⚠️ **Always** place CORS middleware before other middleware that might short-circuit requests.

⚠️ **Always** set appropriate timeouts and limits in production middleware.

## Best Practices

1. Keep middleware fast — avoid blocking operations (DB calls, heavy computation)
2. Use `c.Next()` to pass control to the next handler unless blocking
3. Place CORS middleware early in the chain for proper preflight handling
4. Use route groups (`app.Group()`) for middleware scoped to specific paths
5. Return meaningful error responses with appropriate HTTP status codes
6. Use `c.Locals()` for request-scoped data, not global variables

## References

- [Fiber Middleware Documentation](https://docs.gofiber.io/extra/middleware/)
- [Custom Middleware Guide](https://docs.gofiber.io/guide/middleware/)
- [GitHub - gofiber/fiber](https://github.com/gofiber/fiber)
- [JWT Addon](https://github.com/gofiber/jwt)

---

**Skill successfully created:** `skills/fiber-middleware/SKILL.md`

This skill is now ready. Please renew the Skill Index before using it.
