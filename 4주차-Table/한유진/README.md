# Table

## TanStack Table은 언제 사용하면 좋을까?

공식문서에 따르면 TanStack Table은 테이블의 동작과 상태를 관리하는 엔진에 가깝다. 여기서 "엔진"이라는 표현이 중요하다. 자동차 엔진이 차를 움직이는 핵심 동력을 담당하지만, 차의 외관이나 인테리어는 제조사마다 다르게 만드는 것처럼, TanStack Table도 테이블이 동작하는 핵심 로직만 담당하고 UI는 개발자에게 맡긴다.

그럼 이 엔진이 관리하는 테이블의 동작과 상태는 정확히 무엇일까?

TanStack Table이 말하는 테이블의 동작과 상태는,

- 데이터를 어떻게 해석할지
- 데이터를 어떻게 변형할지
- 사용자와 어떻게 상호작용할지

에 대한 전반적인 로직 레이어를 의미한다.

---

## 데이터를 어떻게 해석하는가?

TanStack Table은 배열 데이터를 그대로 렌더링하지 않는다. 
데이터를 테이블이라는 구조에 맞게 재해석하는 과정을 거친다.

### Row Model 생성 (getCoreRowModel)

일반적인 배열 렌더링은 데이터를 `map()`으로 순회하며 JSX를 반환하는 게 전부다. 

하지만 TanStack Table은 원본 데이터를 그대로 쓰지 않고, 
먼저 내부적으로 Row Model이라는 구조체로 변환한다.

```tsx
// createCoreRowModel.ts

const rowModel = {
  rows: [],       // 계층 구조를 유지한 row 배열
  flatRows: [],   // 모든 row를 평탄화한 배열
  rowsById: {},   // id로 즉시 접근하기 위한 맵
}
```

각 row는 id, index, depth, parentId, subRows 같은 메타데이터를 포함한 구조체로 변환된다. 
이 덕분에 하위 row가 있는 중첩 데이터(트리 구조)도 자연스럽게 표현할 수 있고, 
`table.getRow(id)`처럼 id로 즉시 특정 row에 접근하는 것도 가능해진다.

```tsx
import {
  useReactTable,
  getCoreRowModel,
  flexRender,
} from '@tanstack/react-table'

const data = [
  {
    id: '1',
    name: '유진',
    age: 25,
    children: [
      { id: '1-1', name: '서브유진', age: 5 },
    ],
  },
  {
    id: '2',
    name: '콜리',
    age: 28,
  },
]

const columns = [
  {
    accessorKey: 'name',
    header: '이름',
  },
  {
    accessorKey: 'age',
    header: '나이',
  },
]

export default function Table() {
  const table = useReactTable({
    data,
    columns,
    getSubRows: (row) => row.children, // 👈 트리 구조 연결
    getCoreRowModel: getCoreRowModel(),
  })

  return (
    <table>
      <thead>
        {table.getHeaderGroups().map(headerGroup => (
          <tr key={headerGroup.id}>
            {headerGroup.headers.map(header => (
              <th key={header.id}>
                {flexRender(header.column.columnDef.header, header.getContext())}
              </th>
            ))}
          </tr>
        ))}
      </thead>

      <tbody>
        {table.getRowModel().rows.map(row => (
          <tr key={row.id}>
            {row.getVisibleCells().map(cell => (
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

일반적으로 `map` 을 사용해 테이블을 렌더링하면 다음과 같다.

```tsx
data.map(item => (
  <tr key={item.id}>
    <td>{item.name}</td>
    <td>{item.age}</td>
  </tr>
))
```

이때 item은 그냥 순수한 데이터 객체이다.

Tanstack Table에서는 이 데이터를 한번 가공하게 된다.

```tsx
table.getRowModel().rows.map(row => {
  console.log(row)
})
```

그럼 row는 다음과 같은 구조를 가지게 된다.

```tsx
{
  id: '1',
  index: 0,
  depth: 0,
  original: { id: '1', name: '유진', age: 25 },
  subRows: [ ... ],
  parentId: undefined,

  getValue: (columnId) => any,
  getVisibleCells: () => Cell[],
  getIsExpanded: () => boolean,
}
```

이 구조를 바탕으로 

- `table.getRow('1-1')` 과 같은 로직을 통해 특정 row에 O(1) 로 바로 접근할 수 있으며
- 트리 구조에 대한 UI를 쉽게 구현할 수 있다.
    
    ```tsx
    {table.getRowModel().rows.map(row => (
      <tr style={{ paddingLeft: row.depth * 20 }}>
        <td>{row.getValue('name')}</td>
      </tr>
    ))}
    ```
    

### Accessor - 컬럼이 데이터를 읽는 방식

Accessor는 row에서 컬럼의 값을 뽑아내는 공식이라고 이해하면 된다.

accessorKey와 accessorFn은 컬럼이 각 row로부터 어떤 값을 읽어올지를 정의하는 핵심 설정이다. 이 둘은 단순히 데이터를 “꺼내는 방법”을 지정하는 수준을 넘어서, 테이블 내부에서 사용되는 기준 데이터(source of truth)를 결정한다. 즉, 한 번 정의된 accessor는 렌더링뿐만 아니라 정렬, 필터링, 그룹핑 등 모든 데이터 연산의 기준이 된다.

```tsx
const data = [
  { firstName: '유진', lastName: '한', age: 25 },
  { firstName: '콜리', lastName: '김', age: 28 },
]
```

위와 같은 데이터가 존재한다고 했을 때

`accessorKey`는 가장 단순한 형태로, 객체의 특정 키를 그대로 참조하는 방식이다. 

예를 들어 accessorKey: 'firstName'을 정의하면, 

```tsx
const columns = [
  {
    accessorKey: 'firstName',
    header: '이름',
  },
]
```

TanStack Table은 내부적으로 (row) => row['firstName']와 같은 함수를 자동으로 생성한다. 

```tsx
(row) => row['firstName']
```

이후 row.getValue('firstName')를 호출하면, 이 내부적으로 생성된 함수가 실행되어 해당 값을 반환한다. 

```tsx
row.getValue('firstName') // 👉 '유진'
```

중요한 점은 이 값이 단순히 화면에 출력되는 용도가 아니라, 
테이블 엔진이 인식하는 “공식 컬럼 값”이 된다는 것이다.

반면 `accessorFn`은 개발자가 직접 값을 계산하는 함수를 정의하는 방식이다. 

```tsx
{ accessorFn: (row) => `${row.firstName} ${row.lastName}` }
```

예를 들어 위처럼 작성하면, 해당 컬럼의 값은 더 이상 원본 데이터에 존재하는 필드가 아니라, 
매 row마다 계산된 결과가 된다. 

이 경우에도 TanStack Table은 이 함수를 컬럼의 accessor로 등록하고, row.getValue(‘fullName’) 같은 호출을 통해 동일하게 접근할 수 있게 만든다. 

```tsx
row.getValue('fullName') // 👉 '한유진'
```

즉, accessorFn을 사용하면 원본 데이터 구조를 변경하지 않고도 새로운 “가상 컬럼”을 정의할 수 있다.

이 둘을 정의하면 내부적으로 일어나는 일은 동일하다. TanStack Table은 각 컬럼에 대해 반드시 하나의 accessor 함수(accessorFn)를 가지도록 정규화하고, 이후 모든 로직에서 이 함수를 통해 값을 읽는다. accessorKey는 이 accessor 함수를 자동으로 만들어주는 축약 문법일 뿐이다. 결국 테이블 내부에서는 row.getValue(columnId)가 호출될 때마다 column.accessorFn(row.original)이 실행되는 구조로 동작한다.

두 방식의 차이는 “정적 접근 vs 계산된 접근”으로 정리할 수 있다. accessorKey는 단순히 존재하는 필드를 그대로 참조하는 데 적합하고, 구현이 간단하며 타입 추론도 직관적이다. 반면 accessorFn은 여러 필드를 조합하거나, 값을 변환하거나, 파생 데이터를 만들어야 할 때 사용된다. 특히 정렬이나 필터링 기준을 원본 데이터가 아니라 가공된 값으로 삼고 싶을 때 필수적이다.

---

## 데이터를 어떻게 변형하는가?

TanStack Table의 핵심 강점 중 하나는 데이터를 여러 단계를 거쳐 가공하는 파이프라인 구조를 내장하고 있다는 점이다.

```tsx
import {
  useReactTable,
  getCoreRowModel,
  getFilteredRowModel,
  getSortedRowModel,
  getPaginationRowModel,
  getGroupedRowModel,
  getExpandedRowModel,
  flexRender,
} from '@tanstack/react-table'

import { useState } from 'react'

const data = [
  { name: '유진', age: 25, city: '서울', salary: 3000 },
  { name: '콜리', age: 28, city: '서울', salary: 5000 },
  { name: '민수', age: 32, city: '부산', salary: 4000 },
  { name: '지영', age: 29, city: '부산', salary: 3500 },
]

export default function Table() {
  const [sorting, setSorting] = useState([])
  const [columnFilters, setColumnFilters] = useState([])
  const [grouping, setGrouping] = useState(['city'])

  const columns = [
    {
      accessorKey: 'name',
      header: '이름',
    },
    {
      accessorKey: 'age',
      header: '나이',
    },
    {
      accessorKey: 'city',
      header: '도시',
    },
    {
      accessorKey: 'salary',
      header: '연봉',
      aggregationFn: 'sum', // 👈 그룹핑 시 합계
      cell: info => info.getValue(),
      aggregatedCell: info => `합계: ${info.getValue()}`,
    },
  ]

  const table = useReactTable({
    data,
    columns,

    state: {
      sorting,
      columnFilters,
      grouping,
    },

    onSortingChange: setSorting,
    onColumnFiltersChange: setColumnFilters,
    onGroupingChange: setGrouping,

    getCoreRowModel: getCoreRowModel(),

    // 👇 기능별 row model 연결
    getFilteredRowModel: getFilteredRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getGroupedRowModel: getGroupedRowModel(),
    getExpandedRowModel: getExpandedRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
  })

  return (
    <div>
      {/* 🔍 필터 */}
      <input
        placeholder="이름 검색"
        value={table.getColumn('name')?.getFilterValue() ?? ''}
        onChange={(e) =>
          table.getColumn('name')?.setFilterValue(e.target.value)
        }
      />

      <table border={1}>
        <thead>
          {table.getHeaderGroups().map(headerGroup => (
            <tr key={headerGroup.id}>
              {headerGroup.headers.map(header => (
                <th
                  key={header.id}
                  onClick={header.column.getToggleSortingHandler()} // 🔥 정렬
                >
                  {flexRender(
                    header.column.columnDef.header,
                    header.getContext()
                  )}
                  {header.column.getIsSorted() === 'asc' && ' 🔼'}
                  {header.column.getIsSorted() === 'desc' && ' 🔽'}
                </th>
              ))}
            </tr>
          ))}
        </thead>

        <tbody>
          {table.getRowModel().rows.map(row => (
            <tr key={row.id}>
              {row.getVisibleCells().map(cell => (
                <td key={cell.id}>
                  {flexRender(
                    cell.column.columnDef.cell ??
                      cell.column.columnDef.aggregatedCell,
                    cell.getContext()
                  )}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>

      {/* 📄 페이지네이션 */}
      <button onClick={() => table.previousPage()}>이전</button>
      <button onClick={() => table.nextPage()}>다음</button>

      <span>
        Page {table.getState().pagination.pageIndex + 1}
      </span>
    </div>
  )
}
```

원본 데이터 → 필터링 → 정렬 → 그룹핑 → 페이지네이션

각 단계는 독립적인 Row Model로 처리되며,  useReactTable  옵션에서 해당 모델을 활성화하면 자동으로 파이프라인에 연결된다.

### 필터링 (getFilteredRowModel)

```tsx
state: { columnFilters: [{ id: 'name', value: '김' }] }

getFilteredRowModel: getFilteredRowModel()
column.setFilterValue(value)
```

각 컬럼에 `filterFn`을 지정하면, TanStack Table이 자동으로 조건에 맞는 row만 추려낸다. 내부적으로 `columnFilters` 상태를 참조해 필터링된 Row Model을 새로 생성한다. 기본 제공되는 `includesString`, `equals`, `inNumberRange` 같은 내장 필터 함수 외에도 커스텀 함수를 자유롭게 연결할 수 있다. 글로벌 필터 `globalFilter` 도 지원해 전체 데이터 검색이 간단하다.

### 정렬 (getSortedRowModel)

```tsx
state: { sorting: [{ id: 'age', desc: true }] }

getSortedRowModel: getSortedRowModel()
column.getToggleSortingHandler()
```

`sorting` 상태는 어떤 컬럼 기준으로, 어떤 방향(오름/내림)으로 정렬할지를 담고 있다. 실제 코드(`rowSortingFeature.ts`)를 보면 정렬 상태를 초기화하고, 컬럼마다 자동으로 적합한 정렬 함수(`getSortFn`)를 연결하는 로직이 있다. 문자열, 숫자, 날짜 등 타입에 따라 알맞은 기본 정렬 함수가 자동 선택되고, 필요하면 커스텀 정렬 함수도 지정할 수 있다. 다중 컬럼 정렬도 지원한다.

### 페이지네이션 (getPaginationRowModel)

```tsx
state: { pagination: { pageIndex: 0, pageSize: 10 } }

getPaginationRowModel: getPaginationRowModel()
table.nextPage()
```

전체 데이터 중 현재 페이지에 해당하는 subset만 추출한다. `table.nextPage()`, `table.previousPage()`, `table.setPageSize()` 같은 메서드가 내부적으로 pagination 상태를 업데이트하고, 그 상태에 따라 보여줄 row 범위가 다시 계산된다. 단순해 보이지만, 앞선 필터링과 정렬 이후의 결과를 기준으로 페이지네이션이 적용되기 때문에 파이프라인의 가장 마지막에 위치한다. 서버 사이드 페이지네이션과도 쉽게 연동 가능하다.

### 그룹핑 + Aggregation

```tsx
{ id: 'category', aggregationFn: 'sum' }

getGroupedRowModel: getGroupedRowModel()
state: { grouping: ['city'] }

aggregationFn: 'sum'
aggregatedCell: info => `합계: ${info.getValue()}`
```

특정 컬럼 기준으로 row를 묶고, 그룹 단위로 합계, 평균 같은 집계 연산을 수행한다. 단순한 표 렌더링을 넘어서 “부서별 총 매출”, “카테고리별 평균 수량” 같은 데이터 요약이 필요할 때 유용하다.  `getGroupedRowModel()`으로 활성화되며, 중첩 그룹핑도 지원한다. 이 파이프라인 덕분에 데이터 변형이 모듈화되어, 필요한 기능만 선택적으로 활성화할 수 있다

---

## 사용자와 어떻게 상호작용하는가?

TanStack Table은 UI를 직접 제공하지 않는다. 버튼도 없고, 입력창도 없다. 그러나 사용자 인터랙션에 필요한 상태와 핸들러는 전부 제공한다.

이 차이가 중요하다. TanStack Table은 "무엇을 클릭하면 어떤 상태가 바뀌어야 한다"는 로직은 알고 있지만, 그 클릭 가능한 요소를 어떻게 생긴 버튼으로 만들지는 개발자가 결정한다.

```tsx
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  getFilteredRowModel,
  getExpandedRowModel,
  flexRender,
} from '@tanstack/react-table'

import { useState } from 'react'

const data = [
  {
    id: '1',
    name: '유진',
    age: 25,
    children: [{ id: '1-1', name: '서브유진', age: 5 }],
  },
  { id: '2', name: '콜리', age: 28 },
]

export default function Table() {
  const [sorting, setSorting] = useState([])
  const [columnFilters, setColumnFilters] = useState([])
  const [rowSelection, setRowSelection] = useState({})
  const [columnVisibility, setColumnVisibility] = useState({})

  const columns = [
    {
      id: 'select',
      header: ({ table }) => (
        <input
          type="checkbox"
          checked={table.getIsAllRowsSelected()}
          onChange={table.getToggleAllRowsSelectedHandler()}
        />
      ),
      cell: ({ row }) => (
        <input
          type="checkbox"
          checked={row.getIsSelected()}
          onChange={row.getToggleSelectedHandler()}
        />
      ),
    },
    {
      accessorKey: 'name',
      header: ({ column }) => (
        <span onClick={column.getToggleSortingHandler()}>
          이름 {column.getIsSorted() === 'asc' ? '▲' : '▼'}
        </span>
      ),
    },
    {
      accessorKey: 'age',
      header: '나이',
    },
  ]

  const table = useReactTable({
    data,
    columns,

    state: {
      sorting,
      columnFilters,
      rowSelection,
      columnVisibility,
    },

    onSortingChange: setSorting,
    onColumnFiltersChange: setColumnFilters,
    onRowSelectionChange: setRowSelection,
    onColumnVisibilityChange: setColumnVisibility,

    getSubRows: (row) => row.children,

    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    getExpandedRowModel: getExpandedRowModel(),
  })

  return (
    <div>
      {/* 🔍 필터 */}
      <input
        placeholder="이름 검색"
        value={table.getColumn('name')?.getFilterValue() ?? ''}
        onChange={(e) =>
          table.getColumn('name')?.setFilterValue(e.target.value)
        }
      />

      {/* 👁 컬럼 토글 */}
      <label>
        <input
          type="checkbox"
          checked={table.getColumn('age')?.getIsVisible()}
          onChange={table.getColumn('age')?.getToggleVisibilityHandler()}
        />
        나이 컬럼 보기
      </label>

      <table border={1}>
        <thead>
          {table.getHeaderGroups().map(headerGroup => (
            <tr key={headerGroup.id}>
              {headerGroup.headers.map(header => (
                <th key={header.id}>
                  {flexRender(
                    header.column.columnDef.header,
                    header.getContext()
                  )}
                </th>
              ))}
            </tr>
          ))}
        </thead>

        <tbody>
          {table.getRowModel().rows.map(row => (
            <>
              <tr key={row.id}>
                {row.getVisibleCells().map(cell => (
                  <td key={cell.id}>
                    {flexRender(
                      cell.column.columnDef.cell,
                      cell.getContext()
                    )}
                  </td>
                ))}

                {/* 📂 확장 버튼 */}
                <td>
                  {row.getCanExpand() && (
                    <button onClick={row.getToggleExpandedHandler()}>
                      {row.getIsExpanded() ? '접기' : '열기'}
                    </button>
                  )}
                </td>
              </tr>

              {/* 📂 확장된 row */}
              {row.getIsExpanded() &&
                row.subRows.map(subRow => (
                  <tr key={subRow.id}>
                    <td colSpan={3}>
                      👉 {subRow.original.name} ({subRow.original.age})
                    </td>
                  </tr>
                ))}
            </>
          ))}
        </tbody>
      </table>
    </div>
  )
}
```

### 정렬 인터랙션

```tsx
onClick={column.getToggleSortingHandler()}
```

이 함수는 헤더에 `onClick`으로 연결할 수 있는 핸들러를 반환한다. 클릭할 때마다 오름차순 → 내림차순 → 해제 순서로 상태가 순환한다. `column.getIsSorted()`로 현재 정렬 상태를 읽어서 UI에 화살표 아이콘을 표시하는 것도 가능하다.

```tsx
<th onClick={column.getToggleSortingHandler()}>
  이름 {column.getIsSorted() === 'asc' ? '▲' : '▼'}
</th>

```

### 필터 입력

```tsx
onChange={(e) => column.setFilterValue(e.target.value)}
```

입력창의 `onChange`에 `column.setFilterValue()`를 연결하면 된다. 상태 관리 코드를 별도로 작성할 필요 없이, TanStack Table이 `columnFilters` 상태를 자동으로 업데이트하고 필터링된 결과를 다시 계산해준다.

### 행 선택, 확장, 컬럼 가시성 등

정렬과 필터 외에도 `row.getToggleSelectedHandler()`, 

- 행 선택
    
    ```tsx
    onChange={row.getToggleSelectedHandler()}
    ```
    
- 확장
    
    ```tsx
    onClick={row.getToggleExpandedHandler()}
    ```
    
- 컬럼 가시성
    
    ```tsx
    onChange={column.getToggleVisibilityHandler()}
    ```
    

개발자는 이 핸들러들을 원하는 UI 요소에 연결하기만 하면 된다.

---

## 정리: TanStack Table은 무엇을 책임지는가?

결국 세 가지로 정리할 수 있다.

- 데이터 해석 → 원본 배열을 row/cell/column 구조체로 모델링
- 데이터 변형 → 필터 → 정렬 → 그룹 → 페이지네이션 파이프라인
- 사용자 상호작용 → 인터랙션에 필요한 상태와 핸들러 일괄 제공

TanStack Table은 데이터 → 상태 → UI로 이어지는 흐름에서 "데이터와 상태"를 전담하는 엔진이다. UI는 개발자가 만들고, 로직은 TanStack Table이 책임지는 구조다.

이 분리가 중요한 이유는, 같은 테이블 로직 위에 완전히 다른 UI를 얹을 수 있기 때문이다. 디자인 시스템이 바뀌어도 로직은 그대로 재사용하고, 반대로 정렬·필터·페이지네이션 같은 복잡한 기능을 추가할 때 UI 컴포넌트를 건드릴 필요가 없다. UI와 로직이 명확히 분리된 구조 덕분에 각자의 역할에만 집중할 수 있다.

성능 측면에서 Row Model은 메모리 효율적이며, 
큰 데이터셋에서도 가상화(Virtualization)와 결합해 부드럽게 동작한다. 
TypeScript 완벽 지원으로 런타임 오류도 최소화된다.

단순히 데이터를 표로 보여주기만 하면 TanStack Table은 과할 수 있다. 
하지만 정렬, 필터링, 페이지네이션, 그룹핑, 행 선택처럼 테이블의 상태가 복잡해지는 순간, 
TanStack Table이 그 복잡성을 대신 감당해준다!

## 느낀점,,

id로 맵으로 만들어서 데이터 검색시 시간복잡도 줄이는건 진짜 획기적이다 요거 하나로도 일단 쓸 이유 만족