# 앱 업데이트 스플래시 스크린 구현 가이드

업데이트 클릭 시 진행 화면을 별도 프로세스로 띄우고, 새 앱 인스턴스가 준비되면 종료하는 구현에서 사용한 기술과 삽질 기록.

---

## 구조 요약

```
[사용자 업데이트 클릭]
  1. tray.py       → update_splash.py를 별도 프로세스로 실행
  2. tray.py       → 배치파일 실행 (git pull → pip install → 앱 재시작)
  3. tray.py       → QApplication.quit() (현재 인스턴스 종료)
  4. update_splash → PID 파일 저장 후 스플래시 표시 + 5분 타임아웃 대기
  5. my_flova.pyw  → [1~7/7 로딩] → tray.show() 완료
  6. my_flova.pyw  → PID 파일 읽기 → os.kill(pid, 9) → PID 파일 삭제
```

---

## 기술 1: 서브프로세스로 독립 GUI 실행

앱이 종료되기 전에 스플래시를 띄워야 하므로, 별도 프로세스로 실행해야 한다.

```python
# tray.py
import subprocess, sys, os

cmd = [sys.executable, splash_script]
flags = subprocess.CREATE_NEW_PROCESS_GROUP | subprocess.DETACHED_PROCESS
subprocess.Popen(cmd, env=env, creationflags=flags)
```

- `CREATE_NEW_PROCESS_GROUP`: 부모 프로세스의 Ctrl+C 시그널을 상속받지 않음
- `DETACHED_PROCESS`: 부모가 종료돼도 자식 프로세스가 계속 살아있음
- 두 플래그 조합이 핵심 — 어느 하나만 쓰면 부모 종료 시 같이 죽을 수 있음

---

## 기술 2: 서브프로세스 PYTHONPATH 명시 전달

### 문제

서브프로세스의 `sys.path[0]`은 실행한 스크립트의 디렉터리로 설정된다.
`python.exe C:\...\my_flova\update_splash.py`를 실행하면:
- `sys.path[0]` = `C:\...\my_flova\` (스크립트 디렉터리)
- 부모 프로세스의 `sys.path` 변경은 **상속되지 않음**
- 배치파일이 설정한 `PYTHONPATH`만 상속됨

따라서 `import my_flova`가 `ModuleNotFoundError`로 실패한다.

### 더 나쁜 점

`pythonw.exe`는 stdout/stderr가 없어서 **에러가 완전히 무음 처리**된다.
창도 안 뜨고, 로그도 없고, 아무 반응 없이 프로세스가 죽는다.

### 해결

```python
project_root = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
env = os.environ.copy()
existing = env.get('PYTHONPATH', '')
env['PYTHONPATH'] = project_root + (os.pathsep + existing if existing else '')
subprocess.Popen(cmd, env=env, creationflags=flags)
```

`os.environ.copy()`로 현재 환경변수를 복사한 뒤, 프로젝트 루트를 앞에 붙여서 넘긴다.
기존 `PYTHONPATH`는 보존해야 `flova` 등 다른 패키지도 계속 import 가능하다.

---

## 기술 3: PID 파일로 프로세스 간 종료 제어

두 독립 프로세스 사이에서 "스플래시를 닫아라"는 신호를 보내는 방법으로 PID 파일을 사용했다.

```python
# update_splash.py — 시작 시 자신의 PID 저장
with open(UPDATE_SPLASH_PID_FILE, 'w') as f:
    f.write(str(os.getpid()))
```

```python
# my_flova.pyw — 트레이 등록 후 PID 읽어서 강제 종료
with open(UPDATE_SPLASH_PID_FILE, 'r') as f:
    pid = int(f.read().strip())
os.kill(pid, 9)
os.remove(UPDATE_SPLASH_PID_FILE)
```

### PID 파일 위치

`%TEMP%` 환경변수를 사용해 사용자 임시 디렉터리에 저장한다.

```python
UPDATE_SPLASH_PID_FILE = os.path.join(os.environ.get('TEMP', 'C:/Temp'), 'my_flova_update_splash.pid')
```

- 앱이 쓰기 권한을 항상 갖고 있는 위치
- 재부팅 시 자동 정리되는 경우가 많음
- 여러 사용자가 동시 사용해도 사용자별 `%TEMP%`라 충돌 없음

---

## 기술 4: os.kill(pid, 9) — Windows에서 프로세스 강제 종료

```python
try:
    os.kill(pid, 9)
except ProcessLookupError:
    pass  # 이미 없는 프로세스
```

- Windows에서 `os.kill(pid, 9)`는 내부적으로 `TerminateProcess()`를 호출한다
- `signal.SIGTERM`도 Windows에서는 동일하게 동작함
- `ProcessLookupError`: PID가 없는 경우 — 스플래시가 타임아웃으로 이미 종료됐을 때 발생, 무시하면 됨

---

## 기술 5: QSplashScreen 클릭 차단

`QSplashScreen`은 마우스 클릭 시 기본적으로 `hide()`를 호출한다.
업데이트 중 실수로 클릭해도 닫히면 안 되므로 서브클래스로 막는다.

```python
class _UpdateSplash(InfxSplashScreen):
    def mousePressEvent(self, event):
        pass  # 클릭 이벤트 무력화
```

`super().mousePressEvent(event)` 를 호출하지 않으면 이벤트가 부모 클래스로 전달되지 않아 기본 동작이 실행되지 않는다.

---

## 기술 6: app.processEvents() — 이벤트 루프 진입 전 즉시 렌더링

`app.exec_()`가 실행되기 전에는 Qt 이벤트 루프가 돌지 않는다.
`show()`만 호출하면 화면에 실제로 그려지지 않을 수 있어서, `processEvents()`로 강제 렌더링한다.

```python
splash.show()
app.processEvents()   # 이 시점에 화면에 실제로 표시됨
splash.progress(f'{APP_NAME} 업데이트 중...')
app.processEvents()   # 텍스트도 즉시 반영
```

---

## 실패한 접근: Named Event (Win32 API)

처음에는 Windows Named Event로 신호를 주고받으려 했다.

```
# 의도한 흐름
update_splash  : OpenEvent('MyFlova_UpdateSplash_Done') → WaitForSingleObject
my_flova.pyw   : CreateEvent → SetEvent → CloseHandle
```

### 왜 실패했나

**타이밍 문제**: `my_flova.pyw`가 `CreateEvent → SetEvent → CloseHandle`을 순식간에 실행하면,
splash가 `OpenEvent`를 시도하기 전에 이벤트 객체의 참조 카운트가 0이 돼 OS가 삭제해버린다.
이후 splash의 `OpenEvent`는 영원히 실패 → 스플래시가 5분 타임아웃까지 계속 표시됨.

**핸들을 전역으로 유지**하면 이 문제를 우회할 수 있지만, 그렇게 하더라도
"새 앱이 시작된 시점"과 "이벤트를 최초로 SetEvent한 시점"의 타이밍 제어가 어렵다.

**결론**: 프로세스 간 제어에는 PID 파일처럼 파일시스템 기반의 단순한 방법이 타이밍에 무관해서 더 안정적이다.

---

## 타임아웃 안전장치

`os.kill()`이 실패하거나 PID 파일이 없는 예외 상황을 대비해,
스플래시 스크린 내부에 5분 타임아웃을 두어 무한히 떠있는 상황을 방지한다.

```python
elapsed_ms = 0

def _timeout_poll():
    nonlocal elapsed_ms
    elapsed_ms += UPDATE_SPLASH_POLL_MS
    if elapsed_ms >= UPDATE_SPLASH_TIMEOUT_MS:
        _delete_pid_file()
        app.quit()

timer = QTimer()
timer.timeout.connect(_timeout_poll)
timer.start(UPDATE_SPLASH_POLL_MS)
```

---

## 정리: 방법 선택 기준

| 방법 | 장점 | 단점 | 적합한 경우 |
|------|------|------|-------------|
| **PID 파일** | 단순, 타이밍 무관, 항상 작동 | 파일 I/O | 프로세스 수명 제어 |
| Named Event | OS 네이티브, 빠름 | 참조 카운트 타이밍 민감 | 양쪽이 동시에 살아있을 때 |
| Named Pipe / Socket | 양방향 통신 가능 | 구현 복잡 | 지속적인 데이터 교환 |
