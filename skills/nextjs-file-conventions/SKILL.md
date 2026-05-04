---
name: nextjs-file-conventions
description: Covers Next.js App Router file system conventions including layout, page, loading, error, not-found, route, intercepting routes, parallel routes, metadata files. Use when working with all file convention roles and combinations in Next.js applications.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nextjs
  tags: [nextjs, file-conventions, route-groups, parallel-routes, intercepting-routes, metadata]
---

# Next.js File Conventions

## Overview

Next.js App Router uses file-system-based routing. Specific filenames and folder names automatically determine behavior in Next.js. This skill covers the role and combination methods of all file conventions.

**Core Principle**: Filename = Behavior. Follow standard filenames and Next.js handles everything automatically.

---

## SOP: Step-by-Step Procedures

### 1. Core File Conventions

```
app/
├── layout.js      # Layout (nestable, requires children prop)
├── page.js        # Page (route entry point, export default required)
├── loading.js     # Loading UI (Suspense applied automatically, use with layouts)
├── error.js       # Error boundary (UI, provides reset function)
├── not-found.js   # 404 page (global or per-route)
├── route.js       # API endpoint (GET/POST/PUT/DELETE, etc.)
├── template.js    # Similar to layout but doesn't preserve state (re-renders on transition)
└── globals.css    # Global CSS
```

**Characteristics of Each File**:

| File | Required Export | Characteristics |
|------|-----------------|-----------------|
| `layout.js` | `default` | children prop, nestable |
| `page.js` | `default` | Route entry point, supports async |
| `loading.js` | `default` | Auto-generates Suspense boundary |
| `error.js` | `default(error, reset)` | Displays error UI, recovery via reset |
| `not-found.js` | `default` | Auto-sets 404 status code |
| `route.js` | `GET/POST/PUT/DELETE` | API endpoint |

### 2. Route Groups (Grouping Without URL Impact)

```
app/
├── (auth)/          # URL appears as: /login, /register
│   ├── login/page.js
│   └── register/page.js
├── (dashboard)/     # URL appears as: /dashboard/admin
│   ├── admin/layout.js
│   └── admin/page.js
└── dashboard/       # Regular folder
    └── page.js      # /dashboard
```

**Step-by-Step**:
1. Group with `(group-name)` folder name
2. Group name not reflected in URL path
3. Layouts shared within group
4. Useful for structuring `/admin` and `/dashboard/admin`

### 3. Parallel Routes (Concurrent Rendering)

```
app/
├── layout.js        # Shared layout
├── page.js          # /
├── @sidebar/page.js   # slot: sidebar
├── @main/page.js      # slot: main
└── @alternate/
    ├── page.js        # Default alternate content
    └── feed/page.js   # Replaces on /feed
```

```jsx
// app/layout.jsx
export default function Layout({ children, sidebar, main, alternate }) {
  return (
    <div className="layout">
      <aside>{sidebar}</aside>
      <main>{main || children}</main>
    </div>
  );
}
```

**Step-by-Step**:
1. Define slots with `@slot-name` folder names
2. Access slot props in layout
3. Renders children if `null`
4. Useful for A/B testing, dynamic content replacement

### 4. Intercepting Routes (Route Interception)

```
app/
├── posts/
│   └── [slug]/
│       └── page.js      # /posts/hello-world
├── posts/
│   └── (detail)/
│       └── [slug]/
│           └── page.js  # /posts/(detail)/hello-world
├── @overlay/
│   └── (detail)/
│       └── [slug]/
│           └── page.js  # Intercept: displays as overlay on /posts/hello-world
```

**Step-by-Step**:
1. Group intercepted routes with `(group)`
2. Create intercept layout with `@slot/(group)/` structure
3. Renders as overlay/modal at same URL
4. Shows overlay without page transition on navigation

### 5. Metadata Files (Metadata)

```
app/
├── favicon.ico          # Favicon
├── icon.png             # Icon (OG, etc.)
├── opengraph-image.png  # OG image
├── twitter-image.png    # Twitter card image
├── manifest.json        # PWA manifest
├── robots.txt           # SEO robot directives
└── sitemap.xml          # Sitemap

// Programmatic metadata
export function generateMetadata({ params }) {
  return {
    title: 'Post Title',
    description: 'Post Description',
    openGraph: { title: 'OG Title', images: ['/og.png'] }
  };
}
```

**Step-by-Step**:
1. Place static metadata files in `app/` root
2. Use `generateMetadata` function for dynamic metadata
3. Can generate OG images programmatically with `opengraph-image.js`

### 6. Dynamic Routes

```jsx
// app/blog/[slug]/page.js — Single dynamic segment
export default function Page({ params }) {
  return <h1>{params.slug}</h1>;
}

// app/dashboard/[[...catchAll]]/page.js — Optional catch-all
export default function Page({ params }) {
  return <p>{params.catchAll?.join('/') || 'Home'}</p>;
}

// Static generation: generateStaticParams
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then((r) => r.json());
  return posts.map((post) => ({ slug: post.slug }));
}
```

### 7. Route Segment Config (Segment Settings)

```jsx
// app/dashboard/page.js — Per-segment settings
export const dynamic = 'auto';      // auto / force-dynamic / force-static
export const dynamicParams = true;  // Whether to handle dynamic segments
export const revalidate = 3600;     // ISR revalidation period (seconds)
export const maxDuration = 60;      # Maximum function execution time (seconds)

// preferredRegion: Data center region setting
export const preferredRegion = 'auto'; // auto / iad1 / sin1, etc.
```

---

## Tool Integration

| Tool | Purpose | Example |
|------|---------|---------|
| `run_command` | Run Next.js dev server | `npm run dev`, build test |
| `search_files` | Search file convention patterns | `grep -r "generateMetadata" app/` |
| `read_file` | Analyze convention files | Check layout.js, page.js structure |
| `edit_file` | Add new routes/files | Create page.js, route.js |

---

## Anti-Patterns & Guardrails

❌ **Putting layout code in page.js** — Layout is for structure, page is for content  
❌ **Confusing Route Groups with regular folders** — `(group)` has no URL impact, regular folders do  
❌ **Missing slots in parallel routes** — All slots must be defined, null handling required  

⚠️ **Intercepting routes complexity** — Difficult to debug, use only when necessary  
⚠️ **Misusing catch-all routes** — Limit to specific patterns only  

---

## Best Practices

1. **Follow filename standards** — Use layout/page/loading/error/not-found conventions
2. **Structure with Route Groups** — Logical grouping like `/admin`, `/dashboard`
3. **Use metadata.js or generateMetadata** — SEO optimization
4. **Leverage generateStaticParams** — Performance improvement via static generation
5. **Include error.js mandatory** — Error boundaries for all route groups

---

## References

- [File Conventions Overview](https://nextjs.org/docs/app/api-reference/file-conventions)
- [layout.js](https://nextjs.org/docs/app/api-reference/file-conventions/layout)
- [page.js](https://nextjs.org/docs/app/api-reference/file-conventions/page)
- [route.js](https://nextjs.org/docs/app/api-reference/file-conventions/route)
- [Dynamic Routes](https://nextjs.org/docs/app/api-reference/file-conventions/dynamic-routes)
- [Parallel Routes](https://nextjs.org/docs/app/api-reference/file-conventions/parallel-routes)
- [Intercepting Routes](https://nextjs.org/docs/app/api-reference/file-conventions/intercepting-routes)
- [Route Groups](https://nextjs.org/docs/app/api-reference/file-conventions/route-groups)
- [Metadata Files](https://nextjs.org/docs/app/api-reference/file-conventions/metadata)
