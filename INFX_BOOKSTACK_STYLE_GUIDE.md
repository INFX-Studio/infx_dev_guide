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

## 14. 마무리

**핵심 원칙:**

1. H1 사용 금지 (페이지 제목이 H1)
2. `####` H4를 주요 섹션 제목으로
3. `---`로 섹션 구분
4. Callout으로 중요 정보 강조
5. 간결하고 명확하게
