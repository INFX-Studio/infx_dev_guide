# 코드 스니펫 모음 📝

자주 사용하는 코드 패턴들을 바로 복사해서 쓸 수 있도록 모았습니다.

---

## 📖 목차

- [로깅](#로깅)
- [Shotgrid](#shotgrid)
- [경로 처리](#경로-처리)
- [파일 작업](#파일-작업)
- [GUI (PySide2)](#gui-pyside2)
- [DCC 통합](#dcc-통합)
- [유틸리티](#유틸리티)
- [데코레이터](#데코레이터)

---

## 로깅

### 기본 로거 설정
```python
from flova.log import get_logger

log = get_logger('my_module')

log.debug('디버그 메시지')
log.info('정보 메시지')
log.warning('경고 메시지')
log.error('에러 메시지')
```

### 예외와 함께 로깅
```python
try:
    result = risky_operation()
except Exception as e:
    log.error(f'작업 실패: {e}', exc_info=True)
    raise
```

---

## Shotgrid

### Shotgrid 연결
```python
from flova.shotgrid import InfxShotgrid

# 일반 스크립트
sg = InfxShotgrid(script='td')

# 공통 스크립트
sg = InfxShotgrid(script='infx_common')
```

### 프로젝트 조회
```python
from flova.shotgrid.cache import SGProjectCache

# 모든 프로젝트 가져오기
for project in SGProjectCache().get_data():
    print(f"프로젝트: {project['name']} - {project['sg_display_name']}")
```

### 샷 조회
```python
from flova.shotgrid.cache import SGShotCache

# 특정 프로젝트의 샷 목록
project_code = 'MY_PROJECT'
for shot in SGShotCache(project_code).get_data():
    print(f"샷: {shot['code']}")
    if shot.get('sg_sequence'):
        print(f"  시퀀스: {shot['sg_sequence']['code']}")
```

### 태스크 조회
```python
from flova.shotgrid.cache import SGTaskCache

# 특정 프로젝트의 태스크 목록
for task in SGTaskCache('MY_PROJECT').get_data():
    if task.get('entity'):
        print(f"태스크: {task['content']} - {task['entity']['name']}")
```

### Shotgrid 엔티티 찾기
```python
from flova.shotgrid import InfxShotgrid

sg = InfxShotgrid(script='td')

# 프로젝트 찾기
filters = [['name', 'is', 'MY_PROJECT']]
fields = ['name', 'code', 'sg_display_name']
projects = sg.find('Project', filters, fields)

# 샷 찾기
filters = [
    ['project.Project.name', 'is', 'MY_PROJECT'],
    ['code', 'is', 'SH010'],
]
fields = ['code', 'sg_sequence', 'sg_status_list']
shots = sg.find('Shot', filters, fields)
```

### Shotgrid 엔티티 업데이트
```python
from flova.shotgrid import InfxShotgrid

sg = InfxShotgrid(script='td')

# 샷 상태 업데이트
sg.update('Shot', shot_id, {'sg_status_list': 'ip'})

# 버전 생성
version_data = {
    'project': {'type': 'Project', 'id': project_id},
    'code': 'version_name_v001',
    'entity': {'type': 'Shot', 'id': shot_id},
    'sg_task': {'type': 'Task', 'id': task_id},
    'sg_status_list': 'rev',
}
new_version = sg.create('Version', version_data)
```

---

## 경로 처리

### 경로 정규화
```python
from flova.path import InfxPath

def normalized(path):
    """
    경로를 정리한다.
    - 슬래시(/)로 통일
    - 중복 슬래시 제거
    - 앞뒤 공백 제거
    """
    if path is None:
        raise Exception('None 타입은 정리할 수 없습니다.')
    
    path = path.strip()
    path = path.replace('\\', '/')
    path = path.replace('//', '/')
    
    return path
```

### 확장자 제외 파일명
```python
import os

def basenameex(filename):
    """경로와 확장자를 제외한 순수 파일명을 반환한다."""
    return os.path.splitext(os.path.basename(filename))[0]

# 사용 예
filepath = 'C:/projects/shot_010/render/beauty_v001.exr'
print(basenameex(filepath))  # 'beauty_v001'
```

### 디렉토리 생성 (없으면 만들기)
```python
import os

def dirs(path):
    """주어진 경로를 반드시 생성하여 반환한다."""
    if not os.path.isdir(path):
        os.makedirs(path)
    return path

# 사용 예
output_dir = dirs('C:/projects/shot_010/output')
```

---

## 파일 작업

### 파일 존재 확인
```python
import os

if os.path.isfile(filepath):
    print(f'파일 존재: {filepath}')
else:
    print(f'파일 없음: {filepath}')
```

### 파일 목록 가져오기
```python
import os

# 특정 확장자 파일만
IMAGE_EXT = ('.jpg', '.png', '.exr')
image_files = [f for f in os.listdir(folder) if f.lower().endswith(IMAGE_EXT)]

# 전체 경로로
image_paths = [os.path.join(folder, f) for f in image_files]
```

### JSON 읽기/쓰기
```python
import json

# 읽기
with open('data.json', 'r', encoding='utf-8') as f:
    data = json.load(f)

# 쓰기
with open('output.json', 'w', encoding='utf-8') as f:
    json.dump(data, f, indent=4, ensure_ascii=False)
```

### subprocess로 외부 명령 실행
```python
import subprocess

# 올바른 방법 (보안)
cmd = [
    'ffprobe',
    '-v', 'error',
    '-select_streams', 'v:0',
    '-show_entries', 'stream=width,height',
    '-print_format', 'json',
    str(filepath),
]
result = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)

if result.returncode == 0:
    output = json.loads(result.stdout)
else:
    print(f'에러: {result.stderr}')
```

---

## GUI (PySide2)

### 기본 윈도우 템플릿
```python
from __future__ import annotations

try:
    from PySide6.QtGui import *
    from PySide6.QtCore import *
    from PySide6.QtWidgets import *
except:
    from PySide2.QtGui import *
    from PySide2.QtCore import *
    from PySide2.QtWidgets import *

from flova.ui.widget import FlovaMainWindow
from flova.log import get_logger

log = get_logger('my_window')


class MyWindow(FlovaMainWindow):

    WINDOW_NAME = 'my_window'

    def __init__(self, *args, **kwargs):
        super().__init__(
            window_name=self.WINDOW_NAME,
            app_title='내 윈도우',
        )
        self._data = None
        
        self.ui()
        self.init()

    def ui(self):
        """UI를 구성한다."""
        super().ui()
        
        # 메인 위젯
        self.main_widget = QWidget()
        self.setCentralWidget(self.main_widget)
        
        # 레이아웃
        self.main_layout = QVBoxLayout(self.main_widget)
        
        # 위젯들 추가
        self.label = QLabel('Hello World')
        self.main_layout.addWidget(self.label)

    def init(self):
        """모든 위젯을 초기화한다."""
        self.label.setText('초기화됨')


if __name__ == '__main__':
    import sys
    app = QApplication(sys.argv)
    window = MyWindow()
    window.show()
    sys.exit(app.exec_())
```

### Signal/Slot 연결
```python
class MyWidget(QWidget):

    # Signal 정의
    item_changed = Signal(dict)
    selection_cleared = Signal()

    def __init__(self, parent=None):
        super().__init__(parent=parent)
        self.ui()

    def ui(self):
        self.button = QPushButton('클릭')
        self.button.clicked.connect(self.on_button_clicked)
        
        self.list_widget = QListWidget()
        self.list_widget.itemClicked.connect(self.on_item_clicked)

    def on_button_clicked(self):
        """버튼 클릭 시"""
        self.selection_cleared.emit()

    def on_item_clicked(self, item):
        """아이템 클릭 시"""
        data = item.data(Qt.UserRole)
        self.item_changed.emit(data)
```

### QListWidget에 데이터 추가
```python
# 간단한 텍스트
self.list_widget.addItem('항목 1')

# 데이터와 함께
item = QListWidgetItem('샷 010')
item.setData(Qt.UserRole, {'id': 123, 'code': 'SH010'})
self.list_widget.addItem(item)

# 아이템 가져오기
current_item = self.list_widget.currentItem()
if current_item:
    data = current_item.data(Qt.UserRole)
```

### QComboBox 사용
```python
# 항목 추가
self.combo.addItem('옵션 1')
self.combo.addItem('옵션 2', userData={'id': 123})

# 현재 선택
current_text = self.combo.currentText()
current_data = self.combo.currentData()

# 프로그래밍 방식으로 선택
self.combo.setCurrentText('옵션 2')
self.combo.setCurrentIndex(1)

# Signal 연결
self.combo.currentIndexChanged.connect(self.on_combo_changed)
```

### 메시지 박스
```python
from PySide2.QtWidgets import QMessageBox

# 정보
QMessageBox.information(self, '알림', '작업이 완료되었습니다.')

# 경고
QMessageBox.warning(self, '경고', '이 작업은 되돌릴 수 없습니다.')

# 에러
QMessageBox.critical(self, '에러', '파일을 찾을 수 없습니다.')

# 확인
reply = QMessageBox.question(
    self,
    '확인',
    '정말 삭제하시겠습니까?',
    QMessageBox.Yes | QMessageBox.No,
    QMessageBox.No,
)
if reply == QMessageBox.Yes:
    # 삭제 로직
    pass
```

### DPI 스케일링
```python
class MyWidget(QWidget):

    def _dpi(self, size):
        """주어진 크기를 화면의 DPI에 맞게 스케일링한다."""
        dpi = QApplication.primaryScreen().physicalDotsPerInch()
        scale = dpi / 96.0
        return int(size * scale)

    def __init__(self, parent=None):
        super().__init__(parent=parent)
        
        # DPI 스케일링 적용
        self.setMinimumWidth(self._dpi(400))
        self.setMinimumHeight(self._dpi(300))
        
        # 아이콘 크기
        icon_size = self._dpi(24)
        self.button.setIconSize(QSize(icon_size, icon_size))
```

---

## DCC 통합

### Maya에서 Flova 모듈 import
```python
# Maya 스크립트 에디터에서
import sys
sys.path.append('C:/dev/infx/flova')  # 개발자
# sys.path.append('W:/inhouse/flova')  # 비개발자

from flova.shotgrid import InfxShotgrid
```

### 현재 DCC 확인
```python
import os

def get_current_dcc():
    """현재 실행 중인 DCC를 반환한다."""
    if 'maya' in sys.executable.lower():
        return 'maya'
    elif 'nuke' in sys.executable.lower():
        return 'nuke'
    elif 'houdini' in sys.executable.lower():
        return 'houdini'
    elif 'katana' in sys.executable.lower():
        return 'katana'
    else:
        return 'standalone'
```

---

## 유틸리티

### 싱글톤 패턴
```python
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        if not hasattr(self, '_initialized'):
            # 초기화 로직
            self._initialized = True
```

### 타이머 컨텍스트 매니저
```python
import time
from datetime import timedelta

class Timer:
    
    def __enter__(self):
        self.start = time.perf_counter()
        return self
    
    def __exit__(self, *args):
        self.end = time.perf_counter()
        elapsed = self.end - self.start
        td = timedelta(seconds=elapsed)
        total_seconds = int(td.total_seconds())
        hours, remainder = divmod(total_seconds, 3600)
        minutes, seconds = divmod(remainder, 60)
        milliseconds = int((elapsed - int(elapsed)) * 100)
        
        print(f'실행시간: {hours:02}:{minutes:02}:{seconds:02}.{milliseconds:02}')

# 사용 예
with Timer():
    heavy_operation()
```

### 딕셔너리 안전 접근
```python
# get() 사용
value = my_dict.get('key', 'default_value')

# 중첩 딕셔너리
entity_name = sg_task.get('entity', {}).get('name')

# 존재 여부와 값 동시 확인
if sg_task.get('task_assignees'):
    process_assignees(sg_task['task_assignees'])
```

### 리스트 필터링
```python
# 조건에 맞는 항목만
active_tasks = [t for t in tasks if t['sg_status_list'] == 'ip']

# None이 아닌 항목만
valid_items = [item for item in items if item is not None]

# 특정 필드 추출
task_names = [t['content'] for t in tasks if t.get('content')]
```

---

## 데코레이터

### 실행 시간 측정
```python
import time
import functools
from datetime import timedelta


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
        
        print(f'[{func.__name__}] 실행시간: {hours:02}:{minutes:02}:{seconds:02}.{milliseconds:02}')
        
        return result
    
    return wrapper


# 사용 예
@elapsed_time
def heavy_task():
    time.sleep(2)
```

### 로딩 화면 데코레이터 (GUI)
```python
import functools


def loading(message_or_func=None):
    """
    로딩 스크린을 표시하는 데코레이터.
    
    사용법:
        @loading('파일 로딩 중...')
        def load_file(self):
            pass
        
        @loading
        def heavy_task(self):
            pass
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(self, *args, **kwargs):
            msg = message_or_func if isinstance(message_or_func, str) else '로딩 중...'
            
            # 로딩 화면 표시 (실제 구현 필요)
            # loading_widget = LoadingWidget(self, msg)
            # loading_widget.show()
            # QApplication.processEvents()
            
            try:
                result = func(self, *args, **kwargs)
            finally:
                # loading_widget.close()
                pass
            
            return result
        return wrapper
    
    if callable(message_or_func):
        return decorator(message_or_func)
    return decorator


# 사용 예
class MyWindow(FlovaMainWindow):
    
    @loading('파일 분석 중...')
    def analyze_file(self, filepath):
        pass
    
    @loading
    def heavy_task(self):
        pass
```

### Deprecation Warning
```python
import warnings
import functools


def deprecated(reason):
    """함수가 deprecated되었음을 표시하는 데코레이터."""
    
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            warnings.warn(
                f'{func.__name__}()는 곧 삭제될 예정입니다. {reason}',
                DeprecationWarning,
                stacklevel=2,
            )
            return func(*args, **kwargs)
        return wrapper
    return decorator


# 사용 예
@deprecated('get_sg_user()를 사용하세요.')
def get_user(user_id):
    return get_sg_user(user_id)
```

---

## 템플릿

### 모듈 템플릿
```python
"""
모듈 설명을 여기에 작성한다.
"""
from __future__ import annotations

# 로컬 imports
from flova.shotgrid import InfxShotgrid
from flova.log import get_logger

# 서드파티 imports
import fileseq

# 표준 라이브러리 imports
import os
import sys
import json

log = get_logger('my_module')


# 모듈 레벨 상수
DEFAULT_TIMEOUT = 30
IMAGE_EXT = ('.jpg', '.png', '.exr')


class MyClass:
    """클래스 설명."""
    
    def __init__(self):
        """생성자."""
        self._data = None
    
    def process(self):
        """처리 메서드."""
        pass


def my_function(arg1, arg2):
    """
    함수 설명.
    
    Args:
        arg1 (str):
            첫 번째 인자
        
        arg2 (int):
            두 번째 인자
    
    Returns:
        bool: 성공 여부
    """
    return True


if __name__ == '__main__':
    # 테스트 코드
    pass
```
