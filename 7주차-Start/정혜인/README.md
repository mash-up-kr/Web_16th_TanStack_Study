# TanStack Start란 무엇일까?

---

> **"TanStack Router는 라우팅 라이브러리다. TanStack Start는 그 위에 서버를 올린 풀스택 프레임워크다."**
> 즉, TanStack Start는 TanStack Router의 타입 안전성과 파일 기반 라우팅을 그대로 가져오면서, SSR·스트리밍·서버 함수까지 제공하는 풀스택 React 프레임워크입니다.

TanStack Start는 단순히 "Next.js 대안"이 아니라, **TanStack 생태계 전체를 풀스택으로 확장하는 레이어**입니다.

### 핵심 정의: TanStack Router 기반 풀스택 프레임워크

Next.js가 React 위에 서버를 올렸다면, TanStack Start는 **TanStack Router 위에 서버를 올렸습니다.**

- **설계 철학:** "타입 안전성을 클라이언트에서 서버까지 end-to-end로 보장한다"
- **기반 기술:** Vite + Nitro (서버 엔진) + TanStack Router
- **핵심 차별점:** 라우트 정의 자체가 타입이 되어, 서버 함수 호출까지 타입 추론이 이어짐

### 아키텍처 구조

```
[브라우저] ↔ [TanStack Start (SSR/Streaming)] ↔ [Server Functions] ↔ [DB/외부 API]
               │
               └─ TanStack Router (파일 기반 라우팅, 타입 안전)
               └─ Vite (번들러, HMR)
               └─ Nitro (서버 엔진, 배포 어댑터)
```

컴포넌트가 서버 함수를 직접 호출하는 구조입니다. 별도 API 라우트 없이 함수 호출만으로 서버 로직이 실행됩니다.

---

# TanStack Start는 어떻게 사용하는 걸까?

### 1. 프로젝트 구조

```
app/
  routes/
    __root.tsx          ← 루트 레이아웃 (공통 shell)
    index.tsx           ← / 경로
    posts/
      index.tsx         ← /posts 경로
      $postId.tsx       ← /posts/:postId (동적 라우트)
  routeTree.gen.ts      ← 자동 생성되는 타입 트리 (건드리지 않음)
  router.tsx            ← 라우터 인스턴스 생성
  client.tsx            ← 클라이언트 진입점
  ssr.tsx               ← 서버 진입점
```

> **`routeTree.gen.ts`가 핵심**
> Vite 플러그인이 `routes/` 폴더를 스캔해서 자동으로 생성합니다.
> 이 파일이 존재하기 때문에 `<Link to="/posts/$postId">` 같은 경로 문자열에 타입 자동완성과 오타 에러가 적용됩니다.

### 2. 라우트 정의

```tsx
// app/routes/posts/$postId.tsx
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/posts/$postId")({
  // 서버/클라이언트 모두에서 실행되는 데이터 로더
  loader: async ({ params }) => {
    return fetchPost(params.postId); // params.postId 타입: string (자동 추론)
  },
  component: PostComponent,
});

function PostComponent() {
  // loader 반환값이 그대로 타입 추론됨
  const post = Route.useLoaderData();
  return <div>{post.title}</div>;
}
```

### 3. Server Functions (핵심 기능)

TanStack Start의 가장 큰 차별점입니다. API 라우트 없이 **함수 하나로 서버 로직을 실행**합니다.

```tsx
import { createServerFn } from "@tanstack/react-start";

// 서버에서만 실행됨을 보장
const getPost = createServerFn({ method: "GET" })
  .validator((data: { postId: string }) => data) // 입력값 검증
  .handler(async ({ data }) => {
    // 이 코드는 서버에서만 실행됨
    // DB 직접 접근, 환경변수 접근 가능
    const post = await db.post.findUnique({ where: { id: data.postId } });
    return post;
  });

// 클라이언트에서 일반 함수처럼 호출
function PostPage() {
  const [post, setPost] = useState(null);

  useEffect(() => {
    getPost({ data: { postId: "1" } }).then(setPost);
    // ↑ 실제로는 HTTP 요청이 나가지만, 타입은 완전히 안전
  }, []);
}
```

### 4. TanStack Query와 통합

Server Functions와 TanStack Query를 함께 쓰는 것이 권장 패턴입니다.

```tsx
const getPost = createServerFn({ method: "GET" })
  .validator((d: { postId: string }) => d)
  .handler(async ({ data }) => fetchPost(data.postId));

function PostComponent() {
  const { data } = useQuery({
    queryKey: ["post", postId],
    queryFn: () => getPost({ data: { postId } }),
    // Server Function이 queryFn 안으로 들어감
  });
}
```

> **왜 이 조합이 자연스러운가?**
> Server Function은 "어떻게 데이터를 가져올지"를 정의하고,
> TanStack Query는 "언제, 얼마나 자주, 어떻게 캐싱할지"를 담당합니다.
> 앞서 배운 axios + Query 조합과 동일한 역할 분리입니다.

### +) 추가 팁

- **`loader` vs `Server Function`:** `loader`는 라우트 진입 시 자동 실행(waterfall 방지), Server Function은 명시적 호출. 초기 데이터는 loader, 이벤트성 서버 호출은 Server Function으로 분리하는 것이 권장됩니다.
- **`validateSearch`:** URL 쿼리 파라미터도 타입 안전하게 정의할 수 있습니다. `?page=1&filter=active` 같은 파라미터가 컴파일 타임에 검증됩니다.

---

# TanStack Start는 왜 쓰는 걸까?

### 1. 기존 솔루션의 문제 해결

| **기존 방식의 문제**                       | **TanStack Start 솔루션**                    |
| ------------------------------------------ | -------------------------------------------- |
| Next.js App Router의 복잡한 캐싱 모델      | 명시적 Server Function + TanStack Query 조합 |
| React Router의 타입 불안전한 경로/파라미터 | `routeTree.gen.ts` 기반 완전한 타입 추론     |
| API 라우트와 컴포넌트 로직의 분리          | Server Function으로 같은 파일에서 관리       |
| 서버/클라이언트 경계 불명확                | `createServerFn`으로 명시적 경계 선언        |

### 2. End-to-End 타입 안전성

TanStack Start가 Next.js와 근본적으로 다른 부분입니다.

```tsx
// Next.js — 타입이 끊기는 지점들
// app/api/posts/[id]/route.ts
export async function GET(req: Request) {
  const id = req.nextUrl.pathname.split("/")[3]; // string, 타입 없음
  const post = await getPost(id);
  return Response.json(post);
}

// app/posts/[id]/page.tsx
const res = await fetch(`/api/posts/${id}`); // 반환 타입? any
const post = await res.json(); // 타입 추론 불가

// TanStack Start — 타입이 끊기지 않음
const getPost = createServerFn()
  .validator((d: { id: string }) => d)
  .handler(async ({ data }) => {
    return await db.post.findUnique({ where: { id: data.id } });
    // 반환 타입: Post | null (자동 추론)
  });

const post = await getPost({ data: { id } });
// post 타입: Post | null — 서버 코드의 반환 타입이 그대로 흘러옴
```

### 3. 명시적인 서버/클라이언트 경계

Next.js의 `'use client'` / `'use server'` 지시어는 파일 단위로 경계를 만들지만, 실수하면 조용히 잘못 동작합니다.

```tsx
// TanStack Start — 함수 단위로 명시적
const dangerousServerFn = createServerFn().handler(async () => {
  const secret = process.env.SECRET_KEY; // 서버에서만 실행 보장
  return doSomethingWithSecret(secret);
});
```

`createServerFn`으로 감싸지 않은 함수는 절대 서버에서 실행되지 않습니다. 경계가 선언적이고 명확합니다.

### 4. Vite 기반 개발 경험

Next.js(Webpack/Turbopack)와 달리 Vite를 기반으로 하므로 HMR이 빠르고, Vite 생태계 플러그인을 그대로 활용할 수 있습니다.

---

# 동작을 파헤쳐보자!

## 라우트 정의부터 화면이 그려지기까지 — 전체 내부 흐름

```
전체 흐름 요약도

파일 기반 라우트 스캔 (Vite 플러그인)
  │
  ▼
[1] routeTree.gen.ts 자동 생성       ← 타입 트리 구성
  │
  ▼
[2] createRouter()                   ← 라우터 인스턴스 생성
  │   └─ 라우트 트리 등록
  │   └─ defaultPreload, context 설정
  │
  ▼
[3] 요청 진입 (SSR)
  │   └─ ssr.tsx → StartServer 실행
  │   └─ 매칭되는 라우트 탐색
  │
  ▼
[4] loader 실행                      ← 서버에서 데이터 패칭
  │   └─ 매칭된 라우트들의 loader 병렬 실행
  │   └─ dehydrate → 클라이언트로 직렬화
  │
  ▼
[5] SSR 렌더링
  │   └─ React renderToString / renderToPipeableStream
  │   └─ 초기 HTML + 인라인 스크립트(dehydrated state) 전달
  │
  ▼
[6] 클라이언트 Hydration
  │   └─ client.tsx → StartClient 실행
  │   └─ router.hydrate() → dehydrated state 복원
  │   └─ React hydration
  │
  ▼
[7] 클라이언트 내비게이션 (이후)
      └─ Link 클릭 → 라우트 매칭
      └─ loader 재실행 (필요시)
      └─ 컴포넌트 업데이트
```

### Step 1 — Vite 플러그인이 routeTree를 만드는 방법

```tsx
// vite.config.ts
import { TanStackRouterVite } from "@tanstack/router-plugin/vite";

export default defineConfig({
  plugins: [
    TanStackRouterVite(), // 이 플러그인이 핵심
  ],
});
```

플러그인이 하는 일:

1. `app/routes/` 폴더를 감시 (watch)
2. 파일 추가/변경/삭제 감지
3. `routeTree.gen.ts` 자동 재생성

```ts
// routeTree.gen.ts (자동 생성 — 절대 직접 수정 금지)
export const routeTree = rootRoute.addChildren({
  IndexRoute,
  PostsRoute: PostsRoute.addChildren({
    PostsIndexRoute,
    PostsPostIdRoute, // $postId 파일 → 파라미터 타입 자동 생성
  }),
});
```

이 트리 구조 덕분에 `<Link to="/posts/$postId">` 에서 존재하지 않는 경로를 쓰면 **컴파일 에러**가 납니다.

💡 **TanStack Router와의 차이**

TanStack Router는 라우팅 라이브러리 — 클라이언트 사이드만.
TanStack Start는 TanStack Router를 포함한 풀스택 프레임워크 — 서버까지.

```
TanStack Router: routeTree 정의 → 클라이언트 라우팅
TanStack Start:  routeTree 정의 → 클라이언트 라우팅 + SSR + Server Functions
```

### Step 2 — Server Function 내부 동작

`createServerFn`은 빌드 타임과 런타임, 두 단계에서 동작합니다.

**빌드 타임 (Vite 플러그인 변환)**

```tsx
// 개발자가 작성한 코드
const getPost = createServerFn().handler(async ({ data }) => {
  return await db.post.findById(data.id); // DB 접근 코드
});

// 빌드 후 클라이언트 번들에서는 이렇게 변환됨
const getPost = createServerFn().handler(
  createClientRpc("_serverFn_getPost_abc123"),
);
// DB 코드가 완전히 제거되고 RPC 호출로 교체됨
```

서버 코드가 **클라이언트 번들에 포함되지 않습니다.** `db`, `process.env.SECRET` 같은 민감한 코드가 브라우저로 유출될 위험이 없습니다.

**런타임 (클라이언트에서 호출 시)**

```
getPost({ data: { id: '1' } }) 호출
  │
  ▼
POST /_serverFn/getPost_abc123
  { data: { id: '1' } }  ← 직렬화
  │
  ▼
Nitro 서버가 요청 수신
  │
  ▼
실제 handler 실행 (DB 접근)
  │
  ▼
반환값 직렬화 → 클라이언트 응답
  │
  ▼
getPost의 반환값으로 타입 안전하게 수신
```

💡 **왜 fetch가 아닌 함수 호출처럼 느껴지냐면**
실제로는 HTTP 요청이 나가지만, Vite 플러그인이 클라이언트 코드를 변환해서 마치 일반 async 함수처럼 보이게 만들기 때문입니다. 타입도 서버 handler의 반환 타입을 그대로 추론합니다.

### Step 3 — loader와 SSR 데이터 흐름

```tsx
export const Route = createFileRoute("/posts/$postId")({
  loader: async ({ params, context }) => {
    // 이 함수는 서버(최초 요청)와 클라이언트(내비게이션) 모두에서 실행됨
    return await getPost({ data: { postId: params.postId } });
  },
});
```

**서버 최초 요청 시:**

```
요청 진입
  → 매칭되는 모든 라우트의 loader 병렬 실행
  → 결과를 dehydrate (직렬화)
  → HTML에 인라인 스크립트로 삽입
  → 클라이언트에서 hydrate 시 loader 재실행 없이 바로 사용
```

**클라이언트 내비게이션 시:**

```
Link 클릭
  → loader 실행 (클라이언트에서 Server Function 호출)
  → 결과로 컴포넌트 렌더링
```

💡 **loader vs useQuery 언제 뭘 쓰나?**

```
loader  →  라우트 진입 시 반드시 필요한 데이터 (없으면 화면 자체가 의미 없음)
           병렬 실행 보장, waterfall 방지
           ex) 게시글 상세 페이지의 게시글 데이터

useQuery →  컴포넌트 레벨 데이터, 조건부 패칭, 폴링, 낙관적 업데이트
            ex) 댓글 목록 (게시글과 별도로 로드해도 되는 경우)
```

### Step 4 — Nitro 서버 엔진

TanStack Start의 서버 레이어는 **Nitro**가 담당합니다. Nitro는 Nuxt팀이 만든 서버 엔진으로, 하나의 코드베이스로 다양한 환경에 배포할 수 있게 해줍니다.

```
Nitro 어댑터 → 배포 대상
  node          → Node.js 서버
  vercel        → Vercel Edge / Serverless
  cloudflare    → Cloudflare Workers
  netlify       → Netlify Functions
  bun           → Bun 런타임
```

```ts
// app.config.ts
import { defineConfig } from "@tanstack/react-start/config";

export default defineConfig({
  server: {
    preset: "vercel", // 이 한 줄로 Vercel 배포 설정 완료
  },
});
```

💡 **Next.js와의 차이**
Next.js는 Vercel에 최적화되어 있고 다른 환경 배포 시 설정이 복잡합니다.
TanStack Start는 Nitro 덕분에 배포 환경 변경이 설정 한 줄입니다.

### Step 5 — 스트리밍 (Streaming SSR)

TanStack Start는 React의 `Suspense`와 통합하여 스트리밍 SSR을 지원합니다.

```tsx
export const Route = createFileRoute("/dashboard")({
  component: () => (
    <div>
      <h1>대시보드</h1>

      {/* 이 부분은 데이터 준비되는 즉시 스트리밍으로 전송 */}
      <Suspense fallback={<Skeleton />}>
        <SlowDataComponent />
      </Suspense>
    </div>
  ),
});
```

```
클라이언트에 전달되는 순서:

1. 즉시: <h1>대시보드</h1> + <Skeleton /> (HTML 전송 시작)
2. 데이터 준비되면: <SlowDataComponent /> 내용 스트리밍
   (전체 페이지를 기다릴 필요 없음)
```

💡 **왜 중요한가?**
기존 SSR은 모든 데이터가 준비될 때까지 HTML을 아무것도 보내지 않았습니다.
스트리밍 SSR은 준비된 부분부터 즉시 전송하므로 **Time To First Byte(TTFB)가 줄어듭니다.**

---

# 정리: TanStack Start가 해결하는 문제의 본질

TanStack Query가 **서버 데이터의 비동기 상태**를 해결했다면,
TanStack Router가 **클라이언트 라우팅의 타입 불안전성**을 해결했다면,
TanStack Start는 이 둘을 묶어 **풀스택 타입 안전성**을 해결합니다.

```
TanStack Query  →  "서버 데이터를 어떻게 잘 캐싱하고 동기화할까?"
TanStack Router →  "라우트 경로를 어떻게 타입 안전하게 다룰까?"
TanStack Start  →  "서버와 클라이언트를 어떻게 타입 안전하게 연결할까?"
```

세 가지의 공통점:

```
1. Headless / 선언적  →  "어떻게 보여줄지"는 개발자 몫
2. 타입 안전          →  런타임 에러보다 컴파일 에러
3. 명시적 경계        →  무엇이 서버고 무엇이 클라이언트인지 코드에서 명확
```

`createFileRoute`, `createServerFn`, `loader` 이 세 가지가 실행되는 동안,
내부에서는 **Vite 플러그인(코드 변환) → Nitro(서버 라우팅) → TanStack Router(클라이언트 라우팅) → React(렌더링)** 이라는 레이어가 차례로 작동합니다.
개발자는 라우트와 서버 함수를 선언하고, 타입은 자동으로 end-to-end로 흐릅니다.

---

## TanStack 생태계 전체 관계도

```
                    TanStack Start (풀스택 프레임워크)
                    ┌─────────────────────────────────┐
                    │  Server Functions (createServerFn)│
                    │  SSR / Streaming                 │
                    │  Nitro (서버 엔진)               │
                    └──────────────┬──────────────────┘
                                   │ 포함
                    ┌──────────────▼──────────────────┐
                    │      TanStack Router             │
                    │  파일 기반 라우팅 + 타입 안전    │
                    └──────────────┬──────────────────┘
                                   │ 함께 사용
          ┌────────────────────────┼──────────────────┐
          ▼                        ▼                  ▼
   TanStack Query          TanStack Table       TanStack Virtual
   서버 상태 관리           Headless 테이블      가상화 리스트
```

TanStack Start는 나머지 라이브러리들을 대체하는 게 아니라 **위에서 감싸는 레이어**입니다.
Start 안에서 Query, Table, Virtual을 그대로 사용합니다.
