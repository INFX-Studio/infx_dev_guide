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
| `sg_status` | 상태 | list | - | (null), `요청`, `승인`, `반려` |
| `sg_time` | 시간 | date_time | - | 출퇴근 시간 또는 정정 희망 시간 |
| `sg_related_attendance` | 관련출퇴근 | entity(Self) | - | 연결 대상 레코드 |
| `sg_reason` | 사유 | text | - | 정정 사유 또는 반려 사유 |

### 2.4 sg_time 필드 용도

| 레코드 종류 | sg_time 의미 |
|---------------|----------------|
| 확정 (sg_status=null) | 실제 출퇴근 시간 |
| 요청 (sg_status='요청') | 정정 희망 시간 |
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
| (null) | - | 확정된 출퇴근 기록 |
| `요청` | 요청 | 정정 요청 |
| `승인` | 승인 | 정정 승인 |
| `반려` | 반려 | 정정 반려 |

### 2.8 레코드 종류 (sg_type + sg_status 조합)

| sg_status | sg_type | 의미 | sg_related_attendance |
|-----------|---------|------|----------------------|
| null | 출근 | 확정된 출근 | - 또는 이전 출근 레코드 |
| null | 퇴근 | 확정된 퇴근 | - 또는 이전 퇴근 레코드 |
| 요청 | 출근 | 출근 정정 요청 | 원본 출근 레코드 |
| 요청 | 퇴근 | 퇴근 정정 요청 | 원본 퇴근 레코드 |
| 승인 | 출근 | 출근 정정 승인 | 승인한 요청 레코드 |
| 승인 | 퇴근 | 퇴근 정정 승인 | 승인한 요청 레코드 |
| 반려 | 출근 | 출근 정정 반려 | 반려한 요청 레코드 |
| 반려 | 퇴근 | 퇴근 정정 반려 | 반려한 요청 레코드 |

### 2.9 Code 규칙

```
{이름}_{YYYYMMDD}_{type}_{status}_{HHMMSS}
```

예시:
- `홍길동_20260305_출근_140530` — 확정된 출근
- `홍길동_20260305_출근_요청_163000` — 출근 정정 요청
- `홍길동_20260305_출근_승인_170000` — 출근 정정 승인
- `홍길동_20260305_출근_반려_170500` — 출근 정정 반려

---

## 3. 이벤트 흐름

### 3.1 기본 출퇴근

```
[출근 버튼 클릭]
└─ 레코드 생성: sg_type=출근, sg_status=null
   - sg_time = 출근 시간

[퇴근 버튼 클릭]
└─ 레코드 생성: sg_type=퇴근, sg_status=null
   - sg_time = 퇴근 시간
```

### 3.2 정정 요청 → 승인

```
A0: 출근 (09:30)
 │
 └─ R1: 출근 정정 요청 (08:00으로 변경 요청)
     │  - sg_status = 요청
     │  - sg_time = 08:00
     │  - sg_related_attendance = A0
     │  - sg_reason = "지하철 지연으로 늦게 기록됨"
     │
     └─ D1: 출근 정정 승인
         │  - sg_status = 승인
         │  - sg_related_attendance = R1
         │  - sg_user = 팀장
         │
         └─ A1: 출근 (08:00) — 새 확정 레코드
              - sg_status = null
              - sg_time = 08:00
              - sg_related_attendance = A0
```

### 3.3 정정 요청 → 반려 → 재요청 → 승인

```
A0: 출근 (09:30)
 │
 ├─ R1: 출근 정정 요청 (08:00)
 │   │  - sg_time = 08:00
 │   │  - sg_related_attendance = A0
 │   │
 │   └─ D1: 출근 정정 반려
 │        - sg_status = 반려
 │        - sg_related_attendance = R1
 │        - sg_reason = "증빙 부족"
 │
 └─ R2: 출근 정정 요청 (08:30) — 재요청
     │  - sg_time = 08:30
     │  - sg_related_attendance = A0
     │  - sg_reason = "증빙 첨부함"
     │
     └─ D2: 출근 정정 승인
         │  - sg_related_attendance = R2
         │
         └─ A1: 출근 (08:30) — 새 확정 레코드
              - sg_time = 08:30
              - sg_related_attendance = A0
```

### 3.4 현재 상태 판단 로직

특정 출퇴근(A0)의 현재 상태를 확인하려면:

1. A0를 `sg_related_attendance`로 참조하는 레코드들 조회
2. 가장 최근(`created_at` 기준) 레코드의 `sg_status` 확인

```python
def get_attendance_status(original_attendance):
    """출퇴근 레코드의 현재 상태를 반환한다."""
    # 관련 레코드 조회 (최신순)
    related = sg.find(
        'CustomNonProjectEntity10',
        [['sg_related_attendance', 'is', original_attendance]],
        ['sg_status', 'sg_type', 'created_at'],
        order=[{'field_name': 'created_at', 'direction': 'desc'}]
    )
    
    if not related:
        return '확정'  # 정정 요청 없음
    
    latest = related[0]
    status = latest.get('sg_status')
    
    if status is None:
        return '확정'  # 새 확정 레코드가 생성됨 (승인 완료)
    elif status == '요청':
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
- 요청 레코드 생성 (`sg_status = '요청'`)

### 4.3 승인/반려

- 승인담당자(팀장/실장)만 처리 가능
- 승인 시: 승인 레코드 생성 → 새 확정 출퇴근 레코드 생성
- 반려 시: 반려 레코드 생성 (사유 필수)
- 반려 후 재요청 가능 (횟수 제한 없음)

### 4.4 불변성 규칙

- **모든 레코드는 생성 후 수정하지 않음**
- 상태 변경은 새 레코드 생성으로 표현
- 감사 추적이 데이터 구조 자체로 보장됨

---

## 5. 무결성 규칙

### 5.1 필수 검증

1. **하루 1개의 확정 출근**: 같은 `sg_user + sg_date`에 `sg_type=출근, sg_status=null`은 최대 1개
2. **하루 1개의 확정 퇴근**: 같은 `sg_user + sg_date`에 `sg_type=퇴근, sg_status=null`은 최대 1개
3. **요청 대상 존재**: `sg_status=요청`인 레코드는 `sg_related_attendance`가 반드시 존재
4. **승인/반려 대상 존재**: `sg_status=승인/반려`인 레코드는 `sg_related_attendance`가 요청 레코드를 가리켜야 함

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
            ['sg_status', 'is', None],
        ],
        ['id']
    )
    return len(records) <= 1
```
