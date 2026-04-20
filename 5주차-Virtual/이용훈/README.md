# TanStack Virtual - 대용량 리스트를 위한 가상화 유틸리티

## 들어가며

상품 목록, 채팅 메시지, 로그 뷰어처럼 수천 개의 아이템을 렌더링해야 하는 상황은 실무에서 드물지 않다.<br>
이런 경우 아무 처리 없이 전체를 렌더링하면 DOM 노드가 폭발적으로 늘어나고, 초기 렌더링이 느려지며, 스크롤 시 프레임 드롭이 발생한다.

TanStack Virtual은 이 문제를 **"보이는 것만 렌더링한다"** 는 단순한 원리로 해결하는 Headless 가상화 유틸리티다.

---

## 1. 이게 무엇인지

---

### 1-1. 가상화(Virtualization)란?

가상화는 수천~수만 개의 아이템 중 **현재 뷰포트에 보이는 것만 실제 DOM에 렌더링**하고, 나머지는 존재하지 않는 것처럼 처리하는 기법이다.<br> 스크롤 위치가 바뀌면 보이지 않는 아이템은 DOM에서 제거되고, 새로 보이게 될 아이템이 그 자리를 채운다.

예를 들어 10,000개의 상품 리스트가 있어도, 화면에 한 번에 보이는 건 많아야 20~30개다. 나머지 9,970개를 DOM에 유지할 이유가 없다.

| 구분 | 가상화 없음 | 가상화 적용 |
|---|---|---|
| DOM 노드 수 | 아이템 전체 (예: 10,000개) | 뷰포트 노출분 + 버퍼 (예: 30~50개) |
| 초기 렌더링 | 전체 생성 → 느림 | 일부만 생성 → 빠름 |
| 메모리 사용량 | 높음 | 낮음 |
| 스크롤 성능 | 버벅임 가능 | 일정하게 유지 |

---

### 1-2. TanStack Virtual이란?

TanStack Virtual은 JS/TS, React, Vue, Svelte, Solid, Lit, Angular에서 긴 목록을 가상화하기 위한 Headless UI 유틸리티다. <br>컴포넌트가 아니기 때문에 마크업이나 스타일을 제공하지 않는다.

즉, 테이블의 `<table>`, `<tr>`처럼 렌더링 자체는 개발자가 직접 작성하고, TanStack Virtual은 **"어떤 아이템을 지금 렌더링해야 하는가"** 에 대한 계산만 담당한다.

마크업과 스타일을 직접 작성해야 하지만, 그 덕분에 스타일, 디자인, 구현 방식에 대한 100% 제어권을 유지할 수 있다.

---

### 1-3. Virtualizer - 핵심 개념

Virtualizer 클래스는 TanStack Virtual의 핵심이다.<br> Virtualizer 인스턴스는 보통 프레임워크 어댑터가 생성해주며, 개발자는 Virtualizer를 직접 받아서 사용한다.

Virtualizer는 세로(기본값) 또는 가로 축으로 방향을 설정할 수 있으며, 두 축을 조합하면 세로, 가로, 그리드 형태의 가상화를 모두 구현할 수 있다.

React에서는 `useVirtualizer` 훅이 이 Virtualizer 인스턴스를 반환한다.

---

### 1-4. 지원 범위

| 항목 | 내용 |
|---|---|
| 지원 프레임워크 | React, Vue, Solid, Svelte, Lit, Angular |
| 방향 | 세로(vertical), 가로(horizontal), 그리드(두 방향 조합) |
| 아이템 크기 | 고정(fixed), 가변(variable), 동적 측정(dynamic) |
| 스크롤 대상 | 컨테이너 기준 또는 window 기준 |
| 패키지명 | `@tanstack/react-virtual`, `@tanstack/vue-virtual` 등 |

---

## 2. 어떻게 사용하는지

---

### 2-1. 설치

```bash
npm install @tanstack/react-virtual
```

---

### 2-2. 기본 구조 이해하기

TanStack Virtual을 사용한 렌더링은 항상 **세 가지 요소**로 구성된다.

```
[스크롤 컨테이너]  → overflow: auto, 고정 높이
  └── [내부 컨테이너]  → getTotalSize()로 전체 높이 확보 (스크롤바 크기 결정)
        └── [아이템들]  → position: absolute, 스크롤 위치 기반 offset 적용
```

이 구조가 핵심이다. 안쪽 컨테이너가 전체 높이를 차지해서 스크롤바가 올바르게 동작하게 하고, 아이템들은 `position: absolute`로 뷰포트 안에서만 위치를 잡는다.

---

### 2-3. Fixed Size - 고정 높이 리스트

모든 아이템의 높이가 동일할 때 사용하는 가장 기본적인 형태다.

```tsx
import { useVirtualizer } from '@tanstack/react-virtual'
import { useRef } from 'react'

function VirtualList({ items }: { items: string[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: items.length,           // 전체 아이템 수
    getScrollElement: () => parentRef.current,  // 스크롤 컨테이너
    estimateSize: () => 50,        // 아이템 높이 추정값 (px)
  })

  return (
    // 스크롤 컨테이너: 고정 높이 + overflow: auto 필수
    <div ref={parentRef} style={{ height: '500px', overflow: 'auto' }}>

      {/* 전체 스크롤 높이를 잡아주는 내부 컨테이너 */}
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>

        {/* 실제로 렌더링되는 아이템들 */}
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: virtualItem.size,
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            {items[virtualItem.index]}
          </div>
        ))}
      </div>
    </div>
  )
}
```

**각 옵션의 역할:**

| 옵션 | 역할 |
|---|---|
| `count` | 전체 아이템 수. 실제 데이터 길이와 일치해야 함 |
| `getScrollElement` | 스크롤 이벤트를 감지할 컨테이너 반환 |
| `estimateSize` | 아이템 크기 추정값. Fixed size에서는 실제 크기와 동일하게 설정 |

**각 반환값의 역할:**

| 반환값 | 역할 |
|---|---|
| `getTotalSize()` | 전체 아이템을 렌더링했을 때의 총 높이. 내부 컨테이너 height에 사용 |
| `getVirtualItems()` | 현재 뷰포트에 렌더링해야 할 아이템 목록 |
| `virtualItem.key` | React key로 사용할 고유값 |
| `virtualItem.index` | 원래 데이터 배열의 인덱스 |
| `virtualItem.size` | 아이템의 크기 (px) |
| `virtualItem.start` | 스크롤 컨테이너 기준 아이템의 시작 위치 (px) |

---

### 2-4. Dynamic Size - 가변 높이 리스트

아이템마다 높이가 다를 때 사용한다. `measureElement`를 통해 실제 렌더링된 DOM 크기를 측정한다.

```tsx
const virtualizer = useVirtualizer({
  count: items.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 100,  // 초기 추정값. 실제 측정 후 교체됨
})

// 아이템에 ref 콜백으로 실제 크기 측정을 트리거
{virtualizer.getVirtualItems().map((virtualItem) => (
  <div
    key={virtualItem.key}
    ref={virtualizer.measureElement}    // ref 콜백 연결
    data-index={virtualItem.index}      // index 전달 필수
    style={{
      position: 'absolute',
      top: 0,
      left: 0,
      width: '100%',
      transform: `translateY(${virtualItem.start}px)`,
      // height는 지정하지 않음. 콘텐츠 높이에 따라 자동 결정
    }}
  >
    {items[virtualItem.index]}
  </div>
))}
```

> 공식 문서 권장사항: Dynamic size를 사용할 때는 `estimateSize`를 실제 크기보다 **크게** 설정하는 것이 권장된다. 추정값이 실제보다 작으면 초기 레이아웃에서 스크롤 점프가 발생할 수 있다. ([Virtualizer API](https://tanstack.com/virtual/latest/docs/api/virtualizer))

---

### 2-5. Horizontal - 가로 리스트

```tsx
const virtualizer = useVirtualizer({
  horizontal: true,           // 가로 방향으로 전환
  count: items.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 120,    // 아이템 너비
})

// 스타일도 left / width 기반으로 변경
<div
  style={{
    position: 'absolute',
    top: 0,
    left: 0,
    height: '100%',
    width: virtualItem.size,
    transform: `translateX(${virtualItem.start}px)`,  // translateY → translateX
  }}
/>
```

---

### 2-6. 주요 옵션 정리

| 옵션 | 기본값 | 설명 |
|---|---|---|
| `count` | 필수 | 전체 아이템 수 |
| `getScrollElement` | 필수 | 스크롤 컨테이너 반환 함수 |
| `estimateSize` | 필수 | 아이템 크기 추정 함수 |
| `overscan` | `1` | 뷰포트 밖 추가 렌더링 아이템 수 |
| `horizontal` | `false` | 가로 방향 가상화 여부 |
| `paddingStart` | `0` | 리스트 시작 부분 패딩 (px) |
| `paddingEnd` | `0` | 리스트 끝 부분 패딩 (px) |
| `measureElement` | - | 실제 DOM 크기 측정 함수 (Dynamic size) |

> **overscan 기본값 주의:** 공식 문서 기준 overscan 기본값은 `1`이다. 흔히 `3`으로 알려져 있는 경우가 있는데, v3 API 문서에는 `1`로 명시되어 있다. ([Virtualizer API](https://tanstack.com/virtual/latest/docs/api/virtualizer))

---

## 3. 왜 써야하는지

---

### 3-1. DOM은 생각보다 비싸다

브라우저는 DOM 노드가 늘어날수록 레이아웃 계산, 스타일 적용, 페인팅 비용이 증가한다. <br>1,000개의 `<div>`를 한 번에 렌더링하면 그 자체로 수백 ms의 블로킹이 발생할 수 있다. <br>가상화는 이 문제의 근본 원인인 "불필요한 DOM 노드"를 제거한다.

---

### 3-2. Headless - 스타일에 자유롭다

많은 가상화 라이브러리는 컴포넌트를 통째로 제공하기 때문에 기존 디자인 시스템과 충돌이 발생하거나 커스터마이징이 어렵다. <br>TanStack Virtual은 **로직만 제공**하고 마크업과 스타일은 개발자가 완전히 제어한다. Tailwind, CSS Modules, styled-components, 사내 디자인 시스템 어디든 자연스럽게 붙는다.

---

### 3-3. TanStack 생태계와 자연스럽게 결합된다

TanStack Virtual은 독립적으로도 사용할 수 있지만, TanStack Query와 조합하면 무한 스크롤 같은 패턴을 깔끔하게 구현할 수 있다.

- **TanStack Query**: 현재 페이지 데이터 패칭 + 다음 페이지 프리페칭
- **TanStack Virtual**: 스크롤 위치 추적 → 마지막 아이템 진입 시 Query에 다음 페이지 요청

각 레이어의 책임이 명확하게 분리된다.

---

### 3-4. 언제 TanStack Virtual이 적합하지 않은가

- **아이템 수가 적을 때 (100개 미만):** 가상화 설정 코드가 오히려 오버엔지니어링이다. 단순한 `map` 렌더링이 더 빠르고 읽기 쉽다.
- **아이템 크기가 복잡하고 렌더링 비용이 낮을 때:** 가상화는 DOM 노드 수를 줄이는 것이지, 각 아이템의 렌더링 자체를 빠르게 해주지는 않는다. <br>각 아이템 내부가 복잡하다면 `React.memo`나 코드 분할 같은 다른 최적화를 먼저 고려해야 한다.

---

## 정리

TanStack Virtual은 가상화의 핵심인 **"계산"** 만 담당하고 나머지는 개발자에게 맡긴다. <br>덕분에 스타일과 마크업에 대한 완전한 제어권을 유지하면서, 대용량 리스트의 성능 문제를 근본적으로 해결할 수 있다.

| 키워드 | 설명 |
|---|---|
| 가상화 | 보이는 것만 DOM에 렌더링, 나머지는 교체 |
| Headless | UI는 개발자가, 계산 로직은 TanStack Virtual이 |
| `useVirtualizer` | React 어댑터 훅. count / getScrollElement / estimateSize 필수 |
| `getTotalSize()` | 전체 스크롤 높이 반환, 내부 컨테이너 height에 사용 |
| `getVirtualItems()` | 현재 렌더링할 아이템 목록 반환 |
| `overscan` | 뷰포트 밖 미리 렌더링 수, 기본값 1 |
| `measureElement` | Dynamic size에서 실제 DOM 크기 측정 |

---

## 참고 링크

- [TanStack Virtual 공식 문서](https://tanstack.com/virtual/latest/docs/introduction)
- [Virtualizer API](https://tanstack.com/virtual/latest/docs/api/virtualizer)
- [React Virtual 어댑터](https://tanstack.com/virtual/v3/docs/framework/react/react-virtual)
- [예제 - Dynamic size](https://tanstack.com/virtual/v3/docs/framework/react/examples/dynamic)
- [예제 - Infinite Scroll](https://tanstack.com/virtual/v3/docs/framework/react/examples/infinite-scroll)
- [GitHub - TanStack/virtual](https://github.com/TanStack/virtual)