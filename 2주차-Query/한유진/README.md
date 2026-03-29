# QueryClient, 너 누군데?

## QueryClient의 정의

QueryClient 는 서버 상태(Query, Mutation)를 전역에서 관리하는 중앙 관리자이다.
데이터를 저장하는 캐시의 역할 뿐 아니라, 언제 데이터를 다시 가져와야 하는지 판단하고 실행까지 담당하는 컨트롤러에 가깝다.

```typescript
export class QueryClient {
  #queryCache: QueryCache
  #mutationCache: MutationCache
  #defaultOptions: DefaultOptions
  #queryDefaults: Map<string, QueryDefaults>
  #mutationDefaults: Map<string, MutationDefaults>
  #mountCount: number
  #unsubscribeFocus?: () => void
  #unsubscribeOnline?: () => void

  constructor(config: QueryClientConfig = {}) {
    this.#queryCache = config.queryCache || new QueryCache()
    this.#mutationCache = config.mutationCache || new MutationCache()
    this.#defaultOptions = config.defaultOptions || {}
    this.#queryDefaults = new Map()
    this.#mutationDefaults = new Map()
    this.#mountCount = 0
  }
```

코드를 통해 알 수 있듯이, 쿼리클라이언트는

- QueryCache: query 데이터 캐시
- MutationCache: mutation 상태 관리
- defaultOptions: 전역 옵션
- 각 query / mutation의 기본 설정

과 같은 요소들을 내부에서 관리한다.

## QueryClient의 mount

```typescript

  mount(): void {
    this.#mountCount++
    if (this.#mountCount !== 1) return

    this.#unsubscribeFocus = focusManager.subscribe(async (focused) => {
      if (focused) {
        await this.resumePausedMutations()
        this.#queryCache.onFocus()
      }
    })
    this.#unsubscribeOnline = onlineManager.subscribe(async (online) => {
      if (online) {
        await this.resumePausedMutations()
        this.#queryCache.onOnline()
      }
    })
  }
```

`QueryClient` 는 여러 컴포넌트에서 동일한 인스턴스를 공유하기 때문에,
전역 이벤트 리스너가 중복 등록되지 않도록 mountCount를 사용해 최초 한 번만 등록한다.

이때 등록되는 이벤트는 두 가지이다.

- focusManager: 브라우저 탭이 포커스될 때 감지
- onlineManager: 네트워크 온라인/오프라인 상태 감지

그리고 이 두 가지 이벤트 리스너는 Query의 자동 재검증 기능의 핵심이다.

사용자가 탭을 다시 보았을 때 (focusManager) stale 쿼리를 자동 재검증하고, 사용자의 네트워크가 끊겼다가 다시 연결되면 (onlineManager) 일시 중지되었던 요청을 다시 처리하는 것이다.

## QueryClient의 mount 내부 동작

그러면 두 가지 이벤트 리스너는 어떻게 동작하는 것일까?

### FocusManager의 동작

먼저 FocusManager의 동작을 살펴보자.

```typescript
  constructor() {
    super()
    this.#setup = (onFocus) => {
      // addEventListener does not exist in React Native, but window does
      // eslint-disable-next-line @typescript-eslint/no-unnecessary-condition
      if (typeof window !== 'undefined' && window.addEventListener) {
        const listener = () => onFocus()
        // Listen to visibilitychange
        window.addEventListener('visibilitychange', listener, false)

        return () => {
          // Be sure to unsubscribe if a new handler is set
          window.removeEventListener('visibilitychange', listener)
        }
      }
      return
    }
  }
```

FocusManager는 내부적으로 #setup 함수를 가지고 있으며,
이 함수는 브라우저의 visibilitychange 이벤트를 구독하도록 설정한다.

이때 `#setup` 함수의 타입은 다음과 같다.

```typescript
type SetupFn = (
  setFocused: (focused?: boolean) => void
) => (() => void) | undefined;
```

매개변수로 `setFocused` 콜백 함수를 받고, 클린업 함수 또는 `undefined` 를 반환한다. 즉 이벤트 발생 시 실행할 콜백을 받고, 이벤트 해제를 위한 cleanup 함수를 반환하는 구조이다.

이 `#setup` 함수는 구독이 발생했을 때 실행된다.

```typescript
  protected onSubscribe(): void {
    if (!this.#cleanup) {
      this.setEventListener(this.#setup)
    }
```

`FocusManager` 클래스를 누군가 구독하여 `focusManager.onSubscribe()` 함수를 실행시키면 `this.setEventListener(this.#setup)` 이 호출된다.

그럼 `setEventListener` 는 어떤 동작을 할까?

```typescript
  setEventListener(setup: SetupFn): void {
    this.#setup = setup
    this.#cleanup?.()
    this.#cleanup = setup((focused) => {
      if (typeof focused === 'boolean') {
        this.setFocused(focused)
      } else {
        this.onFocus()
      }
    })
  }
```

`setEventListener` 는 이벤트 리스너의 생애주기를 관리하는 함수이다.

1. 새로운 setup 함수를 매개변수로 받아 내부에 저장
2. 기존에 등록되어 있던 리스너가 있으면 cleanup으로 제거
3. 이후 새로운 이벤트 리스너를 등록
4. 이벤트 리스너 함수 실행 이후 cleanup 함수를 저장

setup 함수를 실행하며 매개변수로 넘기는 콜백은 포커스 상태가 바뀌었을 때 어떻게 처리할지에 대한 로직이다.

- focused 값이 boolean이면 setFocused(focused)를 통해 상태를 명시적으로 업데이트
- boolean이 아니면 단순히 이벤트로 간주하고 onFocus()를 호출

그리고 setup이 반환하는 이벤트 리스너 해제 함수를 this.#cleanup에 저장한다. 그래서 다음에 다시 setEventListener가 호출되면 이 cleanup을 실행해서 기존 리스너를 제거할 수 있게 되는 구조이다.

### FocusManager의 동작 정리

### OnlineManager의 동작

## useQuery는 어떻게 동작할까?

## useMutation은 어떻게 동작할까?
