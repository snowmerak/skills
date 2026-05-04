---
name: nextjs-directives-server-actions
description: Covers Next.js core directives and server actions including 'use client', 'use server' directives, Cache Components (directive), Server Actions (form submission), and related functions (after, cookies, draftMode). Use when implementing type-safe form submissions and server-client boundaries.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nextjs
  tags: [nextjs, use-client, use-server, server-actions, cache-directive, directives]
---

# Next.js Directives & Server Actions

## Overview

Covers core features of Next.js App Router: `use client`, `use server` directives, Cache Components directive, and Server Actions. Learn to clearly define server-client boundaries and implement type-safe form submissions.

**Core Principle**: Mark functions as server with `'use server'` → callable directly from client (type-safe).

---

## SOP: Step-by-Step Procedures

### 1. 'use client' Directive

```jsx
// app/components/SearchBar.jsx — Client component
'use client';

import { useState, useEffect } from 'react';

export default function SearchBar() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  useEffect(() => {
    if (query.length > 2) {
      fetch(`/api/search?q=${query}`).then((r) => r.json()).then(setResults);
    }
  }, [query]);

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ul>{results.map((r) => <li key={r.id}>{r.title}</li>)}</ul>
    </div>
  );
}
```

**When 'use client' is Required**:
- Using React hooks: `useState`, `useEffect`, `useContext`, etc.
- Event handlers (onClick, onChange, etc.)
- Accessing browser APIs (window, localStorage)
- DOM manipulation needed

### 2. 'use server' Directive — Server Actions

```jsx
// app/actions.js — Server action function
'use server';

import { revalidateTag } from 'next/cache';

export async function createPost(formData) {
  const title = formData.get('title');
  const content = formData.get('content');

  // Database save logic...
  
  revalidateTag('posts');
  return { success: true };
}

// Server action with arguments
export async function deletePost(id) {
  // Delete logic...
  revalidateTag('posts');
}
```

```jsx
// app/posts/create.jsx — Using server action in form
import { createPost } from '@/actions';

export default function CreatePost() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Title" />
      <textarea name="content" placeholder="Content" />
      <button type="submit">Create</button>
    </form>
  );
}

// Passing arguments (using useTransition)
'use client';

import { createPost } from '@/actions';
import { useActionState, useTransition } from 'react';

export default function CreatePostForm() {
  const [state, formAction, isPending] = useActionState(createPost, null);
  
  return (
    <form action={formAction}>
      <input name="title" />
      <button disabled={isPending}>Create</button>
      {state?.error && <p>{state.error}</p>}
    </form>
  );
}
```

**Step-by-Step**:
1. Define server actions in separate file with `'use server'`
2. Import and use in client components
3. Call via `form action={action}` or `onClick={() => action()}`
4. Manage state with `useActionState` + `useTransition`

### 3. Cache Components Directive

```jsx
// app/lib/data.js — Cached data function
import { cache } from 'react';

const getCachedUser = cache(async (userId) => {
  return fetch(`https://api.example.com/users/${userId}`).then((r) => r.json());
});

export async function getUser(userId) {
  // Prevent duplicate fetch within same request
  return getCachedUser(userId);
}

// Private cache (personalized data)
'use cache';
cache({ type: 'private' });

const getPrivateData = cache(async () => {
  return fetch('/api/private').then((r) => r.json());
});

// Remote cache (distributed caching)
'use cache';
cache({ type: 'remote', minAge: 60 });

const getRemoteData = cache(async () => {
  return fetch('https://cdn.example.com/data').then((r) => r.json());
});
```

**Cache Types**:

| Type | Scope | Use Case |
|------|-------|----------|
| Default (omitted) | Server instance | General data |
| `private` | User session | Personalized data |
| `remote` | Distributed cache | CDN/external API |

### 4. Related Functions

```jsx
// cookies() — Read cookies (server component)
import { cookies } from 'next/headers';

export default async function Page() {
  const cookieStore = await cookies();
  const theme = cookieStore.get('theme')?.value;
  return <div>Theme: {theme}</div>;
}

// after() — Execute after response (async operations)
import { after } from 'next/headers';

export default async function Page() {
  after(async () => {
    // Background work after page response
    await logVisit();
    await sendNotification();
  });
  
  return <div>Page Content</div>;
}

// draftMode() — Draft mode
import { draftMode } from 'next/headers';

export default async function Page() {
  const draft = await draftMode();
  if (draft.isEnabled) {
    // Render draft version
  }
  return <div>Content</div>;
}

// connection() — Connection reuse
import { connection } from 'next/cache';

export default async function Page() {
  const conn = await connection();
  const data = await conn.fetch('https://api.example.com/data');
  return <pre>{data}</pre>;
}
```

### 5. Server Actions + Form Validation

```jsx
// app/actions.js
'use server';

import { z } from 'zod';

const postSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
});

export async function createPost(formData) {
  const result = postSchema.safeParse(Object.fromEntries(formData));
  
  if (!result.success) {
    return { error: 'Invalid data', fields: result.error.flatten() };
  }

  // Save logic...
  return { success: true };
}
```

---

## Tool Integration

| Tool | Purpose | Example |
|------|---------|---------|
| `run_command` | Run Next.js dev server | `npm run dev`, test Server Actions |
| `search_files` | Search for 'use server' patterns | `grep -r "'use server'" app/` |
| `read_file` | Analyze action functions | Check actions.js structure |
| `edit_file` | Add/modify server actions | Create new action function |

---

## Anti-Patterns & Guardrails

❌ **Using useState/useEffect in server components** — Requires `'use client'`  
❌ **Directly accessing DB from client components** — Wrap with Server Actions  
❌ **Adding client dependencies to 'use server' functions** — Only pure server code allowed  

⚠️ **Sending large FormData** — Check size limits, consider streaming  
⚠️ **Synchronous operations in Server Actions** — Use `after()` for async processing  

---

## Best Practices

1. **Separate server actions into dedicated files** — actions.js or lib/actions.ts
2. **Mandatory schema validation (zod, etc.)** — Type safety + server-side validation
3. **'use server' at function level** — Apply to functions, not entire file
4. **Leverage revalidateTag** — Invalidate cache after data changes
5. **Provide feedback with useActionState** — Pass success/failure state to client

---

## References

- [Directives: use client](https://nextjs.org/docs/app/api-reference/directives/use-client)
- [Directives: use server](https://nextjs.org/docs/app/api-reference/directives/use-server)
- [Directives: use cache](https://nextjs.org/docs/app/api-reference/directives/use-cache)
- [Mutating Data (Server Actions)](https://nextjs.org/docs/app/getting-started/mutating-data)
- [cookies()](https://nextjs.org/docs/app/api-reference/functions/cookies)
- [after()](https://nextjs.org/docs/app/api-reference/functions/after)
