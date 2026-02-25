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

## DPI 스케일링 패턴

고해상도 디스플레이를 지원하기 위해 `_dpi()` 메서드를 사용한다.

```python
class _FlovaDpiMixin:

    def _dpi(self, size):
        """주어진 크기를 화면의 DPI에 맞게 스케일링한다."""
        dpi = QApplication.primaryScreen().physicalDotsPerInch()
        scale = dpi / 96.0
        return int(size * scale)


class MyWidget(_FlovaDpiMixin, QWidget):

    def __init__(self, parent=None):
        super().__init__(parent=parent)

        # DPI에 맞게 크기 조절
        self.setMinimumWidth(self._dpi(400))
        self.setMinimumHeight(self._dpi(300))

        # 아이콘 크기도 DPI 스케일링
        icon_size = self._dpi(24)
        self.button.setIconSize(QSize(icon_size, icon_size))
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