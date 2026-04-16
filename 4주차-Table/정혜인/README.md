# Table이란 무엇일까?

> **"Table은 UI 컴포넌트가 아니라, 테이블 로직을 관리하는 Headless UI 라이브러리다."**
> 즉, TanStack Table은 `<table>` 태그를 렌더링해주는 것이 아니라, **테이블이 가져야 할 상태와 연산(정렬, 필터, 페이지네이션 등)을 관리하는 상태머신**입니다.

TanStack Table은 데이터를 행(Row)과 열(Column)로 구조화하고, 정렬/필터/페이지네이션 같은 복잡한 연산을 **프레임워크에 무관하게** 처리하는 코어엔진을 제공합니다.  
(그냥 쉽게 말하면 테이블에 표시하고 싶은 데이터를 잘 정리된 하나의 객체로 만들어주는데, useReactTable 훅을 통해 정해진 값만 잘 넘겨주면 테이블 객체를 정의가 가능함)

### 핵심 정의: Headless 테이블 상태 관리자

`AG Grid`, `MUI DataGrid` 같은 라이브러리가 **UI + 로직**을 함께 제공한다면, TanStack Table은 **로직만** 전담합니다. 화면에 어떻게 그릴지는 100%개발자 몫입니다.

- **Headless의 특징:** 마크업과 스타일에 대한 어떤 강제도 없음
  - `<div>` 기반이든, `<table>` 기반이든, 심지어 Canvas든 자유롭게 렌더링 가능
  - CSS 프레임워크(Tailwind, styled-components 등)와 충돌 없이 결합됨
  - 디자인 시스템에 자연스럽게 녹아들 수 있음

### 아키텍처 구조

TanStack Table은 **프레임워크 어댑터 패턴**을 사용합니다.

`[React/Vue/Solid/Svelte 어댑터] ↔ [table-core (상태 + 연산)] ↔ [사용자 데이터]`

`table-core`가 모든 로직을 프레임워크 무관하게 처리하고, `react-table`은 이를 React의 상태 시스템(`useStore`)에 연결하는 얇은 어댑터 역할만 합니다.

> 🤔 왜 이런 구조를 사용하나?
>
> - **로직의 일관성:** 우리 회사가 웹과 모바일 앱(React Native)을 동시에 개발하게되면 테이블의 정렬/필터 로직은 `table-core` 하나로 완전히 똑같이 동작. 프레임워크를 바꿔도 '버그'가 생기지 않음.
> - **번들 사이즈 최적화:** 어댑터가 아주 얇기 때문에 내가 쓰는 프레임워크에 필요한 최소한의 코드만 불러올 수 있음
> - **유지보수의 효율성:** 정렬 알고리즘에 버그가 발견되면 `table-core`만 고치면 됨. React 어댑터, Vue 어댑터를 일일이 수정할 필요가 없음!

# Table은 어떻게 사용하는 걸까?

### 1. 기본 설정

데이터와 컬럼 정의를 준비하고, `useTable` 훅에 전달합니다.

```jsx
import { useTable } from '@tanstack/react-table'
import { tableFeatures, createColumnHelper } from '@tanstack/table-core'

// 1. 사용할 기능(feature)들을 선언
const _features = tableFeatures({
  rowSortingFeature,
  columnFilteringFeature,
  rowPaginationFeature,
})

// 2. 컬럼 정의 (columnHelper로 타입 안전하게)
const columnHelper = createColumnHelper<typeof _features, Person>()

const columns = [
  columnHelper.accessor('firstName', {
    header: '이름',
    cell: (info) => info.getValue(),
  }),
  columnHelper.accessor('age', {
    header: '나이',
  }),
  columnHelper.accessor('email', {
    header: '이메일',
  }),
]

// 3. 테이블 인스턴스 생성
function MyTable({ data }: { data: Person[] }) {
  const table = useTable({
    _features,
    data,
    columns,
  })

  return (
    <table>
      <thead>
        {table.getHeaderGroups().map((headerGroup) => (
          <tr key={headerGroup.id}>
            {headerGroup.headers.map((header) => (
              <th key={header.id}>
                <table.FlexRender header={header} />
              </th>
            ))}
          </tr>
        ))}
      </thead>
      <tbody>
        {table.getRowModel().rows.map((row) => (
          <tr key={row.id}>
            {row.getVisibleCells().map((cell) => (
              <td key={cell.id}>
                <table.FlexRender cell={cell} />
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  )
}
```

### 핵심 개념: 4가지 엔티티

TanStack Table의 모든 것은 4가지 엔티티를 중심으로 돌아갑니다.

| **엔티티 (Entity)** | **설명 (Description)**                                                                                | **예시 API**                                     |
| ------------------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| **Table**           | 테이블의 **전체 인스턴스**. 테이블의 상태(state)와 전역 설정을 관리합니다.                            | `table.getRowModel()`, `table.setSorting()`      |
| **Column**          | 테이블의 **열(세로줄) 정의**. 데이터 접근 방식, 헤더 렌더링, 열별 능(정렬, 필터 등)을 담당합니다.     | `column.getIsSorted()`, `column.toggleSorting()` |
| **Row**             | 테이블의 **행(가로줄)**. 원본 데이터 한 건을 나타내며, 해당 행의 선택 상태나 하위 행 정보를 가집니다. | `row.getValue('age')`, `row.getIsSelected()`     |
| **Cell**            | **행과 열의 교차점**. 실제로 화면에 그려질 데이터와 렌더링 컨텍스트를 제공합니다.                     | `cell.getValue()`, `cell.getContext()`           |

- **관심사 분리**: '전체 정렬'은 `Table`에게, '이 칸의 색상'은 `Cell`에게 물어보면 됩니다. 로직이 섞이지 않아 유지보수가 편해집니다.
- **최적화:** "직접 구현했다면 1만 개의 Cell에 일일이 함수를 만들었겠지만, TanStack은 `엔티티 객체`를 통해 메모리를 효율적으로 관리합니다.

여기에 Header가 추가됩니다. Header는 Column의 시각적 표현으로, 다중 행 헤더(그룹 헤더)를 위해 colSpan, rowSpan 계산을 내부적으로 처리합니다.

+) 추가 팁

- columnHelper.accessor: 문자열 키('firstName')를 넘기면 자동으로 accessorFn이 생성됩니다. 'address.city' 같은 깊은 경로(Deep Key)도 . 기준으로자동 분할되어 접근합니다.
- table.Subscribe: 특정 상태 슬라이스만 구독하여 해당 부분이 변할 때만 리렌더링을 발생시키는 HOC입니다. 대형 테이블에서 성능 최적화에 핵심입니다.

# Table은 왜 쓰는 걸까?

## 1. 테이블 UI의 고질적 문제 해결

테이블을 직접 구현하려면 다음과 같은 로직을 모두 직접 짜야 합니다.

```markdown
┌────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────┐
│ 직접 구현 시 단점 │ TanStack Table 솔루션 │
├────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────┤
│ 정렬 (다중 컬럼 정렬, undefined 처리, 방향 토글) │ Multi-column Sorting: Shift+클릭으로 다중 정렬, 자동 정렬 함수 감지 │
├────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────┤
│ 필터링 (컬럼별 + 글로벌 필터, 커스텀 필터 함수) │ Column/Global Filtering: 내장 필터 함수 + 커스텀 함수 주입 │
├────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────┤
│ 페이지네이션 (페이지 수 계산, 이전/다음 활성 여부) │ Pagination Feature: getCanNextPage(), getPageCount() 등 선언적 API │
├────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────┤
│ 행 선택 (단일/다중 선택, 전체 선택, 하위 행 연동) │ Row Selection: 체크박스 상태 관리 + 부모-자식 연동 자동 처리 │
├────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────┤
│ 컬럼 고정 (좌/우 고정 시 스크롤 처리) │ Column Pinning: left/right 핀 상태 관리 │
├────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────┤
│ 메모리 낭비 (1만 행 × 10열 = 10만 셀 인스턴스) │ Prototype 공유: 모든 셀/행/열이 공유 프로토타입 사용 → 메서드 중복 제거 │
└────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────┘
```

- 정렬: sortUndefined 옵션 하나로 빈 값들의 위치를 고정할 수 있고, 다중 정렬 로직이 코어 엔진에 완벽하게 내장되어 있습니다. 데이터 타입을 알아서 추론해 숫자/문자/날짜에 맞는 최적의 정렬 알고리즘(`sortFn: 'auto'`)을 적용해 줍니다.
- 필터링: 열별 필터 상태(`columnFilters`)와 전체 필터 상태(`globalFilter`)를 독립적으로 관리하며, 이 둘을 체이닝(Chaining)하여 결과를 뽑아냅니다.
- 페이지네이션: 코어 내부에서 의존성(데이터 개수, 필터 상태 등)을 추적하여 자동으로 안전한 페이지 인덱스를 보장합니다. `getCanNextPage()` 같은 직관적인 API만 호출해서 UI 버튼을 비활성화(disabled) 처리하면 끝입니다.
- 메모리 낭비: `Object.create()`를 활용한 프로토타입 패턴을 적용해, 10만 개의 셀이 단 하나의 프로토타입 객체(메서드들)를 메모리에서 공유하도록 설계되어있습니다. (만 개의 행과 10개의 열이 있으면 10만 개의 셀이 존재합니다. 만약 각 셀마다 onClick, formatData 같은 화살표 함수를 독립적으로 갖고있으면 브라우저 메모리가 터지거나 엄청난 렉이 걸림)

## 2. Headless = 완전한 디자인 자유

- AG Grid를 쓰면 AG Grid의 CSS를 오버라이드해야 하고, 디자인 시스템과 충돌이 불가피합니다.
- TanStack Table은 마크업 자체를 개발자가 작성하므로 디자인 시스템의 컴포넌트를 그대로 활용할 수 있습니다.

## 3. 프레임워크에 구속받지 않는 코어

- table-core는 React에 의존하지 않습니다. React → Vue 마이그레이션을 해도 테이블 로직(정렬, 필터 등)은 그대로 재사용됩니다.
- 코어가 @tanstack/store를 사용하므로, 프레임워크별 어댑터(react-table, vue-table, solid-table)는 상태 구독 방식만 다릅니다.

## 4. Feature 플러그인 아키텍처

필요한 기능만 골라 쓸 수 있습니다. 정렬이 필요 없으면 rowSortingFeature를 빼면 됩니다. 이는 번들 사이즈 최적화(Tree-shaking)에도 직결됩니다.

# 동작을 파헤쳐보자!

useTable 한 줄이 테이블을 그리기까지 - 전체 내부 흐름

```tsx
전체 흐름 요약도

useTable(options)
  │
  ▼
[1] useState(() => constructTable(options))        ← 테이블 인스턴스 최초 1회 생성
  │    ├─ coreFeatures + _features 병합
  │    ├─ 각 feature의 getInitialState() 수집 → 초기 상태 조립
  │    ├─ createStore(initialState)              ← @tanstack/store 생성
  │    ├─ optionsStore 생성 (defaultOptions + userOptions)
  │    └─ 각 feature의 constructTableAPIs(table)  ← table에 API 부착
  │
[2] table.setOptions(prev => ({ ...prev, ...options }))  ← 매 렌더마다 옵션 동기화
  │
[3] useStore(table.store, selector, shallow)       ← 상태 구독 (선택적 리렌더)
  │
[4] return { ...table, state }                     ← 테이블 + 선택된 상태 반환
  │
  ▼  (렌더링 시)
table.getHeaderGroups()
  └─ buildHeaderGroups(allColumns, visibleColumns)
       ├─ 최대 depth 계산
       ├─ 리프 헤더부터 상위로 재귀 구성 (bottom-up)
       └─ colSpan/rowSpan 재귀 계산

table.getRowModel()
  └─ Row Model Pipeline (체인 실행)
       ├─ getCoreRowModel()      ← data → Row 인스턴스 변환
       ├─ getFilteredRowModel()  ← 필터 적용
       ├─ getSortedRowModel()    ← 정렬 적용
       └─ getPaginatedRowModel() ← 페이지 슬라이싱

row.getVisibleCells()
  └─ constructCell(column, row, table)
       └─ Object.create(cellPrototype)  ← 공유 프로토타입으로 생성

```

### 공장을 세팅해봅시다(초기화와 상태 구독) Step1 ~ Step4

- **요점 1: 공장은 한 번만 짓는다 (useState 초기화)**
  - 매번 렌더링될 때마다 공장을 다시 지으면(초기화하면) 안됨! 그래서 `useState` 초기화 함수에 넣어 최초 1회만 `constructTable`이 실행되게끔 함
- **요점 2: 플러그인(Feature) 조립**
  - 이 공장에는 기본 엔진(Core)이 있고, 우리가 필요한 기능(정렬, 필터 등)만 모듈처럼 끼워 넣어야함. 안 쓰는 기능은 버려지니(Tree-shaking) 가벼워짐
- **요점 3: 똑똑한 구독 시스템 (useStore와 selector)**
  - 데이터가 1만 개인데 3페이지에서 4페이지로 넘어간다고 전체 화면을 다 다시 그려야 하나..?
    놉. `@tanstack/store`를 통해 내가 보고 있는 상태(예: pagination)가 변할 때만 딱 리렌더링 되도록 핀셋으로 집어내는 것, 그게 바로 `selector`의 핵심이 됨

- Step 1 — useQuery → useBaseQuery
  코드 (react-table/src/useTable.ts:88-104)

```tsx
const [table] = useState(() => {
  const tableInstance = constructTable(tableOptions);
  tableInstance.Subscribe = function SubscribeBound(props) {
    return Subscribe({ ...props, table: tableInstance });
  };
  tableInstance.FlexRender = FlexRender;
  return tableInstance;
});
```

useState의 initializer 함수로 넘겼기 때문에 최초 렌더 1회만 실행됩니다. TanStack Query의 QueryObserver 생성과 동일한 패턴입니다.

이 시점에서 React 전용 컴포넌트인 Subscribe와 FlexRender가 테이블 인스턴스에 직접 부착됩니다. 이후 JSX에서 <table.Subscribe>, <table.FlexRender>로
바로 사용할 수 있게 되는 이유입니다.

- Step 2 — constructTable 내부: Feature 시스템 조립
  코드 (table-core/src/core/table/constructTable.ts:29-95)

```tsx
export function constructTable(tableOptions) {
  const table = {
    _features: { ...coreFeatures, ...tableOptions._features }, // ①
    _rowModels: {},
    _rowModelFns: {},
    get options() { return this.optionsStore.state },           // ②
  }

  // ③ 각 feature의 defaultOptions를 수집
  const defaultOptions = featuresList.reduce((obj, feature) => {
    return Object.assign(obj, feature.getDefaultTableOptions?.(table))
  }, {})

  // ④ Store 생성 (3개의 스토어)
  table.optionsStore = createStore({ ...defaultOptions, ...tableOptions })
  table.initialState = getInitialTableState(table._features, ...)
  table.baseStore = table.options.store ?? createStore(table.initialState)
  table.store = createStore(() => ({
    ...table.baseStore.state,
    ...(table.optionsStore.state.state ?? {}),
  }))

  // ⑤ 각 feature가 table에 API를 부착
  for (const feature of featuresList) {
    feature.constructTableAPIs?.(table)
  }

  return table
}
```

6개의 코어 Feature + 사용자가 선택한 Feature가 합쳐집니다.
코어 6개 (coreFeatures.ts:17-24):

```markdown
┌──────────────────────┬──────────────────────────────┐
│ Feature │ 역할 │
├──────────────────────┼──────────────────────────────┤
│ coreTablesFeature │ 테이블 상태/옵션 관리의 뼈대 │
├──────────────────────┼──────────────────────────────┤
│ coreColumnsFeature │ 컬럼 생성, 기본 컬럼 API │
├──────────────────────┼──────────────────────────────┤
│ coreRowsFeature │ 행 생성, 행 접근 API │
├──────────────────────┼──────────────────────────────┤
│ coreCellsFeature │ 셀 생성, 값 접근 API │
├──────────────────────┼──────────────────────────────┤
│ coreHeadersFeature │ 헤더 그룹 빌드 │
├──────────────────────┼──────────────────────────────┤
│ coreRowModelsFeature │ Row Model 파이프라인 관리 │
└──────────────────────┴──────────────────────────────┘
```

> 3개의 Store가 존재하는 이유
>
> - baseStore: 실제 테이블 상태 (sorting, pagination 등). 외부에서 주입 가능
> - optionsStore: 테이블 옵션 (columns, data, onSortingChange 등). 매 렌더마다 동기화됨
> - store: baseStore + optionsStore의 state override를 합친 파생 스토어. React가 구독하는 대상

- Step 3 — Feature의 상태 초기화
  코드 (constructTable.ts:10-20)

```tsx
export function getInitialTableState(features, initialState = {}) {
  Object.values(features).forEach((feature) => {
    initialState = feature.getInitialState?.(initialState) ?? initialState;
  });
  return structuredClone(initialState);
}
```

각 Feature가 자신의 상태 슬라이스를 추가합니다.

```tsx
{} → coreTablesFeature → { ...coreState }
   → rowSortingFeature → { sorting: [] }
   → columnFilteringFeature → { columnFilters: [] }
   → rowPaginationFeature → { pagination: { pageIndex: 0, pageSize: 10 } }
   → rowSelectionFeature → { rowSelection: {} }
   ...
```

structuredClone으로 깊은 복사를 하는 이유는, 여러 테이블 인스턴스가 초기 상태 객체를 공유하며 서로 오염시키는 것을 방지하기 위해서입니다.

💡 왜 Redux처럼 단일 리듀서가 아닌가?

TanStack Query는 Query 내부에서 #dispatch + reducer 패턴을 사용했지만, TanStack Table은 @tanstack/store의 setState를 직접 호출하는 방식입니다.
이유는 테이블의 상태 전환이 Query의 fetch lifecycle(pending → success → error)처럼 예측 가능한 FSM이 아니라, 사용자 인터랙션에 의해 어떤 순서로든 일어날 수 있기 때문입니다.각 Feature의 makeStateUpdater가 생성하는 콜백은 결국 이런 형태로, 특정 상태 키만 업데이트하는 단순한 패턴입니다.

```tsx
(updater) => {
  table.baseStore.setState((old) => ({
    ...old,
    [key]: functionalUpdate(updater, old[key]),
  }));
};
```

- Step 4 — useStore로 React 구독 연결
  코드 (useTable.ts:112-118)

```tsx
const selectorWithDataAndColumns = (state) => ({
  columns: tableOptions.columns,
  data: tableOptions.data,
  ...selector(state),
});

const state = useStore(table.store, selectorWithDataAndColumns, shallow);
```

TanStack Query가 useSyncExternalStore를 직접 사용한 것과 달리, TanStack Table은 @tanstack/react-store의 useStore를 사용합니다. 내부적으로는 동일하게 useSyncExternalStore 기반이지만, selector + shallow equality 패턴이 내장되어 있습니다.

💡 selector의 역할이 핵심적인 이유
useTable의 두 번째 인자로 selector를 전달할 수 있습니다:
`const table = useTable(options, (state) => ({
  sorting: state.sorting,
}))`
이렇게 하면 sorting이 변할 때만 리렌더됩니다. pagination이 변해도 이 컴포넌트는 리렌더되지 않습니다.

selector를 넘기지 않으면 () => ({}) 가 기본값이므로, data와 columns의 참조가 변할 때만 리렌더됩니다. 즉 기본적으로 테이블 상태 변화에 대한 리렌더를 하지 않는 구조이며, 필요한 상태만 명시적으로 구독해야 합니다. 이것은 성능을 위한 의도적 설계입니다.

### 메모리 최적화(feat. 프로토타입 공유) Step 5

1만 개의 행과 10개의 열이 있으면 셀이 10만 개입니다. 10만 개의 객체 안에 각각 `getValue()` 같은 함수를 다 따로 만들어 넣으면 브라우저 메모리가 어떻게 될까? → 응 터짐.

⇒ TanStack은 `Object.create(cellPrototype)`을 씁니다. 즉, 함수는 프로토타입에 딱 하나만 만들어두고 10만 개의 셀은 그 설계도를 쳐다보게만(공유하게) 만듦

- Step 5 — Prototype 공유: 메모리 효율의 비밀
  10,000행 × 10열이면 100,000개의 Cell 인스턴스가 생성됩니다. 각 Cell에 메서드를 매번 할당하면 메모리가 폭발합니다.

```tsx
// 코드 (constructCell.ts:13-24)

function getCellPrototype(table) {
  if (!table._cellPrototype) {
    table._cellPrototype = { table };
    for (const feature of Object.values(table._features)) {
      feature.assignCellPrototype?.(table._cellPrototype, table);
    }
  }
  return table._cellPrototype;
}

// 코드 (constructCell.ts:26-49)

export function constructCell(column, row, table) {
  const cellPrototype = getCellPrototype(table); // 캐시된 프로토타입
  const cell = Object.create(cellPrototype); // 프로토타입 상속

  // 인스턴스 고유 프로퍼티만 할당
  cell.column = column;
  cell.id = `${row.id}_${column.id}`;
  cell.row = row;

  return cell;
}
```

Object.create(cellPrototype)으로 생성하면 100,000개의 Cell이 하나의 프로토타입 객체를 공유합니다. cell.getValue() 같은 메서드는 프로토타입 체인을
타고 올라가서 찾습니다. Row, Column, Header도 동일한 패턴입니다.

💡 일반 클래스 인스턴스와 뭐가 다른가?

JavaScript의 class로 만든 인스턴스도 프로토타입을 공유합니다. 하지만 TanStack Table이 Object.create를 직접 사용하는 이유는 Feature별로 동적으로 프로토타입을 조립해야 하기 때문입니다. 어떤 Feature가 활성화되었느냐에 따라 프로토타입에 붙는 메서드가 달라집니다. rowSortingFeature를 빼면 column.getIsSorted()가 프로토타입에 존재하지 않습니다. 이는 정적 클래스 상속으로는 불가능한 구조입니다.

### 파이프라인(마치 정수기 필터같은..) Step6 ~ Step10

- **파이프라인 흐름 설명:**
  1. **흙탕물 (원본 데이터):** `getCoreRowModel`로 들어옵니다.
  2. **1차 필터 (Filtering):** 검색어에 안 맞는 데이터를 걸러냅니다 (`getFilteredRowModel`).
  3. **2차 정렬 (Sorting):** 남은 물을 깨끗한 순서대로 정렬합니다 (`getSortedRowModel`).
  4. **3차 컵에 따르기 (Pagination):** 정렬된 물을 10개씩 잘라서 현재 페이지의 컵에 담습니다 (`getPaginatedRowModel`).
- 추가로 해당 과정에서 이미 계산된 결과를 기억(`memoDeps`)하고 있다가 재활용해서 브라우저가 버벅거리지 않음
- Step 6 — Row Model Pipeline: 데이터 변환 체인
  이것이 TanStack Table의 진짜 핵심이라고 할 수 있습니다.

table.getRowModel()을 호출하면, 내부적으로 여러 Row Model이 체인 형태로 연결되어 순차 실행됩니다

```markdown
options.data (원본 배열)
│
▼
getCoreRowModel() ← data[] → Row 인스턴스 변환
│
▼
getFilteredRowModel() ← columnFilters/globalFilter 적용
│
▼
getGroupedRowModel() ← 컬럼 그룹핑 (선택적)
│
▼
getSortedRowModel() ← sorting 상태 기반 정렬
│
▼
getExpandedRowModel() ← 하위 행 확장 (선택적)
│
▼
getPaginatedRowModel() ← pageIndex/pageSize로 슬라이싱
│
▼
최종 RowModel { rows, flatRows, rowsById }
```

각 단계는 이전 단계의 결과를 입력으로 받습니다. 예를 들어 getSortedRowModel은 getPreSortedRowModel()을 호출해 필터링이 끝난 결과를 받습니다.

코드 (createSortedRowModel.ts:21-29)

```tsx
return tableMemo({
  feature: "rowSortingFeature",
  table,
  fnName: "table.getSortedRowModel",
  memoDeps: () => [table.store.state.sorting, table.getPreSortedRowModel()],
  fn: () => _createSortedRowModel(table),
  onAfterUpdate: () => table_autoResetPageIndex(table),
});
```

💡 tableMemo가 하는 일
TanStack Query의 notifyManager.batch()가 리렌더를 배칭한다면, TanStack Tabled의 tableMemo는 연산 결과를 메모이제이션합니다.
memoDeps로 지정된 의존성(sorting 상태, 이전 Row Model)이 변하지 않으면 이전 계산 결과를 그대로 반환합니다. 10,000행 정렬 연산을 sorting이 변할 때만 수행하는 것이 이 메모이제이션 덕분입니다. `onAfterUpdate: () => table_autoResetPageIndex(table)`도 중요합니다. 정렬이 바뀌면 페이지 인덱스를 자동으로 0으로 리셋합니다. 3페이지에서 정렬을 바꿨는데 여전히 3페이지에 머무르면 사용자가 혼란스러워하기 때문입니다.

- Step 7 — Feature 구현 패턴: 정렬을 예시로
  하나의 Feature가 어떻게 구성되는지 rowSortingFeature로 살펴봅니다.

```tsx
// 코드 (rowSortingFeature.ts:50-131)

export function constructRowSortingFeature() {
  return {
    // ① 초기 상태 추가
    getInitialState(initialState) {
      return { sorting: [], ...initialState };
    },

    // ② 컬럼 기본값 설정
    getDefaultColumnDef() {
      return { sortFn: "auto", sortUndefined: 1 };
    },

    // ③ 테이블 기본 옵션 설정
    getDefaultTableOptions(table) {
      return {
        onSortingChange: makeStateUpdater("sorting", table),
        isMultiSortEvent: (e) => e.shiftKey, // Shift+클릭 = 다중 정렬
      };
    },

    // ④ Column 프로토타입에 API 부착
    assignColumnPrototype(prototype, table) {
      assignPrototypeAPIs("rowSortingFeature", prototype, table, {
        column_toggleSorting: {
          fn: (column, desc, multi) =>
            column_toggleSorting(column, desc, multi),
        },
        column_getIsSorted: {
          fn: (column) => column_getIsSorted(column),
        },
        // ... 12개의 column API
      });
    },

    // ⑤ Table에 직접 API 부착
    constructTableAPIs(table) {
      assignTableAPIs("rowSortingFeature", table, {
        table_setSorting: {
          fn: (updater) => table_setSorting(table, updater),
        },
        table_resetSorting: {
          fn: (defaultState) => table_resetSorting(table, defaultState),
        },
      });
    },
  };
}
```

💡 assignPrototypeAPIs vs assignTableAPIs

- assignPrototypeAPIs: Row, Column, Cell, Header에 사용. 공유 프로토타입에 메서드를 붙여 메모리 효율적. 메모이제이션이 필요하면 인스턴스별로 lazy하게 memo 상태를 생성
- assignTableAPIs: Table 인스턴스에 직접 부착. Table은 싱글톤이므로 공유할 필요 없음
- Step 8 — 정렬의 실제 동작: createSortedRowModel코드 (createSortedRowModel.ts:74-129)

```tsx
const sortData = (rows) => {
  // 프로토타입 체인을 보존하면서 복사
  const sortedData = rows.map((row) => {
    const cloned = Object.create(Object.getPrototypeOf(row))
    return Object.assign(cloned, row)
  })

  sortedData.sort((rowA, rowB) => {
    for (const sortEntry of availableSorting) {
      // undefined 값 특수 처리
      if (sortUndefined) { ... }

      // 실제 정렬 함수 실행
      sortInt = columnInfo.sortFn(rowA, rowB, sortEntry.id)

      if (sortInt !== 0) {
        if (isDesc) sortInt *= -1          // 내림차순
        if (invertSorting) sortInt *= -1   // 반전 옵션
        return sortInt
      }
    }
    return rowA.index - rowB.index // 동일하면 원래 순서 유지 (stable sort)
  })

  // 하위 행(subRows)도 재귀적으로 정렬
  sortedData.forEach((row) => {
    if (row.subRows.length) {
      row.subRows = sortData(row.subRows)
    }
  })

  return sortedData
}

```

- 핵심 포인트들:
  - 다중 정렬: availableSorting 배열을 순회하며, 첫 번째 기준이 동일(0)하면 다음 기준으로 넘어감
  - Stable sort: 모든 기준이 동일하면 rowA.index - rowB.index로 원래 순서 보존
  - 프로토타입 보존 복사: Object.create(Object.getPrototypeOf(row))로 복사해야 row.getValue() 같은 프로토타입 메서드가 유지됨
  - 재귀 정렬: 트리 구조 데이터(subRows)도 동일한 기준으로 정렬

💡 sortFn: 'auto'는 어떻게 동작하는가?
column_getAutoSortFn은 해당 컬럼의 첫 번째 행 값을 보고 타입을 자동 감지합니다. 문자열이면 alphanumeric 정렬, 숫자면 basic(단순 비교) 정렬 함수를 반환합니다. 이 감지는 컬럼별로 1회만 수행되고 캐시됩니다.

- Step 9 — Header 빌드: Bottom-Up 재귀
  그룹 헤더(여러 컬럼을 묶는 상위 헤더)가 있는 경우:

```tsx
// 컬럼 정의:
group('Name', columns: ['firstName', 'lastName'])
accessor('age')

// 결과 헤더 구조:
depth 0: [  Name (colSpan=2)  ] [ age (rowSpan=2) ]
depth 1: [ firstName ] [ lastName ]
```

빌드 과정:

1. 전체 컬럼 트리에서 최대 depth를 찾음
2. 리프 컬럼들로 가장 하단 헤더 행을 먼저 생성
3. Bottom-up으로 부모 헤더를 재귀 구성 → 같은 부모를 가진 헤더들은 하나의 그룹으로 합침
4. colSpan/rowSpan을 재귀적으로 계산하여 HTML <th> 속성에 바로 사용 가능하게 만듦

- Step 10 — 상태 변경이 화면에 반영되기까지

사용자가 컬럼 헤더를 클릭해서 정렬을 토글하면

```tsx
column.toggleSorting()
  └─ column_toggleSorting(column, desc, multi)
     └─ table.setSorting(updater)
        └─ makeStateUpdater('sorting', table)
           └─ table.baseStore.setState((old) => ({
              ...old,
              sorting: newSorting,
	            }))
              └─ table.store가 baseStore 변화를 감지
                   └─ useStore의 구독 콜백 실행
                      └─ selector로 관련 상태만 비교
                         └─ shallow 비교에서 변경 감지
                            └─ React 리렌더
                               └─ table.getRowModel() 호출
                                  └─ getSortedRowModel()의
                                     memoDeps 변경 감지
                                       └─ 정렬 재계산
```

💡 TanStack Query와의 비교

```markdown
|               | TanStack Query                         | TanStack Table                     |
| ------------- | -------------------------------------- | ---------------------------------- |
| 상태 위치     | QueryCache (전역 캐시)                 | table.baseStore (인스턴스별)       |
| React 연동    | useSyncExternalStore 직접              | useStore (@tanstack/react-store)   |
| 리렌더 최적화 | notifyOnChangeProps (사용한 필드 추적) | selector + shallow (명시적 선택)   |
| 캐싱          | staleTime/gcTime 기반 시간 캐시        | tableMemo 의존성 기반 메모이제이션 |
| 핵심 패턴     | Observer 패턴 (Subscribable)           | Feature 플러그인 패턴              |

14개 Stock Feature 한눈에 보기

┌─────────────────────────┬────────────────────────────────────────┬─────────────────────────────────────────────────┬─────────────────────────┐
│ Feature │ 상태 │ 주요 API │ Row Model │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ rowSortingFeature │ sorting: SortingState │ column.toggleSorting(), column.getIsSorted() │ createSortedRowModel │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ columnFilteringFeature │ columnFilters: ColumnFiltersState │ column.setFilterValue(), column.getCanFilter() │ createFilteredRowModel │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ globalFilteringFeature │ globalFilter: any │ table.setGlobalFilter() │ (필터 Row Model 공유) │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ rowPaginationFeature │ pagination: { pageIndex, pageSize } │ table.nextPage(), table.getPageCount() │ createPaginatedRowModel │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ rowSelectionFeature │ rowSelection: Record<string, boolean> │ row.toggleSelected(), │ - │
│ │ │ table.getSelectedRowModel() │ │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ columnVisibilityFeature │ columnVisibility: Record<string, │ column.toggleVisibility(), │ - │
│ │ boolean> │ column.getIsVisible() │ │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ columnOrderingFeature │ columnOrder: string[] │ table.setColumnOrder() │ - │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ columnPinningFeature │ columnPinning: { left, right } │ column.pin(), table.getLeftHeaderGroups() │ - │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ columnSizingFeature │ columnSizing: Record<string, number> │ column.getSize(), header.getSize() │ - │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ columnResizingFeature │ columnResizing: { ... } │ header.getResizeHandler() │ - │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ columnGroupingFeature │ grouping: string[] │ column.toggleGrouping() │ createGroupedRowModel │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ rowExpandingFeature │ expanded: ExpandedState │ row.toggleExpanded(), row.getIsExpanded() │ createExpandedRowModel │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ rowPinningFeature │ rowPinning: { top, bottom } │ row.pin(), table.getTopRows() │ - │
├─────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────┼─────────────────────────┤
│ columnFacetingFeature │ - (캐시 기반) │ column.getFacetedUniqueValues() │ createFacetedRowModel │
└─────────────────────────┴────────────────────────────────────────┴─────────────────────────────────────────────────┴─────────────────────────┘
```

# 정리: TanStack Table이 해결하는 문제의 본질

TanStack Query가 **서버 데이터의 비동기 상태**라는 문제를 해결했다면, TanStack Table은 **구조화된 데이터의 뷰 변환**이라는 문제를 해결합니다.

> 결국 TanStack Query가 '서버에서 데이터를 어떻게 잘 가져와서(비동기) 캐싱할까?'를 고민한다면, TanStack Table은 '가져온 이 방대한 데이터를 브라우저에서 어떻게 필터링하고 정렬해서 보여줄까?와 같다고 할 수 있음

원본 데이터 배열을 필터 → 그룹 → 정렬 → 페이지네이션이라는 파이프라인을 거쳐 "지금 화면에 보여야 할 행"으로 변환하는 것, 그리고 그 각 단계의 상태를 선언적으로 관리하는 것이 핵심입니다.
`useTable({ _features, data, columns })` 이 한 줄이 실행되는 동안, 내부에서는 Feature 조립 → Store 생성 → Prototype 빌드 → Row Model 파이프라인 메모이제이션 이라는 정교한 시스템이 작동합니다. 개발자는 데이터와 컬럼을 선언하고, 어떤 기능이 필요한지 Feature로 명시하기만 하면 됩니다.
