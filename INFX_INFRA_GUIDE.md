# inFX 개발 환경 및 인프라 가이드

본 문서는 inFX의 Flova 파이프라인 시스템 개발을 위한 환경 설정 및 인프라 구성을 정리한 문서입니다.

---

## 1. Flova 파이프라인 시스템

### 1.1 Flova란?

**Flova**는 'Flow'와 'Nova'의 합성어로, VFX 제작 파이프라인의 혁신을 상징하는 이름입니다.

- **Flow**: 유기적이고 자연스러운 작업 흐름을 의미합니다. 아티스트와 TD, 개발자가 도구와 데이터 사이에서 막힘없이 작업할 수 있는 환경을 추구합니다.

- **Nova**: 초신성(Supernova)처럼 기존 방식을 뛰어넘는 혁신적 변화를 뜻합니다. 낡은 파이프라인 구조를 과감히 개선하고, 새로운 가능성을 여는 시스템을 지향합니다.

Flova는 단순한 도구 모음이 아니라, VFX 제작 전 과정이 하나의 유기체처럼 연결되고 진화하는 통합 파이프라인 플랫폼입니다. 각 요소가 독립적으로 동작하면서도 전체가 조화롭게 흐르도록 설계되어, 아티스트는 창작에 집중하고 기술팀은 안정적인 인프라를 운영할 수 있습니다.

---

## 2. 개발 환경

### 2.1 운영체제 및 기본 환경

- **OS**: Windows 10 64bit 한국어 버전
- **Python**: 3.10.11 (당분간 버전업 계획 없음)
- **GUI 프레임워크**: PySide2 (PyQt 미사용)
- **UI 프레임워크**: Flova UI (PySide2 기반 커스텀 프레임워크)
- **버전 관리**: GitLab

### 2.2 경로 구조

inFX는 개발자와 비개발자의 작업 환경을 명확히 분리하여 운영합니다.

#### 개발자 경로
```
C:\dev\infx\
  ├─ flova\                    # Flova 파이프라인 프로젝트
  │   ├─ flova\               # 소스 코드
  │   │   ├─ maya\            # Maya 통합
  │   │   ├─ nuke\            # Nuke 통합
  │   │   ├─ katana\          # Katana 통합
  │   │   └─ exec\            # DCC 실행 배치파일
  │   └─ requirements.txt     # Python 패키지 목록
  └─ project_timelog_web\     # 타임로그 웹서비스
```

#### 비개발자(아티스트) 경로
```
W:\inhouse\
  ├─ flova\                   # Flova 배포본
  ├─ flova_libs\              # 공용 Python 라이브러리
  ├─ pypi-wheels\             # 로컬 전용 wheel 파일
  ├─ pymel\                   # PyMEL 라이브러리
  └─ ocio\                    # OCIO 설정 파일
      └─ INFX_config.ocio
```

**경로 매핑**
- 개발자의 `C:\dev\infx` = 비개발자의 `W:\inhouse`
- 개발자는 GitLab에서 직접 pull하여 개발
- 비개발자는 배포된 코드 사용

---

## 3. 네트워크 인프라

### 3.1 프록시 서버

inFX는 보안을 위해 폐쇄망을 운영하며, 프록시 서버를 통해 제한적인 외부 접근을 허용합니다.

#### 개발자용 프록시
- **주소**: `192.168.10.40:23128`
- **대상**: TD 및 개발자
- **정책**: IT팀이 승인한 도메인만 접근 가능

**접근 가능 도메인**
- `pypi.org` (Python 패키지)
- `shotgrid.autodesk.com` (Shotgrid API)
- `docs.python.org` (Python 문서)
- `github.com` ❌ (접근 불가)

#### 비개발자용 프록시
- **주소**: `192.168.10.39:23128`
- **대상**: 일반 아티스트
- **정책**: 완전 폐쇄망 (외부 인터넷 접근 불가)

### 3.2 내부 서버

#### 타임로그 서버
- **IP**: `192.168.10.27`
- **접근**: 사내망 전용
- **용도**: 업무 시간 추적/기록 웹서비스
- **기술 스택**: Django, jQuery, JavaScript, Bootstrap
- **OS**: Rocky Linux
- **WSGI**: Gunicorn (24시간 가동)
- **DB 구조**:
  - SQLite3: Django 사용자 인증 정보만
  - Shotgrid: 메인 DB (모든 타임로그 데이터)
- **Flova 연동**: Shotgrid를 통한 데이터 공유

---

## 4. DCC 통합

### 4.1 환경변수 정책

**사용하지 않는 환경변수**
- `PYTHONPATH`: DCC 배치파일과 충돌 문제로 사용 안 함
- `OCIO`: 팀별 유연성 요구로 중앙 설정 안 함
  - OCIO 파일 위치: `W:\inhouse\ocio\INFX_config.ocio`

**DCC별 환경변수**
- IT팀이 Active Directory(AD)로 관리
- 시스템 재부팅 시 적용 (즉각 반영 안 됨)
- 변경 시 IT팀 요청 필요

### 4.2 DCC별 설정

#### Maya

**환경 파일**: `maya.env`

개발자 자리:
```
PYTHONPATH=C:/dev/infx/flova;C:/dev/infx/flova/flova/maya/startup;W:/inhouse/flova_libs;W:/inhouse/pymel;
```

비개발자 자리:
```
PYTHONPATH=W:/inhouse/flova;W:/inhouse/flova/flova/maya/startup;W:/inhouse/flova_libs;W:/inhouse/pymel;
```

#### Katana

**배치파일 실행**: 이미 배치파일 방식 적용 중

개발자 자리 (`katana7.bat`):
```batch
@echo off
chcp 437 >nul
set "KTOA_HOME=%USERPROFILE%\ktoa\ktoa-4.3.4.0-kat7.0-windows"
set "KATANA_TAGLINE=Foundry Support"
set "KATANA_ROOT=C:\Program Files\Katana7.0v4"
set "KATANA_RESOURCES=%KATANA_RESOURCES%;%KATANA_ROOT%\plugins\Resources\Examples"
set "KATANA_RESOURCES=%KATANA_RESOURCES%;%KTOA_HOME%;"
set "KATANA_RESOURCES=%KATANA_RESOURCES%;%KTOA_HOME%\USD\KatanaUsdArnold;"
set "KATANA_RESOURCES=%KATANA_RESOURCES%;C:\dev\infx\flova\flova\katana;"
set "KATANA_RESOURCES=%KATANA_RESOURCES%;S:\Lighting"
set "DEFAULT_RENDERER=arnold"
set "PATH=%KTOA_HOME%\bin;%PATH%"
set "PYTHONPATH=C:\dev\infx\flova;W:\inhouse\flova_libs;C:\Python310\Lib\site-packages;"
set "FNPXR_PLUGINPATH=%KTOA_HOME%\USD\Viewport;%FNPXR_PLUGINPATH%"
"%KATANA_ROOT%\bin\katanaBin.exe" %*
```

**주의**: Katana는 자체 PySide2가 없어서 `C:\Python310\Lib\site-packages` 필요

#### Nuke, Houdini, 3D Equalizer

**계획**: 배치파일 방식으로 전환 예정

**배치파일 표준 템플릿** (개발자/비개발자 자동 판별):
```batch
@echo off
chcp 437 >nul

REM 개발자/비개발자 자동 판별
if exist "C:\dev\infx" (
    set "FLOVA_ROOT=C:\dev\infx\flova"
) else (
    set "FLOVA_ROOT=W:\inhouse\flova"
)

REM DCC별 PYTHONPATH 설정
set "PYTHONPATH=%FLOVA_ROOT%;%FLOVA_ROOT%\flova\maya\startup;W:\inhouse\flova_libs;W:\inhouse\pymel;"

REM DCC 실행
"C:\Program Files\Maya2024\bin\maya.exe" %*
```

#### Hiero

- Nuke 설치 시 제공되는 기본 실행 파일 사용
- 필요 시 배치파일로 전환 가능

### 4.3 배치파일 관리

**배치파일 위치**
```
개발자:   C:\dev\infx\flova\flova\exec\
비개발자: W:\inhouse\flova\flova\exec\
```

**PATH 설정**
- 개발자: 수동으로 `C:\dev\infx\flova\flova\exec` 추가
- 비개발자: IT팀이 `W:\inhouse\flova\flova\exec` 추가 (AD 관리)

**네이밍 규칙**
- `maya2024.bat`, `nuke15.bat`, `houdini20.bat` 등
- 버전별로 배치파일 분리
- 하나의 배치파일에서 개발자/비개발자 자동 판별

---

## 5. 배포 프로세스

### 5.1 Flova 배포

1. 개발자가 GitLab에 push
2. 배포 담당자(개발자 3명 중 1명)가 배치파일 실행
3. `W:\inhouse\flova`로 pull하여 배포
4. 배포 완료 공지

**배포 스크립트**: 별도 배치파일로 관리

### 5.2 타임로그 웹서버 배포

1. 타임로그 서버(192.168.10.27)에 직접 접속
2. GitLab pull로 코드 업데이트
3. Gunicorn 서버 재가동

**주의**: `W:\inhouse`에는 배포하지 않음

### 5.3 Git 워크플로우

- **브랜치 전략**: 없음 (모두 main 브랜치 직접 사용)
- **작업 공유**: 작업 전 팀원 간 내용 공유로 충돌 방지
- **충돌 처리**: 팀장(기경훈)이 서포트
- **개발 방식**: 빠른 배포 → 사용자 피드백 → 수정 → 재배포

---

## 6. 알려진 제약사항 및 이슈

### 6.1 환경변수 즉각 반영 불가

**문제**
- IT팀 AD 관리 환경변수는 재부팅 시에만 반영
- 수동 변경도 락 걸려 있어 어려움

**해결 계획**
- DCC 실행 배치파일로 전환하여 즉각 반영 가능하게 개선

### 6.2 PYTHONPATH 사용 불가

**문제**
- Windows 시스템 환경변수의 PYTHONPATH가 일부 DCC 배치파일과 충돌

**해결**
- 전사적 PYTHONPATH 설정 제거
- DCC별 배치파일 또는 설정 파일에서 개별 관리

### 6.3 OCIO 중앙 관리 불가

**문제**
- 일부 팀에서 OCIO 커스터마이징 요구

**해결**
- OCIO 환경변수 설정하지 않음
- 필요 시 각 팀/프로젝트에서 수동 설정
- 표준 파일 제공: `W:\inhouse\ocio\INFX_config.ocio`

---

## 7. 주요 DCC 및 통합 시스템

### 7.1 사용 중인 DCC

- **Maya**: 3D 모델링/애니메이션
- **Nuke**: 합성
- **Hiero**: 편집/관리
- **Houdini**: 이펙트
- **Katana**: 라이팅/렌더링
- **3D Equalizer**: 매치무브/트래킹

### 7.2 통합 시스템

- **Shotgrid**: 프로젝트/에셋 관리, 메인 DB
- **Deadline**: 렌더 관리

---

## 8. 관련 문서

본 문서와 함께 참고해야 할 스타일 가이드:

- `infx_coding_style_guide.md`: Python 코딩 스타일 규칙
- `infx_pyside2_coding_guide.md`: Flova UI 프레임워크 사용 규칙
- `infx_pip_guide.md`: Python 패키지 관리 가이드

**코딩 시 반드시 위 두 문서를 참고하여 일관된 코드 스타일을 유지해야 합니다.**

---

## 9. 문서 업데이트

- **작성일**: 2026-01-22
- **작성자**: 기경훈
- **버전**: 1.0

이 문서는 inFX 개발 환경 변경 시 지속적으로 업데이트됩니다.
