# TanStack Start란?

> TanStack Router 위에 서버 사이드 렌더링(SSR), 서버 함수, 스트리밍을 얹은 **풀스택 React 프레임워크**. 라우터의 타입 안전성을 그대로 유지하면서 서버와 클라이언트 코드를 한 파일에 공존시킬 수 있게 한다.

```bash
npm create tanstack@latest
# 또는 기존 TanStack Router 프로젝트에 추가
npm install @tanstack/start
```

---

# 왜 만들어졌는가?

TanStack Router는 클라이언트 사이드 라우팅에서 타입 안전성과 파일 기반 라우팅을 제공했지만, 서버에서 데이터를 가져오는 일은 여전히 외부 솔루션에 의존해야 했다.

> "We built the best client-side router. Now we need to make it full-stack."

## 기존 방식의 한계

**① SSR을 위해 별도 프레임워크가 필요했다**

TanStack Router 단독으로는 서버 렌더링이 불가능하다. Next.js나 Remix 같은 메타 프레임워크를 얹으면 라우팅 레이어가 이중화되거나, TanStack Router의 타입 안전성을 포기해야 했다.

**② 서버-클라이언트 경계가 불명확했다**

`useEffect`나 `getServerSideProps` 같은 방식은 서버 코드와 클라이언트 코드가 서로 다른 파일, 다른 패턴으로 쪼개진다. 타입이 서버-클라이언트 경계를 넘을 때 깨지는 문제도 있었다.

**③ 라우터 중심의 데이터 패칭이 없었다**

Next.js의 App Router나 Remix는 라우트 파일 자체가 데이터 로딩의 단위가 된다. TanStack Router만 쓰면 이 패턴을 직접 구현해야 했다.

TanStack Start는 이 세 가지를 해결하기 위해 **Vinxi** 위에 구축됐다.

---

# Vinxi - Start의 엔진

TanStack Start는 직접 번들러를 만드는 대신 **Vinxi**를 기반으로 한다.

```plain text
TanStack Start
 └── Vinxi (메타 프레임워크 툴킷)
      ├── Vite (번들러)
      └── Nitro (서버 엔진)
           ├── Node.js
           ├── Bun
           ├── Deno
           ├── Vercel
           ├── Cloudflare Workers
           └── ... (다양한 배포 타겟)
```

Vinxi가 "클라이언트 번들"과 "서버 번들"을 각각 만들어주고, Nitro가 서버 런타임을 제공한다. 덕분에 TanStack Start는 배포 타겟에 구애받지 않는다.

---

# 서버는 어디에 뜨는가? - Nitro Preset

TanStack Start는 **반드시 서버가 필요하다**. S3, GitHub Pages 같은 순수 정적 호스팅은 불가능하다. 서버 함수(`createServerFn`)와 SSR을 처리할 런타임이 있어야 하기 때문이다.

`app.config.ts`의 `preset` 값 하나로 배포 타겟이 결정된다.

```ts
// app.config.ts
export default defineConfig({
  server: {
    preset: 'node-server', // 여기를 바꾸면 빌드 결과물이 달라진다
  },
})
```

## node-server — 직접 띄우는 Node.js 서버

```ts
preset: 'node-server'
```

```plain text
빌드 결과물
 └── .output/
      ├── server/
      │    └── index.mjs   ← Node.js로 직접 실행하는 서버
      └── public/
           └── ...         ← 정적 에셋 (JS, CSS 등)
```

```bash
node .output/server/index.mjs
# PORT 환경변수로 포트 지정 가능
PORT=8080 node .output/server/index.mjs
```

EC2, GCP Compute Engine, Docker 컨테이너 등 **직접 서버를 관리하는 환경**에 적합하다. 서버가 항상 떠 있고, 요청마다 새로 콜드 스타트하지 않는다는 게 장점이다. DB 커넥션 풀을 서버 수명 동안 유지할 수 있다.

```plain text
장점: 콜드 스타트 없음, DB 커넥션 풀 유지, 설정 자유도 높음
단점: 서버 인프라를 직접 관리해야 함, 트래픽에 따른 수동 스케일링
```

## vercel — Vercel Serverless Functions

```ts
preset: 'vercel'
```

`vercel deploy`만 하면 끝난다. 별도 서버 설정 없이 Vercel이 라우팅, 스케일링, SSL을 모두 처리한다.

```plain text
요청 흐름
브라우저 → Vercel Edge Network (CDN)
  ├── 정적 에셋 (/assets/*.js) → CDN에서 바로 응답
  └── 페이지 / API 요청 → Serverless Function 실행
                           └── 콜드 스타트 후 handler 실행
                                └── 응답 후 인스턴스 소멸
```

**Serverless의 특성**: 요청이 없을 땐 서버가 존재하지 않는다. 요청이 들어오면 인스턴스가 생성(콜드 스타트)되고, 응답 후 일정 시간이 지나면 소멸한다.

```plain text
장점: 배포 간단, 자동 스케일링, 트래픽 없을 때 비용 없음
단점: 콜드 스타트 (수백ms), 실행 시간 제한 (기본 10초), DB 커넥션 풀 유지 불가
```

콜드 스타트와 DB 커넥션 문제는 Vercel의 Edge Runtime이나 PlanetScale, Neon 같은 HTTP 기반 DB를 쓰는 방식으로 우회한다.

## cloudflare-pages — Cloudflare Workers (Edge)

```ts
preset: 'cloudflare-pages'
```

코드가 전 세계 Cloudflare의 엣지 노드(300개 이상의 데이터센터)에서 실행된다. 사용자와 가장 가까운 서버에서 응답하기 때문에 레이턴시가 극도로 낮다.

```plain text
일반 서버 (서울 기준)
사용자(뉴욕) → 서울 서버 → 응답       (약 200ms RTT)

Cloudflare Workers
사용자(뉴욕) → 뉴욕 엣지 노드 → 응답  (약 20ms RTT)
```

단, Workers 환경에는 **제약이 있다**.

```plain text
사용 불가
 ├── Node.js 내장 모듈 (fs, path, crypto 등 일부)
 ├── 기존 Node.js DB 드라이버 (pg, mysql2 등)
 └── 실행 시간 제한 (CPU 시간 기준 10~50ms)

대안
 ├── Cloudflare D1 (SQLite 기반 엣지 DB)
 ├── Cloudflare KV (키-값 스토어)
 └── Hyperdrive (외부 DB를 엣지에서 연결)
```

```plain text
장점: 극저레이턴시, 글로벌 분산, 자동 스케일링, 콜드 스타트 거의 없음
단점: Node.js API 제한, 기존 DB 드라이버 미지원, 러닝 커브
```

## bun — Bun 런타임 서버

```ts
preset: 'bun'
```

Node.js 대신 Bun 런타임으로 서버를 실행한다. API는 `node-server`와 거의 같고, 실행 방식만 다르다.

```bash
bun .output/server/index.mjs
```

```plain text
장점: Node.js 대비 빠른 시작 시간, 내장 번들러/테스트러너, 낮은 메모리 사용량
단점: Node.js 대비 생태계가 작음, 일부 npm 패키지 호환성 이슈
```

## netlify — Netlify Functions

```ts
preset: 'netlify'
```

Vercel과 유사한 Serverless 방식이다. `netlify deploy`로 배포하면 Netlify가 서버 함수를 자동으로 Netlify Functions(AWS Lambda 기반)로 변환한다.

```plain text
장점: 배포 간단, 기존 Netlify 인프라와 통합, 무료 티어 넉넉함
단점: Vercel 대비 Next.js/TanStack Start 최적화 수준이 낮음, 콜드 스타트
```

## 배포 타겟 비교

| preset              | 실행 환경          | 콜드 스타트 | DB 커넥션 풀 | 글로벌 엣지 | 관리 주체    |
| :------------------ | :----------------- | :---------- | :----------- | :---------- | :----------- |
| `node-server`       | 직접 띄운 서버     | 없음        | 가능         | 직접 구성   | 개발자       |
| `bun`               | Bun 런타임 서버    | 없음        | 가능         | 직접 구성   | 개발자       |
| `vercel`            | AWS Lambda 기반    | 있음        | 불가         | Vercel CDN  | Vercel       |
| `netlify`           | AWS Lambda 기반    | 있음        | 불가         | Netlify CDN | Netlify      |
| `cloudflare-pages`  | Cloudflare Workers | 거의 없음   | 불가*        | 300+ 노드   | Cloudflare   |

\* Cloudflare Hyperdrive로 우회 가능

## preset 선택 기준

```plain text
트래픽이 예측 가능하고 DB 커넥션 풀이 필요하다
  → node-server 또는 bun

배포 간편함이 우선이고 트래픽이 간헐적이다
  → vercel 또는 netlify

전 세계 사용자 대상, 레이턴시가 가장 중요하다
  → cloudflare-pages (단, DB 전략도 함께 변경 필요)
```

---

# 핵심 개념: Server Functions

> 서버에서만 실행되는 함수를 `createServerFn`으로 선언하면, 클라이언트에서 호출할 때 자동으로 HTTP 요청으로 변환된다. 타입은 서버-클라이언트 경계를 넘어도 그대로 유지된다.

## createServerFn 기본 사용법

```ts
// app/functions/posts.ts
import { createServerFn } from '@tanstack/start'

export const fetchPost = createServerFn()
  .validator((postId: string) => postId)  // 입력 검증
  .handler(async ({ data: postId }) => {
    // 이 코드는 절대 클라이언트 번들에 포함되지 않는다
    const res = await fetch(`https://api.example.com/posts/${postId}`)
    if (!res.ok) throw new Error('Post not found')
    return res.json() as Promise<Post>
  })
```

```ts
// 클라이언트에서 호출
const post = await fetchPost({ data: '1' })
// 내부적으로 POST /api/server-fn?_serverFnId=fetchPost 요청으로 변환됨
```

## 서버 함수의 특징

**① 번들 분리 보장**

`createServerFn`의 핸들러 내부 코드는 클라이언트 번들에서 완전히 제거된다. DB 연결 문자열, 비밀 키 등이 클라이언트로 유출될 위험이 없다.

**② 타입 안전성**

```ts
// validator의 반환 타입이 handler의 data 타입이 된다
.validator((input: unknown) => {
  if (typeof input !== 'string') throw new Error()
  return input  // string
})
.handler(async ({ data }) => {
  // data: string — 타입이 보장됨
})
```

**③ 미들웨어 지원**

```ts
import { createMiddleware } from '@tanstack/start'

const authMiddleware = createMiddleware().server(async ({ next, context }) => {
  const session = await getSession()
  if (!session) throw new Error('Unauthorized')
  return next({ context: { user: session.user } })
})

export const getPrivateData = createServerFn()
  .middleware([authMiddleware])
  .handler(async ({ context }) => {
    // context.user가 보장된 상태
    return fetchDataForUser(context.user.id)
  })
```

## mutation을 위한 서버 함수

GET이 아닌 변경 작업에는 메서드를 명시한다.

```ts
export const createPost = createServerFn({ method: 'POST' })
  .validator((data: { title: string; body: string }) => data)
  .handler(async ({ data }) => {
    const post = await db.post.create({ data })
    return post
  })
```

---

# 라우팅과 데이터 로딩

TanStack Start는 TanStack Router의 파일 기반 라우팅을 그대로 사용하면서, 각 라우트가 `loader`를 통해 서버 함수를 호출하도록 연결한다.

## 파일 구조

```plain text
app/
├── routes/
│   ├── __root.tsx          ← 루트 레이아웃
│   ├── index.tsx           ← /
│   ├── posts/
│   │   ├── index.tsx       ← /posts
│   │   └── $postId.tsx     ← /posts/:postId (동적 라우트)
├── functions/
│   └── posts.ts            ← 서버 함수 모음
├── client.tsx              ← 클라이언트 진입점
└── router.tsx              ← 라우터 설정
```

## 라우트 파일에서 데이터 로딩

```tsx
// app/routes/posts/$postId.tsx
import { createFileRoute } from '@tanstack/react-router'
import { fetchPost } from '../../functions/posts'

export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    // 서버에서 실행될 때: 직접 함수 호출
    // 클라이언트에서 실행될 때: HTTP 요청으로 자동 변환
    return fetchPost({ data: params.postId })
  },
  component: PostDetail,
})

function PostDetail() {
  const post = Route.useLoaderData()  // 타입이 자동으로 추론됨

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </article>
  )
}
```

## loader vs 서버 함수 직접 호출

```plain text
loader
 ├── 라우트 진입 시 자동 실행
 ├── 병렬 데이터 패칭 최적화
 ├── SPA 탐색 시 클라이언트에서도 호출됨
 └── 라우트의 "무엇이 필요한가"를 선언적으로 정의

서버 함수 직접 호출 (useServerFn 또는 직접 import)
 ├── 버튼 클릭, 폼 제출 등 이벤트 기반 실행
 ├── mutation (POST, PUT, DELETE)
 └── loader 외부에서 필요한 동적 데이터 패칭
```

---

# SSR과 스트리밍

## SSR 동작 방식

```plain text
① 브라우저가 /posts/1 요청
② 서버가 라우트 매칭
③ loader 실행 → fetchPost 서버 함수 호출 (DB 직접 접근 가능)
④ React 컴포넌트를 HTML 문자열로 렌더링
⑤ HTML + 직렬화된 loader 데이터를 응답
⑥ 브라우저가 HTML 파싱 → 즉시 콘텐츠 표시
⑦ JS 번들 로드 → Hydration (인터랙티브 상태로 전환)
```

## 스트리밍 (Suspense 연동)

```tsx
// app/routes/posts/index.tsx
export const Route = createFileRoute('/posts/')({
  loader: async () => {
    return {
      // 이 Promise는 즉시 반환 — 스트리밍 시작
      posts: fetchAllPosts({ data: undefined }),
    }
  },
  component: Posts,
})

function Posts() {
  const { posts } = Route.useLoaderData()

  return (
    <Suspense fallback={<Spinner />}>
      {/* Await가 Promise를 기다려 스트리밍으로 HTML을 점진적으로 전송 */}
      <Await promise={posts}>
        {(data) => data.map((post) => <PostCard key={post.id} post={post} />)}
      </Await>
    </Suspense>
  )
}
```

스트리밍을 쓰면 전체 데이터가 준비되기 전에 HTML의 나머지 부분을 먼저 보낼 수 있다. 사용자는 빈 화면 대신 레이아웃을 먼저 보게 된다.

---

# 폼 처리 - useServerFn

React 폼과 서버 함수를 연결하는 패턴이다.

```tsx
import { useServerFn } from '@tanstack/start'
import { createPost } from '../functions/posts'

function CreatePostForm() {
  const submit = useServerFn(createPost)

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    const formData = new FormData(e.currentTarget)

    await submit({
      data: {
        title: formData.get('title') as string,
        body: formData.get('body') as string,
      },
    })
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="title" />
      <textarea name="body" />
      <button type="submit">작성</button>
    </form>
  )
}
```

`useServerFn`은 서버 함수를 클라이언트에서 안전하게 호출할 수 있는 래퍼다. JavaScript가 비활성화된 환경을 위해 `<form action={createPost.url}>` 방식도 지원한다.

---

# TanStack Start vs 다른 프레임워크

| 항목                | Next.js (App Router)       | Remix                  | TanStack Start            |
| :------------------ | :------------------------- | :--------------------- | :------------------------ |
| 라우팅              | 파일 기반 (자체)           | 파일 기반 (자체)       | TanStack Router 기반      |
| 타입 안전성         | 부분적                     | 부분적                 | 엔드투엔드 완전 보장      |
| 서버 코드           | Server Actions             | action / loader        | createServerFn            |
| 번들러              | Turbopack / Webpack        | Vite + Remix 컴파일러  | Vinxi (Vite + Nitro)      |
| 배포 타겟           | Vercel 최적화              | Fly.io, Remix 어댑터   | Nitro 어댑터 전체         |
| 기존 생태계 호환성  | React 기반                 | React 기반             | TanStack 생태계와 통합    |

## 타입 안전성에서의 차이

```ts
// Next.js Server Action — 리턴 타입을 직접 선언해야 함
'use server'
async function createPost(formData: FormData): Promise<Post> {
  // ...
}

// TanStack Start — validator → handler 체인으로 타입이 자동 흐름
export const createPost = createServerFn({ method: 'POST' })
  .validator((data: CreatePostInput) => data)
  .handler(async ({ data }): Promise<Post> => {
    // data: CreatePostInput이 자동으로 보장됨
  })
```

---

# 프로젝트 설정 상세

## app.config.ts

```ts
// app.config.ts
import { defineConfig } from '@tanstack/start/config'

export default defineConfig({
  server: {
    preset: 'node-server',  // 또는 'vercel', 'cloudflare-pages' 등
  },
  tsr: {
    appDirectory: './app',
  },
})
```

## 루트 라우트 설정

```tsx
// app/routes/__root.tsx
import { createRootRoute, Outlet, ScrollRestoration } from '@tanstack/react-router'
import { Meta, Scripts } from '@tanstack/start'

export const Route = createRootRoute({
  head: () => ({
    meta: [
      { charSet: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      { title: 'My App' },
    ],
  }),
  component: () => (
    <html>
      <head>
        <Meta />
      </head>
      <body>
        <Outlet />
        <ScrollRestoration />
        <Scripts />
      </body>
    </html>
  ),
})
```

`Meta`와 `Scripts`는 서버가 주입하는 CSS, JS 번들 태그를 자동으로 렌더링한다.

---

# 데이터 패칭 흐름 전체 요약

```plain text
[서버 최초 요청]
브라우저 → 서버
  └── 라우트 매칭
       └── loader 실행
            └── createServerFn handler 직접 실행 (네트워크 없음)
                 └── DB / 외부 API 접근
                      └── HTML 스트리밍 응답
                           └── 브라우저 렌더링 → Hydration

[클라이언트 SPA 탐색]
링크 클릭 → TanStack Router
  └── loader 실행
       └── createServerFn → HTTP POST 요청
            └── 서버가 handler 실행 → JSON 응답
                 └── 컴포넌트 리렌더링

[이벤트 기반 mutation]
버튼 클릭 → useServerFn(createPost)
  └── HTTP POST 요청
       └── 서버 handler 실행
            └── DB 업데이트 → 응답
                 └── 클라이언트 상태 갱신 (router.invalidate() 등)
```

---

# 정리

TanStack Start가 추가하는 것은 단순하다.

**"TanStack Router가 알고 있는 라우트 트리 위에, 서버에서 실행되는 함수를 타입 안전하게 붙인다"**

Next.js App Router가 React Server Components라는 새로운 멘탈 모델을 요구하는 것과 달리, TanStack Start는 기존 TanStack Router의 `loader` 패턴을 그대로 유지한다. 서버 함수는 그냥 함수다 — 호출하면 서버에서 실행될 뿐이다.

```plain text
TanStack Router (클라이언트 라우팅 + 타입 안전)
  + createServerFn (서버-클라이언트 경계 타입 안전)
  + Vinxi / Nitro (SSR + 번들링 + 다양한 배포 타겟)
= TanStack Start
```

아직 베타 단계이며, React 19와 함께 성장하고 있다. 기존 TanStack Router 프로젝트를 SSR로 전환하거나, 타입 안전한 풀스택 앱을 새로 시작할 때 선택지로 고려할 만하다.
