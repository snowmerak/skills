---
name: nextjs-data-fetching-caching
description: Covers Next.js data fetching, caching, and revalidation strategies including server component fetch, cache directives, revalidate, ISR, Route Handlers. Use when optimizing data flow and performance in Next.js applications.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nextjs
  tags: [nextjs, data-fetching, caching, revalidation, isr, server-components]
---

# Next.js Data Fetching & Caching

## Overview

Next.js App Router provides a new caching model based on React Server Components. The `fetch` default behavior is automatic caching, controlled precisely with `cache`, `revalidate`, and `connection` directives.

**Core Principle**: Server component `fetch` is cached by default → control via explicit revalidation.

---

## SOP: Step-by-Step Procedures

### 1. Basic Data Fetching (Server Components)

```jsx
// app/posts/page.jsx — Direct fetch in server component
async function getPosts() {
  const res = await fetch('https://api.example.com/posts');
  return res.json();
}

export default async function PostsPage() {
  const posts = await getPosts();

  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

**Step-by-Step**:
1. Fetch data directly with `async/await` in server components
2. `fetch` is cached by default (Next.js internal cache)
3. Client components require `useEffect` + state management

### 2. Control Caching with fetch Options

```jsx
// Disable caching (fetch on every request)
const res = await fetch('https://api.example.com/data', {
  cache: 'no-store'
});

// Cache for 60 seconds (TTL-based revalidation)
const res = await fetch('https://api.example.com/data', {
  next: { revalidate: 60 }
});

// Cache indefinitely (manual revalidation required)
const res = await fetch('https://api.example.com/data', {
  cache: 'force-cache' // default
});
```

**Caching Options Comparison**:

| Option | Behavior | Use Case |
|--------|----------|----------|
| `cache: 'force-cache'` (default) | Cache indefinitely | Static data that doesn't change |
| `next: { revalidate: N }` | Refetch every N seconds | Data needing periodic updates |
| `cache: 'no-store'` | Fetch on every request | Real-time/personalized data |

### 3. use cache Directive (Cache Components)

```jsx
// app/lib/data.js — Cached data function
import { cache } from 'react';

const getCachedPosts = cache(async () => {
  return fetch('https://api.example.com/posts').then((res) => res.json());
});

export async function getPosts() {
  // Prevent duplicate fetching within same request
  return getCachedPosts();
}

// Control cache scope
import { unstable_cache } from 'next/cache';

const getPostById = unstable_cache(
  async (id) => {
    const res = await fetch(`https://api.example.com/posts/${id}`);
    return res.json();
  },
  ['post'], // Cache tag
  { revalidate: 3600, tags: ['post'] }
);
```

### 4. Revalidation

```jsx
// Manual revalidation — called from Route Handler
import { revalidateTag, revalidatePath } from 'next/cache';

export async function POST(request) {
  // After updating data...
  revalidateTag('posts');        // Tag-based revalidation
  revalidatePath('/blog');       // Path-based revalidation
  return Response.json({ ok: true });
}

// ISR (Incremental Static Regeneration)
export const dynamic = 'force-dynamic'; // or
export const revalidate = 3600;         // Static regeneration every hour
```

**Step-by-Step**:
1. Invalidate specific tags with `revalidateTag`
2. Invalidate specific paths with `revalidatePath`
3. Call from Route Handler (POST/PUT) for real-time updates
4. Implement ISR by setting `revalidate` on static pages

### 5. Route Handlers (API Endpoints)

```jsx
// app/api/posts/route.js — GET endpoint
import { NextResponse } from 'next/server';

export async function GET() {
  const posts = await fetch('https://api.example.com/posts').then((r) => r.json());
  return NextResponse.json(posts);
}

// POST endpoint + revalidation
export async function POST(request) {
  const body = await request.json();
  
  // Data saving logic...
  
  revalidateTag('posts');
  return NextResponse.json({ success: true });
}

// Dynamic route
// app/api/posts/[id]/route.js
export async function GET(request, { params }) {
  const post = await fetch(`https://api.example.com/posts/${params.id}`).then((r) => r.json());
  return NextResponse.json(post);
}
```

### 6. connection Directive

```jsx
// Connection reuse (between same-domain fetches)
const res = await fetch('https://api.example.com/data', {
  next: { tags: ['data'], revalidate: 300 },
});

// Control connection pool with connection option
fetch('https://api.example.com/data', {
  next: { cache: 'force-cache' }
}); // By default, same-domain connections are reused
```

---

## Tool Integration

| Tool | Purpose | Example |
|------|---------|---------|
| `run_command` | Run Next.js dev server | `npm run dev`, test cache invalidation |
| `search_files` | Search for fetch patterns | `grep -r "revalidate" app/` |
| `read_file` | Analyze data functions | Check lib/data.js, route.js |
| `edit_file` | Modify caching options | Change revalidate values |

---

## Anti-Patterns & Guardrails

❌ **Directly fetching server data in client components** — Fetch in server component and pass via props  
❌ **Using `no-store` for all fetches** — Disabling cache = fetch on every request = performance degradation  
❌ **Returning HTML from Route Handler** — Route Handlers are for JSON/API responses only  
❌ **Static generation without revalidate** — Revalidation is mandatory for real-time data  

⚠️ **Caching large datasets** — Increases memory usage, set appropriate TTL  
⚠️ **Not preventing duplicate fetches** — Use `cache()` wrapper to eliminate duplicates in same request  

---

## Best Practices

1. **Fetch directly in server components** — Minimize client-server data passing
2. **Choose appropriate caching strategy** — Distinguish between static/semi-static/real-time data
3. **Leverage `revalidateTag`** — Fine-grained control via tag-based revalidation
4. **Separate API with Route Handlers** — Separate business logic from frontend
5. **Use `cache()` wrapper** — Prevent duplicate fetches within same request
6. **Reserve `no-store` for real-time data only** — Limit to personalized/real-time info

---

## References

- [Fetching Data](https://nextjs.org/docs/app/getting-started/fetching-data)
- [Caching](https://nextjs.org/docs/app/getting-started/caching)
- [Revalidating](https://nextjs.org/docs/app/getting-started/revalidating)
- [Route Handlers](https://nextjs.org/docs/app/getting-started/route-handlers)
- [Directives: use cache](https://nextjs.org/docs/app/api-reference/directives/use-cache)
