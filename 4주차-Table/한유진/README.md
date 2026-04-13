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

TanStack Table은 배열 데이터를 그대로 렌더링하지 않는다. 데이터를 테이블이라는 구조에 맞게 재해석하는 과정을 거친다.

`Row Model 생성 (getCoreRowModel)`

일반적인 배열 렌더링은 데이터를 `map()`으로 순회하며 JSX를 반환하는 게 전부다. 하지만 TanStack Table은 원본 데이터를 그대로 쓰지 않고, 먼저 내부적으로 Row Model이라는 구조체로 변환한다.

실제 코드(`createCoreRowModel.ts`)를 보면:

```tsx
const rowModel = {
  rows: [],       // 계층 구조를 유지한 row 배열
  flatRows: [],   // 모든 row를 평탄화한 배열
  rowsById: {},   // id로 즉시 접근하기 위한 맵
}
```

각 row는 단순한 데이터 객체가 아니라, id, index, depth, parentId, subRows 같은 메타데이터를 포함한 구조체로 변환된다. 그리고 모든 row는 테이블 레벨의 프로토타입을 공유해 메모리 효율도 높인다(`constructRow.ts`). 이 덕분에 하위 row가 있는 중첩 데이터(트리 구조)도 자연스럽게 표현할 수 있고, `table.getRow(id)`처럼 id로 즉시 특정 row에 접근하는 것도 가능해진다.

`Accessor: 컬럼이 데이터를 읽는 방식`

컬럼 정의에서 `accessorKey`나 `accessorFn`을 지정하면, 테이블이 각 row에서 해당 컬럼의 값을 어떻게 추출할지를 선언하게 된다.

`// 단순 키 접근` `\{ accessorKey: 'firstName' \}`

`// 커스텀 변환 함수` `\{ accessorFn: (row) =\> \`${row.firstName} ${row.lastName}` }`

이렇게 정의된 accessor는 단순히 값을 꺼내는 것에서 끝나지 않는다. 이 값이 나중에 정렬, 필터, 그룹핑의 기준값이 된다. 즉, 데이터 해석의 첫 번째 관문이다.

---

## 데이터를 어떻게 변형하는가?

TanStack Table의 핵심 강점 중 하나는 데이터를 여러 단계를 거쳐 가공하는 파이프라인 구조를 내장하고 있다는 점이다.

원본 데이터 → 필터링 → 정렬 → 그룹핑 → 페이지네이션

각 단계는 독립적인 Row Model로 처리된다.

`필터링 (getFilteredRowModel)`

`state: \{ columnFilters \}`

각 컬럼에 `filterFn`을 지정하면, TanStack Table이 자동으로 조건에 맞는 row만 추려낸다. 내부적으로 `columnFilters` 상태를 참조해 필터링된 Row Model을 새로 생성한다. 기본 제공되는 `includesString`, `equals`, `inNumberRange` 같은 내장 필터 함수 외에도 커스텀 함수를 자유롭게 연결할 수 있다.

`정렬 (getSortedRowModel)`

`state: \{ sorting \}`

`sorting` 상태는 어떤 컬럼 기준으로, 어떤 방향(오름/내림)으로 정렬할지를 담고 있다. 실제 코드(`rowSortingFeature.ts`)를 보면 정렬 상태를 초기화하고, 컬럼마다 자동으로 적합한 정렬 함수(`getSortFn`)를 연결하는 로직이 있다. 문자열, 숫자, 날짜 등 타입에 따라 알맞은 기본 정렬 함수가 자동 선택되고, 필요하면 커스텀 정렬 함수도 지정할 수 있다.

`페이지네이션 (getPaginationRowModel)`

`state: \{ pagination: \{ pageIndex, pageSize \} \}`

전체 데이터 중 현재 페이지에 해당하는 subset만 추출한다. `table.nextPage()`, `table.previousPage()`, `table.setPageSize()` 같은 메서드가 내부적으로 pagination 상태를 업데이트하고, 그 상태에 따라 보여줄 row 범위가 다시 계산된다. 단순해 보이지만, 앞선 필터링과 정렬 이후의 결과를 기준으로 페이지네이션이 적용되기 때문에 파이프라인의 가장 마지막에 위치한다.

`그룹핑 + Aggregation`

`aggregationFn: 'sum'`

특정 컬럼 기준으로 row를 묶고, 그룹 단위로 합계, 평균 같은 집계 연산을 수행한다. 단순한 표 렌더링을 넘어서 "부서별 총 매출", "카테고리별 평균 수량" 같은 데이터 요약이 필요할 때 유용하다.

---

## 사용자와 어떻게 상호작용하는가?

TanStack Table은 UI를 직접 제공하지 않는다. 버튼도 없고, 입력창도 없다. 그러나 사용자 인터랙션에 필요한 상태와 핸들러는 전부 제공한다.

이 차이가 중요하다. TanStack Table은 "무엇을 클릭하면 어떤 상태가 바뀌어야 한다"는 로직은 알고 있지만, 그 클릭 가능한 요소를 어떻게 생긴 버튼으로 만들지는 개발자가 결정한다.

`정렬 인터랙션`

`column.getToggleSortingHandler()`

이 함수는 헤더에 `onClick`으로 연결할 수 있는 핸들러를 반환한다. 클릭할 때마다 오름차순 → 내림차순 → 해제 순서로 상태가 순환한다. `column.getIsSorted()`로 현재 정렬 상태를 읽어서 UI에 화살표 아이콘을 표시하는 것도 가능하다.

`\<th onClick=\{column.getToggleSortingHandler()\}\> 이름 \{column.getIsSorted() === 'asc' ? '▲' : '▼'\} \</th\>`

`필터 입력`

`column.setFilterValue(value)`

`column.getFilterValue()`

입력창의 `onChange`에 `column.setFilterValue()`를 연결하면 된다. 상태 관리 코드를 별도로 작성할 필요 없이, TanStack Table이 `columnFilters` 상태를 자동으로 업데이트하고 필터링된 결과를 다시 계산해준다.

`행 선택, 확장, 컬럼 가시성 등`

정렬과 필터 외에도 `row.getToggleSelectedHandler()`, `row.getToggleExpandedHandler()`, `column.toggleVisibility()` 같은 핸들러들이 모든 인터랙션 시나리오를 커버한다. 개발자는 이 핸들러들을 원하는 UI 요소에 연결하기만 하면 된다.

---

## 정리: TanStack Table은 무엇을 책임지는가?

결국 세 가지로 정리할 수 있다.

- 데이터 해석 → 원본 배열을 row/cell/column 구조체로 모델링
- 데이터 변형 → 필터 → 정렬 → 그룹 → 페이지네이션 파이프라인
- 사용자 상호작용 → 인터랙션에 필요한 상태와 핸들러 일괄 제공

TanStack Table은 데이터 → 상태 → UI로 이어지는 흐름에서 "데이터와 상태"를 전담하는 엔진이다. UI는 개발자가 만들고, 로직은 TanStack Table이 책임지는 구조다.

이 분리가 중요한 이유는, 같은 테이블 로직 위에 완전히 다른 UI를 얹을 수 있기 때문이다. 디자인 시스템이 바뀌어도 로직은 그대로 재사용하고, 반대로 정렬·필터·페이지네이션 같은 복잡한 기능을 추가할 때 UI 컴포넌트를 건드릴 필요가 없다. UI와 로직이 명확히 분리된 구조 덕분에 각자의 역할에만 집중할 수 있다.

단순히 데이터를 표로 보여주기만 하면 TanStack Table은 과할 수 있다. 하지만 정렬, 필터링, 페이지네이션, 그룹핑, 행 선택처럼 테이블의 상태가 복잡해지는 순간, TanStack Table이 그 복잡성을 대신 감당해준다!