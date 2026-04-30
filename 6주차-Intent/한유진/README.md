# TanStack Intent란?

> npm 패키지에 **Agent Skills**를 함께 번들링해 배포하고, AI 코딩 에이전트가 이를 자동으로 발견해 사용할 수 있게 하는 CLI 도구

```bash
npx @tanstack/intent@latest install
```

---

# 왜 만들어졌는가?

TanStack Intent 공식 문서는 이렇게 시작한다.

> "Your docs are good. Your types are solid. Your agent still gets it wrong."

문서도 잘 써있고, 타입도 탄탄하다. 근데 AI 에이전트는 여전히 틀린 코드를 뱉는다. 왜?

## 기존 방식의 3가지 구조적 문제

**① 문서(Docs)는 인간용이다**

문서는 브라우저로 읽는 인간을 대상으로 쓰인다. AI 에이전트가 파싱하기에 최적화되어 있지 않고, 컨텍스트 창에 통째로 넣기엔 너무 길다.

**② 타입(Types)은 의도를 인코딩하지 못한다**

TypeScript 타입은 개별 API 호출의 형태는 검사해주지만, "이 패턴은 언제 쓰고 저 패턴은 언제 쓰는가", "이건 흔한 실수다" 같은 절차적 지식은 담을 수 없다.

**③ 훈련 데이터(Training data)는 과거의 스냅샷이다**

모델은 학습 시점의 생태계를 학습한다. 그 이후 breaking change가 배포되면 모델은 구버전과 신버전 코드를 모두 학습한 상태가 된다. 두 버전을 구분할 방법이 없으므로 영구적으로 split-brain 상태가 된다.

## 기존 대안들도 불완전하다

`CLAUDE.md`, Cursor rules, `.cursorrules` 같은 에이전트용 설정 파일들이 이미 존재한다. 하지만 이것들은 모두 **수동 copy-paste** 방식이다.

```plain text
기존 방식의 문제
 ├── 커뮤니티가 관리하는 rules 파일을 직접 찾아야 함
 ├── 라이브러리가 업데이트돼도 rules 파일은 그대로
 ├── 버전 관리 없음
 └── stale 여부를 알 수 없음
```

TanStack Intent는 이 전달 방식 자체를 바꾼다. **패키지 매니저를 통해** 스킬이 코드와 함께 배포되고, 패키지를 업데이트하면 스킬도 같이 업데이트된다.

---

# Agent Skills란 무엇인가?

> "A skill is a short, versioned document that tells agents how to use a specific capability of your library — correct patterns, common mistakes, and when to apply them."

Agent Skill은 `SKILL.md` 파일 하나다. 라이브러리의 특정 기능을 AI 에이전트가 올바르게 쓰는 방법을 담은 마크다운 문서로, npm 패키지 내부에 포함되어 배포된다.

## SKILL.md 파일 구조

```
@tanstack/query/
├── dist/
├── src/
└── skills/
    ├── fetching/
    │   └── SKILL.md      ← 이렇게 패키지 안에 동봉됨
    ├── mutations/
    │   └── SKILL.md
    └── _artifacts/
```

## SKILL.md 형식

```markdown
---
name: fetching # 필수. 소문자+숫자+하이픈, 디렉토리명과 일치해야 함
description: | # 필수. 에이전트가 "이 스킬이 언제 필요한지" 판단하는 기준
  Fetching server data with TanStack Query. Use when the user needs to
  fetch, cache, or synchronize server state in a React application.
license: MIT # 선택
---

[스킬 본문: 올바른 패턴, 흔한 실수, 언제 어떤 API를 쓸지 등]
```

`description`이 핵심이다. 에이전트는 이 필드만 먼저 읽고 현재 작업에 이 스킬이 필요한지 판단한다. 본문은 필요할 때만 로드된다.

## 실제 SKILL.md 예시 - TanStack Query fetching 스킬

아래는 `@tanstack/query`의 `skills/fetching/SKILL.md`가 실제로 어떻게 생겼는지를 보여주는 예시다.

```markdown
---
name: fetching
description: |
  Fetching and caching server data with TanStack Query. Use when the user
  needs to fetch, cache, or synchronize server state — e.g. API calls,
  loading states, background refetching, or error handling for async data.
license: MIT
---

## 올바른 패턴

서버 데이터를 패칭할 때는 항상 queryOptions()로 옵션을 정의한 뒤 useQuery에 전달한다.
이렇게 하면 prefetch, suspense, 테스트 등 어디서든 같은 옵션을 재사용할 수 있다.

\`\`\`tsx
import { queryOptions, useQuery } from '@tanstack/react-query'

// ✅ queryOptions로 한 곳에서 정의
const postQueryOptions = (id: string) =>
  queryOptions({
    queryKey: ['posts', id],
    queryFn: () => fetch(`/api/posts/${id}`).then(r => r.json()),
    staleTime: 1000 * 60 * 5, // 5분
  })

function PostDetail({ id }: { id: string }) {
  const { data, isPending, isError } = useQuery(postQueryOptions(id))

  if (isPending) return <Spinner />
  if (isError) return <ErrorMessage />
  return <div>{data.title}</div>
}
\`\`\`

## 흔한 실수

### queryKey 배열에 변수를 빠뜨리는 경우

\`\`\`tsx
// ❌ id가 바뀌어도 캐시를 새로 만들지 않음
queryKey: ['post']

// ✅ 의존하는 모든 변수를 포함
queryKey: ['posts', id]
\`\`\`

### v5에서 제거된 onSuccess 콜백 사용

\`\`\`tsx
// ❌ v5에서 제거됨
useQuery({ queryKey: [...], queryFn: ..., onSuccess: (data) => { ... } })

// ✅ useEffect로 대체
const { data } = useQuery({ queryKey: [...], queryFn: ... })
useEffect(() => {
  if (data) { /* side effect */ }
}, [data])
\`\`\`
```

스킬 본문에는 이렇게 **올바른 패턴 + 흔한 실수** 조합이 들어간다. 타입으로는 표현할 수 없는 "왜 이렇게 써야 하는가"가 담기는 곳이다.

### description 작성 기준

```markdown
# Good

description: Extracts text and tables from PDF files, fills PDF forms, and merges
multiple PDFs. Use when working with PDF documents or when the user mentions PDFs,
forms, or document extraction.

# Bad (너무 모호함)

description: Helps with PDFs.
```

---

# 어떻게 동작하는가?

## Progressive Disclosure - 3단계 로딩

에이전트가 모든 스킬을 한꺼번에 로드하면 컨텍스트 창을 낭비한다. Intent는 3단계로 필요한 것만 점진적으로 로드한다.

```plain text
① Discovery (~100 tokens)
   → 시작 시 name + description만 읽음
   → "언제 이 스킬이 필요한가" 파악

② Activation (< 5,000 tokens 권장)
   → 현재 작업이 스킬과 매칭되면 SKILL.md 본문 전체 로드
   → 올바른 패턴, 흔한 실수, 예제 코드 주입

③ Execution (on demand)
   → scripts/, references/, assets/ 등 추가 리소스
   → 필요한 경우에만 로드
```

## 자동 발견 흐름

```plain text
[개발자]
1. npm install @tanstack/query
   → node_modules/@tanstack/query/skills/ 에 스킬 파일 자동 설치

2. npx @tanstack/intent@latest install
   → CLAUDE.md, AGENTS.md, .cursorrules 등에 스킬 로딩 가이던스 자동 추가

[AI 에이전트]
3. 작업 시작
   → description 스캔으로 관련 스킬 발견
   → SKILL.md 본문 로드
   → 해당 라이브러리 버전에 맞는 올바른 코드 생성
```

---

# CLI 명령어 전체 정리

## 소비자(Consumer) 입장

### `intent list` - 스킬 발견

```bash
npx @tanstack/intent@latest list

# 글로벌 패키지 포함
npx @tanstack/intent@latest list --global

# JSON 출력
npx @tanstack/intent@latest list --json
```

현재 프로젝트의 `node_modules`를 스캔해 Intent 지원 패키지와 사용 가능한 스킬 목록을 출력한다. Yarn PnP 환경도 지원한다.

### `intent install` - 에이전트 설정 파일 구성

```bash
npx @tanstack/intent@latest install
```

`AGENTS.md`, `CLAUDE.md`, `.cursorrules` 같은 에이전트 설정 파일에 스킬 로딩 가이던스를 자동으로 추가한다. 기존 내용이 있으면 in-place 업데이트, 없으면 `AGENTS.md`가 기본 대상이 된다.

라이브러리마다 따로 설정할 필요 없이, 이 명령 한 번으로 모든 Intent 지원 패키지의 스킬이 에이전트에 연결된다.

실행하면 프로젝트 루트의 `AGENTS.md`에 아래와 같은 내용이 자동으로 추가된다.

```markdown
<!-- intent-skills:start -->
## Agent Skills

This project has intent-enabled packages. To use a skill, run:
  npx @tanstack/intent@latest load <package>#<skill>

Available skills:
- @tanstack/react-query#fetching
- @tanstack/react-query#mutations
- @tanstack/react-router#routing
- @tanstack/react-table#core

When working with these libraries, load the relevant skill first.
<!-- intent-skills:end -->
```

에이전트는 이 블록을 읽고 작업에 맞는 스킬을 `load` 명령으로 가져온다. `npm update` 이후 `intent install`을 다시 실행하면 목록이 자동으로 업데이트된다.

### `intent load` - 특정 스킬 로드

```bash
# @패키지명#스킬명 형식
npx @tanstack/intent@latest load @tanstack/query#fetching

# 스킬 파일 경로 확인
npx @tanstack/intent@latest load @tanstack/query#fetching --path
```

설치된 패키지 버전에 맞는 SKILL.md 내용을 직접 출력한다.

---

## 라이브러리 관리자(Maintainer) 입장

### `intent scaffold` - 스킬 생성

```bash
npx @tanstack/intent@latest scaffold
```

AI 에이전트가 인터랙티브 인터뷰를 통해 라이브러리의 도메인을 발견하고, 스킬 트리를 생성하고, SKILL.md 작성을 가이드한다. 각 단계마다 관리자의 리뷰가 포함된다.

### `intent validate` - 스킬 검증

```bash
# 현재 디렉토리 검증
npx @tanstack/intent@latest validate

# 모노레포에서 특정 패키지 검증
npx @tanstack/intent@latest validate packages/router/skills
```

SKILL.md 형식 규칙(name 컨벤션, description 길이 등)과 패키징 요구사항을 배포 전에 검사한다.

### `intent stale` - 버전 드리프트 감지

```bash
npx @tanstack/intent@latest stale

# JSON 출력 (CI에서 파싱용)
npx @tanstack/intent@latest stale --json
```

각 스킬이 참조하는 소스 문서가 변경된 경우를 감지한다. CI 파이프라인에 연결해두면 소스 문서가 바뀔 때 스킬 업데이트를 강제할 수 있다.

```plain text
소스 문서 변경
    → intent stale 이 스킬을 "review 필요" 상태로 표시
    → CI 실패
    → 관리자가 스킬 업데이트 후 재배포
    → 사용자는 npm update 만으로 최신 스킬 수신
```

### `intent setup` - CI 워크플로우 설정

```bash
npx @tanstack/intent@latest setup
```

GitHub Actions 워크플로우 템플릿을 `.github/workflows/`에 복사한다. 모노레포 감지, validate, stale 체크가 포함된 CI 설정이 자동으로 구성된다.

### `intent feedback` - 피드백 제출

```bash
npx @tanstack/intent@latest feedback
```

스킬이 잘못된 코드를 생성했을 때 구조화된 피드백 리포트를 제출한다. 어떤 스킬, 어떤 버전에서 무엇이 잘못됐는지가 라이브러리 팀에 전달되어, 다음 패키지 업데이트로 모든 사용자에게 수정이 배포된다.

---

## 전체 CLI 요약표

| 명령어             | 대상          | 역할                                         |
| :----------------- | :------------ | :------------------------------------------- |
| `list`             | 소비자        | node_modules에서 Intent 지원 패키지 스캔     |
| `install`          | 소비자        | 에이전트 설정 파일에 스킬 로딩 가이던스 추가 |
| `load <pkg#skill>` | 소비자        | 특정 스킬의 SKILL.md 내용 출력               |
| `scaffold`         | 관리자        | AI 가이드 스킬 생성                          |
| `validate`         | 관리자        | SKILL.md 형식 검증                           |
| `stale`            | 관리자        | 소스 문서 변경에 따른 스킬 drift 감지        |
| `setup`            | 관리자        | CI 워크플로우 자동 구성                      |
| `feedback`         | 소비자/관리자 | 잘못된 스킬 결과 리포트                      |

---

# 기존 접근 방식과의 비교

| 방식                  | 버전 관리            | 업데이트              | 소유권            | stale 감지    |
| :-------------------- | :------------------- | :-------------------- | :---------------- | :------------ |
| 훈련 데이터           | 학습 시점 고정       | 불가                  | AI 회사           | 불가          |
| 커뮤니티 rules 파일   | 없음                 | 수동                  | 커뮤니티          | 없음          |
| `CLAUDE.md` 수동 작성 | 없음                 | 수동                  | 개발자 개인       | 없음          |
| **TanStack Intent**   | **패키지 버전 연동** | **npm update와 동일** | **라이브러리 팀** | **CI 자동화** |

핵심 차이는 **소유권**이다. 기존 방식들은 라이브러리 팀이 소유하지 않는 채널을 통해 AI 지식이 전달된다. Intent는 라이브러리 팀이 소스 코드와 함께 스킬을 직접 소유하고 배포한다.

## 스킬 유무에 따른 AI 코드 생성 비교

TanStack Query v5를 처음 접한 에이전트에게 "서버에서 유저 데이터를 가져오는 컴포넌트를 만들어줘"라고 요청했을 때의 차이다.

```tsx
// ❌ 스킬 없이 생성된 코드 (v4 패턴 + 훈련 데이터 기반)
function UserProfile({ userId }) {
  const { data } = useQuery(
    ['user', userId],          // v5에서는 객체 형태로 바뀜
    () => fetchUser(userId),   // queryFn이 두 번째 인자였던 v4 방식
    {
      onSuccess: (data) => {   // v5에서 제거된 콜백
        console.log(data)
      },
    }
  )
  return <div>{data?.name}</div>
}
```

```tsx
// ✅ fetching 스킬 로드 후 생성된 코드 (v5 패턴)
const userQueryOptions = (userId: string) =>
  queryOptions({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 1000 * 60 * 5,
  })

function UserProfile({ userId }: { userId: string }) {
  const { data, isPending, isError } = useQuery(userQueryOptions(userId))

  if (isPending) return <Spinner />
  if (isError) return <ErrorMessage />
  return <div>{data.name}</div>
}
```

훈련 데이터에는 v4와 v5가 혼재되어 있다. 스킬이 없으면 에이전트는 어느 버전인지 구분하지 못하고 v4 스타일을 섞어 쓴다. 스킬은 이 모호함을 "현재 설치된 버전 기준"으로 단정지어준다.

---

# 현재 채택 현황 (2026년 4월 기준)

TanStack Intent가 기반하는 Agent Skills 오픈 스탠다드(`agentskills.io`)를 채택한 도구 목록이다.

**IDE / Editor**
VS Code, Cursor, TRAE

**AI 코딩 에이전트**
GitHub Copilot, Claude Code, OpenAI Codex, Goose, Amp, Roo Code, JetBrains Junie, Gemini CLI, OpenHands, OpenCode

현재 v0.0.36 알파 단계이며, MIT 라이선스로 공개되어 있다.

---

# 정리

TanStack Intent가 해결하는 문제는 단순하다.

**"라이브러리는 계속 업데이트되는데, AI 에이전트가 갖고 있는 지식은 학습 시점에 고정되어 있다"**

그 해법이 npm 패키지 안에 SKILL.md를 동봉하는 것이다. 패키지 업데이트 = 스킬 업데이트. 별도의 채널도, 별도의 관리도 없다.

```plain text
라이브러리 팀이 코드 릴리스 → SKILL.md도 함께 배포
개발자가 npm update → 스킬도 자동 업데이트
AI 에이전트가 node_modules 스캔 → 버전에 맞는 스킬 발견
→ 항상 현재 버전에 맞는 올바른 코드 생성
```

AI 에이전트가 점점 더 많은 코드를 쓰게 되는 시대에, 라이브러리 팀이 "우리 라이브러리를 올바르게 쓰는 방법"을 직접 에이전트에게 전달하는 채널을 갖는 것은 문서나 타입만큼이나 중요한 인프라가 될 것 같다.
