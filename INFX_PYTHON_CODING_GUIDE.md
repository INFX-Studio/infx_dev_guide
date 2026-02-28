# inFX 프로그래밍 스타일 가이드

이 문서는 프로젝트 코드 작성 시 따라야 할 스타일 지침을 정리한 것이다.

---

## PEP-8 준수

Python 코드는 PEP-8 스타일 가이드를 최대한 준수한다.

- 들여쓰기는 스페이스 4칸을 사용한다.
- 한 줄의 최대 길이는 79자를 권장하며, 최대 120자를 넘지 않는다. 단, `from ... import` 문은 예외로 줄 길이 제한을 초과해도 된다.
- 모듈 레벨에서 함수와 함수 사이, 클래스와 클래스 사이는 2줄을 띄운다.
- 클래스 내부에서 메서드와 메서드 사이는 1줄을 띄운다.
- 연산자 앞뒤로 스페이스를 넣는다. 단, 함수 인자의 기본값 `=`는 스페이스 없이 붙여 쓴다.
- 콤마 뒤에는 스페이스를 넣는다.
- 불필요한 공백을 넣지 않는다.
- 파일의 맨 마지막은 반드시 빈 줄 하나로 끝낸다.
- 변수명, 함수명은 반드시 소문자와 언더스코어를 결합한 snake_case를 사용한다. camelCase는 사용하지 않는다.
- 문자열은 작은 따옴표(`'`)를 사용한다. 문자열 내에 작은 따옴표가 포함된 경우에만 큰 따옴표(`"`)를 사용한다.

```python
# 작은 따옴표 사용 (O)
name = 'hello'
message = 'world'

# 문자열 내에 작은 따옴표가 있을 때 큰 따옴표 사용 (O)
message = "It's a good day"

# 큰 따옴표 사용 (X)
name = "hello"
```

- 리스트, 딕셔너리 등을 여러 줄로 작성할 때는 4칸 들여쓰기를 사용하고, 마지막 요소에 trailing comma를 넣고 닫는 괄호는 다음 줄에 둔다.

```python
my_list = [
    'first',
    'second',
    'third',
]

my_dict = {
    'name': 'job_001',
    'status': 'Active',
    'priority': 50,
}
```

---

## 모듈 docstring 사용하지 않음

파일 최상단에 모듈을 설명하는 docstring을 작성하지 않는다.

```python
# 사용하지 않음 (X)
"""
이 모듈은 플로바 태스크 앱의 메인 윈도우를 정의한다.

FlovaTaskWindow 클래스를 포함하며...
"""
from __future__ import annotations

# 파일 시작 (O)
from __future__ import annotations
```

---

## Import 구조

- `import` 구조들 사이에는 빈 줄을 넣지 않는다.
- `from __future__ import annotations`는 맨 위에 위치한다.
- `from ... import`를 먼저, `import`를 나중에 작성한다.
- `from ... import` 시 여러 항목을 가져올 때 괄호로 세로 나열하지 않고 가로로 나열한다. 이 경우 줄 길이 제한(79자/120자)을 초과해도 된다.
- 각 그룹 내에서는 로컬 > 서드파티 > 표준 라이브러리 순서로 작성한다.

```python
from __future__ import annotations
from flova.app.flova_task.constants import log, sg_status_manager, COMP_TASK, ENV_TASK
from flova.app.flova_task.exceptions import UnknownTaskNameException
from flova.ui.widget import *
from flova.ui.selector import UserTaskSelector
from flova.shotgrid import InfxShotgrid, ShotgridTask, ShotgridPipelineStep, ShotgridStatus, ShotgridUtil
from flova.path import InfxPath, WorkPath
import fileseq
import os
import re
import sys
import json
```

```python
# 괄호로 세로 나열하지 않음 (X)
from flova.shotgrid import (
    InfxShotgrid,
    ShotgridTask,
    ShotgridPipelineStep,
)

# 가로 나열 (O)
from flova.shotgrid import InfxShotgrid, ShotgridTask, ShotgridPipelineStep
```

---

## PySide 호환성 Import

PySide6과 PySide2를 모두 지원할 때는 PySide2를 먼저 시도한다. inFX에서는 PySide2를 메인으로 사용하기 때문이다.

```python
try:
    from PySide2.QtGui import QIcon, QPixmap, QColor
    from PySide2.QtCore import Qt, QSettings, QTimer, QSize
    from PySide2.QtWidgets import QSplitter, QWidget, QVBoxLayout, QHBoxLayout
except:
    from PySide6.QtGui import QIcon, QPixmap, QColor
    from PySide6.QtCore import Qt, QSettings, QTimer, QSize
    from PySide6.QtWidgets import QSplitter, QWidget, QVBoxLayout, QHBoxLayout
```

---

## Python 버전 호환성

- Python 3.7 이상을 지원해야 한다.
- 가급적 타입 힌트를 사용한다.
- 타입 힌트 사용 시 3.7에서 지원되지 않는 문법에 주의한다.
  - `dict` 대신 `Dict`, `list` 대신 `List` 등 `typing` 모듈 사용
  - `|` 연산자 대신 `Union` 사용
- 파라미터가 많아 줄이 길어지면 줄바꿈으로 처리한다.

```python
from typing import Dict, List, Optional, Union

# 파라미터가 적을 때
def get_job_status(job_id: str, timeout: int = 30) -> Optional[dict]:
    pass

# 파라미터가 많을 때
def submit_render_job(
    scene_file: str,
    output_path: str,
    frame_range: str,
    pool: str = 'default',
    priority: int = 50,
    timeout: Optional[int] = None,
) -> dict:
    pass
```

---

## 상수 (Constants)

- 모듈 레벨 상수만 대문자 SNAKE_CASE를 사용한다.
- 함수나 메서드 내부의 지역 변수는 소문자 snake_case를 사용한다.
- 확장자 목록 등은 튜플로 정의한다.

```python
# 모듈 레벨 상수 (O)
DEFAULT_TIMEOUT = 30
DELIGATE_ICON_SIZE = 15

# 확장자 목록은 튜플로 정의
IMAGE_EXT = ('.dpx', '.exr', '.jpg', '.jpeg', '.png', '.gif', '.bmp', '.tif', '.tiff', '.tga')
VIDEO_EXT = ('.mp4', '.mov', '.avi', '.mkv', '.flv', '.wmv', '.ogv')

# 딕셔너리 매핑
DCC_EXEC_MAP = {
    'maya': flova.config.CURRENT_USE_MAYA_EXEC,
    'houdini': flova.config.CURRENT_USE_HOUDINI_EXEC,
    'nuke': flova.config.CURRENT_USE_NUKE_EXEC,
}

# 함수 내부 지역 변수 (O)
def get_status(code):
    stat_map = {
        1: 'Active',
        2: 'Suspended',
    }
    return stat_map.get(code)

# 함수 내부에서 대문자 사용 (X)
def get_status(code):
    STAT_MAP = {  # 잘못된 사용
        1: 'Active',
        2: 'Suspended',
    }
    return STAT_MAP.get(code)
```

---

## 로거 (Logger)

- `flova.log.get_logger()`를 사용하여 모듈 레벨에서 로거를 정의한다.
- 로거 변수명은 `log`를 사용한다.
- 로거 이름은 모듈 이름과 동일하게 한다.

```python
from flova.log import get_logger

log = get_logger('module_name')


def some_function():
    log.debug('디버그 메시지')
    log.info('정보 메시지')
    log.warning('경고 메시지')
    log.error('에러 메시지')
```

---

## 섹션 구분 주석

코드의 논리적 섹션을 구분할 때 다음과 같은 주석 스타일을 사용한다.

```python
####################################################################################################
# 대형 섹션 제목
####################################################################################################
some_code_here()


##################################################
# 중형 섹션 제목
##################################################
another_code_here()
```

---

## 클래스 네이밍

- **내부용 클래스**: 언더스코어 접두사를 사용한다. (`_File`, `_FileGroupWidget`, `_Filter`)
- **Public 클래스**: 일반 PascalCase를 사용한다. (`FlovaVersionWindow`, `WorkFile`)
- **Mixin 클래스**: `Mixin` 접미사를 사용한다. (`_ElementListItemCacheImportMixin`)
- **예외 클래스**: `Exception` 또는 `Error` 접미사를 사용한다. (`UnknownTaskNameException`, `InvalidArgumentError`)
- **캐시 클래스**: `Cache` 접미사를 사용한다. (`SGProjectCache`, `SGShotCache`)

```python
# 내부용 클래스 (O)
class _FileGroupWidget(FlovaGroupBox):
    pass


# Public 클래스 (O)
class FlovaVersionWindow(FlovaMainWindow):
    pass


# Mixin 클래스 (O)
class _ElementListItemCacheImportMixin:
    pass


# 예외 클래스 (O)
class UnknownTaskNameException(Exception):
    def __init__(self, unknown_task_name, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.unknown_task_name = unknown_task_name


# 캐시 클래스 (O)
class SGShotCache(_SGCache):
    pass
```

---

## 클래스 구조 패턴

### 캐시 클래스 구조

캐시 클래스는 공통 베이스 클래스를 상속하고 필수 상수를 정의한다.

```python
class SGShotCache(_SGCache):

    SG_ENTITY_TYPE = 'Shot'

    SG_FIELD_LIST = [
        'project',
        'sg_sequence',
        'code',
        'sg_status_list',
    ]

    JOIN_RULES = {
        'sg_sequence': [SGSequenceCache],
    }

    def __init__(self, project, logger=None):
        if isinstance(project, dict):
            self._pcode = project['name']
        elif isinstance(project, str):
            self._pcode = project
        else:
            raise Exception('project 인자는 샷그리드 엔티티(dict) 또는 프로젝트 코드여야 합니다.')

        super().__init__(
            redis_key_name=f'sg:shot:{self._pcode}',
            logger=logger,
        )

        filters = [
            ['project.Project.name', 'is', self._pcode],
        ]
        self.set_sg_filters(filters)
        self.set_sg_fields(self.SG_FIELD_LIST)
```

### 데이터 클래스 구조

단순 데이터를 담는 클래스는 `__init__`에서 속성만 초기화한다.

```python
class SGStatus:

    def __init__(self, sg_status):
        self._sg_status = sg_status
        self.code = sg_status['code']
        self.name = sg_status['name']
        self.id = sg_status['id']
        self.foreground = self.normalize_color(sg_status['sg_font_color'], '#FFFFFF')
        self.background = self.normalize_color(sg_status['bg_color'], '#000000')
```

### 네임스페이스 클래스 구조

인스턴스 생성 없이 `@classmethod`, `@staticmethod`만 사용하는 유틸리티 클래스이다.

```python
class ShotgridUtil:

    @classmethod
    def is_same_artist_reviewer(cls, sg_task):
        if not sg_task:
            return
        # ...

    @staticmethod
    def get_name_from_hostname(hostname):
        sg = InfxShotgrid(script='td')
        # ...
```

---

## 클래스 상수 리스트 패턴

API 필드나 설정 목록은 클래스 레벨 상수로 정의한다.

```python
class SGProjectCache(_SGCache):

    SG_ENTITY_TYPE = 'Project'

    SG_FIELD_LIST = [
        'name',  # 프로젝트 코드
        'code',  # 서버의 프로젝트 루트 폴더의 이름
        'sg_display_name',  # 프로젝트 이름 (한글)
        'sg_fps',  # FPS
    ]
```

---

## 양방향 매핑 딕셔너리 패턴

코드 변환이 필요할 때 양방향 매핑 딕셔너리를 사용한다.

```python
class ShotgridTask:

    MODELING = 'modeling'
    LOOKDEV = 'lookdev'
    MOD = 'mod'
    LKD = 'lkd'

    SHORT_NAME_DATA = {
        MODELING: MOD,
        LOOKDEV: LKD,
    }

    LONG_NAME_DATA = {
        MOD: MODELING,
        LKD: LOOKDEV,
    }

    @classmethod
    def short_name(cls, name):
        if name in cls.SHORT_NAME_DATA:
            return cls.SHORT_NAME_DATA[name]

    @classmethod
    def long_name(cls, code):
        if code in cls.LONG_NAME_DATA:
            return cls.LONG_NAME_DATA[code]
```

---

## 유연한 인자 타입 처리 패턴

dict 또는 str 타입을 모두 허용하는 패턴이다.

```python
def __init__(self, project, logger=None):
    if isinstance(project, dict):
        self._pcode = project['name']
    elif isinstance(project, str):
        self._pcode = project
    else:
        raise Exception('project 인자는 샷그리드 엔티티(dict) 또는 프로젝트 코드여야 합니다.')
```

---

## 전역 커넥션 관리 패턴

전역 변수와 클래스를 조합하여 커넥션을 관리한다.

```python
_sg_conn_data = {}


class InfxShotgridConnection:

    @classmethod
    def get_connection(cls, script: str) -> shotgun_api3.Shotgun:
        global _sg_conn_data
        if script not in _sg_conn_data:
            sg = InfxShotgrid(script=script)
            _sg_conn_data[script] = sg

        return _sg_conn_data[script]
```

---

## 데코레이터 정의 패턴

데코레이터를 정의할 때는 `functools.wraps`를 사용하여 원본 함수의 메타데이터를 보존한다.

```python
from datetime import timedelta
import time
import functools


def elapsed_time(func):

    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        end = time.perf_counter()
        elapsed = end - start
        td = timedelta(seconds=elapsed)
        total_seconds = int(td.total_seconds())
        hours, remainder = divmod(total_seconds, 3600)
        minutes, seconds = divmod(remainder, 60)
        milliseconds = int((elapsed - int(elapsed)) * 100)

        print(f'[{func.__name__}] 실행시간: {hours:02}시간 {minutes:02}분 {seconds:02}.{milliseconds:02}초')

        return result

    return wrapper
```

---

## 지연 로딩 패턴 (Lazy Loading)

비용이 큰 리소스(DB 커넥션, API 호출 등)는 처음 접근할 때 로딩하고 캐싱한다.

```python
class WorkFile:

    def __init__(self, work_file):
        # 샷그리드 커넥션은 나중에 필요할 때 생성
        self._sg = None

        # 샷그리드 엔티티도 지연 로딩
        self._sg_project = None
        self._sg_shot = None
        self._sg_task = None

    def _create_sg_connection(self):
        """샷그리드 커넥션을 생성한다."""
        self._sg = InfxShotgrid(script='infx_common')

    def sg_project(self):
        """샷그리드 프로젝트 엔티티를 반환한다."""
        if not self._sg_project:
            for sg_prj in SGProjectCache().get_data():
                if sg_prj['name'] == self.project_code():
                    self._sg_project = sg_prj
                    break

        return self._sg_project

    def sg_shot(self):
        """샷그리드 샷 엔티티를 반환한다."""
        if not self._sg_shot:
            pcode = self.project_code()
            scode = self.shot_code()
            for sg_shot in SGShotCache(pcode).get_data():
                if sg_shot['code'] == scode:
                    self._sg_shot = sg_shot
                    break

        return self._sg_shot
```

---

## 프로퍼티 스타일 Getter 메서드 패턴

클래스의 속성을 반환하는 메서드는 간결하게 작성한다.

```python
class WorkFile:

    def __init__(self, work_file):
        self._path = os.path.dirname(work_file)
        self._filename = os.path.basename(work_file)
        self._project_code = None
        self._is_valid = None

    def path(self):
        """경로를 반환한다."""
        return self._path

    def filename(self):
        """파일이름을 반환한다."""
        return self._filename

    def project_code(self):
        """프로젝트 코드를 반환한다."""
        return self._project_code

    def is_valid(self):
        """인스턴스가 유효한지 반환한다."""
        return self._is_valid
```

---

## 예외 클래스 정의 패턴

예외 클래스는 명확한 메시지를 포함하여 정의한다.

```python
class InvalidArgumentError(Exception):

    def __init__(self, *args):
        super().__init__(*args)


class DoesNotExistShotGridProjectError(Exception):

    def __init__(self):
        message = (
            '샷그리드 프로젝트 코드를 찾을 수 없습니다. '
            '올바른 프로젝트 코드가 입력된 것인지 다시 한 번 확인해보세요. '
            '또는, 샷그리드 프로젝트 캐쉬를 확인해보세요.'
        )
        super().__init__(message)


class DoesNotExistsShotPathRegularExpressionError(Exception):

    def __init__(self, shot_code=None):
        message = (
            f'샷그리드 프로젝트({shot_code})에 샷 경로를 정의해줄 정규식 패턴이 없습니다. '
            f'sg_shot_path_regular_expression 필드가 없거나 '
            f'필드에 값이 없는 것은 아닌지 확인해보세요.'
        )
        super().__init__(message)
```

---

## 경로 변환 매핑 상수 패턴

플랫폼별 경로 변환이 필요할 때 양방향 매핑 딕셔너리를 사용한다.

```python
UNC_MAPPING = {
    'G:/': '//192.168.10.207/volmulti/',
    'M:/': '//192.168.10.200/vol01/',
    'R:/': '//192.168.10.200/vol02/',
    'S:/': '//192.168.10.200/vol03/',
    'W:/': '//192.168.10.200/vol05/',
}

REVERSE_UNC_MAPPING = {
    '//192.168.10.207/volmulti/': 'G:/',
    '//192.168.10.200/vol01/': 'M:/',
    '//192.168.10.200/vol02/': 'R:/',
    '//192.168.10.200/vol03/': 'S:/',
    '//192.168.10.200/vol05/': 'W:/',
}
```

---

## 순수 classmethod 유틸리티 클래스 패턴

인스턴스 없이 `@classmethod`만 사용하는 유틸리티 클래스이다. 관련 기능을 네임스페이스로 그룹화할 때 사용한다.

```python
class Shot:

    @classmethod
    def get_resolution(cls, project_code: str, sequence_code: str, shot_code: str) -> tuple:
        """
        주어진 프로젝트, 시퀀스, 샷 코드를 이용하여 해당 샷의 해상도를 반환한다.

        Args:
            project_code (str):
                프로젝트 코드

            sequence_code (str):
                시퀀스 코드

            shot_code (str):
                 샷 코드

        Returns:
            tuple(int, int): 해상도 Width, Height

        """
        width, height = None, None
        # ...
        return width, height

    @classmethod
    def get_fps(cls, project_code: str, sequence_code: str, shot_code: str) -> float:
        """해당 샷의 FPS를 반환한다."""
        # ...

    @classmethod
    def get_pixel_aspect_ratio(cls, project_code: str, sequence_code: str, shot_code: str) -> float:
        """해당 샷의 Pixel Aspect Ratio를 반환한다."""
        # ...
```

---

## 모듈 레벨 유틸리티 함수 패턴

자주 사용되는 유틸리티 함수는 모듈 레벨에 정의한다.

```python
def basenameex(filename):
    """경로와 확장자를 제외한 순수 파일명을 반환한다."""
    return os.path.splitext(os.path.basename(filename))[0]


def normalized(path):
    """
    주어진 경로를 프로그래밍 하기 좋은 말끔한 경로로 반환한다.

    1. 경로의 구분자를 슬래쉬(/)로 모두 변경하고,
    2. 중첩된 구분자(//)는 하나로 통일(/)하며,
    3. 앞뒤 공백을 없앤다.

    Args:
        path (str): 정리할 경로

    Returns:
        str: 말끔하게 정리된 경로
    """
    if path is None:
        raise Exception('None 타입은 정리할 수 없습니다.')

    path = path.strip()
    path = path.replace('\\', '/')
    path = path.replace('//', '/')

    return path


def dirs(path):
    """
    주어진 경로를 반드시 생성하여 반환한다.

    Args:
        path (str): 경로

    Returns:
        str: 경로
    """
    if not os.path.isdir(path):
        os.makedirs(path)

    return InfxPath(path)
```

---

## Docstring

- Google 스타일 docstring을 사용한다.
- 파라미터 설명은 다음 줄에서 4칸 들여쓰기로 작성한다.
- 파라미터가 2개 이상일 경우, 각 파라미터 사이에 빈 줄을 넣는다.
- 반환 타입이 복잡한 경우 (dict 등) 각 키와 값을 상세히 문서화한다.
- Python 3.7 호환을 위해 타입 힌트는 docstring 내에 문자열로 작성한다.

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
            Job을 찾지 못한 경우 None을 반환한다.
            Job을 찾은 경우 다음 키를 가진 dict를 반환한다.

            - job_id (str): Deadline Job ID
            - name (str): Job 이름
            - status (str): Job 상태 문자열
    """
```

---

## 문장 마침표 규칙

- 서술형 문장은 마침표로 끝낸다.
- 명사형/개조식 문장은 마침표를 생략한다.

```python
# 서술형 (O)
"""샷그리드 런 카운터 페이지에 데이터를 업로드한다."""

# 명사형/개조식 (O)
"""런 카운터 데이터 업로드"""
"""플러그인 로드 시 최초 1회 실행"""

# 인라인 주석도 동일 (O)
self._cache = None  # 최초 접근 시 로딩
self._connection = None  # 샷그리드 커넥션을 저장한다.
```

---

## 코드 작성 원칙

### 확인된 것만 사용

- API 응답이나 외부 데이터의 값을 매핑할 때, 확인된 값만 명시한다.
- 확인되지 않은 값은 fallback 처리하여 나중에 추적할 수 있게 한다.

```python
stat_map = {
    1: 'Active',
    2: 'Suspended',
    3: 'Completed',
    4: 'Failed',
    6: 'Pending',
}

stat_code = job.get('Stat', 0)
status = stat_map.get(stat_code, f'Unknown({stat_code})')  # 미확인 값 추적 가능
```

### 데이터 구조 확인

- 외부 API 응답을 사용할 때, 실제 데이터 구조를 확인한 후 코드를 작성한다.
- 추측으로 필드명이나 위치를 결정하지 않는다.

```python
# 실제 데이터 구조 확인 후 작성
completed_chunks = job.get('CompletedChunks', 0)  # 최상위 레벨
total_tasks = props.get('Tasks', 0)  # Props 내부
```

---

## 주석

- 불필요한 주석은 작성하지 않는다.
- 코드만으로 명확하지 않은 경우에만 주석을 추가한다.
- 매핑 테이블 등 값의 의미가 명확하지 않은 경우 간단히 설명한다.
- 필드 리스트의 각 항목에는 인라인 주석으로 설명을 추가한다.

```python
# Stat 값 매핑
stat_map = {
    1: 'Active',
    2: 'Suspended',
    3: 'Completed',
    4: 'Failed',
    6: 'Pending',
}

# 필드 리스트 주석 예시
SG_FIELD_LIST = [
    'project',  # 프로젝트
    'entity',  # 엔티티(에셋 또는 샷)
    'sg_task',  # 태스크
    'code',  # 버전 이름
    'sg_status_list',  # 스테이터스
]
```

---

## Deprecation Warning 패턴

메소드나 함수를 deprecated 처리할 때 `warnings.warn()`을 사용한다.

```python
import warnings


class MyClass:

    @classmethod
    def get_user(cls, user_id: int) -> dict:
        """
        Deprecated: get_sg_user()를 사용하세요.
        """
        warnings.warn(
            'get_user()는 곧 삭제될 예정입니다. get_sg_user()를 사용하세요.',
            DeprecationWarning,
            stacklevel=2,
        )
        return cls.get_sg_user(user_id)

    @classmethod
    def get_sg_user(cls, user_id: int) -> dict:
        """새로운 메소드"""
        # 구현
        pass
```

---

## subprocess 사용 패턴

외부 명령어 실행 시 보안을 위해 `shell=True`를 사용하지 않고, 명령어를 리스트 형태로 전달한다.

```python
import subprocess

# 올바른 사용 (O)
cmd = [
    'ffprobe',
    '-v', 'error',
    '-select_streams', 'v:0',
    '-show_entries', 'stream=nb_read_frames',
    '-print_format', 'json',
    str(filepath),
]
result = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)

# 잘못된 사용 (X) - shell=True는 보안 취약점
cmd = f'ffprobe -v error -select_streams v:0 {filepath}'
result = subprocess.run(cmd, shell=True)  # 위험!
```

---

## dict 키 존재 여부 확인 패턴

딕셔너리에서 키의 존재 여부와 값을 동시에 확인할 때 `.get()` 메소드를 사용한다.

```python
# 간결한 방법 (O)
if sg_task.get('task_assignees'):
    process_assignees(sg_task['task_assignees'])

if sg_task.get('entity'):
    entity_name = sg_task['entity'].get('name')

# 장황한 방법 (X)
if 'task_assignees' in sg_task:
    if sg_task['task_assignees']:
        process_assignees(sg_task['task_assignees'])

if 'entity' in sg_task and sg_task['entity']:
    if 'name' in sg_task['entity']:
        entity_name = sg_task['entity']['name']
```

---

## 중복 로직 헬퍼 메소드 추출 패턴

여러 메소드에서 반복되는 로직은 private 헬퍼 메소드로 추출한다.

```python
class MyClass:

    def _filter_valid_tasks(self, pcode: str, target_entity: dict) -> list:
        """
        유효한 태스크 목록을 필터링하여 반환한다.

        Args:
            pcode (str): 프로젝트 코드

            target_entity (dict): 대상 엔티티

        Returns:
            list: 필터링된 태스크 목록
        """
        result = []
        for sg_task in SGTaskCache(pcode).get_data():
            if not sg_task.get('task_assignees'):
                continue
            if not sg_task.get('entity'):
                continue
            if sg_task['entity'].get('name') != target_entity['name']:
                continue
            result.append(sg_task)
        return result

    def process_next_tasks(self):
        """다음 태스크들을 처리한다."""
        tasks = self._filter_valid_tasks(self.pcode, self.entity)
        for task in tasks:
            # 처리 로직
            pass

    def notify_team_leaders(self):
        """팀장들에게 알림을 보낸다."""
        tasks = self._filter_valid_tasks(self.pcode, self.entity)
        for task in tasks:
            # 알림 로직
            pass
```

---

## 지연 Import 패턴 (Lazy Import)

순환 참조를 방지하기 위해 함수 내부에서 import한다.

```python
class InfxUser:

    @classmethod
    def get_all_sg_users(cls) -> list:
        """모든 샷그리드 사용자 데이터를 반환한다."""
        # 순환 참조 방지를 위해 함수 내부에서 import
        from flova.shotgrid.cache import SGUserCache
        return SGUserCache().get_data()
```

---

## 코드 검증 (LSP 기반)

코드를 작성하거나 수정한 후에는 **LSP(Language Server Protocol) 기반 정적 분석 검증을 반드시 수행해야 한다.**
검증을 생략하면 타입 오류, 미정의 변수, 잘못된 참조 등이 런타임에서야 발견된다.

---

### 필수 설치 도구

검증 도구는 **시스템 인터프리터(System Python)**에 설치한다.
가상환경(venv)이 아닌 시스템 Python에 설치해야 에디터와 AI 에이전트(Claude Code 등)가 LSP 서버를 올바르게 인식한다.

```bash
# 타입 검사기 (필수)
pip install pyright

# 린터 + 포매터 (필수)
pip install ruff

# Python LSP 서버 (에디터 연동 시 선택)
pip install python-lsp-server
```

> ⚠️ **주의**: 시스템 Python이 여러 개(2.x, 3.x)라면 `python3 -m pip install pyright` 형태로 명시적으로 지정한다.

---

### Pyright — 타입 검사

Pyright는 Microsoft가 만든 Python 정적 타입 검사기이다.
LSP 서버로도 동작하며, 에디터에서 실시간으로 타입 오류를 확인할 수 있다.

```bash
# 단일 파일 검사
pyright flova/app/flova_task/main_window.py

# 디렉토리 전체 검사
pyright flova/

# 현재 디렉토리 검사
pyright .
```

검사 후 오류가 없어야 코드가 완성된 것으로 간주한다.

---

### Ruff — 린터 + 포매터

Ruff는 Rust 기반의 고속 Python 린터로, PEP-8 검사와 자동 수정을 지원한다.

```bash
# 린트 검사
ruff check flova/app/flova_task/main_window.py

# 자동 수정 가능한 항목 수정
ruff check --fix flova/

# 포맷 확인
ruff format --check flova/
```

---

### 검증 작업 원칙

- 코드 수정 후 **반드시** `pyright`로 타입 오류를 검사한다.
- 커밋 전에 `ruff check` 린트 오류가 없어야 한다.
- AI 에이전트(Claude Code 등)도 코드 작성·수정 후 LSP 검증을 수행해야 한다.
- `# type: ignore`, `# noqa` 주석으로 오류를 억제하지 않는다. 근본 원인을 수정한다.

```python
# 타입 억제 주석 사용 금지 (X)
result = some_function()  # type: ignore
from flova import something  # noqa: E501

# 근본 원인을 수정한다 (O)
result: Optional[str] = some_function()
```

---

### 에디터 설정 (VS Code 기준)

VS Code에서 Pyright를 LSP로 사용하려면 **Pylance** 확장을 설치한다.
Pylance는 Pyright 기반으로 동작한다.

```json
// .vscode/settings.json
{
    "python.analysis.typeCheckingMode": "basic",
    "python.analysis.pythonPath": "C:/Python39/python.exe",
    "[python]": {
        "editor.defaultFormatter": "charliermarsh.ruff"
    }
}
```

> 💡 **팁**: `typeCheckingMode`는 `off` / `basic` / `standard` / `strict` 단계가 있다.
> 레거시 코드베이스에서는 `basic`부터 시작하여 점진적으로 올리는 것을 권장한다.