# inFX 개발 가이드 📚

inFX의 Flova 파이프라인 및 각종 개발 작업을 위한 **공식 가이드 문서 저장소**입니다.

---

## 🎯 프로젝트 취지 및 목적

이 저장소는 **단 하나의 원칙**으로 운영됩니다.

> **"사람이 짜든, AI가 짜든, 같은 규칙으로 같은 코드가 나와야 한다."**

### 두 가지 활용 축

```
inFX Dev Guide
├── 🤖 에이전트 코딩 환경설정
│   └── AI 에이전트가 코드 작성 전 이 저장소의 가이드를 읽고 규칙을 따름
└── 👨‍💻 개발자 코딩 레퍼런스
    └── 개발자가 inFX 규칙을 익히고 작업 중 빠르게 참조
```

#### 🤖 에이전트 코딩 (AI-assisted development)

Claude Code, Cursor, Copilot 등 AI 코딩 에이전트의 **환경설정 파일** (예: `CLAUDE.md`, `.cursorrules`)에
이 저장소의 가이드 파일 경로를 지정해두면, AI가 코드를 작성할 때마다 해당 가이드를 읽고
inFX 스타일에 맞는 코드를 생성한다.

→ AI가 멋대로 코드를 짜는 게 아니라, **이 저장소의 규칙 안에서 동작**한다.

#### 👨‍💻 개발자 코딩 (Manual development)

개발자가 새로운 모듈을 작성하거나, 기존 코드를 수정할 때 **레퍼런스로 참조**한다.
신입 온보딩부터 베테랑의 빠른 조회까지 모두 여기서.

---

## 📋 가이드 파일 목록

### 💻 코딩 스타일

| 파일 | 설명 |
|------|------|
| [`INFX_PYTHON_CODING_STYLE_GUIDE.md`](INFX_PYTHON_CODING_STYLE_GUIDE.md) | Python 코딩 스타일 규칙 및 패턴 |
| [`INFX_PYSIDE2_STYLE_GUIDE.md`](INFX_PYSIDE2_STYLE_GUIDE.md) | PySide2 GUI 코드 작성 규칙 |

### 🏗️ 인프라 & 환경

| 파일 | 설명 |
|------|------|
| [`INFX_INFRA_GUIDE.md`](INFX_INFRA_GUIDE.md) | Flova 파이프라인 개발 환경, 경로, 네트워크 구성 |
| [`INFX_PIP_GUIDE.md`](INFX_PIP_GUIDE.md) | Python 패키지 설치 및 관리 방법 |

### 🔌 시스템 & 플러그인

| 파일 | 설명 |
|------|------|
| [`INFX_SHOTGRID_GUIDE.md`](INFX_SHOTGRID_GUIDE.md) | ShotGrid API 연결 및 사용 방법 |
| [`INFX_DEADLINE_PLUGIN_GUIDE.md`](INFX_DEADLINE_PLUGIN_GUIDE.md) | Deadline 렌더 플러그인 개발 규칙 |
| [`INFX_URL_HANDLER_GUIDE.md`](INFX_URL_HANDLER_GUIDE.md) | `flovac://`, `flovaw://` 커스텀 프로토콜 시스템 |
| [`INFX_ATTENDANCE_SYSTEM_GUIDE.md`](INFX_ATTENDANCE_SYSTEM_GUIDE.md) | 출퇴근 SaaS 시스템 구조 및 설계 |

### 📖 문서 & 참고

| 파일 | 설명 |
|------|------|
| [`INFX_BOOKSTACK_STYLE_GUIDE.md`](INFX_BOOKSTACK_STYLE_GUIDE.md) | 사내 위키(BookStack) 문서 작성 규칙 |
| [`APP_UPDATE_GUIDE.md`](APP_UPDATE_GUIDE.md) | 앱 업데이트 화면 구현 기술 및 삽질 기록 |
| [`CODE_SNIPPETS.md`](CODE_SNIPPETS.md) | 바로 사용 가능한 코드 조각 모음 |
| [`INFX_DEV_FAQ.md`](INFX_DEV_FAQ.md) | 개발 중 자주 나오는 질문과 답변 |

---

## 🤖 에이전트 코딩 환경설정 방법

AI 에이전트가 코드를 작성할 때 이 저장소의 규칙을 자동으로 따르게 하려면,
에이전트 환경설정 파일에 각 가이드 파일의 **절대 경로**를 지정한다.

### Claude Code (`CLAUDE.md`) 설정 예시

```markdown
## 코딩 규칙

코드를 작성하기 전에 아래 가이드 파일을 반드시 읽고 규칙을 따를 것.

### 항상 참조할 가이드
- 코딩 스타일: C:/dev/infx/infx_dev_guide/INFX_PYTHON_CODING_STYLE_GUIDE.md
- 인프라/경로:   C:/dev/infx/infx_dev_guide/INFX_INFRA_GUIDE.md

### 작업별 참조 가이드
- PySide2 GUI 작업 시: C:/dev/infx/infx_dev_guide/INFX_PYSIDE2_STYLE_GUIDE.md
- ShotGrid 작업 시:    C:/dev/infx/infx_dev_guide/INFX_SHOTGRID_GUIDE.md
- Deadline 작업 시:    C:/dev/infx/infx_dev_guide/INFX_DEADLINE_PLUGIN_GUIDE.md
- 패키지 관리 시:      C:/dev/infx/infx_dev_guide/INFX_PIP_GUIDE.md
```

### 작업별 가이드 참조 매핑

| 작업 내용 | 참조할 가이드 |
|-----------|--------------|
| 일반 Python 코드 작성 | `INFX_PYTHON_CODING_STYLE_GUIDE.md` |
| PySide2 GUI 개발 | `INFX_PYSIDE2_STYLE_GUIDE.md` |
| ShotGrid API 연동 | `INFX_SHOTGRID_GUIDE.md` |
| Deadline 플러그인 개발 | `INFX_DEADLINE_PLUGIN_GUIDE.md` |
| 패키지 설치/관리 | `INFX_PIP_GUIDE.md` |
| 개발 환경/경로 확인 | `INFX_INFRA_GUIDE.md` |
| URL Handler 관련 | `INFX_URL_HANDLER_GUIDE.md` |
| 문서 작성 (BookStack) | `INFX_BOOKSTACK_STYLE_GUIDE.md` |

---

## 👨‍💻 개발자 활용 방법

### 온보딩 순서 (신입 개발자)

1. [`INFX_INFRA_GUIDE.md`](INFX_INFRA_GUIDE.md) — 개발 환경 및 경로 구조 파악
2. [`INFX_PYTHON_CODING_STYLE_GUIDE.md`](INFX_PYTHON_CODING_STYLE_GUIDE.md) — 코딩 규칙 숙지
3. 담당 모듈에 맞는 가이드 추가 학습

> 💡 **팁**: 에디터에서 이 저장소를 열어두고 작업하면 가이드를 빠르게 전환할 수 있다.

---

## ✍️ 가이드 파일 작성 방법

> 아래 내용은 **권장사항**이다. 필수 규칙이 아니므로 상황에 따라 유연하게 적용한다.

### 파일 형식

- **마크다운(`.md`)으로만 작성한다** — 가이드 파일은 에이전트와 개발자 모두가 읽는 문서이므로,
  마크다운 외의 형식(PDF, Word 등)은 사용하지 않는다.

### 경로 구분자

- **슬래시(`/`)만 사용한다** — Windows의 `\` 대신 `/`를 써야 Windows·Linux 모두에서 혼란 없이 읽힌다.

```
# 올바름
C:/dev/infx/infx_dev_guide/INFX_CODING_STYLE_GUIDE.md

# 잘못됨
C:\dev\infx\infx_dev_guide\INFX_CODING_STYLE_GUIDE.md
```

### 파일 네이밍 규칙

```
{주최자}_{내용}_{내용}_GUIDE.md
```

| 구성 요소 | 설명 | 예시 |
|-----------|------|------|
| **주최자** | 이 가이드를 실행해야 하는 주체 (회사·그룹·팀명) | `INFX`, `TD`, `ART` |
| **내용** | 가이드 주제를 단어 단위로 언더스코어로 구분 | `CODING_STYLE`, `PIP`, `SHOTGRID` |
| **접미사** | 항상 `_GUIDE`로 끝남 | `_GUIDE` |
| **확장자** | 항상 `.md` | `.md` |
| **대소문자** | 확장자를 제외한 파일명 전체 대문자 | `INFX_PIP_GUIDE.md` |

**예시:**
```
INFX_CODING_STYLE_GUIDE.md   # inFX 전체가 따르는 코딩 스타일
TD_DEPLOY_GUIDE.md           # TD팀이 실행하는 배포 가이드
ART_NAMING_GUIDE.md          # 아트팀이 따르는 파일 네이밍 규칙
```

### 문서 구조 권장 순서

```markdown
# {주최자} {주제} 가이드         ← 제목

한 줄 요약 (이 가이드가 무엇인지)  ← 도입부

---

## 1. 개요                       ← 배경/목적
## 2. 규칙 / 방법                ← 핵심 내용
## 3. 예시                       ← 코드·스크린샷 등
## 4. 주의사항 / FAQ             ← 삽질 방지
```

### 본문 내 기호 사용

| 기호 | 용도 |
|------|------|
| 💡 **팁** | 알아두면 유용한 정보 |
| ⚠️ **주의** | 꼭 확인해야 할 중요 사항 |
| 🔥 **경고** | 문제가 발생할 수 있는 위험 요소 |
| 📌 **참고** | 추가 정보나 관련 문서 링크 |

### 가이드 업데이트 시 체크리스트

- [ ] 에이전트 환경설정 파일의 경로가 여전히 유효한가?
- [ ] 변경 이유와 날짜를 문서 하단 또는 커밋 메시지에 기록했는가?
- [ ] 기존 규칙과 충돌하는 내용은 없는가?

> ⚠️ **주의**: 가이드가 바뀌면 AI 에이전트의 코드 생성 결과도 바뀐다. 중요한 변경은 팀에 공지할 것.

---

## 🔗 주요 링크

| 서비스 | URL | 접근 |
|--------|-----|------|
| 🎬 Shotgrid | https://infx.shotgrid.autodesk.com | 🌍 외부 가능 |
| 📦 GitLab | https://git.infx.kr | 🏢 사내 전용 |
| 📖 BookStack (위키) | https://book.infx.kr | 🏢 사내 전용 |

### 개발 환경 경로

```
개발자 로컬:     C:/dev/infx/
배포 경로:       W:/inhouse/
공용 라이브러리: W:/inhouse/flova_libs/
이 저장소:       C:/dev/infx/infx_dev_guide/
```

---

## 📞 문의

- **가이드 관련 문의**: TD팀
- **기술 지원**: 기경훈 TD
- **인프라 문제**: IT팀
