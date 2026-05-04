---
name: tailwindcss-nextjs
description: Covers Tailwind CSS v4 integration with Next.js App Router, including installation, theme customization, utility classes, responsive design, dark mode, state variants (hover/focus), and component patterns. Use when styling Next.js applications with Tailwind CSS.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: tailwindcss
  tags: [tailwindcss, nextjs, css, responsive-design, dark-mode, utility-classes]
---

# Tailwind CSS + Next.js Integration

## Overview

Tailwind CSS v4 provides a zero-runtime, JIT-based styling system that integrates seamlessly with Next.js App Router. It scans source files for class names and generates optimized static CSS. This skill covers installation, configuration, core concepts, and common patterns specific to the Next.js + Tailwind combination.

**Core Principle**: Utility-first CSS — compose designs directly in HTML/JSX using pre-built classes rather than writing custom CSS.

---

## SOP: Step-by-Step Procedures

### 1. Installation with Next.js (Vite or PostCSS)

```bash
# Using Vite (recommended for new projects)
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install tailwindcss @tailwindcss/vite

# Using PostCSS (standard Next.js approach)
npx create-next-app@latest my-app
cd my-app
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

**postcss.config.mjs**:
```js
const config = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
export default config;
```

**tailwind.config.ts**:
```ts
import type { Config } from 'tailwindcss';

const config: Config = {
  content: [
    './pages/**/*.{js,ts,jsx,tsx,mdx}',
    './components/**/*.{js,ts,jsx,tsx,mdx}',
    './app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {},
  },
  plugins: [],
};

export default config;
```

**Step-by-Step**:
1. Install Tailwind CSS + PostCSS/autoprefixer
2. Initialize config files (`tailwind.config.ts`, `postcss.config.mjs`)
3. Add `@tailwind` directives to global CSS
4. Verify with `npm run dev` — classes should work

### 2. Global CSS Setup (App Router)

**app/globals.css**:
```css
@import "tailwindcss";

/* Optional: custom theme overrides */
@theme {
  --color-primary: #3b82f6;
  --color-secondary: #8b5cf6;
  --font-sans: 'Inter', var(--font-__inter_fallback);
  --breakpoint-sm: 640px;
}

/* Optional: custom utilities */
@utility prose-custom {
  @apply prose prose-lg prose-blue;
}
```

**app/layout.tsx**:
```tsx
import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import './globals.css';

const inter = Inter({ subsets: ['latin'] });

export const metadata: Metadata = {
  title: 'My App',
  description: 'Built with Next.js + Tailwind CSS',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>{children}</body>
    </html>
  );
}
```

### 3. Core Utility Classes — Layout & Spacing

**Flexbox**:
```jsx
// Horizontal centering
<div className="flex items-center justify-center min-h-screen">
  <div>Centered</div>
</div>

// Sidebar layout
<div className="flex h-screen">
  <aside className="w-64 bg-gray-100 p-4">Sidebar</aside>
  <main className="flex-1 p-6 overflow-auto">{/* content */}</main>
</div>

// Card grid
<div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
  {cards.map(card => <Card key={card.id} {...card} />)}
</div>
```

**Spacing**:
```jsx
// Padding scale: p-0, p-1 (0.25rem), ..., p-4 (1rem), p-8 (2rem)
<div className="p-6">Content with padding</div>

// Margin scale: m-2, mt-4, mx-auto (horizontal auto centering)
<div className="mt-8 mb-4 mx-auto max-w-3xl">Centered content</div>

// Gap for flex/grid gaps
<div className="flex gap-4 items-center">
  <Button />
  <Button variant="secondary" />
</div>
```

**Sizing**:
```jsx
// Width utilities
<div className="w-full max-w-screen-xl mx-auto px-4">
  {/* Constrained width, centered */}
</div>

// Height utilities (use with flex parent)
<div className="h-screen overflow-y-auto">
  <div className="min-h-[calc(100vh-4rem)]">{/* content */}</div>
</div>

// Aspect ratio for images/videos
<img src="/hero.jpg" className="w-full h-64 object-cover rounded-lg" />
```

### 4. Responsive Design Breakpoints

```jsx
// Mobile-first responsive classes
<div className="
  p-4              /* mobile */
  sm:p-6           /* ≥640px */
  md:p-8           /* ≥768px */
  lg:p-12          /* ≥1024px */
  xl:p-16          /* ≥1280px */
">
  Responsive padding
</div>

// Conditional rendering by breakpoint
<div className="hidden md:block">Desktop only</div>
<div className="block md:hidden">Mobile only</div>

// Column reflow
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
  {items.map(item => <Card key={item.id} {...item} />)}
</div>

// Navigation pattern
<nav className="flex flex-col md:flex-row items-start md:items-center gap-2 p-4">
  {/* Stacks vertically on mobile, horizontal on desktop */}
</nav>
```

**Breakpoint Reference**:

| Prefix | Min Width | Use Case |
|--------|-----------|----------|
| (none) | 0px | Mobile first (base) |
| `sm:` | 640px | Small phones → tablets |
| `md:` | 768px | Tablets → laptops |
| `lg:` | 1024px | Desktops |
| `xl:` | 1280px | Large screens |
| `2xl:` | 1536px | Extra large displays |

### 5. Dark Mode

```jsx
// Using class strategy (recommended with Next.js)
// In tailwind.config.ts: darkMode: 'class'

// Toggle button in client component
'use client';
import { useTheme } from 'next-themes';

export function ThemeToggle() {
  const { theme, setTheme } = useTheme();
  return (
    <button onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}>
      Toggle Dark Mode
    </button>
  );
}

// Usage in JSX — dark: prefix
<div className="bg-white dark:bg-gray-900 text-gray-900 dark:text-white">
  <h1 className="text-xl font-bold">Title</h1>
  <p className="mt-2 text-gray-600 dark:text-gray-300">Description</p>
</div>

// With next-themes provider in layout.tsx
import { ThemeProvider } from 'next-themes';

export default function RootLayout({ children }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <ThemeProvider attribute="class" defaultTheme="system">
          {children}
        </ThemeProvider>
      </body>
    </html>
  );
}
```

### 6. Hover, Focus & Other States

```jsx
// Hover states — hover: prefix
<button className="bg-blue-500 text-white px-4 py-2 rounded-lg
  hover:bg-blue-600 focus:ring-2 focus:ring-blue-500 focus:outline-none
  active:scale-95 transition-colors duration-200">
  Click Me
</button>

// Focus-visible for keyboard navigation
<input className="border border-gray-300 rounded px-3 py-2
  focus:border-blue-500 focus:ring-2 focus:ring-blue-200 outline-none" />

// Group hover — parent hover affects children
<div className="group relative">
  <img src="/product.jpg" className="w-full h-48 object-cover rounded-lg" />
  <div className="absolute inset-0 bg-black/50 opacity-0 group-hover:opacity-100
    transition-opacity duration-200 flex items-center justify-center">
    <span className="text-white font-medium">View Details</span>
  </div>
</div>

// Peer — sibling selector (for form labels)
<div className="relative">
  <input id="email" type="email" className="peer w-full px-3 py-2 border rounded-lg" />
  <label htmlFor="email" className="absolute left-3 top-2 text-gray-500
    peer-focus:-top-2 peer-focus:left-1 peer-focus:text-xs peer-focus:bg-white
    peer-focus:px-1 transition-all">Email</label>
</div>

// Disabled state
<button disabled className="bg-blue-500 text-white px-4 py-2 rounded-lg
  opacity-50 cursor-not-allowed" />
```

### 7. Common Component Patterns

**Button**:
```jsx
// app/components/Button.tsx
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'outline';
  size?: 'sm' | 'md' | 'lg';
  children: React.ReactNode;
  className?: string;
}

export function Button({ variant = 'primary', size = 'md', children, className = '' }: ButtonProps) {
  const variants = {
    primary: 'bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-500',
    secondary: 'bg-gray-200 text-gray-900 hover:bg-gray-300 focus:ring-gray-400',
    outline: 'border-2 border-blue-600 text-blue-600 hover:bg-blue-50 focus:ring-blue-500',
  };

  const sizes = {
    sm: 'px-3 py-1.5 text-sm',
    md: 'px-4 py-2 text-base',
    lg: 'px-6 py-3 text-lg',
  };

  return (
    <button className={`rounded-lg font-medium transition-colors duration-200
      focus:outline-none focus:ring-2 focus:ring-offset-2 ${variants[variant]} ${sizes[size]} ${className}`}>
      {children}
    </button>
  );
}
```

**Card**:
```jsx
// app/components/Card.tsx
export function Card({ title, description, image, href }: {
  title: string;
  description?: string;
  image?: string;
  href?: string;
}) {
  const Content = (
    <>
      {image && <img src={image} alt={title} className="w-full h-48 object-cover rounded-t-xl" />}
      <div className="p-6">
        <h3 className="text-lg font-semibold text-gray-900">{title}</h3>
        {description && <p className="mt-2 text-gray-600">{description}</p>}
      </div>
    </>
  );

  return href ? (
    <a href={href} className="block bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden hover:shadow-md transition-shadow">
      {Content}
    </a>
  ) : (
    <div className="bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden">
      {Content}
    </div>
  );
}
```

**Input Field**:
```jsx
// app/components/Input.tsx
export function Input({ label, error, ...props }: {
  label?: string;
  error?: string;
} & React.InputHTMLAttributes<HTMLInputElement>) {
  return (
    <div className="w-full">
      {label && <label className="block text-sm font-medium text-gray-700 mb-1">{label}</label>}
      <input
        {...props}
        className={`w-full px-3 py-2 border rounded-lg transition-colors duration-200
          focus:outline-none focus:ring-2 ${error
            ? 'border-red-500 focus:border-red-500 focus:ring-red-200'
            : 'border-gray-300 focus:border-blue-500 focus:ring-blue-200'}`}
      />
      {error && <p className="mt-1 text-sm text-red-600">{error}</p>}
    </div>
  );
}
```

**Navigation Bar**:
```jsx
// app/components/Navbar.tsx
export function Navbar() {
  return (
    <nav className="sticky top-0 z-50 bg-white/80 backdrop-blur-md border-b border-gray-200">
      <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div className="flex items-center justify-between h-16">
          <a href="/" className="text-xl font-bold text-gray-900">Logo</a>
          <div className="hidden md:flex items-center gap-6">
            <a href="/features" className="text-gray-600 hover:text-gray-900 transition-colors">Features</a>
            <a href="/pricing" className="text-gray-600 hover:text-gray-900 transition-colors">Pricing</a>
            <a href="/about" className="text-gray-600 hover:text-gray-900 transition-colors">About</a>
          </div>
          <div className="flex items-center gap-3">
            <Button variant="outline" size="sm">Sign In</Button>
            <Button size="sm">Get Started</Button>
          </div>
        </div>
      </div>
    </nav>
  );
}
```

### 8. Integration with Next.js App Router

**Server Component (default)**:
```tsx
// app/page.tsx — Server component, no 'use client' needed
import { Card } from '@/components/Card';

export default function HomePage() {
  return (
    <main className="max-w-7xl mx-auto px-4 py-12">
      <h1 className="text-4xl font-bold text-gray-900 mb-8">Welcome</h1>
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {data.map(item => <Card key={item.id} {...item} />)}
      </div>
    </main>
  );
}
```

**Client Component (when interactivity needed)**:
```tsx
// app/components/SearchBar.tsx — Client component for state
'use client';
import { useState } from 'react';

export function SearchBar() {
  const [query, setQuery] = useState('');
  return (
    <div className="relative max-w-md mx-auto">
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
        className="w-full px-4 py-2 pl-10 border border-gray-300 rounded-lg
          focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
      />
      <svg className="absolute left-3 top-2.5 w-5 h-5 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
        <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" />
      </svg>
    </div>
  );
}
```

**Loading Skeleton**:
```tsx
// app/components/Skeleton.tsx
export function CardSkeleton() {
  return (
    <div className="bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden animate-pulse">
      <div className="h-48 bg-gray-200" />
      <div className="p-6 space-y-3">
        <div className="h-5 bg-gray-200 rounded w-3/4" />
        <div className="h-4 bg-gray-200 rounded w-full" />
        <div className="h-4 bg-gray-200 rounded w-2/3" />
      </div>
    </div>
  );
}
```

---

## Tool Integration

| Tool | Purpose | Example |
|------|---------|---------|
| `run_command` | Install Tailwind, run dev server | `npm install -D tailwindcss postcss autoprefixer`, `npm run dev` |
| `search_files` | Find class usage patterns | `grep -r "bg-blue-500" app/` |
| `read_file` | Analyze config files | Check tailwind.config.ts, globals.css |
| `edit_file` | Modify styles/config | Update theme variables, add utilities |

---

## Anti-Patterns & Guardrails

❌ **Writing custom CSS when utility exists** — Always check Tailwind docs first  
❌ **Using inline styles with Tailwind classes** — Conflicts, defeats purpose of utility-first  
❌ **Overriding Tailwind with `!important`** — Breaks cascade, makes maintenance impossible  
❌ **Putting all classes in one component** — Extract reusable components (Button, Card, Input)  
❌ **Using arbitrary values excessively** — `w-[37px]` → define in theme instead (`--width-37: 37px`)  
❌ **Forgetting focus states** — Accessibility violation; always add `focus:ring-*`  

⚠️ **Dark mode with `class` strategy requires JS** — Use `next-themes` for SSR compatibility  
⚠️ **Large bundle from unused classes** — Tailwind purges in production, but verify build output  
⚠️ **Arbitrary values for colors** — Prefer theme variables (`text-primary`) over hex codes  

---

## Best Practices

1. **Mobile-first responsive design** — Base styles for mobile, add prefixes for larger screens
2. **Extract reusable components** — Button, Card, Input should be shared across pages
3. **Use `@theme` for custom values** — Centralize colors, fonts, breakpoints in globals.css
4. **Add focus rings to all interactive elements** — Accessibility requirement (`focus:ring-2`)
5. **Prefer `dark:` over manual class toggling** — Let Tailwind handle dark mode variants
6. **Use `group` and `peer` for complex interactions** — Avoid JavaScript for simple hover/sibling states
7. **Keep utility classes organized** — Group related utilities, use multi-line className strings
8. **Run `npm run build` to verify purge** — Ensure unused CSS is removed in production

---

## References

- [Installation: Next.js](https://tailwindcss.com/docs/installation/framework-guides/nextjs)
- [Styling with Utility Classes](https://tailwindcss.com/docs/styling-with-utility-classes)
- [Theme Variables](https://tailwindcss.com/docs/theme)
- [Hover, Focus, and Other States](https://tailwindcss.com/docs/hover-focus-and-other-states)
- [Responsive Design](https://tailwindcss.com/docs/responsive-design)
- [Dark Mode](https://tailwindcss.com/docs/dark-mode)
- [Adding Custom Styles](https://tailwindcss.com/docs/adding-custom-styles)
- [Functions and Directives](https://tailwindcss.com/docs/functions-and-directives)
