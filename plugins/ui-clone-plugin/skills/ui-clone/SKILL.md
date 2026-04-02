---
name: ui-clone
description: screenshots/ 폴더의 레거시 UI 캡처 이미지를 분석해서 spec.md, design.md, React 컴포넌트를 자동 생성한다. 사용자가 "UI 클론", "화면 복제", "스크린샷으로 React 만들어줘", "/ui-clone" 등을 요청할 때 사용한다.
disable-model-invocation: true
argument-hint: [screenshots-folder]
---

# UI Clone Pipeline

`screenshots/` 폴더의 캡처 이미지를 분석해서 레거시 시스템 UI를 React로 클론하는 전체 파이프라인을 실행한다.

## 입력 / 출력

- **Input**: `$ARGUMENTS` 로 지정한 폴더, 없으면 `screenshots/` 폴더의 이미지 파일 전체
- **Output**:
  - `analysis-result.json` — 이미지 분석 결과
  - `spec.md` — 기능 명세
  - `design.md` — 디자인 명세 (Stitch MCP 생성)
  - `src/` — React 컴포넌트 전체

---

## Phase 1 — 이미지 분석

입력 폴더의 모든 이미지를 읽는다. 각 이미지에서 다음을 파악한다:

- 화면 이름과 역할 (로그인, 목록, 상세, 폼, 대시보드 등)
- 화면에 존재하는 UI 요소 전체 (헤더, 사이드바, 네비게이션, 테이블, 버튼, 입력 필드, 모달, 탭 등)
- 표시되거나 입력받는 데이터 필드와 레이블
- 사용자가 취할 수 있는 액션
- 화면 간 이동 흐름
- 색상 팔레트, 폰트 스타일, 간격 패턴, 레이아웃 구조

분석이 끝나면 결과를 메모리에 유지하면서 프로젝트 루트에 `analysis-result.json`으로 저장한다.

### analysis-result.json 구조

```json
{
  "screens": [
    {
      "name": "화면명 (예: LoginPage, CC03001)",
      "role": "이 화면의 역할 한 줄 요약",
      "uiElements": ["헤더", "입력 필드", "버튼", "테이블" 등],
      "dataFields": ["필드명1", "필드명2"],
      "actions": ["로그인 버튼 클릭", "비밀번호 찾기 링크"],
      "navigation": { "액션명": "이동 대상 화면명" },
      "layout": "레이아웃 구조 설명 (예: 상단 헤더 + 좌측 사이드바 + 메인 콘텐츠)"
    }
  ],
  "colorPalette": ["#hex1", "#hex2"],
  "typography": {
    "fontFamily": "폰트명",
    "sizes": { "heading": "px 값", "body": "px 값", "caption": "px 값" }
  },
  "commonComponents": ["반복되는 UI 패턴 목록"]
}
```

---

## Phase 2 — spec.md 생성

`analysis-result.json`을 바탕으로 프로젝트 루트에 `spec.md`를 생성한다. 아래 구조를 따른다:

```
# Functional Specification

## 시스템 개요
[전체 시스템의 목적과 주요 기능 요약]

## 화면 목록
| 화면명 | 역할 |
|--------|------|
| ...    | ...  |

## 화면별 명세

### [화면명]
- **역할**: [이 화면이 하는 일]
- **진입 경로**: [어디서 이 화면으로 오는가]
- **데이터 필드**: [표시되거나 입력받는 데이터]
- **사용자 액션**: [버튼, 링크, 폼 제출 등 가능한 행동]
- **이동 경로**: [각 액션 후 어디로 이동하는가]
- **특이사항**: [레거시 시스템 특유의 패턴, 주의할 점]

## 공통 컴포넌트
[여러 화면에서 반복되는 UI 패턴 목록]

## 데이터 모델 추정
[화면에서 유추할 수 있는 주요 엔티티와 필드]
```

---

## Phase 3 — design.md 생성 (Stitch MCP 필수)

**Claude가 직접 작성하지 않는다.** 반드시 Stitch MCP를 사용해서 design.md를 생성한다.

> **이유**: 디자인 시스템은 단순 문서가 아니라 Stitch MCP가 관리하는 단일 진실 공급원(Single Source of Truth)이다. Claude가 임의로 작성하면 이후 컴포넌트 스타일과 불일치가 발생한다.

### 실행 순서

1. `analysis-result.json`의 색상 팔레트, 타이포그래피, 레이아웃 정보를 바탕으로 `mcp__stitch__create_design_system`을 호출한다.
2. Stitch MCP가 반환한 디자인 시스템 결과를 프로젝트 루트의 `design.md`에 저장한다.

### Stitch MCP 실패 시 폴백

Stitch MCP 호출이 실패하면 Claude가 직접 `design.md`를 작성한다. 이 경우 완료 보고의 `⚠️ 폴백 발생` 항목에 실패 사실을 명시한다.

### design.md 저장 구조

Stitch MCP 결과를 기반으로 아래 구조로 저장한다. CSS 변수 네이밍은 반드시 아래 규칙을 따른다:

```
# Design Specification
> Generated via Stitch MCP — 직접 수정하지 말 것. 변경 시 Stitch MCP를 통해 업데이트할 것.

## 색상 시스템
| 토큰명 | 용도 | Hex |
|--------|------|-----|
| --color-primary    | 주요 액션 | #... |
| --color-background | 페이지 배경 | #... |
| --color-border     | 테두리 | #... |
| --color-text       | 본문 텍스트 | #... |

## 타이포그래피
| 토큰명 | 값 |
|--------|-----|
| --font-family       | 폰트명 |
| --font-body-size    | px 값 |
| --font-heading-size | px 값 |

## 간격 시스템
| 토큰명 | 값 | 설명 |
|--------|-----|------|
| --spacing-unit | 4px  | 기본 단위 |
| --spacing-sm   | 8px  | unit × 2 |
| --spacing-md   | 16px | unit × 4 |
| --spacing-lg   | 24px | unit × 6 |

## 컴포넌트 스타일 가이드

### [컴포넌트명]
- **변형**: [Primary, Secondary, Danger 등]
- **상태**: [Default, Hover, Disabled, Loading 등]
- **CSS 변수**: [구현에 사용할 토큰]

## 화면별 레이아웃

### [화면명]
- **레이아웃 구조**: [헤더/사이드바/메인 등 전체 구성]
- **컴포넌트 트리**: [중첩 구조]
```

---

## Phase 4 — 컴포넌트 생성

각 화면을 아래 3단계로 처리한다. 화면은 spec.md의 화면 목록 순서대로 하나씩 처리한다.

### 4-1. Stitch MCP — HTML 생성

spec.md의 화면 설명과 Phase 3에서 생성된 design.md를 바탕으로, 화면별로 Stitch MCP를 순서대로 호출한다.

**Step 1 — 화면 HTML 생성**

`mcp__stitch__generate_screen_from_text`를 호출한다. 프롬프트에는 다음을 포함한다:
- spec.md에서 해당 화면의 역할, 데이터 필드, 사용자 액션
- 레이아웃 구조 (헤더/사이드바/메인 등)

**Step 2 — 디자인 시스템 적용**

`mcp__stitch__apply_design_system`을 호출해 Phase 3에서 생성한 디자인 시스템을 Step 1 결과에 적용한다. 색상, 타이포그래피, 간격 토큰이 HTML에 반영된다.

**Step 3 — HTML 코드 추출**

`mcp__stitch__get_screen`을 호출해 최종 HTML 코드를 추출한다. 이 HTML이 다음 단계의 변환 입력이 된다.

---

### 4-2. Claude Code — React + CSS Modules 변환

Stitch MCP가 반환한 HTML을 받아 아래 규칙으로 변환한다. 새 코드를 작성하는 것이 아니라 HTML 구조를 그대로 유지하면서 React와 CSS Modules 형식으로 옮기는 것이다.

**변환 규칙**

- HTML 태그 → JSX로 변환 (`class` → `className`, `for` → `htmlFor` 등)
- 인라인 `style` 속성 → `.module.css`로 추출, `className={styles.xxx}`로 교체
- 색상·폰트 리터럴 → design.md의 CSS 변수 토큰으로 교체 (네이밍 규칙 준수)
- 반복되는 데이터 목록 → props로 추상화, mock data는 `src/mocks/[PageName].mock.ts`로 분리
- 모든 props에 TypeScript interface 정의, `any` 사용 금지

**CSS 변수 네이밍 규칙** (design.md 기준 토큰만 사용)

| 분류 | 토큰명 예시 |
|------|-------------|
| 색상 | `--color-primary`, `--color-background`, `--color-border`, `--color-text` |
| 폰트 | `--font-family`, `--font-body-size`, `--font-heading-size` |
| 간격 | `--spacing-unit`, `--spacing-sm`, `--spacing-md`, `--spacing-lg` |

**출력 구조**

```
src/
├── components/          # 공통 컴포넌트 (Button, Input, Table, Modal 등)
│   └── [ComponentName]/
│       ├── index.tsx
│       └── index.module.css
├── pages/               # 화면 단위 컴포넌트
│   └── [PageName]/
│       ├── index.tsx
│       └── index.module.css
├── mocks/               # 화면별 mock data
│   └── [PageName].mock.ts
├── types/
│   └── index.ts         # Entity 모델 + Page Props 타입 정의
└── App.tsx              # 라우팅 연결
```

**타입 정의 규칙** (`src/types/index.ts`)

- **Entity 모델**: `analysis-result.json`에서 추출한 데이터 구조 (예: `User`, `Order`)
- **Page Props**: 각 페이지 컴포넌트의 props interface (예: `LoginPageProps`, `CC03001PageProps`)

---

### 4-3. Claude Code — 라우트 연결

모든 화면 변환이 끝난 뒤 `App.tsx`를 작성한다.

**경로 규칙**

- kebab-case 사용
- 화면 코드 기반 경로: `/cc03001`, `/pc06001` 등
- 탭 기반 중첩 라우트: `/exchange/cc03001`, `/storage/pc06001` 등
- spec.md의 화면 이동 흐름을 반영해 부모-자식 라우트 구조 설계

**예시**

```tsx
// App.tsx
<Routes>
  <Route path="/exchange" element={<ExchangeLayout />}>
    <Route path="cc03001" element={<CC03001Page />} />
    <Route path="cc03002" element={<CC03002Page />} />
  </Route>
  <Route path="/storage" element={<StorageLayout />}>
    <Route path="pc06001" element={<PC06001Page />} />
  </Route>
</Routes>
```

---

### CLAUDE.md 연동 규칙

> 이 규칙은 스킬 완료 후 프로젝트에서 지속적으로 적용된다.

- **스타일 변경 시 Stitch MCP 업데이트 필수**: 색상, 폰트, 간격 등 디자인 토큰 변경은 반드시 Stitch MCP → design.md 업데이트 순서로 진행한 뒤 코드에 반영한다.
- **색상/폰트 하드코딩 금지**: `.module.css`에 리터럴 값(`#fff`, `16px`, `'Pretendard'` 등)을 직접 쓰지 않고 design.md에 정의된 CSS 변수 토큰만 사용한다.

---

## 완료 보고

모든 파일 생성 후 아래 형식으로 요약한다:

```
## UI Clone 완료

### 생성된 파일
- analysis-result.json — [N]개 화면 분석 결과
- spec.md — [N]개 화면 기능 명세
- design.md — Stitch MCP로 생성된 디자인 시스템
- src/components/ — [공통 컴포넌트 목록 + 각 .module.css]
- src/pages/ — [페이지 컴포넌트 목록 + 각 .module.css]
- src/mocks/ — [PageName.mock.ts 목록]
- src/types/index.ts — Entity 모델 + Page Props
- src/App.tsx — 라우트 연결

### 발견된 화면
| 화면명 | 경로 | 역할 |
|--------|------|------|
| ...    | /... | ...  |

### ⚠️ 폴백 발생 (해당 시)
- Phase 3 Stitch MCP 실패 → Claude가 직접 design.md 작성

### 다음 단계 제안
- [ ] API 연동 (mock data → 실제 fetch)
- [ ] 스타일 변경 시 Stitch MCP → design.md 업데이트 후 .module.css 수정
- [ ] 누락된 화면 추가 시 screenshots/ 보완 후 재실행
```
