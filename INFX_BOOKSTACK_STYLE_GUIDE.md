# BookStack 문서 작성 스타일 가이드

## 개요

BookStack 공식 문서를 분석하여 도출한 작성 스타일 가이드입니다. 이 가이드를 따르면 BookStack 스타일에 맞는 일관성 있는 문서를 작성할 수 있습니다.

---

## 1. 문서 구조

### 1.1 제목 체계

BookStack에서는 **페이지 제목이 H1 역할**을 합니다. 따라서 Markdown 본문에서는 H1(`#`)을 사용하지 않습니다.

| 레벨 | Markdown | 용도 |
|------|----------|------|
| H1 | 사용 안 함 | 페이지 제목이 H1 역할 |
| H2 | `##` | 대분류 섹션 (필요시) |
| H3 | `###` | 중분류 섹션 |
| H4 | `####` | 소분류 섹션, 일반적인 섹션 제목 |

<p class="callout info">대부분의 문서에서는 <code>####</code>(H4)를 주요 섹션 제목으로 사용합니다. 문서 규모가 클 때만 H2, H3를 상위 구조로 활용합니다.</p>

### 1.2 페이지 시작

```markdown
개요 문장 1-2줄로 시작. 이 문서가 무엇을 다루는지 명확히 설명.

* [섹션 1](#섹션-1)
* [섹션 2](#섹션-2)
* [섹션 3](#섹션-3)

---

#### 섹션 1

내용...
```

### 1.3 섹션 구분

- 주요 섹션 간에는 `---` 수평선 사용
- 연관된 내용은 함께 그룹화
- 논리적 순서: 개념 설명 → 사용법 → 예제 → 고급 내용

---

## 2. 문장 스타일

### 2.1 기본 원칙

- **간결성**: 한 문장은 1-2개 아이디어만 포함
- **직접성**: 불필요한 수식어 제거
- **명확성**: 기술 용어는 일관되게 사용
- **능동태**: 수동태보다 능동태 선호

### 2.2 문장 길이

- 일반 문장: 15-25단어
- 설명 문장: 최대 30단어
- 리스트 항목: 10-15단어

### 2.3 어조

- 전문적이지만 친근함
- 독자에게 직접 말하는 형태
- 명령형 사용 (예: "클릭하세요" / "실행합니다")

---

## 3. 목록 사용

### 3.1 번호 목록 (Ordered List)

**사용 시기:**
- 순차적 단계 (설치, 설정, 절차)
- 우선순위가 있는 항목
- 특정 순서를 따라야 하는 작업

**예시:**
```markdown
1. 저장소를 복제합니다
2. `composer install` 실행
3. `.env` 파일 설정
4. 데이터베이스 마이그레이션 실행
```

### 3.2 불릿 목록 (Unordered List)

**사용 시기:**
- 기능 나열
- 요구사항 목록
- 옵션 설명
- 순서가 중요하지 않은 항목

**예시:**
```markdown
- Biscuits
    - What kind? Digestive?
- Bananas
- Oranges
- Teabags
```

### 3.3 중첩 목록

- 최대 2단계까지만 중첩
- 각 레벨은 의미적으로 구분되어야 함

---

## 4. 코드 표현

### 4.1 인라인 코드

**사용 대상:**
- 명령어: `composer install`
- 파일명: `.env`
- 디렉토리 경로: `/var/www/bookstack`
- 설정 옵션: `APP_URL`
- 짧은 코드 조각: `php artisan migrate`

**형식:**
```markdown
`php artisan key:generate` 명령으로 애플리케이션 키를 생성합니다.
```

### 4.2 코드 블록

**사용 대상:**
- 여러 줄 명령어
- 설정 파일 내용
- 스크립트
- SQL 쿼리

**형식:**
````markdown
```PHP
/**
 * Get a path to a theme resource.
 */
function theme_path(string $path = ''): ?string
{
    $theme = config('view.theme');
    return base_path('themes/' . $theme);
}
```
````

### 4.3 코드 블록 언어 지정

- `bash`: 쉘 명령어
- `PHP`: PHP 코드
- `javascript`: JavaScript 코드
- `sql`: SQL 쿼리
- `yaml`: YAML 설정
- `json`: JSON 데이터
- `nginx`: Nginx 설정
- `apache`: Apache 설정

---

## 5. 강조 및 포맷팅

### 5.1 볼드 (Bold)

**사용 대상:**
- 중요 개념 첫 등장
- UI 요소 이름
- 경고성 키워드

**예시:**
```markdown
**Modi adipisci.** Cupidatat adipisicing so inventore.
BookStack uses **MySQL** or **MariaDB** for database storage.
```

### 5.2 볼드 + 이탤릭

특별히 강조가 필요한 경우:
```markdown
***so inventore*** 또는 ***Nostrud ab or dicta for ad***
```

### 5.3 기타 텍스트 스타일

BookStack WYSIWYG에서 지원하는 스타일 (HTML 사용):
```html
<span style="background-color: #ffcc00;">하이라이트 텍스트</span>
<span style="color: #339966;">컬러 텍스트</span>
<span style="text-decoration: underline;">밑줄 텍스트</span>
<span style="text-decoration: line-through;">취소선 텍스트</span>
```

---

## 6. 링크 처리

### 6.1 내부 링크

**앵커 링크:**
```markdown
자세한 내용은 [설치](#설치) 섹션을 참고하세요.
```

### 6.2 외부 링크

```markdown
[공식 웹사이트](https://example.com)에서 다운로드하세요.
```

### 6.3 링크 스타일

- 링크 텍스트는 명확하고 설명적으로
- "여기를 클릭"이나 "이 링크" 사용 금지
- URL은 가능한 숨기고 의미있는 텍스트로 표현

---

## 7. Callout (알림 블록)

BookStack은 중요한 정보를 강조하기 위한 callout 블록을 지원합니다.

### 7.1 Callout 종류

| 종류 | 용도 | 색상 |
|------|------|------|
| `info` | 일반 정보, 참고사항, 팁 | 파란색 |
| `success` | 성공, 완료, 권장사항 | 초록색 |
| `warning` | 주의사항, 잠재적 문제 | 노란색 |
| `danger` | 위험, 금지사항, 심각한 경고 | 빨간색 |

### 7.2 Callout 문법

```html
<p class="callout info">일반 정보나 참고사항을 여기에 작성합니다.</p>

<p class="callout success">성공 메시지나 권장사항을 여기에 작성합니다.</p>

<p class="callout warning">주의가 필요한 내용을 여기에 작성합니다.</p>

<p class="callout danger">위험하거나 금지된 내용을 여기에 작성합니다.</p>
```

### 7.3 여러 줄 Callout

여러 줄이 필요한 경우 `<div>` 태그를 사용합니다:

```html
<div class="callout warning">
<p>첫 번째 문단입니다.</p>
<p>두 번째 문단입니다.</p>
</div>
```

### 7.4 Callout 사용 가이드라인

| 종류 | 사용 상황 |
|------|----------|
| `info` | 배경 지식, 유용한 팁, 선택적 정보 |
| `success` | 모범 사례, 권장 설정, 완료 확인 |
| `warning` | 데이터 손실 가능성, 호환성 이슈, 주의 필요 설정 |
| `danger` | 보안 경고, 되돌릴 수 없는 작업, 시스템 영향 |

---

## 8. 표 (Table)

### 8.1 Markdown 표

간단한 데이터 표시용:

```markdown
| 단축키 | 기능 |
|--------|------|
| `Ctrl + S` | 저장 |
| `Ctrl + Z` | 실행 취소 |
```

### 8.2 HTML 표

복잡한 레이아웃이나 스타일이 필요한 경우:

```html
<table border="1" style="border-collapse: collapse; width: 100%;">
<thead>
<tr>
<td style="width: 25%;">Cats</td>
<td style="width: 25%;">Dogs</td>
<td style="width: 25%;">Fish</td>
<td style="width: 25%;">Other</td>
</tr>
</thead>
<tbody>
<tr>
<td>105</td>
<td>101</td>
<td>50</td>
<td>3</td>
</tr>
</tbody>
</table>
```

---

## 9. 인용문 (Blockquote)

인용이나 강조된 텍스트 블록에 사용:

```markdown
> As a quote example: Chocolate caramel digestives are possibly the best type of biscuit there is. The caramel provides a level of structural integrity that suitably perfect for dipping into a hot mug of tea.
```

---

## 10. 이미지

### 10.1 기본 이미지

```markdown
![이미지 설명](이미지_URL)
```

### 10.2 클릭 가능한 이미지 (원본 보기)

```markdown
[![이미지 설명](썸네일_URL)](원본_URL)
```

### 10.3 예시

```markdown
[![nature_image.jpg](https://example.com/scaled-image.jpg)](https://example.com/original-image.jpg)
```

---

## 11. 작성 체크리스트

### 시작 전

- [ ] 문서 목적 명확히 정의
- [ ] 대상 독자 파악 (초보자/관리자/개발자)
- [ ] 필요한 배경지식 확인

### 작성 중

- [ ] H1 사용 안 함 (페이지 제목이 H1)
- [ ] 주요 섹션은 `####`(H4) 사용
- [ ] 섹션 간 `---` 수평선 구분
- [ ] 코드는 백틱으로 감싸기
- [ ] 명령어는 언어 지정하여 코드 블록 사용
- [ ] 중요 알림은 Callout 사용
- [ ] 번호 목록은 순차 작업에만
- [ ] 불릿 목록은 비순차 항목에

### 완료 후

- [ ] 모든 링크 작동 확인
- [ ] 코드 예제 테스트
- [ ] 오타 및 문법 검사
- [ ] 일관성 확인 (용어, 스타일)

---

## 12. 용어 일관성

### 12.1 공식 용어

- BookStack (B, S 대문자)
- PHP (모두 대문자)
- MySQL / MariaDB (공식 표기법)
- Composer (첫 글자 대문자)
- Windows Terminal (공식 표기)

### 12.2 파일/디렉토리

- `.env` (점 포함)
- `composer.json` (소문자)
- `/var/www/` (슬래시 포함)

### 12.3 명령어

- `php artisan` (소문자)
- `composer install` (소문자)
- `winver` (소문자)

---

## 13. 실전 예시

### 나쁜 예시

```markdown
# Windows Terminal 설치 가이드

## 요구사항

Windows 10이 필요합니다.

**Note:** winver로 확인하세요.
```

**문제점:**
- `#` H1 사용 (페이지 제목과 중복)
- `##` H2가 너무 큼
- Callout 대신 볼드 텍스트 사용

### 좋은 예시

```markdown
폐쇄망 환경에서 포터블 버전 Windows Terminal을 설치하고 사용하는 방법을 설명합니다.

* [요구사항](#요구사항)
* [설치](#설치)
* [실행](#실행)

---

#### 요구사항

Windows Terminal을 사용하려면 다음 조건을 충족해야 합니다:

* **Windows 10** 버전 1903 이상 또는 **Windows 11**
* 소스 파일 접근 권한

<p class="callout info"><code>winver</code> 명령으로 현재 Windows 버전을 확인할 수 있습니다.</p>

---

#### 설치

1. 소스 폴더를 로컬 경로로 복사합니다:

```
W:\1_Program\WindowsTerminal
```

2. 환경변수에 경로를 추가합니다.

<p class="callout warning">환경변수 변경 후에는 터미널을 다시 열어야 적용됩니다.</p>
```

---

## 14. 편집기 단축키

| 단축키 | 기능 |
|--------|------|
| `Ctrl+1~4` | 헤더 크기 조절 (H1~H4) |
| `Ctrl+5` | 일반 단락 |
| `Ctrl+6` | 블록인용 |
| `Ctrl+7` | 코드 블록 |
| `Ctrl+8` | 인라인 코드 |
| `Ctrl+9` | 콜아웃 (반복 누르면 info → success → warning → danger 순환) |
| `Ctrl+B` | 굵게 |
| `Ctrl+I` | 기울임 |
| `Ctrl+U` | 밑줄 |
| `Ctrl+K` | 링크 삽입 |
| `Ctrl+S` | 저장 |

---

## 15. WYSIWYG 편집기 HTML 지원

### 15.1 허용되는 HTML 태그

BookStack은 보안을 위해 HTML을 정제(sanitize)합니다. 다음 태그는 편집기에서 사용할 수 있습니다.

| 카테고리 | 허용 태그 |
|----------|----------|
| 텍스트 포맷 | `<strong>`, `<b>`, `<em>`, `<i>`, `<u>`, `<s>`, `<del>`, `<sub>`, `<sup>` |
| 제목 | `<h1>` ~ `<h6>` |
| 목록 | `<ul>`, `<ol>`, `<li>` |
| 테이블 | `<table>`, `<thead>`, `<tbody>`, `<tr>`, `<th>`, `<td>` |
| 미디어/링크 | `<a>`, `<img>`, `<figure>`, `<figcaption>` |
| 코드 | `<pre>`, `<code>` |
| 레이아웃 | `<div>`, `<span>`, `<p>`, `<blockquote>`, `<hr>` |

### 15.2 차단되는 HTML 태그

보안상 다음 태그와 속성은 **차단**됩니다.

- ❌ `<script>` — JavaScript 실행
- ❌ `<style>` — 페이지 내 스타일 태그 (Custom HTML Head에서만 가능)
- ❌ `<iframe>` — 외부 콘텐츠 삽입
- ❌ `<object>`, `<embed>` — 플러그인 객체
- ❌ `onclick`, `onload` 등 이벤트 핸들러 속성

### 15.3 인라인 스타일은 허용

`<style>` 태그는 차단되지만, **요소에 직접 `style` 속성을 넣는 것은 허용**됩니다.

```html
<!-- 허용됨 (O) -->
<div style="background: #e3f2fd; padding: 12px; border-left: 4px solid #2196f3;">
  강조할 내용
</div>

<!-- 차단됨 (X) -->
<style>
  .my-class { color: red; }
</style>
```

### 15.4 커스텀 CSS 클래스 사용

관리자가 **Custom HTML Head Content**에 CSS 클래스를 정의해두면, 페이지 본문에서 해당 클래스를 사용할 수 있습니다. WYSIWYG 편집기의 소스 코드 모드(</> 버튼)에서 직접 입력합니다.

```html
<!-- Custom HTML Head에 정의된 클래스를 페이지에서 사용 -->
<div class="infx-card">카드 내용</div>
<span class="infx-badge infx-badge-danger">중요</span>
```

---

## 16. 관리자 커스터마이제이션

BookStack의 시각적 커스터마이제이션은 **관리자 권한**이 필요합니다. 난이도별 3가지 방법이 있습니다.

| 방법 | 난이도 | 위치 | 용도 |
|------|--------|------|------|
| Custom HTML Head Content | ⭐ 쉬움 | Settings > Customization | 전역 CSS/JS, 폰트 변경 |
| Tag Classes | ⭐ 쉬움 | 페이지/책 태그 | 조건부 스타일 적용 |
| Visual Theme System | ⭐⭐⭐ 어려움 | 서버 파일시스템 | 템플릿, 아이콘, 번역 오버라이드 |

### 16.1 Custom HTML Head Content (권장)

**위치**: Settings > Customization > Custom HTML Head Content

모든 페이지의 `<head>`에 삽입되는 HTML입니다. 전역 CSS, 웹폰트, JavaScript를 추가할 수 있습니다.

<p class="callout warning">설정(Settings) 페이지 자체에는 적용되지 않습니다. 이는 잘못된 CSS로 설정 페이지에 접근할 수 없게 되는 상황을 방지하기 위함입니다.</p>

**예시 — 웹폰트 변경:**

```html
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;600;700&display=swap" rel="stylesheet">
<style>
  body {
    --font-body: 'Noto Sans KR', sans-serif;
    --font-heading: 'Noto Sans KR', sans-serif;
    --font-code: 'Source Code Pro', monospace;
  }
</style>
```

<p class="callout info">BookStack은 CSS 변수를 지원합니다. <code>--font-body</code>, <code>--font-heading</code>, <code>--font-code</code>로 폰트를 전역 변경할 수 있습니다.</p>

**예시 — inFX 커스텀 CSS 컴포넌트 등록:**

```html
<style>
  /* === inFX BookStack 커스텀 컴포넌트 === */

  /* 카드 */
  .infx-card {
    background: #ffffff;
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    padding: 20px;
    margin: 16px 0;
    box-shadow: 0 2px 4px rgba(0,0,0,0.08);
  }

  .infx-card-header {
    font-size: 18px;
    font-weight: 600;
    color: #2c3e50;
    margin-bottom: 12px;
    padding-bottom: 8px;
    border-bottom: 2px solid #4a90d9;
  }

  /* 배지 */
  .infx-badge {
    display: inline-block;
    padding: 2px 8px;
    font-size: 11px;
    font-weight: 600;
    border-radius: 3px;
    text-transform: uppercase;
    vertical-align: middle;
  }

  .infx-badge-primary { background: #4a90d9; color: white; }
  .infx-badge-success { background: #27ae60; color: white; }
  .infx-badge-warning { background: #f39c12; color: white; }
  .infx-badge-danger  { background: #e74c3c; color: white; }
  .infx-badge-default { background: #95a5a6; color: white; }
</style>
```

### 16.2 Tag Classes (조건부 스타일)

BookStack은 페이지/책/챕터에 부여한 태그를 CSS 클래스로 자동 변환하여 `<body>`에 적용합니다.

**클래스 생성 규칙:**

- 태그 이름: `tag-name-{name}`
- 태그 값: `tag-value-{value}`
- 이름+값 쌍: `tag-pair-{name}-{value}`
- 텍스트는 소문자화, 공백/하이픈 제거

**예시:**

태그 `Priority: Critical`을 가진 페이지에 특별 스타일 적용:

```html
<!-- Custom HTML Head Content에 추가 -->
<style>
  .tag-pair-priority-critical .page-content {
    border-left: 4px solid #e74c3c;
  }
</style>
```

### 16.3 Visual Theme System (고급)

서버 파일시스템에 직접 접근하여 테마를 구성하는 방법입니다.

**설정 방법:**

1. BookStack 설치 경로에 `themes/{테마명}/` 폴더 생성
2. `.env` 파일에 `APP_THEME={테마명}` 추가

**폴더 구조:**

```
themes/infx/
├── public/           # CSS, JS, 이미지 (/theme/infx/... URL로 접근)
│   ├── css/
│   │   └── custom.css
│   └── images/
│       └── logo.png
├── common/           # Blade 템플릿 오버라이드
│   └── custom-head.blade.php
├── icons/            # SVG 아이콘 오버라이드
└── lang/             # 번역 텍스트 오버라이드
    └── ko/
        └── common.php
```

<p class="callout warning">Blade 템플릿 오버라이드는 BookStack 업데이트 시 깨질 수 있습니다. 가능하면 Custom HTML Head Content 방식을 우선 사용하세요.</p>

### 16.4 기본 UI 설정

Settings > Customization에서 코드 없이 변경 가능한 항목:

- 애플리케이션 이름
- 로고 이미지
- 핵심 색상 (Primary, Link, Page/Draft/Chapter/Book/Shelf 색상)
- 기본 다크 모드 (`.env`에 `APP_DEFAULT_DARK_MODE=true`)

---

## 17. 고급 스타일링 패턴

Custom HTML Head에 CSS를 등록해두면 페이지 본문에서 클래스로 사용할 수 있습니다. 아래는 inFX에서 권장하는 컴포넌트 패턴입니다.

### 17.1 카드 (Card)

**Custom HTML Head에 등록:**

```html
<style>
  .infx-card {
    background: #ffffff;
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    padding: 20px;
    margin: 16px 0;
    box-shadow: 0 2px 4px rgba(0,0,0,0.08);
  }
  .infx-card-header {
    font-size: 18px;
    font-weight: 600;
    color: #2c3e50;
    margin-bottom: 12px;
    padding-bottom: 8px;
    border-bottom: 2px solid #4a90d9;
  }
  .infx-card-body { color: #555; line-height: 1.6; }
  .infx-card-footer {
    margin-top: 12px;
    padding-top: 8px;
    border-top: 1px solid #eee;
    font-size: 12px;
    color: #888;
  }
  .infx-card-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
    gap: 20px;
    margin: 20px 0;
  }
</style>
```

**페이지에서 사용:**

```html
<div class="infx-card">
  <div class="infx-card-header">카드 제목</div>
  <div class="infx-card-body">카드 본문 내용을 여기에 작성합니다.</div>
  <div class="infx-card-footer">추가 정보</div>
</div>

<!-- 그리드 레이아웃 -->
<div class="infx-card-grid">
  <div class="infx-card">
    <div class="infx-card-header">항목 1</div>
    <div class="infx-card-body">설명...</div>
  </div>
  <div class="infx-card">
    <div class="infx-card-header">항목 2</div>
    <div class="infx-card-body">설명...</div>
  </div>
</div>
```

### 17.2 배지 (Badge)

**Custom HTML Head에 등록:**

```html
<style>
  .infx-badge {
    display: inline-block;
    padding: 2px 8px;
    font-size: 11px;
    font-weight: 600;
    border-radius: 3px;
    text-transform: uppercase;
    vertical-align: middle;
  }
  .infx-badge-primary { background: #4a90d9; color: white; }
  .infx-badge-success { background: #27ae60; color: white; }
  .infx-badge-warning { background: #f39c12; color: white; }
  .infx-badge-danger  { background: #e74c3c; color: white; }
  .infx-badge-default { background: #95a5a6; color: white; }
</style>
```

**페이지에서 사용:**

```html
<span class="infx-badge infx-badge-danger">필수</span>
<span class="infx-badge infx-badge-success">완료</span>
<span class="infx-badge infx-badge-warning">검토중</span>
<span class="infx-badge infx-badge-default">선택</span>
```

### 17.3 타임라인 (Timeline)

**Custom HTML Head에 등록:**

```html
<style>
  .infx-timeline {
    position: relative;
    padding-left: 30px;
    margin: 20px 0;
  }
  .infx-timeline::before {
    content: '';
    position: absolute;
    left: 10px;
    top: 0;
    bottom: 0;
    width: 2px;
    background: #4a90d9;
  }
  .infx-timeline-item {
    position: relative;
    margin-bottom: 20px;
  }
  .infx-timeline-item::before {
    content: '';
    position: absolute;
    left: -24px;
    top: 4px;
    width: 12px;
    height: 12px;
    background: #4a90d9;
    border-radius: 50%;
    border: 2px solid white;
  }
  .infx-timeline-date {
    font-size: 12px;
    color: #888;
    margin-bottom: 4px;
  }
  .infx-timeline-title {
    font-weight: 600;
    color: #2c3e50;
  }
</style>
```

**페이지에서 사용:**

```html
<div class="infx-timeline">
  <div class="infx-timeline-item">
    <div class="infx-timeline-date">2026-03-18</div>
    <div class="infx-timeline-title">Flova 2.0 릴리스</div>
    <p>새로운 UI 프레임워크 적용 완료</p>
  </div>
  <div class="infx-timeline-item">
    <div class="infx-timeline-date">2026-03-10</div>
    <div class="infx-timeline-title">QA 테스트 완료</div>
    <p>모든 DCC 통합 테스트 통과</p>
  </div>
</div>
```

### 17.4 스텝 프로세스 (Steps)

**Custom HTML Head에 등록:**

```html
<style>
  .infx-steps {
    display: flex;
    gap: 16px;
    margin: 20px 0;
    flex-wrap: wrap;
  }
  .infx-step {
    flex: 1;
    min-width: 120px;
    text-align: center;
    padding: 16px;
    background: #f8f9fa;
    border-radius: 8px;
  }
  .infx-step-number {
    display: inline-block;
    width: 32px;
    height: 32px;
    line-height: 32px;
    background: #4a90d9;
    color: white;
    border-radius: 50%;
    font-weight: bold;
    margin-bottom: 8px;
  }
  .infx-step-title {
    font-weight: 600;
    font-size: 14px;
  }
</style>
```

**페이지에서 사용:**

```html
<div class="infx-steps">
  <div class="infx-step">
    <div class="infx-step-number">1</div>
    <div class="infx-step-title">모델링</div>
  </div>
  <div class="infx-step">
    <div class="infx-step-number">2</div>
    <div class="infx-step-title">리깅</div>
  </div>
  <div class="infx-step">
    <div class="infx-step-number">3</div>
    <div class="infx-step-title">애니메이션</div>
  </div>
  <div class="infx-step">
    <div class="infx-step-number">4</div>
    <div class="infx-step-title">렌더링</div>
  </div>
</div>
```

### 17.5 컬러 헤더 (Colored Header)

**Custom HTML Head에 등록:**

```html
<style>
  .infx-header-blue {
    background: linear-gradient(135deg, #667eea, #764ba2);
    color: white;
    padding: 20px;
    border-radius: 8px;
    margin: 20px 0;
  }
  .infx-header-blue h3,
  .infx-header-blue h4 { color: white; margin: 0; }

  .infx-header-dark {
    background: #2c3e50;
    color: white;
    padding: 20px;
    border-radius: 8px;
    margin: 20px 0;
  }
  .infx-header-dark h3,
  .infx-header-dark h4 { color: white; margin: 0; }
</style>
```

**페이지에서 사용:**

```html
<div class="infx-header-blue">
  <h4>프로젝트 개요</h4>
  <p>이 섹션은 프로젝트의 전체 구조를 설명합니다.</p>
</div>
```

### 17.6 인라인 스타일 패턴 (CSS 등록 없이)

Custom HTML Head에 클래스를 등록하지 않고도, 인라인 스타일만으로 사용할 수 있는 패턴입니다.

**정보 박스:**

```html
<div style="background: #e3f2fd; border-left: 4px solid #2196f3; padding: 12px 16px; margin: 12px 0; border-radius: 4px;">
  <strong>ℹ️ 참고</strong>
  <p style="margin: 8px 0 0 0;">알려드릴 내용을 여기에 작성합니다.</p>
</div>
```

**경고 박스:**

```html
<div style="background: #fff3e0; border-left: 4px solid #ff9800; padding: 12px 16px; margin: 12px 0; border-radius: 4px;">
  <strong>⚠️ 주의</strong>
  <p style="margin: 8px 0 0 0;">주의가 필요한 내용을 여기에 작성합니다.</p>
</div>
```

**2단 레이아웃 (테이블 기반):**

```html
<table style="width: 100%; border-collapse: collapse; border: none;">
  <tr>
    <td style="width: 50%; vertical-align: top; padding: 8px; border: none;">
      <p>왼쪽 열 내용</p>
    </td>
    <td style="width: 50%; vertical-align: top; padding: 8px; border: none;">
      <p>오른쪽 열 내용</p>
    </td>
  </tr>
</table>
```

**캡션 있는 이미지:**

```html
<figure style="margin: 16px 0; text-align: center;">
  <img src="이미지_URL" alt="설명" style="max-width: 100%; height: auto; border-radius: 4px;">
  <figcaption style="color: #666; font-size: 14px; margin-top: 8px;">이미지 설명</figcaption>
</figure>
```

---

## 18. 유용한 Hacks

BookStack 공식 [Hacks 디렉토리](https://www.bookstackapp.com/hacks/)에서 제공하는 확장 기능입니다. Custom HTML Head Content에 추가하여 사용합니다.

| Hack | 용도 | 링크 |
|------|------|------|
| Mermaid Diagrams | 플로차트, 시퀀스 다이어그램 렌더링 | [보기](https://www.bookstackapp.com/hacks/mermaid-viewer/) |
| MathJax / LaTeX | 수학 수식 렌더링 | [보기](https://www.bookstackapp.com/hacks/mathjax-tex/) |
| WYSIWYG Autocompleter | 에디터 자동완성 기능 | [보기](https://www.bookstackapp.com/hacks/wysiwyg-autocompleter/) |
| Footnotes | 각주 기능 | [보기](https://www.bookstackapp.com/hacks/wysiwyg-footnotes/) |

<p class="callout info">Hacks는 Custom HTML Head Content에 <code>&lt;script&gt;</code>를 추가하는 방식이므로, 관리자 권한이 필요합니다. 페이지 본문의 <code>&lt;script&gt;</code>와는 다릅니다.</p>

---

## 19. 마무리

**핵심 원칙:**

1. H1 사용 금지 (페이지 제목이 H1)
2. `####` H4를 주요 섹션 제목으로
3. `---`로 섹션 구분
4. Callout으로 중요 정보 강조
5. 간결하고 명확하게

**스타일링 원칙:**

6. 반복 사용하는 스타일은 Custom HTML Head에 CSS 클래스로 등록
7. 일회성 스타일은 인라인 `style` 속성 사용
8. 모든 커스텀 클래스에 `infx-` 접두사를 붙여 충돌 방지
9. 페이지 본문에 `<style>`, `<script>` 태그 사용 금지 (차단됨)
10. 다크 모드 호환을 고려하여 하드코딩 색상보다 CSS 변수 우선 사용
