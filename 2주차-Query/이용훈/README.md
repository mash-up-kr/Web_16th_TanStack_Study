# 2주차 - useQuery는 내부적으로 어떻게 상태를 관리하는가

`useQuery`를 매일 쓰면서도 "왜 이게 알아서 캐싱되지?", "같은 키를 여러 컴포넌트에서 써도 왜 요청이 한 번만 가지?" 같은 질문을 제대로 파고든 적이 없었다.
그래서 이번엔 `useQuery` 내부 구조를 직접 들여다봤다.

> 이 글의 내용은 TanStack Query 코어 메인테이너 Dominik Dorfmeister(TkDodo)의 글과 `@tanstack/query-core` 소스코드를 직접 참조했다.

---

## 1. 이게 무엇인지

---

### 1-1. 4개 레이어로 이루어진 내부 구조

`useQuery` 한 줄 뒤에는 4개의 클래스가 협력하고 있다.

```
QueryClient
  └── QueryCache
        └── Query  ←──────── QueryObserver
                                   │
                              useQuery (React)
                                   │
                              컴포넌트 re-render
```

| 클래스 | 역할 | 위치 |
|---|---|---|
| `QueryClient` | 사용자가 직접 생성하는 진입점. 캐시와 기본 옵션을 보유 | `queryClient.ts` |
| `QueryCache` | queryHash → Query 인스턴스를 저장하는 Map. 이벤트 시스템 포함 | `queryCache.ts` |
| `Query` | 실제 상태를 보유하는 상태 머신. fetch 실행과 Observer 목록 관리 | `query.ts` |
| `QueryObserver` | 하나의 Query를 구독하고, 컴포넌트 리렌더를 트리거 | `queryObserver.ts` |

이 4개는 모두 `@tanstack/query-core` 패키지에 있다. React, Vue, Solid 등 어떤 프레임워크에도 의존하지 않는 순수 TS 코어다.
React용 어댑터(`@tanstack/react-query`)는 이 위에 얹히는 얇은 레이어일 뿐이다.

> *"The react package only has around 100 lines of code and mostly delegates to the framework agnostic core."*
> → "react 패키지는 약 100줄 정도의 코드만 있고, 대부분 프레임워크에 무관한 코어에 위임한다."
> — Dominik Dorfmeister ([Inside React Query](https://tkdodo.eu/blog/inside-react-query))

---

### 1-2. QueryCache — 캐시의 실체

QueryCache의 내부는 단순하다. `queryHash` 문자열을 키로, `Query` 인스턴스를 값으로 갖는 **Map** 하나다.

```typescript
// packages/query-core/src/queryCache.ts
#queries = new Map<string, Query>()
```

`queryHash`는 `queryKey` 배열로부터 생성된다. `['todos', 1]`과 `['todos', 2]`는 다른 해시 → 다른 Query 인스턴스.
`useQuery({ queryKey: ['todos'] })`를 앱 어디서 몇 번을 호출하든, 같은 해시면 **동일한 Query 인스턴스**를 반환한다.

```typescript
// QueryCache.build() — Query를 가져오거나 없으면 새로 만든다
let query = this.get(queryHash)
if (!query) {
  query = new Query({ client, queryKey, queryHash, ... })
  this.add(query)
}
return query
```

이것이 캐싱과 중복 요청 방지의 출발점이다.

---

## 2. 어떻게 동작하는지

---

### 2-1. Query의 상태: status와 fetchStatus는 독립적이다

많은 사람들이 Query의 상태를 단순히 `loading | success | error` 세 가지로 이해한다.
실제로는 **두 개의 독립적인 축**으로 관리된다.

```typescript
// packages/query-core/src/query.ts
export interface QueryState<TData, TError> {
  data: TData | undefined
  error: TError | null
  status: QueryStatus    // 'pending' | 'success' | 'error'
  fetchStatus: FetchStatus  // 'idle' | 'fetching' | 'paused'
  // ... 그 외 카운터, 타임스탬프 등
}
```

- **`status`**: 지금까지 성공적으로 데이터를 받은 적 있는지. 한번 `success`가 되면 웬만해선 내려가지 않는다.
- **`fetchStatus`**: 지금 이 순간 네트워크 요청이 진행 중인지.

즉 `{ status: 'success', fetchStatus: 'fetching' }`은 완전히 정상적인 상태다.
**"캐시된 데이터를 보여주면서 백그라운드에서 최신 데이터를 가져오는 중"** 을 의미한다.
이 덕분에 이전 데이터가 화면에서 사라지지 않고, 조용히 업데이트된다.

---

### 2-2. 상태 전이: Reducer 패턴

Query 내부에서 상태 변경은 Redux 스타일의 `#dispatch()` 메서드를 통해서만 일어난다.

```typescript
// packages/query-core/src/query.ts (간략화)
#dispatch(action: Action<TData, TError>): void {
  const reducer = (state) => {
    switch (action.type) {
      case 'fetch':
        return { ...state, fetchStatus: 'fetching', ... }
      case 'success':
        return { ...state, data: action.data, status: 'success', fetchStatus: 'idle' }
      case 'error':
        return { ...state, error: action.error, status: 'error', fetchStatus: 'idle' }
      // ...
    }
  }
  this.state = reducer(this.state)

  // 상태가 바뀌면 즉시 모든 Observer에게 알린다
  notifyManager.batch(() => {
    this.observers.forEach((observer) => observer.onQueryUpdate())
    this.#cache.notify({ query: this, type: 'updated', action })
  })
}
```

주목할 점: `case 'fetch'`에서 기존 `data`가 있으면 `status`를 `'pending'`으로 되돌리지 않는다.
백그라운드 리페치 시 `fetchStatus`만 `'fetching'`으로 바뀌고 `status`는 `'success'`를 유지한다. 화면에 있는 데이터가 사라지지 않는 이유다.

---

### 2-3. useQuery 호출 시 실제로 일어나는 일

```typescript
// packages/react-query/src/useBaseQuery.ts

// 1. Observer를 딱 한 번만 생성 (React.useState 초기화 함수 이용)
const [observer] = React.useState(
  () => new Observer(client, defaultedOptions)
)

// 2. useSyncExternalStore로 구독
React.useSyncExternalStore(
  React.useCallback(
    (onStoreChange) => {
      const unsubscribe = observer.subscribe(
        notifyManager.batchCalls(onStoreChange)
      )
      observer.updateResult()
      return unsubscribe
    },
    [observer],
  ),
  () => observer.getCurrentResult(),  // getSnapshot
)
```

**흐름 정리:**

1. `useQuery()` 호출 → `QueryObserver` 인스턴스 생성 (컴포넌트당 1개)
2. `observer.subscribe()` → `QueryCache`에서 해당 키의 `Query` 인스턴스를 가져오거나 생성
3. Query에 Observer가 등록되고, GC 타이머가 취소됨 (`clearGcTimeout()`)
4. 마운트 조건에 따라 fetch 실행 여부 결정 (`shouldFetchOnMount`)
5. 이후 Query 상태 변경 시 → Observer → `onStoreChange()` 호출 → React 리렌더

```typescript
// packages/query-core/src/queryObserver.ts
protected onSubscribe(): void {
  if (this.listeners.size === 1) {
    this.#currentQuery.addObserver(this)

    if (shouldFetchOnMount(this.#currentQuery, this.options)) {
      this.#executeFetch()
    }
    // ...
  }
}
```

언마운트 시엔 반대로:

```typescript
protected onUnsubscribe(): void {
  if (!this.hasListeners()) {
    this.destroy()  // Observer 제거 → Query의 observers 배열에서도 제거
  }
}
```

Query에서 마지막 Observer가 떠나면 GC 타이머가 시작된다 (`scheduleGc()`). `gcTime`(기본 5분) 후에 QueryCache에서 제거된다.

---

### 2-4. 중복 요청 방지 (Deduplication)

동일한 `queryKey`를 쓰는 컴포넌트가 동시에 10개 마운트된다고 가정하면:

1. `QueryCache.build()` → 10번 호출되지만 같은 `Query` 인스턴스 반환
2. 첫 번째 `observer.subscribe()` → fetch 시작, `fetchStatus: 'fetching'`
3. 나머지 9개의 `observer.subscribe()` → Query 등록은 하되, fetch는 건너뜀

```typescript
// packages/query-core/src/query.ts
// fetch() 내부 - 이미 fetching 중이면 같은 promise 반환
if (this.state.fetchStatus !== 'idle') {
  if (this.#retryer) {
    return this.#retryer.promise  // 동일한 promise 객체 반환
  }
}
```

네트워크 요청은 **단 1번**만 일어나고, 10개의 Observer 모두 결과를 받는다.

---

### 2-5. Observer에서 컴포넌트까지: 리렌더 체인

```
Query.#dispatch(action)
  → this.state = reducer(this.state)
  → notifyManager.batch(() => {
      observers.forEach(o => o.onQueryUpdate())
    })
      → observer.updateResult()
        → shouldNotifyListeners() 체크
          → listeners.forEach(l => l())
            → onStoreChange() (useSyncExternalStore 콜백)
              → React가 리렌더 예약
                → observer.getCurrentResult() 호출 (getSnapshot)
                  → 컴포넌트가 새 데이터로 렌더링
```

모든 알림은 `notifyManager.batch()`로 묶인다. 여러 상태 변경이 동시에 일어나도 Observer 알림은 한 번만 플러시된다.

---

### 2-6. Proxy 기반 Tracked Props — 불필요한 리렌더 방지

`useQuery`의 반환값은 단순한 객체가 아니라 **Proxy**로 감싸져 있다.

```typescript
// packages/query-core/src/queryObserver.ts
trackResult(result) {
  return new Proxy(result, {
    get: (target, key) => {
      this.trackProp(key)  // 접근한 프로퍼티를 기록
      return Reflect.get(target, key)
    },
  })
}
```

컴포넌트가 `result.data`만 접근하면 `#trackedProps = Set { 'data' }`가 된다.
이후 `isFetching`이 바뀌어도 (백그라운드 리페치 등), Observer는 `data`가 변경되지 않았으므로 리렌더를 트리거하지 않는다.

```typescript
const shouldNotifyListeners = () => {
  // trackedProps에 포함된 프로퍼티가 실제로 바뀌었을 때만 true
  return Object.keys(this.#currentResult).some((key) => {
    const changed = this.#currentResult[key] !== prevResult[key]
    return changed && this.#trackedProps.has(key)
  })
}
```

`notifyOnChangeProps`를 수동으로 지정할 수도 있지만, 기본값이 이미 Proxy 추적이라 대부분의 경우 자동으로 최적화된다.

---

## 3. 왜 이렇게 설계했는지

---

### 3-1. 왜 status와 fetchStatus를 분리했는가

단일 status 필드로 설계했다면 "데이터가 있는데 지금 재요청 중"인 상태를 표현할 방법이 없다.
`loading`으로 가면 기존 데이터가 사라지고, `success`로 유지하면 재요청 중임을 알 수 없다.

두 축을 분리함으로써 가능한 조합이 생긴다:

| status | fetchStatus | 의미 |
|---|---|---|
| `pending` | `idle` | 데이터 없음, 요청도 없음 (enabled: false) |
| `pending` | `fetching` | 첫 로딩 중 |
| `success` | `idle` | 데이터 있음, 조용한 상태 |
| `success` | `fetching` | 데이터 있음 + 백그라운드 리페치 중 |
| `error` | `fetching` | 오류 상태에서 재시도 중 |

---

### 3-2. 왜 useState가 아닌 useSyncExternalStore를 쓰는가

React 18의 Concurrent Mode에서 `useState`로 외부 상태를 구독하면 **Tearing** 현상이 발생할 수 있다.
하나의 렌더 사이클 내에서 동일한 외부 스토어의 스냅샷이 다른 값을 반환하는 상황이다.

`useSyncExternalStore`는 React가 공식 제공하는 외부 스토어 구독 훅으로, 동일 렌더 내에서 스냅샷 일관성을 보장한다.
`getSnapshot` (`getCurrentResult()`)이 항상 일관된 값을 반환하도록 강제하는 구조다.

---

### 3-3. 왜 Reducer 패턴을 쓰는가

모든 상태 전이를 `#dispatch()` 한 곳으로 모으면:
- 상태 변경 이력 추적이 쉽다 (DevTools 지원)
- 예측 불가능한 상태 조합이 발생하지 않는다
- 각 action에 대한 처리가 명확히 분리된다

TanStack Query DevTools가 상태 변화를 실시간으로 보여줄 수 있는 것도 이 구조 덕분이다.

---

### 3-4. 핵심 흐름 한 장 요약

```
useQuery({ queryKey, queryFn })
  │
  ├─ QueryObserver 생성 (컴포넌트당 1개)
  │
  ├─ QueryCache.build(queryHash)
  │     └─ 기존 Query 반환 or 새 Query 생성 (캐시 적중/미스)
  │
  ├─ query.addObserver(observer) + clearGcTimeout()
  │
  ├─ shouldFetchOnMount? → query.fetch()
  │     ├─ fetchStatus !== 'idle'? → 기존 promise 반환 (dedup)
  │     └─ Retryer 생성 → queryFn() 실행
  │           ├─ dispatch('success') → state 업데이트
  │           │     └─ observers.forEach(o => o.onQueryUpdate())
  │           │           └─ shouldNotifyListeners(trackedProps)?
  │           │                 └─ onStoreChange() → React 리렌더
  │           └─ dispatch('error') → 재시도 or 최종 실패
  │
  └─ 언마운트 → observer.destroy() → query.removeObserver()
                                          └─ observers.length === 0?
                                                └─ scheduleGc(gcTime)
```

---

## 참고 자료

- [Inside React Query — Dominik Dorfmeister (TkDodo)](https://tkdodo.eu/blog/inside-react-query) — 전체 구조의 출발점이 된 글
- [query-core/src/query.ts — GitHub](https://github.com/TanStack/query/blob/main/packages/query-core/src/query.ts) — Query 상태 머신, Reducer, Observer 관리
- [query-core/src/queryObserver.ts — GitHub](https://github.com/TanStack/query/blob/main/packages/query-core/src/queryObserver.ts) — Proxy 추적, shouldNotifyListeners
- [query-core/src/queryCache.ts — GitHub](https://github.com/TanStack/query/blob/main/packages/query-core/src/queryCache.ts) — QueryHash → Query Map
- [query-core/src/queryClient.ts — GitHub](https://github.com/TanStack/query/blob/main/packages/query-core/src/queryClient.ts) — 진입점, invalidateQueries
- [react-query/src/useBaseQuery.ts — GitHub](https://github.com/TanStack/query/blob/main/packages/react-query/src/useBaseQuery.ts) — useSyncExternalStore 통합
- [query-core/src/subscribable.ts — GitHub](https://github.com/TanStack/query/blob/main/packages/query-core/src/subscribable.ts) — Pub/Sub 베이스 클래스
