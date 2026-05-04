---
name: go-cache
description: Go caching patterns including x/singleflight for deduplication, ristretto in-memory cache with SampledLFU eviction, rueidis Redis client with Client Side Caching, multi-level cache architecture (L1 ristretto → L2 rueidis → DB), and invalidation strategies. Use when implementing high-performance caching layers or building distributed cache systems.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: core
  tags: [go, cache, singleflight, ristretto, rueidis]
---

# Go Caching Patterns (v3)

This skill covers Go caching — `x/singleflight` for request deduplication, `ristretto` in-memory cache, `rueidis` Redis client with Client Side Caching, multi-level cache architecture, and invalidation strategies.

## SOP: Step-by-Step Procedures

### SOP 1: x/singleflight — Deduplicate Concurrent Requests

**Basic Usage:**

```go
import (
	"context"
	"golang.org/x/sync/singleflight"
)

var group singleflight.Group

func getUser(ctx context.Context, id string) (*User, error) {
	result, err, shared := group.Do(id, func() (interface{}, error) {
		return db.QueryUser(ctx, id) // Only ONE goroutine executes this
	})
	if err != nil {
		return nil, err
	}
	if shared {
		log.Println("Result was shared from concurrent request")
	}
	return result.(*User), nil
}
```

**With Context Timeout:**

```go
func getUserWithTimeout(ctx context.Context, id string) (*User, error) {
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	resultCh := make(chan interface{}, 1)
	errCh := make(chan error, 1)

	go func() {
		result, err, _ := group.Do(id, func() (interface{}, error) {
			return db.QueryUser(ctx, id)
		})
		resultCh <- result
		errCh <- err
	}()

	select {
	case <-ctx.Done():
		return nil, ctx.Err()
	case err := <-errCh:
		if err != nil { return nil, err }
		return (<-resultCh).(*User), nil
	}
}
```

**Handler-Level Isolation (Important):**

```go
// BAD - global group shared across all handlers
var group singleflight.Group // All requests share same group!

// GOOD - per-handler or per-resource group
type UserHandler struct {
	userGroup    singleflight.Group
	profileGroup singleflight.Group
}

func (h *UserHandler) GetUser(ctx context.Context, id string) (*User, error) {
	result, err, _ := h.userGroup.Do(id, func() (interface{}, error) {
		return db.QueryUser(ctx, id)
	})
	if err != nil { return nil, err }
	return result.(*User), nil
}

// GOOD - per-request group in middleware
func SingleFlightMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		group := &singleflight.Group{} // New group per request
		r = r.WithContext(context.WithValue(r.Context(), "singleflight", group))
		next.ServeHTTP(w, r)
	})
}

func GetUserFromRequest(r *http.Request, id string) (*User, error) {
	group := r.Context().Value("singleflight").(*singleflight.Group)
	result, err, _ := group.Do(id, func() (interface{}, error) {
		return db.QueryUser(r.Context(), id)
	})
	if err != nil { return nil, err }
	return result.(*User), nil
}
```

### SOP 2: Ristretto — In-Memory Cache (L1)

**Basic Setup:**

```go
import "github.com/dgraph-io/ristretto"

cache, err := ristretto.NewCache(&ristretto.Config{
	NumCounters: 1e7,   // 10x expected unique items
	MaxCost:     1 << 30, // 1GB (bytes as cost unit)
	BufferItems: 64,    // Optimal value
	Metrics:     true,  // Enable statistics (~10% overhead)
})
if err != nil { log.Fatal(err) }

// IMPORTANT: Call Wait() after Set to flush buffers
cache.Set("user:123", userData, int64(len(userData)))
cache.Wait() // Block until buffer is processed
```

**Cache-Aside Pattern:**

```go
func getUserFromCacheOrDB(ctx context.Context, id string) (*User, error) {
	// L1: Check in-memory cache
	if item, found := cache.Get("user:" + id); found {
		return item.(*User), nil // Cache hit — return immediately
	}

	// L2: Query database (with singleflight to prevent thundering herd)
	user, err := getUserWithSingleflight(ctx, id)
	if err != nil { return nil, err }

	// Set in cache with cost = serialized size
	data, _ := json.Marshal(user)
	cache.Set("user:"+id, user, int64(len(data)))
	cache.Wait() // Ensure write completes before returning

	return user, nil
}
```

**TTL Support (Ristretto v0.2+):**

```go
// Set with 5-minute TTL
cache.SetWithTTL("session:abc", sessionData, int64(len(sessionData)), 5*time.Minute)
cache.Wait()

// Check remaining TTL
if ttl, found := cache.GetTTL("session:abc"); found {
	log.Printf("Session expires in %v", ttl)
}
```

**Metrics Monitoring:**

```go
func logCacheStats() {
	ratio := cache.Metrics.Ratio() // Hit ratio (0.0 - 1.0)
	log.Printf("Hit ratio: %.2f | Hits: %d | Misses: %d",
		ratio, cache.Metrics.Hits(), cache.Metrics.Misses())

	log.Printf("Keys added: %d | Keys evicted: %d",
		cache.Metrics.KeysAdded(), cache.Metrics.KeysEvicted())
}
```

**Cleanup:**

```go
func ShutdownCache() {
	cache.Clear() // Empty cache and reset counters
	cache.Close() // Stop goroutines, close channels
}
```

### SOP 3: Rueidis — Redis Client with Client Side Caching (L2)

**Basic Setup:**

```go
import "github.com/redis/rueidis"

client, err := rueidis.NewClient(rueidis.ClientOption{
	InitAddress:      []string{"localhost:6379"},
	MaxConnPoolSize:  50,
	DisableCache:     false, // Enable Client Side Caching (default)
})
if err != nil { log.Fatal(err) }
defer client.Close()

// Simple GET/SET
var result string
err = client.Do(ctx, client.B().Get().Key("user:123").Build()).Scan(&result)
err = client.Do(ctx, client.B().Set().Key("user:123").Value(jsonData).Ex().Seconds(300).Build()).Error()
```

**Client Side Caching (CSC):**

```go
// CSC automatically caches Redis responses on the client side
// No manual cache management needed — rueidis handles it

// First call: fetches from Redis, caches locally
user1, err := getUserFromRedis(ctx, "123") // Miss → Redis query + local cache

// Second call (same key): served from local CSC cache
user2, err := getUserFromRedis(ctx, "123") // Hit → no network roundtrip!

func getUserFromRedis(ctx context.Context, id string) (*User, error) {
	var result User
	err := client.Do(ctx, client.B().Get().Key("user:"+id).Build()).Scan(&result)
	return &result, err // CSC handles caching automatically
}
```

**CSC Invalidation (Pub/Sub):**

```go
// When data changes, publish invalidation key to Redis channel
func invalidateUserCache(client *rueidis.Client, id string) error {
	// Publish to the special __rueidis__cache_invalidate channel
	return client.Do(ctx,
		client.B().Publish().Channel("__rueidis__cache_invalidate").Message("user:" + id).Build(),
	).Error()
}

// rueidis automatically receives pub/sub messages and removes matching keys from CSC cache
```

**Cache-Aside with Rueidis:**

```go
func getUserWithCSC(ctx context.Context, id string) (*User, error) {
	var user User
	err := client.Do(ctx, client.B().Get().Key("user:"+id).Build()).Scan(&user)

	if err == rueidis.Nil {
		user, err = db.QueryUser(ctx, id) // L3: Database fallback
		if err != nil { return nil, err }
		data, _ := json.Marshal(user)
		client.Do(ctx, client.B().Set().Key("user:"+id).Value(data).Ex().Seconds(300).Build())
	} else if err != nil {
		return nil, err
	}

	return &user, nil
}
```

### SOP 4: Multi-Level Cache Architecture (L1 → L2 → DB)

**Complete Layered Cache:**

```go
type CacheService struct {
	l1     *ristretto.Cache
	client *rueidis.Client
	db     *sql.DB
}

func NewCacheService() (*CacheService, error) {
	l1, err := ristretto.NewCache(&ristretto.Config{
		NumCounters: 1e6, MaxCost: 1 << 28, // 256MB for L1
		BufferItems: 64,
	})
	if err != nil { return nil, err }

	client, err := rueidis.NewClient(rueidis.ClientOption{
		InitAddress:     []string{"localhost:6379"},
		DisableCache:    false, // Enable CSC
		MaxConnPoolSize: 50,
	})
	if err != nil { return nil, err }

	return &CacheService{l1: l1, client: client, db: sqlDB}, nil
}

func (s *CacheService) GetUser(ctx context.Context, id string) (*User, error) {
	// L1: In-memory cache (fastest)
	if item, found := s.l1.Get("user:" + id); found {
		return item.(*User), nil // Hit — return immediately
	}

	// L2: Redis Client Side Cache (automatic local caching)
	var user User
	err := s.client.Do(ctx, s.client.B().Get().Key("user:"+id).Build()).Scan(&user)

	if err == rueidis.Nil {
		// L3: Database fallback
		user, err = s.db.QueryUser(ctx, id)
		if err != nil { return nil, err }
		data, _ := json.Marshal(user)
		s.client.Do(ctx, s.client.B().Set().Key("user:"+id).Value(data).Ex().Seconds(300).Build())
	} else if err != nil {
		return nil, err
	}

	// Populate L1 for future requests
	cacheData, _ := json.Marshal(user)
	s.l1.Set("user:"+id, &user, int64(len(cacheData)))
	s.l1.Wait()

	return &user, nil
}
```

### SOP 5: Cache Invalidation Strategies

**TTL-Based (Recommended for most cases):**

```go
// Set with automatic expiration
client.Do(ctx, client.B().Set().Key("user:"+id).Value(data).Ex().Seconds(300).Build())

// Ristretto TTL
cache.SetWithTTL("session:"+token, sessionData, cost, 5*time.Minute)
cache.Wait()
```

**Write-Through (Update cache on write):**

```go
func updateUser(ctx context.Context, id string, data *User) error {
	// Update database first
	if err := s.db.UpdateUser(ctx, id, data); err != nil { return err }

	// Then update both caches
	jsonData, _ := json.Marshal(data)

	// L2: Redis
	s.client.Do(ctx, s.client.B().Set().Key("user:"+id).Value(jsonData).Ex().Seconds(300).Build())

	// L1: In-memory
	s.l1.Set("user:"+id, data, int64(len(jsonData)))
	s.l1.Wait()

	return nil
}
```

**Cache-Aside (Lazy Loading — Default):**

```go
func getUser(ctx context.Context, id string) (*User, error) {
	// Try L1 → L2 → DB on read
	user, err := s.GetUser(ctx, id) // Multi-level lookup
	if err != nil { return nil, err }
	return user, nil
}
```

**Pub/Sub Invalidation (For critical data):**

```go
func invalidateUser(ctx context.Context, id string) error {
	// L1: Remove from ristretto
	s.l1.Del("user:" + id)

	// L2: Invalidate CSC via pub/sub
	return s.client.Do(ctx,
		s.client.B().Publish().Channel("__rueidis__cache_invalidate").Message("user:"+id).Build(),
	).Error()
}
```

## Tool Integration

| Task | Tool | Usage |
|------|------|-------|
| Install dependencies | `run_command` | `go get golang.org/x/sync github.com/dgraph-io/ristretto github.com/redis/rueidis` |
| Verify cache stats | `read_file` + `search_files` | Find `cache.Metrics.Ratio()` usage in monitoring code |
| Test CSC invalidation | `run_command` | Run pub/sub test with `redis-cli PUBLISH __rueidis__cache_invalidate "key"` |

## Anti-Patterns & Guardrails

❌ **Never** use a global `singleflight.Group` across all handlers — isolate per-handler or per-request:
```go
// BAD - thundering herd from different resources collide
var group singleflight.Group // All Do() calls share same namespace!

// GOOD - separate groups per resource type
type Handler struct { userGroup, profileGroup singleflight.Group }
```

❌ **Never** forget `cache.Wait()` after `Set()` in ristretto — writes are buffered:
```go
// BAD - value may not be in cache yet on next Get()
cache.Set("key", value, 1)
cache.Get("key") // May miss!

// GOOD - wait for buffer flush
cache.Set("key", value, 1)
cache.Wait() // Ensures write completes
```

❌ **Never** store large objects in L1 (ristretto) — keep it for hot, small data:
```go
// BAD - fills L1 with huge payloads
cache.Set("report:2024", fullReportData, int64(len(fullReportData))) // 50MB!

// GOOD - only cache identifiers or summaries
cache.Set("user:123:id", userId, 1) // Small key lookup
```

❌ **Never** disable Client Side Caching in rueidis unless you need real-time data:
```go
// BAD - loses CSC benefits
client, _ := rueidis.NewClient(rueidis.ClientOption{DisableCache: true})

// GOOD - enable CSC for automatic local caching
client, _ := rueidis.NewClient(rueidis.ClientOption{DisableCache: false})
```

⚠️ **Always** set appropriate TTLs — never let cache entries live forever without invalidation.

⚠️ **Always** call `cache.Close()` on shutdown to stop ristretto goroutines and prevent leaks.

## Best Practices

1. Use per-handler/singleflight groups — never a global group for all requests
2. Call `cache.Wait()` after every `Set()` in ristretto to ensure buffer flush
3. Keep L1 (ristretto) small and hot — use for frequently accessed, low-latency data
4. Enable CSC in rueidis (`DisableCache: false`) for automatic Redis client-side caching
5. Use pub/sub invalidation (`__rueidis__cache_invalidate`) when data changes externally
6. Monitor hit ratios with `cache.Metrics.Ratio()` — tune NumCounters/MaxCost based on actual usage

## References

- [x/sync/singleflight](https://pkg.go.dev/golang.org/x/sync/singleflight)
- [ristretto Documentation](https://pkg.go.dev/github.com/dgraph-io/ristretto)
- [rueidis Documentation](https://github.com/redis/rueidis)
- [Ristretto Blog Post](https://blog.dgraph.io/post/introducing-ristretto-high-perf-go-cache/)

---

**Skill successfully created:** `skills/go-cache/SKILL.md`

This skill is now ready. Please renew the Skill Index before using it.
