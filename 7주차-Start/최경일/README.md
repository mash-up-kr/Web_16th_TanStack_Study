# TanStack Start 딥다이브

## TanStack Start란?

**TanStack Router 위에 올라탄 풀스택 React 프레임워크.**

Vite 기반으로 SSR, 스트리밍, 서버 함수, 번들링을 지원하며, 어느 호스팅 플랫폼에도 배포 가능하다.
2025년 초에 beta를 거쳐 RC(Release Candidate) 단계에 진입했으며, 현재는 기능이 완성되고 API가 안정적인 상태다.

> "90% of any framework usually comes down to the router."
>
> — TanStack 팀, Start Overview

라우터가 프레임워크의 핵심이라는 관점에서, Start는 TanStack Router를 중심으로 SSR과 서버 통합 기능을 얹은 구조다.

<br/>

## 0️⃣ 등장 배경 — 어떤 불편함을 해결하려 했나

### React 생태계의 SSR 문제

React 팀이 Server Components를 발표하면서 Next.js의 App Router가 "서버 컴포넌트를 기본값으로"라는 방향을 잡았다. 이 방향 자체가 나쁜 건 아니지만, 개발자들이 겪은 실제 불편함은 명확했다.

**1. 멘탈 모델이 무너졌다**

Next.js App Router가 도입되면서 "이 컴포넌트는 서버인가 클라이언트인가"를 항상 의식해야 하는 구조가 됐다. `"use client"` 를 빠뜨리면 런타임 오류가 나고, 어떤 상황에서 서버/클라이언트가 결정되는지 예측이 어렵다.

**2. 캐싱이 블랙박스였다**

Next.js는 `fetch` 호출을 자동으로 캐싱한다. 이 캐싱 레이어가 요청 메모이제이션, 데이터 캐시, 풀 라우트 캐시 등 여러 층으로 구성돼 있고, 개발자가 예상치 못한 시점에 캐시 미스/히트가 일어난다. Next.js 팀이 캐싱 기본값을 Next 13 → 14 → 15에 걸쳐 여러 번 뒤집었던 게 그 증거다.

**3. 타입 안전성의 구멍**

Next.js는 TypeScript를 지원하지만, `params`, `searchParams`, 서버 액션의 입력/출력에 대한 타입 보장이 약하다. `params.id`가 `string`인지 `string[]`인지, 쿼리스트링이 존재하는지 여부를 런타임 전까지 알기 어렵다.

**4. 배포 플랫폼 종속**

Next.js는 Vercel에서 가장 잘 동작한다. Edge Runtime, ISR, Image Optimization 같은 기능들이 Vercel에 최적화돼 있고, 다른 플랫폼으로 이전할 때 기능 일부가 지원 안 되거나 직접 구현해야 하는 경우가 생긴다.

### TanStack Start가 제시한 해법

TanStack 팀은 이 불편함들에 대해 "개발자 중심의 명시적인 방식"으로 응답했다.

- 서버 컴포넌트가 기본이 아닌, **대화형 컴포넌트(클라이언트)가 기본** — 필요할 때만 서버 렌더링을 선택
- 캐싱은 TanStack Query의 SWR 패턴으로 **명시적**으로
- 라우트 파라미터, 서치 파라미터, 서버 함수 입출력 모두 **컴파일 타임에 타입 검증**
- Vite 기반 빌드로 **플랫폼 독립적인 배포**

<br/>

## 1️⃣ 핵심 기능

### Full-Document SSR

초기 요청에 매칭되는 라우트는 서버에서 렌더링된다. `beforeLoad`와 `loader`가 서버에서 실행되고, 결과 HTML이 클라이언트로 전송된다. 클라이언트는 이 마크업을 하이드레이션해서 인터랙티브 앱으로 만든다.

```ts
// routes/posts/$id.tsx
export const Route = createFileRoute('/posts/$id')({
  loader: async ({ params }) => {
    // 서버에서 실행됨
    const post = await fetchPost(params.id)
    return { post }
  },
  component: PostPage,
})
```

### Selective SSR

라우트별로 SSR 여부를 세밀하게 제어할 수 있다. 특정 라우트는 SSR 없이 클라이언트에서만 렌더링하거나, 특정 컴포넌트만 서버에서 렌더링하는 식의 선택이 가능하다.

```ts
export const Route = createFileRoute('/dashboard')({
  // ssr: false로 설정하면 클라이언트에서만 렌더링
  ssr: false,
  component: Dashboard,
})
```

### Server Functions

서버에서만 실행되는 함수를 정의하고, 클라이언트에서 직접 호출하는 RPC 방식이다. 입력과 출력이 TypeScript로 타입 보장된다.

```ts
import { createServerFn } from '@tanstack/start'

const getUser = createServerFn({ method: 'GET' })
  .validator((data: { id: string }) => data)
  .handler(async ({ data }) => {
    // 이 코드는 서버에서만 실행됨
    return await db.user.findById(data.id)
  })

// 클라이언트에서 호출
const user = await getUser({ data: { id: '123' } })
```

### API Routes

Express나 Hono 스타일의 API 라우트를 파일 기반으로 정의한다.

```ts
// routes/api/hello.ts
import { createAPIFileRoute } from '@tanstack/start/api'

export const APIRoute = createAPIFileRoute('/api/hello')({
  GET: async ({ request }) => {
    return new Response(JSON.stringify({ message: 'Hello!' }), {
      headers: { 'Content-Type': 'application/json' },
    })
  },
})
```

### Streaming

`Suspense`와 함께 스트리밍을 지원한다. 무거운 데이터를 기다리는 동안 나머지 HTML을 먼저 보내고, 준비되는 대로 청크를 전송한다.

```tsx
function PostPage() {
  return (
    <div>
      <h1>빠르게 보여지는 부분</h1>
      <Suspense fallback={<Spinner />}>
        <HeavyComponent />
      </Suspense>
    </div>
  )
}
```

<br/>

## 2️⃣ Next.js와의 차이점

### 철학 비교

| 관점 | Next.js | TanStack Start |
|------|---------|----------------|
| 컴포넌트 기본값 | 서버 컴포넌트 기본, `"use client"` 로 선택 | 클라이언트 컴포넌트 기본, 필요시 서버 선택 |
| 렌더링 전략 | RSC 우선, 서버에서 모든 걸 처리 | SSR은 옵션, 명시적으로 선택 |
| 캐싱 | 암시적 다층 캐싱 (예측 어려움) | TanStack Query의 명시적 SWR 패턴 |
| 빌드 도구 | Turbopack / Webpack | Vite |
| 배포 | Vercel 최적화 | 플랫폼 무관 (Cloudflare, AWS, Deno, Bun 등) |
| 타입 안전성 | TypeScript 지원이지만 경계에서 갭 존재 | 컴파일 타임 엔드투엔드 타입 보장 |

### 라우팅 차이

Next.js와 TanStack Start 모두 파일 기반 라우팅을 사용한다. 그러나 세부 기능에서 차이가 크다.

**Next.js 방식**

```
app/
├── page.tsx          ← /
├── posts/
│   ├── page.tsx      ← /posts
│   └── [id]/
│       └── page.tsx  ← /posts/:id
```

`params`가 `{ id: string | string[] }` 타입으로 넓게 추론되어 런타임에서 좁혀야 한다.

**TanStack Start 방식**

```
routes/
├── index.tsx              ← /
├── posts/
│   ├── index.tsx          ← /posts
│   └── $id.tsx            ← /posts/:id
```

`$id`라는 파일명 컨벤션에서 파라미터 타입이 자동으로 `string`으로 추론된다. `useParams()` 호출 시 `{ id: string }` 타입이 그대로 따라온다.

### 서버 데이터 패칭 차이

**Next.js App Router 방식**

```tsx
// 서버 컴포넌트에서 직접 fetch
async function PostPage({ params }: { params: { id: string } }) {
  const post = await fetch(`/api/posts/${params.id}`).then(r => r.json())
  return <div>{post.title}</div>
}
```

**TanStack Start 방식**

```tsx
// loader에서 데이터를 가져오고, 컴포넌트는 순수하게 유지
export const Route = createFileRoute('/posts/$id')({
  loader: ({ params }) => fetchPost(params.id),
  component: function PostPage() {
    const { post } = Route.useLoaderData()
    return <div>{post.title}</div>
  },
})
```

컴포넌트가 비동기 함수가 될 필요 없고, 데이터와 UI의 관심사가 명확히 분리된다.

### 캐싱 전략 차이

Next.js의 암시적 캐싱은 강력하지만, 예측하기 어렵다는 비판을 꾸준히 받아왔다. TanStack Start는 TanStack Query와 통합해서 캐싱을 명시적으로 다룬다.

```ts
// TanStack Start + TanStack Query
const postsQuery = queryOptions({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  staleTime: 1000 * 60 * 5, // 5분 명시
})

export const Route = createFileRoute('/posts')({
  loader: ({ context: { queryClient } }) =>
    queryClient.ensureQueryData(postsQuery),
  component: PostList,
})
```

캐싱 만료 시간, 백그라운드 리페치, 낙관적 업데이트 등을 개발자가 직접 제어한다.

<br/>

## 3️⃣ 장단점 정리

### 장점

| 항목 | 설명 |
|------|------|
| 엔드투엔드 타입 안전성 | 라우트 파라미터, 서치 파라미터, 서버 함수 입출력이 컴파일 타임에 검증됨 |
| 예측 가능한 렌더링 | 클라이언트가 기본, SSR은 선택. `"use client"` / `"use server"` 고민 없음 |
| Vite 기반 개발 경험 | 빠른 HMR, esbuild 번들링, Vite 플러그인 생태계 그대로 활용 |
| 플랫폼 독립 배포 | Node.js, Cloudflare, AWS Lambda, Deno, Bun — 어디든 동등 지원 |
| 작은 번들 + 높은 성능 | Next.js 대비 ~30% 작은 클라이언트 번들, SSR 벤치마크 최고 처리량 |

### 단점

| 항목 | 설명 |
|------|------|
| 짧은 역사 | 2024~2025년 출시. 예제·플러그인·써드파티 통합이 Next.js보다 적음 |
| 이미지/폰트 최적화 없음 | Next.js `<Image>` 같은 내장 최적화 컴포넌트 없음 |
| RSC 미지원 | Server Functions 방식 사용, RSC 기반 패턴/생태계 활용 불가 |
| 낮은 마인드셰어 | 구인·채용 시 Next.js 경험 요구가 압도적으로 많음 |
| 레퍼런스 부족 | 에러 검색 시 스택오버플로우·블로그 자료가 절대적으로 적음 |

<br/>

## 4️⃣ 어떤 프로젝트에 적합한가

### TanStack Start가 더 맞는 경우

- **인터랙션이 많은 앱** — 대시보드, SaaS, 관리도구처럼 사용자 조작이 많은 경우
- **타입 안전성이 최우선인 경우** — 큰 팀, 복잡한 도메인, 백엔드와 프론트가 같은 레포에 있는 경우
- **멀티 플랫폼 배포** — Cloudflare Workers, AWS, Deno 등 다양한 런타임에 배포해야 할 때
- **TanStack Query를 이미 쓰고 있는 경우** — 기존 쿼리 레이어를 서버 렌더링과 자연스럽게 통합 가능

### Next.js가 더 맞는 경우

- **콘텐츠 중심 사이트** — 블로그, 마케팅 페이지, 문서 사이트처럼 정적/서버 렌더링이 핵심인 경우
- **Vercel 배포** — Vercel의 플랫폼 기능(ISR, Edge, Image CDN)을 최대로 활용하는 경우
- **팀 레퍼런스** — 팀원들이 Next.js에 익숙하고 채용 시 Next.js 경험자가 많은 경우
- **생태계 의존도가 높은 경우** — Next.js 전용 라이브러리, Auth.js, Payload CMS 등의 통합이 필요한 경우

<br/>

## 5️⃣ 빠른 시작

```bash
# 새 프로젝트 생성
npx create-tsrouter-app@latest my-app --template start-bare

# 혹은 패키지 직접 설치
pnpm add @tanstack/start @tanstack/react-router vite
```

핵심 설정 파일은 세 개다.

```
app/
├── routes/
│   ├── __root.tsx      ← 루트 레이아웃
│   └── index.tsx       ← / 라우트
├── client.tsx          ← 클라이언트 엔트리
├── router.tsx          ← 라우터 인스턴스
└── ssr.tsx             ← 서버 엔트리
```

```ts
// app/router.tsx
import { createRouter } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'

export function createAppRouter() {
  return createRouter({ routeTree })
}
```

라우트 트리(`routeTree.gen.ts`)는 Vite 플러그인이 자동 생성한다. 라우트 파일을 추가하거나 수정하면 타입도 자동으로 재생성된다.

<br/>

## 6️⃣ 실제 적용 후기

개인 블로그 사이트([gyeung-il.com](https://www.gyeung-il.com))에 TanStack Start를 도입했다. 기존 CSR 구조에 SSR을 붙이고, 그걸 기반으로 동적 OG 태그를 적용하는 게 목표였다. 기본 개념만 잡고 바이브 코딩으로 실전에 뛰어들었다.

### 1단계 — TanStack Start 도입 ([PR #58](https://github.com/inhachoi/inha-choi/pull/58))

**파일 기반 라우팅으로 전환**

기존엔 `router.tsx` 한 파일에서 `createRoute`로 직접 라우트를 조립했다. TanStack Start로 오면서 `routes/` 디렉토리 파일 기반 라우팅으로 바꿨고, `routeTree.gen.ts`는 Vite 플러그인이 자동 생성한다. 라우트 파일을 추가하면 타입이 즉시 재생성되는 게 체감이 컸다.

**`__root.tsx` `head()` API**

`index.html`에 59줄 분량으로 하드코딩돼 있던 메타태그, OG 태그, GA 스크립트를 전부 `__root.tsx`의 `head()` API로 옮겼다. 덕분에 `index.html`은 5줄짜리 껍데기만 남고, 라우트별로 `head()`를 오버라이드할 수 있는 구조가 자연스럽게 생겼다.

**그 외 마이그레이션 이슈**

- `createRoot` → `hydrateRoot` 교체 (서버 HTML을 재사용하지 않고 버리는 문제)
- `tsconfig`의 `verbatimModuleSyntax: true` 제거 — TanStack Start 내부 번들러와 호환되지 않아 빌드 실패. 문서에 없어서 에러 추적으로 찾았다.

---

### 2단계 — 동적 OG 태그 ([PR #60](https://github.com/inhachoi/inha-choi/pull/60))

Start를 붙인 진짜 이유였다. 기존엔 모든 페이지 카카오톡 공유 미리보기가 동일했다.

**`key` 필드로 OG 태그 오버라이드**

`__root.tsx`에 기본 OG 태그를 정의하고, 자식 라우트에서 같은 `key`로 덮어쓴다. `key` 없이 쓰면 부모·자식 meta가 중복 렌더링된다는 걸 직접 겪으면서 알았다.

```ts
// __root.tsx
meta: [{ key: "og:title", property: "og:title", content: "개발자 최경일" }]

// posts.tsx — 같은 key로 override
head: () => ({
  meta: [{ key: "og:title", property: "og:title", content: "블로그 | 개발자 최경일" }],
}),
```

**`loader` + `head()`로 포스트별 동적 OG**

포스팅 상세 페이지는 slug마다 다른 thumbnail·제목이 필요해서 Velog API를 `loader`에서 호출했다. `head()`가 `loaderData`를 인자로 받아서 loader 결과를 바로 meta에 쓸 수 있는 구조가 깔끔했다.

```ts
export const Route = createFileRoute("/_mainLayout/$slug")({
  loader: ({ params }) => fetchVelogPost(params.slug),
  head: ({ loaderData }) => ({
    meta: [
      { key: "og:title", property: "og:title",
        content: loaderData?.title ? `${loaderData.title} | 개발자 최경일` : "개발자 최경일" },
      { key: "og:image", property: "og:image",
        content: loaderData?.thumbnail ?? FALLBACK_IMAGE },
    ],
  }),
  component: PostModal,
});
```

**빌드 타임 pre-render**

카카오톡 크롤러는 JS를 실행하지 않는다. `loader` + `head()`로 동적 OG를 아무리 잘 만들어도, 서버 HTML에 태그가 없으면 크롤러는 빈 `<head>`만 읽는다. TanStack Start의 `tanstackStart({ pages })` 옵션으로 빌드 시 Velog API를 호출해 전체 slug를 수집하고 57개 페이지를 HTML로 사전 생성했다.

**그 외 이슈**

- 다크모드 hydration mismatch: pre-render HTML은 항상 라이트모드라, `useTheme` 초기값이 `localStorage`를 즉시 읽으면 서버·클라이언트 불일치 → 초기값을 `false`로 고정하고 마운트 후 동기화로 해결
- nginx `root` 경로를 `dist/` → `dist/client/`로 수정 (TanStack Start 빌드 output 구조 변경, 문서에 없어서 배포 후 404로 발견)

<br/>

## 🤔 느낀 점

- 적용해야하는 필요성을 못느낀 상태에서 적용하다보니 아직 잘 모르겠네요 ㅎㅎ 리액트 짱!

<br/>

## ❓ 질문

- 가장 체감되는 Next.js와의 차이점 + TanStack Start를 써야하는 이유가 궁금합니당~
