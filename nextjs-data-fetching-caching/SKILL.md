---
name: nextjs-data-fetching-caching
description: Next.js의 데이터 페칭, 캐싱, 리발리데이션 전략을 다룹니다. 서버 컴포넌트 fetch, cache directive, revalidate, ISR, Route Handlers 등 데이터 흐름과 성능 최적화를 학습합니다.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nextjs
  tags: [nextjs, data-fetching, caching, revalidation, isr, server-components]
---

# Next.js Data Fetching & Caching

## Overview

Next.js App Router는 React Server Components 기반의 새로운 캐싱 모델을 제공합니다. `fetch` 기본 동작이 자동 캐싱되며, `cache`, `revalidate`, `connection` 지시어로 세밀하게 제어합니다.

**핵심 원칙**: 서버 컴포넌트의 `fetch`는 기본적으로 캐시됨 → 명시적 리발리데이션으로 제어.

---

## SOP: Step-by-Step Procedures

### 1. 기본 데이터 페칭 (서버 컴포넌트)

```jsx
// app/posts/page.jsx — 서버 컴포넌트에서 직접 fetch
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
1. 서버 컴포넌트에서 `async/await`로 직접 데이터 페칭
2. `fetch`는 기본적으로 캐싱됨 (Next.js 내부 캐시)
3. 클라이언트 컴포넌트에서는 `useEffect` + 상태 관리 필요

### 2. fetch 옵션으로 캐싱 제어

```jsx
// 캐싱 비활성화 (매 요청마다 페칭)
const res = await fetch('https://api.example.com/data', {
  cache: 'no-store'
});

// 60초 동안 캐시 (TTL 기반 리발리데이션)
const res = await fetch('https://api.example.com/data', {
  next: { revalidate: 60 }
});

// 무한정 캐시 (수동 리발리데이션 필요)
const res = await fetch('https://api.example.com/data', {
  cache: 'force-cache' // 기본값
});
```

**캐싱 옵션 비교**:

| 옵션 | 동작 | 사용처 |
|------|------|--------|
| `cache: 'force-cache'` (기본) | 무한정 캐시 | 변경 안 되는 정적 데이터 |
| `next: { revalidate: N }` | N초마다 재페칭 | 주기적 업데이트 필요 데이터 |
| `cache: 'no-store'` | 매 요청 페칭 | 실시간/개인화 데이터 |

### 3. use cache 지시문 (Cache Components)

```jsx
// app/lib/data.js — 캐시된 데이터 함수
import { cache } from 'react';

const getCachedPosts = cache(async () => {
  return fetch('https://api.example.com/posts').then((res) => res.json());
});

export async function getPosts() {
  // 같은 요청 내에서 중복 페칭 방지
  return getCachedPosts();
}

// 캐시 범위 제어
import { unstable_cache } from 'next/cache';

const getPostById = unstable_cache(
  async (id) => {
    const res = await fetch(`https://api.example.com/posts/${id}`);
    return res.json();
  },
  ['post'], // 캐시 태그
  { revalidate: 3600, tags: ['post'] }
);
```

### 4. 리발리데이션 (Revalidation)

```jsx
// 수동 리발리데이션 — Route Handler에서 호출
import { revalidateTag, revalidatePath } from 'next/cache';

export async function POST(request) {
  // 데이터 업데이트 후...
  revalidateTag('posts');        // 태그 기반 리발리데이션
  revalidatePath('/blog');       // 경로 기반 리발리데이션
  return Response.json({ ok: true });
}

// ISR (Incremental Static Regeneration)
export const dynamic = 'force-dynamic'; // 또는
export const revalidate = 3600;         // 1시간마다 정적 재생성
```

**Step-by-Step**:
1. `revalidateTag`로 특정 태그의 캐시 무효화
2. `revalidatePath`로 특정 경로의 캐시 무효화
3. Route Handler(POST/PUT)에서 호출하여 실시간 업데이트
4. 정적 페이지에 `revalidate` 설정으로 ISR 구현

### 5. Route Handlers (API 엔드포인트)

```jsx
// app/api/posts/route.js — GET 엔드포인트
import { NextResponse } from 'next/server';

export async function GET() {
  const posts = await fetch('https://api.example.com/posts').then((r) => r.json());
  return NextResponse.json(posts);
}

// POST 엔드포인트 + 리발리데이션
export async function POST(request) {
  const body = await request.json();
  
  // 데이터 저장 로직...
  
  revalidateTag('posts');
  return NextResponse.json({ success: true });
}

// 동적 라우트
// app/api/posts/[id]/route.js
export async function GET(request, { params }) {
  const post = await fetch(`https://api.example.com/posts/${params.id}`).then((r) => r.json());
  return NextResponse.json(post);
}
```

### 6. connection 지시문

```jsx
// 연결 재사용 (동일 도메인 fetch 간)
const res = await fetch('https://api.example.com/data', {
  next: { tags: ['data'], revalidate: 300 },
});

// connection 옵션으로 연결 풀 제어
fetch('https://api.example.com/data', {
  next: { cache: 'force-cache' }
}); // 기본적으로 동일 도메인 연결 재사용
```

---

## Tool Integration

| 도구 | 사용 목적 | 예시 |
|------|-----------|------|
| `run_command` | Next.js 개발 서버 실행 | `npm run dev`, 캐시 무효화 테스트 |
| `search_files` | fetch 패턴 검색 | `grep -r "revalidate" app/` |
| `read_file` | 데이터 함수 분석 | lib/data.js, route.js 확인 |
| `edit_file` | 캐싱 옵션 수정 | revalidate 값 변경 |

---

## Anti-Patterns & Guardrails

❌ **클라이언트 컴포넌트에서 서버 데이터 직접 fetch** — 서버 컴포넌트에서 페칭 후 prop 전달  
❌ **모든 fetch에 `no-store` 사용** — 캐싱 비활성화 = 매 요청 페칭 = 성능 저하  
❌ **Route Handler에서 HTML 반환** — Route Handler는 JSON/API 응답 전용  
❌ **revalidate 없이 정적 생성만** — 실시간 데이터에는 revalidate 필수  

⚠️ **대용량 데이터 캐시** — 메모리 사용량 증가, 적절한 TTL 설정 필요  
⚠️ **중복 fetch 방지 안 함** — `cache()` 래퍼로 같은 요청 중복 제거  

---

## Best Practices

1. **서버 컴포넌트에서 직접 fetch** — 클라이언트-서버 데이터 전달 최소화
2. **적절한 캐싱 전략 선택** — 정적/반정적/실시간 데이터 구분
3. **`revalidateTag` 활용** — 태그 기반 리발리데이션으로 세밀한 제어
4. **Route Handler로 API 분리** — 비즈니스 로직과 프론트엔드 분리
5. **`cache()` 래퍼 사용** — 같은 요청 내 중복 fetch 방지
6. **`no-store`는 실시간 데이터에만** — 개인화/실시간 정보에 한정

---

## References

- [Fetching Data](https://nextjs.org/docs/app/getting-started/fetching-data)
- [Caching](https://nextjs.org/docs/app/getting-started/caching)
- [Revalidating](https://nextjs.org/docs/app/getting-started/revalidating)
- [Route Handlers](https://nextjs.org/docs/app/getting-started/route-handlers)
- [Directives: use cache](https://nextjs.org/docs/app/api-reference/directives/use-cache)
