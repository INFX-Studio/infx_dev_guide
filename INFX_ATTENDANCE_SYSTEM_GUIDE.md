# 출퇴근 SaaS 시스템 설계 문서

## 1. 개요

### 1.1 목적

- 프로젝트 타임로그 웹서비스에 출퇴근 기능 추가
- 출근/퇴근 시간 기록 및 근무시간 자동 계산
- 정정 요청 및 승인 워크플로우 지원

### 1.2 주요 기능

- 출근/퇴근 버튼으로 시간 기록
- 일별/월별 근무시간 계산 (야근 포함)
- 정정 요청 → 팀장 승인/반려 워크플로우
- 모든 이력 보존 (완전 불변 이벤트 로그)
- 데이터 무결성 검증 시스템

---

## 2. DB 설계

### 2.1 설계 방식

- **완전 불변 이벤트 로그 방식** 채택
- 엔티티 1개로 모든 기록 관리
- **레코드는 생성만 하고, 절대 수정하지 않음**
- 출퇴근/정정요청/승인/반려를 각각 별도 레코드로 기록
- 2개 필드 조합(`sg_type` + `sg_status`)으로 레코드 종류 구분

### 2.2 엔티티 정보

| 항목 | 값 |
|------|-----|
| Entity Type | CustomNonProjectEntity10 |
| Display Name | 출퇴근 기록 |

### 2.3 필드 정의

| 필드명 | Display Name | 타입 | 필수 | 설명 |
|--------|--------------|------|------|------|
| `code` | Code | text | O | `{이름}_{YYYYMMDD}_{type}_{status}_{HHMMSS}` |
| `sg_user` | 사용자 | entity(HumanUser) | O | 레코드 종류별 의미 다름 (아래 표 참고) |
| `sg_date` | 근무일 | date | O | 출퇴근 날짜 |
| `sg_type` | 구분 | list | O | `출근`, `퇴근` |
| `sg_status` | 상태 | list | - | `정상`, `정정요청`, `승인`, `반려` |
| `sg_time` | 시간 | date_time | - | 출퇴근 시간 또는 정정 희망 시간 |
| `sg_parent` | Parent | entity(Self) | - | 연결 대상 레코드 |
| `sg_reason` | 사유 | text | - | 정정 사유 또는 반려 사유 |

### 2.4 sg_time 필드 용도

| 레코드 종류 | sg_time 의미 |
|---------------|----------------|
| 정상 (sg_status='정상') | 실제 출퇴근 시간 |
| 정정요청 (sg_status='정정요청') | 정정 희망 시간 |
| 승인/반려 | null (의미 없음) |

### 2.5 sg_user 필드 용도

| 레코드 종류 | sg_user 의미 |
|---------------|----------------|
| 확정 출퇴근 | 대상 직원 (본인) |
| 정정 요청 | 대상 직원 (본인) |
| 승인/반려 | 처리자 (팀장) |
### 2.6 시스템 필드 활용

| 필드명 | 용도 |
|--------|------|
| `created_at` | 레코드 생성 시점 |
| `created_by` | 레코드 생성자 |

### 2.7 List 필드 값

#### sg_type

| Code | Display Name | 설명 |
|------|--------------|------|
| `출근` | 출근 | 출근 관련 레코드 |
| `퇴근` | 퇴근 | 퇴근 관련 레코드 |

#### sg_status

| Code | Display Name | 설명 |
|------|--------------|------|
| `정상` | 정상 | 확정된 출퇴근 기록 |
| `정정요청` | 정정요청 | 정정 요청 |
| `승인` | 승인 | 정정 승인 |
| `반려` | 반려 | 정정 반려 |

### 2.8 레코드 종류 (sg_type + sg_status 조합)

| sg_status | sg_type | 의미 | sg_parent |
|-----------|---------|------|----------------------|
| 정상 | 출근 | 확정된 출근 | - 또는 이전 출근 레코드 |
| 정상 | 퇴근 | 확정된 퇴근 | - 또는 이전 퇴근 레코드 |
| 정정요청 | 출근 | 출근 정정 요청 | 원본 출근 레코드 |
| 정정요청 | 퇴근 | 퇴근 정정 요청 | 원본 퇴근 레코드 |
| 승인 | 출근 | 출근 정정 승인 | 승인한 요청 레코드 |
| 승인 | 퇴근 | 퇴근 정정 승인 | 승인한 요청 레코드 |
| 반려 | 출근 | 출근 정정 반려 | 반려한 요청 레코드 |
| 반려 | 퇴근 | 퇴근 정정 반려 | 반려한 요청 레코드 |

### 2.9 Code 규칙

```
{이름}_{YYYYMMDD}_{type}_{status}_{HHMMSS}
```

예시:
- `홍길동_20260305_출근_정상_140530` — 확정된 출근
- `홍길동_20260305_출근_정정요청_163000` — 출근 정정 요청
- `홍길동_20260305_출근_승인_170000` — 출근 정정 승인
- `홍길동_20260305_출근_반려_170500` — 출근 정정 반려

---

## 3. 이벤트 흐름

### 3.1 기본 출퇴근

```
[출근 버튼 클릭]
└─ 레코드 생성: sg_type=출근, sg_status=정상
   - sg_time = 출근 시간

[퇴근 버튼 클릭]
└─ 레코드 생성: sg_type=퇴근, sg_status=정상
   - sg_time = 퇴근 시간
```

### 3.2 정정 요청 → 승인

```
A0: 출근 (09:30)
 │
 └─ R1: 출근 정정 요청 (08:00으로 변경 요청)
     │  - sg_status = 정정요청
     │  - sg_time = 08:00
     │  - sg_parent = A0
     │  - sg_reason = "지하철 지연으로 늦게 기록됨"
     │
     └─ D1: 출근 정정 승인
         │  - sg_status = 승인
         │  - sg_parent = R1
         │  - sg_user = 팀장
         │
         └─ A1: 출근 (08:00) — 새 확정 레코드
              - sg_status = 정상
              - sg_time = 08:00
              - sg_parent = A0
```

### 3.3 정정 요청 → 반려 → 재요청 → 승인

```
A0: 출근 (09:30)
 │
 ├─ R1: 출근 정정 요청 (08:00)
 │   │  - sg_time = 08:00
 │   │  - sg_parent = A0
 │   │
 │   └─ D1: 출근 정정 반려
 │        - sg_status = 반려
 │        - sg_parent = R1
 │        - sg_reason = "증빙 부족"
 │
 └─ R2: 출근 정정 요청 (08:30) — 재요청
     │  - sg_time = 08:30
     │  - sg_parent = A0
     │  - sg_reason = "증빙 첨부함"
     │
     └─ D2: 출근 정정 승인
         │  - sg_parent = R2
         │
         └─ A1: 출근 (08:30) — 새 확정 레코드
              - sg_time = 08:30
              - sg_parent = A0
```

### 3.4 현재 상태 판단 로직

특정 출퇴근(A0)의 현재 상태를 확인하려면:

1. A0를 `sg_parent`로 참조하는 레코드들 조회
2. 가장 최근(`created_at` 기준) 레코드의 `sg_status` 확인

```python
def get_attendance_status(original_attendance):
    """출퇴근 레코드의 현재 상태를 반환한다."""
    # 관련 레코드 조회 (최신순)
    related = sg.find(
        'CustomNonProjectEntity10',
        [['sg_parent', 'is', original_attendance]],
        ['sg_status', 'sg_type', 'created_at'],
        order=[{'field_name': 'created_at', 'direction': 'desc'}]
    )
    
    if not related:
        return '확정'  # 정정 요청 없음
    
    latest = related[0]
    status = latest.get('sg_status')
    
    if status == '정상':
        return '확정'  # 새 확정 레코드가 생성됨 (승인 완료)
    elif status == '정정요청':
        return '대기'  # 승인 대기 중
    elif status == '승인':
        return '승인됨'  # 승인 처리됨 (새 확정 레코드 생성 대기)
    elif status == '반려':
        return '반려됨'  # 반려됨 (재요청 가능)
    
    return '확정'
```

---

## 4. 비즈니스 규칙

### 4.1 퇴근 처리

#### 정식 퇴근 시간

- 정식 퇴근 시간: **오후 7시 (19:00)**

#### 조기 퇴근 확인 다이얼로그

퇴근 버튼 클릭 시 현재 시각에 따라 동작이 달라진다.

| 현재 시각 | 동작 |
|-----------|------|
| 19:00 미만 | 확인 다이얼로그 표시 후 사용자 선택에 따라 처리 |
| 19:00 이상 | 확인 없이 즉시 퇴근 처리 |

**확인 다이얼로그 메시지 예시**

```
아직 퇴근 시간(오후 7시)이 되지 않았습니다.
지금 퇴근 처리하시겠습니까?

[확인]  [취소]
```

- **확인** 선택 시: 퇴근 처리 진행
- **취소** 선택 시: 퇴근 처리 없이 닫기

### 4.2 정정 요청

- 출퇴근 시간 클릭 시 정정 요청 다이얼로그 표시
- 희망 시간과 정정 사유 입력 필수
- 요청 레코드 생성 (`sg_status = '정정요청'`)

### 4.3 승인/반려

- 승인담당자만 처리 가능
- 승인 시: 승인 레코드 생성 → 새 확정 출퇴근 레코드 생성
- 반려 시: 반려 레코드 생성 (사유 필수)
- 반려 후 재요청 가능 (횟수 제한 없음)

### 4.4 승인 체계

#### 조직 구조와 ShotGrid 필드 매핑

| 직급 | ShotGrid 필드 | 설명 |
|------|--------------|------|
| 팀원 | `sg_part_supervisor` = 실장 | 실장이 정정 요청을 승인 |
| 팀장 | `sg_part_supervisor` = 실장 | 실장이 정정 요청을 승인 |
| 실장 | `sg_part_supervisor` = 비어있음 | 상위 승인자 없음 (직접 수정 가능) |

#### 승인 권한 판별

승인 대기 뱃지 표시 여부는 **사전 판별 없이** `Attendance.get_pending_corrections()`를 호출하여 결과가 있으면 표시한다. department와 무관하게, 요청자의 `sg_part_supervisor`에 등록된 사람만 승인할 수 있다.

#### 정정 요청 조회 범위 (누구의 정정을 볼 수 있는가)

`Attendance.get_pending_corrections(approver)` 메서드의 필터링 기준:

1. `sg_status = '정정요청'`인 레코드 전체 조회
2. **요청자의 `sg_part_supervisor`에 승인자가 포함**되고 **본인 건이 아닌 것**만 필터링
3. 이미 승인/반려된 요청은 제외

```python
# 요청자의 sg_part_supervisor에 승인자가 포함된 것만 필터링
supervisors = item.get('sg_user.HumanUser.sg_part_supervisor') or []
if isinstance(supervisors, dict):
    supervisors = [supervisors]
for sup in supervisors:
    if sup and sup.get('id') == approver_id:
        filtered.append(item)
        break
```

#### 승인 흐름 요약

| 요청자 | 승인자 | 판별 기준 |
|--------|--------|----------|
| 팀원 | 요청자의 `sg_part_supervisor` | 요청자의 실장이 승인 |
| 팀장 | 요청자의 `sg_part_supervisor` | 요청자의 실장이 승인 |
| 실장 | 본인 (직접 수정 가능) | `sg_part_supervisor` 비어있음 → 정정 요청 없이 직접 시간 변경 |

### 4.5 불변성 규칙

- **모든 레코드는 생성 후 수정하지 않음**
- 상태 변경은 새 레코드 생성으로 표현
- 감사 추적이 데이터 구조 자체로 보장됨

---

## 5. 무결성 규칙

### 5.1 필수 검증

1. **하루 1개의 확정 출근**: 같은 `sg_user + sg_date`에 `sg_type=출근, sg_status=정상`은 최대 1개
2. **하루 1개의 확정 퇴근**: 같은 `sg_user + sg_date`에 `sg_type=퇴근, sg_status=정상`은 최대 1개
3. **요청 대상 존재**: `sg_status=정정요청`인 레코드는 `sg_parent`가 반드시 존재
4. **승인/반려 대상 존재**: `sg_status=승인/반려`인 레코드는 `sg_parent`가 요청 레코드를 가리켜야 함

### 5.2 검증 쿼리 예시

```python
def validate_single_valid_attendance(sg_user, sg_date, sg_type):
    """하루에 확정 출퇴근이 1개인지 검증한다."""
    records = sg.find(
        'CustomNonProjectEntity10',
        [
            ['sg_user', 'is', sg_user],
            ['sg_date', 'is', sg_date],
            ['sg_type', 'is', sg_type],
            ['sg_status', 'is', '정상'],
        ],
        ['id']
    )
    return len(records) <= 1
```

---

## 6. 설계 결정 (Design Decisions)

### 6.1 sg_date + sg_time 중복 허용

**문제**: `sg_time`(DateTime)에 이미 날짜가 포함되어 있어 `sg_date`(Date)와 중복됨.

**결정**: 의도적 비정규화로 유지

**이유**:
- 야근 시 날짜 불일치가 정상 케이스 (3/6 출근 → 3/7 01:30 퇴근)
- `sg_date`는 논리적 근무일 (비즈니스 키), `sg_time`은 물리적 시간
- 쿼리 편의성 (`sg_date`로 필터링)
- 무결성은 코드에서 보장

### 6.2 완전 불변 이벤트 로그 방식

**문제**: 출퇴근 기록을 어떻게 관리할 것인가?

**결정**: 레코드는 생성만 하고, 절대 수정하지 않음

**이유**:
- 감사 추적이 데이터 구조 자체로 보장됨
- 모든 변경 이력이 자동으로 보존됨
- 데이터 위변조 불가능
- 디버깅 및 문제 추적 용이

### 6.3 sg_user 필드 통합 (sg_approved_by 제거)

**문제**: 대상 직원과 승인자를 별도 필드로 관리할 것인가?

**결정**: `sg_user` 하나로 통합, `sg_approved_by` 제거

**이유**:
- 하나의 레코드는 하나의 성질만 기록함
- 확정/요청 레코드: `sg_user` = 대상 직원
- 승인/반려 레코드: `sg_user` = 처리자
- 대상 직원은 `sg_parent` 체인을 따라가면 확인 가능
- 필드 수 최소화로 스키마 단순화

### 6.4 sg_parent 필드명 선택

**문제**: 연결 대상 레코드를 뭐라고 부를 것인가?

**검토한 후보**:
- `sg_related_attendance` — 너무 김
- `sg_upstream` / `sg_downstream` — 방향 햷갈림 (데이터 흐름 vs 이력 추적 방향)
- `sg_parent` — 계층 구조 명확, 직관적

**결정**: `sg_parent` 사용

**이유**:
- "이 레코드의 Parent는 뭐야?"가 직관적
- upstream/downstream보다 비즈니스 용어로 자연스러움
- 코드가 깔끔해짐 (`request_log['sg_parent']`)
