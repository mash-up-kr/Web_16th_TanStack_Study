# TanStack Start

Next.js로 풀스택 앱을 만들다 보면 마주치는 익숙한 풍경이 있다.
어떤 페이지는 SSR이 필요하고, 어떤 페이지는 클라이언트 렌더링이 더 자연스럽다.
서버에서만 돌아야 하는 코드는 API Route로 분리하고, 클라이언트에서 호출할 때마다 fetch 요청을 손수 짠다.
타입은 클라이언트와 서버 사이에서 끊어진다.

TanStack Start는 이 흐름을 다른 방식으로 풀어낸다.
**TanStack Router**의 타입 안전성을 그대로 가져오면서, 위에 SSR과 서버 함수 레이어를 얹은 풀스택 React 프레임워크다.

> 이 글은 [TanStack Start 공식 문서](https://tanstack.com/start/latest)를 직접 참조했다.
> 현재 Start는 **Release Candidate (v0)** 단계다.

---

## 1. 이게 무엇인지

---

### 1-1. TanStack Start란?

TanStack Start는 **TanStack Router** 위에 풀스택 기능을 얹은 React 프레임워크다.

> 공식 정의 ([공식 문서](https://tanstack.com/start/latest/docs/framework/react/overview)):
> *"TanStack Start is a full-stack React framework powered by TanStack Router. It provides a full-document SSR, streaming, server functions, bundling, and more."*

핵심은 다음 한 줄이다.

> *"90% of any framework usually comes down to the router, and TanStack Start is no different. TanStack Start relies 100% on TanStack Router for its routing system."*

즉, Start는 별도의 라우터를 만들지 않는다.
**TanStack Router를 그대로 사용**하면서, SSR 파이프라인과 서버 함수 RPC만 추가로 붙인다.

---

### 1-2. 빌드/실행 스택

TanStack Start의 내부 구성은 다음과 같다.

| 레이어 | 역할 |
|--------|------|
| **TanStack Router** | 파일 기반 라우팅, 타입 안전 내비게이션, 데이터 로딩 |
| **Vite** | 빌드 도구, HMR, 클라이언트/서버 번들 분리 |
| **Nitro** | SSR 서버 엔진, 다양한 호스팅 어댑터 제공 |
| **seroval** | dehydrated router state 직렬화 |

Nitro 덕분에 **Cloudflare Workers, Netlify, Vercel, AWS, Node, Bun** 등 어디든 배포할 수 있다.
Vercel에 종속되는 Next.js와 차별화되는 지점이다.

---

### 1-3. Start가 제공하는 것 vs Router가 제공하는 것

Start와 Router의 책임을 명확히 구분하는 게 중요하다.

| 기능 | TanStack Router | TanStack Start |
|------|-----------------|----------------|
| 파일 기반 라우팅 | ✅ | (Router의 기능 사용) |
| 타입 안전 내비게이션 | ✅ | (Router의 기능 사용) |
| `loader` / `beforeLoad` | ✅ | (Router의 기능 사용) |
| Search params 검증 | ✅ | (Router의 기능 사용) |
| **Full-document SSR** | ❌ (수동 구현 가능) | ✅ |
| **Streaming SSR** | ❌ | ✅ |
| **Server Functions (RPC)** | ❌ | ✅ |
| **Selective SSR** | ❌ | ✅ |
| **Vite 플러그인 / 번들 분리** | ❌ | ✅ |

Router만으로도 SPA를 만들 수 있다.
Start는 거기에 **서버 사이드 기능**을 더한 것이다.

---

## 2. 어떻게 사용하는지

---

### 2-1. 파일 기반 라우팅 + Loader

TanStack Router의 라우팅 방식을 그대로 사용한다.
각 라우트는 `createFileRoute`로 정의하며, `loader`에서 데이터를 미리 패칭할 수 있다.

```ts
// src/routes/posts/$postId.tsx
import { createFileRoute } from '@tanstack/react-router'
import { getPost } from '../serverFns/post'

export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => getPost({ data: { id: params.postId } }),
  component: PostDetail,
})

function PostDetail() {
  const post = Route.useLoaderData()
  return <article>{post.title}</article>
}
```

- `$postId` 같은 동적 파라미터는 **타입이 자동 추론**된다.
- `loader`는 SSR 시점엔 서버에서, 클라이언트 내비게이션 시엔 브라우저에서 실행된다.

---

### 2-2. Server Functions

`createServerFn`으로 정의한 함수는 **서버에서만 실행**되지만, 클라이언트에서 일반 함수처럼 호출할 수 있다.
빌드 시 자동으로 클라이언트 번들에서 제거되고, HTTP RPC 엔드포인트로 변환된다.

```ts
// src/serverFns/post.ts
import { createServerFn } from '@tanstack/react-start'
import { notFound } from '@tanstack/react-router'

export const getPost = createServerFn()
  .inputValidator((data: { id: string }) => data)
  .handler(async ({ data }) => {
    const post = await db.findPost(data.id)
    if (!post) throw notFound()
    return post
  })
```

이 함수는 다음 두 가지 위치에서 동일하게 호출 가능하다.

```ts
// 1. 라우트 loader에서
export const Route = createFileRoute('/posts')({
  loader: () => getPost({ data: { id: '1' } }),
})

// 2. 컴포넌트에서 TanStack Query와 함께
function PostList() {
  const getPosts = useServerFn(getServerPosts)
  const { data } = useQuery({
    queryKey: ['posts'],
    queryFn: () => getPosts(),
  })
}
```

#### 내부 동작 (공식 문서 기준)

> *"Server functions are addressed by a generated, stable function ID under the hood. These IDs are embedded into the client/SSR builds and used by the server to locate and import the correct module at runtime."*
> — [Server Functions 가이드](https://tanstack.com/start/latest/docs/framework/react/guide/server-functions)

- 빌드 시 각 서버 함수에 **SHA256 기반 ID**가 부여됨
- 클라이언트 번들엔 ID만 남고, 실제 구현은 서버 모듈에서 import됨
- 동일 ID 충돌 시 `_1`, `_2` 같은 suffix로 자동 dedup

이 메커니즘 덕분에 tRPC나 별도 REST API 없이 **타입 안전한 클라이언트-서버 통신**이 가능하다.

---

### 2-3. Selective SSR

라우트 단위로 **렌더링 전략을 개별 지정**할 수 있다. `ssr` 옵션은 세 가지 값을 가진다.

| 값 | beforeLoad / loader 실행 위치 | 컴포넌트 렌더링 위치 |
|---|---|---|
| `true` (기본값) | 서버 (초기 요청 시) | 서버 + 클라이언트 hydration |
| `'data-only'` | 서버 (초기 요청 시) | **클라이언트만** |
| `false` | **클라이언트만** | **클라이언트만** |

```ts
// 브라우저 전용 API를 쓰는 컴포넌트는 'data-only'로
export const Route = createFileRoute('/dashboard')({
  ssr: 'data-only',
  loader: () => fetchDashboardData(), // 서버에서 실행됨
  component: Dashboard, // 클라이언트에서만 렌더링
})
```

런타임에 동적으로 결정할 수도 있다.

```ts
export const Route = createFileRoute('/docs/$docType/$docId')({
  validateSearch: z.object({ details: z.boolean().optional() }),
  ssr: ({ params, search }) => {
    if (params.value.docType === 'sheet') return false
    if (search.value.details) return 'data-only'
    return true
  },
  component: DocPage,
})
```

#### 부모-자식 상속 규칙

> *"At runtime, a child route inherits the Selective SSR configuration of its parent. However, the inherited value can only be changed to be more restrictive."*
> — [Selective SSR 가이드](https://tanstack.com/start/latest/docs/framework/react/guide/selective-ssr)

자식 라우트는 부모보다 **더 제한적인 방향으로만** 변경 가능하다.
(예: 부모가 `'data-only'`이면, 자식은 `true`로 올릴 수 없고 `false`로만 내릴 수 있음)

---

### 2-4. 점진적 도입

이미 TanStack Router나 TanStack Query를 쓰고 있다면, Start의 SSR이나 서버 함수 기능만 골라서 추가할 수 있다.
Start는 Router를 **대체하는 것이 아니라 확장**하는 구조이기 때문이다.

> *"TanStack Start supports incremental integration: existing TanStack Router or TanStack Query applications can adopt Start's server-function and SSR features gradually."*
> — [InfoQ, 2025-11](https://www.infoq.com/news/2025/11/tanstack-start-v1/)

Next.js에서 마이그레이션할 경우, 공식 [Next.js 마이그레이션 가이드](https://tanstack.com/start/latest)도 제공된다.

---

## 3. 왜 써야하는지

---

### 3-1. End-to-End 타입 안전성

Start의 가장 큰 차별점은 **라우팅부터 서버 함수까지 전 구간 타입 추론**이다.

| 구간 | TanStack Start | Next.js |
|------|----------------|---------|
| 라우트 파라미터 | 자동 추론 (Router) | App Router에서 수동 타입 정의 필요 |
| Search params | `validateSearch`로 스키마 정의 → 자동 추론 | `useSearchParams()` 반환값은 `string \| null` |
| 서버 → 클라이언트 데이터 | `loader`/`createServerFn` 반환 타입 자동 추론 | Server Components는 추론되지만, Server Actions는 별도 타입 정의 |
| 라우트 간 내비게이션 | `Link`의 `to` prop이 타입 체크됨 | `href`는 `string` 타입 |

큰 코드베이스에서 **링크 깨짐이나 파라미터 오타**를 컴파일 타임에 잡을 수 있다는 게 실무 가치다.

---

### 3-2. tRPC가 필요 없어진다

`createServerFn`이 사실상 **빌트인 RPC**다.

| 도구 | 역할 | TanStack Start에서 |
|------|------|---------------------|
| tRPC | 타입 안전 클라이언트-서버 통신 | `createServerFn`이 대체 |
| REST API Route | 서버 전용 로직 | `createServerFn`이 대체 |
| Zod validation | 입력 검증 | `inputValidator`로 통합 |

세 개의 의존성을 하나로 줄일 수 있다. 의존성, 학습 비용, 빌드 설정 모두 단순해진다.

---

### 3-3. 라우트 단위 렌더링 전략

Next.js에서도 `dynamic`, `force-static`, `force-dynamic` 같은 옵션이 있지만,
**라우트 단위로 SSR / data-only / CSR을 명확히 분리**하는 인터페이스는 없다.

Selective SSR은 다음과 같은 시나리오를 명시적으로 다룬다.

- 대시보드처럼 **브라우저 전용 API에 의존**하는 페이지 → `'data-only'` (loader는 서버에서, 렌더링은 클라이언트에서)
- 마케팅 페이지처럼 **SEO가 중요**한 페이지 → `true`
- 어드민처럼 **로그인된 사용자만 보는** 페이지 → `false` (순수 CSR)

런타임에 search params나 path params로 결정할 수도 있어서,
"같은 라우트지만 조건에 따라 다르게 렌더링"이 가능하다.

---

### 3-4. 배포 자유도

Nitro 어댑터를 사용해 **호스팅 종속성이 거의 없다**.

| 호스팅 | TanStack Start | Next.js |
|--------|----------------|---------|
| Vercel | ✅ | ✅ (최적화됨) |
| Cloudflare Workers | ✅ | 부분 지원 (Edge Runtime 제약) |
| Netlify | ✅ | ✅ |
| AWS / Railway / Fly.io | ✅ | ⚠️ (수동 설정 필요한 경우 많음) |
| 자체 Node 서버 | ✅ | ✅ |

자체 인프라에 배포해야 하는 환경에서 의미가 있다.

---

### 3-5. 한계와 주의점

사실 기반으로 짚어야 할 단점도 있다.

- **RC 단계 (v0):** stable 릴리스 전이라 API가 바뀔 가능성이 있다.
- **React Server Components 미지원:** 실험적으로만 제공되며, 정식 지원은 v1.x에서 non-breaking으로 추가될 예정.
- **Node.js >=22.12.0 필요:** 구버전 Node 환경에선 사용 불가.
- **Vite >=8.0.0 권장:** 최신 Vite 환경 기준으로 동작.
- **생태계 크기:** Next.js 대비 튜토리얼, 스타터 템플릿, 서드파티 예제가 적다.

---

## 정리

| 항목 | 내용 |
|---|---|
| **무엇** | TanStack Router 위에 SSR + Server Functions + Vite 빌드를 얹은 풀스택 React 프레임워크 |
| **언제 쓸지** | 타입 안전성이 최우선이고, Vercel 외 환경에 배포해야 하며, RSC 없이 충분한 클라이언트 중심 앱일 때 |
| **언제 안 쓸지** | RSC가 필수인 서버 퍼스트 아키텍처, 풍부한 레퍼런스가 필요한 팀, 빠른 온보딩이 중요한 프로젝트 |

---

## 참고 자료

- [TanStack Start 공식 사이트](https://tanstack.com/start/latest)
- [TanStack Start Overview (공식 문서)](https://tanstack.com/start/latest/docs/framework/react/overview)
- [Server Functions 가이드](https://tanstack.com/start/latest/docs/framework/react/guide/server-functions)
- [Selective SSR 가이드](https://tanstack.com/start/latest/docs/framework/react/guide/selective-ssr)
- [TanStack Start v1 RC 소개 (InfoQ, 2025-11)](https://www.infoq.com/news/2025/11/tanstack-start-v1/)
- [TanStack Start and Router: What You Need to Know (Certificates.dev)](https://certificates.dev/blog/tanstack-start-and-router-what-you-need-to-know)
- [DeepWiki — TanStack Start](https://deepwiki.com/TanStack/router/5-tanstack-start)