---
name: nextjs-app-router
description: Covers the core architecture of Next.js App Router, including layouts, pages, routing, server/client component separation, links and navigation, error handling. Use when designing App Router-based application structures.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nextjs
  tags: [nextjs, app-router, server-components, client-components, routing, layouts]
---

# Next.js App Router

## Overview

Next.js 13+ App Router is a new routing system based on React Server Components. File-system-based routing, nested layouts, and server/client component separation are its core features.

**Core Principle**: By default, all components are **server components**; only with the `use client` directive do they become client components.

---

## SOP: Step-by-Step Procedures

### 1. Understand Project Structure

```
app/
├── layout.js          # Root layout (shared across entire page)
├── page.js            # Root page (/)
├── loading.js         # Loading UI (Suspense applied automatically)
├── error.js           # Error UI (Boundary)
├── not-found.js       # 404 page
├── globals.css        # Global CSS
├── favicon.ico        # Favicon
├── about/
│   └── page.js        # /about
├── blog/
│   ├── page.js        # /blog (list)
│   └── [slug]/
│       └── page.js    # /blog/[slug] (dynamic routing)
├── dashboard/
│   ├── layout.js      # Sub-layout under /dashboard
│   └── page.js        # /dashboard
└── (auth)/            # Route Group (no impact on URL)
    └── login/
        └── page.js    # /login
```

### 2. Design Layouts and Pages

```jsx
// app/layout.js — Root layout
export default function RootLayout({ children }) {
  return (
    <html lang="ko">
      <body>
        <Header />
        <main>{children}</main>
        <Footer />
      </body>
    </html>
  );
}

// app/dashboard/layout.js — Sub-layout
export default function DashboardLayout({ children, sidebar }) {
  return (
    <div className="dashboard">
      <Sidebar>{sidebar}</Sidebar>
      <section>{children}</section>
    </div>
  );
}
```

**Step-by-Step**:
1. `layout.js` is shared across all pages in the same route group
2. Nested layouts are possible (root → group → dynamic segment)
3. Render child pages via `children` prop
4. Use with `loading.js`, `error.js`

### 3. Routing Patterns

```jsx
// Static routing: app/about/page.js → /about
// Dynamic routing: app/blog/[slug]/page.js → /blog/[slug]
// Parallel routing: app/@sidebar/page.js + app/@main/page.js
// Intercepting routing: app/(auth)/login/page.js → /login (group name not in URL)

// Dynamic segment examples
app/blog/[slug]/page.js       // /blog/hello-world
app/dashboard/[[...catchAll]]/page.js  // /dashboard/* (optional catch-all)
```

**Step-by-Step**:
1. Create static routes via folder names
2. Use `[param]` for dynamic segments
3. Use `[[...catchAll]]` for optional catch-all
4. Use `(group)` for grouping without URL impact

### 4. Server/Client Component Separation

```jsx
// app/components/Header.jsx — Default: server component
export default function Header() {
  return <header>...</header>;
}

// app/components/SearchBar.jsx — Client component when needed
'use client';

import { useState } from 'react';

export default function SearchBar() {
  const [query, setQuery] = useState('');
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}

// app/page.jsx — Using client component in server component
import SearchBar from '@/components/SearchBar';

export default function Page() {
  return (
    <div>
      <SearchBar /> {/* Client component */}
      <DataContent /> {/* Server component */}
    </div>
  );
}
```

**Step-by-Step**:
1. `'use client'` at top of file → client component
2. No `'use client'` → server component (default)
3. Server components can import client components (reverse not possible)
4. Client components can use `useState`, `useEffect`, etc.

### 5. Error Handling

```jsx
// app/error.js — Error boundary
'use client';

export default function Error({ error, reset }) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <pre>{error.message}</pre>
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}

// app/not-found.js — 404 page
export default function NotFound() {
  return <h2>Not Found</h2>;
}

// Programmatic error throwing
import { notFound } from 'next/navigation';
import { forbidden } from 'next/navigation';

if (!data) notFound();
if (!isAuthorized) forbidden();
```

### 6. Links and Navigation

```jsx
import Link from 'next/link';

// Basic link
<Link href="/about">About</Link>

// Dynamic route
<Link href={`/blog/${slug}`}>{title}</Link>

// Open in new tab (disable client-side navigation)
<Link href="https://external.com" target="_blank">External</Link>

// Configure prefetch
<Link href="/about" prefetch={false}>About</Link>
```

---

## Tool Integration

| Tool | Purpose | Example |
|------|---------|---------|
| `run_command` | Run Next.js dev server | `npm run dev`, `npx next lint` |
| `search_files` | Search components/routes | `grep -r "use client" app/` |
| `read_file` | Analyze layout/page structure | Check layout.js, page.js |
| `edit_file` | Modify routes/components | Add new page, change layout |

---

## Anti-Patterns & Guardrails

❌ **Overusing client components** — Do not add `'use client'` to every component  
❌ **Using useState/useEffect in server components** — Requires `'use client'`  
❌ **Putting page content in layout.js** — Layout is for structure, page is for content  
❌ **Fetching all data in dynamic routes** — Consider `generateStaticParams` for static generation  

⚠️ **Client components increase bundle size** — Use only when necessary  
⚠️ **Nested layouts only render children** — Avoid duplicate structures  

---

## Best Practices

1. **Default to server components** — Only use `'use client'` when client features are needed
2. **Leverage nested layouts** — Manage common UI in parent layout
3. **Use Route Groups** — Structure `/dashboard/admin`, `/dashboard/user`
4. **Minimize dynamic segments** — Use `generateStaticParams` where static generation is possible
5. **Mandatory error.js + not-found.js** — Error handling for all route groups

---

## References

- [Layouts and Pages](https://nextjs.org/docs/app/getting-started/layouts-and-pages)
- [Linking and Navigating](https://nextjs.org/docs/app/getting-started/linking-and-navigating)
- [Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components)
- [Error Handling](https://nextjs.org/docs/app/getting-started/error-handling)
- [File Conventions: layout.js](https://nextjs.org/docs/app/api-reference/file-conventions/layout)
- [File Conventions: page.js](https://nextjs.org/docs/app/api-reference/file-conventions/page)
