---
name: nextjs-file-conventions
description: Next.js App Router의 파일 시스템 컨벤션을 다룹니다. layout, page, loading, error, not-found, route, intercepting routes, parallel routes, metadata files 등 모든 파일 컨벤션의 역할과 사용법을 학습합니다.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: nextjs
  tags: [nextjs, file-conventions, route-groups, parallel-routes, intercepting-routes, metadata]
---

# Next.js File Conventions

## Overview

Next.js App Router는 파일 시스템 기반 라우팅을 사용합니다. 특정 파일명과 폴더명으로 Next.js가 자동으로 동작을 결정합니다. 이 스킬은 모든 파일 컨벤션의 역할과 조합 방법을 다룹니다.

**핵심 원칙**: 파일명 = 동작. 표준 파일명을 지키면 Next.js가 자동 처리.

---

## SOP: Step-by-Step Procedures

### 1. 핵심 파일 컨벤션

```
app/
├── layout.js      # 레이아웃 (중첩 가능, children prop 필수)
├── page.js        # 페이지 (라우트의 진입점, export default 필수)
├── loading.js     # 로딩 UI (Suspense 자동 적용, 레이아웃과 함께 사용)
├── error.js       # 에러 경계 (UI, reset 함수 제공)
├── not-found.js   # 404 페이지 (전역 또는 라우트별)
├── route.js       # API 엔드포인트 (GET/POST/PUT/DELETE 등)
├── template.js    # 레이아웃과 유사하지만 state 유지 안 함 (transition 시 재렌더링)
└── globals.css    # 전역 CSS
```

**각 파일의 특징**:

| 파일 | 필수 export | 특징 |
|------|-------------|------|
| `layout.js` | `default` | children prop, 중첩 가능 |
| `page.js` | `default` | 라우트 진입점, async 지원 |
| `loading.js` | `default` | Suspense boundary 자동 생성 |
| `error.js` | `default(error, reset)` | UI 에러 표시, reset으로 복구 |
| `not-found.js` | `default` | 404 상태 코드 자동 설정 |
| `route.js` | `GET/POST/PUT/DELETE` | API 엔드포인트 |

### 2. Route Groups (URL 영향 없는 그룹화)

```
app/
├── (auth)/          # URL에 나타남: /login, /register
│   ├── login/page.js
│   └── register/page.js
├── (dashboard)/     # URL에 나타남: /dashboard/admin
│   ├── admin/layout.js
│   └── admin/page.js
└── dashboard/       # 일반 폴더
    └── page.js      # /dashboard
```

**Step-by-Step**:
1. `(group-name)` 폴더명으로 그룹화
2. URL 경로에 그룹명 반영 안 됨
3. 레이아웃은 그룹 내에서 공유
4. `/admin`과 `/dashboard/admin` 구조화에 유용

### 3. Parallel Routes (동시 렌더링)

```
app/
├── layout.js        # 공통 레이아웃
├── page.js          # /
├── @sidebar/page.js   # slot: sidebar
├── @main/page.js      # slot: main
└── @alternate/
    ├── page.js        # 기본 대체 콘텐츠
    └── feed/page.js   # /feed 대체
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
1. `@slot-name` 폴더명으로 슬롯 정의
2. 레이아웃에서 slot prop으로 접근
3. `null`이면 children 렌더링
4. A/B 테스트, 동적 콘텐츠 교체에 유용

### 4. Intercepting Routes (경로 인터셉트)

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
│           └── page.js  # 인터셉트: /posts/hello-world에서 오버레이로 표시
```

**Step-by-Step**:
1. `(group)`으로 인터셉트 대상 라우트 그룹화
2. `@slot/(group)/` 구조로 인터셉트 레이아웃 생성
3. 같은 URL에서 오버레이/모달로 렌더링
4. 네비게이션 시 페이지 전환 없이 오버레이 표시

### 5. Metadata Files (메타데이터)

```
app/
├── favicon.ico          # 파비콘
├── icon.png             # 아이콘 (OG 등)
├── opengraph-image.png  # OG 이미지
├── twitter-image.png    # Twitter 카드 이미지
├── manifest.json        # PWA 매니페스트
├── robots.txt           # SEO 로봇 지시
└── sitemap.xml          # 사이트맵

// 프로그램matic 메타데이터
export function generateMetadata({ params }) {
  return {
    title: 'Post Title',
    description: 'Post Description',
    openGraph: { title: 'OG Title', images: ['/og.png'] }
  };
}
```

**Step-by-Step**:
1. 정적 메타데이터 파일은 `app/` 루트에 배치
2. 동적 메타데이터는 `generateMetadata` 함수 사용
3. `opengraph-image.js`로 프로그램matic OG 이미지 생성 가능

### 6. Dynamic Routes

```jsx
// app/blog/[slug]/page.js — 단일 동적 세그먼트
export default function Page({ params }) {
  return <h1>{params.slug}</h1>;
}

// app/dashboard/[[...catchAll]]/page.js — 옵셔널 catch-all
export default function Page({ params }) {
  return <p>{params.catchAll?.join('/') || 'Home'}</p>;
}

// 정적 생성: generateStaticParams
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then((r) => r.json());
  return posts.map((post) => ({ slug: post.slug }));
}
```

### 7. Route Segment Config (세그먼트 설정)

```jsx
// app/dashboard/page.js — 세그먼트별 설정
export const dynamic = 'auto';      // auto / force-dynamic / force-static
export const dynamicParams = true;  // 동적 세그먼트 처리 여부
export const revalidate = 3600;     // ISR 리발리데이션 주기 (초)
export const maxDuration = 60;      # 함수 실행 최대 시간 (초)

// preferredRegion: 데이터 센터 영역 설정
export const preferredRegion = 'auto'; // auto / iad1 / sin1 등
```

---

## Tool Integration

| 도구 | 사용 목적 | 예시 |
|------|-----------|------|
| `run_command` | Next.js 개발 서버 실행 | `npm run dev`, 빌드 테스트 |
| `search_files` | 파일 컨벤션 패턴 검색 | `grep -r "generateMetadata" app/` |
| `read_file` | 컨벤션 파일 분석 | layout.js, page.js 구조 확인 |
| `edit_file` | 새 라우트/파일 추가 | page.js, route.js 생성 |

---

## Anti-Patterns & Guardrails

❌ **page.js에 레이아웃 코드 넣기** — layout은 구조, page는 콘텐츠 분리  
❌ **Route Groups와 일반 폴더 혼동** — `(group)`은 URL 영향 없음, 일반 폴더는 영향 있음  
❌ **parallel routes에서 slot 누락** — 모든 슬롯 정의 필요, null 처리 필수  

⚠️ **intercepting routes 복잡도** — 디버깅 어려움, 필요한 경우에만 사용  
⚠️ **catch-all 라우트 오남용** — 특정 패턴에만 한정  

---

## Best Practices

1. **파일명 표준 준수** — layout/page/loading/error/not-found 컨벤션 활용
2. **Route Groups로 구조화** — `/admin`, `/dashboard` 등 논리적 그룹핑
3. **metadata.js 또는 generateMetadata 사용** — SEO 최적화
4. **generateStaticParams 활용** — 정적 생성으로 성능 향상
5. **error.js 필수 포함** — 모든 라우트 그룹에 에러 경계 설정

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
