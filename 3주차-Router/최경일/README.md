## 🍀 TanStack Router란?

> URL을 관리하는 라이브러리. 모든 게 TypeScript 타입으로 연결되어 있다.
> 

---

### ✏️ 다른 라우터와 뭐가 다른가

먼저 레이어부터 구분해야 한다.

```
Next.js   →  Pages Router 또는 App Router (내장)
React SPA →  React Router 또는 TanStack Router (설치 필요)
```

**Pages Router** (Next.js 12 이하 기본 방식)
`pages/` 폴더 구조가 곧 URL이다. 전역 레이아웃이 `_app.tsx` 하나뿐이라 중첩 레이아웃 구성이 불편하다. 데이터는 `getServerSideProps` / `getStaticProps`로 가져온다.

```
pages/
  index.tsx            →  /
  products/
    index.tsx          →  /products
    [id].tsx           →  /products/123
  cart.tsx             →  /cart
```

**App Router** (Next.js 13+ 신규 방식)
`app/` 폴더를 쓰고, 각 폴더에 `layout.tsx`를 놓을 수 있어서 중첩 레이아웃이 자연스럽다. React Server Components를 기본으로 지원한다. `page.tsx`, `loading.tsx`, `error.tsx` 파일명이 예약어로 동작한다.

```
app/
  layout.tsx           ←  전역 레이아웃
  page.tsx             →  /
  products/
    layout.tsx         ←  상품 전용 레이아웃
    page.tsx           →  /products
    [id]/
      page.tsx         →  /products/123
  cart/
    page.tsx           →  /cart
```

**React Router** (독립 라이브러리, 2014~)
라우트를 코드로 직접 선언한다. 가장 오래된 표준이고 커뮤니티가 크다. v7부터 loader, action이 추가됐지만 타입 안전성은 framework mode에서만 동작한다.

```tsx
<Routes>
  <Route path="/" element={<Layout />}>
    <Route index element={<Home />} />
    <Route path="products" element={<Products />} />
    <Route path="products/:id" element={<ProductDetail />} />
    <Route path="cart" element={<Cart />} />
  </Route>
</Routes>

const { id } = useParams()
// id: string | undefined — 타입 불안전, 캐스팅 필요
```

**TanStack Router** (독립 라이브러리)
파일명이 URL이 되고, 파라미터 타입이 자동 추론됨. React Router에서 타입 때문에 불편했던 부분을 해결한 버전이라고 보면 됨.

---

### ✏️ 타입 안전성이 왜 좋은가

URL 오타, 파라미터 누락 같은 실수를 **런타임(사용자가 클릭할 때)** 이 아니라 **컴파일 타임(코딩할 때)** 에 잡아준다.

```tsx
// React Router — 오타여도 코드가 돌아감. 클릭하면 404
<Link to="/productz/123">상품 보기</Link>

// TanStack Router — 존재하지 않는 경로면 저장 전에 빨간 줄
<Link to="/productz/$id" params={{ id: '123' }}>상품 보기</Link>
//         ^^^^^^^^ 컴파일 에러
```

파라미터 타입도 자동으로 추론된다.

```tsx
// React Router
const { id } = useParams()
// id: string | undefined

// TanStack Router
const { id } = Route.useParams()
// id: string — 자동 추론, 캐스팅 불필요
```

---

### ✏️ 파일 구조와 routeTree.gen.ts

파일명 = URL. `$`를 붙이면 동적 파라미터가 된다.

```
src/routes/
  __root.tsx           ←  전역 레이아웃
  index.tsx            →  /
  products/
    index.tsx          →  /products
    $id.tsx            →  /products/123  ($가 파라미터)
  cart.tsx             →  /cart
  routeTree.gen.ts     ←  자동 생성 (수정 금지)
```

`routeTree.gen.ts`는 내가 만드는 파일이 아니다. 파일을 저장하면 Vite 플러그인이 `routes/` 폴더를 스캔해서 자동으로 만들어준다. 이 파일 덕분에 TypeScript가 앱에 어떤 URL이 존재하는지 알게 된다.

```tsx
// products/$id.tsx
export const Route = createFileRoute('/products/$id')({
  component: ProductDetail,
})

function ProductDetail() {
  const { id } = Route.useParams()
  // id: string — 자동 추론
}
```

---

### ✏️ 한눈에 비교

|  | Pages Router | App Router | React Router | TanStack Router |
| --- | --- | --- | --- | --- |
| 종류 | Next.js 내장 | Next.js 내장 | 독립 라이브러리 | 독립 라이브러리 |
| 라우팅 방식 | 파일 기반 | 파일 기반 | 선언적 코드 | 파일 + 코드 선택 |
| 타입 안전성 | 없음 | 부분적 | framework mode만 | 완전 자동 추론 |
| 데이터 로딩 | getServerSideProps | Server Components | loader (v7~) | loader + 내장 캐시 |
| 중첩 레이아웃 | 불편 | layout.tsx | Outlet | Outlet |

→ 공식 비교 문서: https://tanstack.com/router/latest/docs/framework/react/comparison

<br/>

## 🍀 어떻게 타입이 야무지게 추론되는가

**총 3단계 파이프라인**으로 동작한다.

```
① 파일 저장  →  ② routeTree.gen.ts 자동 생성  →  ③ 타입이 앱 전체로 흘러들어감
```

각 단계를 차례로 파보자.

---

### ✏️ 1단계. 파일 저장 → 코드 생성기가 스캔

`$id.tsx` 같은 파일을 저장하는 순간, Vite 플러그인(내부적으로는 `@tanstack/router-generator`)이 `routes/` 폴더 전체를 스캔해서 `routeTree.gen.ts`를 자동으로 다시 작성한다.

이 과정을 단계로 나누면 이렇다.

```tsx
파일 저장
    ↓
Vite 플러그인이 routes/ 폴더 스캔
    ↓
파일명에서 경로 추출
  products/$id.tsx  →  /products/$id
  $id 부분          →  params: { id: string }
    ↓
routeTree.gen.ts 자동 생성
```

이 파일 안에는 이런 타입 맵이 생성된다.

```tsx
// routeTree.gen.ts (자동 생성)
interface FileRoutesByPath {
  '/': { ... }
  '/products': { ... }
  '/products/$id': {       // ← $id 파일명에서 추출
    id: '/products/$id'
    path: '/products/$id'
    fullPath: '/products/$id'
    params: { id: string } // ← $ 뒤 이름이 키가 됨
  }
}
```

`$id`라는 파일명에서 `$` 뒤에 오는 `id`를 뽑아서 `params: { id: string }`으로 만드는 것이다. 파일명이 `$userId.tsx`였다면 `params: { userId: string }`이 된다.

이 인터페이스가 만들어지고 나면 라우터 전체에서 자동완성, 타입 체크, `params` / `search` / `loader data` 접근에 활용된다. 내가 타입을 직접 선언하지 않아도 되는 이유가 이 파일 덕분이다.

---

### ✏️ 2단계. Register — 타입을 앱 전체에 주입하는 관문

`routeTree.gen.ts`가 생성됐다고 끝이 아니다. TypeScript는 이걸 "앱 전체에서 쓰는 라우터"라고 인식해야 해. 이걸 담당하는 게 `Register` 인터페이스다.

```tsx
// main.tsx — 딱 한 번만 씀
const router = createRouter({ routeTree })

declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router  // ← 여기에 꽂으면 끝
  }
}
```

`declare module`은 TypeScript의 **모듈 증강** 문법이다. 외부 라이브러리의 타입을 내가 직접 확장할 수 있는 방법인데, 여기선 `@tanstack/react-router` 안에 있는 빈 `Register` 인터페이스에 우리 앱의 라우터 타입을 끼워 넣는 것이다.

비유하자면 이렇다.

```tsx
routeTree.gen.ts  →  타입 정보가 담긴 USB 기기
Register          →  USB 허브 (꽂는 곳)
main.tsx에서 등록  →  허브에 꽂는 행위
```

한 번 꽂으면 앱 어디서든 — `<Link>`, `useParams()`, `useSearch()` 등 — 파일마다 따로 import 안 해도 그 타입을 가져다 쓸 수 있게 된다.

---

### ✏️ 3단계. createFileRoute — 경로 문자열로 타입이 연결됨

JSX로 라우트를 정의하면 TypeScript가 타입을 추론할 수 없다. 코드가 실행되기 전까지 어떤 경로들이 존재하는지 알 수 없기 때문이다. 그래서 TanStack Router는 JSX 방식을 쓰지 않는다.

대신 `createFileRoute`에 경로 문자열을 직접 넘긴다.

```tsx
export const Route = createFileRoute('/products/$id')({
  loader: ({ params }) => {
    params.id  // string — 자동 추론
  },
  component: ProductDetail,
})

function ProductDetail() {
  const { id } = Route.useParams()
  // id: string — 별도 타입 선언 없이 추론됨
}
```

`'/products/$id'`를 넘기는 순간 내부적으로 이렇게 동작한다.

```tsx
// createFileRoute 내부 시그니처 (단순화)
function createFileRoute<TPath extends keyof FileRoutesByPath>(path: TPath) {
  return function(options: RouteOptions<FileRoutesByPath[TPath]>) {
    // 경로 문자열 → FileRoutesByPath 조회 → params/search 타입 확정
  }
}
```

`TPath`에 `'/products/$id'`가 들어가면, `FileRoutesByPath['/products/$id']`에서 `{ params: { id: string } }`를 꺼내온다. 이게 `loader`, `useParams()`, `useSearch()` 전체에 흘러들어간다.

```
'/products/$id' 입력
    ↓
FileRoutesByPath['/products/$id'] 조회
    ↓
{ params: { id: string } } 꺼내옴
    ↓
loader, useParams(), useSearch() 전부 자동 추론
```

`id: string`이라고 직접 안 써도 되는 이유다. 경로 문자열 하나로 타입이 전부 따라오는 구조.

---

### ✏️ 전체 흐름 요약

```
1. $id.tsx 파일 저장
        ↓
2. Vite 플러그인이 스캔
        ↓
3. routeTree.gen.ts 자동 생성
   "이 앱에 /products/$id 경로가 있고, params는 { id: string }이야"
        ↓
4. main.tsx에서 Register에 router 등록
   "이게 우리 앱 라우터야" 라고 TypeScript한테 알림
        ↓
5. createFileRoute('/products/$id') 호출
   → routeTree.gen.ts에서 해당 경로 타입 조회
   → loader, useParams() 전부 자동 추론
        ↓
6. <Link to="/productz"> 오타 치면 바로 빨간 줄
```

TanStack Router는 손실 없는 타입 추론을 제공한다. 수많은 제네릭 타입 파라미터를 통해 개발자가 제공한 타입 정보를 API 전체와 앱 전반에 전파한다.

---

### ✏️ 왜 JSX로는 이게 안 되나

React Router처럼 JSX로 라우트를 쓰면:

```tsx
// TypeScript 입장에서 이건 "그냥 함수 호출들"
<Routes>
  <Route path="/products/:id" element={<ProductDetail />} />
</Routes>
```

TypeScript는 `<Route path="...">` 를 렌더링 시점에 평가하기 때문에, 어떤 경로가 존재하는지 컴파일 타임에 알 수 없다. 경로 목록이 코드 실행 전까지 확정이 안 되는 거야.

반면 TanStack Router는 `routeTree.gen.ts`에 경로 목록을 **정적으로** 다 적어두고, `createFileRoute`의 제네릭으로 그걸 연결하니까 TypeScript가 컴파일 타임에 전부 알 수 있다.

→ 공식 문서 (Type Safety): https://tanstack.com/router/v1/docs/framework/react/guide/type-safety

→ 공식 문서 (DX 결정): https://tanstack.com/router/v1/docs/framework/react/decisions-on-dx

<br/>

## 🍀 고급 기능

## ✏️ Route Masking

실제 URL은 `/posts/123`인데, 주소창에는 `/feed`로 보여주는 것이다.

**왜 쓰는가 — 모달 + URL의 딜레마**

인스타그램 피드를 생각해보자. 게시물을 클릭하면 모달이 뜨는데, 동시에 주소창 URL도 바뀐다.

URL이 바뀌면 생기는 이점이 있다.

```
공유      →  그 URL을 카톡으로 보내면 상대방도 같은 게시물을 볼 수 있다
뒤로가기  →  모달이 닫히고 피드로 돌아온다 (브라우저 히스토리에 쌓이기 때문)
새로고침  →  모달이 유지된다
```

그런데 URL을 바꾸려면 문제가 생긴다. 모달과 풀페이지는 **다른 컴포넌트**라서, 라우터에 별도 경로로 등록해야 한다.

```
/feed              →  피드 페이지
/posts/123         →  게시물 풀페이지
/posts/123/modal   →  게시물 모달  ← 이게 필요
```

근데 `/posts/123/modal`이 주소창에 그대로 노출되면 어색하다. 사용자 입장에서 모달을 띄웠을 뿐인데 URL이 이상하게 길어진다.

**마스킹이 이 딜레마를 해결한다.**

```
내부적으로는 /posts/123/modal  (모달 컴포넌트와 연결)
주소창에는   /posts/123        (깔끔하게 보임)
```

```tsx
<Link
  to="/posts/$id/modal"     // 실제 이동할 라우트 (모달 컴포넌트)
  params={{ id: '123' }}
  mask={{ to: '/posts/$id', params: { id: '123' } }}  // 주소창에는 이걸로
>
  게시물
</Link>
```

또는 라우터 레벨에서 전역으로 설정할 수도 있다. 링크가 여러 곳에 있을 때 하나씩 설정하면 빠뜨릴 수 있어서, 이 방식이 더 안전하다.

```tsx
const postModalMask = createRouteMask({
  routeTree,
  from: '/posts/$id/modal',  // 실제 경로
  to: '/posts/$id',          // 주소창에 보일 경로
  params: true,              // $id 파라미터 그대로 넘김
})

const router = createRouter({
  routeTree,
  routeMasks: [postModalMask],  // 배열 — 여러 마스크 등록 가능
})
```

**동작 정리**

```
피드에서 게시물 클릭
  → 내부적으로 /posts/123/modal 로 이동 (모달 컴포넌트 렌더링)
  → 주소창에는 /posts/123 으로 표시

뒤로가기
  → 모달 닫히고 피드로 복귀

URL 복사 후 새 탭에서 열기
  → /posts/123 풀페이지로 열림 (마스크 벗겨짐)

새로고침
  → /posts/123 풀페이지로 이동 (마스크 기준)
```

URL을 복사해서 새 탭에서 열면 풀페이지가 열리는 게 핵심이다. 피드 위에 모달로 보는 것과, URL로 직접 접근해서 풀페이지로 보는 것 — 같은 URL로 두 가지 경험을 모두 제공할 수 있다.

→ 공식 문서: https://tanstack.com/router/v1/docs/framework/react/guide/route-masking

---

### ✏️ Navigation Blocking

페이지 이탈을 막는 기능. 폼에 내용을 쓰다가 실수로 다른 페이지로 이동하면 작성 내용이 날아가는 걸 방지한다.

두 가지 방식이 있다.

**심플 방식** — `window.confirm` 브라우저 기본 팝업 사용

```jsx
import { useBlocker } from '@tanstack/react-router'

function 글쓰기() {
  const [내용, set내용] = useState('')

  useBlocker({
    shouldBlockFn: () => 내용.length > 0,
    // withResolver 없으면 브라우저 기본 confirm 뜸
  })
  // ...
}
```

**커스텀 UI 방식** — `withResolver: true`로 직접 confirm UI 구현

```jsx
const { proceed, reset, status } = useBlocker({
  shouldBlockFn: () => 내용.length > 0,
  withResolver: true,  // 라우터가 이탈을 멈추고 대기
})

return (
  <>
    {/* ... */}
    {status === 'blocked' && (
      <모달>
        <p>정말 떠나실 건가요?</p>
        <button onClick={proceed}>예</button>  {/* 이동 허용 */}
        <button onClick={reset}>아니요</button> {/* 현재 페이지 유지 */}
      </모달>
    )}
  </>
)
```

`shouldBlockFn`에서 `current`와 `next` 라우트 정보가 넘어오기 때문에, "특정 페이지에서 특정 페이지로 이동할 때만" 막는 세밀한 조건도 설정할 수 있다.

탭 닫기, 새로고침처럼 라우터가 직접 제어할 수 없는 이탈은 `enableBeforeUnload: true`를 추가하면 브라우저의 기본 떠남 방지 다이얼로그가 뜬다.

→ 공식 문서: https://tanstack.com/router/v1/docs/framework/react/guide/navigation-blocking

---

### ✏️ Deferred Data Loading

TanStack Router는 기본적으로 모든 loader가 완료될 때까지 기다렸다가 다음 라우트를 렌더링한다. 그런데 API마다 응답 속도가 다를 때 문제가 생긴다.

포스트 상세 페이지를 예로 들면:

```jsx
포스트 본문  API → 100ms
댓글 목록   API → 3000ms
추천 포스트 API → 2000ms
```

Deferred 없이는 가장 느린 API가 올 때까지 페이지 전환 자체가 안 된다. 사용자는 3초 동안 이전 페이지에서 기다려야 한다.

```jsx
Deferred 없음:  링크 클릭 → 3000ms 대기 → 그제서야 페이지 전환
Deferred 있음:  링크 클릭 → 100ms 대기 → 페이지 전환 → 느린 데이터는 나중에 채워짐
```

스켈레톤 UI와 역할이 다르다. 스켈레톤은 **페이지 전환 후** 빈 자리를 채우는 거고, Deferred는 **어떤 데이터가 올 때까지 페이지 전환을 기다릴지**를 결정하는 거다. 둘은 함께 쓴다.

`await` 유무로 "기다릴 데이터"와 "나중에 채울 데이터"를 구분한다.

```jsx
export const Route = createFileRoute('/posts/$id')({
  loader: async ({ params }) => ({
    post: await fetchPost(params.id),   // await — 이게 올 때까지만 기다리고 전환
    comments: fetchComments(params.id), // await 없음 — 전환 후에 알아서 채워짐
    related: fetchRelated(params.id),   // await 없음 — 마찬가지
  }),
  component: PostDetail,
})

function PostDetail() {
  const { post, comments, related } = Route.useLoaderData()

  return (
    <div>
      <본문 data={post} />               {/* ✅ 즉시 렌더링 */}

      <Suspense fallback={<스켈레톤 />}> {/* ⏳ 댓글 도착 전까지 스켈레톤 */}
        <Await promise={comments}>
          {(data) => <댓글목록 data={data} />}  {/* 🔜 댓글 도착 후 렌더링 */}
        </Await>
      </Suspense>

      <Suspense fallback={<스켈레톤 />}> {/* ⏳ 추천 도착 전까지 스켈레톤 */}
        <Await promise={related}>
          {(data) => <추천포스트 data={data} />}  {/* 🔜 추천 도착 후 렌더링 */}
        </Await>
      </Suspense>
    </div>
  )
}
```

```jsx
페이지 이동
  → 본문 즉시 렌더링
  → 댓글/추천 자리엔 스켈레톤
  → 각각 도착하는 대로 자동 교체
```

`<Await>` 컴포넌트는 Suspense 바운더리를 트리거하고, 서버에서도 스트리밍으로 동작한다. Promise가 resolve되면 결과가 직렬화되어 인라인 스크립트 태그로 클라이언트에 스트리밍된다.

→ 공식 문서: https://tanstack.com/router/v1/docs/framework/react/guide/deferred-data-loading

---

### ✏️ Scroll Restoration

SPA에서 뒤로가기 하면 스크롤이 맨 위로 올라가는 문제를 해결한다. SPA는 `history.pushState` API로 탐색하기 때문에 브라우저가 스크롤 위치를 자동 복원하지 못하고, 비동기 렌더링 때문에 페이지 높이도 미리 알 수 없다. 

설정은 한 줄이다.

```jsx
const router = createRouter({
  routeTree,
  scrollRestoration: true,
})
```

```jsx
상품 목록 스크롤 → 상품 클릭 → 상세 페이지
  → 뒤로가기
  → 아까 보던 스크롤 위치로 복원 ✅
```

특정 링크에서만 복원을 끄고 싶으면 resetScroll 속성을 false로 하면 된다.

```jsx
<Link to="/about" resetScroll={false}>
  About (스크롤 유지 안 함)
</Link>
```

중첩된 스크롤 영역도 각각 독립적으로 복원된다. 채팅 앱처럼 사이드바 스크롤과 채팅 영역 스크롤을 따로 기억할 수 있다. 

→ 공식 문서: https://tanstack.com/router/v1/docs/framework/react/guide/scroll-restoration

---

### ✏️ Render Optimizations

URL search params가 바뀔 때 관련 없는 컴포넌트까지 리렌더되는 문제를 막아준다. 두 가지 기법이 있다.

**① Structural Sharing — 바뀐 키만 새 참조로 교체**

`/details?foo=f1&bar=b1`에서 `/details?foo=f1&bar=b2`로 이동할 때, `bar`만 바뀌었으면 `search.foo`는 참조가 그대로 유지되고 `search.bar`만 교체된다.

```jsx
// bar가 바뀌어도 foo의 참조는 유지 → foo를 쓰는 컴포넌트는 리렌더 안 됨
```

**② select — 구독 범위 좁히기**

```jsx
// 전체 구독 — page 바뀌면 이 컴포넌트도 리렌더됨
const { keyword, page } = Route.useSearch()

// keyword만 구독 — page가 바뀌어도 이 컴포넌트는 리렌더 안 됨
const keyword = Route.useSearch({ select: (s) => s.keyword })
```

`select` 함수가 객체를 반환하면 매번 새 객체가 만들어져서 계속 리렌더가 발생한다. 이 경우엔 `useShallow` 같은 shallow equality 비교를 함께 써야 한다.

**"리액트도 어차피 바뀐 부분만 리렌더하지 않나?"**

흔한 오해인데, 리액트는 실제로 **상태가 바뀐 컴포넌트 + 그 자식 전부**를 리렌더링한다.

```jsx
function 부모() {
  const [count, setCount] = useState(0)
  return (
    <>
      <자식A count={count} />
      <자식B />  {/* count랑 관계없는데도 리렌더됨 */}
    </>
  )
}
```

URL 상태는 이 문제가 더 심하다. search params는 라우터가 관리하는 전역 상태라서, URL이 조금이라도 바뀌면 이걸 구독하는 컴포넌트들이 줄줄이 리렌더될 수 있다. Structural Sharing과 select는 이 문제를 `memo` 없이 해결해준다.

→ 공식 문서: https://tanstack.com/router/v1/docs/framework/react/guide/render-optimizations

<br/>

TMI

- TanStack Router 공식문서를 AI 에디터에 주입해주는 vibe-rules라는 도구가 예전에 있었네요. 근데 26.03에 생긴 TanStack Intent로 이제 역사속으로…

<br/>

### **🤔 최종 내 느낌**

- 타입 체킹도 좋지만, 전 다른 유용한 기능들이 더 끌렸습니다 ㅎㅎ