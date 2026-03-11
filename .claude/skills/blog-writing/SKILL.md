---
name: blog-writing
description: 주차별 블로그 작성 시 톤, 구조, HTML 디자인 템플릿을 일관되게 유지한다.
---

# Blog Writing Skill

## 작성 프로세스

1. **참고 자료 수집** (병렬로 수행)
   - `docs/logs/{n}-weeks-talk-log.md` — 해당 주차 대화 로그
   - `docs/mentoring/{n}-weeks-mentoring.md` — 멘토링 피드백
   - 해당 주차에 작성한 소스 코드 (커밋 히스토리 참고)
2. **이전 블로그 톤 확인** — `docs/blog/1-week/BLOG_POST.md` 기준 대화체 톤 유지
3. **마크다운 초안 작성** — `docs/blog/{n}-week/BLOG_POST.md`
4. **HTML 변환** — `docs/blog/{n}-week/BLOG_POST.html` (스타일드 HTML)

## 콘텐츠 구조 (글 뼈대)

### 필수 구성요소

| 순서 | 섹션 | 설명 |
|------|------|------|
| 1 | **헤더** | 제목 + 한 줄 부제 (subtitle) |
| 2 | **TL;DR** | 3줄 이내 핵심 요약 |
| 3 | **본문 섹션들** | 고민 → 선택지 → 결정 → 깨달음 흐름 반복 |
| 4 | **정리 비교표** | 도메인별/항목별 비교 테이블 |
| 5 | **마무리** | 인사이트 인용구로 마감 |

### 본문 섹션 패턴

각 섹션은 다음 흐름을 따른다:

```
h2: 주제 (질문 형태 권장)
  → 기존 상태 / 처음 생각
  → 문제 발견 (멘토링 피드백, 실험 결과 등)
  → 코드 before/after 비교
  → 동시성 테스트 등 검증 코드
  → 깨달음 / 결론
```

## 톤 & 스타일 규칙

### Do

- 대화체, 1인칭 경험 중심 서술
- "처음엔 ~라고 생각했는데, 실제로는 ~" 패턴
- 멘토/수강생 피드백은 인용구(`blockquote`)로 표시하고 출처 명시
- 독자에게 직접 말 걸기 ("~해보셨나요?", "~인 거죠.")
- 개인적 실수/깨달음을 솔직하게 서술

### Don't

- 이모지 사용하지 않음 (HTML 블로그 기준)
- 교과서 톤 ("~이다.", "~해야 한다.") 지양
- 기술 용어만 나열하는 설명 지양 — 항상 "왜"를 함께 서술

## 참고 소스 경로

| 파일 | 경로 |
|------|------|
| 1주차 블로그 (톤 참고) | `docs/blog/1-week/BLOG_POST.md` |
| 대화 로그 | `docs/logs/{n}-weeks-talk-log.md` |
| 멘토링 문서 | `docs/mentoring/{n}-weeks-mentoring.md` |
| HTML 디자인 참고 | `docs/blog/4-week/BLOG_POST.html` |

## HTML 디자인 템플릿 (티스토리용)

티스토리 HTML 에디터에 직접 붙여넣는 형태로 작성한다.
- `<!DOCTYPE html>`, `<head>`, `<body>` 없이 `<style>` + 콘텐츠 HTML만 작성
- CSS 변수(`var()`) 사용하지 않음 — 티스토리 호환성을 위해 직접 값 사용
- 모든 CSS를 `.container` 클래스 하위로 스코핑하여 티스토리 기본 스타일과 충돌 방지
- 티스토리 전용 속성(`data-ke-size`, `data-ke-style`, `data-ke-align`)을 HTML 요소에 유지

### 전체 HTML 뼈대

```html
<style>
/* ── 아래 CSS 전체를 복사 ── */
</style>

<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">&nbsp;</p>
<div class="container">

  <!-- Header -->
  <header class="post-header">
    <h1>{제목}<br />({부제})</h1>
    <p class="subtitle" data-ke-size="size16">{한 줄 요약}</p>
  </header>

  <!-- TL;DR -->
  <div class="tldr">
    <blockquote data-ke-style="style3">
      <span style="color: #006dd7;">TL;DR</span><br />
      {핵심 요약 1}<br />
      {핵심 요약 2}<br />
      {핵심 요약 3}
    </blockquote>
  </div>

  <hr data-ke-style="style1" />

  <!-- 본문 섹션 반복 -->
  <h2 data-ke-size="size26">{섹션 제목}</h2>
  <p data-ke-size="size16">...</p>

  <!-- 코드 블록 -->
  <span class="code-label">kotlin &mdash; 파일명.kt</span>
  <pre class="kotlin"><code>코드 내용</code></pre>

  <!-- 섹션 사이 구분 -->
  <hr data-ke-style="style1" />

</div>
```

### CSS 전체 (복사용)

```css
/* ===== 전체 레이아웃 ===== */
.container {
    max-width: 100%;
    margin: 0 auto;
    line-height: 1.85;
    color: #1a1a1a;
    word-break: keep-all;
}

/* ===== 헤더 ===== */
.post-header {
    text-align: center;
    padding: 20px 0 10px;
    margin-bottom: 8px;
}

.post-header h1 {
    font-size: 1.7em;
    font-weight: 800;
    line-height: 1.4;
    color: #111;
    margin-bottom: 12px;
    letter-spacing: -0.02em;
}

.subtitle {
    font-size: 1.05em;
    color: #666;
    font-style: italic;
}

/* ===== TL;DR 섹션 ===== */
.tldr blockquote {
    background: linear-gradient(135deg, #f0f7ff 0%, #e8f4fd 100%);
    border-left: 4px solid #006dd7;
    border-radius: 0 8px 8px 0;
    padding: 20px 24px;
    margin: 16px 0;
    font-size: 0.97em;
    line-height: 1.9;
}

/* ===== 코드 라벨 ===== */
.code-label {
    display: inline-block;
    background: #2d2d2d;
    color: #ccc;
    font-size: 0.78em;
    font-family: 'Menlo', 'Consolas', 'Courier New', monospace;
    padding: 4px 14px;
    border-radius: 6px 6px 0 0;
    margin-top: 18px;
    margin-bottom: -2px;
    letter-spacing: 0.03em;
}

/* ===== 코드 블록 ===== */
.container pre {
    background: #1e1e1e !important;
    color: #d4d4d4 !important;
    border-radius: 0 6px 6px 6px;
    padding: 20px 22px;
    margin-top: 0;
    margin-bottom: 18px;
    overflow-x: auto;
    font-size: 0.88em;
    line-height: 1.65;
    border: 1px solid #333;
}

.container pre code {
    background: transparent !important;
    color: inherit !important;
    padding: 0;
    font-family: 'Menlo', 'Consolas', 'Courier New', monospace;
}

/* code-label 없는 단독 pre는 둥근 모서리 전부 적용 */
.container p + pre,
.container blockquote + pre {
    border-radius: 6px;
}

/* ===== 인라인 코드 ===== */
.container code {
    background: #f1f3f5;
    color: #c7254e;
    padding: 2px 7px;
    border-radius: 4px;
    font-size: 0.88em;
    font-family: 'Menlo', 'Consolas', 'Courier New', monospace;
}

/* ===== 테이블 ===== */
.container table {
    width: 100%;
    border-collapse: collapse;
    margin: 20px 0;
    font-size: 0.93em;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 1px 4px rgba(0,0,0,0.08);
}

.container table thead tr {
    background: #2c3e50;
    color: #fff;
}

.container table th {
    padding: 13px 16px;
    text-align: left;
    font-weight: 600;
    font-size: 0.92em;
    letter-spacing: 0.02em;
}

.container table td {
    padding: 12px 16px;
    border-bottom: 1px solid #e9ecef;
    vertical-align: top;
}

.container table tbody tr:nth-child(even) {
    background: #f8f9fa;
}

.container table tbody tr:hover {
    background: #e9f5ff;
    transition: background 0.15s ease;
}

/* ===== 시나리오 박스 ===== */
.scenario {
    background: #fafafa;
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    padding: 20px 24px;
    margin: 18px 0;
}

.step {
    padding: 10px 16px;
    margin: 6px 0;
    border-radius: 6px;
    font-size: 0.94em;
    background: #fff;
    border-left: 4px solid #adb5bd;
    color: #333;
}

.step.success {
    background: #f0fdf4;
    border-left-color: #22c55e;
    color: #15803d;
    font-weight: 500;
}

.step.fail {
    background: #fef2f2;
    border-left-color: #ef4444;
    color: #b91c1c;
    font-weight: 500;
}

/* ===== 인용구 ===== */
.container blockquote {
    border-left: 3px solid #3b82f6;
    background: #f8fafc;
    margin: 16px 0;
    padding: 14px 20px;
    border-radius: 0 6px 6px 0;
    color: #374151;
}

.container blockquote i {
    color: #555;
}

.container blockquote b {
    color: #1e3a5f;
}

/* ===== 리스트 ===== */
.container ul {
    margin: 14px 0;
    padding-left: 24px;
}

.container ul li {
    margin-bottom: 8px;
    line-height: 1.8;
}

/* ===== 구분선 ===== */
.container hr {
    border: none;
    height: 1px;
    background: linear-gradient(to right, transparent, #d1d5db, transparent);
    margin: 36px 0;
}

/* ===== h2 섹션 제목 ===== */
.container h2 {
    font-size: 1.45em;
    font-weight: 700;
    color: #111;
    margin-top: 10px;
    margin-bottom: 16px;
    padding-bottom: 8px;
    border-bottom: 2px solid #3b82f6;
    letter-spacing: -0.01em;
}

/* ===== h3 소제목 ===== */
.container h3 {
    font-size: 1.15em;
    font-weight: 600;
    color: #1e3a5f;
    margin-top: 28px;
    margin-bottom: 12px;
}

/* ===== 강조 ===== */
.container b {
    color: #111;
}

/* ===== source 클래스 ===== */
.source {
    font-size: 0.85em;
    color: #999;
    text-align: right;
}
```

### HTML 컴포넌트 사용법

#### 인용구 (멘토/수강생 피드백)

```html
<blockquote data-ke-style="style1">
    <p data-ke-size="size16">"인용 내용"</p>
</blockquote>
```

강조 인용 (깨달음, 핵심 메시지):

```html
<blockquote data-ke-style="style1">
    <p data-ke-size="size16"><b>"강조 인용 내용"</b></p>
</blockquote>
```

내면 독백 스타일:

```html
<blockquote data-ke-style="style1">
    <p data-ke-size="size16"><i>"독백 내용"</i></p>
</blockquote>
```

#### 코드 블록

코드 라벨 + pre 블록 조합. 티스토리 코드 하이라이팅 클래스(`class="kotlin"`, `class="sql"` 등)를 활용한다.

```html
<span class="code-label">kotlin &mdash; 파일명.kt</span>
<pre class="kotlin"><code>@Transactional
fun methodName(): ReturnType {
    val value = "문자열"  // 주석
    return value
}</code></pre>
```

코드 라벨 없는 단독 블록 (짧은 의사코드 등):

```html
<pre class="yaml"><code>Thread A: 값을 읽음
Thread B: 값을 읽음
Thread A: 업데이트
Thread B: 업데이트 &larr; Lost Update!</code></pre>
```

**pre 태그 class 참고 (티스토리 syntax highlighting):**

| class | 언어 |
|-------|------|
| `kotlin` | Kotlin |
| `sql` | SQL |
| `less` | Kotlin (어노테이션 포함 시 대체) |
| `reasonml` | Kotlin (제네릭 포함 시 대체) |
| `yaml` | 의사코드, 시퀀스 설명 |
| `angelscript` | 번호 리스트형 의사코드 |
| `bash` | 쉘, 단계 설명 |

#### 시나리오 박스 (단계별 흐름 시각화)

```html
<div class="scenario">
    <div class="step">1. 정상 단계 설명</div>
    <div class="step">2. 중립 단계</div>
    <div class="step success">3. 성공 케이스</div>
    <div class="step fail">4. 실패 케이스</div>
    <div class="step fail">5. 연쇄 실패</div>
</div>
```

#### TL;DR 카드

```html
<div class="tldr">
    <blockquote data-ke-style="style3">
        <span style="color: #006dd7;">TL;DR</span><br />
        핵심 요약 1<br />
        핵심 요약 2<br />
        핵심 요약 3
    </blockquote>
</div>
```

#### 비교 테이블

```html
<table data-ke-align="alignLeft">
    <thead>
    <tr>
        <th>항목</th>
        <th>A</th>
        <th>B</th>
    </tr>
    </thead>
    <tbody>
    <tr>
        <td><b>레이블</b></td>
        <td>값 A</td>
        <td>값 B</td>
    </tr>
    </tbody>
</table>
```

### 티스토리 HTML 작성 주의사항

- `<p>` 태그에 `data-ke-size="size16"` 속성을 반드시 포함
- `<h2>` 태그에 `data-ke-size="size26"`, `<h3>`에 `data-ke-size="size23"` 포함
- `<hr>` 태그에 `data-ke-style="style1"` 포함
- `<blockquote>` 태그에 `data-ke-style="style1"` 또는 `"style3"` 포함
- `<table>` 태그에 `data-ke-align="alignLeft"` 포함
- `<ul>` 태그에 `style="list-style-type: disc;" data-ke-list-type="disc"` 포함
- HTML 엔티티 사용: `→` = `&rarr;`, `—` = `&mdash;`, `←` = `&larr;`, `≈` = `&asymp;`
- 줄바꿈 공백: `<p data-ke-size="size16">&nbsp;</p>` 사용
- 이미지는 티스토리 이미지 문법(`[##_Image|..._##]`)으로 삽입
