# inFX Python 패키지 관리 가이드

본 문서는 inFX 환경에서 Python 패키지를 설치하고 관리하는 방법을 정리한 문서입니다.

---

## 1. 기본 환경

### 1.1 Python 버전
- **Python**: 3.10.11 (당분간 버전업 계획 없음)

### 1.2 시스템 Python 경로
```
C:\Programs\Python310\Lib\site-packages\
```

**중요**: 이 경로는 PYTHONPATH에 추가하지 않습니다.
- DCC 자체 PySide2/shiboken과 충돌 발생 가능

---

## 2. 패키지 설치 전략

inFX는 패키지를 두 가지 방식으로 분리하여 관리합니다.

### 2.1 공용 라이브러리 (W:\inhouse\flova_libs)

DCC와 충돌 없는 일반 패키지를 설치합니다.

**설치 방법 (개발자 자리)**
```bash
pip install <package> --target=W:\inhouse\flova_libs
```

**예시**
```bash
pip install requests --target=W:\inhouse\flova_libs
pip install fileseq --target=W:\inhouse\flova_libs
pip install pillow --target=W:\inhouse\flova_libs
```

### 2.2 로컬 전용 패키지 (C:\Programs\Python310\Lib\site-packages)

DCC와 충돌 가능성이 있는 패키지를 각 자리의 로컬에 설치합니다.

**대상 패키지**
- `numpy==1.26.4` (2.x 버전은 shiboken 충돌)
- `pyside2`
- `pywin32`
- `openexr`

**설치 방법**
```bash
pip install numpy==1.26.4
pip install pyside2
pip install pywin32
pip install openexr
```

---

## 3. 비개발자 자리 패키지 설치

비개발자(아티스트) 자리는 인터넷 폐쇄망 환경이므로, 로컬 wheel 파일을 통해 설치합니다.

### 3.1 Wheel 파일 다운로드 (개발자 자리에서)

```bash
pip3.10 download numpy==1.26.4 pyside2 pywin32 openexr -d W:\inhouse\pypi-wheels --only-binary=:all:
```

### 3.2 Wheel 파일로 설치 (비개발자 자리에서)

```bash
pip install numpy==1.26.4 pyside2 pywin32 openexr --no-index --find-links=W:\inhouse\pypi-wheels
```

---

## 4. 알려진 충돌 및 제약사항

### 4.1 PySide2/shiboken 충돌

**문제**
- `C:\Programs\Python310\Lib\site-packages`를 PYTHONPATH에 추가 시
- Maya 자체 PySide2의 shiboken과 버전 충돌 발생

**해결**
- 공용 라이브러리: `W:\inhouse\flova_libs` 사용
- 충돌 가능 패키지: 로컬 site-packages에만 설치

### 4.2 numpy 버전 고정

**문제**
- numpy 2.x 버전은 특정 라이브러리(shiboken)와 충돌

**해결**
- `numpy==1.26.4` 버전 고정
- 로컬 site-packages에 설치

---

## 5. 경로 구조

### 5.1 공용 라이브러리 경로
```
W:\inhouse\flova_libs\  # DCC 충돌 없는 일반 패키지
```

### 5.2 Wheel 파일 저장소
```
W:\inhouse\pypi-wheels\  # 오프라인 설치용 wheel 파일
```

### 5.3 로컬 패키지 경로
```
C:\Programs\Python310\Lib\site-packages\  # 충돌 가능 패키지
```

---

## 6. 패키지 분류 가이드

### 6.1 공용 라이브러리로 설치할 패키지
- 순수 Python 패키지 (C 확장 없음)
- DCC와 독립적인 유틸리티 라이브러리
- API 클라이언트 라이브러리
- 파일 처리 라이브러리

**예시**
- `requests`
- `fileseq`
- `pillow`
- `shotgun_api3`

### 6.2 로컬에만 설치할 패키지
- Qt 관련 패키지 (PySide2, PyQt 등)
- C 확장을 포함한 바이너리 패키지
- DCC와 버전 충돌 가능성이 있는 패키지
- Windows API 관련 패키지

**예시**
- `numpy==1.26.4`
- `pyside2`
- `pywin32`
- `openexr`
