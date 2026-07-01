**FastAPI 서비스 사용 가이드 (실서버 기준)**

호스트: `192.168.10.17`
서비스: `infx-fastapi.service`
실행 상태: `active (running)`
바인드: `0.0.0.0:8081`
백엔드: `http://localhost:8000/v1` 의 vLLM로 프록시

---

## 1. 현재 이 서버에서 실행 중인 FastAPI 엔드포인트

현재 동작 중인 FastAPI 서비스는 다음입니다.
- **서비스명:** `infx-fastapi.service`
- **systemd 유닛:** `/etc/systemd/system/infx-fastapi.service`
- **Python 엔트리포인트:** `/data/infx_ai_server/api_server.py`
- **리스닝 주소:** `0.0.0.0:8081`

실제 코드 기준으로 현재 호출 가능한 라우트는 **3개**입니다.

| 메서드 | 경로 | 용도 |
|--------|------|------|
| `GET` | `/` | 기본 서비스 정보 + 엔드포인트 목록 |
| `GET` | `/health` | 상태 확인 |
| `POST` | `/chat/completions` | vLLM chat completions로 프록시 |

---

## 2. 지금 실제로 호출 가능한 API 경로

### A. 루트 정보
```bash
curl http://192.168.10.17:8081/
```

**현재 실제 응답 형식:**
```json
{
  "name": "INFX AI Server",
  "version": "1.0.0",
  "model": "gemma-4-26B-A4B-it-AWQ-4bit",
  "endpoints": [
    {"path": "/health", "method": "GET", "description": "Health check"},
    {"path": "/chat/completions", "method": "POST", "description": "Chat completion"}
  ]
}
```

### B. 헬스 체크
```bash
curl http://192.168.10.17:8081/health
```

**현재 실제 응답 형식:**
```json
{
  "status": "healthy",
  "model": "gemma-4-26B-A4B-it-AWQ-4bit",
  "version": "1.0.0"
}
```

### C. Chat/completions
```bash
curl -X POST http://192.168.10.17:8081/chat/completions \
  -H 'Content-Type: application/json' \
  --data '{
    "messages": [
      {"role": "user", "content": "Say hello in one short sentence."}
    ],
    "temperature": 0.2,
    "max_tokens": 64,
    "stream": false
  }'
```

**현재 실제 응답 형식 (실테스트 결과):**
```json
{
  "id": "chatcmpl-bbff43b7137199b6",
  "object": "chat.completion",
  "created": 1776412392,
  "model": "/models/gemma-4-26B-A4B-it-AWQ-4bit",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello there!"
      },
      "finish_reason": "stop",
      "stop_reason": 106
    }
  ],
  "usage": {
    "prompt_tokens": 20,
    "total_tokens": 24,
    "completion_tokens": 4
  }
}
```

---

## 3. 바로 복사해서 쓸 수 있는 테스트 명령

### 먼저 어떤 경로를 써야 하나?
추천 순서는 다음과 같습니다.
1. **`GET /health` 먼저** — FastAPI 프로세스가 살아있는지 가장 빨리 확인 가능
2. **`POST /chat/completions` 다음** — FastAPI → vLLM 전체 경로 확인 가능
3. **`GET /`** — 엔드포인트 목록과 기본 정보 확인용

### 서버 내부에서 헬스 체크
```bash
curl -i http://127.0.0.1:8081/health
```

### 내부망 다른 PC에서 헬스 체크
```bash
curl -i http://192.168.10.17:8081/health
```

### 서버 내부에서 기본 채팅 요청
```bash
curl -i -X POST http://127.0.0.1:8081/chat/completions \
  -H 'Content-Type: application/json' \
  --data '{
    "messages": [
      {"role": "user", "content": "Write one short greeting."}
    ],
    "temperature": 0.3,
    "max_tokens": 32,
    "stream": false
  }'
```

### 내부망 다른 PC에서 기본 채팅 요청
```bash
curl -i -X POST http://192.168.10.17:8081/chat/completions \
  -H 'Content-Type: application/json' \
  --data '{
    "messages": [
      {"role": "user", "content": "What model are you? Answer in one sentence."}
    ],
    "temperature": 0.2,
    "max_tokens": 64,
    "stream": false
  }'
```

### JSON 보기 좋게 출력
```bash
curl -s -X POST http://192.168.10.17:8081/chat/completions \
  -H 'Content-Type: application/json' \
  --data '{
    "messages": [
      {"role": "user", "content": "List 3 fruits."}
    ],
    "temperature": 0.2,
    "max_tokens": 64,
    "stream": false
  }' | python3 -m json.tool
```

---

## 4. 필요한 헤더 / 인증 / 요청 바디

### 필요한 헤더
실제로 POST에 필요한 헤더는 이것 하나입니다.
```http
Content-Type: application/json
```

### 인증
**현재 인증은 전혀 필요 없습니다.**

실제 코드(`/data/infx_ai_server/api_server.py`)를 보면:
- API 키 검사 없음
- Bearer 토큰 처리 없음
- OAuth/JWT 없음
- `Depends()` 기반 인증 미들웨어 없음

따라서 아래 항목은 보낼 필요가 없습니다.
- `Authorization: Bearer ...`
- API 키
- 쿠키

### `/chat/completions` 요청 바디
현재 요청 모델은 다음과 같습니다.
```json
{
  "messages": [
    {"role": "user", "content": "Your prompt here"}
  ],
  "temperature": 0.7,
  "max_tokens": 1024,
  "stream": false
}
```

필드 설명:
- `messages` = 필수
- `temperature` = 선택, 기본값 `0.7`
- `max_tokens` = 선택, 기본값 `1024`
- `stream` = 선택, 기본값 `false`

---

## 5. 내부망에서 직접 테스트하는 방법

서비스는 `0.0.0.0:8081` 에 바인드되어 있어서 내부망에서 접근 가능합니다.

현재 방화벽 상태:
- `8081/tcp` 는 **`192.168.10.0/24`** 에 대해 허용됨
- 즉, 해당 서브넷에 있는 PC는 직접 테스트 가능

내부망 다른 PC에서 예시:
```bash
curl http://192.168.10.17:8081/health
```

만약 워크스테이션이 `192.168.10.0/24` 대역이 아니면, 프로세스는 떠 있어도 UFW에서 차단될 수 있습니다.

---

## 6. 현재 FastAPI 레이어의 한계 / 주의사항

이 FastAPI 서비스는 **얇은 내부 프록시(thin internal proxy)** 이며, 완전한 API 게이트웨이는 아닙니다.

### 현재 주의사항
- **인증 없음** — 8081 포트에 도달 가능한 누구나 호출 가능
- **라우트 3개뿐** — `/`, `/health`, `/chat/completions`
- **스트리밍 구현 없음** — 요청 모델에 `stream` 필드는 있지만, 프록시는 단순히 `response.json()` 을 반환하므로 자체 SSE/스트리밍 처리는 안 함
- **vLLM 오류 상태 세밀 처리 없음** — Python 예외일 때만 500, 그 외는 JSON을 그대로 반환
- **백엔드 URL 하드코딩** — `VLLM_BASE_URL = "http://localhost:8000/v1"` 가 소스에 직접 박혀 있음
- **내부용 성격이 강함** — Open WebUI는 이미 vLLM과 직접 통신하므로 이 FastAPI가 필수는 아님
- **Pydantic deprecation warning 존재** — `request.dict()` 사용 중이며, 향후 `model_dump()` 로 바꾸는 것이 좋음

### 실무 추천
연결 확인만 하려면:
- 먼저 `GET /health`
- 그 다음 `POST /chat/completions`

커스텀 클라이언트에서 가장 직접적인 모델 API가 필요하면 `:8000/v1` 의 vLLM 쪽이 더 네이티브합니다.
현재 **이 FastAPI 레이어 자체를 테스트**하려면 위 예시처럼 `:8081` 을 사용하면 됩니다.

---

## 7. 요약

### 가장 먼저 해볼 테스트
```bash
curl http://192.168.10.17:8081/health
```

### 가장 좋은 end-to-end 테스트
```bash
curl -X POST http://192.168.10.17:8081/chat/completions \
  -H 'Content-Type: application/json' \
  --data '{
    "messages": [{"role": "user", "content": "Say hello."}],
    "temperature": 0.2,
    "max_tokens": 32,
    "stream": false
  }'
```

### 현재 상태
- FastAPI 서비스: **running**
- 포트: **8081**
- 인증: **없음**
- 내부망 접근: **192.168.10.0/24 허용**
- Chat proxy 테스트: **정상 동작**

---

**FastAPI Service Usage Guide (Live Server State)**

Host: `192.168.10.17`
Service: `infx-fastapi.service`
Runtime: `active (running)`
Bind: `0.0.0.0:8081`
Backend: proxies to vLLM at `http://localhost:8000/v1`

---

## 1. Currently running FastAPI endpoint(s)

The live FastAPI service is:
- **Service name:** `infx-fastapi.service`
- **Systemd unit:** `/etc/systemd/system/infx-fastapi.service`
- **Python entrypoint:** `/data/infx_ai_server/api_server.py`
- **Listen address:** `0.0.0.0:8081`

Live code confirms **3 callable routes**:

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/` | Basic service info + endpoint list |
| `GET` | `/health` | Health/status check |
| `POST` | `/chat/completions` | Thin proxy to vLLM chat completions |

---

## 2. Available API paths you can call right now

### A. Root info
```bash
curl http://192.168.10.17:8081/
```

**Current live response format:**
```json
{
  "name": "INFX AI Server",
  "version": "1.0.0",
  "model": "gemma-4-26B-A4B-it-AWQ-4bit",
  "endpoints": [
    {"path": "/health", "method": "GET", "description": "Health check"},
    {"path": "/chat/completions", "method": "POST", "description": "Chat completion"}
  ]
}
```

### B. Health check
```bash
curl http://192.168.10.17:8081/health
```

**Current live response format:**
```json
{
  "status": "healthy",
  "model": "gemma-4-26B-A4B-it-AWQ-4bit",
  "version": "1.0.0"
}
```

### C. Chat/completions
```bash
curl -X POST http://192.168.10.17:8081/chat/completions \
  -H 'Content-Type: application/json' \
  --data '{
    "messages": [
      {"role": "user", "content": "Say hello in one short sentence."}
    ],
    "temperature": 0.2,
    "max_tokens": 64,
    "stream": false
  }'
```

**Current live response format (actual tested):**
```json
{
  "id": "chatcmpl-bbff43b7137199b6",
  "object": "chat.completion",
  "created": 1776412392,
  "model": "/models/gemma-4-26B-A4B-it-AWQ-4bit",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello there!"
      },
      "finish_reason": "stop",
      "stop_reason": 106
    }
  ],
  "usage": {
    "prompt_tokens": 20,
    "total_tokens": 24,
    "completion_tokens": 4
  }
}
```

---

## 3. Copy-pasteable test commands

### First route to use
Use this order:
1. **`GET /health` first** — quickest way to verify the FastAPI process is alive
2. **`POST /chat/completions` second** — verifies end-to-end FastAPI → vLLM path
3. **`GET /`** if you want a simple route list / metadata

### Health check from the server itself
```bash
curl -i http://127.0.0.1:8081/health
```

### Health check from another machine on the internal network
```bash
curl -i http://192.168.10.17:8081/health
```

### Basic chat request from the server itself
```bash
curl -i -X POST http://127.0.0.1:8081/chat/completions \
  -H 'Content-Type: application/json' \
  --data '{
    "messages": [
      {"role": "user", "content": "Write one short greeting."}
    ],
    "temperature": 0.3,
    "max_tokens": 32,
    "stream": false
  }'
```

### Basic chat request from another internal machine
```bash
curl -i -X POST http://192.168.10.17:8081/chat/completions \
  -H 'Content-Type: application/json' \
  --data '{
    "messages": [
      {"role": "user", "content": "What model are you? Answer in one sentence."}
    ],
    "temperature": 0.2,
    "max_tokens": 64,
    "stream": false
  }'
```

### Pretty-print the JSON response
```bash
curl -s -X POST http://192.168.10.17:8081/chat/completions \
  -H 'Content-Type: application/json' \
  --data '{
    "messages": [
      {"role": "user", "content": "List 3 fruits."}
    ],
    "temperature": 0.2,
    "max_tokens": 64,
    "stream": false
  }' | python3 -m json.tool
```

---

## 4. Required headers / auth / body

### Required headers
Only one header is required in practice for POST:
```http
Content-Type: application/json
```

### Authentication
**No authentication is currently required.**

Live code inspection of `/data/infx_ai_server/api_server.py` shows:
- no API key check
- no Bearer token handling
- no OAuth/JWT
- no `Depends()` auth middleware

So you do **not** need to send:
- `Authorization: Bearer ...`
- API keys
- cookies

### Request body for `/chat/completions`
Current request model:
```json
{
  "messages": [
    {"role": "user", "content": "Your prompt here"}
  ],
  "temperature": 0.7,
  "max_tokens": 1024,
  "stream": false
}
```

Field notes:
- `messages` = required
- `temperature` = optional, default `0.7`
- `max_tokens` = optional, default `1024`
- `stream` = optional, default `false`

---

## 5. Internal-network access

The service listens on `0.0.0.0:8081`, so it is reachable on the LAN.

Current firewall state:
- `8081/tcp` is allowed for **`192.168.10.0/24`**
- This means machines in that subnet can test it directly

Example from another internal machine:
```bash
curl http://192.168.10.17:8081/health
```

If your workstation is **not** in `192.168.10.0/24`, the request may be blocked by UFW even though the process is listening.

---

## 6. Current limitations / caveats of this FastAPI layer

This FastAPI service is a **thin internal proxy**, not a full API gateway.

### Current caveats
- **No authentication** — anyone who can reach port 8081 can call it
- **Only 3 routes exist** — `/`, `/health`, `/chat/completions`
- **No streaming implementation** — the request model accepts `stream`, but the proxy simply returns `response.json()` from vLLM; it does not implement streaming/SSE handling itself
- **No validation of vLLM error status** — it returns `response.json()` directly and only raises 500 on Python exceptions
- **Hardcoded backend URL** — `VLLM_BASE_URL = "http://localhost:8000/v1"` is in source, not config
- **Likely intended for internal use** — Open WebUI already talks directly to vLLM and does not need this FastAPI layer
- **Pydantic deprecation warning exists** — code still uses `request.dict()` under Pydantic v2; works now, but should eventually be changed to `model_dump()`

### Practical recommendation
If you are just testing connectivity:
- start with `GET /health`
- then test `POST /chat/completions`

If you want the simplest direct model API for custom clients, vLLM on `:8000/v1` is the more native endpoint.
If you specifically want to test **this** FastAPI layer, use `:8081` exactly as shown above.

---

## 7. Summary

### Best first test
```bash
curl http://192.168.10.17:8081/health
```

### Best end-to-end test
```bash
curl -X POST http://192.168.10.17:8081/chat/completions \
  -H 'Content-Type: application/json' \
  --data '{
    "messages": [{"role": "user", "content": "Say hello."}],
    "temperature": 0.2,
    "max_tokens": 32,
    "stream": false
  }'
```

### Current status
- FastAPI service: **running**
- Port: **8081**
- Auth: **none**
- Internal subnet access: **allowed for 192.168.10.0/24**
- Chat proxy test: **working**
