## 🛠️ TanStack이란?

**설립자:** Tanner Linsley

`공식문서 첫 페이지 문구`

> High-quality open-source software for web developers.

= 개발자를 위한 좋은 도구다.

> Headless, type-safe, & powerful utilities for Web Applications, Routing, State Management, Data Visualization, Datagrids/Tables, and more.

= 다양한 기능을 지원한다.

<br/>

### 탄생 배경

Tanner Linsley가 공동 창업한 nozzle.io(구글 검색 결과 기반 대용량 데이터 분석 툴)에서
대용량 데이터를 표시할 테이블이 필요했는데, Angular에서 React로 전환하는 시점에
마땅한 React 테이블 라이브러리가 없어 직접 만든 게 시작이었다.

처음부터 오픈소스를 목적으로 만든 게 아니었다.
실제 프로덕트에서 겪은 문제를 하나씩 해결하다 보니 라이브러리가 됐고,
그것들이 쌓이면서 지금의 TanStack 생태계가 됐다.

```
2016  Table   (테이블이 없어서)
2019  Query   (Redux로 서버 state 관리가 불편해서)
2020~ Virtual (대용량 리스트 성능 문제)
2021~ Router  (React Router search params 한계)
```

<br/>

### 설계 철학

TanStack의 철학은 [Ethos](https://tanstack.com/ethos)와 [Tenets](https://tanstack.com/tenets)에 명시되어 있다.

**1. Framework-agnostic**
모든 라이브러리는 framework에 종속되지 않는 core에서 시작한다.
React, Vue, Solid, Angular 등 프레임워크별 어댑터는 그 위에 얹히는 선택지일 뿐,
기반이 되어선 안 된다.
→ React가 지배적이어도 Vue/Solid 팀도 같은 문제를 겪는다. core를 공유하면 유지보수 비용이 줄고 생태계도 넓어진다.

**2. Headless**
로직과 UI를 분리한다. 상태, 연산, 이벤트 처리는 라이브러리가 담당하고
마크업과 스타일은 개발자가 100% 제어한다.
→ 스타일 의견을 라이브러리가 가져가는 순간 디자인 시스템이 다른 팀은 못 쓴다. 로직만 제공해야 어디서든 붙일 수 있다.

**3. Type-safe**
TypeScript를 단순히 사용하는 게 아니라, 타입이 API 전체를 관통하도록 설계한다.
개발자가 직접 타입을 선언하지 않아도 함수 반환값에서 읽어서 자동으로 흘려보내준다.

**4. Composable**
거대한 "all-in-one" 프레임워크가 아닌 작고 조합 가능한 빌딩 블록을 제공한다.
TanStack 라이브러리들 사이에도 강한 결합은 없고, 필요한 것만 골라서 점진적으로 도입할 수 있다.

<br/><br/>

## 📊 Table

강력한 테이블과 데이터그리드 구축을 위한 헤드리스 UI

- Headless UI?
  - 클릭, 키보드 동작, 접근성은 라이브러리가 처리하고, 스타일은 개발자가 직접 입힌다.
  - Radix/ui, shadcn/ui 도 헤드리스 기반이다. (shadcn은 Radix + Tailwind 스타일 초안)
- 그럼 탠스택 테이블 굳이 왜씀? 그냥 shadcn의 테이블 컴포넌트 쓰지?
  - 둘이 하는 일 자체가 다르다.
  - shadcn Table = 어떻게 보이는가 (스타일) / TanStack Table = 데이터를 어떻게 처리하는가 (로직)
  - 단순히 데이터 보여주기만 하면 shadcn Table로 충분하고, 정렬/필터/페이지네이션/컬럼 visibility 같은 게 필요해지는 순간 TanStack Table을 붙이는 것

<br/>

### 무엇이 편하고 왜 써야하는지

- 데이터 로직과 UI가 분리되어 있어서, 기능이 늘어나도 렌더링 코드는 안 바뀐다.
- 정렬 / 필터 / 페이지네이션 같은 기능도 옵션 한 줄씩 추가하는 방식이라 관리가 편하다.
- row가 많아지는 경우엔 TanStack Virtual과 연동해서 가상화같은 최적화에도 좋다.

<br/>

### 사용 예시

TanStack 없이: 직접 구현한다면...

- 정렬: onClick + useState + [...data].sort() 직접 작성
- 필터: input onChange + data.filter() 직접 작성
- 페이지네이션: 현재 페이지 state + slice() 직접 계산
- 컬럼 visibility: 어떤 컬럼 보여줄지 Set으로 직접 관리
- → 100줄짜리 로직을 컴포넌트 안에 직접 다 쑤셔넣게 됨

<br/>

**공식문서 예시코드**

```ts
import {
  useReactTable,
  getCoreRowModel,
  flexRender
} from '@tanstack/react-table'

const data = [{ id: 1, name: 'Ada' }]
const columns = [{ accessorKey: 'name', header: 'Name' }]

export default function SimpleTable() {
  const table = useReactTable({
    data,
    columns,
    getCoreRowModel: getCoreRowModel(),
    // ✅ 정렬 추가하려면? getSortedRowModel() 한 줄
    // ✅ 필터 추가하려면? getFilteredRowModel() 한 줄
    // ✅ 페이지네이션? getPaginationRowModel() 한 줄
    // → 기능을 갈아끼우는 구조 (플러그인 패턴)
  })

  return (
    <table>
      <thead>
        {table.getHeaderGroups().map((hg) => (
          // ✅ getHeaderGroups(): 컬럼 그룹핑, 다중 헤더 행 자동 처리
          // ❌ 직접 구현하면 중첩 컬럼 구조를 손으로 재귀 렌더링해야 함
          <tr key={hg.id}>
            {hg.headers.map((header) => (
              <th key={header.id}>
                // ✅ flexRender(): 컬럼 정의가 string이든 ReactNode든 함수든 알아서 렌더링해줌
                // ❌ 직접 하면 typeof로 분기 처리 직접 해야 함
                {flexRender(header.column.columnDef.header, header.getContext())}
              </th>
            ))}
          </tr>
        ))}
      </thead>
      <tbody>
        {table.getRowModel().rows.map((row) => (
          // ✅ getRowModel(): 현재 적용된 정렬/필터/페이지네이션이 반영된 rows를 돌려줌. 로직 바꿔도 여기 코드는 안 건드려도 됨
          // ❌ 직접 구현하면 sortedData, filteredData, pagedData를 각각 useMemo로 파이프라인 연결해야 함
          <tr key={row.id}>
            {row.getVisibleCells().map((cell) => (
              // ✅ getVisibleCells(): 컬럼 visibility 상태가 자동 반영됨
              // ❌ 직접 하면 hiddenColumns Set을 매번 체크해야 함
              <td key={cell.id}>
                {flexRender(cell.column.columnDef.cell, cell.getContext())}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  )
}
```

<br/><br/>

## 📦 Query

Redux가 server state를 제대로 다루지 못한다는 문제에서 시작

`공식문서`

> TanStack Query(이전 명칭: React Query)는 웹 애플리케이션에서 누락된 데이터 가져오기 라이브러리로 자주 설명되지만, 좀 더 기술적으로 웹 애플리케이션에서 서버 상태 가져오기, 캐싱, 동기화 및 업데이트가 매우 수월하게 만들어줍니다.

서버에서 오는 데이터인데 `isLoading`, `error`까지 클라이언트 상태처럼 직접 관리하다 보면 그게 서버 상태인지 클라이언트 상태인지 모호한 채로 쌓인다. TanStack Query는 서버 상태를 전담하는 레이어를 분리해서 이 모호함을 끊는다.

<br/>

### 어떻게 사용하는지

`useQuery`에는 두 가지 핵심 옵션이 들어간다.

- **`queryKey`** — 캐시의 식별자. `['todos']`처럼 배열로 넘기고, 값이 바뀌면 자동으로 재요청한다.
- **`queryFn`** — 실제 데이터를 가져오는 비동기 함수. Promise를 반환하면 된다.

반환값으로 `data`, `isPending`, `error`를 바로 꺼내 쓸 수 있어서 로딩/에러 분기를 별도 상태 없이 처리할 수 있다.

<br/>

### 사용 예시

**공식문서 예시코드**

```ts
// ❌ 직접 구현
// useState로 data, isPending, error 각각 관리
// useEffect 안에서 fetch → then → catch → finally

// ✅ TanStack Query
import { useQuery } from '@tanstack/react-query'

function Todos() {
  const { data, isPending, error } = useQuery({
    queryKey: ['todos'],
    queryFn: () => fetch('/api/todos').then(r => r.json()),
  })

  if (isPending) return <span>Loading...</span>
  if (error) return <span>Oops!</span>

  return <ul>{data.map(t => <li key={t.id}>{t.title}</li>)}</ul>
}

export default Todos
```

### 왜 써야 하는지

단순히 코드가 짧아지는 게 아니라, **"서버 상태는 TanStack Query가 관리한다"는 역할 분리**가 핵심이다. Zustand 같은 전역상태관리는 클라이언트 상태에 집중하고, 서버 데이터의 신선도·동기화·재시도는 TanStack Query에 맡기는 구조가 지금 프론트엔드 생태계의 표준에 가까워졌다.

<br/><br/>

## ⚡Virtual

리스트 아이템 10,000개를 `.map()`으로 전부 렌더링하면 DOM이 감당하지 못한다는 문제에서 시작

`공식문서`

> TanStack Virtual은 JS/TS, React, Vue, Svelte 등에서 긴 목록을 가상화하는 헤드리스 UI 유틸리티입니다. 컴포넌트가 아니기 때문에 마크업이나 스타일을 직접 제공하지 않습니다.

아이템 10,000개가 있어도 화면에 보이는 건 20개 남짓이다. 나머지는 전부 DOM에 올라와 있을 필요가 없다. TanStack Virtual은 스크롤 위치를 기준으로 현재 뷰포트에 보이는 아이템만 계산해서 렌더링하고, 나머지는 DOM에서 제거한다.

<br/>

### 어떻게 사용하는지

`useVirtualizer`에 세 가지 핵심 옵션이 들어간다.

- **`count`** — 전체 아이템 수. 실제 렌더링 개수가 아닌 논리적 총 개수다.
- **`getScrollElement`** — 스크롤 컨테이너를 반환하는 함수. `ref`로 연결한다.
- **`estimateSize`** — 아이템 하나의 예상 높이(px). 실제 높이를 측정하기 전 초기값으로 쓰인다.

반환값인 `getVirtualItems()`로 현재 뷰포트에 보여야 할 아이템 목록만 꺼내 쓰고, `getTotalSize()`로 전체 스크롤 높이를 유지해 스크롤바가 자연스럽게 동작하게 한다.

<br/>

### 사용 예시

**공식문서 예시코드**

```ts
import { useVirtualizer } from '@tanstack/react-virtual'

function App() {
  const parentRef = React.useRef(null)

  const rowVirtualizer = useVirtualizer({
    count: 10000,                              // 전체 아이템 수
    getScrollElement: () => parentRef.current, // 스크롤 컨테이너 연결
    estimateSize: () => 35,                    // 아이템 예상 높이(px)
  })

  return (
    // 스크롤 컨테이너
    <div ref={parentRef} style={{ height: `400px`, overflow: 'auto' }}>

      {/* 전체 높이 유지 → 스크롤바가 10000개짜리인 것처럼 동작 */}
      <div style={{ height: `${rowVirtualizer.getTotalSize()}px`, position: 'relative' }}>

        {/* 뷰포트에 보이는 아이템만 렌더링 */}
        {rowVirtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              // 스크롤 위치에 따라 정확한 y좌표로 이동
              transform: `translateY(${virtualItem.start}px)`,
              height: `${virtualItem.size}px`,
            }}
          >
            ... {/* 실제 아이템 내용 */}
          </div>
        ))}
      </div>
    </div>
  )
}
```

### 왜 써야 하는지

DOM 노드 수를 고정된 소수로 유지하기 때문에, 데이터가 아무리 많아도 렌더링 비용이 선형으로 늘어나지 않는다. **헤드리스** 설계라서 마크업과 스타일은 100% 직접 제어하고, 라이브러리는 "지금 뭐가 보여야 하는가"만 계산해준다. 무한 스크롤, 채팅 피드, 대용량 테이블처럼 데이터 크기가 불확실한 상황에서 선택지가 된다.

<br/><br/>

### 🗺️ Router

React Router가 TypeScript를 제대로 지원하지 못하고, search params를 그냥 문자열로 다루는 문제에서 시작

`공식문서`

> TanStack Router는 React와 Solid 애플리케이션을 위한 라우터입니다. 경로, 경로 파라미터, 검색 파라미터, 컨텍스트 등 모든 라우트 설정을 코드 전반에 걸쳐 완전히 인식합니다.

URL에 `?page=2&filter=active`를 쓰면 대부분의 라우터는 그냥 문자열로 넘긴다. 파싱도, 타입도 직접 챙겨야 한다. TanStack Router는 search params를 first-class citizen으로 다뤄서, 타입 추론과 직렬화까지 라우터 레벨에서 해결한다.

<br/>

### 어떻게 사용하는지

라우트를 `createFileRoute`로 정의하고, `component`에 렌더링할 컴포넌트를 연결한다.

- **`createFileRoute`** — 파일 기반 라우트 정의. 경로는 파일 시스템 구조에서 자동 추론된다.
- **`loader`** — 라우트 진입 전 데이터를 미리 fetch. TanStack Query와 조합해서 쓰는 게 일반적이다.
- **`useSearch` / `useParams`** — search params와 path params를 타입 안전하게 꺼내 쓰는 훅.

<br/>

### 사용 예시

**공식문서 예시코드**
```ts
import { createRootRoute, createRoute, createRouter, RouterProvider } from '@tanstack/react-router'

// 최상위 레이아웃 라우트 (모든 라우트의 부모)
const rootRoute = createRootRoute()

// 개별 라우트 정의
const indexRoute = createRoute({
  getParentRoute: () => rootRoute, // 부모 라우트 연결
  path: '/',
  component: () => <div>Hello World</div>,
})

// 라우트 트리 조립
const routeTree = rootRoute.addChildren([indexRoute])

// 라우터 인스턴스 생성
const router = createRouter({ routeTree })

export default function App() {
  return <RouterProvider router={router} /> // 라우터 주입
}
```

### 왜 써야 하는지

단순히 타입이 생기는 게 아니라, **"라우터가 URL 전체를 상태로 이해한다"는 패러다임 전환**이 핵심이다. search params가 타입 안전하게 관리되면 필터, 페이지네이션, 탭 상태 같은 것들을 전역 상태 없이 URL만으로 깔끔하게 처리할 수 있다. React Router와 달리 런타임이 아닌 컴파일 타임에 잘못된 링크를 잡아준다.

<br/><br/>

## 🤔 최종 내 느낌

**"우리는 로직 문제만 풀고, 나머지 결정은 너희가 해"**

- Table → 정렬/필터/페이지네이션 로직만 품. 마크업은 너가 해
- Virtual → 뭐가 보여야 하는지 계산만 함. 마크업은 너가 해
- Query → 서버 상태 관리 로직만 품. 어떻게 보여줄지는 너가 해
- Router → URL 파싱/타입 추론 로직만 품. 어떤 구조로 쓸지는 너가 해

