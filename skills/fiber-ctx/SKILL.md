---
name: fiber-ctx
description: Fiber context API for handling HTTP requests and responses including parameter extraction, JSON binding, form parsing, file uploads, cookies, headers, and response formatting. Use when processing incoming requests, extracting data, or sending structured responses in Fiber applications.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: core
  tags: [fiber, go, context, request-response]
---

# Fiber Context API (v3)

This skill covers the Fiber `Ctx` interface for handling HTTP requests and responses — parameter extraction, JSON binding, form parsing, file uploads, cookies, headers, and response formatting.

## SOP: Step-by-Step Procedures

### SOP 1: Request Parameter Extraction

**Path Parameters:**

```go
app.Get("/users/:id/posts/:postId", func(c fiber.Ctx) error {
	id := c.Params("id")        // "123"
	postId := c.Params("postId") // "456"

	// With default value if parameter is optional
	name := c.Params("name", "anonymous")

	return c.JSON(fiber.Map{
		"id":     id,
		"postId": postId,
	})
})
```

**Query Parameters:**

```go
app.Get("/search", func(c fiber.Ctx) error {
	// Single value
	q := c.Query("q")

	// With default value
	page := c.QueryInt("page", 1)       // Returns int, defaults to 1
	limit := c.QueryInt("limit", 20)    // Returns int, defaults to 20
	search := c.Query("search", "")     // Returns string

	// All query params as map
	allQuery := c.Queries()             // map[string][]string

	return c.JSON(fiber.Map{
		"query": q,
		"page":  page,
		"limit": limit,
	})
})
```

**Header Extraction:**

```go
app.Get("/info", func(c fiber.Ctx) error {
	// Single header
	userAgent := c.Get("User-Agent")
	contentType := c.Get("Content-Type")

	// With default value
	referer := c.Get("Referer", "unknown")

	// All headers
	allHeaders := c.GetReqHeaders() // map[string]string

	return c.JSON(fiber.Map{
		"userAgent": userAgent,
	})
})
```

### SOP 2: Request Body Parsing

**JSON Binding:**

```go
type CreateUserRequest struct {
	Name  string `json:"name"`
	Email string `json:"email" validate:"required,email"`
	Age   int    `json:"age"`
}

app.Post("/users", func(c fiber.Ctx) error {
	var req CreateUserRequest

	if err := c.BodyParser(&req); err != nil {
		return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
			"error": "Invalid JSON body",
		})
	}

	return c.JSON(fiber.Map{
		"name":  req.Name,
		"email": req.Email,
		"age":   req.Age,
	})
})
```

**Form Data Parsing:**

```go
app.Post("/login", func(c fiber.Ctx) error {
	username := c.FormValue("username")
	password := c.FormValue("password")

	// All form values as map
	allForm := c.FormValues() // map[string][]string

	return c.JSON(fiber.Map{
		"username": username,
	})
})
```

**Raw Body:**

```go
app.Post("/raw", func(c fiber.Ctx) error {
	body := c.Body()        // []byte
	bodyString := string(body)

	return c.SendString(bodyString)
})
```

### SOP 3: File Uploads

**Single File Upload:**

```go
app.Post("/upload", func(c fiber.Ctx) error {
	file, err := c.FormFile("file")
	if err != nil {
		return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
			"error": "No file uploaded",
		})
	}

	// Save to disk
	filename := filepath.Base(file.Filename)
	if err := c.SaveFile(file, "./uploads/"+filename); err != nil {
		return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
			"error": "Failed to save file",
		})
	}

	return c.JSON(fiber.Map{
		"message":  "File uploaded successfully",
		"filename": filename,
	})
})
```

**Multiple File Uploads:**

```go
app.Post("/uploads", func(c fiber.Ctx) error {
	form, err := c.MultipartForm()
	if err != nil {
		return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
			"error": "Invalid multipart form",
		})
	}

	files := form.File["files"] // []*multipart.FileHeader

	for _, file := range files {
		filename := filepath.Base(file.Filename)
		c.SaveFile(file, "./uploads/"+filename)
	}

	return c.JSON(fiber.Map{
		"message":  "Files uploaded successfully",
		"count":    len(files),
	})
})
```

### SOP 4: Response Formatting

**JSON Response:**

```go
app.Get("/user/:id", func(c fiber.Ctx) error {
	user := fiber.Map{
		"id":    c.Params("id"),
		"name":  "John Doe",
		"email": "john@example.com",
	}

	return c.JSON(user)
})
```

**JSON with Custom Status:**

```go
app.Post("/users", func(c fiber.Ctx) error {
	var req CreateUserRequest
	if err := c.BodyParser(&req); err != nil {
		return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
			"error": "Invalid request body",
		})
	}

	// Success with custom status
	return c.Status(fiber.StatusCreated).JSON(fiber.Map{
		"id":    123,
		"name":  req.Name,
		"email": req.Email,
	})
})
```

**Other Response Types:**

```go
// Plain text
app.Get("/text", func(c fiber.Ctx) error {
	return c.SendString("Hello, World!")
})

// HTML
app.Get("/html", func(c fiber.Ctx) error {
	return c.SendString("<h1>Hello</h1>")
})

// XML
app.Get("/xml", func(c fiber.Ctx) error {
	type User struct {
		Name  string `xml:"name"`
		Email string `xml:"email"`
	}
	return c.XML(User{"John", "john@example.com"})
})

// File download
app.Get("/download", func(c fiber.Ctx) error {
	return c.Download("./files/report.pdf")
})

// Redirect
app.Get("/old-path", func(c fiber.Ctx) error {
	return c.Redirect("/new-path", fiber.StatusMovedPermanently)
})
```

### SOP 5: Cookies

**Set Cookie:**

```go
app.Get("/set-cookie", func(c fiber.Ctx) error {
	cookie := new(fiber.Cookie)
	cookie.Key = "session_id"
	cookie.Value = "abc123"
	cookie.Expires = time.Now().Add(24 * time.Hour)
	cookie.HTTPOnly = true
	cookie.Secure = true
	cookie.SameSite = "Strict"

	c.Cookie(cookie)
	return c.SendString("Cookie set")
})
```

**Read Cookie:**

```go
app.Get("/get-cookie", func(c fiber.Ctx) error {
	sessionID := c.Cookies("session_id", "default-value")
	return c.JSON(fiber.Map{
		"session_id": sessionID,
	})
})
```

### SOP 6: Response Headers

**Set Custom Headers:**

```go
app.Get("/headers", func(c fiber.Ctx) error {
	c.Set("X-Custom-Header", "custom-value")
	c.Set("X-RateLimit-Limit", "100")
	c.Set("X-RateLimit-Remaining", "99")

	return c.JSON(fiber.Map{"message": "Custom headers set"})
})
```

**Remove Headers:**

```go
app.Get("/remove-header", func(c fiber.Ctx) error {
	c.Response().Header.Del("Server") // Remove Server header
	return c.SendString("Headers modified")
})
```

### SOP 7: Context Values (Request-scoped Storage)

```go
// Set values in handler/middleware
app.Use(func(c fiber.Ctx) error {
	c.Locals("user_id", "12345")
	c.Locals("request_id", uuid.New().String())
	return c.Next()
})

// Read values in downstream handlers
app.Get("/data", func(c fiber.Ctx) error {
	userID := c.Locals("user_id").(string)
	requestID := c.Locals("request_id").(string)

	return c.JSON(fiber.Map{
		"userId":    userID,
		"requestId": requestID,
	})
})
```

## Tool Integration

| Task | Tool | Usage |
|------|------|-------|
| Create handler file | `write_file` → `edit_file` | Add new route handlers in handlers/ package |
| Test API endpoint | `run_command` | `curl -X POST http://localhost:3000/users -H "Content-Type: application/json" -d '{"name":"test"}'` |
| Verify JSON binding | `search_files` + `read_file` | Find struct tags and BodyParser usage |

## Anti-Patterns & Guardrails

❌ **Never** ignore errors from `BodyParser()` — always check the return value:
```go
// BAD - silently ignores parse errors
c.BodyParser(&req)
// GOOD - handle error properly
if err := c.BodyParser(&req); err != nil {
    return c.Status(fiber.StatusBadRequest).JSON(...)
}
```

❌ **Never** use `c.SendString()` for complex JSON — always use `c.JSON()`:
```go
// BAD - manual JSON string construction
c.SendString(`{"id": "` + id + `"}`)
// GOOD - proper JSON serialization
c.JSON(fiber.Map{"id": id})
```

❌ **Never** store sensitive data in `c.Locals()` without encryption:
```go
// BAD - plain text password storage
c.Locals("password", req.Password)
// GOOD - use secure session/cookie mechanism
```

⚠️ **Always** validate and sanitize user input before processing.

⚠️ **Always** set appropriate Content-Type headers when sending non-JSON responses.

## Best Practices

1. Use `fiber.Map` for JSON responses (cleaner than manual serialization)
2. Always check errors from `BodyParser()`, file operations, etc.
3. Use struct tags (`json:"field_name"`) for clean request/response mapping
4. Set `HTTPOnly`, `Secure`, and `SameSite` flags on cookies in production
5. Use `c.Locals()` for request-scoped data (not global variables)
6. Return `nil` error when response is successfully sent

## References

- [Fiber Ctx Documentation](https://docs.gofiber.io/api/ctx)
- [Request/Response Guide](https://docs.gofiber.io/guide/request-response/)
- [GitHub Repository](https://github.com/gofiber/fiber)

---

**Skill successfully created:** `skills/fiber-ctx/SKILL.md`

This skill is now ready. Please renew the Skill Index before using it.
