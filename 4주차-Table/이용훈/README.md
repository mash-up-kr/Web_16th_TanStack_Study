# TanStack Table — 서버사이드 테이블 설계 관점에서 보기

> **TanStack Table v8 / @tanstack/react-table 기준**  
> 공식 문서: https://tanstack.com/table/v8/docs/introduction

---

## 들어가며

이번 주 TanStack Table을 공부하면서, 단순히 "어떻게 렌더링하는가"보다 **서버사이드 환경에서 어떤 설계 철학을 갖고 있는가**에 집중해보고 싶었다. 백오피스, 주문/상품 리스트, 로그 뷰어처럼 **수만 건 이상의 데이터를 서버에서 페이지 단위로 받아오는 테이블**은 클라이언트 내장 정렬·필터·페이지네이션이 사실상 무의미하다. 상태만 들고 있고, 실제 계산은 서버가 하게 해야 한다.

그 관점에서 TanStack Table을 들여다보니, 처음부터 그 시나리오를 1급 시나리오(first-class scenario)로 설계해두었다는 인상을 받았다. 아래에 그 근거와 패턴을 정리한다.

---

# 1. 이게 무엇인지 (WHAT)

## 1-1. 서버사이드 테이블 관점에서 본 TanStack Table

TanStack Table(구 React Table)은 **Headless 테이블 엔진**이다. 공식 문서([Introduction](https://tanstack.com/table/v8/docs/introduction))는 이를 "UI를 렌더링하지 않는 테이블 유틸리티"로 정의한다. 즉, `<table>`, `<tr>`, `<td>` 마크업을 직접 작성하고, TanStack Table은 그 안에 담길 **상태(state)와 파생 값(derived values)** 만 제공한다.

서버사이드 테이블에서 요구되는 특성을 나열해보면 다음과 같다.

- **정렬·필터·페이지네이션 상태가 외부로 노출되어야 한다.** 이 상태들이 API 쿼리 파라미터로 변환되어야 하기 때문이다.
- **클라이언트 계산 로직(row model)을 비활성화할 수 있어야 한다.** 서버가 이미 정렬·필터링된 결과를 내려주는데, 클라이언트가 다시 계산하면 결과가 꼬인다.
- **전체 row 수(rowCount)를 외부에서 주입할 수 있어야 한다.** 페이지 개수를 계산하려면 서버가 내려준 총 건수가 필요하다.
- **상태 변경을 감지해서 API를 재호출할 수 있어야 한다.** 정렬 방향이 바뀌거나 필터 값이 변경되면 즉시 새 데이터를 요청해야 한다.

TanStack Table은 이 네 가지 요구사항을 모두 `manual*` 옵션과 controlled state 패턴으로 충족시킨다. 그리고 이것이 의견이 아닌 공식 문서에 명시된 설계 의도다.

## 1-2. Headless 구조가 서버사이드와 어떻게 연결되는가

Headless라는 것은 단순히 "UI가 없다"는 말이 아니다. **상태와 파생 로직을 완전히 외부에 위임할 수 있다**는 뜻이다.

일반적인 테이블 컴포넌트(`<DataGrid />` 형태)는 내부 상태를 스스로 관리하고, 개발자는 `onSortChange` 같은 콜백만 받는다. 이 경우 서버사이드로 전환하려면 내부 상태를 우회하거나 덮어써야 하는데, 그 과정에서 라이브러리와 싸우게 된다.

TanStack Table은 반대다. **상태를 처음부터 개발자가 소유(own)하는 것을 기본으로 설계**했다. 공식 Table State 가이드([링크](https://tanstack.com/table/v8/docs/framework/react/guide/table-state))에서는 이를 "controlled state"라고 부르며, `state` 옵션에 우리가 직접 관리하는 상태를 주입하고, `on[State]Change` 콜백으로 변경을 감지하는 패턴을 권장한다.

```
[사용자 인터랙션]
      ↓
[TanStack Table 상태 변경 감지]
      ↓
[on[State]Change 콜백 호출]
      ↓
[외부 상태(useState / URL params) 업데이트]
      ↓
[API 쿼리 재실행]
      ↓
[새 data를 table에 주입]
```

이 흐름에서 TanStack Table은 중간 단계의 **상태 감지와 UI 파생(헤더 정렬 아이콘, 페이지 버튼 활성화 여부 등)** 을 담당하고, 나머지는 개발자 영역이다. 서버사이드 테이블이 자연스럽게 맞아 들어오는 이유가 여기 있다.

---

# 2. 어떻게 사용하는지 (HOW)

## 2-1. 서버사이드에서 관리해야 할 상태 정리

공식 문서의 타입 정의 기준으로, 서버사이드에서 controlled state로 관리해야 할 핵심 상태 세 가지는 다음과 같다.

**SortingState** ([Sorting API](https://tanstack.com/table/v8/docs/api/features/sorting))
```ts
type SortingState = ColumnSort[]
type ColumnSort = {
  id: string   // 컬럼 accessorKey
  desc: boolean // true = 내림차순
}
```
정렬은 배열이므로 다중 정렬(multi-sort)도 기본 지원한다.

**ColumnFiltersState** ([Column Filtering API](https://tanstack.com/table/v8/docs/api/features/column-filtering))
```ts
type ColumnFiltersState = ColumnFilter[]
type ColumnFilter = {
  id: string  // 컬럼 accessorKey
  value: unknown  // 필터 값 (string, number, string[] 등)
}
```

**PaginationState** ([Pagination API](https://tanstack.com/table/v8/docs/api/features/pagination))
```ts
type PaginationState = {
  pageIndex: number  // 0-indexed
  pageSize: number
}
```

이 세 상태가 서버 쿼리의 입력값이 되고, 서버는 이를 받아 정렬·필터링·페이지네이션된 결과를 반환한다.

## 2-2. 서버사이드 기본 구조 예제 (React + @tanstack/react-table)

아래는 공식 문서([Table State Guide](https://tanstack.com/table/v8/docs/framework/react/guide/table-state))의 패턴을 기반으로 작성한 실무형 예제다.

```tsx
import { useState } from 'react'
import {
  useReactTable,
  getCoreRowModel,
  flexRender,
  createColumnHelper,
  SortingState,
  ColumnFiltersState,
  PaginationState,
} from '@tanstack/react-table'
import { useQuery } from '@tanstack/react-query'

// 1. 도메인 타입 정의
type Product = {
  id: number
  name: string
  category: string
  price: number
  stock: number
}

// 2. 컬럼 정의 — 제네릭으로 Product 타입 바인딩
const columnHelper = createColumnHelper<Product>()

const columns = [
  columnHelper.accessor('name', {
    header: '상품명',
    cell: info => info.getValue(),
  }),
  columnHelper.accessor('category', {
    header: '카테고리',
    cell: info => info.getValue(),
  }),
  columnHelper.accessor('price', {
    header: '가격',
    cell: info => info.getValue().toLocaleString() + '원',
  }),
  columnHelper.accessor('stock', {
    header: '재고',
    cell: info => info.getValue(),
  }),
]

// 3. 서버 API 호출 함수
async function fetchProducts(params: {
  sorting: SortingState
  columnFilters: ColumnFiltersState
  pagination: PaginationState
}) {
  const query = buildServerQuery(params) // 아래 2-4 참고
  const res = await fetch(`/api/products?${new URLSearchParams(query)}`)
  return res.json() as Promise<{ rows: Product[]; rowCount: number }>
}

export function ProductTable() {
  // 4. 상태를 외부에서 소유 (controlled state)
  const [sorting, setSorting] = useState<SortingState>([])
  const [columnFilters, setColumnFilters] = useState<ColumnFiltersState>([])
  const [pagination, setPagination] = useState<PaginationState>({
    pageIndex: 0,
    pageSize: 20,
  })

  // 5. TanStack Query로 데이터 패칭
  //    sorting/columnFilters/pagination이 queryKey에 포함 → 상태 변경 시 자동 재호출
  const { data, isLoading, isError } = useQuery({
    queryKey: ['products', sorting, columnFilters, pagination],
    queryFn: () => fetchProducts({ sorting, columnFilters, pagination }),
    placeholderData: prev => prev, // 페이지 전환 시 이전 데이터 유지 (깜빡임 방지)
  })

  // 6. 테이블 인스턴스 생성
  const table = useReactTable({
    data: data?.rows ?? [],
    columns,
    rowCount: data?.rowCount,   // 서버가 내려준 총 건수 주입
    state: {
      sorting,
      columnFilters,
      pagination,
    },
    onSortingChange: setSorting,
    onColumnFiltersChange: setColumnFilters,
    onPaginationChange: setPagination,
    // manual* = 클라이언트 계산 비활성화
    manualSorting: true,
    manualFiltering: true,
    manualPagination: true,
    getCoreRowModel: getCoreRowModel(),
    // getSortedRowModel, getFilteredRowModel, getPaginationRowModel은 생략
    // (서버가 처리하므로 클라이언트 row model 불필요)
  })

  // 7. 렌더링 (2-5 참고)
  if (isError) return <div>데이터를 불러오는 데 실패했습니다.</div>

  return (
    <div>
      <table>
        <thead>
          {table.getHeaderGroups().map(headerGroup => (
            <tr key={headerGroup.id}>
              {headerGroup.headers.map(header => (
                <th
                  key={header.id}
                  onClick={header.column.getToggleSortingHandler()}
                  style={{ cursor: header.column.getCanSort() ? 'pointer' : 'default' }}
                >
                  {flexRender(header.column.columnDef.header, header.getContext())}
                  {/* 정렬 방향 표시 */}
                  {{ asc: ' ↑', desc: ' ↓' }[header.column.getIsSorted() as string] ?? null}
                </th>
              ))}
            </tr>
          ))}
        </thead>
        <tbody>
          {isLoading
            ? (
              // 로딩 중: skeleton row (2-5 참고)
              Array.from({ length: pagination.pageSize }).map((_, i) => (
                <tr key={i}>
                  {columns.map((_, j) => (
                    <td key={j}><div className="skeleton" /></td>
                  ))}
                </tr>
              ))
            )
            : table.getRowModel().rows.length === 0
            ? (
              <tr>
                <td colSpan={columns.length}>조회 결과가 없습니다.</td>
              </tr>
            )
            : table.getRowModel().rows.map(row => (
              <tr key={row.id}>
                {row.getVisibleCells().map(cell => (
                  <td key={cell.id}>
                    {flexRender(cell.column.columnDef.cell, cell.getContext())}
                  </td>
                ))}
              </tr>
            ))
          }
        </tbody>
      </table>

      {/* 페이지네이션 UI */}
      <div>
        <button onClick={() => table.firstPage()} disabled={!table.getCanPreviousPage()}>{'<<'}</button>
        <button onClick={() => table.previousPage()} disabled={!table.getCanPreviousPage()}>{'<'}</button>
        <span>{table.getState().pagination.pageIndex + 1} / {table.getPageCount()}</span>
        <button onClick={() => table.nextPage()} disabled={!table.getCanNextPage()}>{'>'}</button>
        <button onClick={() => table.lastPage()} disabled={!table.getCanNextPage()}>{'>>'}</button>
      </div>
    </div>
  )
}
```

### `manual*` 옵션의 의미

공식 문서([Pagination Guide](https://tanstack.com/table/v8/docs/guide/pagination), [Sorting Guide](https://tanstack.com/table/v8/docs/guide/sorting))에서 명시하는 동작은 다음과 같다.

| 옵션 | 효과 |
|---|---|
| `manualPagination: true` | `getPaginationRowModel()` 없이도 동작. data가 이미 페이지 단위로 잘려 있다고 가정 |
| `manualSorting: true` | `getSortedRowModel()` 비활성화. data가 이미 정렬되어 있다고 가정 |
| `manualFiltering: true` | `getFilteredRowModel()` 비활성화. data가 이미 필터링되어 있다고 가정 |
| `rowCount` | 서버가 내려준 총 건수. 내부에서 `pageCount = Math.ceil(rowCount / pageSize)` 계산 |

공식 문서 기준 사실: sorting 가이드는 "클라이언트 정렬과 서버사이드 페이지네이션을 혼용하면 현재 페이지의 데이터만 정렬되는 문제가 생긴다"고 명시한다. **일관성이 중요하다 — 서버사이드로 가면 세 가지 모두 `manual`로 통일해야 한다.**

## 2-3. URL 동기화(URL-as-state)와의 결합

서버사이드 테이블의 상태(정렬·필터·페이지)를 URL에 올리면 다음과 같은 운영상 이점이 생긴다.
- 새로고침해도 같은 화면 유지
- 특정 필터/정렬 조건을 공유 가능
- 브라우저 뒤로가기로 이전 조회 조건 복원

TanStack Router의 search params와 결합하는 패턴이 가장 자연스럽다. (공식 예제: [Query Router Search Params](https://tanstack.com/table/v8/docs/framework/react/examples/query-router-search-params))

```tsx
import { useSearch, useNavigate } from '@tanstack/react-router'
import { Route } from './route' // TanStack Router의 Route 정의 파일

// Route 정의 (별도 파일)
// import { z } from 'zod'
// export const Route = createFileRoute('/products')({
//   validateSearch: z.object({
//     page: z.number().default(0),
//     pageSize: z.number().default(20),
//     sort: z.string().optional(),       // 예: "price_desc"
//     filters: z.string().optional(),    // 예: JSON.stringify(columnFilters)
//   }),
// })

export function ProductTableWithUrl() {
  const search = useSearch({ from: Route.id })
  const navigate = useNavigate({ from: Route.id })

  // URL → 테이블 상태 파싱
  const sorting: SortingState = search.sort
    ? [{ id: search.sort.split('_')[0], desc: search.sort.endsWith('_desc') }]
    : []

  const pagination: PaginationState = {
    pageIndex: search.page,
    pageSize: search.pageSize,
  }

  // 테이블 상태 변경 → URL 업데이트
  const handleSortingChange: OnChangeFn<SortingState> = updater => {
    const newSorting = updater instanceof Function ? updater(sorting) : updater
    navigate({
      search: prev => ({
        ...prev,
        page: 0, // 정렬 바뀌면 첫 페이지로
        sort: newSorting[0]
          ? `${newSorting[0].id}_${newSorting[0].desc ? 'desc' : 'asc'}`
          : undefined,
      }),
    })
  }

  const handlePaginationChange: OnChangeFn<PaginationState> = updater => {
    const newPagination = updater instanceof Function ? updater(pagination) : updater
    navigate({
      search: prev => ({ ...prev, page: newPagination.pageIndex, pageSize: newPagination.pageSize }),
    })
  }

  const table = useReactTable({
    // ...
    state: { sorting, pagination },
    onSortingChange: handleSortingChange,
    onPaginationChange: handlePaginationChange,
    manualSorting: true,
    manualPagination: true,
    getCoreRowModel: getCoreRowModel(),
  })

  // ...
}
```

TanStack Router 없이 React Router를 사용하는 경우에도 `useSearchParams`로 동일한 패턴을 구현할 수 있다. 다만 TanStack Router는 `validateSearch`에서 Zod 스키마를 사용할 수 있어, 쿼리 파라미터의 타입 안전성이 함께 보장된다.

## 2-4. 서버 API 쿼리 모델 설계 (SortingState / ColumnFiltersState → 서버 DSL)

테이블 상태를 그대로 API에 넘기는 것보다, 서버가 이해할 수 있는 형태로 변환하는 레이어를 두는 것이 좋다.

```ts
// TanStack Table 상태 → 서버 쿼리 파라미터 변환
function buildServerQuery(params: {
  sorting: SortingState
  columnFilters: ColumnFiltersState
  pagination: PaginationState
}): Record<string, string> {
  const { sorting, columnFilters, pagination } = params

  const query: Record<string, string> = {
    page: String(pagination.pageIndex),
    pageSize: String(pagination.pageSize),
  }

  // 정렬: [{ id: 'price', desc: true }] → sort=price&order=desc
  if (sorting.length > 0) {
    query.sort = sorting[0].id
    query.order = sorting[0].desc ? 'desc' : 'asc'
  }

  // 필터: [{ id: 'category', value: '스킨케어' }] → filter_category=스킨케어
  columnFilters.forEach(filter => {
    query[`filter_${filter.id}`] = String(filter.value)
  })

  return query
}
```

서버가 받을 요청 예시:
```
GET /api/products?page=2&pageSize=20&sort=price&order=desc&filter_category=스킨케어
```

서버 응답 예시:
```json
{
  "rows": [
    { "id": 1, "name": "토너패드", "category": "스킨케어", "price": 18000, "stock": 42 },
    { "id": 2, "name": "앰플", "category": "스킨케어", "price": 45000, "stock": 15 }
  ],
  "rowCount": 183
}
```

백엔드에서 받은 `rowCount`를 `useReactTable`의 `rowCount` 옵션에 주입하면, `table.getPageCount()`가 자동으로 `Math.ceil(183 / 20) = 10`을 반환한다.

복잡한 필터 구조(범위 필터, 다중 선택 등)가 필요한 경우에는 필터 상태를 JSON으로 직렬화해서 넘기거나, 별도의 필터 DSL을 정의하는 방법을 고려할 수 있다.

## 2-5. 렌더링 레이어와 로딩/에러/빈 상태 UX 고려사항

서버사이드 테이블에서 렌더링 레이어는 `table.getHeaderGroups()`와 `table.getRowModel()` 패턴을 기본으로 한다. 여기에 서버사이드 특유의 상태 처리가 추가된다.

**로딩 상태**: 데이터 패칭 중에는 Skeleton UI를 보여주는 것이 권장된다. 단, TanStack Query의 `placeholderData: prev => prev` 옵션을 활용하면 페이지 전환/정렬 변경 시 이전 데이터를 유지하다가 새 데이터로 교체할 수 있어 깜빡임(flash)이 없다. 이 경우에는 `isFetching` 플래그로 "조용한 로딩 인디케이터"(스피너 등)를 표시하는 것이 더 자연스럽다.

**에러 상태**: 네트워크 오류, 서버 오류를 명확하게 사용자에게 알려야 한다. TanStack Query의 `isError` / `error` 상태를 활용해 테이블 영역 내에 에러 메시지를 표시하거나, 재시도 버튼을 제공한다.

**빈 상태**: `table.getRowModel().rows.length === 0`이고 로딩이 완료된 경우, "검색 결과가 없습니다" UI를 표시한다. 필터가 적용된 상태라면 "필터를 초기화하세요"와 같은 액션도 함께 제공하면 좋다.

```tsx
// 렌더링 레이어 구조 요약
<tbody>
  {isLoading && <SkeletonRows count={pagination.pageSize} colSpan={columns.length} />}
  {isError && <ErrorRow colSpan={columns.length} onRetry={refetch} />}
  {!isLoading && !isError && table.getRowModel().rows.length === 0 && (
    <EmptyRow colSpan={columns.length} hasFilters={columnFilters.length > 0} onReset={() => setColumnFilters([])} />
  )}
  {!isLoading && !isError && table.getRowModel().rows.map(row => (
    <tr key={row.id}>
      {row.getVisibleCells().map(cell => (
        <td key={cell.id}>{flexRender(cell.column.columnDef.cell, cell.getContext())}</td>
      ))}
    </tr>
  ))}
</tbody>
```

---

# 3. 왜 써야 하는지 (WHY)

## 3-1. 클라이언트 ↔ 서버사이드 전환이 구조를 덜 흔드는 이유

많은 프로젝트에서 초기에는 클라이언트 정렬·필터로 시작했다가, 데이터가 늘면서 서버사이드로 전환하는 상황을 겪는다. 일반적인 테이블 라이브러리에서 이 전환은 컴포넌트 구조를 상당 부분 다시 짜야 하는 작업이다.

TanStack Table에서의 전환은 상대적으로 국소적이다.

```diff
const table = useReactTable({
  data: data?.rows ?? [],
  columns,
+ rowCount: data?.rowCount,
  state: { sorting, columnFilters, pagination },
  onSortingChange: setSorting,
  onColumnFiltersChange: setColumnFilters,
  onPaginationChange: setPagination,
+ manualSorting: true,
+ manualFiltering: true,
+ manualPagination: true,
  getCoreRowModel: getCoreRowModel(),
- getSortedRowModel: getSortedRowModel(),
- getFilteredRowModel: getFilteredRowModel(),
- getPaginationRowModel: getPaginationRowModel(),
})
```

`useReactTable` 옵션 수정과 API 연동 로직 추가가 전부다. **렌더링 코드(`getHeaderGroups`, `getRowModel`)는 전혀 바뀌지 않는다.** 이미 상태를 외부에서 관리하고 있었다면, 렌더링 레이어는 손댈 필요가 없다.

이 점이 서버사이드 전환을 겪어본 입장에서 가장 인상적이었다. 아키텍처를 갈아엎는 게 아니라, 옵션 몇 개와 API 연결만 추가하면 된다.

## 3-2. Router / Query / Table 세 레이어와의 역할 분리

TanStack 생태계는 각 라이브러리의 역할을 명확하게 구분한다.

| 레이어 | 라이브러리 | 역할 |
|---|---|---|
| **URL 상태** | TanStack Router | 정렬·필터·페이지 상태를 URL search params로 관리 |
| **서버 데이터 패칭·캐싱** | TanStack Query | API 호출, 로딩/에러 상태, 캐싱, 재조회 |
| **테이블 UI 로직** | TanStack Table | 헤더 렌더링, 정렬 토글 핸들러, 페이지 버튼 활성화 여부 |

이 세 레이어의 역할이 겹치지 않기 때문에, 각 레이어를 독립적으로 교체하거나 테스트할 수 있다. 예를 들어, URL 상태 관리를 React Router로 바꾸거나, 데이터 패칭을 SWR로 교체해도 TanStack Table 쪽 코드는 건드릴 필요가 없다.

공식 문서의 Table State 가이드에서도 이 패턴을 명시적으로 권장한다: "필터링·정렬·페이지네이션 상태는 자신의 state management에 저장하고, API가 신경 쓰지 않는 컬럼 순서·컬럼 가시성 등의 상태는 테이블 내부에 맡겨도 된다."

## 3-3. TypeScript DX와 도메인 모델 안정성 관점

`createColumnHelper<T>()`는 제네릭 타입 `T`를 컬럼 정의 전체에 전파한다.

```ts
type Order = {
  id: number
  productName: string
  status: 'pending' | 'shipped' | 'delivered'
  amount: number
}

const columnHelper = createColumnHelper<Order>()

columnHelper.accessor('status', {
  cell: info => {
    const status = info.getValue() // 타입: 'pending' | 'shipped' | 'delivered'
    return <StatusBadge status={status} />
  }
})

// 잘못된 키는 컴파일 타임에 오류
columnHelper.accessor('nonExistentField', { ... }) // ❌ TypeScript 오류
```

서버 도메인 모델(`Order` 타입)이 변경되면, 해당 필드를 사용하는 컬럼 정의에서 즉시 컴파일 오류가 발생한다. 타입을 통해 컬럼 정의와 서버 모델이 동기화되어 있음을 보장할 수 있다.

이것은 단순한 DX 개선이 아니다. 백엔드 API 스펙 변경이 프론트엔드 컴파일 오류로 이어지는 구조를 만드는 것이며, 런타임 오류보다 훨씬 안전하다.

## 3-4. 언제 TanStack Table이 *적합하지 않은지*

솔직하게 정리하면 다음 세 가지 경우에는 다른 선택이 나을 수 있다.

**엑셀 수준의 편집·피벗·내장 차트가 필요한 경우**  
AG Grid Enterprise는 셀 인라인 편집, 피벗 테이블, 내장 차트 등 엔터프라이즈 기능을 포함한다. TanStack Table은 이런 기능을 직접 구현해야 하며, 그 비용이 도메인 복잡도에 따라 커질 수 있다. 순수 데이터 조회 목적이라면 TanStack Table이 유리하지만, 스프레드시트에 가까운 편집 경험이 필요하다면 AG Grid 쪽이 현실적이다.

**아주 단순한 리스트 화면**  
컬럼이 3~4개이고, 정렬도 없고, 페이지네이션도 서버사이드가 아닌 화면이라면, TanStack Table을 도입하는 것이 오히려 오버엔지니어링이다. `useReactTable`을 설정하고 `getHeaderGroups`·`getRowModel` 패턴을 쓰는 보일러플레이트는 단순한 `<table>` 마크업보다 분명히 복잡하다. 단순한 케이스에서는 오히려 직접 짜는 쪽이 빠르고 읽기 쉽다.

**팀의 TypeScript / 상태 관리 숙련도가 낮은 경우**  
Controlled state 패턴, `on[State]Change` 콜백, Updater 함수 패턴은 TypeScript에 익숙하지 않으면 처음에 진입장벽이 될 수 있다. 특히 `onPaginationChange`의 콜백이 `Updater<PaginationState>` 타입(직접 값 또는 이전 상태를 받아 새 상태를 반환하는 함수)을 받는다는 것을 이해하지 못하면 버그로 이어지기 쉽다. 팀 전체가 이 패턴에 익숙해질 때까지의 온보딩 비용은 실재한다.

---

# 4. 정리

TanStack Table은 서버사이드 테이블 시나리오를 "나중에 지원할 수 있는 것"이 아니라 **처음부터 설계의 일부로 포함시킨 라이브러리**다. `manual*` 옵션으로 클라이언트 계산을 비활성화하고, controlled state로 정렬·필터·페이지네이션 상태를 외부에서 소유하며, `rowCount`로 서버의 총 건수를 주입하는 패턴이 공식 문서에 명시된 권장 방식이다.

이 구조는 TanStack Query(데이터 패칭)·TanStack Router(URL 동기화)와 역할이 겹치지 않고 자연스럽게 결합된다. 정렬 상태가 URL에 있고, URL 변경이 Query의 재패칭을 트리거하고, 새 데이터가 Table에 주입되는 흐름이 각 레이어의 책임 범위 안에서 이루어진다. TypeScript 제네릭 기반 컬럼 정의는 서버 도메인 모델 변경을 컴파일 타임에 감지할 수 있게 해준다.

물론 만능은 아니다. 엑셀 수준의 편집이 필요하거나, 아주 단순한 리스트거나, 팀 숙련도가 낮은 경우에는 도입을 재고해야 한다. 하지만 "백오피스나 운영 도구에서 서버사이드 테이블을 안정적으로 유지보수해야 한다"는 조건이라면, TanStack Table은 현재 시점에서 가장 균형 잡힌 선택지 중 하나라고 생각한다.

---

## 참고 링크

- [TanStack Table 공식 문서](https://tanstack.com/table/v8/docs/introduction)
- [Sorting Guide](https://tanstack.com/table/v8/docs/guide/sorting)
- [Column Filtering Guide](https://tanstack.com/table/v8/docs/guide/column-filtering)
- [Pagination Guide](https://tanstack.com/table/v8/docs/guide/pagination)
- [Table State (React) Guide](https://tanstack.com/table/v8/docs/framework/react/guide/table-state)
- [Sorting API](https://tanstack.com/table/v8/docs/api/features/sorting)
- [Pagination API](https://tanstack.com/table/v8/docs/api/features/pagination)
- [Column Filtering API](https://tanstack.com/table/v8/docs/api/features/column-filtering)
- [공식 예제: Query Router Search Params](https://tanstack.com/table/v8/docs/framework/react/examples/query-router-search-params)