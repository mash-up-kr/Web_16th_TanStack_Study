# TanStack Table의 Headless

## 0️⃣ Headless란?

> "로직과 상태는 라이브러리가. UI는 내가."

`<table>` 태그 짜는 건 어렵지 않다.
진짜 어려운 건 **정렬, 필터, 페이지네이션 같은 로직**이다.

TanStack Table은 그 어려운 부분만 해결해주고, 마크업과 스타일은 전혀 건드리지 않는다.
그래서 `<table>` 태그를 쓰든, `<div>`로 만들든, shadcn/ui 컴포넌트를 쓰든 — **전부 된다.**

### Headless 테이블 라이브러리는 거의 없다

테이블 라이브러리는 크게 두 종류다.

| 종류              | 예시                  | 특징                                                                  |
| ----------------- | --------------------- | --------------------------------------------------------------------- |
| **컴포넌트 기반** | AG Grid, MUI DataGrid | 마크업 + 스타일 + 로직 전부 제공. 쓰기는 쉬운데 커스터마이징이 어렵다 |
| **Headless**      | **TanStack Table**    | 로직만 제공. UI는 내 마음대로                                         |

Headless 테이블 라이브러리는 TanStack Table이 사실상 유일한 선택지다.
그만큼 이 접근 방식 자체가 희귀하고, 그래서 강력하다.

<br/>

## 1️⃣ Before / After — 정렬 하나 만들어보기

### ❌ Before: 직접 구현

```tsx
function Table({ data }) {
  const [sortKey, setSortKey] = useState<string | null>(null)
  const [sortDir, setSortDir] = useState<'asc' | 'desc'>('asc')

  // 정렬 로직 직접 구현
  const sorted = [...data].sort((a, b) => {
    if (!sortKey) return 0
    if (a[sortKey] < b[sortKey]) return sortDir === 'asc' ? -1 : 1
    if (a[sortKey] > b[sortKey]) return sortDir === 'asc' ? 1 : -1
    return 0
  })

  const handleSort = (key: string) => {
    if (sortKey === key) setSortDir(d => d === 'asc' ? 'desc' : 'asc')
    else { setSortKey(key); setSortDir('asc') }
  }

  return ( /* ... 렌더링 */ )
}
```

정렬 하나에 state 2개, 정렬 함수, 핸들러 함수.
여기에 필터·페이지네이션까지 붙으면 로직이 얽히기 시작한다.

---

### ✅ After: TanStack Table 사용

```tsx
// 공식 문서 basic example 기반
function Table({ data }) {
  const [sorting, setSorting] = useState<SortingState>([])

  const table = useReactTable({
    data,
    columns,
    state: { sorting },
    onSortingChange: setSorting,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(), // ← 이 한 줄로 정렬 완성
  })

  return ( /* ... 렌더링 */ )
}
```

정렬 로직을 직접 짜지 않았다. `getSortedRowModel()` 한 줄이 전부다.

필터를 추가하고 싶으면? `getFilteredRowModel()` 한 줄.
페이지네이션은? `getPaginationRowModel()` 한 줄.
**기능이 늘어나도 로직이 얽히지 않는다.**

<br/>

## 2️⃣ TanStack Table Headless의 핵심 — DOM을 만들지 않는다

After 코드를 다시 보면 뭔가 이상하다.

```tsx
const table = useReactTable({ data, columns, ... })

table.getHeaderGroups()   // 헤더 정보를 준다
table.getRowModel().rows  // 정렬 적용된 rows를 준다
```

`table`이라는 이름인데, HTML 태그 얘기를 전혀 안 한다.
렌더링 결과가 없다. 화면에 아무것도 안 그린다.

**`useReactTable`이 반환하는 `table`은 `<table>` DOM이 아니다.**
정렬·필터 상태와 그걸 다루는 메서드만 담긴 **순수 JS 객체**다.

이게 TanStack Table Headless의 핵심이다.
AG Grid 같은 컴포넌트 기반 라이브러리는 `<div class="ag-root">...` 같은 마크업을 직접 뱉는다. 그래서 디자인을 바꾸려면 라이브러리가 허용하는 범위 안에서만 가능하다.

TanStack Table은 마크업을 뱉지 않는다.
`table.getRowModel().rows`는 그냥 데이터 배열이다.
그걸 `<tr>`로 그리든, `<div>`로 그리든, shadcn `<TableRow>`로 그리든 — **전적으로 내가 결정한다.**

---

### 그게 가능한 이유 — `table`이 JS 객체인 이유

`useReactTable`은 내부적으로 `createTable()`을 호출한다.

```ts
// packages/table-core/src/core/table.ts

export function createTable<TData>(options) {
  // feature 목록 구성 (정렬, 필터, 페이지네이션...)
  const _features = [...builtInFeatures, ...(options._features ?? [])];

  let table = { _features } as Table<TData>;

  // 각 feature가 table 객체에 메서드를 주입
  _features.forEach((feature) => {
    feature.createTable?.(table);
    // table.getSortedRows, table.nextPage 같은 메서드가 하나씩 붙음
  });

  return table; // DOM 없음. 순수 JS 객체 반환
}
```

`createTable`은 DOM을 만들지 않는다.
렌더링 코드가 없다. `document.createElement`도, JSX도 없다.
그냥 메서드 묶음을 반환할 뿐이다.

이건 TanStack Query나 Router도 같은 패턴이긴 하다.
차이는 **Table은 그 결과물이 "화면에 보이는 테이블"이어야 한다는 것**.
Query는 데이터를 가져오는 거라 UI 결과물 자체가 없지만, Table은 "표"를 만들어야 하는데도 DOM을 안 만든다. 그게 Headless가 의미 있는 지점이다.

---

### flexRender — 렌더링은 어댑터가 담당

`createTable`이 DOM을 안 만든다고 했는데, 그러면 컬럼 정의의 `header`, `cell`은 어떻게 그려지나?

컬럼 정의에 문자열이 올 수도 있고, JSX 컴포넌트가 올 수도 있다.

```ts
{ header: '이름' }                                          // 문자열
{ header: ({ column }) => <SortButton column={column} /> }  // 컴포넌트
```

코어는 이걸 어떻게 그려야 하는지 모른다. 알 필요도 없다.
**렌더링은 어댑터(`@tanstack/react-table`)가 `flexRender`로 처리한다.**

```ts
// packages/react-table/src/index.tsx

export function flexRender<TProps extends object>(
  Comp: React.ReactNode | React.ComponentType<TProps>,
  props: TProps
): React.ReactNode {
  if (!Comp) return null
  if (isReactComponent(Comp)) return <Comp {...props} />  // 컴포넌트면 렌더
  return Comp                                              // 문자열 등은 그대로 반환
}
```

React 코드는 이 파일에만 있다.
`table-core`에는 `import React`가 한 줄도 없다.

렌더링 책임이 어댑터에만 격리되어 있기 때문에,
**같은 `createTable` 위에 Vue 어댑터, Svelte 어댑터가 올라갈 수 있고**,
어떤 마크업을 쓸지도 전적으로 개발자 몫이 된다.

---

### TableFeature — 기능도 DOM 없이 주입된다

정렬, 필터, 페이지네이션은 `TableFeature` 인터페이스로 구현된다.
각 feature는 table 객체에 **메서드만 주입**한다. DOM은 건드리지 않는다.

```ts
// packages/table-core/src/types.ts

export interface TableFeature<TData> {
  getInitialState?: (state?) => Partial<TableState>; // 초기 상태 기여
  getDefaultOptions?: (table) => Partial<TableOptions>; // 기본 옵션 기여
  createTable?: (table) => void; // table에 메서드 주입 (DOM 없음)
  createRow?: (row, table) => void;
  createCell?: (cell, column, row, table) => void;
}
```

페이지네이션 feature를 예로 보면:

```ts
// packages/table-core/src/features/RowPagination.ts

export const RowPagination: TableFeature = {
  getInitialState: (state) => ({
    ...state,
    pagination: { pageIndex: 0, pageSize: 10 },
  }),

  createTable: (table) => {
    // table.nextPage(), table.previousPage(), table.getPageCount() 메서드만 주입
    // 페이지네이션 UI(버튼, 숫자 등)는 전혀 만들지 않는다
  },
};
```

페이지네이션 버튼 UI가 없다. 어떻게 생겼는지 모른다. 관심도 없다.
`table.nextPage()`를 호출하면 상태만 바꾼다.
그 상태를 보고 버튼을 어떻게 그릴지는 개발자 몫이다.

**`getPaginationRowModel()`을 import하지 않으면 번들에 포함조차 되지 않는다.**

<br/>

## 3️⃣ 그래서 shadcn/ui랑 이렇게 쓴다

Headless의 진짜 장점이 여기서 나온다.

shadcn/ui의 `<Table>`은 스타일만 담당하는 컴포넌트다.
TanStack Table이 로직을 계산하면, shadcn이 예쁘게 그린다.
둘 다 "UI를 강요하지 않는" 구조라 조합이 자연스럽다.

```tsx
// shadcn/ui 컴포넌트 (스타일 담당)
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";

// TanStack Table (로직 담당)
import {
  useReactTable,
  flexRender,
  getCoreRowModel,
} from "@tanstack/react-table";

function DataTable({ data, columns }) {
  const table = useReactTable({
    data,
    columns,
    getCoreRowModel: getCoreRowModel(),
    // 정렬이 필요하면 getSortedRowModel() 추가, 필터는 getFilteredRowModel() 추가...
  });

  return (
    <Table>
      {" "}
      {/* shadcn 컴포넌트 */}
      <TableHeader>
        {table.getHeaderGroups().map((headerGroup) => (
          <TableRow key={headerGroup.id}>
            {headerGroup.headers.map((header) => (
              <TableHead key={header.id}>
                {flexRender(
                  header.column.columnDef.header,
                  header.getContext(),
                )}
              </TableHead>
            ))}
          </TableRow>
        ))}
      </TableHeader>
      <TableBody>
        {table.getRowModel().rows.map((row) => (
          <TableRow key={row.id}>
            {row.getVisibleCells().map((cell) => (
              <TableCell key={cell.id}>
                {flexRender(cell.column.columnDef.cell, cell.getContext())}
              </TableCell>
            ))}
          </TableRow>
        ))}
      </TableBody>
    </Table>
  );
}
```

`<table>` 태그 대신 `<Table>` 컴포넌트로만 바꿨다.
**나머지 코드는 전혀 달라지지 않는다.**

### 조합의 장점

| 항목               | 설명                                                 |
| ------------------ | ---------------------------------------------------- |
| 디자인 시스템 유지 | 팀 shadcn 스타일 그대로, 정렬/필터만 추가            |
| 로직과 UI 독립     | TanStack Table 버전 올려도 UI 코드 안 건드림         |
| 점진적 도입        | 기존 `<Table>` 컴포넌트에 TanStack Table만 얹으면 됨 |

<br/>

### **🤔 최종 내 느낌**

- Headless는 신이야!
