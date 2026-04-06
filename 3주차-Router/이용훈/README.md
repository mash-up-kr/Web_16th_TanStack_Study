# 서치 파라미터를 상태로 — TanStack Router의 URL-as-State

필터, 정렬, 페이지네이션을 구현할 때 대부분 `useState`로 시작한다.
그런데 새로고침하면 상태가 날아가고, URL을 공유하면 필터가 초기화된다.
그래서 `useSearchParams`로 넘어가는데, 이번엔 타입이 전부 `string`이라 파싱을 손수 짜야 한다.

TanStack Router는 이 문제를 다르게 접근한다.
**URL의 서치 파라미터 자체가 타입 안전한 상태**라는 철학으로 설계되어 있다.

> 이 글은 TanStack Router 공식 문서와 소스코드를 직접 참조했다.

---

## 1. 이게 무엇인지

---

### 1-1. URL-as-State 철학

TanStack Router의 서치 파라미터 설계는 하나의 전제에서 시작한다.

> *"Search params are a first-class citizen of the router."*
> — TanStack Router 공식 문서

`useState`로 관리하는 상태와 URL의 서치 파라미터는 근본적으로 다르다.

| | `useState` | URL 서치 파라미터 |
|---|---|---|
| 새로고침 | 초기화됨 | 유지됨 |
| URL 공유 | 상태 손실 | 상태 포함 |
| 뒤로가기 | 복원 불가 | 복원됨 |
| 타입 | 자유롭게 정의 | 기본적으로 `string`\* |

> \* 브라우저 기본 `URLSearchParams`는 string 중심이다. 하지만 TanStack Router는 다르다.
> 서치 스트링을 먼저 **JSON-friendly하게 파싱**한 뒤 `validateSearch`로 최종 검증/타이핑한다.
> 그래서 `number`, `boolean` 같은 기본 타입은 URL에서 꺼내도 이미 원래 타입으로 복원되어 있다.

TanStack Router는 마지막 행의 단점을 없앤다.
라우트를 정의할 때 서치 파라미터의 **스키마를 함께 선언**하면, 이후 모든 접근에서 타입이 추론된다.

---

### 1-2. 구조: validateSearch

서치 파라미터는 라우트 정의 시점에 `validateSearch` 옵션으로 명시한다.
```typescript
// routes/products.tsx
import { createFileRoute } from '@tanstack/react-router'
import { zodValidator } from '@tanstack/zod-adapter'
import { z } from 'zod'

const productSearchSchema = z.object({
  page: z.number().int().min(1).catch(1),       // 잘못된 값 → 기본값으로 폴백
  sort: z.enum(['price', 'rating', 'newest']).catch('newest'),
  inStock: z.boolean().catch(false),
  q: z.string().optional(),
})

export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(productSearchSchema),
})
```

`validateSearch`는 `(rawSearch: Record<string, unknown>) => T` 형태의 함수를 받는다.
Zod 스키마를 직접 넘기는 것도 가능하지만, **실무에서는 `zodValidator` 어댑터 사용을 권장한다.**
input/output 타입 추론이 어댑터를 통해야 정확하게 동작하기 때문이다.

**`default()`와 `catch()`는 다르다.**

| | 동작 |
|---|---|
| `.default(1)` | 값이 **없을 때** 기본값 제공 |
| `.catch(1)` | 값이 있지만 **유효하지 않을 때** 폴백 |

`?page=abc`처럼 잘못된 값이 들어오면 `.default()`는 검증 에러를 일으킬 수 있다.
URL 파라미터처럼 외부에서 오는 값은 `.catch()`로 방어하는 게 공식 문서의 권장 방식이다.

---

## 2. 어떻게 사용하는지

---

### 2-1. 읽기 — useSearch
```typescript
function ProductList() {
  // 방법 1: Route.useSearch() — 파일 기반 라우팅에서 더 간결
  const { page, sort, inStock, q } = Route.useSearch()

  // 방법 2: useSearch({ from }) — 라우트 경로를 직접 지정
  const { page, sort, inStock, q } = useSearch({ from: '/products' })
  // 타입이 전부 추론됨
}
```

두 방법 모두 동일한 타입 추론을 제공한다.
파일 기반 라우팅 환경에서는 `Route.useSearch()`가 더 간결하고, 라우트 경로 문자열을 직접 관리하지 않아도 된다.

`page`는 `string`이 아니라 `number`로 바로 쓸 수 있다. `parseInt()` 없이.

---

### 2-2. 쓰기 — useNavigate + search

서치 파라미터를 바꿀 때는 `navigate`의 `search` 옵션을 쓴다.
```typescript
function ProductFilter() {
  const navigate = useNavigate({ from: '/products' })
  const search = Route.useSearch()

  const handleSortChange = (sort: 'price' | 'rating' | 'newest') => {
    navigate({
      search: (prev) => ({
        ...prev,
        sort,
        page: 1, // 정렬 변경 시 첫 페이지로 초기화
      }),
    })
  }
  // ...
}
```

`search`에 함수를 넘기면 이전 값(`prev`)을 기반으로 업데이트할 수 있다.
`setState(prev => ...)` 패턴과 동일한 느낌이다. 다만 상태가 URL에 산다.

타입도 강제된다. `sort`에 스키마에 없는 값을 넣으면 TypeScript가 컴파일 타임에 잡는다.

---

### 2-3. Link에서도 타입 안전하게
```typescript
<Link
  to="/products"
  search={{ page: 2, sort: 'price', inStock: true }}
>
  다음 페이지
</Link>
```

`search` prop도 `/products` 라우트의 스키마에 맞게 타입이 체크된다.
`page: 'two'`로 넣으면 TypeScript 에러가 난다.

---

### 2-4. 실전 패턴: 필터 + 페이지네이션 + loader 연동
```typescript
export const Route = createFileRoute('/products')({
  validateSearch: zodValidator(productSearchSchema),
  loaderDeps: ({ search }) => search, // 서치 파라미터가 바뀌면 loader 재실행
  loader: ({ deps }) => fetchProducts(deps),
})
```

`loaderDeps`와 연동하면 서치 파라미터가 바뀔 때마다 `loader`가 자동으로 재실행된다.
필터/정렬/페이지 변경 → URL 업데이트 → loader 재실행 → UI 업데이트.
이 흐름이 명시적이고 예측 가능하다.

---

## 3. 왜 써야 하는지

---

### 3-1. 왜 라우터 레벨에서 검증하는가

URL은 외부 입력이다. 사용자가 직접 조작하거나, 잘못된 링크를 통해 접근할 수 있다.
컴포넌트에서 `useSearchParams()`로 꺼낸 값을 믿고 쓰는 건 사실 위험한 일이다.
```typescript
// 기존 방식 — 위험
const [searchParams] = useSearchParams()
const page = parseInt(searchParams.get('page') ?? '1') // NaN 가능성
const sort = searchParams.get('sort') as 'price' | 'rating' // 단순 캐스팅, 검증 없음
```

`validateSearch`는 이 파싱/검증 책임을 라우터로 올린다.
컴포넌트가 받는 시점엔 이미 안전하다.

`.catch()`를 쓰면 잘못된 값도 조용히 기본값으로 처리되어, 사용자 경험을 해치지 않으면서 방어할 수 있다.

---

### 3-2. 왜 URL이 상태 저장소인가

`useState`는 컴포넌트가 살아있는 동안만 유지된다.
URL은 탭을 닫고 다시 열어도, 다른 사람에게 공유해도 유지된다.

필터가 적용된 상품 목록을 동료에게 공유할 때,
URL 하나로 동일한 화면을 재현할 수 있다는 건 사용자 경험에서 꽤 큰 차이다.

TanStack Router의 접근은 이걸 부가 기능이 아니라 기본 동작으로 만든다.
`useState` 대신 `Route.useSearch()` + `navigate`를 쓰는 것만으로 상태가 URL에 올라가고,
공유 가능하고, 새로고침에 살아남고, 브라우저 히스토리와 통합된다.

---

### 3-3. 가능한 조합 정리

| 상황 | 적절한 상태 저장소 |
|---|---|
| 모달 open/close | `useState` |
| 탭 선택 (URL 반영 불필요) | `useState` |
| 필터, 정렬, 페이지 | URL 서치 파라미터 |
| 검색어 | URL 서치 파라미터 |
| 선택된 리소스 ID | URL 경로 파라미터 |

URL에 올릴 상태와 로컬 상태를 구분하는 기준은 하나다.
**"이 상태가 URL에 있으면 유용한가?"**

---

### 3-4. 핵심 흐름 한 장 요약
```
라우트 정의 시 validateSearch(zodValidator(schema)) 선언
  │
  ├─ URL 접근 시 → JSON-friendly 파싱 → Zod 검증
  │     ├─ 값 없음 → .default() 적용
  │     ├─ 값 유효하지 않음 → .catch() 로 폴백
  │     └─ 타입 안전한 search 객체 완성
  │
  ├─ Route.useSearch() → 컴포넌트에서 타입 추론된 값 접근
  │
  ├─ navigate({ search: prev => ({ ...prev, page: 2 }) })
  │     └─ URL 업데이트 → 브라우저 히스토리 적재 → loaderDeps 감지
  │           └─ loader 재실행 → 컴포넌트 리렌더
  │
  └─ Link search prop → 컴파일 타임 타입 체크
```

---

## 참고 자료

- [Search Params — TanStack Router 공식 문서](https://tanstack.com/router/latest/docs/framework/react/guide/search-params)
- [Type-safe Search Params — TanStack Router 공식 문서](https://tanstack.com/router/latest/docs/framework/react/guide/type-safe-search-params)
- [Zod Adapter — TanStack Router 공식 문서](https://tanstack.com/router/latest/docs/framework/react/guide/search-params#zod-adapter)