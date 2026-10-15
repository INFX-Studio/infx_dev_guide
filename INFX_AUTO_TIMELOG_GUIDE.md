# 자동 타임로그 종합 가이드

INFX Works **자동 타임로그** 기능의 단일 기준 문서. 작동 알고리즘, 아키텍처, 구현 계획, 구현 상태, 확인된 문제점, 해결된 문제점, 결정 필요 사항을 이 문서에서 일괄 관리. 설계 변경, 진행 상황, 이슈는 반드시 이 문서에 반영.

- 기능 공식 명칭: **자동 타임로그**
- 영문 식별자: `auto_timelog`
- 관련 서비스: INFX Works (`project_timelog_web`), INFX Works 데스크톱 앱 (`desktop/`), Flova (`flova.timelog`)
- 문서 상태: **전사 수집 베타(collect 모드)** 진행 단계 (2026-09-15). 자동 확정(confirm 모드) 코드는 완료, 베타 데이터로 초안 품질 확인 후 전환

---

## 1. 개요

### 1.1 목적

- 현재 프로젝트 타임로그는 사용자가 기억에 의존해 주관적으로 입력
- 자동 타임로그는 작업자 PC의 실제 작업 활동을 수집하고 Project + Main Job 단위로 자동 귀속하여 **타임로그 초안을 자동 생성**
- 자동 귀속된 시간은 시스템이 확정하고 잠금. 사용자는 파서가 인식하지 못한 `미분류` 시간만 다음 날 알림을 받아 직접 입력. 파이프라인 규칙대로 작업하면 입력할 것이 없음

### 1.2 기존 기능과의 구분

| 명칭 | 식별자 | 내용 | 관계 |
|------|--------|------|------|
| 자동 프로젝트 타임로그 | `sg_auto_project_timelog` (HumanUser) | 대상 사용자의 타임로그를 지정된 기본 프로젝트/태스크/Main Job으로 고정 자동 채움. 화면은 읽기 전용 | 운영 중. 활동 측정 없이 고정 값 배정 |
| **자동 타임로그** | `auto_timelog` | 실제 작업 활동을 측정해 프로젝트/태스크별 시간 추정 및 초안 생성 | 이 문서의 대상. 신규 기능 |

- 두 기능은 명칭이 유사하므로 코드, 문서, 대화에서 위 명칭으로 반드시 구분
- 자동 프로젝트 타임로그 대상자(`sg_auto_project_timelog = True`)는 **자동 타임로그 대상에서 제외** (D-3). 에이전트 미가동, 세그먼트 미수집
- 해당 사용자는 매월 초 직전 달 근무일 전체에 대해 `Default Project`·`Default Main Job`으로 TimeLog를 일괄 생성하는 월별 배치로 처리. 이 배치는 자동 프로젝트 타임로그 기능의 일부이며 자동 타임로그 범위 밖 (기존 구현 여부 확인 후 없으면 별도 작업)
- 대상 판정 우선순위: `sg_auto_project_timelog = True`이면 `sg_auto_timelog` 값과 무관하게 제외

### 1.2.1 운영 모드 (2026-09-15 베타 결정)

| 모드 | 설정 | 대상 | 동작 |
|------|------|------|------|
| `collect` (현재, 베타) | `AUTO_TIMELOG_MODE=collect` (기본값) | INFX Works에 로그인한 전 사용자. 자동 프로젝트 타임로그 대상자 포함 | 데스크톱 앱이 실행 중인 동안 수집·업로드. 판정 배치는 초안만 저장(`Confirm Status = draft`), TimeLog 미생성. 타임로그 화면·알림·제출 검증 변화 없음. 관리자 화면에서 초안 vs 수동 입력 일치율 관찰 |
| `confirm` | `AUTO_TIMELOG_MODE=confirm` | HumanUser `Auto Timelog` 대상자 (자동 프로젝트 타임로그 대상자 제외, D-3) | §2.8 자동 확정·잠금, §2.9 미분류 입력 |

- 베타 원칙: 사용자 업무를 방해하지 않고 웍스 오류를 만들지 않는다. 수집 실패는 모두 무시·로그만 남기고, 앱·DCC·웹 어느 쪽도 예외를 사용자에게 노출하지 않는다
- 앱 내 수집 안내 화면은 폐지(D-4 변경). 고지는 전사 공지로만 수행

### 1.3 설계 원칙

1. **자동분 잠금, 미분류분만 입력**: 자동 귀속된 시간은 사용자가 수정할 수 없음. 파서 미인식 `미분류` 시간만 사용자가 14일 이내 직접 입력. 파일럿 단계에서만 자동분도 사용자 확정
2. **단일 컨텍스트 귀속**: 어느 순간에도 사람의 시간은 정확히 하나의 컨텍스트에만 귀속. 시간 중복 계상은 구조적으로 불가
3. **사람 시간과 머신 시간 분리**: 백그라운드 렌더·시뮬레이션은 근무시간이 아닌 머신 시간으로 별도 집계
4. **개인정보 최소 수집**: 창 제목, 프로세스 이름, 파일 경로, 입력 유무만 수집. 스크린샷, 키 입력 내용, 클립보드, 브라우저 URL 본문 수집 금지
5. **기존 원칙 준수**: 비즈니스 데이터는 ShotGrid 저장, Django DB는 세션/인증 전용. 대량 조회 경로는 Redis cache 우선. 폐쇄망이므로 외부 리소스 참조 금지
6. **투명성**: 사용자는 자신의 수집 데이터와 귀속 근거를 언제든 확인 가능

### 1.4 용어

| 용어 | 정의 |
|------|------|
| 에이전트 | 작업자 PC에서 활동을 수집하는 상주 프로그램. INFX Works 데스크톱 앱(Electron) 내부 모듈로 구현 |
| 샘플 | 에이전트가 일정 주기로 기록하는 순간 스냅샷 (포어그라운드 창, idle 시간, 프로세스 목록 등) |
| 컨텍스트 | 샘플에서 판별한 작업 단위. **Project + Main Job** 조합 또는 `미분류`, `idle`. TimeLog는 Task를 기록하지 않으므로 Task 판별 불필요 |
| 세그먼트 | 같은 컨텍스트가 연속된 시간 구간 (시작, 종료, 컨텍스트, 근거) |
| 귀속 | 세그먼트를 특정 Project + Main Job에 배정하는 것 |
| 초안 | 세그먼트를 일별 집계해 만든 타임로그 제안. 확정 전 상태 |
| 자동 확정 | 초안이 자동 확정 조건을 충족해 시스템이 ShotGrid TimeLog로 저장한 상태 |
| 미분류 입력 기한 | 사용자가 미분류 시간을 직접 입력해야 하는 기한. 해당 일자로부터 14일 |
| 머신 시간 | 사람이 조작하지 않는 동안 프로세스가 CPU를 점유한 시간 (렌더, 시뮬레이션, 캐시 계산) |
| 미분류 시간 | 감시 대상 DCC에서 파일이 열려 있었으나 Flova 경로 파서가 인식하지 못한 시간. 비DCC 앱·idle 시간은 미분류가 아니라 상속 대상 |

---

## 2. 작동 알고리즘

### 2.1 전체 흐름

```
[작업자 PC]
  DCC 플러그인 ──(현재 파일 경로/컨텍스트 push)──┐
  OS 포어그라운드/입력 감시 ──────────────────────┤
  프로세스/CPU 감시 ──────────────────────────────┤
                                                  ▼
                                      에이전트 샘플러 (N초 주기)
                                                  ▼
                                      로컬 이벤트 로그 (SQLite)
                                                  ▼
                                      세그먼트 병합 → 서버 업로드
[INFX Works 서버]
                                      매핑 규칙 엔진 (Flova 경로 파서 → Project + Main Job)
                                                  ▼
                                      일별 초안 생성 (분 단위 귀속, 상속, 반올림)
                                                  ▼
                                      자동분 확정·잠금 → ShotGrid TimeLog 생성 (기존 submit 경로 재사용)
                                      미분류 시간 있음 → 사용자 입력 요청 알림
[웹 UI]                               자동분은 잠금 표시, 미분류분만 입력 → TimeLog 추가 생성
```

### 2.2 샘플링

- 기본 주기 **5초**

| 항목 | 수집 방법 | 용도 |
|------|-----------|------|
| 포어그라운드 창 | OS API (Windows `GetForegroundWindow`) → 프로세스 이름, 창 제목 | 활성 컨텍스트 판별 |
| 마지막 입력 후 경과 시간 | OS API (`GetLastInputInfo`), 기존 `idle-presence-monitor` 재사용 | idle 판별 |
| 현재 DCC 문서 | DCC 플러그인이 push한 현재 파일 경로 (PID별) | Project + Main Job 매핑 1순위 근거 |
| 프로세스 CPU 사용률 | 감시 대상 프로세스의 CPU 점유율 | 머신 시간 판별 |
| 세션 잠금/절전 | 기존 `idle-presence` power event | 강제 idle 처리 |

- 창 제목은 판별 근거로만 사용. 원문은 서버 전송 금지, 판별 결과와 근거 유형만 전송
- 프로세스 이름은 `non_dcc` 세그먼트에도 기록. 미등록 툴 탐지용 (P-13)

### 2.3 컨텍스트 판별 우선순위

- 먼저 성립하는 규칙 적용

1. **세션 잠금·절전·로그오프** → `idle`
2. **마지막 입력 후 경과 시간 ≥ idle 임계값(기본 10분, D-6)** → `idle`. 단, 포어그라운드가 감시 대상 DCC이고 해당 프로세스 CPU 사용률이 임계값 이상이면 `대기` 컨텍스트(직전 활성 컨텍스트에 귀속). 렌더 대기, 파일 로딩, 플레이백 확인 시간의 idle 오분류 방지
3. **포어그라운드가 감시 대상 DCC, 파서가 파일 경로 인식** → Project + Main Job (`work`)
4. **포어그라운드가 감시 대상 DCC, 파서 인식 실패** → `미분류`. 사용자 입력 대상 ([§2.6](#26-dcc-밖-시간과-미분류-시간-처리-d-7d-11-결정))
   - 예외 (2026-09-21): 문서 경로 훅이 없는 DCC(Photoshop, Unreal, Substance, RealityScan, Gaea 등 `auto_timelog_service.HOOK_CAPABLE_PROCESS_PATTERNS` 밖)에서 경로가 없는 세그먼트는 미분류가 아니라 `non_dcc`처럼 직전 컨텍스트 상속. 훅이 있는 Maya·Nuke·Houdini만 경로 없음 = 훅 미동작으로 보고 미분류 유지
5. **포어그라운드가 비DCC 앱** (브라우저, 메신저, 탐색기, 오피스 등) → `non_dcc`. 초안 생성 시 직전 활성 DCC 컨텍스트로 상속
6. 그 외 → `non_dcc`

- 런처 컨텍스트(env var)는 사용하지 않음. 조사 결과 Flova 런처 bat는 `FLOVA_ROOT`, `PYTHONPATH`, `NUKE_PATH`, `HOUDINI_PATH`, `OCIO`만 설정하며 프로젝트·샷·스텝 env var 규약이 없음. 컨텍스트는 항상 씬 경로에서 파생

### 2.4 다중 DCC 처리

- 여러 DCC 동시 실행 시 **포어그라운드 창 하나**만 사람 시간 귀속 대상
- 백그라운드 DCC는 사람 시간 0
- 백그라운드 DCC의 CPU 사용률이 임계값 이상이면 해당 프로세스의 컨텍스트로 **머신 시간** 별도 기록
- 결과적으로 하루 사람 시간 합계는 실제 활동 시간을 초과 불가

### 2.5 세그먼트 생성

- 연속 샘플의 컨텍스트가 같으면 하나의 세그먼트로 병합
- 짧은 끊김(예: 알림 확인으로 5초간 타 앱 포커스)은 **최소 세그먼트 길이 미만이면 인접 세그먼트로 흡수**. 기본값 60초
- 샘플은 5초마다 로컬 SQLite에 즉시 저장. PC 종료·장애 시 유실은 마지막 5초 이내로 한정
- 서버 업로드는 주기적(기본 5분) + **앱 종료·윈도우 종료·로그아웃 시 즉시 동기 업로드**(최대 3초 대기). 업로드 실패분은 로컬에 남아 다음 앱 시작 시 자동 업로드. 데이터는 지연될 뿐 유실되지 않음
- 서버 업로드는 멱등. 세그먼트에 클라이언트 생성 UUID와 `device_id`(호스트명) 부여

### 2.5.1 다중 기기 병합 (원격 환경)

- 한 사용자가 여러 PC(노트북 + 원격 워크스테이션 등)에서 에이전트를 가동하면 서버가 사용자·시각 기준으로 병합
- 같은 시각에 기기 간 세그먼트가 겹칠 때 우선순위
  1. 파서가 인식한 DCC 컨텍스트가 있는 기기
  2. 둘 다 DCC면 마지막 입력이 있었던 기기
  3. 둘 다 비DCC면 상속 처리
- 가동률은 기기별이 아니라 사용자 기준 합집합. 어느 기기든 샘플이 있으면 가동으로 인정
- 원격 접속 클라이언트(RDP 창, Guacamole 브라우저 탭)는 로컬 기기에서 비DCC 앱으로 보임. 실제 DCC 컨텍스트는 원격 워크스테이션 에이전트가 제공하므로 우선순위 1에 의해 워크스테이션 기준으로 귀속
- 원격 환경별 특성

| 환경 | 세션 특성 | 예상 동작 | PoC 검증 |
|------|-----------|-----------|----------|
| Windows RDP | 콘솔과 분리된 원격 세션. 접속 끊김 시 세션 잠금 이벤트 | 입력·포어그라운드 원격 세션에서 정상 감지. 끊김 → idle, 재접속 → 재개 | 필수 |
| Guacamole (RDP 백엔드) | 서버 측 RDP 세션과 동일 | RDP와 동일 | 필수 |
| Guacamole (VNC 백엔드) | 콘솔 세션 공유. 접속 끊겨도 세션 잠금 없음 | 입력 주입으로 idle 감지 정상. 끊김은 idle 임계값(10분)으로만 감지 | 필수 |
| Parsec 등 스트리밍 | 콘솔 세션 | 로컬과 동일 | 해당 시 |

- 전제: 원격 워크스테이션에 INFX Works 데스크톱 앱과 Flova Python이 설치되어 있어야 함. 미설치 워크스테이션의 작업은 전부 비DCC로 보여 상속 처리됨 (조용한 오귀속). 설치 여부는 §5.4 관리자 목록의 "DCC 세그먼트 0건 사용자"로 탐지

### 2.6 DCC 밖 시간과 미분류 시간 처리 (D-7·D-11 결정)

- 시간 구분

| 구분 | 정의 | 처리 |
|------|------|------|
| 상속 대상 | 비DCC 앱(회의, 리뷰, 메신저, 브라우저, 문서), idle, 대기, 에이전트 미기록 | **직전 활성 DCC 컨텍스트 상속, 상한 없음**. "오늘 마지막으로 작업하던 것이 오늘 한 일" |
| `미분류` | 감시 대상 DCC에서 파일이 열려 있었으나 Flova 경로 파서가 인식하지 못함 (로컬 개인 폴더, 임시 폴더, 규칙 밖 파일명) | 상속하지 않음. **사용자가 직접 입력**. 다음 날 웍스 알림 |

- 직전 활성의 정의: 그날 마지막으로 포어그라운드였던 감시 대상 DCC 중 파서가 인식한 컨텍스트. DCC 종료 여부 무관. 다중 DCC는 마지막 포어그라운드 기준. 미분류 세그먼트는 "직전 활성"이 될 수 없음
- 상속 fallback 사슬

| 상황 | 처리 |
|------|------|
| 인식된 DCC 작업 후 회의·리뷰 등 | 마지막 인식 컨텍스트 |
| 출근 직후 DCC 열기 전 시간 | 그날 첫 인식 컨텍스트로 소급 |
| 그날 인식된 DCC 없음, 미분류도 없음 | 직전 근무일의 마지막 인식 컨텍스트 |
| 이력 없음 (신규 입사 등) | HumanUser `Default Project`·`Default Main Job` |
| 그날 인식된 DCC 없음, 미분류만 있음 | 근무시간 전체가 미분류. 사용자 입력 |

- 미분류가 상속 대상 사이에 끼어 있는 경우: 미분류 앞뒤의 비DCC·idle 시간은 가장 가까운 인식 컨텍스트로 상속. 미분류 구간 자체만 사용자 입력
- 상속된 세그먼트는 `evidence = inherited`로 기록, 근거 타임라인에서 추정 구간으로 구분 표시
- 상속 오귀속(긴 회의가 직전 프로젝트에 귀속)은 수용. 자동분은 수정 불가
- 기각안: 미분류 상속 (로컬 작업이 조용히 오귀속), 비DCC 시간까지 사용자 입력 (매일 입력 발생)

### 2.7 일별 초안 생성 (D-5 결정)

- 귀속 원칙: **출퇴근 인정 근무 구간의 모든 분(minute)은 그 시점의 직전 활성 DCC 컨텍스트에 귀속**. 비DCC·idle·대기·에이전트 미기록 구간 모두 동일 규칙. `미분류` 구간만 예외로 사용자 입력 대상. 비율 정규화 없음

1. 출퇴근 시스템(`flova.timelog.attendance`)에서 해당 일자의 인정 근무 구간(주간·야근, 식사공백 제외)을 가져옴. 출퇴근 규칙은 Attendance 가이드 준수, 이 문서에서 재정의 금지
2. 세그먼트 타임라인을 인정 근무 구간 위에 겹쳐 각 분의 컨텍스트 결정. 인식된 DCC 세그먼트가 없는 분은 직전 활성 컨텍스트 상속 (§2.6). `미분류` 세그먼트 구간은 `미분류`로 유지
3. 컨텍스트별 합산. 주간 구간은 `sg_day_hour`, 야근 인정 구간은 `sg_night_hour`. `미분류` 합계는 별도 보관 (`unclassified_hours`), TimeLog 미생성
4. **반올림**: 0.25시간(15분) 단위. 반올림 후 자동분 합계 + 미분류 합계가 인정 근무시간과 어긋나면 자동분 최대 항목에서 차이 조정. 미분류가 있는 날은 사용자가 입력할 시간 = 인정 근무시간 − 자동분 합계
5. 5분 미만 항목 삭제, 해당 시간은 최대 항목으로 병합
6. 머신 시간은 초안 미포함, 별도 항목으로 표시
7. 실측값 보존: 초안 JSON에 아래 항목 기록. 귀속 결과와 별개이며 통계·규칙 튜닝용

| 항목 | 정의 |
|------|------|
| `measured_hours` | 입력 활동이 있고 DCC 기반 컨텍스트가 판별된 시간 |
| `unclassified_hours` | 파서 미인식 DCC 파일이 열려 있던 시간. 사용자 입력 대상 |
| `non_dcc_hours` | 비DCC 앱 시간. 상속 처리됨 |
| `idle_hours` | 입력 없음(임계값 초과) 시간 |
| `uncovered_hours` | 에이전트 기록 자체가 없는 시간 |
| `evidence_summary` | 사용된 근거 유형 요약. 예: `no_dcc_today, inherited_from_prev_day` |

- 기각안: 비율 정규화 (`태스크 시간 = 활성시간 비율 × 근무시간`). 모든 분이 귀속되므로 불필요
- 그날 인식된 DCC 없음: `measured_hours = 0`으로 보존하고 §2.6 fallback 사슬로 귀속. 미분류만 있는 날은 전부 사용자 입력

### 2.8 자동분 확정 및 잠금

- 판정 시점: 해당 일자의 **다음 근무일 12:00** 일괄. 오전에 PC를 켜면 전날 잔여 세그먼트가 업로드되므로 이 시점까지 대기 (P-12)
- 재판정: 판정 후 지연 업로드 세그먼트가 도착하면 자동 재판정. 단, 사용자가 이미 미분류 입력을 완료한 일자는 재판정하지 않음. 재판정으로 자동 확정 조건이 충족되면 기존 사용자 확정 요청 알림은 취소
- 자동 확정 조건

| 조건 | 기본값 | 미달 시 |
|------|--------|---------|
| 에이전트 가동률 | 출근 인정 시간의 80% 이상 | 자동분 생성 보류, 사용자 확정 요청 알림 (PC 미사용일에 한정) |
| 출퇴근 기록 | 해당 일자 퇴근 기록 존재 | 퇴근 기록 생성 후 재판정 |

- 조건 충족 시 자동분(상속 포함)을 기존 `/api/timelog/submit` 경로로 ShotGrid TimeLog 생성. 자동 타임로그 전용 TimeLog 쓰기 경로 없음
- 자동 생성 TimeLog는 `sg_auto_timelog_source = auto`. **사용자·관리자 모두 수정 불가**. 웹 UI와 제출 API 양쪽에서 잠금
- 자동분 오류(상속 오귀속, 미등록 툴)는 개별 수정이 아니라 규칙·목록 보강으로 대응. 보강 후 재생성은 관리자 기능(§3.5 `rebuild`)으로만 수행

### 2.9 미분류 시간 사용자 입력 (D-11 결정)

- 대상: `unclassified_hours > 0`인 일자
- 알림: 판정 직후 INFX Works 알림 발송. 문안 예: "9월 9일 자동 입력되지 않은 2.5시간이 있습니다. 프로젝트와 업무를 입력하세요"
- 입력: 타임로그 화면에서 해당 일자의 미분류 시간만 프로젝트·Main Job·시간으로 배분. 합계는 미분류 시간과 일치해야 저장 가능
- 저장: 기존 `/api/timelog/submit`으로 TimeLog 생성, `sg_auto_timelog_source = manual_unclassified`
- 기한: 해당 일자로부터 **14일**. 기한 내 미입력은 기존 타임로그 미입력 알림·관리자 미입력 목록에 포함
- 기한 경과 후 입력 불가 (D-9). 해당 일자는 `Confirm Status = expired`로 잠금. 미분류 시간은 영구 미입력으로 남고, 관리자 미입력 통계에 집계
- 미분류 시간이 없는 일자는 사용자에게 아무 요청도 하지 않음

---

## 3. 아키텍처

### 3.1 구성 요소

| 구성 요소 | 위치 | 역할 |
|-----------|------|------|
| 에이전트 | `desktop/src/main/auto-timelog/` (Electron main) | 샘플링, 세그먼트 생성, 로컬 저장, 업로드 |
| 프로브 사이드카 | `flova/timelog/auto_timelog/agent_probe.py` (Python 3.7 문법) | 포어그라운드 창·프로세스·CPU 조회, JSON 스트림 출력 |
| DCC 플러그인 | Flova 파이프라인 DCC 설정 (Maya, Nuke, Houdini 등) | 현재 문서 경로를 에이전트로 push |
| 매핑 규칙 엔진 | `flova/timelog/auto_timelog/` (Python 3.7 문법) | `WorkFile` 파서 결과 → Project + Main Job 변환 |
| 서버 API | `backend/timelog/api_views_auto_timelog.py` | 세그먼트 수신, 초안 생성, 초안 조회 |
| 웹 UI | `frontend/src/components/timelog/AutoTimelog*.vue` | 자동분 잠금 표시, 근거 표시, 미분류 입력 |
| 관리자 화면 | `frontend/src/views/admin/` | 매핑 규칙 관리, 미분류 비율 통계, 정책 설정 |

### 3.2 에이전트

- INFX Works 데스크톱 앱 모듈로 포함. 별도 상주 프로그램 추가 금지
- 기존 `idle-presence-monitor.ts`, `idle-presence-client.ts`의 idle 판별과 power event 재사용
- 포어그라운드 창·프로세스·CPU 조회는 **Python 사이드카** 방식 (D-2 결정)
  - 데스크톱 앱이 Flova Python(`AMI_PYTHONW_PATH`)으로 `flova/timelog/auto_timelog/agent_probe.py`를 상주 실행
  - 사이드카는 5초마다 `ctypes`로 `GetForegroundWindow`, `GetWindowThreadProcessId`, `GetWindowTextW` 호출, `psutil`로 프로세스 이름·CPU 사용률 수집 후 JSON 1줄을 stdout으로 출력
  - 데스크톱 앱 `auto-timelog/probe-process.ts`가 생명주기 관리. 비정상 종료 시 재시작은 `ami-exec-pool.ts`의 respawn 패턴 재사용
  - idle 시간은 기존 Electron `powerMonitor.getSystemIdleTime()` 사용, 사이드카에서 중복 수집하지 않음
  - 기각안: Node 네이티브 addon (Electron 버전별 바이너리 호환, 폐쇄망 반입 절차), PowerShell 주기 호출 (호출당 0.5~1초로 5초 주기 부적합)
- 에이전트–DCC 플러그인 통신은 로컬 loopback HTTP 또는 named pipe. AMI 실행 서버(`ami-exec-server.ts`)와 동일 방식
- 로컬 저장은 SQLite. 업로드 성공 후 30일 경과 데이터 삭제
- 트레이 메뉴에 수집 일시 중지 제공. 중지 구간은 `수집 중지`로 기록, 초안에서 상속 처리
- 앱 업데이트 후 최초 실행 시 §12 안내 문안을 표시하고 "확인" 클릭 시각을 서버에 기록 (D-4). 거부 선택지 없음. 확인 전에는 수집을 시작하지 않음

### 3.3 DCC 플러그인 (조사 반영 2026-09-11)

- 각 DCC의 문서 열기/저장/신규 이벤트에서 현재 파일 경로와 PID를 에이전트로 push
- 훅 공통 모듈: `flova/timelog/auto_timelog/dcc_hook.py` (Python 3.7 문법). DCC별 startup에서는 이 모듈의 등록 함수 1줄만 호출
- 전송: 기존 AMI 실행 named pipe와 같은 방식의 전용 pipe `\\.\pipe\infx-works-auto-timelog`. 연결 timeout 0.2초, 실패 시 무시 (DCC 시작·저장을 절대 막지 않음). 참고 구현 `flova/app/ami_exec/dispatch.py`, `flova/temp/infx_protocol_sender.py`
- 페이로드: `{"pid", "dcc", "doc_path", "event"}`. 컨텍스트 해석은 DCC 안에서 하지 않고 에이전트·서버가 수행
- 주기 재전송 (2026-10-07): 등록 함수가 `dcc_hook.start_heartbeat(dcc)`로 daemon 스레드를 띄워 콜백이 마지막으로 알려준 문서 경로를 5분마다 `open` 이벤트로 다시 보낸다 (`HEARTBEAT_INTERVAL_SEC`). 데스크톱 앱의 PID→경로 레지스트리가 메모리 전용이라 앱 재시작·자동 업데이트 때 사라지고, 훅 등록 전에 열린 세션은 다음 저장까지 경로가 없던 문제(P-18)를 복구한다. 스레드는 DCC API를 호출하지 않고 콜백이 갱신한 캐시 값만 보낸다

| DCC | 부트스트랩 위치 | 훅 삽입 지점 | 이벤트 | 현황 |
|-----|-----------------|--------------|--------|------|
| Maya | `flova/maya/startup/userSetup.py` (`PYTHONPATH` 경유, `flova/exec/maya20xx.bat`) | `init_deadline` 뒤 `cmds.evalDeferred(_register_auto_timelog_hook)` | `scriptJob` `SceneOpened`, `SceneSaved`, `NewSceneOpened` | **연결 완료 2026-09-14** |
| Nuke | `flova/nuke/menu.py` (`NUKE_PATH` 경유, `flova/app/flova_task/utils.py:53`) | `_setup_callbacks()` 끝에 `dcc_hook.register_nuke()` | `addOnScriptLoad`, `addOnScriptSave`, `addOnScriptClose` | **연결 완료 2026-09-14** |
| Houdini | `HOUDINI_PATH=flova/houdini/menus` (`flova/exec/houdini20.5.bat:8`) | `flova/houdini/menus/scripts/456.py` 신규 (세션당 1회 등록 표시) | `hou.hipFile.addEventCallback` | **연결 완료 2026-09-14**. 기존 `flova/houdini/scripts/123.py`는 HOUDINI_PATH 밖이라 여전히 미로드 |
| Katana | `flova/katana` (`KATANA_RESOURCES`) | `UIPlugins/katana_pulldown_menu.py` onStartup 콜백 옆 | Katana scene callbacks | Phase 2 범위 밖, 추후 |
| Hiero | `flova/hiero/Python/Startup/init.py` | 동일 | Nuke API 동일 | Phase 2 범위 밖, 추후 |
| 3ds Max, Blender, After Effects, Photoshop, Substance, Mari | Flova 통합 없음 | 창 제목 파싱 fallback | 제목에 파일 경로가 있는 앱만 | 파일럿 미분류 통계로 우선순위 결정 |
| Unreal, Substance Painter, RealityScan, Gaea | Flova 통합 없음 | 없음 | 없음 | 2026-09-21 감시 대상 추가. 경로 없는 세그먼트는 상속 처리(§2.3 예외) |

- DCC Python 버전: Maya 2022 = 3.7.7 (제약 기준), Maya 2024 = 3.10, Nuke 14/15 = 3.9/3.10, Houdini 20.5 = 3.11, Katana 7 = 3.10. 훅 모듈은 3.7 문법 준수

### 3.3.1 구현 메모 (2026-09-14, Phase 0)

- 구현 파일

| 위치 | 파일 | 역할 |
|------|------|------|
| flova | `flova/timelog/auto_timelog/agent_probe.py` | 사이드카 엔트리. `--interval`, `--watch` 인자. stdin 닫히면 종료 |
| flova | `flova/timelog/auto_timelog/probe_core.py` | WatchList, CpuUsageTracker, 샘플 조립 (순수 로직, 테스트 대상) |
| flova | `flova/timelog/auto_timelog/probe_win32.py` | ctypes Win32 (GetForegroundWindow, Toolhelp32, GetProcessTimes) |
| flova | `flova/timelog/auto_timelog/dcc_hook.py` | named pipe 전송, `register_maya/nuke/houdini()` |
| desktop | `src/main/auto-timelog/config.ts` | 상수(주기 5초, idle 600초, CPU 임계 20%, 최소 세그먼트 60초, 감시 목록) |
| desktop | `src/main/auto-timelog/probe-process.ts` | 사이드카 spawn·파싱·재시작(연속 실패 5회 시 중단) |
| desktop | `src/main/auto-timelog/classify.ts` | 샘플 → 컨텍스트, 머신 시간 후보 |
| desktop | `src/main/auto-timelog/segments.ts` | 세그먼트 병합, 짧은 구간 흡수(`onDiscard`), 시간 공백 분리 |
| desktop | `src/main/auto-timelog/store.ts` | `node:sqlite` 로컬 DB (`%APPDATA%/inFX Works 데스크톱 앱/auto-timelog.sqlite`) |
| desktop | `src/main/auto-timelog/hook-server.ts` | 훅 수신 파이프 서버, PID→문서 경로 레지스트리 |
| desktop | `src/main/auto-timelog/service.ts` | 조립·수명·잠금/절전·일시 중지 |
| desktop | `src/main/index.ts` | 앱 시작 시 서비스 기동, powerMonitor 연결, 트레이 메뉴 |

- 에이전트 컨텍스트 종류는 `dcc`, `non_dcc`, `idle`, `wait`, `paused`, `machine`. 경로 파서 판정(`work`/`project_only`/`unclassified`)은 서버가 수행하므로 에이전트는 문서 경로 유무까지만 기록
- 열린 세그먼트는 샘플마다 스냅샷 upsert. 강제 종료 시에도 마지막 샘플까지 남음
- 사이드카는 `pythonw -u agent_probe.py`로 실행. Python에 psutil이 없어 ctypes만 사용
- 확인 완료: 앱 기동 시 사이드카 시작·훅 파이프 리슨·샘플 저장, Python `dcc_hook.send_payload()` → Node 파이프 수신. Git Bash(MSYS)에서는 파이프 경로가 변형되어 실패하므로 검증은 PowerShell·cmd에서 수행
- 테스트: flova `python -m unittest discover -s flova/timelog/auto_timelog/tests -t .` (18건), desktop `npx vitest run` (auto-timelog 36건)
- Phase 1~5 구현 (2026-09-14)

| 위치 | 파일 | 역할 |
|------|------|------|
| flova | `flova/timelog/auto_timelog/mapping.py` | `PathContextResolver`(WorkFile → work/project_only/unclassified), `StepMainJobTable`(§5.2 표) |
| flova | `flova/timelog/attendance.py` `AttendanceCalculator.get_work_intervals(_from_logs)` | calc_daily와 같은 규칙으로 정규·야근 인정 구간을 구간 목록으로 반환 (자동 타임로그 전용 추가) |
| flova | `flova/shotgrid/partition_schema.py`, `cache.py` | TimeLog `sg_auto_timelog_source`, HumanUser `sg_auto_timelog`·`sg_auto_timelog_notice_at` cache 필드 |
| flova | Maya `userSetup.py`, Nuke `menu.py`, Houdini `menus/scripts/456.py` | `dcc_hook.register_*()` 연결 |
| backend | `timelog/auto_timelog_service.py` | 세그먼트 병합, `AutoTimelogDayRepository`, `DraftBuilder`(분 단위 귀속·상속·반올림·미분류), `judge_day`/`judge_pending_days`(자동 확정, TimeLog 생성), `validate_manual_records_against_auto`, `expire_overdue_days`, 관리자 개요 |
| backend | `timelog/api_views_auto_timelog.py`, `project_timelog_web/urls.py` | §3.5 API |
| backend | `timelog/management/commands/auto_timelog_judge.py` | 판정 배치 (cron 12:00) |
| backend | `timelog/api_views_timelog.py` | 조회 응답에 자동 상태 포함, submit의 auto 잠금·미분류 검증·`manual_unclassified` 태깅 |
| frontend | `api/autoTimelog.ts`, `stores/timelog.ts`, `types/timelog.ts` | 상태·잠금 플래그(`lockedDayFlags`/`lockedNightFlags`) |
| frontend | `components/timelog/DayInput.vue`, `NightInput.vue`, `TimeRow.vue` | 자동 셀 잠금 표시(`data-auto-locked`), 잠긴 행은 콤보·삭제 비활성 |
| frontend | `views/TimelogView.vue` | 미분류 입력 요청·기한 경과 배너 (`data-section="auto-timelog-banner"`) |
| frontend | `views/NotificationView.vue` | "자동 타임로그 입력 요청" 섹션 (`/api/auto-timelog/unclassified/`) |
| frontend | `views/admin/AutoTimelogView.vue`, `config/manage-menu.ts`, `router` | 관리자 > 자동 타임로그 (`/manage/auto-timelog`): 상태 집계, 재판정·판정 실행, 미등록 툴·미인식 경로·사용자별 지표 |
| desktop | `auto-timelog/uploader.ts` | `/status/` 부트스트랩(CSRF), 5분 주기·종료 시 세그먼트 업로드, 403 재부트스트랩 |
| desktop | `auto-timelog/notice-window.ts` | §12 안내 창. "확인" → `/notice/` 기록 후 수집 시작 |
| desktop | `auto-timelog/service.ts` | 서버 상태(`enabled`, 안내 확인)로 수집 게이트. 비대상은 수집 안 함. 1시간마다 재확인 |

- 알림: 별도 알림 엔티티 없이 알림 센터가 `/api/auto-timelog/unclassified/`를 조회해 표시. 타임로그 화면 배너와 동일 데이터
- Main Job 정적 목록(`stores/timelog.ts` `STATIC_MAIN_JOBS`)을 룩뎁·CFX 통합에 맞춰 갱신
- 베타 전환 (2026-09-15): `settings.AUTO_TIMELOG_MODE`, `auto_timelog_service.is_collect_only()/is_auto_confirm_enabled()`, 판정 배치 `draft` 저장, `compare_draft_with_manual()`(초안 vs 수동 TimeLog 일치율), 관리자 화면 모드·일치율 표시. 데스크톱: 안내 창 제거, 업로드 15분 주기·닫힌 세그먼트만. DCC 훅 전송은 daemon 스레드
- 운영 대기: 배포 서버 `start-project-timelog`(cron 설치 포함) 실행, cache rebuild, D-3 월별 배치(범위 밖)

### 3.4 매핑 규칙 엔진

- 판별 근거 우선순위: **DCC 플러그인이 push한 문서 경로 → 창 제목 파싱**
- 파일 경로에서 **프로젝트 루트 폴더**(Project `code`)와 **스텝 폴더명**만 추출. 샷·에셋·시퀀스 구분은 귀속에 불필요 (TimeLog는 Task 미기록)
- 프로젝트 목록·`code`는 Redis cache(`SGProjectCache`) 사용. cache miss 시 ShotGrid 직접 조회 fallback 금지. 매핑 실패는 `미분류`
- 스텝 → Main Job 변환은 `flova.timelog.model.MainJob` 기준 매핑 테이블을 규칙 엔진에 배치
- Flova 내부 구현. Python 3.7.7 문법 준수, 모듈 레벨 함수 없이 클래스 기반 작성

### 3.5 서버 API

| URL | 메서드 | 설명 |
|-----|--------|------|
| `/api/auto-timelog/status/` | GET | 데스크톱 앱 부트스트랩: 대상 여부(`enabled`), 안내 확인 여부, CSRF 토큰 |
| `/api/auto-timelog/notice/` | POST | 안내 화면 확인 시각 기록 (`sg_auto_timelog_notice_at`) |
| `/api/auto-timelog/segments/` | POST | 세그먼트 업로드 (멱등, uuid 기준 upsert). 로컬 일자별 `Auto Timelog Day`에 병합 |
| `/api/auto-timelog/draft/` | GET | 지정 날짜의 세그먼트·초안·상태 (`?date=YYYY-MM-DD`). 근거 타임라인용 |
| `/api/auto-timelog/unclassified/` | GET | 미분류 입력 대기·기한 경과 일자 목록 |
| `/api/admin/auto-timelog/overview/` | GET | 상태 집계, 비DCC 상위 프로세스, 미인식 경로 상위, 사용자별 지표 (`?start&end`) |
| `/api/admin/auto-timelog/rebuild/` | POST | 재판정 (force). `{date, user_id\|null}` |
| `/api/admin/auto-timelog/judge/` | POST | 판정 배치 수동 실행 + 기한 경과 잠금. `{date}` |

- 구현: `backend/timelog/api_views_auto_timelog.py`, 서비스 `backend/timelog/auto_timelog_service.py`
- 판정 배치: `python manage.py auto_timelog_judge` (cron, 근무일 12:00). `--date`, `--user-id`, `--force`, `--days` 옵션
- `/api/timelogs/` 응답에 `is_auto_timelog`, `auto_timelog_days`(날짜→상태·미분류 시간·기한), 항목별 `auto_source` 추가. `/api/timelogs/submit/`은 `sg_auto_timelog_source = auto` 레코드를 교체 대상에서 제외하고, 같은 key 입력은 무시하며, 자동 확정 일자의 수동 입력을 §2.9 규칙으로 검증(400)한 뒤 `manual_unclassified`로 저장하고 `Confirm Status`를 `completed`로 바꾼다

- 응답 형식은 기존 규칙(`{'status': True, 'data': ...}`) 준수
- 자동분 생성은 서비스가 `sg.batch`로 TimeLog를 직접 생성(기존 auto 레코드 삭제 후 재생성, 월 파티션 cache rebuild). 미분류 입력 저장은 기존 `/api/timelogs/submit/`

### 3.6 웹 UI

- 타임로그 화면에 자동 확정된 값이 채워진 상태로 진입. 자동 항목은 수동 입력 항목과 시각적 구분 (`DESIGN_GUIDE.md` 준수)
- 각 항목에 근거 보기 버튼. 클릭 시 세그먼트 타임라인(시간대별 컨텍스트) 표시
- 미분류 시간과 머신 시간은 별도 영역 표시
- 자동 생성 항목은 잠금 아이콘과 함께 읽기 전용. 편집 컨트롤 미표시
- 미분류 시간이 있는 일자는 입력 요청 상태로 강조, 남은 시간과 입력 기한 잔여일 표시. 기한 경과 일자는 미입력 시간을 표시하되 잠금
- 자동 확정 조건 미달 일자는 확정 요청 상태로 표시

---

## 4. 데이터 모델

### 4.1 로컬 이벤트 로그 (에이전트 SQLite)

| 테이블 | 주요 컬럼 | 비고 |
|--------|-----------|------|
| `sample` | `ts`, `fg_process`, `fg_title_hash`, `idle_seconds`, `doc_path`, `launcher_ctx`, `power_state` | 원시 샘플. 창 제목은 해시만 저장 |
| `process_cpu` | `ts`, `pid`, `process`, `cpu_percent`, `doc_path` | 감시 대상 프로세스만 |
| `segment` | `uuid`, `device_id`, `start_ts`, `end_ts`, `context_type`, `fg_process`, `project_code`, `main_job`, `evidence_type`, `is_machine_time`, `uploaded_at` | 업로드 단위. `fg_process`는 미분류 세그먼트에서 미등록 툴 탐지용 |
| `pause` | `start_ts`, `end_ts`, `reason` | 사용자 수집 중지 구간 |

### 4.2 서버 저장 (D-1 결정)

- 저장 위치: ShotGrid CustomEntity `Auto Timelog Day`, **사용자·일자당 1레코드**
- 세그먼트는 해당 레코드의 JSON 텍스트 필드에 배열로 저장. 세그먼트 단위 ShotGrid 검색은 지원하지 않음
- 조회 단위는 항상 (사용자, 날짜). 근거 타임라인·미분류 입력 화면은 레코드 1건 조회 후 JSON 파싱
- 업로드는 세그먼트 배열을 서버가 병합(UUID 기준 upsert)한 뒤 레코드 1건 갱신
- 세그먼트 JSON 스키마

```json
{
  "uuid": "…",
  "start": "2026-09-08T09:12:00+09:00",
  "end": "2026-09-08T10:40:00+09:00",
  "context_type": "work | project_only | unclassified | non_dcc | idle | wait | paused",
  "device_id": "WS-COMP-07",
  "fg_process": "maya.exe",
  "unmapped_path_root": "D:/work",
  "project_id": 123,
  "main_job": "comp",
  "evidence": "doc_path | launcher | title | inherited",
  "machine_time": false
}
```

- 기각안: 세그먼트 1건 = 1레코드 (연간 수십만 레코드, 부하 과다), Redis 영구 저장 (cache 원칙 충돌, 영구성 미보장)

### 4.3 ShotGrid 스키마 (예정)

- 코드 배포 전 ShotGrid 필드 선행 생성 필수
- 표시 이름과 field code 병기

| 엔티티 | 표시 이름 | field code | 용도 |
|--------|-----------|------------|------|
| HumanUser | `Auto Timelog` | `sg_auto_timelog` | 자동 타임로그 대상 여부 (checkbox). **생성 완료 2026-09-14** |
| HumanUser | `Auto Timelog Notice At` | `sg_auto_timelog_notice_at` | 앱 안내 화면 확인 시각 (date_time). **생성 완료 2026-09-14** |
| TimeLog | `Auto Timelog Source` | `sg_auto_timelog_source` | TimeLog 생성 경로 (list: `manual`, `auto`, `manual_unclassified`). `auto`는 수정 잠금. **생성 완료 2026-09-14**. `TimeLogSchema.SG_FIELD_LIST`에 추가됨 |
| CustomNonProjectEntity21 `Auto Timelog Day` | `User` | `sg_user` | 대상 사용자 (entity). **엔티티·필드 생성 완료 2026-09-15** (엔티티는 Site Preferences에서 수동 활성화, 필드 8개는 API 생성). 코드 상수 `auto_timelog_service.AUTO_DAY_ENTITY`. 레코드 create/get/update/find_range 실측 확인 |
| `Auto Timelog Day` | `Date` | `sg_date` | 대상 일자 (date) |
| `Auto Timelog Day` | `Segments` | `sg_segments` | 세그먼트 JSON 배열 (text) |
| `Auto Timelog Day` | `Draft` | `sg_draft` | 초안 JSON (text) |
| `Auto Timelog Day` | `Confirm Status` | `sg_confirm_status` | `pending`, `draft`, `auto`, `auto_with_unclassified`, `completed`, `expired`, `manual` (list). `draft`는 collect 모드 초안 생성 완료(2026-09-23 리스트 값 추가). `auto_with_unclassified`는 자동분 확정 후 미분류 입력 대기, `completed`는 미분류 입력 완료, `expired`는 기한 경과 잠금 |
| `Auto Timelog Day` | `Unclassified Ratio` | `sg_unclassified_ratio` | 미분류 비율, 통계용 (float) |
| `Auto Timelog Day` | `Agent Coverage` | `sg_agent_coverage` | 에이전트 가동률 (float) |
| `Auto Timelog Day` | `Confirmed At` | `sg_confirmed_at` | 확정 시각 (date_time) |

- 사용 프로그램은 TimeLog 필드로 두지 않음. 한 태스크를 여러 DCC로 작업하면 행 분할이 필요해 기존 화면·통계·제출 API 구조가 깨짐. 프로그램별 시간은 `Auto Timelog Day` 세그먼트에서 집계

---

## 5. 매핑 규칙

### 5.1 경로 규칙 (P-3·P-4 반영, 조사 반영 2026-09-11)

- 경로 해석은 **`flova.model.file.WorkFile`** 사용 (`flova/model/file.py:22`, `WorkFile(work_file, project_code=None)`). 경로 + 파일명으로 프로젝트, 엔티티 타입, 시퀀스, 샷, 에셋타입, 에셋, 스텝, 태스크, 버전을 판별. 자동 타임로그가 별도 경로 규칙을 만들지 않음
- `WorkFile` 특성
  - 프로젝트: 경로 세그먼트가 `flova/template/*.yaml` 템플릿 이름과 일치하는 것으로 판별. `project_code` 인자 생략 가능
  - 스텝: 템플릿의 `ASSET_STEP_CODE_INDEX` / `SHOT_STEP_CODE_INDEX` 위치의 폴더명. 스텝 폴더명 = ShotGrid Step `short_name`
  - 태스크·버전: 파일명 정규식 (`ASSET_FILENAME_REG` / `SHOT_FILENAME_REG`)
  - 파싱 자체는 ShotGrid·Redis 불필요. `sg_*()` 계열 접근자만 Redis cache 사용, `sg_asset()`만 ShotGrid 직접 조회 → 자동 타임로그는 `sg_asset()` 호출 금지
  - 인식 실패 시 예외 없이 `is_valid()`가 False. 파일 존재 여부는 판별에 영향 없음
  - 드라이브 제한 없음 (`flova.path.WorkPath`는 `G:/M:/V:` 한정이라 사용하지 않음)
- 자동 타임로그가 사용하는 접근자: `is_valid()`, `project_code()`, `entity_type()`, `step_code()`. 시퀀스·샷·에셋·버전은 사용하지 않음
- Main Job 변환은 `(entity_type, step_code)` 조합으로 §5.2 표 조회. `layout`, `hair`처럼 Shot과 Asset에서 의미가 다른 스텝이 있으므로 스텝만으로 조회하지 않음
- TimeLog `sg_main_job`은 한글 표시명 문자열로 저장됨 (`flova/timelog/scripts/set_auto_timelog.py`, `partition_schema.py` 확인). 규칙 엔진은 `MainJob._JOBS[key]` 값을 출력
- 파서 결과가 있으면 `work` 컨텍스트. 스텝만 불명이면 `project_only`, Main Job은 HumanUser `Default Main Job`
- **파서가 인식하지 못하는 경로는 무조건 `미분류`**. 로컬 개인 폴더, 바탕화면, 임시 폴더, 파이프라인 규칙 밖 파일명 모두 해당. 보조 판별(파일명 추측, 부분 일치) 금지
- 인식 불가 = "DCC를 사용하지 않은 것"과 동일하게 취급. §2.6 규칙대로 직전 활성 DCC 컨텍스트 상속. 사용자 알림 없음
- 미인식 세그먼트는 `fg_process`와 `unmapped_path_root`(드라이브 + 최상위 2단계 폴더)를 기록. §5.4 관리자 목록에서 상습 로컬 작업 파악용

### 5.2 스텝 → Main Job (조사 반영 2026-09-11)

- 기존 코드에 스텝 → Main Job 매핑은 존재하지 않음. 이 표가 최초 정의이며 규칙 엔진 상수로 구현
- 조회 키: `(entity_type, step_code)`. 표에 없는 조합은 HumanUser `Default Main Job` (`sg_auto_project_timelog_main_job`)
- 스텝 코드 출처: ShotGrid Step `short_name` (Redis `sg:step`, `SGStepCache`) 및 `flova/path.py` `create_shot()`/`create_asset()` 분기
- 확정 필요 표시(`확인`)는 파이프라인 슈퍼바이저 확인 후 값 고정

Shot 스텝

| step_code | Main Job 키 | 한글명 | 상태 |
|-----------|-------------|--------|------|
| comp | `comp` | 합성 | 확정 |
| lighting | `lit` | 라이팅 | 확정 |
| fx | `fx` | FX | 확정 |
| animation | `anim` | Anim 애니메이션 | 확정. `match_anim`은 경로로 구분 불가 |
| layout | `layout` | Anim 레이아웃 | 확정 |
| matchmove | `mm` | 매치무브 | 확정. `mm_asset`, `mm_out`은 경로로 구분 불가 |
| roto | `roto` | 로토스코프 | 확정 |
| remove | `remove` | 리무브 | 확정 |
| matte | `matte` | 매트페인팅 | 확정 |
| motion | `motion` | 모션그래픽 | 확정 (`path.py`에서는 주석 처리, 실사용 여부 확인) |
| ai | `ai` | AI | 확정 |
| previz | `previz` | 프리비즈 | 확정 |
| concept | `concept` | 컨셉 | 확정 |
| cfx | `cfx` | CFX | 확정 (2026-09-11). Main Job `hair`·`cloth`·`crowd`를 `cfx`/CFX로 통합 |
| hair | `cfx` | CFX | 확정 (2026-09-11) |
| cloth | `cfx` | CFX | 확정 (2026-09-11) |
| crowd | `cfx` | CFX | 확정 (2026-09-11) |
| env | `env` | ENV | 확정 |
| retime | `comp` | 합성 | 확정 (2026-09-11) |
| env_layout | `env` | ENV | 확정 (2026-09-11) |
| lookdev | `lookdev` | 룩뎁 | 확정 (2026-09-11). Main Job `tex`/텍스쳐를 `lookdev`/룩뎁으로 변경 |
| asset | — | Default Main Job | 확인 (`asset_layout` 후보) |
| edit | — | Default Main Job | 확인 (`prod_manager` 또는 `etc` 후보) |
| plate | — | Default Main Job | 확인 (`tech` 또는 `etc` 후보) |
| zero | — | Default Main Job | 확인 (`comp` 또는 `tech` 후보) |
| unreal | — | Default Main Job | 확인 |

Asset 스텝

| step_code | Main Job 키 | 한글명 | 상태 |
|-----------|-------------|--------|------|
| model | `mod` | 모델링 | 확정 |
| rig | `rig` | 리깅 | 확정 |
| animation | `anim` | Anim 애니메이션 | 확정 |
| layout | `asset_layout` | 에셋 레이아웃 | 확정 (Shot `layout`과 다름) |
| hair | `asset_hair` | 에셋 헤어 | 확정 (Shot `hair`와 다름) |
| env | `env` | ENV | 확정 |
| fx | `fx` | FX | 확정 |
| motion | `motion` | 모션그래픽 | 확정 |
| previz | `previz` | 프리비즈 | 확정 |
| concept | `concept` | 컨셉 | 확정 |
| RND | `rnd` | RnD | 확정 |
| cloth | `cfx` | CFX | 확정 (2026-09-11) |
| crowd | `cfx` | CFX | 확정 (2026-09-11) |
| matte | `matte` | 매트페인팅 | 확인 |
| lookdev | `lookdev` | 룩뎁 | 확정 (2026-09-11) |
| art | `concept` | 컨셉 | 확인 |
| HDRI | — | Default Main Job | 확인 |

- 경로로 판별 불가능해 수동 입력 전용인 Main Job: `supervising`, `asset_fur`, `mm_asset`, `mm_out`, `match_anim`, `tech`, `prod_manager`, `manage`, `etc`

### 5.3 앱 분류

| 분류 | 프로세스 예시 | 처리 |
|------|---------------|------|
| 감시 대상 DCC | maya.exe, Nuke*.exe, houdini*.exe, katana*.exe, hiero*.exe, 3dsmax.exe, blender.exe, AfterFX.exe, Photoshop.exe, mocha*.exe, UnrealEditor.exe, Adobe Substance 3D Painter.exe, RealityScan.exe, Gaea.exe (`desktop/src/main/auto-timelog/config.ts` `WATCHED_PROCESS_PATTERNS`) | 문서 경로 기반 귀속. 훅 없는 툴은 상속 |
| 뷰어·모니터 | rv.exe, pureref.exe, mplay.exe, deadlinemonitor.exe | 파이프라인 도구로 취급, `non_dcc` (상속). 감시 대상에 넣지 않음 (2026-09-21 결정) |
| 파이프라인 도구 | RV, 사내 PySide2 툴 | `non_dcc` (상속) |
| 비DCC 업무 앱 | 브라우저, Mattermost, 오피스, 탐색기 | `미분류` (상속 정책 적용) |
| 제외 앱 | 게임, 미디어 플레이어 등 명시 목록 | 항상 `idle` |

- 분류 목록은 서버 배포, 에이전트가 주기적 갱신
- 태스크 분류는 감시 대상 DCC 검출에 전적으로 의존. 미등록 툴 시간은 미분류 → 상속되어 조용히 오귀속됨

### 5.4 미등록 툴 탐지 및 목록 보강

1. 미분류 세그먼트의 `fg_process`를 관리자 화면에서 사용자·프로세스별 집계, "미분류 상위 프로세스" 목록 제공. 파서 미인식 경로는 `unmapped_path_root`별 집계, "미인식 경로 상위 목록" 제공
2. 미분류 시간이 반복되는 사용자는 관리자 목록에 노출. 미등록 툴 시간은 비DCC로 분류되어 상속되므로 이 목록으로만 발견 가능
3. 관리자가 프로세스를 감시 대상 DCC 또는 파이프라인 도구로 등록 → 서버 배포 목록으로 전 에이전트 반영
4. 플러그인 없는 툴은 창 제목 파싱 규칙으로 귀속
- 도입 초기에는 이 목록을 주 1회 이상 확인, 안정화 후 새 툴 도입 시 확인

---

## 6. 구현 계획

### Phase 0. 타당성 검증 (PoC)

- TD 부서 2~3명 PC + 원격 사용자 1명(RDP 1대, Guacamole 1대) 에이전트 프로토타입 2주 실행
- 측정 항목: 미분류 비율, idle 오판율, 초안·실제 입력 일치율, CPU/메모리 부담
- 원격 환경 검증 항목

| 항목 | 확인 방법 | 합격 기준 |
|------|-----------|-----------|
| 포어그라운드 조회 | 원격 세션에서 Maya를 앞에 두고 사이드카 출력 확인 | `maya.exe` 출력 |
| 입력 감지 | 원격에서 키·마우스 입력 후 idle 시간 확인 | 0으로 리셋 |
| 접속 끊김 | 원격 클라이언트 종료 | RDP: 세션 잠금 → idle. VNC: 임계값 후 idle |
| 재접속 | 다시 접속 | 정상 샘플링 재개 |
| 트레이 메뉴 | 원격 세션에서 수집 중지 메뉴 동작 | 정상 |
| 다중 기기 병합 | 노트북 + 워크스테이션 동시 가동 하루 | 초안이 워크스테이션 DCC 기준으로 귀속 |
- PoC 대상자에게 §12 안내 문안 개별 고지 후 시작
- 산출물: PoC 결과를 [§8](#8-문제점-확인)에 기록

### Phase 1. 에이전트 기본

- 포어그라운드/idle 샘플링, 세그먼트 병합, SQLite 저장
- 수집 일시 중지 트레이 메뉴
- 세그먼트 업로드 API 및 서버 저장

### Phase 2. DCC 플러그인 및 매핑 엔진

- Maya, Nuke, Houdini 플러그인
- Flova 매핑 규칙 엔진, Redis cache 기반 경로 → Task 조회
- Houdini 부트스트랩(`menus/scripts/456.py`) 신설

### Phase 3. 초안 생성 및 웹 UI

- 일별 초안 생성 (주간/야근 분리, 정규화, 반올림)
- 타임로그 화면 자동분 잠금 표시, 근거 타임라인, 미분류 입력 UI
- 미분류 입력 요청 알림

### Phase 4. 파일럿 운영 (사용자 확정 모드)

- 부서 공지 후 1개 부서 대상 1개월 운영. 자동 확정 비활성, 사용자가 초안 확정
- 미분류 비율·초안 일치율 지표로 매핑 규칙 개선
- 관리자 화면 (규칙 관리, 통계)
- 자동 확정 전환 기준: 초안 일치율 70% 이상 2주 연속

### Phase 5. 자동 확정 전환 및 전사 확대

- 자동 확정·잠금 활성화, 미분류 입력 알림 운영
- 대상 사용자 단계적 확대 (`sg_auto_timelog` 활성화)
- 자동 프로젝트 타임로그 대상자는 제외 (D-3). 월별 배치 존재 여부 확인
- 전사 공지 + 앱 안내 화면으로 사용자 안내 문안(§12) 고지

### 배포·운영 체크리스트

1. ShotGrid 스키마: 완료 (2026-09-15). `Auto Timelog Day` 엔티티·필드, HumanUser·TimeLog 필드 모두 생성됨
2. 대상 지정: collect 모드에서는 불필요(전 사용자). confirm 전환 시 HumanUser `Auto Timelog` 체크. 기경훈 TD(4510)는 2026-09-15 `sg_auto_project_timelog=False`, `Auto Timelog=True`로 변경된 상태 (원복 가능)
3. Redis: TimeLog 월 파티션·사용자 cache rebuild (새 필드 반영)
4. 배포 서버 cron: 완료(코드). `start-project-timelog`가 `deploy/cron/infx-works-auto-timelog-judge`를 `/etc/cron.d/`에 설치 (근무일 12:00). 다음 배포 시 자동 반영
5. flova 갱신·앱 릴리즈: gmdirect devsync push 완료, 데스크톱 v1.02.088 `W:/inhouse/InfxWorks` 배치 완료. 배포 서버 `start-project-timelog` 실행은 keekee에서 SSH 키 인증이 안 되어(비밀번호 인증만 허용) devtask `20260915-095126-deploy-auto-timelog`(manual)로 gmdirect 콘솔에 절차를 넣어 둠. 실행 후 사용자·TimeLog 2026-09 cache rebuild와 `auto_timelog_judge --days 1` 확인
6. 폐쇄망 확인: Git Bash가 아닌 PowerShell·cmd에서 파이프 검증
8. 베타 1차 점검 (2026-09-21, 서버 로그 9/1~9/21 + ShotGrid 87건 분석): 수집은 9/16 18:47부터 정상(일 17~26명, 세그먼트 4,143건, 형식 오류 0). 판정 배치는 cron 실행 사용자(sysadmin)의 `/mnt/W/inhouse_logs` 권한 오류로 9/17·9/18 모두 실패 → cron을 root로 변경. 서버 `timelog` 로거 파일 기록이 9/11부터 소실(flova `get_logger` 핸들러 제거) → flova 수정. 상세는 §9 P-14·P-15
9. 배포 검증: `start-project-timelog`가 gunicorn 재시작 후 `/api/auto-timelog/status/` 응답 코드를 확인(200/302 아니면 실패 종료). 9/15 배포 시 gunicorn이 구버전으로 남아 전 사용자 앱이 404를 받은 사례 재발 방지
7. 판정 배치 실측: 개발 Redis(출퇴근·사용자·TimeLog cache 실데이터)로 2026-09-15 실행. 출퇴근 인정 구간 산출·퇴근 판정·저장소 upsert 정상, 세그먼트 없음 → `pending/low_coverage`. 새 필드 반영을 위해 사용자 cache와 TimeLog 월 파티션 rebuild 필요(개발 Redis는 rebuild 완료, 배포 서버는 배포 후 수행)

### 테스트 원칙

- 매핑 규칙 엔진, 세그먼트 병합, 초안 생성(정규화·반올림)은 TDD 작성
- 에이전트 샘플러는 OS API mock 단위 테스트
- 초안 생성 테스트에 주간/야근 경계, 휴가, 휴일근무, 세션 잠금 케이스 필수 포함

---

## 7. 구현 상태

| Phase | 항목 | 상태 | 담당 | 비고 |
|-------|------|------|------|------|
| 0 | 에이전트 프로토타입 | 완료 | | 2026-09-14. 사이드카·샘플러·세그먼트·SQLite·훅 서버·트레이 중지 |
| 0 | PoC 결과 정리 | 미착수 | | |
| 1 | 프로브 사이드카 `agent_probe.py` | 완료 | | ctypes 전용, psutil 미사용 |
| 1 | 샘플러/세그먼트/SQLite | 완료 | | `desktop/src/main/auto-timelog/` |
| 1 | 수집 중지 UI | 완료 | | 트레이 메뉴 토글 |
| 1 | 세그먼트 업로드 API | 완료 | | 2026-09-14. 서버 저장은 엔티티 슬롯 활성화 후 동작 |
| 2 | Maya 플러그인 (userSetup 연결) | 완료 | | 2026-09-14 |
| 2 | Nuke 플러그인 (menu.py 연결) | 완료 | | 2026-09-14 |
| 2 | Houdini 플러그인 (456.py 연결) | 완료 | | 2026-09-14 |
| 2 | 매핑 규칙 엔진 (Flova) | 완료 | | 2026-09-14 `mapping.py` |
| 2 | Houdini 부트스트랩 신설 | 완료 | | 2026-09-14 |
| 2 | DCC 훅 공통 모듈 `dcc_hook.py` | 완료 | | 전송·등록 함수 구현. DCC startup 연결은 미착수 (PoC 후) |
| 3 | 초안 생성 | 완료 | | 2026-09-14 `DraftBuilder` |
| 3 | 웹 UI 자동분 잠금/미분류 입력 | 완료 | | 2026-09-14 |
| 3 | 미분류 입력 요청 알림 | 완료 | | 2026-09-14 알림 센터 섹션·타임로그 배너 |
| 3 | ShotGrid 필드 생성 | 완료 | | 2026-09-15. `Auto Timelog Day` 엔티티(수동 활성화)·필드 8개, HumanUser 2개, TimeLog 1개 |
| 3 | 앱 안내 화면 및 확인 기록 | 완료 | | 2026-09-14 |
| 4 | 전사 수집 베타 (collect 모드) | 진행 중 | | 2026-09-15 시작. 데스크톱 v1.02.089 배포 후 관리자 화면 초안 일치율·미등록 툴·미인식 경로 관찰 |
| 4 | 관리자 화면 | 완료 | | 2026-09-14 `/manage/auto-timelog` |
| 4 | 관리자 화면 미수집 사용자 목록 | 완료 | | 2026-09-21. 기간 종료일 기준 활성·대상 사용자 중 레코드 없는 사용자 (`uncollected`) |
| 4 | 베타 1차 점검·수정 | 완료 | | 2026-09-21. §6 체크리스트 8·9, §9 P-14·P-15·P-16 |
| 5 | 자동 확정 전환 | 완료(코드) | | 판정 배치가 자동 확정 수행. cron 등록은 운영 항목 |
| 5 | 전사 확대 | 미착수 | | 운영 항목. HumanUser `Auto Timelog` 체크로 단계적 확대 |

- 상태 값: `미착수`, `진행 중`, `완료`, `보류`

---

## 8. 문제점 확인

- 확인된 문제는 ID 부여 후 기록. 해결 시 [§9](#9-해결된-문제점)로 이동

| ID | 구분 | 내용 | 영향 | 상태 |
|----|------|------|------|------|
| P-2 | 알고리즘 | 회의·리뷰 시간이 상속 정책으로 직전 프로젝트에 오귀속 가능 | 프로젝트별 시간 왜곡 | 수용. D-7에서 규칙 단순성 우선. 자동분은 수정 불가이므로 오귀속은 그대로 확정됨. `inherited` 표시로 근거만 남김. 향후 캘린더 연동 검토 가능 |
| P-10 | 통합 | 원격 환경(RDP, Guacamole RDP/VNC) 입력·포어그라운드 감지 미검증. 사내 원격 사용자 존재 확인됨 | 측정 누락 | 미해결. §2.5.1 다중 기기 병합 규칙 반영, Phase 0 PoC에 원격 사용자 포함하여 검증 |
| P-11 | 정책 | 자동분은 수정 불가이므로 오류가 그대로 확정됨. 기한 경과 미분류는 영구 미입력 | 통계·원가 신뢰도 | 대응 방침 확정. 자동분 오류는 규칙·감시 목록 보강 후 관리자 `rebuild`로만 정정, 미분류 미입력은 사용자 책임으로 통계에 노출. 파일럿에서 오류율·미입력률 측정 후 재검토 |
| P-13 | 알고리즘 | 감시 목록에 없는 툴(신규 DCC, 실행 파일명 변경, 사내 툴)의 시간이 비DCC로 분류 → 상속으로 조용히 오귀속. 미분류가 아니므로 사용자 입력 요청도 발생하지 않음 | 특정 사용자 시간 전체 왜곡 | 미해결. §5.4 탐지·보강 절차로 대응, 파일럿에서 검증 |

---

## 9. 해결된 문제점

| ID | 내용 | 해결 방법 | 해결일 | 관련 커밋/문서 |
|----|------|-----------|--------|----------------|
| P-18 | 앱 재시작·자동 업데이트(9/21 v1.02.095) 후 열려 있던 DCC 세션의 경로가 다음 저장·열기까지 비어 Maya·Nuke 세그먼트 일부가 경로 없음. Houdini(FX-1·FX-3)는 일주일간 경로 0건 | 훅 주기 재전송(§3.3 heartbeat). Houdini 0건은 별개 원인으로, T-3 실측에서는 hython+HOUDINI_PATH 경로로 훅이 정상 동작하므로 FX PC의 실행 경로(flova bat 미경유 `houdini.exe`, `houdini.env`의 `HOUDINI_PATH` 덮어쓰기) 확인 필요 | 2026-10-07 | `flova/timelog/auto_timelog/dcc_hook.py` |
| P-17 | collect 모드 초안 저장 시 `Confirm Status='draft'`가 ShotGrid 리스트 값에 없어 CRUD ERROR. 9/21~9/23 배치 3회 모두 초안 대상 전원 error (9/23: 7일 합계 error 151건) | ShotGrid `sg_confirm_status` 리스트에 `draft` 추가 (API `schema_field_update`, 2026-09-23). 코드 상수 추가 시 리스트 값 선행 생성 규칙 재확인 | 2026-09-23 | §4.3 |
| P-14 | 판정 배치 cron이 sysadmin으로 실행되어 flova 로거의 `/mnt/W/inhouse_logs`(root 전용) 열기에서 PermissionError로 시작 즉시 중단. 9/17·9/18 초안 0건 | `deploy/cron/infx-works-auto-timelog-judge` 실행 사용자를 gunicorn과 같은 root로 변경 | 2026-09-21 | PUB_GUIDE §9 |
| P-15 | flova `get_logger('timelog', no_file=True)`(`scripts/export_xls.py`)가 Django dictConfig의 `timelog` 로거 핸들러를 전부 제거해 9/11 17:13부터 서버 파일 로그에서 `timelog.*` 기록(세그먼트 수신 포함)이 사라짐 | flova `get_logger`가 flova 외부(호스트 앱)가 붙인 핸들러가 있는 로거는 손대지 않고 그대로 반환. flova가 붙인 핸들러만 교체 (`_flova_owned_handler` 표시) | 2026-09-21 | `flova/log/core.py` |
| P-16 | 9/15 배포에서 gunicorn이 구버전 코드로 남아 `/api/auto-timelog/*` 404. 앱은 60초마다 재시도만 하며 수집 미시작 (9/16 하루 3.2만 건 404 경고) | 서비스 재시작으로 해소. 재발 방지로 배포 스크립트에 API 응답 확인 단계 추가 | 2026-09-16 | `start-project-timelog` |
| P-8 | 활동 수집 고지·동의 절차 미정 | 단계별 공지 + 앱 최초 실행 안내 화면·확인 시각 기록 (D-4) | 2026-09-10 | 이 문서 §3.2, §6 |
| P-9 | 자동 프로젝트 타임로그 대상자 적용 여부 | 제외. 월별 일괄 기입 배치로 처리 (D-3) | 2026-09-10 | 이 문서 §1.2 |
| P-12 | 지연 업로드로 판정 시점에 세그먼트 누락 | 로컬 즉시 저장으로 유실 없음. 앱·윈도우 종료 시 즉시 업로드, 판정 시점을 다음 근무일 12:00로 이동, 이후 지연분 자동 재판정 | 2026-09-10 | 이 문서 §2.5, §2.8 |
| P-4 | 파이프라인 외부 경로 작업 시 매핑 불가 | 설계상 의도. 기존 Flova 경로 파서가 인식하지 못하면 무조건 `미분류`, DCC 미사용과 동일하게 직전 컨텍스트 상속. 보조 판별 금지. 관리자 목록에 `unmapped_path_root` 집계로 상습 로컬 작업 파악 | 2026-09-10 | 이 문서 §5.1 |
| P-3 | 시퀀스 단위 씬의 Task 귀속 곤란 | TimeLog가 Task를 기록하지 않음을 확인(제출 코드는 `sg_project`, `sg_main_job`, 시간만 기록). 귀속 목표를 Project + Main Job으로 확정하여 문제 소멸. ShotGrid 실측: Asset Task 정식 운영, Sequence Task는 Step 없는 임시 20건 | 2026-09-08 | 이 문서 §5.1 |
| P-1 | idle 오분류로 활성 시간 과소 측정 | D-5 분 단위 귀속으로 idle도 직전 컨텍스트 상속. 귀속 결과에 영향 없음. `대기`/`idle` 구분은 기록·통계용으로만 유지 | 2026-09-08 | 이 문서 §2.7 |
| P-7 | 실측 시간과 보고 시간 상이 | 정규화 폐기. 실측값은 초안 JSON `measured_hours` 등으로 보존 | 2026-09-08 | 이 문서 §2.7 |
| P-5 | 포어그라운드 창 조회 방법 미확정 | Python 사이드카 `agent_probe.py` (D-2) | 2026-09-08 | 이 문서 §3.2 |
| P-6 | 세그먼트 서버 저장 위치 미확정 | ShotGrid `Auto Timelog Day` 사용자·일자 1레코드 + JSON (D-1) | 2026-09-08 | 이 문서 §4.2 |

---

## 10. 결정 필요 사항

### 10.1 검토 진행 지침

- 개발 착수 전 미결정 사항(D-*)과 미해결 문제(P-*)를 **한 번에 하나씩** 검토하여 방법 확정
- 검토 순서: 착수 조건(Phase 1 이전)에 해당하는 항목 우선, 이후 Phase 순
- 각 항목 설명 시 아래 두 가지 버전을 함께 제시
  1. 기술 설명: 선택지, 장단점, 구현 영향을 기술 용어로 간결하게
  2. 쉬운 설명: 비유나 실제 상황 예시로 결정 포인트만 전달
- 각 항목은 권장안과 그 이유를 명시하여 결정 부담 최소화
- 결정 즉시 이 문서의 해당 항목 상태를 `결정`으로 갱신하고 관련 절(알고리즘, 데이터 모델 등)에 반영
- 결정으로 새 문제가 드러나면 P-* 항목으로 추가

### 10.2 결정 목록

| ID | 결정 사항 | 선택지 | 권장 | 상태 |
|----|-----------|--------|------|------|
| D-1 | 세그먼트·초안 서버 저장 위치 | §4.2 후보 3가지 | 사용자·일자 1레코드 + JSON 텍스트 필드 (ShotGrid) | **결정** (2026-09-08). 세그먼트 단위 검색 불필요, 사용자·일자 단위 조회만 필요 |
| D-2 | 포어그라운드 창 조회 구현 방식 | Python 사이드카 / 네이티브 addon / PowerShell 주기 호출 | Python 사이드카 | **결정** (2026-09-08). 기존 Flova Python·AMI 워커 관리 코드 재사용, 빌드 체인 무영향 |
| D-3 | 자동 프로젝트 타임로그 대상자 처리 | 제외 / 대체 / 병행 | 제외 | **결정** (2026-09-10): 제외. 해당 사용자는 매월 초 직전 달 일괄 기입 배치로 처리 (자동 타임로그 범위 밖) |
| D-4 | 수집 고지·동의 절차 | 사내 공지 / 앱 최초 실행 시 안내 화면 / 둘 다 | 둘 다 | **변경** (2026-09-15): 앱 안내 화면 폐지, 전사 공지만. `/api/auto-timelog/notice/`·`sg_auto_timelog_notice_at`는 미사용으로 잔존 |
| D-5 | 하루 합계 산정 방식 | 실측 / 비율 정규화 / 분 단위 귀속 | 비율 정규화 + 실측 보존 | **결정** (2026-09-08): **출퇴근 인정 구간 위 분 단위 귀속**, idle도 직전 컨텍스트 상속. 정규화 폐기, 실측값은 초안 JSON에 보존 |
| D-6 | idle 임계값 | 3분 / 5분 / 10분 | 5분 | **결정** (2026-09-08): **10분**. 오탐 최소화 우선. 서버 배포 정책 값으로 조정 가능 |
| D-7 | 미분류 시간 기본 정책 | 상속(상한 30분) / 상속(상한 없음) / 비율 배분 / 미배정 | 상속(상한 30분) | **결정** (2026-09-08): **직전 활성 DCC 컨텍스트 상속, 상한 없음** + fallback 사슬. 규칙 단순성과 완전 자동화 우선 |
| D-8 | 자동 확정 조건 | 미분류 비율 포함 / 가동률·퇴근 기록만 | 가동률 80% + 퇴근 기록 | **결정** (2026-09-08): 가동률 ≥ 80%, 퇴근 기록 존재. 미분류 비율 조건 삭제 |
| D-9 | 미분류 입력 기한 경과 후 입력 | 불가 / 관리자 승인 / 자유 / Default 자동 채움 | 관리자 승인 | **결정** (2026-09-10): **입력 불가**. 기한 경과 일자는 `expired` 잠금, 영구 미입력으로 통계 집계 |
| D-11 | 미분류 처리와 자동분 수정 가능 여부 | 미분류 상속 + 자동분 14일 수정 / 미분류 사용자 입력 + 자동분 잠금 / 비DCC까지 사용자 입력 | 미분류 사용자 입력 + 자동분 잠금 | **결정** (2026-09-10): 파서 미인식 시간만 `미분류`로 남겨 사용자가 14일 내 직접 입력. 자동분은 수정 불가. 비DCC·idle은 상속 유지 |
| D-10 | TimeLog에 주 사용 프로그램 요약 필드(`Auto Timelog Tools`, `sg_auto_timelog_tools`) 추가 | 추가 / 미추가 | 미추가 | 보류. 현재 범위 밖 |

---

## 11. 운영 지표

- 파일럿 이후 관리자 화면에서 확인 가능해야 하는 지표

| 지표 | 정의 | 목표 |
|------|------|------|
| 미분류 비율 | `unclassified_hours` / 인정 근무시간 | 20% 이하. 초과 사용자는 미등록 툴 점검 |
| 초안 일치율 | 수정 없이 확정한 항목 비율 | 70% 이상 |
| 수정 폭 | 초안 대비 확정값 변화량 평균 | 15% 이하 |
| 자동 확정률 | 자동 확정 조건 충족 일자 / 전체 근무일 | 90% 이상 |
| 미분류 발생률 | 미분류 시간이 있는 일자 / 전체 근무일 | 10% 이하 |
| 미분류 입력 완료율 | 기한 내 입력 완료 일자 / 미분류 발생 일자 | 95% 이상 |
| 에이전트 가동률 | 출근 인정 시간 중 에이전트 샘플 존재 비율 | 95% 이상 |

---

- **통계 사용자 범위 (2026-10-15 결정)**: 모든 집계·통계 화면은 수집 대상 사용자 전원을 행으로 포함한다. 세그먼트·DCC 사용 기록이 없는 사용자(DCC 미사용 직군, 앱 미실행)도 0으로 표시해 "DCC를 쓰지 않았다"는 사실이 드러나게 한다. 레코드가 있는 사용자만 나열하는 방식은 금지. 관리자 개요 `users`는 이 규칙을 따른다 (`build_admin_overview`)

## 12. 사용자 안내 문안

- 전사 공지, 앱 최초 실행 안내 화면, 타임로그 화면 도움말에 공통 사용

**자동 타임로그 안내**

1. 집계 기준: 현재 활성화된 DCC에 열려 있는 작업 파일
2. 여러 DCC 동시 실행 시: 실제 조작 중인 창 하나만 집계
3. 백그라운드 렌더·시뮬레이션: 근무시간 미포함
4. 회의·리뷰 등 DCC 밖 시간: 직전 작업에 이어서 배정
5. 하루 합계: 출퇴근 기록의 근무시간과 동일
6. 자동 입력된 타임로그: 수정 불가
7. 프로젝트 폴더 밖(개인 폴더 등) 파일 작업 시간: 자동 입력 안 됨. 다음 날 알림을 받고 14일 이내 직접 입력
8. 수집 항목: 프로그램 이름, 파일 경로
9. 미수집 항목: 화면, 키 입력, 메시지 내용

---

## 13. 변경 이력

| 일자 | 내용 |
|------|------|
| 2026-09-08 | 최초 작성. 설계 원칙, 알고리즘, 아키텍처, 구현 계획, 문제점·결정 사항 정리 |
| 2026-09-08 | 문체를 개조식·명사형 종결로 통일 |
| 2026-09-08 | 완전 자동화로 목표 변경. 자동 확정 조건, 이의 제기 기간 14일, 사용자 안내 문안 추가 |
| 2026-09-08 | §10.1 검토 진행 지침 추가 |
| 2026-09-15 | 전사 수집 베타 전환: `AUTO_TIMELOG_MODE`(collect/confirm), 초안 전용 판정, 초안 일치율 지표, 안내 창 폐지(D-4 변경), 업로드 완화, DCC 훅 스레드 전송. §1.2.1 신설 |
| 2026-09-15 | ShotGrid `Auto Timelog Day` 엔티티 활성화(사용자)·필드 8개 생성, 저장소 실측 확인. 판정 배치를 개발 Redis 실데이터로 실측(인정 구간·퇴근 판정 정상). cron을 배포 스크립트에 편입. flova·가이드 devsync push. §4.3·§6·§7 갱신 |
| 2026-09-14 | Phase 1~5 코드 구현: 매핑 엔진, 인정 구간 함수, 서버 서비스·API·판정 배치, submit 잠금·검증, 웹 UI(잠금·배너·알림·관리자), 데스크톱 업로더·안내 창, DCC startup 연결, ShotGrid HumanUser·TimeLog 필드 생성. §3.5·§4.3·§6·§7 갱신 |
| 2026-09-14 | Phase 0 에이전트 프로토타입 구현 (flova 사이드카·훅, desktop 에이전트). §3.3.1 구현 메모, §7 상태 갱신 |
| 2026-09-11 | §5.2 확정: cfx·hair·cloth·crowd→cfx. Main Job CFX 헤어/클로스/군중을 CFX로 통합, TimeLog 239건 변경, stats alias 보강 |
| 2026-09-11 | §5.2 확정: retime→comp, env_layout→env, lookdev→lookdev(신규 Main Job, 기존 텍스쳐 항목을 룩뎁으로 변경) |
| 2026-09-11 | 조사 반영: 경로 파서 `WorkFile` 확정(§5.1), 스텝→Main Job 표 작성(§5.2), DCC 훅 삽입 지점·전송 방식(§3.3). 런처 컨텍스트 개념 삭제 |
| 2026-09-11 | §2.1 흐름도·§2.3 판별 규칙을 D-5·D-11 결정과 일치하도록 정리. `non_dcc` 컨텍스트 명시 |
| 2026-09-10 | D-9 결정: 미분류 입력 기한 경과 시 입력 불가, `expired` 잠금. P-11 대응 방침 확정 |
| 2026-09-10 | D-4 결정: 공지 + 앱 안내 화면 확인 기록. P-8 해결 |
| 2026-09-10 | D-3 결정: 자동 프로젝트 타임로그 대상자 제외, 월별 배치로 처리. P-9 해결 |
| 2026-09-10 | P-12 해결: 종료 시 즉시 업로드, 판정 시점 다음 근무일 12:00, 지연분 재판정 |
| 2026-09-10 | P-10 대응: §2.5.1 다중 기기 병합 규칙, `device_id` 추가, PoC 원격 환경 검증 항목 추가 |
| 2026-09-10 | D-11 결정: 파서 미인식 시간은 미분류로 남겨 사용자 입력(14일), 자동분 잠금. 이의 제기 폐기, §2.6·§2.8·§2.9 재작성 |
| 2026-09-10 | P-4 해결: 파서 미인식 경로는 무조건 미분류, 보조 판별 금지. 경로 해석은 기존 Flova 파서 사용으로 §5.1 갱신 |
| 2026-09-08 | P-3 해결: TimeLog는 Task 미기록. 귀속 단위를 Project + Main Job으로 확정, §5.1 단순화 |
| 2026-09-08 | D-5 결정: 분 단위 귀속, 정규화 폐기, 실측값 보존. §5.4 미등록 툴 탐지 추가, P-13 신규, P-1·P-7 해결, D-10 보류 |
| 2026-09-08 | D-7·D-8 결정: 미분류 시간은 직전 활성 DCC 컨텍스트 상속(상한 없음), 자동 확정 조건에서 미분류 비율 삭제 |
| 2026-09-08 | D-6 결정: idle 임계값 10분 |
| 2026-09-08 | D-2 결정: 포어그라운드 조회는 Python 사이드카. P-5 해결 |
| 2026-09-08 | D-1 결정: 서버 저장은 ShotGrid `Auto Timelog Day` 사용자·일자 1레코드. P-6 해결 |
| 2026-09-21 | 베타 1차 점검: P-14(cron root), P-15(flova 로거 핸들러 보존), P-16(배포 API 검증) 해결. 감시 대상 DCC 4종 추가, 훅 없는 DCC 상속 규칙(§2.3 예외), 테스트·R&D 프로젝트 일치율 제외(`is_test_project`), 관리자 미수집 사용자 목록, 머신 세그먼트 60초 미만 폐기. §3.3·§5.3·§6·§7·§9 갱신 |
| 2026-09-23 | P-17 해결: `Confirm Status` 리스트에 `draft` 추가. 베타 2차 점검(9/21~23 로그): 수집 일 16~21명·수신 330~570건/일 정상, cron root 전환 정상 |
| 2026-10-07 | 베타 2차 점검(9/24~10/6): 초안 237건 중 항목 있는 41건 일치율 평균 0.85, 빈 초안 167건(미수집 11명·비DCC 직군·훅 미동작). P-18 훅 주기 재전송 추가. 운영 로그 폴링 INFO 17곳 DEBUG 전환 |
| 2026-10-15 | 베타 3차 점검(10/7~10/15): heartbeat 효과로 Nuke 경로 누락 거의 0, Houdini 10/13부터 인식. 항목 있는 초안 51%. §11 통계 사용자 범위 결정, 관리자 개요 users에 대상 전원 포함. nav 진단 로그·미인증 frontend log 거부를 DEBUG로 |
