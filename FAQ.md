# 자주 묻는 질문 (FAQ) 🤔

inFX 개발 중 자주 나오는 질문과 답변을 모았습니다.

---

## 📖 목차

- [환경 설정](#환경-설정)
- [Python & 패키지](#python--패키지)
- [네트워크 & 인터넷](#네트워크--인터넷)
- [DCC 통합](#dcc-통합)
- [Git & 배포](#git--배포)
- [코딩 스타일](#코딩-스타일)
- [Shotgrid](#shotgrid)
- [기타](#기타)

---

## 환경 설정

### Q. 개발자 자리와 비개발자 자리의 차이가 뭔가요?

**A.** 경로 구조가 다릅니다:

- **개발자**: `C:\dev\infx\flova` (GitLab에서 직접 pull)
- **비개발자**: `W:\inhouse\flova` (배포된 코드 사용)

개발자는 코드를 수정하고 테스트할 수 있지만, 비개발자는 배포된 안정 버전만 사용합니다.

---

### Q. PYTHONPATH를 왜 환경변수로 설정하지 않나요?

**A.** Windows 시스템 환경변수의 PYTHONPATH가 일부 DCC 배치파일과 충돌하기 때문입니다.

대신 각 DCC의 설정 파일이나 배치파일에서 개별적으로 관리합니다:
- Maya: `maya.env`
- Katana: 배치파일 (`.bat`)
- 기타 DCC: 배치파일로 전환 예정

---

### Q. 새로운 패키지를 설치하려면 어떻게 하나요?

**A.** 패키지 종류에 따라 다릅니다:

**1. 일반 패키지 (DCC와 충돌 없음)**
```bash
pip install <package> --target=W:\inhouse\flova_libs
```

**2. 충돌 가능 패키지 (numpy, pyside2, pywin32, openexr 등)**
```bash
# 개발자 자리: 로컬에 바로 설치
pip install <package>

# 비개발자 자리: wheel 파일 사용
pip install <package> --no-index --find-links=W:\inhouse\pypi-wheels
```

> 💡 **팁**: 잘 모르겠으면 TD팀에 문의하세요!

---

### Q. 환경변수 변경이 바로 적용되지 않아요!

**A.** IT팀이 Active Directory로 관리하는 환경변수는 **시스템 재부팅 시에만 적용**됩니다.

즉시 반영이 필요하면 배치파일 방식을 사용하세요. 이것이 우리가 DCC 실행을 배치파일로 전환하는 이유입니다.

---

## Python & 패키지

### Q. Python 버전을 올려도 되나요?

**A.** 당분간은 **3.10.11로 고정**입니다.

DCC들이 지원하는 Python 버전을 고려해야 하고, 모든 자리의 Python 버전을 동기화해야 하기 때문에 신중하게 결정합니다.

---

### Q. numpy 2.x를 사용하면 안 되나요?

**A.** 안 됩니다! **numpy 1.26.4로 고정**해야 합니다.

numpy 2.x는 shiboken과 충돌이 발생합니다. 반드시 1.26.4 버전을 사용하세요.
```bash
pip install numpy==1.26.4 --break-system-packages
```

---

### Q. PySide2와 PySide6 중 뭘 써야 하나요?

**A.** 무조건 **PySide2**입니다.

DCC 호환성과 기존 코드와의 일관성을 위해 PySide2만 사용합니다. PyQt는 사용하지 않습니다.

---

### Q. `C:\Programs\Python310\Lib\site-packages`를 PYTHONPATH에 추가하면 안 되나요?

**A.** 절대 안 됩니다! 🔥

Maya 등 DCC 자체 PySide2의 shiboken과 버전 충돌이 발생합니다. 충돌 가능한 패키지는 로컬 site-packages에만 설치하고, PYTHONPATH에는 추가하지 마세요.

---

## 네트워크 & 인터넷

### Q. 인터넷이 안 돼요!

**A.** 우리는 **폐쇄망**을 사용합니다.

- **개발자**: 프록시 서버 `192.168.10.40:23128` 사용 (승인된 도메인만 접근)
- **비개발자**: 완전 폐쇄 (외부 인터넷 불가)

개발자가 새 도메인에 접근하려면 IT팀에 요청해야 합니다.

---

### Q. GitHub에 접속이 안 돼요!

**A.** GitHub은 **접근 불가** 도메인입니다.

대신 사내 GitLab(`https://git.infx.kr`)을 사용합니다.

---

### Q. pip install이 안 돼요!

**A.** 프록시 설정을 확인하세요:
```bash
# 개발자 자리
pip install <package> --proxy=192.168.10.40:23128

# 또는 환경변수 설정
set HTTP_PROXY=192.168.10.40:23128
set HTTPS_PROXY=192.168.10.40:23128
pip install <package>
```

비개발자 자리에서는 wheel 파일을 사용해야 합니다.

---

## DCC 통합

### Q. Maya에서 Flova 모듈을 import하지 못해요!

**A.** `maya.env` 파일의 PYTHONPATH를 확인하세요:

**개발자:**
```
PYTHONPATH=C:/dev/infx/flova;C:/dev/infx/flova/flova/maya/startup;W:/inhouse/flova_libs;W:/inhouse/pymel;
```

**비개발자:**
```
PYTHONPATH=W:/inhouse/flova;W:/inhouse/flova/flova/maya/startup;W:/inhouse/flova_libs;W:/inhouse/pymel;
```

> ⚠️ **주의**: 경로 구분자는 슬래시(`/`)를 사용하세요!

---

### Q. DCC 배치파일은 어디에 있나요?

**A.** 

- **개발자**: `C:\dev\infx\flova\flova\exec\`
- **비개발자**: `W:\inhouse\flova\flova\exec\`

개발자는 수동으로 PATH에 추가해야 하고, 비개발자는 IT팀이 자동으로 설정합니다.

---

### Q. OCIO 설정은 어디에 있나요?

**A.** `W:\inhouse\ocio\INFX_config.ocio`에 표준 파일이 있습니다.

하지만 OCIO 환경변수는 전사적으로 설정하지 않습니다. 필요한 팀/프로젝트에서 수동으로 설정하세요.

---

## Git & 배포

### Q. 브랜치 전략이 있나요?

**A.** 없습니다. 모두 **main 브랜치를 직접 사용**합니다.

작업 전 팀원들과 내용을 공유하여 충돌을 방지합니다. 충돌 발생 시 팀장(기경훈)이 서포트합니다.

---

### Q. 코드를 수정했는데 아티스트들이 볼 수 없어요!

**A.** 배포가 필요합니다!

1. GitLab에 push
2. 배포 담당자(개발자 3명 중 1명)가 배치파일 실행
3. `W:\inhouse\flova`로 pull
4. 배포 완료 공지

개발자 자리(`C:\dev\infx`)의 수정 사항은 자동으로 배포되지 않습니다.

---

### Q. Git 충돌이 났어요!

**A.** 일단 침착하게 팀장(기경훈)에게 연락하세요.

단순 충돌이면 직접 해결해도 되지만, 복잡하면 도움을 받는 것이 안전합니다.

---

## 코딩 스타일

### Q. camelCase를 써도 되나요?

**A.** 안 됩니다! 무조건 **snake_case**입니다.

변수명, 함수명 모두 소문자와 언더스코어만 사용하세요.
```python
# 올바른 예
def get_user_name(user_id):
    pass

# 잘못된 예
def getUserName(userId):  # ❌
    pass
```

---

### Q. 문자열은 작은따옴표? 큰따옴표?

**A.** 기본은 **작은따옴표(`'`)**입니다.

문자열 내에 작은따옴표가 포함된 경우에만 큰따옴표(`"`)를 사용하세요.
```python
name = 'hello'  # ✅
message = "It's a good day"  # ✅ (작은따옴표 포함)
name = "hello"  # ❌
```

---

### Q. 리스트 마지막에 쉼표를 찍어야 하나요?

**A.** 네, **trailing comma**를 사용합니다.
```python
my_list = [
    'first',
    'second',
    'third',  # ✅ trailing comma
]
```

이렇게 하면 나중에 항목 추가 시 diff가 깔끔해집니다.

---

### Q. docstring은 어떤 스타일을 쓰나요?

**A.** **Google 스타일**입니다.
```python
def get_job_status(job_id, timeout):
    """
    Deadline Job의 상태를 확인한다.

    Args:
        job_id (str):
            Deadline Job ID

        timeout (int):
            최대 대기 시간 (초)

    Returns:
        dict or None:
            Job 정보 또는 None
    """
    pass
```

---

### Q. 클래스명은 어떻게 지어야 하나요?

**A.** 용도에 따라 다릅니다:

- **내부용 클래스**: `_FileGroupWidget` (언더스코어 접두사)
- **Public 클래스**: `FlovaVersionWindow` (PascalCase)
- **Mixin 클래스**: `_ElementListItemCacheImportMixin` (Mixin 접미사)
- **예외 클래스**: `UnknownTaskNameException` (Exception 접미사)
- **캐시 클래스**: `SGProjectCache` (Cache 접미사)

---

## Shotgrid

### Q. Shotgrid API는 어떻게 사용하나요?

**A.** `InfxShotgrid` 클래스를 사용하세요:
```python
from flova.shotgrid import InfxShotgrid

sg = InfxShotgrid(script='td')
projects = sg.find('Project', [], ['name', 'code'])
```

자세한 내용은 추후 추가될 "Shotgrid API 코딩 가이드"를 참고하세요.

---

### Q. Shotgrid 캐시는 어떻게 사용하나요?

**A.** 프로젝트별 캐시 클래스를 사용하세요:
```python
from flova.shotgrid.cache import SGProjectCache, SGShotCache

# 프로젝트 목록
for project in SGProjectCache().get_data():
    print(project['name'])

# 특정 프로젝트의 샷 목록
for shot in SGShotCache('MY_PROJECT').get_data():
    print(shot['code'])
```

---

## 기타

### Q. 로거는 어떻게 사용하나요?

**A.** `flova.log.get_logger()`를 사용하세요:
```python
from flova.log import get_logger

log = get_logger('my_module')

log.debug('디버그 메시지')
log.info('정보 메시지')
log.warning('경고 메시지')
log.error('에러 메시지')
```

---

### Q. 타임로그 서버는 어떻게 접속하나요?

**A.** 사내망에서 `http://192.168.10.27`로 접속하세요.

외부에서는 접속할 수 없습니다.

---

### Q. 궁금한 게 이 문서에 없어요!

**A.** TD팀 채널에 질문하거나 팀장(기경훈)에게 직접 물어보세요!

자주 나오는 질문은 이 FAQ에 추가됩니다.
