## AI 서비스 인프라 구성 보고: Gemma4 + vLLM(Docker) + Open WebUI + FastAPI

### 1. 목적
사내 AI 서비스는 `Gemma4 26B AWQ 4bit` 모델을 `vLLM Docker`로 서빙하고, 사용자는 `Open WebUI`를 통해 접속합니다. `FastAPI`는 별도 API proxy 계층으로 운영됩니다.

### 2. 전체 접속 흐름
```text
사용자 브라우저
  -> https://ai.infx.kr
  -> Gateway/Nginx: 192.168.10.25:443
  -> AI Server(Open WebUI): 192.168.10.17:8080
  -> vLLM Docker(OpenAI API): 127.0.0.1:8000/v1
  -> Gemma4 model: /models/gemma-4-26B-A4B-it-AWQ-4bit
```

FastAPI API 경로:
```text
Client/API caller
  -> AI Server FastAPI: 192.168.10.17:8081
  -> vLLM Docker: 127.0.0.1:8000/v1
```

### 3. 시스템 및 서버 정보
| 구분 | 값 |
| --- | --- |
| AI 서버 hostname | `inaiinc01` |
| AI 서버 IP | `192.168.10.17/24` |
| Docker bridge | `172.17.0.1/16` |
| Gateway/Nginx IP | `192.168.10.25` |
| 서비스 domain | `ai.infx.kr` |
| DNS 해석 | `ai.infx.kr -> 192.168.10.25` |
| GPU | `NVIDIA GeForce RTX 3090`, VRAM `24GB` |
| NVIDIA Driver | `580.126.09` |

### 4. 서비스 구성
| 서비스 | 실행 방식 | 주소/Port | 상태 | 비고 |
| --- | --- | --- | --- | --- |
| Gateway Nginx | 별도 gateway 서버 | `192.168.10.25:443` | 운영 필요 | SSL termination, reverse proxy |
| Open WebUI | systemd + Python venv | `192.168.10.17:8080` | active | 사용자 UI, version `0.9.2` |
| vLLM | Docker only | `192.168.10.17:8000` | running | container `infx-vllm`, image `vllm/vllm-openai:gemma4` |
| Gemma4 model | Docker volume mount | `/data/infx_ai_server/models:/models` | loaded | `/models/gemma-4-26B-A4B-it-AWQ-4bit` |
| FastAPI | systemd + Python venv | `192.168.10.17:8081` | active | `/chat/completions` proxy |
| Docker | systemd | local daemon | active | vLLM container runtime |

### 5. Port 및 방화벽 요청 사항
| 방향 | Source | Destination | Port | Protocol | 용도 |
| --- | --- | --- | ---: | --- | --- |
| 사용자 -> Gateway | 사내 사용자망 | `ai.infx.kr` / `192.168.10.25` | 443 | TCP/HTTPS | Open WebUI 접속 |
| Gateway -> AI 서버 | `192.168.10.25` | `192.168.10.17` | 8080 | TCP/HTTP | Nginx reverse proxy to Open WebUI |
| 내부 API client -> AI 서버 | 허용된 사내망 | `192.168.10.17` | 8081 | TCP/HTTP | FastAPI proxy 사용 시 |
| AI 서버 내부 | local | `127.0.0.1` | 8000 | TCP/HTTP | Open WebUI/FastAPI -> vLLM |
| AI 서버 -> Docker container | Docker bridge | container `infx-vllm` | 8000 | TCP/HTTP | vLLM OpenAI-compatible API |

보안상 `8000(vLLM)`은 일반 사용자망에 직접 공개하지 않고 AI 서버 내부 또는 제한된 관리망에서만 접근하는 것을 권장합니다. 일반 사용자는 `https://ai.infx.kr`만 사용하도록 안내합니다.

### 6. Gateway Nginx 필수 설정
`ai.infx.kr` gateway는 `https://ai.infx.kr` 요청을 AI 서버 `http://192.168.10.17:8080`으로 proxy해야 합니다.

필수 항목:
- SSL: `443/tcp`
- `server_name ai.infx.kr`
- `proxy_pass http://192.168.10.17:8080`
- WebSocket 지원: `Upgrade`, `Connection` header 전달
- Streaming 지원: `proxy_buffering off`, `proxy_cache off`
- LLM 응답 대기용 timeout: `proxy_read_timeout 300s`, `proxy_send_timeout 300s`
- Upload 여유: `client_max_body_size 100M`

### 7. Open WebUI 설정
```text
OPENAI_API_BASE_URL=http://localhost:8000/v1
OPENAI_API_KEY=not-needed
ENABLE_OLLAMA_API=false
WEBUI_URL=https://ai.infx.kr
DATA_DIR=/data/infx_ai_server/open_webui_data
CORS_ALLOW_ORIGIN=https://ai.infx.kr;http://ai.infx.kr;http://localhost:8080
ENABLE_SIGNUP=false
```

Open WebUI는 vLLM을 OpenAI-compatible API로 직접 호출합니다. Ollama는 사용하지 않으므로 `ENABLE_OLLAMA_API=false`로 비활성화되어 있습니다.

### 8. vLLM 운영 정책
vLLM은 이 서버에서 **Docker 전용**입니다. Python venv에는 vLLM을 설치하거나 실행하지 않습니다.

Docker container:
```text
name: infx-vllm
image: vllm/vllm-openai:gemma4
restart policy: always
port: 8000:8000
volume: /data/infx_ai_server/models:/models
gpu: --gpus all
```

실행 argument:
```text
--model /models/gemma-4-26B-A4B-it-AWQ-4bit
--tensor-parallel-size 1
--gpu-memory-utilization 0.85
--max-model-len 8192
--quantization compressed-tensors
--dtype bfloat16
--port 8000
--host 0.0.0.0
```

### 9. 정상 동작 확인 명령
```bash
systemctl status open-webui.service infx-fastapi.service docker.service --no-pager
docker ps --filter name=infx-vllm
curl http://127.0.0.1:8080/health
curl http://127.0.0.1:8080/_app/version.json
curl http://127.0.0.1:8081/health
curl http://127.0.0.1:8000/v1/models
curl -k https://ai.infx.kr/health
```

Chat completion 확인:
```bash
curl -sS -H 'Content-Type: application/json' \
  -d '{"messages":[{"role":"user","content":"Reply with only: ok"}],"max_tokens":8,"temperature":0,"stream":false}' \
  http://127.0.0.1:8000/v1/chat/completions
```

### 10. 현재 검증 결과
| 항목 | 결과 |
| --- | --- |
| Open WebUI | 정상, `0.9.2`, health OK |
| FastAPI | 정상, `/health` healthy |
| Docker | 정상, active |
| vLLM container | 정상, `infx-vllm` running |
| Gemma4 model | 정상, `/models/gemma-4-26B-A4B-it-AWQ-4bit` loaded |
| Domain | 정상, `https://ai.infx.kr/health` HTTP 200 |
| Python venv dependencies | 정상, `pip check` no broken requirements |
| GPU | RTX 3090, VRAM 사용 약 `22208/24576 MiB` |

### 11. 인프라팀 요청 사항
1. `ai.infx.kr` DNS가 gateway `192.168.10.25`를 계속 가리키도록 유지해 주십시오.
2. Gateway `443/tcp` HTTPS 접근을 사용자망에서 허용해 주십시오.
3. Gateway `192.168.10.25`에서 AI 서버 `192.168.10.17:8080/tcp` 접근을 허용해 주십시오.
4. Open WebUI streaming/WebSocket을 위해 Nginx proxy header와 buffering 설정을 유지해 주십시오.
5. FastAPI를 외부 시스템에서 사용할 경우, 허용된 source에서 `192.168.10.17:8081/tcp` 접근을 별도 허용해 주십시오.
6. vLLM `8000/tcp`는 일반 사용자망에 직접 공개하지 말고, AI 서버 내부 또는 제한된 관리망으로만 제한해 주십시오.
7. 서버 reboot 후에도 `docker.service`, `open-webui.service`, `infx-fastapi.service`, Docker container `infx-vllm`이 자동 복구되는지 monitoring해 주십시오.

### 12. 운영 담당 참고
- 사용자 접속 URL: `https://ai.infx.kr`
- Open WebUI data: `/data/infx_ai_server/open_webui_data`
- Open WebUI backup: `/data/infx_ai_server/backups/open-webui-20260428-110146.tar.gz`
- vLLM 시작 script: `/data/infx_ai_server/start_vllm.sh`
- vLLM test script: `/data/infx_ai_server/test_docker_vllm.sh`
- vLLM은 Docker 전용이며 Python venv rollback 또는 venv 실행 계획은 없습니다.