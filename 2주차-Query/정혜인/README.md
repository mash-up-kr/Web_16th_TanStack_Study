# Query란 무엇일까?

---

> **"React는 UI 라이브러리이지, 데이터 페칭 라이브러리가 아니다."**
> 즉, \*\*\*\*TanStack Query는 React가 해결해주지 못하는 '서버 상태(Server State)'의 복잡성을 우아하게 해결하기 위해 탄생했습니다.

TanStack Query는 단순히 데이터를 가져오는(fetching) 도구가 아니라, **비동기 상태를 관리하는 상태 관리 라이브러리**입니다.

### **핵심 정의: 서버 상태 관리자**

우리가 흔히 아는 `Redux`, `Zustand`가 **클라이언트 상태(사용자 UI 인터랙션)**를 관리한다면, TanStack Query는 **서버 상태(API 데이터)**를 전담합니다.

- **서버 상태의 특징:** 사용자가 제어할 수 없는 원격 위치에 저장됨
  - 가져오거나 업데이트할 때 비동기 API가 필요함
  - 공유 소유권(Shared ownership)을 가지며 내가 모르는 사이 남에 의해 바뀔 수 있음
  - 주의하지 않으면 'Stale(오래된)' 상태가 되기 쉬움

### **아키텍처 구조**

TanStack Query는 **캐시 계층(Cache Layer)** 역할을 합니다.
`[UI 컴포넌트] ↔ [TanStack Query (Cache)] ↔ [서버 API]` 구조를 만들어, 컴포넌트가 서버에 직접 묻는 대신 캐시에 먼저 물어보게 합니다.

# Query는 어떻게 사용하는 걸까?

### 1. 기본 설정

애플리케이션 최상단에서 `QueryClient`를 생성하고 주입합니다.

```jsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5분 동안은 Fresh한 데이터로 간주
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Todos />
    </QueryClientProvider>
  );
}
```

### 2. 핵심 훅 `useQuery` 사용법

가장 중요한 것은 **Query Key**와 **Query Function**의 분리입니다.

```jsx
const { data, isLoading, error } = useQuery({
  queryKey: ["todos", { status: "done" }], // 1. 유니크한 키 (의존성 배열 역할)
  queryFn: fetchTodos, // 2. 실제 프로미스를 반환하는 함수
  staleTime: 5000, // 3. 옵션 (5초 후 stale로 변경)
});
```

### +) 추가 팁

아래에서 설명드릴 Step 3(낙관적 결과 읽기), Step 10(배칭)을 이해한다면 다음과 같이 활용할 수 있습니다.

- **Prefetching:** 사용자가 페이지를 이동하기 전에 미리 데이터를 캐시에 넣어두면, `getOptimisticResult` 덕분에 사용자는 로딩 창 없이 즉시 데이터를 보게 됩니다.
- **Tracked Queries:** `data`만 구조 분해 할당해서 사용하면 `isFetching`이 변해도 리렌더링되지 않는 최적화 기능을 누릴 수 있습니다.

# Query는 왜 쓰는 걸까?

### 1. 서버 상태의 고질적 문제 해결

서버 데이터를 직접 관리하려면 다음과 같은 복잡한 로직을 **직접** 짜야 합니다. TanStack Query는 이러한 문제를 해결해줍니다.

| **직접 구현 시 단점**                                               | **TanStack Query 솔루션**                                                   |
| ------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| **중복 요청** (동일 페이지 내 3개 컴포넌트가 같은 호출하는 경우 등) | **Query Deduplication**: 단 한 번의 요청만 수행                             |
| **캐시 만료** (언제 다시 불러와야 할지 판단 어려움)                 | **Stale-While-Revalidate**: 오래된 데이터 먼저 보여주고 백그라운드 업데이트 |
| **메모리 관리** (사용하지 않는 데이터가 메모리 점유)                | **Garbage Collection**: 지정된 시간(gcTime) 후 자동 삭제                    |
| **에러 처리** (일시적 네트워크 오류 시 대응)                        | **Smart Retries**: 지수 백오프($$1000 \times 2^{failureCount}$$             |
| ) 자동 재시도                                                       |

### 2. 선언적 코드

명령형으로 "데이터를 가져와서(fetch), 상태에 넣고(setState), 로딩 처리를 해라"라고 지시할 필요가 없습니다. 단지 "이 키에 해당하는 데이터는 이 함수로 가져오고, 상태는 이 정도 유지해줘"라고 선언하는 것이 가능해집니다.

### 3. UX 개선

- **Window Focus Refetching:** 사용자가 다른 탭을 보다가 돌아오면 최신 데이터를 자동으로 가져옵니다. (항상 최신 정보 유지)
- **Optimistic Updates:** 서버 응답이 오기 전 UI를 먼저 업데이트하여 "빛의 속도"로 반응하는 앱을 만들 수 있습니다.

즉, TanStack Query를 사용하면 이와 같은 장점이 있습니다. 비동기 데이터가 가질 수 있는 수많은 예외 상황(네트워크 끊김, 중복 호출, 오래된 데이터 등)을 **프레임워크 수준에서 견고하게 처리하기 때문에 개발자가 UI 로직에만 집중하게 만들기 위해서** 라고 할 수 있습니다.

> `useQuery({ queryKey, queryFn })` 한 줄이 실행되는 동안, 내부에서는 3개의 레이어가 차례로 작동합니다.
> `Subscribable` → `QueryObserver` → `useBaseQuery` → React

**흐름 다이어그램**

```jsx
useQuery(options)
  └─ useBaseQuery(options, QueryObserver)
       ├─ new QueryObserver(client, options)   ← Subscribable 상속
       ├─ observer.getOptimisticResult()
       └─ useSyncExternalStore(
            subscribe: observer.subscribe(onStoreChange),
            getSnapshot: observer.getCurrentResult()
          )
```

### 1. `Subscribable` — Observer의 뼈대

**WHAT**
모든 핵심 클래스(`QueryObserver`, `QueryCache`, `MutationObserver`)가 상속하는 추상 베이스 클래스입니다. 구독자(listener) 목록을 `Set`으로 관리하고 구독/해지 인터페이스를 통일합니다.

핵심 코드 (subscribable.ts:1-30)

- `listeners: Set<TListener>` — 구독자 집합
- `subscribe(listener)` — 리스너 등록 후 `onSubscribe()` 훅 호출, 해지 함수 반환
- `onSubscribe() / onUnsubscribe()` — 빈 메서드로, 서브클래스에서 오버라이드

**HOW**

`QueryObserver`는 이를 상속해서 `onSubscribe()`에서 실제 쿼리 fetch를 트리거합니다. `useBaseQuery`는 `observer.subscribe(onStoreChange)`를 호출해 React와 연결을 해줍니다.

**WHY**

- 구독 로직을 한 곳에서 관리 → 모든 클래스가 동일한 인터페이스를 가짐
- `Set` 기반이라 중복 구독 자동 방지
- `hasListeners()` 로 구독자가 없을 때 fetch를 멈추는 최적화가 가능

### 2. `useQuery` — 진입점

**WHAT**
사용자가 직접 호출하는 훅입니다. 내부는 단 한 줄로 `QueryObserver`를 `useBaseQuery`에 주입하는 것이 전부입니다.

핵심 코드 (useQuery.ts:50-52)

`export function useQuery(options, queryClient?) {
  return useBaseQuery(options, QueryObserver, queryClient)
}`

타입 오버로드가 3개인 이유는 `initialData` 유무에 따라 반환 타입이 `DefinedUseQueryResult` 이거나 `UseQueryResult`로 달라지기 때문입니다.

**HOW**

`useInfiniteQuery`, `useSuspenseQuery` 등도 동일하게 `useBaseQuery`에 다른 Observer(ex. `InfiniteQueryObserver`)를 주입하는 방식으로 구현됩니다.

**WHY**
Observer를 외부에서 주입받는 구조 덕분에 `useBaseQuery` 하나로 모든 쿼리 훅의 React 연동 로직을 공유할 수 있습니다. 변하는 것(어떤 Observer)과 변하지 않는 것(React 연동 방식)을 분리한 설계입니다.

### 3. `useBaseQuery` — React와 코어를 잇는 다리

**WHAT**
React 훅과 `query-core` 사이의 실제 연결 지점입니다. Observer를 생성하고, `useSyncExternalStore`로 상태 변화를 구독해 React가 리렌더를 할 수 있게 만들어쥽니다.

| 단계          | 코드                                           | 역할                                 |
| ------------- | ---------------------------------------------- | ------------------------------------ |
| Observer 생성 | `React.useState(() => new Observer(...))`      | 최초 1회만 생성, 리렌더 무관         |
| 낙관적 결과   | `observer.getOptimisticResult(options)`        | 구독 전에 현재 캐시 상태를 먼저 읽음 |
| 구독          | `useSyncExternalStore(subscribe, getSnapshot)` | Observer 변화 → React 리렌더 연결    |
| 옵션 동기화   | `useEffect(() => observer.setOptions(...))`    | 렌더마다 최신 options 반영           |

**WHY(왜 `useSyncExternalStore`인가)**

- React 18 Concurrent Mode에서 `useState`/`useReducer`로 외부 스토어를 구독하면 **tearing**(UI가 서로 다른 시점의 데이터를 동시에 보여주는 현상)이 발생할 수 있습니다.
- `useSyncExternalStore`는 React가 공식적으로 외부 스토어 구독을 위해 제공하는 훅으로, tearing을 방지하고 서버/클라이언트 스냅샷을 분리 처리해줍니다.

**`notifyManager.batchCalls(onStoreChange)`가 있는 이유**
여러 쿼리가 동시에 업데이트될 때 각각 리렌더를 일으키지 않스빈다. 하나의 배치로 묶어서 한 번만 리렌더되도록 최적화가 됩니다.

# 동작을 파헤쳐보자

## useQuery 한 줄이 화면에 데이터를 그리기까지 - 전체 내부 흐름

### 전체 흐름 요약도

```jsx
useQuery(options)
  │
  ▼
useBaseQuery(options, QueryObserver)           ← React 레이어 진입
  ├─ [1] new QueryObserver(client, options)    ← Observer 생성
  │       └─ setOptions() → #updateQuery()
  │               └─ queryCache.build()        ← Query 인스턴스 확보
  │
  ├─ [2] observer.getOptimisticResult()        ← 구독 전 현재 캐시 스냅샷 읽기
  │
  └─ [3] useSyncExternalStore(
           subscribe: observer.subscribe(onStoreChange)
             └─ Subscribable.subscribe()
               └─ onSubscribe()
                 ├─ query.addObserver(this)
                 └─ #executeFetch()
                   └─ query.fetch()
                     └─ createRetryer({ fn: queryFn })
                       └─ retryer.start()
                         └─ queryFn({ queryKey, signal })
                                 │
                      ┌──────────┘
                      ▼  (성공)
               query.setData(data)
                 └─ #dispatch({ type: 'success' })
                         ├─ reducer → query.state 갱신
                         └─ notifyManager.batch()
                           └─ observer.onQueryUpdate()
                             └─ updateResult()
                               └─ #notify()
                                 └─ listener(currentResult)  ← onStoreChange 호출
	                                           │
                         ┌───────────────────┘
                         ▼
                    useSyncExternalStore → React 리렌더
           getSnapshot: observer.getCurrentResult()
         )

```

### Step 1 — `useQuery` → `useBaseQuery`

**코드** (useQuery.ts:50-51)

```jsx
export function useQuery(options, queryClient?) {
  return useBaseQuery(options, QueryObserver, queryClient)
}
```

`useQuery`는 진입점일 뿐 실질적으로 하는 일이 없습니다. `QueryObserver`를 "어떤 Observer를 쓸지"의 전략으로 주입합니다.

> **왜 이렇게 분리했나?**
> `useInfiniteQuery`는 `InfiniteQueryObserver`를, `useSuspenseQuery`는 별도 옵션을 넘기는 방식으로 동일한 `useBaseQuery` 로직을 재사용합니다. Observer만 바꾸면 동작이 달라지는 전략 패턴입니다.

### Step 2 — Observer 생성 & Query 인스턴스 확보

**코드** (useBaseQuery.ts:91-97)

```jsx
const [observer] = React.useState(() => new Observer(client, defaultedOptions));
```

`React.useState`의 initializer 함수 형태로 넘겼기 때문에 **최초 렌더 1회만 실행**됩니다. 이후 리렌더에서는 같은 Observer 인스턴스를 재사용합니다.

Observer 생성자 내부 (queryObserver.ts:81-88):

```jsx
constructor(client, options) {
  super()                  // Subscribable 초기화 (listeners = new Set())
  this.#client = client
  this.#currentThenable = pendingThenable()
  this.bindMethods()
  this.setOptions(options) // ← 여기서 Query 인스턴스를 확보
}

```

`setOptions()` 안에서 `#updateQuery()`가 호출되고 여기서 `queryCache.build()`로 **캐시에서 기존 Query를 찾거나 새로 생성**합니다 (queryObserver.ts:699-700):

```jsx
#updateQuery(): void {
  const query = this.#client.getQueryCache().build(this.#client, this.options)
  // queryHash가 같으면 캐시에서 꺼내고, 없으면 새로 만들어서 캐시에 저장
  ...
  this.#currentQuery = query
}

```

> **핵심 포인트**
> 같은 `queryKey`를 가진 `useQuery`가 여러 컴포넌트에서 호출되어도, `Query` 인스턴스는 `queryHash` 기준으로 **단 하나**만 존재합니다. Observer만 여러 개가 붙습니다.

### Step 3 — 낙관적 결과 먼저 읽기

**코드** (useBaseQuery.ts:99-100)

```jsx
const result = observer.getOptimisticResult(defaultedOptions);
```

`useSyncExternalStore` **호출 이전에** 현재 캐시 상태를 먼저 읽습니다.

- **이유**: 컴포넌트가 렌더될 때 이미 캐시에 데이터가 있으면(`staleTime` 내 재접속 등) 구독을 설정하기 전에 그 데이터를 즉시 반환해서 불필요한 로딩 상태를 건너뜁니다. 구독 설정과 첫 렌더 사이의 간격에서 발생할 수 있는 **데이터 누락을 방지**하는 안전장치의 역할도 합니다.

### Step 4 — `useSyncExternalStore`로 구독 연결

**코드** (useBaseQuery.ts:102-120)

```jsx
React.useSyncExternalStore(
  React.useCallback(
    (onStoreChange) => {
      const unsubscribe = shouldSubscribe
        ? observer.subscribe(notifyManager.batchCalls(onStoreChange))
        : noop;

      observer.updateResult(); // 구독 설정 사이에 놓친 업데이트 반영
      return unsubscribe;
    },
    [observer, shouldSubscribe],
  ),
  () => observer.getCurrentResult(), // client snapshot
  () => observer.getCurrentResult(), // server snapshot
);
```

`useSyncExternalStore`가 하는 일:

- **첫 번째 인자 (subscribe)**: 외부 스토어 변화를 구독하고 해지 함수를 반환
- **두 번째 인자 (getSnapshot)**: 현재 스토어 값을 읽는 함수. React가 리렌더 여부를 판단할 때 호출
- **세 번째 인자 (getServerSnapshot)**: SSR용 스냅샷

> **왜 `useState`가 아니라 `useSyncExternalStore`인가?**
>
> React 18의 Concurrent Mode에서는 렌더를 중단하고 재개하는 과정에서 `useState`로 외부 스토어를 구독하면 **tearing**이 발생할 수 있습니다. 같은 화면 안에서 컴포넌트 A는 이전 데이터를, B는 새 데이터를 동시에 보여주는 현상입니다. `useSyncExternalStore`는 React 팀이 이 문제를 해결하기 위해 공식적으로 제공한 훅입니다.

### Step 5 — `Subscribable.subscribe()` → `onSubscribe()` 트리거

`observer.subscribe(onStoreChange)` 호출 시 (subscribable.ts:8-17):

```jsx
subscribe(listener: TListener): () => void {
  this.listeners.add(listener)   // onStoreChange를 Set에 등록
  this.onSubscribe()             // QueryObserver에서 오버라이드된 훅 실행
  return () => {
    this.listeners.delete(listener)
    this.onUnsubscribe()
  }
}
```

> **`listeners.size === 1` 조건이 왜 중요한가?**
>
> 같은 Query에 Observer가 여러 개 붙어도 fetch는 **딱 한 번**만 실행되어야 합니다. 두 번째, 세 번째 Observer가 구독해도 이미 첫 번째가 fetch를 시작했기 때문에 중복 실행하지 않습니다.

`shouldFetchOnMount`의 판단 기준 (queryObserver.ts:755-764):

```jsx
function shouldLoadOnMount(query, options) {
  return (
    resolveEnabled(options.enabled, query) !== false &&
    query.state.data === undefined &&
    !(query.state.status === "error" && options.retryOnMount === false)
  );
}

function shouldFetchOnMount(query, options) {
  return (
    shouldLoadOnMount(query, options) ||
    (query.state.data !== undefined &&
      shouldFetchOn(query, options, options.refetchOnMount))
  );
}
```

- 캐시에 데이터가 없거나 (`data === undefined`)
- 데이터가 있더라도 stale 상태이고 `refetchOnMount`가 허용된 경우
- **`enabled !== false`** 체크와 **에러 상태 + `retryOnMount === false`이면 fetch를 하지 않습니다**. 즉, enabled: false 일 때 fetch를 하지 않는 이유라고 할 수 있습니다.

### Step 6 — `Query.fetch()` → `createRetryer`

**코드** (query.ts:386-544):
`Query.fetch()`는 단순히 queryFn을 실행하는 게 아니라, 7단계의 준비 과정을 거칩니다.

> 6-1. 중복 요청 방지\*\* (query.ts:390-406)

이미 fetch 중이면 상황에 따라 분기합니다.

- `cancelRefetch: true` + 데이터 존재 → 기존 fetch를 취소하고 새로 시작
- 그 외 → 기존 retryer의 promise를 반환 (중복 요청 방지)
- retryer가 이미 rejected 상태 → 무조건 새로 fetch 시작

```jsx
if (
  this.state.fetchStatus !== "idle" &&
  this.#retryer?.status() !== "rejected"
) {
  if (this.state.data !== undefined && fetchOptions?.cancelRefetch) {
    this.cancel({ silent: true }); // 기존 fetch 취소 후 새로 시작
  } else if (this.#retryer) {
    this.#retryer.continueRetry();
    return this.#retryer.promise; // 기존 promise 재사용
  }
}
```

> 6-2. AbortController 생성 + signal 트래킹 (query.ts:430-443)

Object.defineProperty로 signal을 Proxy처럼 감싸서, queryFn이 signal을 읽었는지 여부를 추적합니다. signal을 사용한 queryFn만 취소 시 상태를 되돌리기 위한 장치입니다.

```jsx
const abortController = new AbortController()
const addSignalProperty = (object: unknown) => {
  Object.defineProperty(object, 'signal', {
    enumerable: true,
    get: () => {
      this.#abortSignalConsumed = true  // "signal을 읽었다" 기록
      return abortController.signal
    },
  })
}
```

> 6-3. fetchFn 클로저 생성 (query.ts:446-475)

queryFnContext(`queryKey`, `meta`, `signal`, `client`)를 조합하고,
사용자가 작성한 queryFn을 호출하는 클로저를 만듭니다.

> 6-4. behavior.onFetch() 호출 (query.ts:502)

`this.options.behavior?.onFetch(context, this)`이 한 줄이 `useInfiniteQuery`가 동작하는 이유와 같습니다. `InfiniteQueryBehavior`가 여기서 fetchFn을 **페이지 병합 로직으로 교체**합니다. 일반 useQuery에서는 behavior가 없으므로 이 줄은 건너뜁니다.

> 6-5. revertState 저장 (query.ts:505)

현재 state를 백업해둡니다. fetch가 취소(`CancelledError`)되면 이 상태로 롤백할 수 있게 하기 위한 안전장치입니다.

> 6-6. dispatch({ type: 'fetch' }) (query.ts:508-513)

`fetchStatus`를 `'fetching'`으로 전환합니다.
데이터가 없으면(`data === undefined`) `status`도 `'pending'`으로 바꿉니다.

> 6-7. createRetryer 생성 및 start (query.ts:516-546)

```jsx
this.#retryer = createRetryer({
  fn: context.fetchFn,
  retry: context.options.retry,
  retryDelay: context.options.retryDelay,
  networkMode: context.options.networkMode,
  onFail: (failureCount, error) => {
    this.#dispatch({ type: "failed", failureCount, error });
  },
  onPause: () => this.#dispatch({ type: "pause" }),
  onContinue: () => this.#dispatch({ type: "continue" }),
});

const data = await this.#retryer.start();
this.setData(data); // → dispatch({ type: 'success' })
```

- 단순히 "기존 promise 반환"이 아닌, `cancelRefetch` 옵션에 따라 **기존 fetch를 취소하고 새로 시작**할 수도 있습니다.. 또한 `retryer.status() !== 'rejected'` 조건도 있어서 retryer가 이미 reject된 상태면 무조건 새로 fetch를 시작합니다.

### Step 7 — `Retryer` 내부: 실제 fetch와 재시도 로직

**코드** (retryer.ts:141-204):

```jsx
const run = () => {
  try {
    promiseOrValue = config.fn(); // queryFn 실행
  } catch (error) {
    promiseOrValue = Promise.reject(error);
  }

  Promise.resolve(promiseOrValue)
    .then(resolve) // 성공 → thenable resolve
    .catch((error) => {
      const retry = config.retry ?? (isServer ? 0 : 3);
      const delay = defaultRetryDelay(failureCount); // 1s → 2s → 4s → 최대 30s

      const shouldRetry = failureCount < retry;

      if (!shouldRetry) {
        reject(error); // 최종 실패
        return;
      }

      failureCount++;
      config.onFail?.(failureCount, error); // dispatch('failed')

      sleep(delay)
        .then(() => (canContinue() ? undefined : pause())) // 오프라인이면 대기
        .then(() => run()); // 재시도
    });
};
```

재시도 지연 계산 (retryer.ts:48-50):

```jsx
function defaultRetryDelay(failureCount: number) {
  return Math.min(1000 * 2 ** failureCount, 30000)
  // 1회 실패: 1000ms, 2회: 2000ms, 3회: 4000ms, 최대 30초
}
```

> **`canContinue()` 조건** (retryer.ts:103-106):
> 네트워크가 오프라인이거나 탭이 백그라운드 상태이면 재시도를 즉시 하지 않고 대기(`pause`)합니다. 탭이 다시 포커스되거나 온라인이 되면 자동으로 재개합니다.

### Step 8 — 성공 후 상태 전파: `#dispatch` → `notifyManager`

fetch 성공 시 `query.setData(data)` 호출 → `#dispatch({ type: 'success' })` (query.ts:607-687):

```jsx
#dispatch(action: Action<TData, TError>): void {
  const reducer = (state: QueryState<TData, TError>): QueryState<TData, TError> => {
    switch (action.type) {  // action을 클로저로 캡처
      case 'success': ...
      case 'error': ...
    }
  }
  this.state = reducer(this.state)
  // ...
}
```

- reducer가 `#dispatch` **안에서 로컬 함수로 정의**됩니다. 클래스 외부에 별도 reducer가 있는 게 아니라 `#dispatch` 내부에서 클로저로 `action`을 캡처하는 구조임을 확인할 수 있습니다.

**`notifyManager.batch()`가 하는 일** (notifyManager.ts:52-65):

```jsx
batch: (callback) => {
  transactions++;
  try {
    result = callback(); // 모든 알림을 일단 큐에 쌓음
  } finally {
    transactions--;
    if (!transactions) {
      flush(); // 트랜잭션이 끝나면 한꺼번에 실행
    }
  }
};
```

**왜 배칭이 필요한가?**

예를 들어 `queryClient.invalidateQueries()`로 10개의 Query가 동시에 무효화되면, 10개의 Observer가 각각 리렌더를 트리거할 수 있습니다. `notifyManager.batch()`는 이 모든 알림을 하나의 마이크로태스크로 묶어서 **단 한 번의 React 배치 렌더**로 처리합니다.

### Step 9 — Observer 알림 → `useSyncExternalStore` → 리렌더”

- 1차 필터: shallowEqual (queryObserver.ts:656-658)
  → 결과 객체 전체가 이전과 같으면 리스너 호출 자체를 건너뜀
- 2차 필터: notifyOnChangeProps (queryObserver.ts:662-693)
  → 1차를 통과해도, 컴포넌트가 실제로 읽은 프로퍼티가 안 바뀌었으면 건너뜀
- #notify 호출 (queryObserver.ts:726-741)
  → 위 두 필터를 모두 통과해야 listener(currentResult) 호출
