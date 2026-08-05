# inFX PySide2 GUI 스타일 가이드

이 문서는 PySide2를 사용한 GUI 코드 작성 시 따라야 할 스타일 지침을 정리한 것이다.

---

## PySide 호환성 패턴

PySide6과 PySide2를 모두 지원하기 위해 try/except 구문을 사용한다. PySide2를 먼저 시도한다.

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

## 커스텀 위젯 클래스 네이밍

- **Flova 접두사**: 프로젝트 전용 커스텀 위젯 (`FlovaGroupBox`, `FlovaMainWindow`, `FlovaSplitter`)
- **Infx 접두사**: 기본 Qt 위젯의 스타일 오버라이드 (`InfxButton`, `InfxLabel`, `InfxLineEdit`)

```python
# Flova 접두사: 프로젝트 전용 커스텀 위젯
class FlovaGroupBox(_FlovaStyleMixin, QGroupBox):
    pass


class FlovaMainWindow(_FlovaStyleMixin, QMainWindow):
    pass


# Infx 접두사: 기본 위젯 스타일 오버라이드
class InfxButton(_FlovaStyleMixin, QPushButton):

    STYLE = '''
        QPushButton {
            background-color: #3d3d3d;
            border: 1px solid #555555;
            padding: 5px 15px;
        }
        QPushButton:hover {
            background-color: #4d4d4d;
        }
    '''


class InfxLabel(_FlovaStyleMixin, QLabel):
    pass


class InfxLineEdit(_FlovaStyleMixin, QLineEdit):
    pass
```

---

## UI Mixin 클래스 패턴

UI 컴포넌트에서 재사용 가능한 기능은 Mixin 클래스로 분리한다.

```python
class _FlovaStyleMixin:

    STYLE = ''

    def _setup_style(self):
        self.setStyleSheet(self.STYLE)


class _FlovaDpiMixin:

    def _dpi(self, size):
        """주어진 크기를 화면의 DPI에 맞게 스케일링한다."""
        dpi = QApplication.primaryScreen().physicalDotsPerInch()
        scale = dpi / 96.0
        return int(size * scale)


class _FlovaSplitterMixin:

    def _setup_splitter(self):
        self.setHandleWidth(self._dpi(1))
        self.setChildrenCollapsible(False)


# 위젯 클래스에서 Mixin 사용
class FlovaSplitter(_FlovaSplitterMixin, _FlovaDpiMixin, QSplitter):

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._setup_splitter()
```

---

## UI 클래스 구조

UI 클래스는 다음과 같은 구조를 따른다.

1. 클래스 상수 정의
2. `__init__`에서 멤버 변수 초기화 후 `ui()`, `init()` 호출
3. `ui()` 메서드에서 UI 구성
4. `init()` 메서드에서 위젯 초기화

```python
class MyWindow(FlovaMainWindow):

    WINDOW_NAME = 'my_window'  # 클래스 상수

    def __init__(self, *args, **kwargs):
        super().__init__(
            window_name=self.WINDOW_NAME,
            app_title='윈도우 제목',
        )
        # private 멤버는 _ 접두사
        self._last_selected_item = None
        self._current_data = None

        # UI 구성 및 초기화
        self.ui()
        self.init()

    def ui(self):
        """UI를 구성한다."""
        super().ui()

        ####################################################################################################
        # 메인 레이아웃
        ####################################################################################################
        self.main_widget = QWidget()
        # ...

    def init(self):
        """모든 위젯을 초기화한다."""
        self.main_widget.clear()
        # ...
```

---

## 클래스 레벨 STYLE 상수 패턴

위젯의 스타일시트는 클래스 레벨 상수 `STYLE`로 정의한다.

```python
class InfxComboBox(_FlovaStyleMixin, QComboBox):

    STYLE = '''
        QComboBox {
            background-color: #3d3d3d;
            border: 1px solid #555555;
            padding: 3px 5px;
        }
        QComboBox:hover {
            border: 1px solid #777777;
        }
        QComboBox::drop-down {
            border: none;
        }
        QComboBox QAbstractItemView {
            background-color: #2d2d2d;
            selection-background-color: #4a90d9;
        }
    '''

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._setup_style()
```

---

## Qt 시그널/슬롯 패턴

- Signal은 클래스 레벨에서 정의한다.
- 슬롯 메서드는 `on_` 접두사를 사용한다.

```python
class MyWidget(QWidget):

    # 클래스 레벨에서 Signal 정의
    item_changed = Signal(object)
    path_selected = Signal(str)

    def __init__(self, parent=None):
        super().__init__(parent=parent)
        self.ui()

    def ui(self):
        self.button = QPushButton('클릭')
        self.button.clicked.connect(self.on_button_clicked)

        self.combo = QComboBox()
        self.combo.currentIndexChanged.connect(self.on_combo_changed)

    def on_button_clicked(self):
        """버튼을 클릭했을 때 실행되는 메서드"""
        self.item_changed.emit(self.get_current_item())

    def on_combo_changed(self, index):
        """콤보박스 선택이 변경되었을 때 실행되는 메서드"""
        pass
```

---

## Signal 정의 패턴

Signal은 클래스 레벨에서 정의하고, 타입을 명시한다.

```python
class UserTaskSelector(QWidget):

    # Signal 정의 - 클래스 레벨에서 타입과 함께 선언
    task_selected = Signal(dict)  # SG Task 엔티티
    task_changed = Signal(str)    # Task 이름
    selection_cleared = Signal()  # 인자 없음

    def __init__(self, parent=None):
        super().__init__(parent=parent)
        self._current_task = None
        self.ui()

    def ui(self):
        self.task_list = QListWidget()
        self.task_list.itemClicked.connect(self._on_task_clicked)

    def _on_task_clicked(self, item):
        """태스크 항목 클릭 시 Signal 발생"""
        task = item.data(Qt.UserRole)
        self._current_task = task
        self.task_selected.emit(task)
        self.task_changed.emit(task['content'])

    def clear_selection(self):
        """선택 해제"""
        self._current_task = None
        self.task_list.clearSelection()
        self.selection_cleared.emit()
```

---

## DPI 스케일링

### 기본 원칙

DPI 스케일링은 Qt의 `AA_EnableHighDpiScaling`에 맡기고, 수동 DPI 계산은 하지 않는다.

- non-DCC 앱(My Flova 등): `QApplication` 생성 전에 `AA_EnableHighDpiScaling`을 활성화한다.
- DCC 앱(Maya, Nuke 등): DCC가 이미 `QApplication`을 생성하므로 별도 설정 불가. Windows 비트맵 스케일링이 알아서 처리한다.
- 두 환경 모두 `setFixedHeight(N)` 코드가 동일하게 동작한다.

### non-DCC 앱 진입점 설정

`QApplication` 생성 전에 반드시 아래 두 줄을 추가한다.

```python
from PySide2.QtWidgets import QApplication
from PySide2.QtCore import Qt

# 반드시 QApplication 생성 전에 호출
QApplication.setAttribute(Qt.AA_EnableHighDpiScaling, True)
QApplication.setAttribute(Qt.AA_UseHighDpiPixmaps, True)

app = QApplication(sys.argv)
```

⚠️ **주의**: `setAttribute`는 `QApplication(sys.argv)` 호출 이전에 실행해야 한다. 이후에 호출하면 아무 효과 없다.

### 위젯 높이 통일

QLabel, QComboBox, QPushButton 등의 높이가 위젯마다 달라서 수평 배치 시 삐뚤빼뚤해지는 문제를 방지한다.
`setFixedHeight`로 높이를 통일하되, 값은 논리 픽셀(logical pixel)로 지정한다.

- `AA_EnableHighDpiScaling` 활성화 시: Qt가 논리 픽셀을 물리 픽셀로 자동 변환한다.
- DCC 환경: 물리 픽셀 고정이지만, Windows 비트맵 스케일링이 처리한다.

```python
WIDGET_HEIGHT = 25  # 논리 픽셀 기준


class MyWidget(QWidget):

    def __init__(self, parent=None):
        super().__init__(parent=parent)

        self.label = QLabel('이름')
        self.label.setFixedHeight(WIDGET_HEIGHT)

        self.combo = QComboBox()
        self.combo.setFixedHeight(WIDGET_HEIGHT)

        self.button = QPushButton('확인')
        self.button.setFixedHeight(WIDGET_HEIGHT)
```

### QSS에서 크기 단위

스타일시트의 크기 단위는 반드시 `px`만 사용한다. `pt` 단위는 사용하지 않는다.

```python
# 올바른 사용 (O)
STYLE = '''
    QLabel {
        font-size: 12px;
        padding: 4px 8px;
    }
'''

# 잘못된 사용 (X) - pt는 DPI에 따라 크기가 달라진다
STYLE = '''
    QLabel {
        font-size: 9pt;
    }
'''
```

### 수동 DPI 계산 금지

`physicalDotsPerInch()`, `devicePixelRatio()` 등을 직접 읽어서 크기를 계산하지 않는다.
`AA_EnableHighDpiScaling`이 활성화되면 Qt가 자동으로 처리하므로 수동 계산은 이중 스케일링을 유발한다.

```python
# 잘못된 사용 (X) - 이중 스케일링 발생
dpi = QApplication.primaryScreen().physicalDotsPerInch()
scale = dpi / 96.0
self.setFixedHeight(int(25 * scale))

# 올바른 사용 (O) - Qt에 맡긴다
self.setFixedHeight(25)
```

---

## 컨텍스트 메뉴 클래스 패턴

컨텍스트 메뉴는 별도 클래스로 분리하여 관리한다.

```python
class FlovaContextMenu(QMenu):

    def __init__(self, parent=None):
        super().__init__(parent=parent)
        self._actions = {}
        self._setup_menu()

    def _setup_menu(self):
        """메뉴 항목을 구성한다."""
        self._actions['open'] = self.addAction('열기')
        self._actions['open'].triggered.connect(self._on_open)

        self.addSeparator()

        self._actions['delete'] = self.addAction('삭제')
        self._actions['delete'].triggered.connect(self._on_delete)

    def _on_open(self):
        """열기 메뉴 클릭 시 실행"""
        pass

    def _on_delete(self):
        """삭제 메뉴 클릭 시 실행"""
        pass


# 위젯에서 컨텍스트 메뉴 사용
class MyListWidget(QListWidget):

    def __init__(self, parent=None):
        super().__init__(parent=parent)
        self.setContextMenuPolicy(Qt.CustomContextMenu)
        self.customContextMenuRequested.connect(self._show_context_menu)

    def _show_context_menu(self, pos):
        menu = FlovaContextMenu(self)
        menu.exec_(self.mapToGlobal(pos))
```

---

## Message/MessageBox 패턴

메시지 박스는 정적 메서드를 제공하는 유틸리티 클래스로 구현한다.

```python
class Message:

    @staticmethod
    def info(title, message, parent=None):
        """정보 메시지 박스를 표시한다."""
        QMessageBox.information(parent, title, message)

    @staticmethod
    def warning(title, message, parent=None):
        """경고 메시지 박스를 표시한다."""
        QMessageBox.warning(parent, title, message)

    @staticmethod
    def error(title, message, parent=None):
        """에러 메시지 박스를 표시한다."""
        QMessageBox.critical(parent, title, message)

    @staticmethod
    def question(title, message, parent=None):
        """확인 메시지 박스를 표시한다."""
        result = QMessageBox.question(
            parent,
            title,
            message,
            QMessageBox.Yes | QMessageBox.No,
            QMessageBox.No,
        )
        return result == QMessageBox.Yes
```

---

## loading 데코레이터 패턴

UI에서 시간이 오래 걸리는 작업 시 로딩 화면을 표시하는 데코레이터이다.

```python
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
            # 메시지가 문자열이면 사용, 아니면 기본 메시지
            msg = message_or_func if isinstance(message_or_func, str) else '로딩 중...'

            # 로딩 화면 표시
            loading_widget = _LoadingWidget(self, msg)
            loading_widget.show()
            QApplication.processEvents()

            try:
                result = func(self, *args, **kwargs)
            finally:
                loading_widget.close()

            return result
        return wrapper

    # @loading 또는 @loading('message') 둘 다 지원
    if callable(message_or_func):
        return decorator(message_or_func)
    return decorator


# 사용 예시
class MyWindow(FlovaMainWindow):

    @loading('파일 분석 중...')
    def analyze_file(self, filepath):
        """시간이 오래 걸리는 파일 분석 작업"""
        # 분석 로직
        pass

    @loading
    def heavy_task(self):
        """기본 메시지로 로딩 화면 표시"""
        pass
```

---

## 데코레이터 사용

```python
# 로딩 스크린 표시
@loading('파일 분석 중')
def analyze_file(self, filename: str) -> None:
    pass


# 로딩 스크린 (메시지 없이)
@loading
def heavy_task(self):
    pass


# 정적 메서드
@staticmethod
def get_mov_frame_info(filepath):
    pass


# 클래스 메서드
@classmethod
def create_from_path(cls, path: str):
    pass
```

---

## DCC 안에서의 객체 소멸 처리 (중요)

DCC(Houdini, Maya, Nuke) 안에서 동작하는 GUI 코드는 Qt 객체의 **소멸 경로**를 다룰 때 특히 주의해야 합니다. 이 구역에서 발생하는 문제는 파이썬 예외가 아니라 C++ 레벨의 강제종료로 나타나기 때문에, 로그도 트레이스백도 남지 않습니다.

### destroyed 시그널 안에서 그 객체의 연결을 끊지 않는다

가장 위험한 패턴입니다. 실제로 Houdini 강제종료를 일으킨 사례가 있습니다.

`destroyed` 시그널은 `QObject::~QObject()` 실행 **도중**에 발생합니다. 이 시점의 객체는 파생 클래스 소멸자가 이미 끝난 상태이고 소멸 절차가 진행 중입니다. 그 안에서 해당 객체의 연결 목록을 수정하면 정의되지 않은 동작이 됩니다.

```python
# 잘못된 사용 (X) - Houdini 강제종료
def _drop_registration(self, registration):
    window = registration.window
    if window is not None and is_qt_object_valid(window):
        window.destroyed.disconnect(registration.detach)   # ~QObject 실행 중 호출됨
```

```python
# 올바른 사용 (O) - destroyed 연결은 끊지 않는다
def _drop_registration(self, registration):
    # 창의 destroyed 연결은 절대 끊지 않는다.
    # 여기서 끊는 연결의 소유자는 창이 아니라 매니저 자신이라 안전하다.
    self.changed.disconnect(registration.apply)
```

`destroyed` 연결을 정리하고 싶다면, 끊는 대신 **연결을 창당 한 번만 걸고 그대로 두는** 방식을 씁니다. 슬롯이 여러 번 불려도 무해하도록 재진입 가드를 두면 됩니다.

```python
def _watch_window_destroyed(self, window, key):
    if getattr(window, _WINDOW_DESTROY_WATCH_ATTRIBUTE, None) is self:
        return
    # 등록 객체가 아니라 등록 키에 연결한다.
    # 같은 창은 재등록해도 같은 키를 쓰므로 최초 연결 하나로 계속 정리된다.
    window.destroyed.connect(functools.partial(self._drop_key, key))
    setattr(window, _WINDOW_DESTROY_WATCH_ATTRIBUTE, self)
```

### isValid()를 소멸 진행 중인 객체의 가드로 믿지 않는다

`shiboken2.isValid()`는 **검사한 그 순간**만 보장합니다. `destroyed` 슬롯 안에서는 shiboken이 아직 래퍼를 무효화하기 전일 수 있어 `True`를 돌려주며, 그래서 소멸 진행 중인 객체를 걸러내지 못합니다.

```python
# 잘못된 사용 (X) - 이 가드는 소멸 중인 객체를 통과시킨다
if is_qt_object_valid(window):
    window.destroyed.disconnect(slot)
```

`isValid()`는 "이미 파괴가 끝난 객체를 건드리지 않기 위한" 용도로만 씁니다. 소멸이 진행 중인 상황에서는 안전장치가 되지 못합니다.

### id()를 객체 등록 키로 쓰지 않는다

CPython의 `id()`는 메모리 주소이고, 객체가 수거되면 같은 주소가 곧바로 재사용됩니다. 창을 자주 열고 닫는 DCC 환경에서는 **다른 객체가 같은 키를 갖는 상황**이 실제로 발생하며, 잔존 등록이 남아 있으면 엉뚱한 객체를 정리하게 됩니다.

```python
# 잘못된 사용 (X)
self._registrations[id(window)] = registration
```

```python
# 올바른 사용 (O) - 창에 심어 둔 UUID를 키로 쓴다
def _window_key(window):
    key = getattr(window, _WINDOW_KEY_ATTRIBUTE, None)
    if key is None:
        key = uuid.uuid4().hex
        setattr(window, _WINDOW_KEY_ATTRIBUTE, key)
    return key
```

### 파이썬 GC가 아무 때나 돈다는 것을 전제한다

CPython의 가비지 컬렉션은 임의의 시점, 임의의 C++ 스택 깊이에서 실행됩니다. Qt 객체를 대량으로 만들고 버리는 구간(예: 목록 로딩)에서 GC가 돌면 그 자리에서 파괴 연쇄가 시작될 수 있습니다.

- 앱 수명 싱글턴이 창 등록부를 들고 있으면, 잔존 등록은 창보다 오래 살아남아 반드시 이 경로에서 호출됩니다.
- 그러므로 "정상적으로 닫히는 경우"만 가정한 정리 코드는 DCC에서 통하지 않습니다. `closeEvent`를 거치지 않는 파괴 경로(`deleteLater`, `WA_DeleteOnClose`, C++ 부모에 의한 파괴, DCC 종료)를 항상 함께 고려합니다.

### UI 서브모듈을 reload 하지 않는다 (Houdini 강제종료)

DCC 메뉴는 소스 갱신을 반영하려고 흔히 `importlib.reload(패키지)`를 호출합니다. 여기까지는 안전합니다. 패키지 `__init__`만 다시 실행될 뿐, 그 안의 `from 패키지 import 서브모듈`은 `sys.modules`에 있는 **같은 모듈 객체**를 다시 가져오므로 클래스 아이덴티티가 유지됩니다.

위젯, 델리게이트, 다이얼로그를 정의한 **서브모듈까지 reload 하면 이야기가 달라집니다.** Houdini에서 실제로 강제종료가 재현되었고, 해당 호출을 제거하자 해소되었습니다.

```python
# 잘못된 사용 (X) - Houdini 강제종료
# 패키지 __init__ 안에서 서브모듈을 순서대로 reload 한다.
_SUBMODULE_RELOAD_ORDER = (
    '...delegates', '...filters', '...dialogs', '...ui_main_window',
)


def _reload_submodules():
    for module_name in _SUBMODULE_RELOAD_ORDER:
        module = sys.modules.get(module_name)
        if module is not None:
            importlib.reload(module)


_reload_submodules()   # import 시점에 실행 → 메뉴를 누를 때마다 실행된다
```

무엇이 문제인지는 다음과 같습니다.

- reload는 모듈 딕셔너리를 **제자리에서 갈아끼웁니다.** 창이 열려 있는 상태에서 메뉴를 다시 누르면, 살아있는 위젯이 붙들고 있는 클래스와 모듈 전역에서 참조되는 클래스가 서로 다른 객체가 됩니다.
- 그 순간 구 클래스와 모듈 레벨 객체의 참조가 한꺼번에 버려집니다. 앞 절의 "GC는 아무 때나 돈다"가 그대로 적용되어, 임의의 C++ 스택 깊이에서 Qt 객체 파괴 연쇄가 시작될 수 있습니다.
- `isinstance` 검사가 조용히 깨집니다. 구 창의 자식 위젯은 구 클래스의 인스턴스인데, 그 창의 메서드가 참조하는 이름은 신 클래스를 가리킵니다. 크래시 없이 필터 저장 같은 기능만 동작을 멈춥니다.

reload 호출 위치가 패키지 `__init__`이면 특히 위험합니다. 메뉴 스크립트에는 reload가 한 줄도 없어 보이는데 실제로는 매 실행마다 UI 전체가 교체되기 때문입니다.

소스 갱신을 반영해야 한다면 DCC를 재시작하거나, 개발자 전용 플래그로 한정하고 **살아있는 창이 완전히 파괴된 뒤에만** reload 합니다.

#### Maya, Nuke에서 문제가 없어 보이는 이유

같은 reload 코드를 Maya와 Nuke에서 돌려도 크래시가 보고되지 않는 경우가 있습니다. 안전하다는 뜻이 아니라 **재현 조건을 밟지 않았을 가능성**이 큽니다. 확인된 차이는 다음과 같습니다.

- **창 부모가 다릅니다.** Maya는 Maya 메인 윈도우, Houdini는 `hou.qt.mainWindow()`를 부모로 지정하지만, Nuke는 부모 없이 띄우는 구현이 흔합니다. 부모가 없으면 이전 창의 C++ 수명이 DCC에 묶이지 않아, reload로 파이썬 쪽 아이덴티티가 바뀌어도 파괴 시점이 어긋날 여지가 적습니다.
- **메뉴 스크립트의 실행 컨텍스트가 다릅니다.** Houdini의 `scriptItem`은 Qt 메뉴 액션 디스패치 스택 안에서 곧바로 실행됩니다. Maya의 `cmds.menuItem(command=...)`은 Maya 커맨드 엔진을 거칩니다. GC가 도는 시점의 C++ 스택 깊이가 달라집니다.
- **파이썬 버전이 다릅니다.** Maya 2022는 3.7.7, 최신 Houdini는 3.10 이상입니다. 세대별 GC의 수집 임계와 승격 동작이 달라 같은 양의 가비지에도 수집 시점이 달라집니다.

이 중 어느 것이 결정적인지는 실측으로만 가릅니다. **크래시가 보고되지 않은 DCC라도 이 패턴은 그대로 위험**하다고 간주합니다.

### C++가 소유한 자식 객체의 시그널에 connect 하지 않는다 (Houdini 강제종료)

PySide2 5.15.2에는 **C++가 소유한 자식 객체의 시그널에 `connect` 하면 그 파이썬 래퍼가 영구히 남는 버그**가 있습니다. 부모가 파괴되어 C++ 객체가 사라져도 래퍼만 살아남고, `gc.collect()`로도 회수되지 않으며 `disconnect`로도 해소되지 않습니다.

대표 사례는 `QTableWidget.horizontalHeader()`가 돌려주는 기본 `QHeaderView`입니다.

```python
# 잘못된 사용 (X) - 창을 열 때마다 죽은 래퍼가 하나씩 쌓인다
header = self.horizontalHeader()          # C++가 소유한 헤더
header.sectionResized.connect(self.on_header_resized)
header.sectionClicked.connect(self.on_header_clicked)
```

순수 PySide2로 실측한 대조 결과입니다.

| 동작 | 누수 |
| --- | --- |
| `horizontalHeader()` 호출만 | 없음 |
| 헤더를 지역변수에 보관 | 없음 |
| **헤더 시그널에 `connect`** | **사이클당 1개** |
| `connect` 후 `disconnect` | 사이클당 1개 |
| 테이블 자기 시그널에 `connect` | 없음 |
| 파이썬이 소유한 헤더를 `setHorizontalHeader`로 지정 후 연결 | 없음 |

```
PySide2 5.15.2  기본 헤더 3사이클 → +3 / 9사이클 → +9  (정비례)
PySide2 5.15.2  소유 헤더 3사이클 →  0 / 9사이클 →  0
PySide6 6.11.0  기본 헤더 연결해도 → 0                 (PySide2 한정 버그)
```

이 누수는 오픈망(정품 Python 3.10 + PySide2 5.15.2)에서는 조용히 쌓이기만 하지만, **Houdini 20.5의 Python 3.11용 자체 패치 shiboken2에서는 그 죽은 래퍼가 GC에서 파괴되는 순간 `signal 11`로 즉사**합니다. 트레이스백은 남지 않습니다.

#### 올바른 사용 — 파이썬이 소유한 헤더로 교체하고 강참조를 유지한다

```python
# 올바른 사용 (O)
header = QtWidgets.QHeaderView(QtCore.Qt.Horizontal, self)
self.setHorizontalHeader(header)
# 래퍼를 반드시 강참조로 붙들어 둔다. 지역변수로만 두면 안 된다.
self._owned_horizontal_header = header

header.sectionResized.connect(self.on_header_resized)
header.sectionClicked.connect(self.on_header_clicked)
```

두 가지를 반드시 지켜야 합니다.

**1. 래퍼를 인스턴스 속성으로 강참조 유지합니다.** 교체한 헤더를 지역변수로만 들고 있으면 `__init__` 종료 후 래퍼가 GC 대상이 됩니다. 정품 PySide2는 부모 있는 객체의 래퍼가 죽어도 C++ 객체를 지우지 않지만, Houdini의 패치 shiboken2는 이때 C++ 헤더까지 삭제합니다. 그러면 뷰가 매달린 헤더를 참조한 채 다음 갱신(`item.setData` 등)에서 즉사합니다. 즉 같은 특성이 **역방향으로도** 터집니다. 실측으로 확인된 사항입니다.

**2. 기본 헤더와 다른 속성 기본값을 복원합니다.** 새로 만든 `QHeaderView`는 기본값이 다릅니다.

| 속성 | 기본 헤더 | 새로 만든 헤더 |
| --- | --- | --- |
| `sectionsClickable` | `True` | `False` |
| `highlightSections` | `True` | `False` |
| `sectionResizeMode` | `Interactive` | `Fixed` |

이것을 빠뜨리면 테이블의 정렬과 폭 조절이 조용히 동작을 멈춥니다.

#### 판별 방법

증상은 "Houdini 기동 직후 실행하면 거의 100% 죽고, 10초 기다리거나 다른 툴을 먼저 띄우면 확률이 크게 떨어지는" 형태로 나타납니다. 이는 기동 경과 시간 자체가 원인이 아니라, **그 시점의 GC가 한 번에 수거하는 가비지 더미에 죽은 래퍼가 섞일 확률** 문제입니다. 기다리면 평상시 GC가 가비지를 잘게 나눠 수거하므로 확률이 낮아질 뿐입니다.

- `gc.disable()`로 GC를 봉쇄하면 어떤 실행 패턴에서도 죽지 않습니다. 크래시가 GC 경로 안에 있다는 것을 가릅니다.
- `gc.DEBUG_SAVEALL`을 걸고 같은 수거를 돌리면 생존합니다. 크래시가 순회(`tp_traverse`)가 아니라 **수거물 파괴 단계**임을 가릅니다.

이 두 대조가 모두 성립하면 죽은 Qt 래퍼의 `dealloc`을 의심합니다.

### DCC 메인 윈도우를 부모로 지정한다

부모 없는 top-level 창은 Qt 소유권이 파이썬에 남아 예기치 않은 시점에 파괴됩니다. Houdini에서는 실행 직후 강제종료로 이어집니다.

```python
parent = None
if DCCDetector.is_maya():
    from flova.maya.ui import maya_widget
    parent = maya_widget()
elif DCCDetector.is_houdini():
    import hou
    parent = hou.qt.mainWindow()

super().__init__(window_name=..., parent=parent)
```

SideFX 공식 문서도 같은 취지로, 창을 `hou.session` 같은 곳에 보관하거나 메인 윈도우에 부모를 지정해 Houdini가 수명을 관리하게 하라고 안내합니다.

### 창을 다시 열 때는 close()를 먼저 부른다

`deleteLater()`만 부르면 `closeEvent`가 발생하지 않아 등록 해제나 상태 저장 같은 정리가 이루어지지 않습니다.

```python
# 올바른 사용 (O)
if window is not None and is_qt_object_valid(window):
    window.close()          # closeEvent가 발생해 정리가 이루어진다
    window.deleteLater()
    window = None
```

### Qt 객체는 메인 스레드에서만 다룬다

SideFX 공식 문서는 Qt 코드를 반드시 Houdini 메인 스레드에서 실행하라고 명시합니다. PyQt/PySide는 Qt 객체 조작이 스레드 안전하지 않으며, 그중에서도 **삭제가 가장 위험**합니다. 백그라운드 스레드에서는 위젯을 만들지도, 접근하지도, 파괴하지도 않습니다.

### PySide6에서는 소멸 관련 버그 일부가 사라진다

위의 "C++가 소유한 자식 객체의 시그널에 connect" 누수는 PySide6에서 재현되지 않습니다. Houdini 21 이상은 Qt6를 사용하므로 DCC 버전 상향과 함께 이 문제군은 소멸합니다. PySide6 전용 구현(`flova/ui/pyside6/*`)에는 헤더 교체를 적용하지 않아도 됩니다.

다만 나머지 항목(`destroyed` 안에서 disconnect 금지, `isValid()` 신뢰 금지, `id()` 키 금지, 서브모듈 reload 금지, 메인 스레드 제약)은 Qt 자체의 소멸 의미론에서 오는 것이므로 PySide6에서도 그대로 적용합니다.

### 참고

- [HOM cookbook — Qt (SideFX 공식)](https://www.sidefx.com/docs/houdini/hom/cb/qt.html)
- [PyQt와 파이썬 GC의 상호작용 문제](https://www.riverbankcomputing.com/pipermail/pyqt/2011-August/030378.html)
