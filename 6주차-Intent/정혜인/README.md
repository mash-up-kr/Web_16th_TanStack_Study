# Intent란 무엇일까?

> **“AI 에이전트는 코드를 잘 짜지만, 라이브러리를 ‘제대로’ 쓰는 법은 모른다.”**
> 즉, TanStack Intent는 AI 코딩 에이전트가 해결해주지 못하는 **’라이브러리 지식(Library Knowledge)’의 버전 관리와 배포 문제**를 우아하게 해결하기 위해 탄생했습니다.

TanStack Intent는 단순히 문서를 생성하는 도구가 아닌 **AI 에이전트를 위한 지식을 npm 패키지와 함께 배포하는 인프라**라고 할 수 있습니다.

### **Agent Skill 배포 시스템**

우리가 흔히 아는 `CLAUDE.md`, `.cursorrules`가 **프로젝트 수준의 에이전트 설정**을 관리한다면, TanStack Intent는 **라이브러리 수준의 에이전트 지식(Agent Skills)**을 전담합니다.

- **Agent Skill의 특징:** 라이브러리 메인테이너가 작성하고 npm 패키지에 포함되어 배포됨
  - 프로젝트의 `node_modules`에서 자동으로 발견(discovery)됨
  - 라이브러리 버전과 함께 버저닝되어 내가 설치한 버전에 맞는 정확한 사용법을 제공함
  - 관리하지 않으면 라이브러리 업데이트 시 Stale 상태가 되기 쉬움

### **아키텍처 구조**

TanStack Intent는 **스킬 배포 파이프라인(Skill Distribution Pipeline)** 역할을 합니다.
`[AI 에이전트] ↔︎ [TanStack Intent (Discovery + Resolution)] ↔︎ [npm 패키지 내 skills/]` 구조를 만들어, 에이전트가 라이브러리 문서를 직접 찾는 대신 **설치된 패키지에서 바로 정확한 사용법을 로드**하게 합니다.

```
[AI 에이전트 (Claude Code, Cursor, Copilot)]
        ↕  load @tanstack/query#fetching
[TanStack Intent CLI]
   ├─ Scanner: node_modules 스캔 → intent 패키지 발견
   ├─ Resolver: @package#skill → 실제 SKILL.md 경로 해석
   └─ Loader: SKILL.md 읽기 + 상대 링크 리라이팅
        ↕
[npm 패키지 내 skills/ 디렉토리]
   └─ skills/
       ├─ fetching/SKILL.md
       ├─ mutations/SKILL.md
       └─ sync-state.json
```

# Intent는 어떻게 사용하는 걸까?

### 1. 라이브러리 사용자 관점

### 스킬 발견

프로젝트에 설치된 의존성 중 Intent를 지원하는 패키지를 자동으로 스캔합니다.

```bash
npx @tanstack/intent@latest list
```

### 스킬 로드

특정 스킬의 내용을 로드합니다. **Skill Use 포맷**은 `<패키지명>#<스킬명>`입니다.

```bash
npx @tanstack/intent@latest load @tanstack/query#fetching
# → SKILL.md 내용이 stdout으로 출력됨
```

### 에이전트 설정 파일에 설치

```bash
npx @tanstack/intent@latest install --map
```

이 명령은 `AGENTS.md`(또는 `CLAUDE.md`, `.cursorrules` 등)에 아래와 같은 매핑 블록을 생성합니다.

```markdown
<!-- intent-skills:start -->

# Skill mappings - load `use` with `npx @tanstack/intent@latest load <use>`.

skills:
-when: "데이터 페칭을 구현할 때"
use: "@tanstack/query#fetching"
-when: "뮤테이션을 작성할 때"
use: "@tanstack/query#mutations"

<!-- intent-skills:end -->
```

### 2. Maintainer(라이브러리 메인테이너) 관점

### 스킬 스캐폴딩 (AI 인터뷰 기반 생성)

```bash
npx @tanstack/intent@latest scaffold
```

4단계 메타스킬(domain-discovery → tree-generator → generate-skill → feedback-collection)을 통해 AI와 인터뷰하며 스킬을 생성합니다.

### 스킬 검증 & Staleness 체크

```bash
npx @tanstack/intent@latest validate    # SKILL.md 포맷 검증
npx @tanstack/intent@latest stale       # 버전 드리프트 / 소스 변경 감지
```

### 3. 핵심 데이터 구조: Skill Use 포맷

가장 중요한 것은 **패키지명**과 **스킬명**의 분리입니다.

```tsx
// skill-use.ts:44-63
export function parseSkillUse(value: string): SkillUse {
  const trimmed = value.trim();
  const separatorIndex = trimmed.indexOf("#");

  if (separatorIndex === -1) {
    throw new SkillUseParseError("missing-separator", value);
  }

  const packageName = trimmed.slice(0, separatorIndex).trim();
  const skillName = trimmed.slice(separatorIndex + 1).trim();

  return { packageName, skillName };
}
```

`@tanstack/query#fetching`이라는 한 줄의 문자열이 `{ packageName: "@tanstack/query", skillName: "fetching" }`으로 파싱되고, 이것이 전체 시스템의 **유니크한 식별자** 역할을 합니다.

### +) 추가 팁

- **Global 스킬:** `-global` 플래그를 추가하면 전역 설치된 패키지의 스킬도 포함됩니다. 로컬 패키지가 항상 우선함
- **Yarn PnP 지원:** `node_modules`가 없는 Yarn PnP 프로젝트에서도 PnP API를 통해 스킬을 발견함
- **모노레포 지원:** pnpm-workspace, npm workspaces, lerna 등의 워크스페이스 패턴을 자동 감지함

# Intent는 왜 쓰는 걸까?

### 1. AI 에이전트의 고질적 문제 해결

AI 에이전트에게 라이브러리 사용법을 직접 알려주려면 다음과 같은 문제를 **직접** 해결해야 합니다. TanStack Intent는 이러한 문제를 해결해줍니다.

| **직접 관리 시 단점**                                              | **TanStack Intent 솔루션**                                                        |
| ------------------------------------------------------------------ | --------------------------------------------------------------------------------- |
| **버전 불일치** (에이전트가 v4 문법을 v5 프로젝트에 적용)          | **Version-locked Skills**: npm 패키지에 포함되어 설치된 버전의 정확한 사용법 제공 |
| **지식 분산** (CLAUDE.md에 모든 라이브러리 사용법을 수동 작성)     | **Auto-discovery**: `node_modules` 스캔으로 자동 발견, `install` 한 줄로 설정     |
| **Staleness** (라이브러리가 업데이트되었는데 가이드가 옛날 그대로) | **Staleness Tracking**: 버전 드리프트와 소스 문서 변경을 CI에서 자동 감지         |
| **중복 관리** (같은 라이브러리 스킬을 프로젝트마다 복사)           | **npm 배포**: 패키지 설치만으로 스킬이 따라옴, 모든 프로젝트에서 재사용           |

### 2. 선언적 스킬 매핑

명령형으로 “이 파일을 읽어서 이 규칙을 적용해라”라고 지시할 필요가 없습니다. 단지 “이 작업을 할 때는 이 스킬을 로드해줘”라고 선언하면 됩니다.

```markdown
skills:
-when: "데이터 페칭을 구현할 때"
use: "@tanstack/query#fetching"
```

### 3. 메인테이너-소비자 피드백 루프

- **Feedback Collection:** 에이전트가 스킬을 사용한 후 “뭐가 잘 됐고 뭐가 안 됐는지”를 구조화된 형태로 수집하여 GitHub Issue를 생성합니다.
- **CI 자동화:** `intent stale --github-review`로 라이브러리 릴리스 시 오래된 스킬을 자동 감지하고 PR을 생성합니다.

**Q) 왜 CLAUDE.md에 직접 적는 것보다 Intent가 나을까?**

Intent는 **라이브러리 메인테이너**가 스킬을 작성합니다. 즉, 라이브러리를 가장 잘 아는 사람이 에이전트에게 가르치는 것입니다. 프로젝트 개발자가 개별적으로 정리하는 것보다 정확하고 npm 릴리스에 포함되므로 버전이 올라가면 스킬도 자동으로 업데이트됩니다.

`intent install` 한 줄이 실행되면 내부에서는 다음 과정이 일어납니다

```
install 호출
  └─ scanForIntents()       ← node_modules 전체 스캔
       ├─ 패키지 매니저 감지
       ├─ skills/ 디렉토리가 있는 패키지 발견
       └─ SKILL.md frontmatter 파싱 → 스킬 목록 수집
  └─ buildIntentSkillsBlock()  ← 매핑 블록 생성
  └─ writeIntentSkillsBlock()  ← AGENTS.md에 기록
  └─ verifyIntentSkillsBlockFile()  ← 검증
```

즉, TanStack Intent를 사용하면 이와 같은 장점이 있습니다. AI 에이전트가 가질 수 있는 수많은 라이브러리 지식 문제(버전 불일치, 오래된 패턴, 분산된 설정 등)를 **패키지 매니저 수준에서 해결하기 때문에 메인테이너는 스킬 작성에, 소비자는 코드 작성에만 집중하게 만들기 위해서**라고 할 수 있습니다.

> `npx intent load @tanstack/query#fetching` 한 줄이 실행되는 동안, 내부에서는 4개의 레이어가 차례로 작동합니다.
> `CLI Parser` → `Scanner` → `Resolver` → `Loader`

**흐름 다이어그램**

```
intent load @tanstack/query#fetching
  └─ main(argv)                         ← CLI 진입점
       ├─ cac('intent').parse()          ← 커맨드 파싱
       └─ runLoadCommand(use, options)
            ├─ parseSkillUse(use)        ← @tanstack/query + fetching 분리
            ├─ scanIntentsOrFail()       ← node_modules 전체 스캔
            │    └─ scanForIntents(root)
            │         ├─ detectPackageManager()
            │         ├─ createPackageRegistrar()
            │         ├─ createDependencyWalker()
            │         ├─ scanTarget() → tryRegister() → discoverSkills()
            │         ├─ walkWorkspacePackages()
            │         ├─ walkKnownPackages()
            │         ├─ walkProjectDeps()
            │         └─ topoSort(packages)
            ├─ resolveSkillUse(use, scanResult)
            │    ├─ packages.filter(name === packageName)
            │    ├─ prefer local over global
            │    └─ find skill in package.skills
            └─ rewriteLoadedSkillMarkdownDestinations()
                 └─ stdout.write(content)
```

---

# 동작을 파헤쳐보자

## `intent load` 한 줄이 SKILL.md를 출력하기까지 — 전체 내부 흐름

### 전체 흐름 요약도

```
intent load @tanstack/query#fetching
  │
  ▼
main(argv)                                      ← CLI 레이어 진입
  ├─ [1] cac('intent').parse(argv)              ← 커맨드 매칭
  │       └─ runMatchedCommand()
  │
  ▼
runLoadCommand(use, options)                     ← Load 커맨드 진입
  │
  ├─ [2] parseSkillUse(use)                     ← Skill Use 파싱
  │       └─ { packageName: "@tanstack/query",
  │            skillName: "fetching" }
  │
  ├─ [3] scanIntentsOrFail(options)              ← 전체 스캔 시작
  │       └─ scanForIntents(projectRoot)
  │            ├─ detectPackageManager()          ← 패키지 매니저 감지
  │            ├─ createPackageRegistrar()        ← 패키지 등록기 생성
  │            │     └─ tryRegister(dirPath)
  │            │          ├─ skills/ 존재 확인
  │            │          ├─ package.json 읽기
  │            │          ├─ intent 필드 검증 or 자동 유추
  │            │          ├─ discoverSkills(skillsDir)
  │            │          │     └─ 재귀적 SKILL.md 탐색 + frontmatter 파싱
  │            │          ├─ rewriteSkillLoadPaths()
  │            │          └─ 중복 패키지 → 깊이/버전으로 선택
  │            ├─ createDependencyWalker()        ← 의존성 워커 생성
  │            │     ├─ walkWorkspacePackages()
  │            │     ├─ walkKnownPackages()
  │            │     └─ walkProjectDeps()
  │            ├─ 버전 충돌 감지
  │            └─ topoSort(packages)              ← requires 기반 위상 정렬
  │
  ├─ [4] resolveSkillUse(use, scanResult)        ← 스킬 해석
  │       ├─ packages.filter(pkg.name === packageName)
  │       ├─ prefer local over global
  │       └─ pkg.skills.find(s.name === skillName)
  │
  └─ [5] rewriteLoadedSkillMarkdownDestinations() ← 링크 리라이팅
         └─ process.stdout.write(content)         ← 최종 출력
```

### Step 1 — CLI 진입: `main()` → `cac`

**코드** (cli.ts:186-222)

```tsx
export async function main(argv: Array<string> = process.argv.slice(2)) {
  try {
    const cli = createCli();

    if (argv.length === 0) {
      cli.outputHelp();
      return 0;
    }

    // cac expects process.argv format: first two entries are ignored
    cli.parse(["intent", "intent", ...argv], { run: false });

    if (!cli.matchedCommand) {
      cli.outputHelp();
      return 1;
    }

    await cli.runMatchedCommand();
    return 0;
  } catch (err) {
    if (isCliFailure(err)) {
      console.error(err.message);
      return err.exitCode;
    }
    // ...
  }
}
```

`cac` 라이브러리를 사용해 커맨드를 파싱합니다. `argv`가 비어있으면 help를 출력하고 매칭되는 커맨드가 없으면 exit code 1을 반환합니다.

> **왜 `cac`인가?**`commander`나 `yargs`보다 가볍고(~3KB), TypeScript 친화적이며 TanStack 에코시스템의 미니멀한 철학과 맞습니다. `cli.parse()`와 `cli.runMatchedCommand()`로 파싱과 실행을 분리할 수 있어 에러 핸들링이 깔끔해집니다.

### Step 2 — Skill Use 파싱: `parseSkillUse()`

**코드** (skill-use.ts:44-63)

```tsx
export function parseSkillUse(value: string): SkillUse {
  const trimmed = value.trim();
  const separatorIndex = trimmed.indexOf("#");

  if (separatorIndex === -1) {
    throw new SkillUseParseError("missing-separator", value);
  }

  const packageName = trimmed.slice(0, separatorIndex).trim();
  const skillName = trimmed.slice(separatorIndex + 1).trim();

  if (packageName === "") {
    throw new SkillUseParseError("empty-package", value);
  }

  if (skillName === "") {
    throw new SkillUseParseError("empty-skill", value);
  }

  return { packageName, skillName };
}
```

`@tanstack/query#fetching` → `{ packageName: "@tanstack/query", skillName: "fetching" }`

`#` 구분자를 기준으로 패키지명과 스킬명을 분리합니다. 에러 코드가 3가지(`missing-separator`, `empty-package`, `empty-skill`)로 명확하게 분류되어, 사용자에게 정확한 실패 원인을 알려줍니다.

> **왜 `#` 구분자인가?**
> npm 패키지명에는 `#`이 올 수 없으므로 모호함 없이 패키지와 스킬을 분리할 수 있습니다. `@scope/package#skill/sub-skill` 형태의 중첩 스킬명도 안전하게 처리됩니다(첫 번째 `#`만 구분자로 사용)

### Step 3 — 스캔: `scanForIntents()` — 시스템의 심장

`node_modules`를 탐색하여 intent-enabled 패키지를 모두 발견하는 핵심 로직입니다.

**코드** (scanner.ts:412-622)

```tsx
export function scanForIntents(
  root?: string,
  options: ScanOptions = {},
): ScanResult {
  const projectRoot = root ?? process.cwd()
  const scanScope = getScanScope(options)
  const packageManager = detectPackageManager(projectRoot)

  // ... 초기화 ...

  const { scanTarget, tryRegister } = createPackageRegistrar({ ... })
  const { walkKnownPackages, walkProjectDeps, walkWorkspacePackages } =
    createDependencyWalker({ ... })

  switch (scanScope) {
    case 'local':
      scanLocalPackages()
      break
    case 'local-and-global':
      scanLocalPackages()
      scanGlobalPackages()
      walkKnownPackages()
      walkProjectDeps()
      break
    case 'global':
      scanGlobalPackages()
      break
  }

  // 버전 충돌 감지
  for (const pkg of packages) {
    const variants = packageVariants.get(pkg.name)
    // ...
  }

  // 의존성 순서로 정렬
  const sorted = topoSort(packages)
  return { packageManager, packages: sorted, warnings, conflicts, nodeModules }
}
```

스캔은 크게 5단계로 이루어집니다:

> **3-1. 패키지 매니저 감지** (scanner.ts:83-97)

```tsx
function detectPackageManager(root: string): PackageManager {
  const dirsToCheck = [root];
  const wsRoot = findWorkspaceRoot(root);
  if (wsRoot && wsRoot !== root) dirsToCheck.push(wsRoot);

  for (const dir of dirsToCheck) {
    if (isYarnPnpProject(dir)) return "yarn";
    if (existsSync(join(dir, "pnpm-lock.yaml"))) return "pnpm";
    if (existsSync(join(dir, "bun.lockb")) || existsSync(join(dir, "bun.lock")))
      return "bun";
    if (existsSync(join(dir, "yarn.lock"))) return "yarn";
    if (existsSync(join(dir, "package-lock.json"))) return "npm";
  }
  return "unknown";
}
```

프로젝트 루트와 워크스페이스 루트 두 곳에서 lock 파일을 확인합니다. **Yarn PnP를 가장 먼저 체크**하는 이유는 PnP 프로젝트에서는 `node_modules`가 없을 수 있어서 다른 로직을 타야 하기 때문입니다.

> **3-2. 패키지 등록기 (Package Registrar)** — 스킬이 있는 패키지 발견

**코드** (discovery/register.ts:48-127)

```tsx
function tryRegister(
  dirPath: string,
  fallbackName: string,
  source: IntentPackage['source'] = 'local',
): boolean {
  const skillsDir = join(dirPath, 'skills')
  if (!existsSync(skillsDir)) return false             // 1. skills/ 있나?

  const pkgJson = opts.readPkgJson(dirPath)
  if (!pkgJson) return false                            // 2. package.json 읽기

  const intent =
    opts.validateIntentField(name, pkgJson.intent) ??   // 3. intent 필드 검증
    opts.deriveIntentConfig(pkgJson)                     //    없으면 자동 유추
  if (!intent) return false

  const skills = opts.discoverSkills(skillsDir, name)   // 4. SKILL.md 탐색

  if (isLocalToProject(dirPath, opts.projectRoot)) {
    rewriteSkillLoadPaths({ ... })                      // 5. 경로 리라이팅
  }

  // 6. 중복 패키지 처리
  const existingIndex = opts.packageIndexes.get(name)
  if (existingIndex === undefined) {
    // 새 패키지 → 등록
    opts.packageIndexes.set(name, opts.packages.push(candidate) - 1)
    return true
  }

  // 이미 있으면 → 깊이가 더 얕거나, 같으면 버전이 더 높은 쪽으로 교체
  const shouldReplace =
    candidateDepth < existingDepth ||
    (candidateDepth === existingDepth &&
      opts.comparePackageVersions(candidate.version, existing.version) > 0)

  if (shouldReplace) {
    opts.packages[existingIndex] = candidate
  }
  return true
}
```

`tryRegister`가 하는 일을 정리하면:

1. `skills/` 디렉토리 존재 확인
2. `package.json` 읽기
3. `intent` 필드 검증 또는 `repository`/`homepage`에서 자동 유추
4. `discoverSkills()`로 재귀적 SKILL.md 탐색
5. 로컬 패키지이면 경로를 `node_modules/<pkg>/...` 형태로 리라이팅
6. 같은 이름의 패키지가 이미 있으면 **깊이(depth) → 버전(version)** 우선순위로 선택

**Q) `intent` 필드가 없는 패키지도 스킬을 제공할 수 있을까?**

네, `deriveIntentConfig()`가 있기 때문입니다. `package.json`에 `repository`와 `homepage` 필드만 있으면 intent config를 자동으로 유추합니다:

```tsx
// scanner.ts:180-220
function deriveIntentConfig(
  pkgJson: Record<string, unknown>,
): IntentConfig | null {
  let repo: string | null = null;
  if (typeof pkgJson.repository === "string") {
    repo = pkgJson.repository;
  } else if (pkgJson.repository?.url) {
    repo = repo
      .replace(/^git\+/, "")
      .replace(/\.git$/, "")
      .replace(/^https?:\/\/github\.com\//, "");
  }

  const docs =
    typeof pkgJson.homepage === "string" ? pkgJson.homepage : undefined;
  if (!repo) return null;

  return { version: 1, repo, docs: docs ?? "" };
}
```

즉, `skills/` 디렉토리만 있으면 어떤 패키지든 Intent를 지원할 수 있습니다. 명시적 `intent` 필드는 선택사항입니다.

**3-3. 스킬 탐색: `discoverSkills()`** — 패키지 내 모든 SKILL.md 수집

**코드** (scanner.ts:226-267)

```tsx
function discoverSkills(
  skillsDir: string,
  _baseName: string,
): Array<SkillEntry> {
  const skills: Array<SkillEntry> = [];

  function walk(dir: string): void {
    let entries: Array<Dirent<string>>;
    try {
      entries = readdirSync(dir, { withFileTypes: true, encoding: "utf8" });
    } catch {
      return;
    }

    for (const entry of entries) {
      if (!entry.isDirectory()) continue;
      const childDir = join(dir, entry.name);
      const skillFile = join(childDir, "SKILL.md");

      if (existsSync(skillFile)) {
        const fm = parseFrontmatter(skillFile);
        const relName = toPosixPath(relative(skillsDir, childDir));
        skills.push({
          name: typeof fm?.name === "string" ? fm.name : relName,
          path: skillFile,
          description:
            typeof fm?.description === "string"
              ? fm.description.replace(/\s+/g, " ").trim()
              : "",
          type: typeof fm?.type === "string" ? fm.type : undefined,
          framework:
            typeof fm?.framework === "string" ? fm.framework : undefined,
        });
      }
      // 항상 하위 디렉토리도 재귀 탐색
      walk(childDir);
    }
  }

  walk(skillsDir);
  return skills;
}
```

핵심: `SKILL.md`를 찾아도 **항상 하위 디렉토리를 계속 탐색**합니다. `skills/core/SKILL.md`와 `skills/core/advanced/SKILL.md`가 동시에 존재할 수 있기 때문입니다.

`parseFrontmatter()`로 YAML 프런트매터를 파싱하여 `name`, `description`, `type`, `framework` 메타데이터를 추출합니다.

**3-4. 의존성 워커 (Dependency Walker)** — 숨겨진 패키지도 찾아내기

**코드** (discovery/walk.ts:20-128)

```tsx
export function createDependencyWalker(opts: CreateDependencyWalkerOptions) {
  const walkVisited = new Set<string>(); // 순환 의존성 방지

  function walkDeps(pkgDir: string, pkgName: string): void {
    if (walkVisited.has(pkgDir)) return;
    walkVisited.add(pkgDir);

    const pkgJson = opts.readPkgJson(pkgDir);
    if (!pkgJson) return;

    walkDepsOf(pkgJson, pkgDir);
  }

  function walkDepsOf(pkgJson, fromDir, includeDevDeps = false): void {
    for (const depName of getDeps(pkgJson, includeDevDeps)) {
      const depDir = resolveDepDirCached(depName, fromDir);
      if (!depDir || walkVisited.has(depDir)) continue;

      opts.tryRegister(depDir, depName); // 의존성도 등록 시도
      walkDeps(depDir, depName); // 재귀적으로 그 의존성의 의존성도 탐색
    }
  }

  // ... walkWorkspacePackages, walkKnownPackages, walkProjectDeps
}
```

**왜 의존성 워킹이 필요한가?**

`node_modules` 최상위 스캔만으로는 pnpm의 `.pnpm/` 가상 스토어나 호이스팅되지 않은 패키지를 놓칠 수 있습니다. 의존성 트리를 직접 걸어가며 `tryRegister()`를 호출하면 어떤 패키지 매니저 구조에서도 intent 패키지를 빠짐없이 발견합니다.

> **`walkVisited` Set이 왜 중요한가?**
>
> npm 생태계에서는 순환 의존성이 드물지 않습니다(A → B → C → A). `walkVisited`가 없으면 무한 루프에 빠집니다. 또한 같은 패키지가 여러 경로에서 참조될 수 있는데 한 번 방문한 디렉토리는 다시 탐색하지 않아 성능을 보장합니다.
>
> - 퀴즈) pnpm에서 `@tanstack/query`가 `.pnpm/@tanstack+query@5.0.0/node_modules/@tanstack/query`에 실제로 있고, 프로젝트 루트의 `node_modules/@tanstack/query`는 심링크라면 어떻게 될까?
>   `tryRegister`의 `isLocalToProject()` 체크와 `rewriteSkillLoadPaths()`가 이 경우를 처리합니다. 심링크의 존재 여부를 확인하고, 심링크가 있으면 `node_modules/@tanstack/query/...` 형태의 안정적인 경로를 사용합니다.

**3-5. 위상 정렬: `topoSort()`** — 스킬 로드 순서 보장

**코드** (scanner.ts:273-293)

```tsx
function topoSort(packages: Array<IntentPackage>): Array<IntentPackage> {
  const byName = new Map(packages.map((p) => [p.name, p]));
  const visited = new Set<string>();
  const sorted: Array<IntentPackage> = [];

  function visit(name: string): void {
    if (visited.has(name)) return;
    visited.add(name);
    const pkg = byName.get(name);
    if (!pkg) return;
    for (const dep of pkg.intent.requires ?? []) {
      visit(dep); // 의존하는 패키지를 먼저 방문
    }
    sorted.push(pkg);
  }

  for (const pkg of packages) {
    visit(pkg.name);
  }
  return sorted;
}
```

**왜 위상 정렬이 필요한가?**

예를 들어 `@tanstack/react-query` 스킬이 “먼저 `@tanstack/query-core` 스킬을 읽어야 이해할 수 있다”고 선언했다면, `intent.requires: ["@tanstack/query-core"]`로 의존 관계를 표현합니다. 위상 정렬은 이 순서를 보장하여 에이전트가 기초 지식부터 순서대로 로드할 수 있게 합니다.

### Step 4 — 해석: `resolveSkillUse()` — 스캔 결과에서 정확한 스킬 찾기

**코드** (resolver.ts:66-117)

```tsx
export function resolveSkillUse(
  use: string,
  scanResult: ScanResult,
): ResolveSkillResult {
  const { packageName, skillName } = parseSkillUse(use)

  // 1. 패키지 필터링 — 이름이 같은 패키지 모두 찾기
  const packages = scanResult.packages.filter(
    (pkg) => pkg.name === packageName,
  )

  // 2. 로컬 우선 — local이 있으면 local, 없으면 첫 번째
  const pkg =
    packages.find((candidate) => candidate.source === 'local') ?? packages[0]

  if (!pkg) {
    throw new ResolveSkillUseError({
      availablePackages: scanResult.packages.map((c) => c.name),
      code: 'package-not-found',
      packageName, skillName, use,
    })
  }

  // 3. 스킬 검색
  const skill = pkg.skills.find(
    (candidate) => candidate.name === skillName,
  )

  if (!skill) {
    throw new ResolveSkillUseError({
      availableSkills: pkg.skills.map((c) => c.name),
      code: 'skill-not-found',
      packageName, skillName, use,
    })
  }

  // 4. 버전 충돌 정보 첨부
  const conflict = scanResult.conflicts.find(
    (candidate) => candidate.packageName === packageName,
  ) ?? null

  return {
    packageName, skillName, path: skill.path,
    source: pkg.source, version: pkg.version,
    packageRoot: pkg.packageRoot, warnings: ..., conflict,
  }
}
```

3단계 필터링: **패키지명 매칭 → 소스 우선순위(local > global) → 스킬명 매칭**

에러 발생 시 `availablePackages`나 `availableSkills`를 함께 반환하여 “혹시 이것을 찾으셨나요?” 형태의 도움을 줍니다. `ResolveSkillUseError`가 에러 코드를 `'package-not-found' | 'skill-not-found'`로 분류하는 것처럼, 실패 원인을 프로그래밍적으로 구별할 수 있게 설계되어 있습니다.

### Step 5 — 로딩 + 링크 리라이팅: 최종 출력

**코드** (commands/load.ts:294-337)

```tsx
function rewriteLoadedSkillMarkdownDestinations({
  content, packageRoot, skillFilePath,
}: { ... }): string {
  const context: MarkdownDestinationRewriteContext = {
    cwd: process.cwd(),
    resolvedPackageRoot: resolveFromCwd(packageRoot),
    skillDir: dirname(skillFilePath),
  }

  let inFence: '`' | '~' | null = null
  const parts = content.split(/(\r?\n)/)
  let output = ''

  for (let index = 0; index < parts.length; index += 2) {
    const line = parts[index] ?? ''
    const newline = parts[index + 1] ?? ''
    const marker = getCodeFenceMarker(line)

    if (inFence) {
      output += line + newline
      if (marker === inFence) inFence = null  // 코드 펜스 닫힘
      continue
    }

    if (marker) {
      inFence = marker  // 코드 펜스 시작 → 내부 링크 변환 건너뜀
      output += line + newline
      continue
    }

    output +=
      rewriteMarkdownLineDestinations({ context, line }) + newline
  }

  return output
}
```

SKILL.md의 상대 링크(`[API 참고](../docs/api.md)`)를 현재 작업 디렉토리 기준으로 리라이팅합니다.

**핵심 안전장치들:**

- **코드 펜스 감지:** 코드 블록(```또는`~`) 안의 링크는 변환하지 않음
- **인라인 코드 감지:** `[not-a-link](path)` 안의 텍스트는 건너뜀
- **패키지 경계 체크:** 링크가 패키지 루트 밖으로 나가면 변환하지 않음
- **외부 링크 보존:** `https://`, `#anchor`, `?query` 등은 그대로 유지

```tsx
// commands/load.ts:182-214
function rewriteMarkdownDestination({ context, destination }): string {
  if (isExternalOrAbsoluteDestination(destination)) return destination;

  const resolvedPath = resolve(context.skillDir, pathPart);
  const relToPackageRoot = relative(context.resolvedPackageRoot, resolvedPath);

  // 패키지 루트 밖으로 나가면 변환하지 않음
  if (relToPackageRoot.startsWith("..") || isAbsolute(relToPackageRoot)) {
    return destination;
  }

  const relativeToCwd = relative(context.cwd, resolvedPath);
  return `${toPosixPath(rewrittenPath)}${suffix}`;
}
```

**Q) 왜 코드 펜스와 인라인 코드를 감지해야 할까?**

마크다운에서 코드 블록 안의 `[text](path)`는 링크가 아니라 코드입니다. 만약 이를 무작정 변환하면 코드 예제가 깨집니다:

````markdown
# 올바른 동작

```js
const link = "[docs](./api.md)"; // ← 이 문자열은 변환하면 안 됨
```
````

[실제 링크](./api.md) // ← 이것만 변환해야 함

````

`getCodeFenceMarker()`가 코드 펜스의 시작/끝을 추적하고, `rewriteMarkdownLineDestinations()`가 인라인 코드 스팬을 감지하여 변환을 건너뜁니다. 마크다운 파서를 사용하지 않고 직접 상태를 추적하는 이유는 외부 의존성을 최소화하기 위함입니다.

### Step 6 — Staleness 감지: 스킬은 어떻게 “오래된” 것을 알까?

`intent stale`이 호출되면 (staleness.ts:430-526):

```tsx
export async function checkStaleness(
  packageDir: string,
  packageName?: string,
): Promise<StalenessReport> {
  // 1. skills/ 내 모든 SKILL.md의 frontmatter에서 library_version 추출
  const skillMetas = skillFiles.map((filePath) => {
    const fm = parseFrontmatter(filePath)
    return {
      name: fm?.name ?? relName,
      libraryVersion: fm?.library_version,
      sources: fm?.sources,
    }
  })

  // 2. 현재 버전 확인: 로컬 package.json → 없으면 npm registry 조회
  const currentVersion = await fetchCurrentVersion(packageDir, library)

  // 3. 버전 드리프트 분류
  const versionDrift = classifyVersionDrift(skillVersion, currentVersion)
  // major: 5.x → 6.x, minor: 5.0 → 5.1, patch: 5.0.0 → 5.0.1

  // 4. sync-state.json에서 소스 SHA 비교
  const syncState = readSyncState(packageDir)
  // skills/sync-state.json
  // 새 소스가 추가되었으면 → reasons.push(`new source (${source})`)

  // 5. 아티팩트 시그널 빌드
  // (skill-missing, source-drift, version-drift)
  const signals = buildArtifactSignals({
    artifacts, skillMetas, ...
  })

  return {
    library, currentVersion, skillVersion,
    versionDrift, skills, signals,
  }
}
````

**3가지 Staleness 시그널:**

1. **Version Drift** — SKILL.md의 `library_version`과 현재 `package.json`의 `version` 비교
2. **Source Drift** — `sync-state.json`에 저장된 소스 SHA와 현재 소스 비교
3. **Artifact Drift** — `_artifacts/`의 스킬 정의와 실제 `skills/`의 SKILL.md 불일치

이것은 TanStack Query의 `staleTime`/`gcTime` 개념과 유사한 패턴입니다. Query가 “데이터가 오래되었는가?”를 시간 기반으로 판단한다면 Intent는 “스킬이 오래되었는가?”를 **버전과 소스 해시 기반**으로 판단합니다.

## 핵심 설계 원칙 정리

| **Query의 설계 패턴**                    | **Intent의 대응 패턴**                            | **공통 원칙**                     |
| ---------------------------------------- | ------------------------------------------------- | --------------------------------- |
| `Subscribable` → 구독 패턴 통일          | `createPackageRegistrar` → 등록 패턴 통일         | 변하는 것과 변하지 않는 것의 분리 |
| `QueryObserver` 주입으로 훅 다형성       | `tryRegister`를 워커에 주입으로 발견 전략 다형성  | 의존성 주입(DI)                   |
| `queryHash`로 중복 쿼리 방지             | `packageIndexes` Map으로 중복 패키지 방지         | 유니크 키 기반 캐싱               |
| `staleTime`/`gcTime`으로 데이터 생명주기 | `sync-state.json` + 버전 드리프트로 스킬 생명주기 | 생명주기 관리                     |
| `notifyManager.batch()`로 리렌더 최적화  | `topoSort()`로 로드 순서 최적화                   | 배치/순서 최적화                  |
| `ResolveSkillUseError`의 에러 코드 분류  | `SkillUseParseError`의 에러 코드 분류             | 프로그래밍적 에러 핸들링          |
