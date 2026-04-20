# Virtual이란 무엇일까?

---

> **"화면에 보이지 않는 1만 개의 DOM을 만들 필요가 있을까?"**
> TanStack Virtual은 "**눈에 보이는 것만 그린다**"는 단 하나의 아이디어를 집요하게 파고든 라이브러리입니다.

TanStack Virtual은 단순한 리스트 라이브러리가 아니라, **"얼마나 스크롤 되었는가(offset)"** 와 **"각 아이템은 어디에 있는가(measurements)"** 를 계산해서 **화면에 보일 아이템의 인덱스 배열만 반환**해주는 **계산 엔진**입니다.

### **핵심 정의: 렌더링 계산기 (Headless Windowing)**

Virtual은 **JSX를 리턴하지 않습니다.** 대신 "지금 몇 번 ~ 몇 번 인덱스를 그려라"를 알려주는 `VirtualItem[]` 배열을 리턴합니다.

- **Headless:** 스크롤 컨테이너 DOM도, 아이템 DOM도 당신이 직접 만듭니다.
- **계산 대상:** `count` (총 아이템 수), `estimateSize(i)` (예상 크기), `getScrollElement()` (스크롤 컨테이너)
- **결과물:** `getVirtualItems()` → 지금 렌더할 `VirtualItem[]` (key, index, start, end, size, lane)

### **아키텍처 구조**

Virtual은 **"외부 DOM의 상태를 구독하고 계산 결과를 콜백으로 밀어내는"** 구조입니다.

```
[스크롤 이벤트]      [ResizeObserver]          [옵션 변경]
      ↓                    ↓                       ↓
 ┌─────────────────────────────────────────────────────┐
 │              Virtualizer (1 클래스)                 │
 │   - scrollOffset       - measurementsCache          │
 │   - itemSizeCache      - range                      │
 │   - memo 기반 계산 파이프라인                        │
 └──────────────┬──────────────────────────────────────┘
                │ onChange(instance, sync)
                ▼
        [React Adapter: useVirtualizer]
                │ rerender()
                ▼
        [Component: virtualizer.getVirtualItems()]
```

- `virtual-core`: **1개의 클래스 (`Virtualizer`)** 가 전부. (index.ts, 1421줄)
- `react-virtual`: **훅 1개 (`useVirtualizer`)** 로 React lifecycle에 연결. (index.tsx, 101줄)

# Virtual은 어떻게 사용하는 걸까?

### 1. 기본 설정

컴포넌트 내부에서 `useVirtualizer`를 호출하고, 스크롤 컨테이너와 아이템을 직접 렌더링합니다.

```tsx
import { useVirtualizer } from "@tanstack/react-virtual";
import { useRef } from "react";

function VirtualList() {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: 10000, // 1. 총 아이템 수
    getScrollElement: () => parentRef.current, // 2. 스크롤 컨테이너
    estimateSize: () => 40, // 3. 예상 아이템 높이(px)
    overscan: 5, // 4. 버퍼(보이지 않는 영역도 몇 개 더 그릴지)
  });

  return (
    <div ref={parentRef} style={{ height: 400, overflow: "auto" }}>
      {/* 전체 높이를 차지하는 "spacer" */}
      <div style={{ height: virtualizer.getTotalSize(), position: "relative" }}>
        {virtualizer.getVirtualItems().map((item) => (
          <div
            key={item.key}
            data-index={item.index} // ← 필수! measureElement가 이걸로 찾음
            ref={virtualizer.measureElement} // ← 동적 높이일 때 필수
            style={{
              position: "absolute",
              top: 0,
              transform: `translateY(${item.start}px)`, // ← 핵심
              width: "100%",
            }}
          >
            Row {item.index}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 2. `VirtualItem`의 정체

`getVirtualItems()`가 리턴하는 객체의 형태입니다. (`virtual-core/src/index.ts:31-38`)

```ts
export interface VirtualItem {
  key: Key; // 안정적인 식별자 (getItemKey의 결과)
  index: number; // 원본 배열 인덱스
  start: number; // 스크롤 컨테이너 상단에서의 offset (px)
  end: number; // start + size
  size: number; // 측정된(또는 추정된) 크기
  lane: number; // 그리드/메이슨리용 열 번호 (기본 0)
}
```

### +) 추가 팁

- **동적 높이:** `ref={virtualizer.measureElement}` 를 꼭 넣어야 실제 DOM 크기를 측정해 캐시합니다. (`ResizeObserver` 기반)
- **prefetching / scrollToIndex:** `virtualizer.scrollToIndex(500, { align: 'center' })` 으로 특정 인덱스로 이동 가능.
- **Lanes (그리드):** `lanes: 3` 옵션 하나로 3열 그리드가 됩니다. 각 아이템의 `lane` 필드가 0/1/2로 계산됩니다.
- **Window 스크롤:** `useWindowVirtualizer`를 쓰면 `window` 자체를 스크롤 컨테이너로 사용 (SEO 친화적 무한스크롤).

# Virtual은 왜 쓰는 걸까?

### 1. DOM의 근본적 한계 해결

10,000개의 `<div>`를 React에 한 번에 렌더링하면 다음 문제가 생깁니다. Virtual은 이것을 전부 **계산 레벨**에서 해결합니다.

| **직접 구현 시 단점**                                       | **TanStack Virtual 솔루션**                                                   |
| ----------------------------------------------------------- | ----------------------------------------------------------------------------- |
| **초기 렌더 폭발** (DOM 1만 개 마운트 시 수 초의 blocking)  | **Windowing**: `calculateRange` 로 화면에 보이는 인덱스만 반환                |
| **스크롤 위치 계산 O(n)** (매 scroll 이벤트마다 탐색)       | **Binary Search**: `findNearestBinarySearch` 로 O(log n) (index.ts:1329-1353) |
| **동적 높이 측정 어려움** (이미지/텍스트 길이가 제각각)     | **ResizeObserver + itemSizeCache**: DOM 변화 자동 추적, Map 캐시              |
| **리렌더 폭증** (스크롤마다 전체 리스트 리렌더)             | **memo 파이프라인**: deps 비교로 같은 입력이면 이전 결과 재사용               |
| **측정 후 점프 현상** (추정치 vs 실측의 차이로 스크롤이 튐) | **scrollAdjustments**: delta만큼 스크롤 보정                                  |

### 2. 선언적이지만 명령형이기도 한 독특한 API

- **선언적 계산:** "count와 estimateSize만 줘. 내가 뭘 그릴지 알려줄게."
- **명령형 제어:** `scrollToIndex`, `scrollToOffset`, `measureElement` 는 사용자가 직접 호출
- → 계산은 TanStack이, DOM 조작은 개발자가. **헤드리스 철학의 교과서.**

### 3. UX 개선

- **Sync 리렌더 (`useFlushSync`):** 스크롤 이벤트가 발생한 **같은 프레임**에 DOM이 갱신됨. 흰 화면(빈 공간이 잠깐 보임) 방지.
- **Overscan:** 화면 밖 아이템을 미리 그려두어 빠른 스크롤에도 컨텐츠가 보이게 함.
- **Smooth Scroll Reconciliation:** `scrollToIndex` 호출 후 측정값이 바뀌어도 목표 위치로 자동 재보정 (`reconcileScroll`, index.ts:603-657).

> 💡
> **Q) 왜 `translateY(item.start)` 를 써야 할까? 그냥 `top`을 쓰면 안 되나?**  
> `transform`은 **GPU 합성 레이어**에서 처리되어 브라우저의 레이아웃(layout) / 페인트(paint) 단계를 건너뜁니다. `top`은 매번 레이아웃 재계산을 유발합니다. 60fps 스크롤을 유지하려면 `transform`이 필수입니다.
> (가이드상 대안으로 `top`을 써도 되지만, 긴 리스트에서는 스크롤 프레임 드랍이 바로 느껴집니다.)

즉, TanStack Virtual을 사용하면 **"스크롤 가능한 모든 UI의 근본 성능 문제"** 를 프레임워크 수준에서 해결해줍니다. 개발자는 **어떤 DOM을 그릴지**에만 집중할 수 있게 됩니다.

> `useVirtualizer({ count, estimateSize })` 한 줄이 실행되는 동안, 내부에서는 3개의 레이어가 차례로 작동합니다.
> `Virtualizer` → `memo 파이프라인` → `onChange → rerender` → React

**흐름 다이어그램**

```
useVirtualizer(options)
  └─ useVirtualizerBase(options)
       ├─ const [instance] = useState(() => new Virtualizer(resolvedOptions))
       ├─ instance.setOptions(resolvedOptions)   ← 매 렌더마다
       ├─ useLayoutEffect(() => instance._didMount(), [])
       └─ useLayoutEffect(() => instance._willUpdate())
             └─ scrollElement 연결 + ResizeObserver 구독
                  └─ scroll/resize 이벤트 → maybeNotify()
                       └─ calculateRange() 재실행
                            └─ (변화 있으면) onChange(instance, sync)
                                 └─ rerender() ← React 리렌더
                                      └─ virtualizer.getVirtualItems()
```

### 1. `memo` — Virtualizer의 뼈대

**WHAT**
TanStack Virtual에는 `Subscribable` 같은 pub/sub 베이스 클래스가 **없습니다.** 대신 **`memo` 함수**가 그 자리를 대신합니다. Virtualizer의 모든 계산 파이프라인(`getMeasurements`, `calculateRange`, `getVirtualIndexes`, `getVirtualItems`)이 이 하나의 유틸로 구성됩니다.

핵심 코드 (`virtual-core/src/utils.ts:5-82`)

```ts
export function memo<TDeps, TResult>(
  getDeps: () => [...TDeps],
  fn: (...args: [...TDeps]) => TResult,
  opts: { key, debug, onChange, ... }
) {
  let deps = opts.initialDeps ?? []
  let result: TResult | undefined

  function memoizedFunction(): TResult {
    const newDeps = getDeps()
    const depsChanged =
      newDeps.length !== deps.length ||
      newDeps.some((dep, i) => deps[i] !== dep)

    if (!depsChanged) return result!   // ← 이전 결과 재사용 (참조 동일)

    deps = newDeps
    result = fn(...newDeps)
    opts.onChange?.(result)
    return result
  }
  return memoizedFunction
}
```

**HOW**
`Virtualizer` 클래스 필드들이 전부 `memo()` 로 감싸져 있습니다.

```ts
// virtual-core/src/index.ts:771
private getMeasurements = memo(
  () => [this.getMeasurementOptions(), this.itemSizeCache], // deps
  (options, itemSizeCache) => { /* 각 아이템의 start/end/size/lane 계산 */ },
  { key: 'getMeasurements', debug: () => this.options.debug },
)

// virtual-core/src/index.ts:920
calculateRange = memo(
  () => [this.getMeasurements(), this.getSize(), this.getScrollOffset(), this.options.lanes],
  (measurements, outerSize, scrollOffset, lanes) => { /* 화면 범위 계산 */ },
  ...
)
```

**WHY**

- **참조 동일성 보장**: deps가 같으면 같은 배열 참조를 리턴 → React 리렌더에서 불필요한 재계산 스킵
- **체이닝 가능**: `calculateRange` 의 deps에 `getMeasurements()` 를 넣으면, 아이템 사이즈가 바뀌지 않는 한 range 재계산도 안 일어남
- **debug 모드**: `opts.debug` 가 true면 각 memo 블록 실행 시간을 콘솔에 색깔로 찍어줌. FPS 분석용
- **React.useMemo 아님**: Virtualizer는 React 외부 객체이므로 클래스 필드로 memo를 직접 관리

> 💡
> **`memo`가 TanStack Query의 `Subscribable`과 하는 일이 다른 이유**
>
> - Query는 **데이터 소유자가 여러 구독자에게 알림** → pub/sub 필요
> - Virtual은 **입력 (scrollOffset, itemSizes)이 바뀌면 출력 (range, items) 재계산** → 의존성 기반 파생만 필요
>   둘 다 "불필요한 연산 방지"가 목적이지만, 접근 방식이 다릅니다. TanStack은 문제의 성격에 맞게 패턴을 선택합니다.

### 2. `useVirtualizer` — 진입점

**WHAT**
사용자가 호출하는 훅. 내부는 `useVirtualizerBase`를 호출하면서 **DOM 기반 기본 observer들을 주입**하는 것이 전부입니다.

핵심 코드 (`react-virtual/src/index.tsx:67-82`)

```tsx
export function useVirtualizer(options) {
  return useVirtualizerBase({
    observeElementRect, // ResizeObserver로 컨테이너 크기 감지
    observeElementOffset, // scroll 이벤트로 offset 감지
    scrollToFn: elementScroll, // element.scrollTo 호출
    ...options,
  });
}
```

**HOW**
`useWindowVirtualizer` 도 동일 구조. observer만 window 버전으로 바꿔 주입합니다.

```tsx
// react-virtual/src/index.tsx:84-101
export function useWindowVirtualizer(options) {
  return useVirtualizerBase({
    getScrollElement: () => window,
    observeElementRect: observeWindowRect, // window.resize 이벤트
    observeElementOffset: observeWindowOffset, // window.scroll 이벤트
    scrollToFn: windowScroll,
    ...options,
  });
}
```

**WHY**
Query의 `useQuery(options, QueryObserver, client)` 와 **정확히 같은 전략 패턴**입니다.

- 변하는 것: 어디서 스크롤을 감지하나(element vs window)
- 변하지 않는 것: Virtualizer 생성, setOptions, rerender 연결
- → `useVirtualizerBase` 하나로 재사용. 새로운 스크롤 컨테이너 타입이 생겨도 observer 3개만 바꿔 주입하면 됨.

### 3. `useVirtualizerBase` — React와 코어를 잇는 다리

**WHAT**
Virtualizer 인스턴스 생성, 옵션 동기화, 마운트/언마운트 연결을 담당합니다.
**Query의 `useBaseQuery`에 해당하는 역할이지만, `useSyncExternalStore`를 쓰지 않습니다.**

핵심 코드 (`react-virtual/src/index.tsx:26-65`)

```tsx
function useVirtualizerBase({ useFlushSync = true, ...options }) {
  const rerender = React.useReducer(() => ({}), {})[1]; // ① 리렌더 트리거

  const resolvedOptions = {
    ...options,
    onChange: (instance, sync) => {
      if (useFlushSync && sync) {
        flushSync(rerender); // ② 동기 업데이트 (스크롤 프레임과 같은 tick)
      } else {
        rerender();
      }
      options.onChange?.(instance, sync);
    },
  };

  const [instance] = React.useState(
    () => new Virtualizer(resolvedOptions), // ③ 최초 1회만 생성
  );
  instance.setOptions(resolvedOptions); // ④ 매 렌더마다 옵션 동기화

  useIsomorphicLayoutEffect(() => {
    return instance._didMount(); // ⑤ 마운트/언마운트
  }, []);

  useIsomorphicLayoutEffect(() => {
    return instance._willUpdate(); // ⑥ 매 업데이트마다 scrollElement 재연결
  });

  return instance;
}
```

| 단계             | 코드                                              | 역할                                |
| ---------------- | ------------------------------------------------- | ----------------------------------- |
| Virtualizer 생성 | `React.useState(() => new Virtualizer(...))`      | 최초 1회만 생성                     |
| 옵션 동기화      | `instance.setOptions(resolvedOptions)`            | 매 렌더마다 옵션 병합               |
| 리렌더 트리거    | `useReducer(() => ({}), {})[1]`                   | `onChange` 에서 호출해 React를 깨움 |
| sync 렌더        | `flushSync(rerender)`                             | 스크롤 이벤트 tick 안에서 동기 커밋 |
| 마운트           | `useLayoutEffect(() => instance._didMount(), [])` | cleanup 함수 등록                   |
| 업데이트         | `useLayoutEffect(() => instance._willUpdate())`   | scrollElement 바뀌면 재구독         |

**WHY(왜 `useSyncExternalStore`가 아니라 `useReducer`인가)**

- TanStack Query: **외부 데이터가 언제든 바뀌며 tearing 위험 있음** → `useSyncExternalStore` 필요
- TanStack Virtual: **리렌더만 트리거하면 됨. snapshot 비교는 memo가 이미 처리** → `useReducer` + `flushSync` 로 충분
- 왜 `flushSync`? → 스크롤 이벤트는 브라우저의 **같은 프레임**에 DOM에 반영되어야 흰 공간(빈 spacer)이 보이지 않습니다. React 18의 자동 배칭을 우회해 즉시 커밋.

**`useFlushSync` 가 있는 이유**
일반 업데이트(옵션 변경, 비동기 측정 결과)까지 `flushSync` 로 처리하면 성능 손해 → **scroll/resize 같은 "sync가 필수인 경우"** 에만 `flushSync`, 나머지는 일반 리렌더로 처리하는 2-mode 구조. (`Virtualizer` 가 `notify(sync: boolean)` 로 구분해 알림)

# 동작을 파헤쳐보자

## useVirtualizer 한 줄이 화면에 1만 개의 row를 그리기까지 - 전체 내부 흐름

### 전체 흐름 요약도

```
useVirtualizer(options)
  │
  ▼
useVirtualizerBase(options)                    ← React 레이어 진입
  ├─ [1] new Virtualizer(options)              ← 인스턴스 1회 생성
  │       └─ setOptions() → 기본값 병합
  │
  ├─ [2] instance.setOptions(options)          ← 렌더마다 옵션 동기화
  │
  ├─ [3] useLayoutEffect(() => _didMount())    ← 마운트: cleanup 등록
  │
  └─ [4] useLayoutEffect(() => _willUpdate())
            └─ getScrollElement() 호출
                 ├─ cleanup() 이전 구독 해제
                 ├─ observeElementRect(cb)      ← ResizeObserver 시작
                 ├─ observeElementOffset(cb)    ← scroll 리스너 등록
                 │      └─ scroll 발생 시
                 │           ├─ scrollOffset 업데이트
                 │           ├─ isScrolling, scrollDirection 갱신
                 │           └─ maybeNotify()
                 │                 ├─ calculateRange() ← memo 파이프라인
                 │                 │     └─ getMeasurements()
                 │                 │           └─ itemSizeCache 기반 start/end 계산
                 │                 │     └─ findNearestBinarySearch (O(log n))
                 │                 └─ (deps 변경 시) onChange(instance, sync)
                 │                       └─ flushSync(rerender) ← React 리렌더
                 │                             └─ virtualizer.getVirtualItems()
                 │                                   └─ 화면에 보이는 VirtualItem[] 반환
                 └─ ResizeObserver (per-item)
                       └─ 아이템 크기 변경 감지
                             └─ resizeItem(index, size)
                                   ├─ itemSizeCache.set(key, size)
                                   ├─ pendingMeasuredCacheIndexes.push(index)
                                   ├─ (필요시) scrollAdjustments += delta
                                   └─ notify(false)  ← 일반 리렌더
```

### Step 1 — `useVirtualizer` → `useVirtualizerBase`

**코드** (`react-virtual/src/index.tsx:67-82`)

```tsx
export function useVirtualizer(options) {
  return useVirtualizerBase({
    observeElementRect,
    observeElementOffset,
    scrollToFn: elementScroll,
    ...options,
  });
}
```

`useVirtualizer`는 **어떤 DOM을 감지할 것인가(element vs window)** 의 전략만 주입합니다. 실제 로직은 `useVirtualizerBase`에 있음.

> **왜 이렇게 분리했나?**
> `useWindowVirtualizer`는 `observeWindowRect`, `observeWindowOffset`, `windowScroll`을 주입. 커스텀 스크롤 라이브러리(Lenis 등)와 통합하고 싶다면 이 세 함수만 override 하면 됩니다.

### Step 2 — Virtualizer 인스턴스 생성

**코드** (`react-virtual/src/index.tsx:50-52`)

```tsx
const [instance] = React.useState(
  () => new Virtualizer<TScrollElement, TItemElement>(resolvedOptions),
);
```

- `useState`의 initializer 형태 → **최초 렌더 1회만 생성**. 이후 렌더에서는 같은 인스턴스 재사용.
- 이 인스턴스가 `measurementsCache`, `itemSizeCache`, `laneAssignments`, `elementsCache` 등 **모든 상태의 소유자**가 됩니다.

Virtualizer 생성자 (`virtual-core/src/index.ts:449-451`):

```ts
constructor(opts: VirtualizerOptions<TScrollElement, TItemElement>) {
  this.setOptions(opts)
}
```

생성자는 매우 짧습니다. `setOptions()`가 기본값 병합만 처리하고 **실제 DOM 연결은 `_willUpdate()` 까지 미뤄둡니다** (SSR 대응).

> **핵심 포인트**
> `QueryObserver`처럼 생성 시점에 무언가를 트리거하지 않습니다. Virtualizer는 **마운트 후에** DOM에 연결됩니다.

### Step 3 — `setOptions` 로 옵션 동기화

**코드** (`virtual-core/src/index.ts:453-485`)

```ts
setOptions = (opts) => {
  Object.entries(opts).forEach(([key, value]) => {
    if (typeof value === "undefined") delete (opts as any)[key];
  });

  this.options = {
    debug: false,
    initialOffset: 0,
    overscan: 1,
    paddingStart: 0,
    // ... 17개 기본값
    ...opts,
  };
};
```

매 렌더마다 호출되며, 사용자가 넘긴 옵션과 기본값을 병합합니다.

- `undefined` 제거: 사용자가 명시적으로 `undefined`를 넘긴 경우 기본값을 쓰도록
- `overscan: 1`: 기본값이 1이라는 것에 주목. 성능이 안 좋다면 `overscan: 0`이 아니라 줄여야 할 옵션
- `lanes: 1`: 기본은 1열. `lanes: 3` 으로 바꾸면 그리드 모드

### Step 4 — `_willUpdate`: DOM 실제 연결

**코드** (`virtual-core/src/index.ts:534-589`)

```ts
_willUpdate = () => {
  const scrollElement = this.options.enabled
    ? this.options.getScrollElement()
    : null;

  if (this.scrollElement !== scrollElement) {
    // ← 컨테이너가 바뀌었을 때만
    this.cleanup();

    if (!scrollElement) {
      this.maybeNotify();
      return;
    }

    this.scrollElement = scrollElement;
    // targetWindow 결정 (iframe 지원)
    this.targetWindow =
      /* ... */

      // 기존 캐시된 아이템들을 ResizeObserver에 재등록
      this.elementsCache.forEach((cached) => {
        this.observer.observe(cached);
      });

    // ResizeObserver → 컨테이너 크기 변화 구독
    this.unsubs.push(
      this.options.observeElementRect(this, (rect) => {
        this.scrollRect = rect;
        this.maybeNotify();
      }),
    );

    // scroll 이벤트 → offset 변화 구독
    this.unsubs.push(
      this.options.observeElementOffset(this, (offset, isScrolling) => {
        this.scrollAdjustments = 0;
        this.scrollDirection = isScrolling
          ? this.getScrollOffset() < offset
            ? "forward"
            : "backward"
          : null;
        this.scrollOffset = offset;
        this.isScrolling = isScrolling;

        if (this.scrollState) {
          this.scheduleScrollReconcile(); // programmatic scroll 중이면 보정
        }
        this.maybeNotify();
      }),
    );

    this._scrollToOffset(this.getScrollOffset(), {
      adjustments: undefined,
      behavior: undefined,
    });
  }
};
```

**핵심 포인트**:

- 매 렌더마다 호출되지만 `scrollElement` 가 **바뀌지 않으면 아무 일도 안 함**. 불필요한 재구독 방지.
- 두 개의 observer 구독이 여기서 시작됩니다:
  1. **`ResizeObserver`** → 컨테이너 크기 변화 (반응형 대응)
  2. **scroll 이벤트** → offset 변화 (스크롤 추적)
- `ResizeObserver` 는 **per-item**으로도 작동합니다 (index.ts:400-446). 각 아이템 DOM의 크기 변화를 감지해 `resizeItem()` 호출.

### Step 5 — 사용자가 스크롤하면: `maybeNotify` 파이프라인

**코드** (`virtual-core/src/index.ts:491-513`)

```ts
private maybeNotify = memo(
  () => {
    this.calculateRange()
    return [
      this.isScrolling,
      this.range ? this.range.startIndex : null,
      this.range ? this.range.endIndex : null,
    ]
  },
  (isScrolling) => {
    this.notify(isScrolling)   // ← onChange 콜백 호출
  },
  { key: 'maybeNotify', initialDeps: [...] },
)
```

이 memo의 특이점:

- **`getDeps`에서 부수 효과를 실행**합니다. `this.calculateRange()` 를 먼저 실행해서 `range` 를 업데이트하고, **변화된 값을 deps로 사용**
- deps가 같으면 `notify()` 가 호출되지 않음 → **같은 범위를 보고 있는 동안은 리렌더 0회**
- 스크롤은 해도 visible range가 같으면 React는 아무것도 모름 (스크롤 이벤트 자체는 브라우저가 처리)

**왜 중요한가?**
스크롤은 초당 수백 번 발생할 수 있습니다. 그중 **visible index가 바뀔 때만** React를 깨우는 것이 Virtual의 성능 비밀.

### Step 6 — `calculateRange`: 화면에 보일 범위 계산

**코드** (`virtual-core/src/index.ts:920-942`, `1355-1421`)

```ts
calculateRange = memo(
  () => [this.getMeasurements(), this.getSize(), this.getScrollOffset(), this.options.lanes],
  (measurements, outerSize, scrollOffset, lanes) => {
    return (this.range = /* ... */ calculateRange({ measurements, outerSize, scrollOffset, lanes }))
  },
  ...
)
```

내부의 `calculateRange` 함수 (index.ts:1355-1421):

```ts
function calculateRange({ measurements, outerSize, scrollOffset, lanes }) {
  const lastIndex = measurements.length - 1;
  const getOffset = (index) => measurements[index]!.start;

  // ① startIndex를 바이너리 서치로 O(log n) 에 찾음
  let startIndex = findNearestBinarySearch(
    0,
    lastIndex,
    getOffset,
    scrollOffset,
  );
  let endIndex = startIndex;

  if (lanes === 1) {
    // ② 하단이 뷰포트를 넘을 때까지 endIndex 확장
    while (
      endIndex < lastIndex &&
      measurements[endIndex]!.end < scrollOffset + outerSize
    ) {
      endIndex++;
    }
  } else if (lanes > 1) {
    // ③ 그리드: 각 lane별로 확장
    // ... (lane 정렬 로직)
  }

  return { startIndex, endIndex };
}
```

**바이너리 서치가 핵심** (index.ts:1329-1353):

```ts
const findNearestBinarySearch = (low, high, getCurrentValue, value) => {
  while (low <= high) {
    const middle = ((low + high) / 2) | 0; // ← `| 0` 은 Math.floor 대체 (비트 연산이 더 빠름)
    const currentValue = getCurrentValue(middle);
    if (currentValue < value) low = middle + 1;
    else if (currentValue > value) high = middle - 1;
    else return middle;
  }
  return low > 0 ? low - 1 : 0;
};
```

**왜 바이너리 서치인가?**

- 100만 개 아이템에서 "scrollOffset 50000px에 해당하는 아이템 찾기" → 선형 탐색이면 최대 100만 번 비교
- 바이너리 서치 → 최대 20번 비교 (log₂ 1,000,000 ≈ 20)
- `measurements` 배열은 항상 `start` 기준 오름차순 정렬이라는 불변식 덕분에 가능

### Step 7 — `getMeasurements`: 각 아이템의 좌표 계산

**코드** (`virtual-core/src/index.ts:771-918`)

대략적 의사 코드로 정리하면:

```ts
private getMeasurements = memo(
  () => [this.getMeasurementOptions(), this.itemSizeCache],
  (options, itemSizeCache) => {
    // lanes가 바뀌면 전체 초기화
    if (this.lanesChangedFlag) {
      this.measurementsCache = []
      this.itemSizeCache.clear()
      this.laneAssignments.clear()
    }

    // 증분 재계산: min 인덱스부터만 다시 계산 (성능 최적화)
    const min = this.pendingMeasuredCacheIndexes.length > 0
      ? Math.min(...this.pendingMeasuredCacheIndexes)
      : 0
    this.pendingMeasuredCacheIndexes = []

    const measurements = this.measurementsCache.slice(0, min)

    for (let i = min; i < count; i++) {
      const key = getItemKey(i)
      const measuredSize = itemSizeCache.get(key)
      const size = typeof measuredSize === 'number'
        ? measuredSize                    // ← 실측값 있으면 사용
        : this.options.estimateSize(i)    // ← 없으면 추정치

      // 이전 아이템의 end + gap 에 붙임
      const start = prevInLane ? prevInLane.end + gap : paddingStart
      const end = start + size

      measurements[i] = { index: i, start, size, end, key, lane }
    }
    return measurements
  },
)
```

**핵심 포인트**:

- **증분 계산**: 중간 아이템 크기가 바뀌면 그 **뒤에 있는 아이템들만** 다시 계산. `pendingMeasuredCacheIndexes` 로 추적.
- **laneAssignments 캐시** (index.ts:382): 이전에 계산된 lane 번호를 재사용 → O(n) → O(1) 최적화
- **itemSizeCache**: key 기준 Map. `getItemKey(index)` 로 안정적 식별

### Step 8 — 측정 후 스크롤 보정: `scrollAdjustments`

이것이 **Virtual의 가장 어려운 문제**입니다.

**시나리오**: 사용자가 리스트 중간쯤 스크롤된 상태 → 상단의 보이지 않는 아이템의 실제 크기가 `estimateSize` 보다 50px 더 크다는 것이 측정됨 → 그대로 두면 화면이 위로 **50px 점프**

**해결** (`virtual-core/src/index.ts:1058-1086`):

```ts
resizeItem = (index: number, size: number) => {
  const item = this.measurementsCache[index];
  if (!item) return;

  const itemSize = this.itemSizeCache.get(item.key) ?? item.size;
  const delta = size - itemSize;

  if (delta !== 0) {
    // 현재 스크롤 위치보다 "위쪽"에 있는 아이템의 크기가 바뀌었다면
    if (item.start < this.getScrollOffset() + this.scrollAdjustments) {
      this._scrollToOffset(this.getScrollOffset(), {
        adjustments: (this.scrollAdjustments += delta), // ← delta만큼 스크롤 보정
        behavior: undefined,
      });
    }

    this.pendingMeasuredCacheIndexes.push(item.index);
    this.itemSizeCache = new Map(this.itemSizeCache.set(item.key, size));
    this.notify(false);
  }
};
```

- `item.start` 가 현재 scrollOffset 보다 작다 = "내 위쪽에 있는 아이템"
- 그 아이템의 크기가 `delta` 만큼 늘었다면 → 현재 스크롤도 `delta` 만큼 내려야 시각적 안정
- 이 계산을 모든 ResizeObserver 콜백마다 수행 → **유저는 점프를 못 느낌**

### Step 9 — `onChange` → React 리렌더

**코드** (`virtual-core/src/index.ts:487-489` + `react-virtual/src/index.tsx:40-47`)

```ts
// virtual-core
private notify = (sync: boolean) => {
  this.options.onChange?.(this, sync)
}

// react-virtual
onChange: (instance, sync) => {
  if (useFlushSync && sync) {
    flushSync(rerender)    // ← 스크롤은 sync = true
  } else {
    rerender()             // ← 그 외는 일반 리렌더
  }
}
```

**2-mode 배칭**:

| notify 종류    | sync    | 경로            | 언제                                                  |
| -------------- | ------- | --------------- | ----------------------------------------------------- |
| scroll, resize | `true`  | `flushSync`     | 사용자 스크롤 → 같은 프레임에 DOM 반영 (흰 공간 방지) |
| item 측정 변화 | `false` | 일반 `rerender` | 비동기 ResizeObserver 콜백 → React 배칭에 맡김        |

**왜 `flushSync`가 필요한가?**

React 18은 기본적으로 업데이트를 배칭해 다음 tick에 커밋합니다. 하지만 **스크롤 이벤트와 DOM 반영 사이에 한 프레임이라도 비면** 새로 등장해야 할 아이템 영역이 빈 채로 보입니다 = **흰 플래시**.

`flushSync` 는 이를 우회해 **스크롤 이벤트와 같은 동기 tick에서 DOM 커밋**을 강제합니다.

### Step 10 — 최종 소비: `getVirtualItems`

**코드** (`virtual-core/src/index.ts:1088-1106`)

```ts
getVirtualItems = memo(
  () => [this.getVirtualIndexes(), this.getMeasurements()],
  (indexes, measurements) => {
    const virtualItems: Array<VirtualItem> = []
    for (let k = 0, len = indexes.length; k < len; k++) {
      const i = indexes[k]!
      virtualItems.push(measurements[i]!)
    }
    return virtualItems
  },
  ...
)
```

- `getVirtualIndexes()` → `rangeExtractor({ startIndex, endIndex, overscan, count })` → 렌더할 인덱스 배열
- 각 인덱스에 해당하는 `measurements[i]` 를 꺼내 반환
- **memo 덕분에 deps가 같으면 같은 배열 참조 반환** → React가 리렌더를 해도 `map` 결과의 key 들이 동일 참조 → 자식 컴포넌트 memo 최적화 그대로 유지

---

## 🎯 이 레포에서 반드시 알아야 할 5가지 핵심 개념

> 스터디 진행 시 이 5개만 전달되면 나머지 1421줄은 스스로 읽힙니다.

### 개념 1. **헤드리스** — DOM을 만들지 않는 라이브러리

| Query                                             | Virtual                                  |
| ------------------------------------------------- | ---------------------------------------- |
| 데이터 fetching을 안 함 (queryFn은 사용자가 작성) | DOM 렌더링을 안 함 (JSX는 사용자가 작성) |
| 캐싱·재시도·구독 **계산만** 함                    | 측정·범위 계산·스크롤 보정 **계산만** 함 |

**→ TanStack의 공통 철학**: "우리는 **상태와 계산**을 책임진다. UI는 당신 몫."

### 개념 2. **`memo` 패턴** — Subscribable의 Virtual 버전

Query의 `Subscribable` 이 **pub/sub 뼈대**라면, Virtual의 `memo` 는 **dep 기반 파생 뼈대**입니다.

- `getMeasurements` → `calculateRange` → `getVirtualIndexes` → `getVirtualItems`
- 각 단계가 `memo` 로 감싸져 **deps가 같으면 이전 결과 재사용**
- 어떤 옵션 하나가 바뀌어도 **영향 받는 단계부터만** 재계산

### 개념 3. **`VirtualItem` — `{ key, index, start, end, size, lane }`**

Virtual의 모든 출력은 이 6개 필드로 귀결됩니다.

- `start`, `end` 는 **절대 offset (px)** — 컨테이너 상단 기준
- `translateY(item.start)` 로 절대 포지셔닝
- `lane` 은 그리드/메이슨리에서 열 번호 (기본 `lanes: 1` 이면 항상 0)

### 개념 4. **ResizeObserver × 2** — 컨테이너 vs 아이템

| Observer    | 용도                                   | 콜백                      |
| ----------- | -------------------------------------- | ------------------------- |
| 컨테이너 RO | 뷰포트 크기 변화 (창 리사이즈)         | `observeElementRect`      |
| 아이템 RO   | 개별 아이템 크기 변화 (이미지 로드 등) | `measureElement` ref 콜백 |

**→ 두 observer가 독립적으로 작동**하며, 각자 다른 memo 체인을 무효화합니다.

### 개념 5. **`scrollAdjustments` — 점프 방지의 핵심**

추정치(`estimateSize`)와 실측치의 차이가 누적되면 화면이 튑니다.
`scrollAdjustments` 는 **보이지 않는 상단 아이템의 delta를 모두 누적**해 스크롤 위치를 실시간으로 보정합니다.

→ 사용자는 스크롤이 튀는 걸 **거의** 못 느낍니다. (완벽히 없애려면 모든 아이템을 미리 측정해야 함)

---

## 전체 아키텍처 흐름

```
┌──────────────────────────────────────────────────────────────┐
│ User Component                                                │
│   const virtualizer = useVirtualizer({ count, estimateSize })│
│   virtualizer.getVirtualItems().map(item => <div ... />)     │
└────────────────────────┬─────────────────────────────────────┘
                         ▼
┌──────────────────────────────────────────────────────────────┐
│ react-virtual/ (프레임워크 어댑터, 101줄)                    │
│                                                               │
│   useVirtualizer ─ DOM observer들을 주입                     │
│        │                                                      │
│        ▼                                                      │
│   useVirtualizerBase                                         │
│   • new Virtualizer() (한 번만)                              │
│   • instance.setOptions() (렌더마다 옵션 동기화)             │
│   • useLayoutEffect(_didMount / _willUpdate)                 │
│   • onChange → flushSync(rerender) or rerender()             │
└────────────────────────┬─────────────────────────────────────┘
                         ▼
┌──────────────────────────────────────────────────────────────┐
│ virtual-core/ (프레임워크 중립 엔진, 1421줄, 1 클래스)       │
│                                                               │
│   Virtualizer                                                │
│    ├─ state                                                   │
│    │   ├─ scrollOffset, scrollRect, isScrolling              │
│    │   ├─ measurementsCache: VirtualItem[]                   │
│    │   ├─ itemSizeCache: Map<Key, number>                    │
│    │   ├─ laneAssignments: Map<index, lane>                  │
│    │   └─ elementsCache: Map<Key, Element>                   │
│    │                                                          │
│    ├─ memo 파이프라인                                         │
│    │   getMeasurementOptions → getMeasurements →             │
│    │   calculateRange → getVirtualIndexes → getVirtualItems  │
│    │                                                          │
│    ├─ DOM 구독                                                │
│    │   ├─ observeElementRect (ResizeObserver, 컨테이너)      │
│    │   ├─ observeElementOffset (scroll 이벤트)               │
│    │   └─ ResizeObserver (per-item, elementsCache 대상)      │
│    │                                                          │
│    └─ scrollToIndex / reconcileScroll (프로그래매틱 스크롤)  │
│                                                               │
│   utils.ts                                                    │
│    ├─ memo() ── dep 기반 메모이제이션                         │
│    ├─ debounce() ── 스크롤 종료 감지                          │
│    └─ approxEqual() ── 부동소수 비교 (1.01 tolerance)         │
└──────────────────────────────────────────────────────────────┘
```

### 핵심 파일 읽는 순서

| #   | 파일                                                          | 줄 수 | 역할                                                                      |
| --- | ------------------------------------------------------------- | ----- | ------------------------------------------------------------------------- |
| 1   | `virtual-core/src/utils.ts`                                   | 104   | `memo`, `debounce`, `approxEqual` (선행 학습 필수)                        |
| 2   | `react-virtual/src/index.tsx`                                 | 101   | React 연결 전체 (짧음)                                                    |
| 3   | `virtual-core/src/index.ts` 상단 (1-295)                      | -     | 타입 정의, `observeElementRect`, `observeElementOffset`, `measureElement` |
| 4   | `virtual-core/src/index.ts` 생성자/setOptions (370-485)       | -     | 상태 필드 + 기본값                                                        |
| 5   | `virtual-core/src/index.ts` `_willUpdate` (534-589)           | -     | DOM 연결 순간                                                             |
| 6   | `virtual-core/src/index.ts` memo 파이프라인 (726-1106)        | -     | **핵심**: getMeasurements → calculateRange → getVirtualItems              |
| 7   | `virtual-core/src/index.ts` `calculateRange` 함수 (1329-1421) | -     | 바이너리 서치 + lanes 로직                                                |
| 8   | `virtual-core/src/index.ts` scrollTo/reconcile (1205-1300)    | -     | 프로그래매틱 스크롤                                                       |

### Query vs Virtual — 같은 철학, 다른 구현

|                 | Query                                 | Virtual                                  |
| --------------- | ------------------------------------- | ---------------------------------------- |
| **뼈대 패턴**   | `Subscribable` (pub/sub)              | `memo` (dep 기반 파생)                   |
| **상태 소유자** | `Query` (캐시당 1개)                  | `Virtualizer` (훅당 1개)                 |
| **React 연결**  | `useSyncExternalStore` (tearing 방지) | `useReducer` + `flushSync` (스크롤 동기) |
| **배칭**        | `notifyManager` (microtask 큐)        | `onChange(sync)` 2-mode                  |
| **외부 이벤트** | focusManager, onlineManager           | ResizeObserver × 2, scroll               |
| **GC**          | `Removable` + gcTime                  | `cleanup()` + `unsubs`                   |
| **구독자 수**   | N Observers per Query                 | 1 Virtualizer per 훅 (공유 없음)         |
