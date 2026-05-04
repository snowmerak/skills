---
name: shadcn-ui-nextjs
description: Covers shadcn/ui integration with Next.js App Router, including CLI initialization, components.json configuration, component installation/management, theming (CSS variables), dark mode, form integration (React Hook Form), and common component patterns. Use when building UIs with shadcn/ui in Next.js applications.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: shadcnui
  tags: [shadcn-ui, nextjs, components, theming, dark-mode, forms, radix-ui]
---

# shadcn/ui + Next.js Integration

## Overview

shadcn/ui is not a component library — it's a collection of **reusable components** you copy into your project. Built on Radix UI (headless, accessible primitives) and styled with Tailwind CSS. You own the code; customize freely.

**Core Principle**: Components live in your project → full control over styling and behavior. CLI handles installation, updates, and management.

---

## SOP: Step-by-Step Procedures

### 1. Prerequisites & Installation

```bash
# Ensure Tailwind CSS v4 is installed first
npm install tailwindcss @tailwindcss/vite
# or for PostCSS setup
npm install -D tailwindcss postcss autoprefixer

# Initialize shadcn/ui
npx shadcn@latest init
```

**Interactive Init Prompts**:
```
Would you like to use CSS variables for theming? › Yes
Where are your Tailwind CSS styles located? › app/globals.css
Project is a Next.js app. Would you like to initialize a new config? › Yes
```

### 2. components.json Configuration

**components.json** (created by CLI):
```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "app/globals.css",
    "baseColor": "neutral",
    "cssVariables": true,
    "prefix": ""
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  }
}
```

**Key Settings**:

| Setting | Values | Description |
|---------|--------|-------------|
| `style` | `new-york`, `default` (deprecated) | Visual style preset |
| `rsc` | `true` / `false` | Enable React Server Components support |
| `tsx` | `true` / `false` | TypeScript vs JavaScript output |
| `baseColor` | `neutral`, `stone`, `zinc`, `slate`, `gray`, `olive`, `mauve`, `mist`, `taupe` | Base color palette for theme tokens |
| `cssVariables` | `true` / `false` | Use CSS variables (recommended) or inline utilities |

**⚠️ Important**: `style`, `baseColor`, and `cssVariables` **cannot be changed after initialization**. To switch, delete components and re-initialize.

### 3. Tailwind CSS v4 + shadcn/ui Setup

**app/globals.css**:
```css
@import "tailwindcss";

/* Import shadcn/ui theme tokens */
@source '../../../node_modules/@radix-ui/themes';

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-card: var(--card);
  --color-card-foreground: var(--card-foreground);
  --color-popover: var(--popover);
  --color-popover-foreground: var(--popover-foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --color-secondary: var(--secondary);
  --color-secondary-foreground: var(--secondary-foreground);
  --color-muted: var(--muted);
  --color-muted-foreground: var(--muted-foreground);
  --color-accent: var(--accent);
  --color-accent-foreground: var(--accent-foreground);
  --color-destructive: var(--destructive);
  --color-destructive-foreground: var(--destructive-foreground);
  --color-border: var(--border);
  --color-input: var(--input);
  --color-ring: var(--ring);
  --color-chart-1: var(--chart-1);
  --color-chart-2: var(--chart-2);
  --color-chart-3: var(--chart-3);
  --color-chart-4: var(--chart-4);
  --color-chart-5: var(--chart-5);
  --radius-sm: calc(var(--radius) - 4px);
  --radius-md: calc(var(--radius) - 2px);
  --radius-lg: var(--radius);
  --radius-xl: calc(var(--radius) + 4px);
}

/* Root CSS variables (CSS Variables mode) */
:root {
  --background: oklch(1 0 0);
  --foreground: oklch(0.145 0 0);
  --card: oklch(1 0 0);
  --card-foreground: oklch(0.145 0 0);
  --popover: oklch(1 0 0);
  --popover-foreground: oklch(0.145 0 0);
  --primary: oklch(0.205 0 0);
  --primary-foreground: oklch(0.985 0 0);
  --secondary: oklch(0.97 0 0);
  --secondary-foreground: oklch(0.205 0 0);
  --muted: oklch(0.97 0 0);
  --muted-foreground: oklch(0.556 0 0);
  --accent: oklch(0.97 0 0);
  --accent-foreground: oklch(0.205 0 0);
  --destructive: oklch(0.577 0.245 27.325);
  --destructive-foreground: oklch(0.577 0.245 27.325);
  --border: oklch(0.922 0 0);
  --input: oklch(0.922 0 0);
  --ring: oklch(0.708 0 0);
  --chart-1: oklch(0.646 0.222 41.116);
  --chart-2: oklch(0.6 0.118 184.704);
  --chart-3: oklch(0.398 0.07 227.392);
  --chart-4: oklch(0.828 0.189 84.429);
  --chart-5: oklch(0.769 0.188 70.08);
  --radius: 0.5rem;
}

/* Dark mode */
.dark {
  --background: oklch(0.145 0 0);
  --foreground: oklch(0.985 0 0);
  --card: oklch(0.145 0 0);
  --card-foreground: oklch(0.985 0 0);
  --popover: oklch(0.145 0 0);
  --popover-foreground: oklch(0.985 0 0);
  --primary: oklch(0.985 0 0);
  --primary-foreground: oklch(0.205 0 0);
  --secondary: oklch(0.269 0 0);
  --secondary-foreground: oklch(0.985 0 0);
  --muted: oklch(0.269 0 0);
  --muted-foreground: oklch(0.708 0 0);
  --accent: oklch(0.269 0 0);
  --accent-foreground: oklch(0.985 0 0);
  --destructive: oklch(0.396 0.141 25.723);
  --destructive-foreground: oklch(0.637 0.237 25.331);
  --border: oklch(0.269 0 0);
  --input: oklch(0.269 0 0);
  --ring: oklch(0.439 0 0);
  --chart-1: oklch(0.488 0.243 264.376);
  --chart-2: oklch(0.696 0.17 162.48);
  --chart-3: oklch(0.769 0.188 70.08);
  --chart-4: oklch(0.627 0.265 303.9);
  --chart-5: oklch(0.645 0.246 16.439);
}

/* Body */
body {
  background-color: var(--background);
  color: var(--foreground);
}
```

### 4. lib/utils.ts — cn() Utility

**lib/utils.ts**:
```ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

/**
 * Merge Tailwind classes with proper specificity handling.
 * Always use this instead of template literals for class concatenation.
 */
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

**Usage**:
```tsx
import { cn } from '@/lib/utils';

// Correct — merges classes, last one wins
<div className={cn("p-4 rounded-lg", isActive && "bg-blue-500")}>
  Content
</div>

// Incorrect — template literal concatenation may cause conflicts
<div className={`p-4 rounded-lg ${isActive ? 'bg-blue-500' : ''}`}>
```

### 5. Installing Components via CLI

```bash
# Install a single component
npx shadcn@latest add button
npx shadcn@latest add card dialog input label

# Install multiple components at once
npx shadcn@latest add button card dialog input label select textarea

# Install with custom name (alias)
npx shadcn@latest add button --name MyButton

# Update all components to latest version
npx shadcn@latest update

# Remove a component
npx shadcn@latest remove button

# List installed components
npx shadcn@latest list
```

**Component Locations**:
- UI components → `components/ui/` (e.g., `components/ui/button.tsx`)
- App components → `components/` (your custom components)
- Utilities → `lib/utils.ts`
- Hooks → `hooks/`

### 6. Common Component Patterns

**Button**:
```tsx
// components/ui/button.tsx — Installed via CLI
import { Button } from '@/components/ui/button';

export function LoginActions() {
  return (
    <div className="flex gap-2">
      <Button>Login</Button>
      <Button variant="outline">Sign Up</Button>
      <Button variant="ghost" disabled>Disabled</Button>
      <Button variant="destructive">Delete Account</Button>
      <Button size="sm">Small</Button>
      <Button size="lg" className="w-full">Full Width</Button>
    </div>
  );
}

// Button variants: default, destructive, outline, secondary, ghost, link
// Button sizes: default, sm, lg, icon
```

**Card**:
```tsx
import { Card, CardHeader, CardFooter, CardTitle, CardDescription, CardContent } from '@/components/ui/card';

export function ProductCard({ title, description, price }) {
  return (
    <Card className="w-[350px]">
      <CardHeader>
        <CardTitle>{title}</CardTitle>
        <CardDescription>{description}</CardDescription>
      </CardHeader>
      <CardContent>
        <p className="text-2xl font-bold">${price}</p>
      </CardContent>
      <CardFooter className="flex justify-between">
        <Button variant="outline">Cancel</Button>
        <Button>Add to Cart</Button>
      </CardFooter>
    </Card>
  );
}
```

**Input + Label**:
```tsx
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';

export function LoginForm() {
  return (
    <form className="grid w-full max-w-sm items-center gap-4">
      <div className="space-y-2">
        <Label htmlFor="email">Email</Label>
        <Input id="email" type="email" placeholder="name@example.com" />
      </div>
      <div className="space-y-2">
        <Label htmlFor="password">Password</Label>
        <Input id="password" type="password" />
      </div>
      <Button type="submit">Sign In</Button>
    </form>
  );
}
```

**Dialog**:
```tsx
import { Button } from '@/components/ui/button';
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '@/components/ui/dialog';

export function EditProfileDialog() {
  return (
    <Dialog>
      <DialogTrigger asChild>
        <Button variant="outline">Edit Profile</Button>
      </DialogTrigger>
      <DialogContent className="sm:max-w-[425px]">
        <DialogHeader>
          <DialogTitle>Edit profile</DialogTitle>
          <DialogDescription>
            Make changes to your profile here. Click save when you're done.
          </DialogDescription>
        </DialogHeader>
        <div className="grid gap-4 py-4">
          {/* Form fields */}
        </div>
        <DialogFooter>
          <Button type="submit">Save changes</Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

**Select**:
```tsx
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';

export function RoleSelector() {
  return (
    <Select>
      <SelectTrigger className="w-[180px]">
        <SelectValue placeholder="Select a role" />
      </SelectTrigger>
      <SelectContent>
        <SelectItem value="admin">Admin</SelectItem>
        <SelectItem value="user">User</SelectItem>
        <SelectItem value="viewer">Viewer</SelectItem>
      </SelectContent>
    </Select>
  );
}
```

**Tabs**:
```tsx
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';

export function SettingsTabs() {
  return (
    <Tabs defaultValue="account" className="w-[400px]">
      <TabsList>
        <TabsTrigger value="account">Account</TabsTrigger>
        <TabsTrigger value="billing">Billing</TabsTrigger>
        <TabsTrigger value="notifications">Notifications</TabsTrigger>
      </TabsList>
      <TabsContent value="account">Account settings...</TabsContent>
      <TabsContent value="billing">Billing info...</TabsContent>
      <TabsContent value="notifications">Notification preferences...</TabsContent>
    </Tabs>
  );
}
```

**Toast**:
```tsx
'use client';

import { Button } from '@/components/ui/button';
import { useToast } from '@/hooks/use-toast';
import { ToastAction } from '@/components/ui/toast';

export function NotificationDemo() {
  const { toast } = useToast();

  return (
    <Button
      variant="outline"
      onClick={() => {
        toast({
          title: 'Scheduled: Catch up',
          description: 'Friday, February 10, 2023 at 5:57 PM',
          action: (
            <ToastAction altText="Goto schedule to undo">Undo</ToastAction>
          ),
        });
      }}
    >
      Add to calendar
    </Button>
  );
}

// Don't forget Toaster in layout.tsx
import { Toaster } from '@/components/ui/toaster';

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        {children}
        <Toaster />
      </body>
    </html>
  );
}
```

**Table**:
```tsx
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table';

const users = [
  { id: 1, name: 'Alice', email: 'alice@example.com' },
  { id: 2, name: 'Bob', email: 'bob@example.com' },
];

export function UserTable() {
  return (
    <Table>
      <TableHeader>
        <TableRow>
          <TableHead>Name</TableHead>
          <TableHead>Email</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        {users.map((user) => (
          <TableRow key={user.id}>
            <TableCell>{user.name}</TableCell>
            <TableCell>{user.email}</TableCell>
          </TableRow>
        ))}
      </TableBody>
    </Table>
  );
}
```

### 7. Dark Mode Integration

**Using next-themes with shadcn/ui**:

```bash
npm install next-themes
```

**app/layout.tsx**:
```tsx
import { ThemeProvider } from 'next-themes';
import { Toaster } from '@/components/ui/toaster';

export default function RootLayout({ children }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body className="antialiased">
        <ThemeProvider
          attribute="class"
          defaultTheme="system"
          enableSystem
          disableTransitionOnChange
        >
          {children}
          <Toaster />
        </ThemeProvider>
      </body>
    </html>
  );
}
```

**Theme Toggle Component**:
```tsx
'use client';

import { useTheme } from 'next-themes';
import { Button } from '@/components/ui/button';
import { Moon, Sun } from 'lucide-react';

export function ThemeToggle() {
  const { theme, setTheme } = useTheme();

  return (
    <Button
      variant="ghost"
      size="icon"
      onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}
    >
      <Sun className="h-5 w-5 rotate-0 scale-100 transition-all dark:-rotate-90 dark:scale-0" />
      <Moon className="absolute h-5 w-5 rotate-90 scale-0 transition-all dark:rotate-0 dark:scale-100" />
      <span className="sr-only">Toggle theme</span>
    </Button>
  );
}
```

### 8. Form Integration — React Hook Form + shadcn/ui

```bash
npm install react-hook-form zod @hookform/resolvers
npx shadcn@latest add form
```

**lib/validations.ts**:
```ts
import { z } from 'zod';

export const userSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email address'),
  role: z.enum(['admin', 'user', 'viewer'], {
    required_error: 'Please select a role',
  }),
});

export type UserFormValues = z.infer<typeof userSchema>;
```

**components/user-form.tsx**:
```tsx
'use client';

import { zodResolver } from '@hookform/resolvers/zod';
import { useForm } from 'react-hook-form';
import { Button } from '@/components/ui/button';
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form';
import { Input } from '@/components/ui/input';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';
import { userSchema, type UserFormValues } from '@/lib/validations';

export function UserForm() {
  const form = useForm<UserFormValues>({
    resolver: zodResolver(userSchema),
    defaultValues: {
      name: '',
      email: '',
      role: undefined,
    },
  });

  function onSubmit(data: UserFormValues) {
    console.log('Submitted:', data);
    // API call...
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-8">
        <FormField
          control={form.control}
          name="name"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Name</FormLabel>
              <FormControl>
                <Input placeholder="John Doe" {...field} />
              </FormControl>
              <FormDescription>Your public display name.</FormDescription>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input type="email" placeholder="john@example.com" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="role"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Role</FormLabel>
              <Select onValueChange={field.onChange} defaultValue={field.value}>
                <FormControl>
                  <SelectTrigger>
                    <SelectValue placeholder="Select a role" />
                  </SelectTrigger>
                </FormControl>
                <SelectContent>
                  <SelectItem value="admin">Admin</SelectItem>
                  <SelectItem value="user">User</SelectItem>
                  <SelectItem value="viewer">Viewer</SelectItem>
                </SelectContent>
              </Select>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit">Submit</Button>
      </form>
    </Form>
  );
}
```

### 9. Customizing Components

**Override a component**:
```tsx
// components/ui/button.tsx — Copy and modify the installed file
import * as React from 'react';
import { Slot } from '@radix-ui/react-slot';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

const buttonVariants = cva(
  'inline-flex items-center justify-center whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
        // Add custom variant
        gradient: 'bg-gradient-to-r from-blue-500 to-purple-600 text-white hover:from-blue-600 hover:to-purple-700',
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : 'button';
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    );
  }
);
Button.displayName = 'Button';

export { Button, buttonVariants };
```

### 10. Directory Structure

```
my-app/
├── app/
│   ├── layout.tsx           # Root layout with ThemeProvider + Toaster
│   ├── page.tsx             # Home page
│   └── globals.css          # Tailwind + shadcn theme variables
├── components/
│   ├── ui/                  # shadcn/ui components (managed by CLI)
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── dialog.tsx
│   │   ├── input.tsx
│   │   └── ...              # 50+ components
│   └── my-component.tsx     # Your custom components
├── hooks/                   # Custom hooks (use-toast, etc.)
│   └── use-toast.ts
├── lib/
│   └── utils.ts             # cn() utility + validation schemas
├── components.json          # shadcn/ui configuration
├── tailwind.config.ts       # Tailwind CSS configuration
└── tsconfig.json            # Path aliases (@/, @/*)
```

---

## Tool Integration

| Tool | Purpose | Example |
|------|---------|---------|
| `run_command` | Install/update components via CLI | `npx shadcn@latest add button`, `npx shadcn@latest update` |
| `search_files` | Find component usage patterns | `grep -r "from '@/components/ui" app/` |
| `read_file` | Analyze component source code | Check components/ui/button.tsx for customization |
| `edit_file` | Customize installed components | Modify buttonVariants, add new variants |

---

## Anti-Patterns & Guardrails

❌ **Modifying node_modules/@radix-ui** — shadcn copies Radix UI into your project; never modify the package directly  
❌ **Using template literals for class concatenation** — Always use `cn()` utility to prevent Tailwind conflicts  
❌ **Ignoring TypeScript errors in components** — shadcn is fully typed; fix type errors properly  
❌ **Adding `'use client'` unnecessarily** — Only add when component uses hooks, events, or browser APIs  
❌ **Overriding all component styles with custom CSS** — Use Tailwind classes and `cn()` instead  

⚠️ **Cannot change style/baseColor after init** — Requires full re-installation of components  
⚠️ **Forgetting Toaster in layout.tsx** — Toast notifications won't render  
⚠️ **Using shadcn components without Radix UI dependencies** — CLI handles this automatically; don't skip  

---

## Best Practices

1. **Always use `cn()` for class merging** — Prevents Tailwind specificity conflicts
2. **Keep UI components in `components/ui/`** — Separates managed vs custom code
3. **Use CSS variables (cssVariables: true)** — Enables seamless dark mode switching
4. **Validate forms with Zod + React Hook Form** — Type-safe validation pipeline
5. **Run `npx shadcn@latest update` regularly** — Keep components up to date
6. **Customize via copy-and-edit, not forks** — You own the code; maintain it directly
7. **Add Toaster in root layout** — Required for toast notifications
8. **Use `next-themes` with class attribute** — Best SSR compatibility

---

## References

- [Installation](https://ui.shadcn.com/docs/installation)
- [components.json](https://ui.shadcn.com/docs/components-json)
- [Theming](https://ui.shadcn.com/docs/theming)
- [Dark Mode](https://ui.shadcn.com/docs/dark-mode)
- [CLI Reference](https://ui.shadcn.com/docs/cli)
- [Forms: React Hook Form](https://ui.shadcn.com/docs/forms/react-hook-form)
- [Components Overview](https://ui.shadcn.com/docs/components)
