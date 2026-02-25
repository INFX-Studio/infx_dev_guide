# ShotGrid 커넥션 가이드

## 개요

inFX 환경에서 ShotGrid API에 연결할 때, 네트워크 환경에 따라 프록시 사용 여부가 달라진다.

- **내부망**: 프록시 서버(`192.168.10.39:23128`)를 통해 연결
- **외부망/오픈망**: 프록시 없이 직접 연결

## 기본 사용법

### InfxShotgridConnection 사용 (권장)

전역 커넥션 관리 클래스를 사용하면 커넥션을 재사용할 수 있어 효율적이다.

```python
from flova.shotgrid import InfxShotgridConnection

# 스크립트 이름으로 커넥션 획득 (이미 생성된 커넥션은 캐싱됨)
sg = InfxShotgridConnection.get_connection('td')

# 일반적인 ShotGrid API 사용
sg_project = sg.find_one('Project', [['name', 'is', 'ABC']])
```

### InfxShotgrid 직접 사용

새로운 커넥션이 필요한 경우 직접 인스턴스를 생성할 수 있다.

```python
from flova.shotgrid import InfxShotgrid

# 스크립트 인증 (권장)
sg = InfxShotgrid(script='td')

# 사용자 인증 (필요 시)
sg = InfxShotgrid(user_id='user@infx.kr', password='password')

# 현재 로그인 사용자로 자동 인증
sg = InfxShotgrid()
```

## 네트워크 판단 로직

`InfxShotgrid` 클래스는 내부적으로 `is_outter_network()` 메서드를 통해 네트워크 환경을 자동 판단한다.

### 동작 방식

1. 프록시 서버를 통해 ShotGrid URL에 실제 접근 시도
2. 접근 성공 → 내부망으로 판단 → 프록시 사용
3. 접근 실패 (연결 불가, 403 Forbidden 등) → 외부망으로 판단 → 직접 연결

### 캐싱

네트워크 판단 결과는 클래스 변수에 캐싱되어 세션 동안 재사용된다. 매번 네트워크 테스트를 수행하지 않으므로 성능에 영향이 없다.

```python
class InfxShotgrid(shotgun_api3.Shotgun):
    # 네트워크 판단 결과 캐싱 (세션 동안 유지)
    _is_outter_network_cache = None
```

## 주요 클래스 및 상수

### InfxShotgrid

| 상수 | 값 | 설명 |
|------|-----|------|
| `URL` | `https://infx.shotgrid.autodesk.com` | ShotGrid 서버 URL |
| `PROXY_IP` | `192.168.10.39` | 프록시 서버 IP |
| `PROXY_PORT` | `23128` | 프록시 서버 포트 |
| `CA_CERTS` | `W:/inhouse/data/sg_cacerts/cacerts.txt` | CA 인증서 경로 |

### InfxShotgridConnection

전역 커넥션 관리 클래스. 스크립트 이름을 키로 하여 커넥션을 캐싱한다.

```python
# 내부 구조
_sg_conn_data = {}  # {'td': <Shotgun>, 'infx_common': <Shotgun>, ...}
```

## 유틸리티 클래스

### ShotgridTask

태스크 이름 변환 유틸리티.

```python
from flova.shotgrid import ShotgridTask

# 긴 이름 → 짧은 이름
ShotgridTask.short_name('modeling')  # 'mod'
ShotgridTask.short_name('composition')  # 'comp'

# 짧은 이름 → 긴 이름
ShotgridTask.long_name('mod')  # 'modeling'
ShotgridTask.long_name('comp', to_korean=True)  # '합성'
```

### ShotgridStatus

스테이터스 코드/이름/색상 관리.

```python
from flova.shotgrid import ShotgridStatus

ShotgridStatus.get_name('wip')  # 'Work in Progress'
ShotgridStatus.get_code('Finished')  # 'fin'
ShotgridStatus.get_color('wip')  # ((0, 0, 0), (115, 191, 67))
```

### ShotgridUtil

기타 유틸리티 함수 모음.

```python
from flova.shotgrid import ShotgridUtil

# ShotGrid 페이지 열기
ShotgridUtil.open_sg_page(sg_entity=sg_shot)
ShotgridUtil.open_sg_page(url='/detail/Shot/12345')

# 작업자와 리뷰어가 같은지 확인
ShotgridUtil.is_same_artist_reviewer(sg_task)

# 월 오프셋 계산 (ShotGrid 필터용)
ShotgridUtil.get_month_offset(2026, 3)  # 현재 1월 기준 → 2
```

---

## 트러블슈팅

### WinError 10013: 소켓 액세스 권한 에러

#### 증상

네트워크 판단 시 다음과 같은 에러 발생:

```
URLError: <urlopen error [WinError 10013] 액세스 권한에 의해 숨겨진 소켓에 액세스를 시도했습니다>
```

내부망임에도 외부망으로 오판되어 ShotGrid 연결 실패.

#### 원인

Windows 방화벽에서 Python의 아웃바운드(나가는) 연결을 차단하고 있음.

#### 해결 방법

**방법 1: GUI로 설정**

1. `Win + R` → `wf.msc` 입력 → 엔터 (고급 방화벽 열림)
2. 왼쪽에서 **아웃바운드 규칙** 클릭
3. 오른쪽에서 **새 규칙** 클릭
4. **프로그램** 선택 → 다음
5. **다음 프로그램 경로** 선택 → `C:\Programs\Python310\python.exe` 입력 → 다음
6. **연결 허용** 선택 → 다음
7. 도메인, 개인, 공용 전부 체크 → 다음
8. 이름: `Python 3.10 Outbound` → 마침

**방법 2: 명령어로 설정 (관리자 권한 CMD)**

```cmd
netsh advfirewall firewall add rule name="Python 3.10 Outbound" dir=out action=allow program="C:\Programs\Python310\python.exe" enable=yes
```

**방법 3: 특정 포트만 허용**

프록시 포트(23128)만 열어주고 싶으면:

```cmd
netsh advfirewall firewall add rule name="Proxy 23128 Outbound" dir=out action=allow protocol=tcp remoteport=23128 enable=yes
```

---

## 히스토리

### 2026-01-27 WinError 10013 소켓 권한 에러 해결

#### 문제점

일부 시스템에서 `is_outter_network()` 메서드 실행 시 소켓 액세스 권한 에러 발생:

```
URLError: <urlopen error [WinError 10013] 액세스 권한에 의해 숨겨진 소켓에 액세스를 시도했습니다>
Error Errno: 13
```

Windows 방화벽에서 Python의 아웃바운드 연결을 차단하여 프록시 접근 테스트 자체가 실패, 내부망임에도 외부망으로 오판됨.

#### 해결 방법

해당 시스템의 Windows 방화벽에서 Python 아웃바운드 규칙 추가:

```cmd
netsh advfirewall firewall add rule name="Python 3.10 Outbound" dir=out action=allow program="C:\Programs\Python310\python.exe" enable=yes
```

코드 수정 없이 시스템 설정으로 해결.

---

### 2026-01-27 SSL 인증서 검증 실패 이슈 수정

#### 문제점

`is_outter_network()` 메서드에서 프록시를 통해 ShotGrid 접근 테스트 시 SSL 인증서 검증 실패 발생:

```
URLError: <urlopen error [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: self signed certificate in certificate chain (_ssl.c:1007)>
```

프록시 서버가 self-signed 인증서를 사용하고 있어서, `urllib.request`가 기본 SSL 검증 과정에서 거부함. 이로 인해 내부망임에도 외부망으로 오판되는 문제 발생.

#### 해결 방법

기존 `shotgun_api3`에서 사용하는 CA 인증서 파일(`CA_CERTS`)을 `urllib.request`에도 적용:

```python
def is_outter_network(self):
    if InfxShotgrid._is_outter_network_cache is not None:
        return InfxShotgrid._is_outter_network_cache

    import urllib.request
    import ssl

    try:
        # 기존 CA 인증서 파일 사용
        ssl_context = ssl.create_default_context(cafile=str(self.CA_CERTS))

        proxy = urllib.request.ProxyHandler({
            'http': f'http://{self.PROXY_IP}:{self.PROXY_PORT}',
            'https': f'http://{self.PROXY_IP}:{self.PROXY_PORT}',
        })
        https_handler = urllib.request.HTTPSHandler(context=ssl_context)
        opener = urllib.request.build_opener(proxy, https_handler)
        opener.open(self.URL, timeout=5)
        InfxShotgrid._is_outter_network_cache = False  # 프록시 통해 접근 성공 = 내부망
    except:
        InfxShotgrid._is_outter_network_cache = True   # 실패 = 외부망

    return InfxShotgrid._is_outter_network_cache
```

#### 변경사항

- `ssl.create_default_context(cafile=str(self.CA_CERTS))`로 CA 인증서 파일 지정
- `HTTPSHandler`에 ssl_context 전달하여 HTTPS 요청 시 인증서 검증 통과
- 타임아웃 3초 → 5초로 증가 (여유 확보)

---

### 2026-01 네트워크 판단 로직 개선

#### 기존 문제점

기존 `is_outter_network()` 메서드는 KT 네임서버(`168.126.63.1:53`) 연결 가능 여부로 외부망을 판단했으나, 다음과 같은 문제가 있었다:

1. 소켓 타임아웃(0.5초)이 너무 짧아 Windows에서 `WSAEWOULDBLOCK`(10035) 에러 발생
2. 연결 실패 시 `None`을 반환하여 내부망으로 오판
3. 네트워크 환경에 따라 불안정한 결과

#### 검토한 해결 방법

**방법 1: 프록시 서버 직접 확인 (단순)**

```python
def is_outter_network(self):
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.settimeout(1.0)
        result = sock.connect_ex((self.PROXY_IP, self.PROXY_PORT))
        return result != 0  # 연결 실패 = 외부망
```

- 장점: 단순하고 빠름
- 단점: 프록시 연결은 되지만 403으로 거부되는 환경에서 오판 가능

**방법 2: 프록시 통해 ShotGrid 실제 접근 테스트 (채택)**

```python
def is_outter_network(self):
    import urllib.request
    try:
        proxy = urllib.request.ProxyHandler({
            'https': f'http://{self.PROXY_IP}:{self.PROXY_PORT}'
        })
        opener = urllib.request.build_opener(proxy)
        opener.open(self.URL, timeout=3)
        return False  # 성공 = 내부망
    except:
        return True   # 실패 = 외부망
```

- 장점: 실제 동작 테스트, 403 거부 환경 대응 가능, 정확한 판단
- 단점: 첫 연결 시 약간의 지연 (timeout 3초)

#### 최종 적용

방법 2를 채택하고, 추가로 결과 캐싱을 구현하여 성능 문제 해결.

```python
class InfxShotgrid(shotgun_api3.Shotgun):
    _is_outter_network_cache = None  # 세션 동안 캐싱

    def is_outter_network(self):
        if InfxShotgrid._is_outter_network_cache is not None:
            return InfxShotgrid._is_outter_network_cache
        # ... 테스트 로직 ...
        InfxShotgrid._is_outter_network_cache = result
        return result
```
