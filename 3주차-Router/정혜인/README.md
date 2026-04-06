# TanStack Router란?

> "React Router는 URL을 React에 연결한다. TanStack Router는 URL을 TypeScript 타입 시스템에 연결한다.”

TanStack Router는 단순한 라우팅 라이브러리가 아니라, URL 상태를 타입 안전하게 관리하는 상태 관리 시스템입니다.

# 어떻게 사용할까?

1. 라우트 정의하기

```jsx
import {
  createRootRoute,
  createRoute,
  createRouter,
} from "@tanstack/react-router";

// 루트 라우트
const rootRoute = createRootRoute({
  component: RootLayout,
});

// 자식 라우트
const usersRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: "/users",
  // ① 검색 파라미터 스키마 정의 → 타입 자동 추론
  validateSearch: (search) => ({
    page: Number(search.page ?? 1),
    filter: String(search.filter ?? ""),
  }),
  // ② 데이터 로더
  loader: async ({ context }) => {
    return fetchUsers();
  },
  component: UsersPage,
});

const userDetailRoute = createRoute({
  getParentRoute: () => usersRoute,
  path: "$userId", // ← path params
  beforeLoad: async ({ params }) => {
    // ③ 가드 / 인증 체크
    if (!isAuthenticated()) throw redirect({ to: "/login" });
  },
  loader: async ({ params }) => {
    return fetchUser(params.userId); // params.userId: string (타입 보장)
  },
});

// 라우트 트리 조립
const routeTree = rootRoute.addChildren([
  usersRoute.addChildren([userDetailRoute]),
]);

// 라우터 생성
export const router = createRouter({ routeTree });
```

2. React에 주입

```jsx
// main.tsx
import { RouterProvider } from "@tanstack/react-router";

function App() {
  return <RouterProvider router={router} />;
}
```

3. React 내에서 사용하기

```jsx
function UsersPage() {
  // search params — 타입 자동 추론 (validateSearch에서)
  const { page, filter } = useSearch({ from: "/users" });

  // loader 데이터
  const users = useLoaderData({ from: "/users" });

  return (
    // Link도 타입 안전
    <Link to="/users/$userId" params={{ userId: "123" }}>
      ...
    </Link>
  );
}
```

# 왜 사용하는 걸까?

React Router보다 더 강력한 타입 추론이 가능해집니다. 아래 예시를 보겠습니다.

React Router의 경우 타입이 string, undefined로 나옵니다.

```jsx
// React Router
const params = useParams();
params.userId; // string | undefined... -> 뭔지 모름
```

TanStack Router는 라우트 정의에서 타입이 자동으로 추론됩니다.

```jsx
// TanStack Router
const { userId } = useParams({ from: "/users/$userId" });
// userId: string  -> 100% 보장, undefined 없음
```

| 기존 라우팅                                  | TanStack Router                                |
| -------------------------------------------- | ---------------------------------------------- |
| 타입 없는 params (string \| undefined)       | path/search params 전부 TypeScript 타입 추론   |
| 수동 데이터 페칭 (컴포넌트 안에서 useEffect) | Loaders: 라우트 진입 전에 데이터 미리 로드     |
| 코드 스플리팅 번거로움                       | lazy() 내장, 파일 기반 라우팅 지원             |
| search params가 raw string                   | validateSearch로 스키마 정의, 타입+검증 동시에 |
| 인증 가드 구현 복잡                          | beforeLoad로 라우트별 가드 선언적으로          |
| Preload 직접 구현                            | hover/focus 시 자동 preload (intent 모드)      |

# 좀 더 깊게 파보기

전체 아키텍처 구조는 아래와 같습니다.

```jsx
[사용자 코드]
      <Link>, useNavigate(), useParams(), useSearch(), useLoaderData()
          ↓
  [react-router/]                ← React 어댑터 레이어
      RouterProvider.tsx         ← router를 Context에 주입
      Matches.tsx                ← 현재 매치된 라우트를 렌더링
      useRouterState.tsx         ← RouterStores에 구독
      useSearch, useParams...    ← 각 훅들
          ↓
  [router-core/]                 ← 프레임워크 무관 라우팅 엔진
      router.ts                  ← Router 클래스 (중심 클래스)
      route.ts                   ← 라우트 정의 타입/클래스
      load-matches.ts            ← beforeLoad + loader 실행 엔진
      stores.ts                  ← RouterStores (반응형 상태)
      history.ts                 ← 브라우저 History API 래퍼
      searchParams.ts            ← search param 파싱/직렬화
      new-process-route-tree.ts  ← URL → RouteMatch[] 매칭 로직
```

## Route 정의 객체

route.ts를 보면 라우트 하나가 갖는 속성들은 아래와 같습니다.

```jsx
// route.ts — 주요 옵션들
  {
    path: '/users',            // URL 패턴
    validateSearch: (raw) => parsed,  // search params 스키마
    beforeLoad: async (ctx) => void,  // 가드/컨텍스트 주입
    loader: async (ctx) => data,      // 데이터 로딩
    component: () => <JSX />,        // UI 컴포넌트
    pendingComponent: () => <JSX />, // 로딩 중 UI
    errorComponent: () => <JSX />,   // 에러 UI
    notFoundComponent: () => <JSX />,// 404 UI
  }
```

라우트들은 addChildren()으로 트리 구조를 만들고, createRouter({ routeTree })로 라우터에 등록됩니다.

## RouterStores — 반응형 상태 시스템

Query는 Subscribable + QueryObserver로 반응형을 구현했는데, Router는 어떻게 할까요?

Router는 @tanstack/react-store를 기반으로 한 세분화된 Store 시스템을 씁니다.

(stores.ts:72-116에서 RouterStores 인터페이스)

```jsx
// stores.ts
  export interface RouterStores<TRouteTree> {
    // 개별 atom 스토어들
    status: RouterWritableStore<'idle' | 'pending'>
    isLoading: RouterWritableStore<boolean>
    location: RouterWritableStore<ParsedLocation>
    resolvedLocation: RouterWritableStore<ParsedLocation | undefined>
    matchesId: RouterWritableStore<Array<string>>      // 현재 매치 ID 목록

    // 파생(derived) 스토어들
    activeMatchesSnapshot: ReadableStore<Array<AnyRouteMatch>>  // matchesId에서 계산
    hasPendingMatches: ReadableStore<boolean>

    // 라우트 ID별 매치 스토어 캐시
    getMatchStoreByRouteId: (routeId: string) => RouterReadableStore<AnyRouteMatch | undefined>

    // 호환성용 전체 상태 스토어
    __store: RouterReadableStore<RouterState>
  }
```

> [!TIP]
> ✨ **왜 하나의 큰 상태 대신 여러 스토어로 쪼갰을까?**
>
> `useParams({ from: '/users/$userId' })`는 해당 라우트의 params가 바뀔 때만 리렌더돼야 합니다. isLoading이 바뀌거나 다른 라우트의 data가 바뀌어도 불필요하게 리렌더되면 안 됩니다.
>
> 각 구독자가 자신에게 필요한 스토어만 구독 → 최소한의 리렌더링 가능해집니다.

```jsx
실제 연결 지점 — react-router/routerStores.ts:13-26:

  // routerStores.ts — React 어댑터가 스토어 구현체를 주입하는 곳
  export const getStoreFactory: GetStoreConfig = (opts) => {
    if (isServer ?? opts.isServer) {
      // SSR: 반응성 불필요 → 단순 값 저장
      return {
        createMutableStore: createNonReactiveMutableStore,
        createReadonlyStore: createNonReactiveReadonlyStore,
        batch: (fn) => fn(),
      }
    }
    // 클라이언트: @tanstack/react-store의 반응형 createStore 사용
    return {
      createMutableStore: createStore,   // ← 반응형!
      createReadonlyStore: createStore,
      batch: batch,
    }
  }
```

> Query와 비교를 해보면? 🤔
> Query는 Subscribable → useSyncExternalStore로 React와 연결했습니다. Router는 @tanstack/react-store의 createStore + useStore 훅으로 연결합니다. 둘 다 "외부 스토어를 React에 안전하게 구독하는" 동일한 목표지만, Router는 더 세분화된 atom 방식을 씁니다.

## 네비게이션 전체 흐름

```jsx
router.navigate({ to: '/users/$userId', params: { userId: '123' } })
    │
    ▼
  history.push(href)            ← 브라우저 History API 호출
    │
    ▼
  matchRoutes(newLocation)      ← URL → RouteMatch[] 매칭
    │  (new-process-route-tree.ts)
    │  "어떤 라우트가 이 URL에 해당하는가?"
    ▼
  loadMatches(matches)          ← 각 매치에 대해 순서대로:
    ├─ [1] beforeLoad()         ← 인증 체크, 컨텍스트 주입
    │       (실패 시 redirect throw)
    ├─ [2] loader()             ← 데이터 로딩 (병렬 가능)
    │       (완료 시 match.status = 'success')
    └─ [3] updateMatch()        ← match 상태 업데이트
    │
    ▼
  RouterStores 업데이트          ← location, matchesId, 각 match store
    │
    ▼
  useStore() → React 리렌더     ← 변경된 store를 구독 중인 컴포넌트만 리렌더
    │
    ▼
  Matches → Match 렌더링        ← 라우트 트리를 재귀적으로 렌더링
```

## loadMatches — beforeLoad와 loader의 실행 엔진

load-matches.ts가 네비게이션의 핵심입니다. 매치된 라우트 배열을 받아서, 루트부터 리프(leaf)까지 순서대로 실행합니다.\

```jsx
루트 라우트 beforeLoad
    └─ 자식 라우트 beforeLoad
         └─ 손자 라우트 beforeLoad
              └─ (모두 성공 시) 각 라우트 loader 실행
```

왜 beforeLoad가 loader보다 먼저일까?

⇒ beforeLoad는 인증, 권한 체크, 또는 하위 라우트들이 공통으로 필요한 컨텍스트를 준비하는 곳입니다. 예를 들어 부모 라우트의 beforeLoad에서 { user: currentUser }를 반환하면,
자식 라우트의 loader에서 ctx.user로 접근 가능합니다.

## React 어댑터 연결 — useRouterState

```jsx
// useRouterState.tsx:74-86
return useStore(router.stores.__store, (state) => {
  if (opts?.select) {
    if (opts.structuralSharing ?? router.options.defaultStructuralSharing) {
      const newSlice = replaceEqualDeep(
        previousResult.current,
        opts.select(state),
      );
      previousResult.current = newSlice;
      return newSlice; // 내용물이 같으면 이전 참조 재사용 → 불필요한 리렌더 방지
    }
    return opts.select(state);
  }
  return state;
});
```

useStore는 Query의 useSyncExternalStore와 동일한 역할입니다. select 함수로 전체 RouterState 중 필요한 부분만 구독할 수 있고, structuralSharing으로 내용물이 같으면 참조를
재사용해 불필요한 리렌더를 막습니다.

### 전체 다이어그램

```jsx

  [앱 초기화]
  createRouter({ routeTree })
    └─ processRouteTree()   ← 라우트 트리 파싱, 패턴 컴파일
    └─ createRouterStores() ← 반응형 store들 초기화
    └─ history.listen()     ← 브라우저 popstate 이벤트 구독

  <RouterProvider router={router}>
    └─ routerContext.Provider  ← router 인스턴스를 Context에 저장
    └─ <Matches />             ← 현재 매치 렌더링

  [네비게이션 발생]
  <Link to="/users/123">  or  router.navigate(...)
    │
    ├─ history.push()
    ├─ matchRoutes() → RouteMatch[]
    ├─ loadMatches()
    │   ├─ beforeLoad (루트→리프 순)
    │   └─ loader (병렬 가능)
    └─ RouterStores 업데이트
        └─ useStore() → 해당 store를 구독 중인 컴포넌트만 리렌더
            └─ <Match> 컴포넌트가 route.component 렌더링
```
