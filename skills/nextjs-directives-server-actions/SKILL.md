---
name: nextjs-directives-server-actions
description: Next.js의 핵심 지시문과 서버 액션을 다룹니다. 'use client', 'use server' 지시문, Cache Components(directive), Server Actions(폼 제출), 그리고 관련 함수(after, cookies, draftMode) 사용법을 학습합니다.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nextjs
  tags: [nextjs, use-client, use-server, server-actions, cache-directive, directives]
---

# Next.js Directives & Server Actions

## Overview

Next.js App Router의 핵심 기능인 `use client`, `use server` 지시문과 Cache Components directive, Server Actions를 다룹니다. 서버-클라이언트 경계를 명확히 하고, 타입 세이프한 폼 제출을 구현하는 방법을 학습합니다.

**핵심 원칙**: `'use server'`로 함수를 서버로 마킹 → 클라이언트에서 직접 호출 가능 (타입 안전).

---

## SOP: Step-by-Step Procedures

### 1. 'use client' 지시문

```jsx
// app/components/SearchBar.jsx — 클라이언트 컴포넌트
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

**'use client' 필요 조건**:
- `useState`, `useEffect`, `useContext` 등 React 훅 사용
- 이벤트 핸들러 (onClick, onChange 등)
- 브라우저 API 접근 (window, localStorage)
- DOM 조작 필요

### 2. 'use server' 지시문 — Server Actions

```jsx
// app/actions.js — 서버 액션 함수
'use server';

import { revalidateTag } from 'next/cache';

export async function createPost(formData) {
  const title = formData.get('title');
  const content = formData.get('content');

  // 데이터베이스 저장 로직...
  
  revalidateTag('posts');
  return { success: true };
}

// 인자 있는 서버 액션
export async function deletePost(id) {
  // 삭제 로직...
  revalidateTag('posts');
}
```

```jsx
// app/posts/create.jsx — 폼에서 서버 액션 사용
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

// 인자 전달 (useTransition 활용)
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
1. 별도 파일에 `'use server'`로 서버 액션 정의
2. 클라이언트 컴포넌트에서 import하여 사용
3. `form action={action}` 또는 `onClick={() => action()}`으로 호출
4. `useActionState` + `useTransition`으로 상태 관리

### 3. Cache Components Directive

```jsx
// app/lib/data.js — 캐시된 데이터 함수
import { cache } from 'react';

const getCachedUser = cache(async (userId) => {
  return fetch(`https://api.example.com/users/${userId}`).then((r) => r.json());
});

export async function getUser(userId) {
  // 같은 요청 내에서 중복 fetch 방지
  return getCachedUser(userId);
}

// private 캐시 (개인화된 데이터)
'use cache';
cache({ type: 'private' });

const getPrivateData = cache(async () => {
  return fetch('/api/private').then((r) => r.json());
});

// remote 캐시 (분산 캐싱)
'use cache';
cache({ type: 'remote', minAge: 60 });

const getRemoteData = cache(async () => {
  return fetch('https://cdn.example.com/data').then((r) => r.json());
});
```

**캐시 타입**:

| 타입 | 범위 | 사용처 |
|------|------|--------|
| 기본 (생략) | 서버 인스턴스 | 일반 데이터 |
| `private` | 사용자 세션 | 개인화된 데이터 |
| `remote` | 분산 캐시 | CDN/외부 API |

### 4. 관련 함수들

```jsx
// cookies() — 쿠키 읽기 (서버 컴포넌트)
import { cookies } from 'next/headers';

export default async function Page() {
  const cookieStore = await cookies();
  const theme = cookieStore.get('theme')?.value;
  return <div>Theme: {theme}</div>;
}

// after() — 응답 후 실행 (비동기 작업)
import { after } from 'next/headers';

export default async function Page() {
  after(async () => {
    // 페이지 응답 후 백그라운드 작업
    await logVisit();
    await sendNotification();
  });
  
  return <div>Page Content</div>;
}

// draftMode() — 드래프트 모드
import { draftMode } from 'next/headers';

export default async function Page() {
  const draft = await draftMode();
  if (draft.isEnabled) {
    // 드래프트 버전 렌더링
  }
  return <div>Content</div>;
}

// connection() — 연결 재사용
import { connection } from 'next/cache';

export default async function Page() {
  const conn = await connection();
  const data = await conn.fetch('https://api.example.com/data');
  return <pre>{data}</pre>;
}
```

### 5. Server Actions + 폼 유효성 검사

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

  // 저장 로직...
  return { success: true };
}
```

---

## Tool Integration

| 도구 | 사용 목적 | 예시 |
|------|-----------|------|
| `run_command` | Next.js 개발 서버 실행 | `npm run dev`, Server Actions 테스트 |
| `search_files` | 'use server' 패턴 검색 | `grep -r "'use server'" app/` |
| `read_file` | 액션 함수 분석 | actions.js 구조 확인 |
| `edit_file` | 서버 액션 추가/수정 | 새 action 함수 생성 |

---

## Anti-Patterns & Guardrails

❌ **서버 컴포넌트에서 useState/useEffect 사용** — `'use client'` 필요  
❌ **클라이언트 컴포넌트에서 직접 DB 접근** — Server Actions로 래핑  
❌ **'use server' 함수에 클라이언트 의존성 넣기** — 순수 서버 코드만 허용  

⚠️ **대용량 FormData 전송** — 크기 제한 확인, 스트리밍 고려  
⚠️ **Server Actions에서 동기 작업** — `after()`로 비동기 처리  

---

## Best Practices

1. **서버 액션 별도 파일에 분리** — actions.js 또는 lib/actions.ts
2. **zod 등 스키마 검증 필수** — 타입 안전 + 서버 사이드 검증
3. **'use server'는 함수 단위** — 전체 파일이 아닌 함수에 적용
4. **revalidateTag 활용** — 데이터 변경 후 캐시 무효화
5. **useActionState로 피드백** — 성공/실패 상태 클라이언트에 전달

---

## References

- [Directives: use client](https://nextjs.org/docs/app/api-reference/directives/use-client)
- [Directives: use server](https://nextjs.org/docs/app/api-reference/directives/use-server)
- [Directives: use cache](https://nextjs.org/docs/app/api-reference/directives/use-cache)
- [Mutating Data (Server Actions)](https://nextjs.org/docs/app/getting-started/mutating-data)
- [cookies()](https://nextjs.org/docs/app/api-reference/functions/cookies)
- [after()](https://nextjs.org/docs/app/api-reference/functions/after)
