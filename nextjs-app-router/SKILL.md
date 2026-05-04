---
name: nextjs-app-router
description: Next.js App Router의 핵심 아키텍처를 다룹니다. 레이아웃, 페이지, 라우팅, 서버/클라이언트 컴포넌트 구분, 링크 및 네비게이션, 에러 처리 등 App Router 기반 애플리케이션 구조 설계에 사용합니다.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nextjs
  tags: [nextjs, app-router, server-components, client-components, routing, layouts]
---

# Next.js App Router

## Overview

Next.js 13+의 App Router는 React Server Components를 기반으로 한 새로운 라우팅 시스템입니다. 파일 시스템 기반 라우팅, 레이아웃 중첩, 서버/클라이언트 컴포넌트 분리가 핵심입니다.

**핵심 원칙**: 기본적으로 모든 컴포넌트는 **서버 컴포넌트**이며, `use client` 지시어로만 클라이언트 컴포넌트가 됩니다.

---

## SOP: Step-by-Step Procedures

### 1. 프로젝트 구조 이해

```
app/
├── layout.js          # 루트 레이아웃 (전체 페이지 공유)
├── page.js            # 루트 페이지 (/)
├── loading.js         # 로딩 UI (Suspense 자동 적용)
├── error.js           # 에러 UI (Boundary)
├── not-found.js       # 404 페이지
├── globals.css        # 전역 CSS
├── favicon.ico        # 파비콘
├── about/
│   └── page.js        # /about
├── blog/
│   ├── page.js        # /blog (목록)
│   └── [slug]/
│       └── page.js    # /blog/[slug] (동적 라우팅)
├── dashboard/
│   ├── layout.js      # /dashboard 하위 레이아웃
│   └── page.js        # /dashboard
└── (auth)/            # Route Group (URL에 영향 없음)
    └── login/
        └── page.js    # /login
```

### 2. 레이아웃과 페이지 설계

```jsx
// app/layout.js — 루트 레이아웃
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

// app/dashboard/layout.js — 하위 레이아웃
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
1. `layout.js`는 같은 라우트 그룹 내의 모든 페이지에서 공유
2. 중첩 레이아웃 가능 (루트 → 그룹 → 동적 세그먼트)
3. `children` prop으로 하위 페이지 렌더링
4. `loading.js`, `error.js`는 레이아웃과 함께 사용

### 3. 라우팅 패턴

```jsx
// 정적 라우팅: app/about/page.js → /about
// 동적 라우팅: app/blog/[slug]/page.js → /blog/[slug]
// 병렬 라우팅: app/@sidebar/page.js + app/@main/page.js
// 인터셉팅 라우팅: app/(auth)/login/page.js → /login (URL에 그룹명 안 나타남)

// 동적 세그먼트 예시
app/blog/[slug]/page.js       // /blog/hello-world
app/dashboard/[[...catchAll]]/page.js  // /dashboard/* (옵셔널 catch-all)
```

**Step-by-Step**:
1. 폴더명으로 정적 라우트 생성
2. `[param]`으로 동적 세그먼트
3. `[[...catchAll]]`으로 옵셔널 catch-all
4. `(group)`으로 URL에 영향 없는 그룹화

### 4. 서버/클라이언트 컴포넌트 구분

```jsx
// app/components/Header.jsx — 기본: 서버 컴포넌트
export default function Header() {
  return <header>...</header>;
}

// app/components/SearchBar.jsx — 클라이언트 컴포넌트 필요 시
'use client';

import { useState } from 'react';

export default function SearchBar() {
  const [query, setQuery] = useState('');
  return <input value={query} onChange={(e) => setQuery(e.target.value)} />;
}

// app/page.jsx — 서버 컴포넌트에서 클라이언트 컴포넌트 사용
import SearchBar from '@/components/SearchBar';

export default function Page() {
  return (
    <div>
      <SearchBar /> {/* 클라이언트 컴포넌트 */}
      <DataContent /> {/* 서버 컴포넌트 */}
    </div>
  );
}
```

**Step-by-Step**:
1. 파일 상단에 `'use client'` → 클라이언트 컴포넌트
2. `'use client'` 없음 → 서버 컴포넌트 (기본)
3. 서버 컴포넌트에서 클라이언트 컴포넌트 import 가능 (역은 불가)
4. 클라이언트 컴포넌트는 `useState`, `useEffect` 등 사용 가능

### 5. 에러 처리

```jsx
// app/error.js — 에러 경계
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

// app/not-found.js — 404 페이지
export default function NotFound() {
  return <h2>Not Found</h2>;
}

// 프로그래매틱 에러 발생
import { notFound } from 'next/navigation';
import { forbidden } from 'next/navigation';

if (!data) notFound();
if (!isAuthorized) forbidden();
```

### 6. 링크 및 네비게이션

```jsx
import Link from 'next/link';

// 기본 링크
<Link href="/about">About</Link>

// 동적 라우트
<Link href={`/blog/${slug}`}>{title}</Link>

// 새 탭 열기 (클라이언트 사이드 네비게이션 비활성화)
<Link href="https://external.com" target="_blank">External</Link>

// prefetch 설정
<Link href="/about" prefetch={false}>About</Link>
```

---

## Tool Integration

| 도구 | 사용 목적 | 예시 |
|------|-----------|------|
| `run_command` | Next.js 개발 서버 실행 | `npm run dev`, `npx next lint` |
| `search_files` | 컴포넌트/라우트 검색 | `grep -r "use client" app/` |
| `read_file` | 레이아웃/페이지 구조 분석 | layout.js, page.js 확인 |
| `edit_file` | 라우트/컴포넌트 수정 | 새 페이지 추가, 레이아웃 변경 |

---

## Anti-Patterns & Guardrails

❌ **클라이언트 컴포넌트 남용** — 모든 컴포넌트에 `'use client'` 붙이지 말 것  
❌ **서버 컴포넌트에서 useState/useEffect 사용** — `'use client'` 필요  
❌ **layout.js에 페이지 콘텐츠 넣기** — layout은 구조, page는 콘텐츠 분리  
❌ **동적 라우트에서 모든 데이터 페칭** — `generateStaticParams`로 정적 생성 고려  

⚠️ **클라이언트 컴포넌트는 번들 크기 증가** — 필요한 경우에만 사용  
⚠️ **중첩 레이아웃은 children만 렌더링** — 중복 구조 피할 것  

---

## Best Practices

1. **서버 컴포넌트 기본** — 클라이언트 기능 필요 시에만 `'use client'`
2. **레이아웃 중첩 활용** — 공통 UI는 상위 layout에서 관리
3. **Route Groups 사용** — `/dashboard/admin`, `/dashboard/user` 구조화
4. **동적 세그먼트 최소화** — 정적 생성 가능한 경우 `generateStaticParams` 활용
5. **error.js + not-found.js 필수** — 모든 라우트 그룹에 에러 처리

---

## References

- [Layouts and Pages](https://nextjs.org/docs/app/getting-started/layouts-and-pages)
- [Linking and Navigating](https://nextjs.org/docs/app/getting-started/linking-and-navigating)
- [Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components)
- [Error Handling](https://nextjs.org/docs/app/getting-started/error-handling)
- [File Conventions: layout.js](https://nextjs.org/docs/app/api-reference/file-conventions/layout)
- [File Conventions: page.js](https://nextjs.org/docs/app/api-reference/file-conventions/page)
