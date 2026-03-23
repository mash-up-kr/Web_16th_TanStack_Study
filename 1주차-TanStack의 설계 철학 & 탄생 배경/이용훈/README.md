# 1주차 - TanStack의 설계 철학 & 탄생 배경

## 1. 이게 무엇인지

---

### 1-1. TanStack이란?

TanStack은 **Tanner Linsley**가 만든 오픈소스 JavaScript/TypeScript 라이브러리 생태계다.
단일 라이브러리가 아니라, 공통된 설계 철학을 공유하는 **라이브러리들의 모음(ecosystem)** 이다.

> 공식 정의 ([tanstack.com](https://tanstack.com)):
> *"Headless, type-safe, & powerful utilities for Web Applications, Routing, State Management, Data Visualization, Datagrids/Tables, and more."*
> → 웹 애플리케이션, 라우팅, 상태 관리, 데이터 시각화, 테이블/데이터그리드 등을 위한 Headless하고 타입 안전한 고성능 유틸리티 모음
 

---

### 1-2. 이름의 유래

"TanStack"이라는 이름은 처음부터 계획된 게 아니었다.  
Tanner Linsley의 친구 **Shaun Wang**이 농담처럼 "너만의 스택을 만들고 있네, TanStack이네"라고 했고,  
처음엔 어색했지만 Jamstack처럼 들린다는 생각에 그대로 채택했다.

> *"One of my friends at the time, Shaun Wang, he was just joking around that I was building my own stack and called it the TanStack."* 
> → "그 당시 친구였던 Shaun Wang이 농담 삼아 내가 나만의 스택을 만들고 있다며 TanStack이라고 불렀다."   
> — Tanner Linsley ([The New Stack 인터뷰](https://thenewstack.io/tanstack-introduces-new-meta-framework-based-on-its-router/))

---

### 1-3. 탄생 배경

Tanner Linsley는 자신이 공동창업한 스타트업 **[Nozzle.io](https://nozzle.io)** (Google 검색 키워드 순위 추적 툴)를 개발하면서 실질적인 문제에 부딪혔다.

- **대량의 데이터를 테이블로 표시해야 했는데**, Angular 생태계에서 마땅한 도구가 없었다 → **React Table** 탄생
- **Redux로 API 데이터를 관리했는데**, 결국 API 응답을 저장하는 캐시 역할밖에 못 했다 → **React Query** 탄생
- Nozzle.io가 직접 유지보수할 수 없을 정도로 커지자 **오픈소스화** 결정

즉, **실제 프로덕션 문제를 해결하기 위해** 만들어진 라이브러리들이다. 해피패스 데모용이 아니다.

---

### 1-4. 현재 TanStack 생태계 (2026년 기준)

| 라이브러리 | 역할 | 상태 |
|---|---|---|
| **TanStack Query** | 서버 상태 관리 / 데이터 패칭 | Stable |
| **TanStack Router** | 타입 안전 라우터 | Stable |
| **TanStack Table** | Headless 테이블/데이터그리드 | Stable |
| **TanStack Form** | 폼 상태 관리 | Stable |
| **TanStack Virtual** | 가상 스크롤 | Stable |
| **TanStack Start** | 풀스택 메타프레임워크 (React) | RC |
| **TanStack Store** | 프레임워크 agnostic 상태 저장소 | Stable |
| **TanStack DB** | 클라이언트 리액티브 DB | Beta |
| **TanStack AI** | AI SDK (멀티 프로바이더) | Alpha |
| **TanStack Pacer** | Debounce / Throttle / Rate Limiting | Beta |
| **TanStack Devtools** | 개발자 도구 패널 | Alpha |

지원 프레임워크: **React, Vue, Solid, Svelte, Angular, Lit, Qwik** (라이브러리마다 상이)

---

## 2. 어떻게 사용하는지

---

### 2-1. 핵심 사용 방식: 각 라이브러리는 독립적으로, 혹은 함께

TanStack의 라이브러리들은 **독립적으로 사용 가능**하다.  
Query만 쓰고 Router는 쓰지 않아도 된다. 그러나 함께 쓰면 시너지가 크다.

```
TanStack Start (풀스택)
  └── TanStack Router (라우팅)
        └── TanStack Query (서버 상태)
              └── TanStack Table (데이터 표시)
                    └── TanStack Form (입력 처리)
```

---

### 2-2. 프레임워크별 패키지 설치 방식

모든 TanStack 라이브러리는 **프레임워크별 어댑터**를 분리해서 제공한다.  
코어 로직은 프레임워크에 무관하고, 어댑터가 연결을 담당한다.

```bash
# React에서 Query 사용
npm install @tanstack/react-query

# Vue에서 Query 사용
npm install @tanstack/vue-query

# React에서 Table 사용
npm install @tanstack/react-table
```

---

### 2-3. Headless 방식이란?

TanStack의 핵심 사용 패턴은 **Headless UI**다.  
라이브러리는 **로직과 상태만** 제공하고, **UI 렌더링은 개발자가** 직접 한다.

```tsx
// TanStack Table 예시 - 마크업과 스타일은 100% 개발자 몫
const table = useReactTable({
  data,
  columns,
  getCoreRowModel: getCoreRowModel(),
  getSortedRowModel: getSortedRowModel(),
})

return (
  <table>  {/* 어떤 마크업이든 자유롭게 */}
    {table.getHeaderGroups().map(headerGroup => (
      <tr key={headerGroup.id}>
        {headerGroup.headers.map(header => (
          <th key={header.id}>
            {flexRender(header.column.columnDef.header, header.getContext())}
          </th>
        ))}
      </tr>
    ))}
  </table>
)
```

→ Shadcn/ui, 사내 디자인 시스템, Tailwind 등 **어떤 스타일 레이어와도 자유롭게 조합** 가능하다.

---

### 2-4. 점진적 도입 (Progressive Adoption)

공식 Product Tenets에 명시된 원칙 중 하나다:

> *"Every library should be adoptable one piece at a time, without rewrites or hard coupling to the rest of TanStack."*
> → "모든 라이브러리는 전체를 재작성하거나 다른 TanStack 라이브러리에 강하게 결합되지 않고도, 하나씩 점진적으로 도입할 수 있어야 한다."  
> ([tanstack.com/tenets](https://tanstack.com/tenets))

기존 프로젝트에 한 번에 전부 교체할 필요가 없다.  
Query만 먼저 붙이고, 나중에 Router를 붙이는 식의 점진적 도입이 가능하도록 설계되어 있다.

---

## 3. 왜 써야하는지

---

### 3-1. TanStack의 4가지 핵심 설계 철학

TanStack의 모든 라이브러리는 아래 4가지 원칙을 공유한다.  
이게 단순한 마케팅이 아니라 라이브러리 구조 자체에 녹아있다.

#### ① Headless

> 로직과 UI를 분리한다. 디자인 시스템에 종속되지 않는다.

- 컴포넌트가 마크업을 강제하지 않는다
- 어떤 CSS 프레임워크, 어떤 디자인 시스템과도 조합 가능
- MUI, Shadcn, Tailwind, 사내 컴포넌트 → 아무거나 써도 됨

#### ② Framework-Agnostic

> React에 종속되지 않는다. 프레임워크 전환 비용이 낮다.

- 코어 로직은 순수 TS/JS
- 어댑터 레이어만 교체하면 React → Vue → Solid 이전 가능
- 팀이 쓰는 프레임워크가 바뀌어도 **사고 방식을 다시 배울 필요가 없다**

#### ③ Type-Safe

> TypeScript를 나중에 추가한 게 아니라, 처음부터 TypeScript를 중심으로 설계했다.

```tsx
// TanStack Router - URL 파라미터 타입이 자동 추론됨
const Route = createFileRoute('/posts/$postId')({
  component: PostComponent,
})

function PostComponent() {
  const { postId } = Route.useParams()
  // postId는 string 타입으로 정확하게 추론됨
}
```

> *"Type safety as leverage, not dogma: TypeScript support should guide correct use and catch real bugs without drowning users in generics."*  
> ([tanstack.com/tenets](https://tanstack.com/tenets))

#### ④ Dependency-Light / Primitives-First

> 거대한 올인원 프레임워크가 아니라, 작고 조합 가능한 빌딩블록을 제공한다.

- 각 라이브러리는 최소한의 의존성만 가짐
- 필요한 것만 번들에 포함됨
- 어떤 레이어든 "Escape hatch"(우회로)가 존재함

---

### 3-2. React 생태계의 파편화 문제를 해결한다

기존 React 생태계는 라이브러리마다 패턴이 제각각이다.

| 문제 | 기존 생태계 | TanStack |
|---|---|---|
| 상태 관리 패턴 | Redux(selector), Zustand(store), Recoil(atom) 제각각 | 일관된 API |
| 타입 지원 | 나중에 추가된 경우 많음 | 설계 단계부터 TypeScript |
| 프레임워크 전환 | 라이브러리 통째로 교체 필요 | 어댑터만 교체 |
| UI 스타일 종속 | 내장 스타일 강제하는 경우 있음 | Headless로 완전 분리 |

---

### 3-3. 실제 프로덕션 문제에서 나온 라이브러리

TanStack이 해결한 핵심 개념 중 하나는 **"서버 상태 vs 클라이언트 상태"의 구분**이다.

기존에는 Redux로 API 응답 데이터와 UI 상태를 함께 관리했다.  
TanStack Query는 이 둘을 명확히 분리했다:

- **서버 상태** (API 데이터, 캐싱, 동기화, 재검증) → TanStack Query
- **클라이언트 상태** (UI 모달, 선택 여부 등) → Zustand, Jotai 등

이 개념적 분리가 프론트엔드 상태 관리 논의를 바꿨고, 이후 많은 라이브러리들이 같은 방향으로 따라갔다.

---

### 3-4. 독립성과 지속 가능성

> *"TanStack is independently operated with no controlling interests over our technical direction and no hidden agendas."*
> → "TanStack은 기술 방향에 대한 통제 지분도, 숨겨진 의도도 없이 독립적으로 운영된다."
> ([tanstack.com/ethos](https://tanstack.com/ethos))

- 투자자나 대기업의 기술 방향 통제를 받지 않음
- 파트너사도 코어 기술 방향에 영향을 줄 수 없음
- "성장을 위한 성장"이 아니라 **지속 가능한 개발자 경험 개선**이 목표

Vercel(Next.js), Meta(React) 같은 거대 플랫폼에 종속되지 않는 독립적인 생태계를 지향한다.

---

### 3-5. 요약: 언제 TanStack을 선택해야 하는가?

| 상황 | TanStack이 적합한 이유 |
|---|---|
| 사내 디자인 시스템이 있는 경우 | Headless로 완전히 통합 가능 |
| 프레임워크 전환 가능성이 있는 경우 | 어댑터만 바꾸면 됨 |
| TypeScript를 진지하게 쓰는 팀 | 설계부터 타입 안전 |
| 서버 상태 관리가 복잡한 앱 | Query의 캐싱/동기화 추상화 |
| 대량 데이터 테이블이 필요한 경우 | Virtual + Table 조합 |

---

## 참고 자료

- [TanStack 공식 사이트](https://tanstack.com)
- [TanStack Ethos](https://tanstack.com/ethos) — 조직 철학
- [TanStack Product Tenets](https://tanstack.com/tenets) — 기술 원칙
- [The New Stack 인터뷰 — TanStack 탄생 배경](https://thenewstack.io/tanstack-introduces-new-meta-framework-based-on-its-router/)
- [Callstack Podcast — Tanner Linsley 인터뷰](https://www.callstack.com/podcasts/exploring-tanstack-ecosystem-with-tanner-linsley)
- [GitHub — TanStack 전체 레포](https://github.com/tanstack)