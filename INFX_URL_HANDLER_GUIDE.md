# Flova URL Handler 가이드

커스텀 프로토콜(`flovac://`, `flovaw://`)을 이용해 특정 Python 스크립트를 실행하는 시스템.
주로 매터모스트 등 메신저에 클릭 가능한 링크를 삽입할 때 사용한다.

## 프로토콜

| 프로토콜 | 설명 |
|----------|------|
| `flovac://` | 콘솔 모드 실행 |
| `flovaw://` | 윈도우 모드 실행 |

## URL 구조

```
{protocol}://{plugin}/{data_file}
```

예시:
```
flovac://open_folder/a1b2c3d4-e5f6-7890-abcd-ef1234567890.json
```

- `flovac://` - 프로토콜
- `open_folder` - 실행할 플러그인 이름
- `a1b2c3d4-...json` - 데이터 파일 (자동 생성됨)

## 사용 가능한 플러그인

### open_folder
폴더를 탐색기로 연다.

```python
from flova.handler.url_handler import FlovaUrlHandler

url = FlovaUrlHandler.create_url(
    plugin='open_folder',
    data={'path': 'W:/project/shot/abc010'}
)
# 결과: flovac://open_folder/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.json
```

### open_file
파일을 기본 프로그램으로 연다.

```python
url = FlovaUrlHandler.create_url(
    plugin='open_file',
    data={'path': 'W:/project/document.pdf'}
)
```

### play_rv
RV 플레이어로 영상/시퀀스를 재생한다.

```python
url = FlovaUrlHandler.create_url(
    plugin='play_rv',
    data={'path': 'W:/project/shot/abc010/comp/render.%04d.exr'}
)
```

## 매터모스트 메시지에 링크 삽입

### 기본 마크다운 링크

```python
from flova.handler.url_handler import FlovaUrlHandler

# URL 생성
folder_url = FlovaUrlHandler.create_url(
    plugin='open_folder',
    data={'path': 'W:/project/shot/abc010'}
)

# 마크다운 링크 형식
message = f'작업 폴더: [폴더 열기]({folder_url})'
```

### 여러 링크가 포함된 메시지 예시

```python
from flova.handler.url_handler import FlovaUrlHandler

shot_path = 'W:/project/shot/abc010'
mov_path = 'W:/project/shot/abc010/output/abc010_comp_v001.mov'

# URL 생성
folder_url = FlovaUrlHandler.create_url(
    plugin='open_folder',
    data={'path': shot_path}
)

play_url = FlovaUrlHandler.create_url(
    plugin='play_rv',
    data={'path': mov_path}
)

# 메시지 작성
message = f'''
### abc010 컴프 완료

| 항목 | 링크 |
|------|------|
| 폴더 | [열기]({folder_url}) |
| 영상 | [RV 재생]({play_url}) |
'''
```

### 고정 파일명 사용 (중복 방지)

동일한 링크를 여러 번 생성할 경우, `filename` 파라미터로 고정 파일명을 지정하면 데이터 파일이 덮어쓰기 된다.

```python
# 고정 파일명 사용 - 같은 이름이면 덮어쓰기
url = FlovaUrlHandler.create_url(
    plugin='open_folder',
    data={'path': shot_path},
    filename='shot_abc010_folder'  # .json 자동 추가됨
)
```

## 새 플러그인 만들기

`flova/handler/plugins/` 폴더에 새 플러그인 파일 생성:

```python
# flova/handler/plugins/my_plugin.py

from flova.handler.url_handler import FlovaUrlHandlerPlugin


class MyPlugin(FlovaUrlHandlerPlugin):

    def __init__(self, data, log):
        super().__init__(data, log)
        self.log = log

    def execute(self):
        # 필수 키 확인
        value = self.check_required_key('my_key')
        
        # 플러그인 로직 구현
        self.log.info(f'처리 중: {value}')


def execute(data, log):
    _plugin = MyPlugin(data, log)
    _plugin.execute()
```

사용:
```python
url = FlovaUrlHandler.create_url(
    plugin='my_plugin',
    data={'my_key': 'some_value'}
)
```

## 주의사항

1. **레지스트리 등록 필요**: `flovac://`, `flovaw://` 프로토콜이 시스템에 등록되어 있어야 URL 클릭 시 작동함
2. **데이터 파일 경로**: `W:/inhouse/flova/flova/handler/data/`에 JSON 파일 생성됨
3. **네트워크 접근**: 데이터 파일이 네트워크 드라이브에 저장되므로, 접근 권한 필요
