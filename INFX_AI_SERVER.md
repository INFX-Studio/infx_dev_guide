# INFX AI SERVER

## 접속
| 구분 | 내용 |
| --- | --- |
| IP | `192.168.10.17` |
| Domain | `ai.infx.kr` |
| Hostname | `INAIINC01` |
| NTP | `DC01`, `DC02` |
| Account | `root` |
| Password | `Infx$123!@#$` <세팅 끝나면 변경> |

## 관리자 권한 획득 방법
```bash
su
Password: {Password}
```

## UFW
- enable 
- 3000/tcp  ALLOW 192.168.20.0/24
- 3000/tcp  ALLOW 192.168.30.0/24
- 3000/tcp  ALLOW 192.168.40.0/24
- 8000/tcp  ALLOW 192.168.20.0/24
- 8000/tcp  ALLOW 192.168.30.0/24
- 8000/tcp  ALLOW 192.168.40.0/24
- 123/udp  ALLOW 192.168.10.10
- 123/udp  ALLOW 192.168.10.11 
- 22389/tcp ALLOW 192.168.10.23 
- 22389/tcp ALLOW 192.168.40.103 

## Mount
| 경로 | 용량 |
| --- | --- |
| `/` | 1 TB |
| `/data` | 2 TB | 
| `/var` | 600 GB |

## 접속방법 
- remote.infx.kr → AI-TMP

## 도메인 접근불가
- `192.168.10.25` 게이트웨이 서버의 nginx 설정 등의 부담스러움으로 그냥 IP로 접근하는 방식으로 변경
- IP로 접속하니 잘 처리됨

---

# AI 서버 성공 리포트

**INFX AI Server — 운용/배포 보고서 (1/2)**

**호스트:** inaiinc01 (192.168.10.17) | **날짜:** 2026-04-17 | **상태:** ✅ 정상 가동

---

## 1. 서버 하드웨어 사양

- **CPU:** AMD Ryzen 9 7950X (16코어/32스레드, 최대 5.88 GHz)
- **RAM:** 124 GiB DDR5
- **GPU:** NVIDIA GeForce RTX 3090 24 GiB (드라이버 580.126.09)
- **디스크:** 시스템 1TB NVMe (4% 사용), 데이터 2TB NVMe (2% 사용)
- **OS:** Ubuntu 22.04.5 LTS, 커널 5.15.0-174
- **Docker:** 29.1.3 | **Python:** 3.10 (시스템), 3.11 (venv)

---

## 2. Gemma 4 모델 상세

| 항목 | 값 |
|------|-----|
| 모델명 | gemma-4-26B-A4B-it-AWQ-4bit |
| 타입 | 멀티모달 (텍스트 + 비전) |
| 양자화 | INT4 AWQ (compressed-tensors, group_size=32) |
| 디스크 크기 | 17 GB (safetensors 4개 샤드) |
| 텍스트 구조 | 30 레이어, 16 어텐션 헤드, MoE 128 전문가(top-8) |
| 컨텍스트 | 최대 8,192 토큰 (원래 262K, VRAM 제한) |
| 런타임 | bfloat16 |

### RTX 3090에 맞춘 방법
- FP16 원본은 ~52GB 필요 (3090은 24GB)
- **INT4 AWQ 양자화**로 17GB로 축소 (품질 손실 최소)
- **gpu-memory-utilization 0.85** 로 VRAM 90% 활용
- **트레이드오프:** 컨텍스트 262K→8K 제한, 동시 요청 시 KV 캐시 경합

---

## 3. Open WebUI 설정

**서비스:** open-webui.service (systemd) | **포트:** 8080
**접속:** http://192.168.10.17:8080

### 필수 환경변수
- `OPENAI_API_BASE_URL=http://localhost:8000/v1` — vLLM 연결
- `WEBUI_URL=http://192.168.10.17:8080` — 리다이렉트 URL
- `CORS_ALLOW_ORIGIN=http://192.168.10.17:8080;http://192.168.10.17;http://localhost:8080`
- `COOKIE_SECURE=false` — HTTP에서 필수 (true면 로그인 불가)
- `ENABLE_SIGNUP=false` — 가입 제한

### 종속성: Open WebUI → vLLM API (:8000) → GPU

---

## 4. FastAPI 설정

**서비스:** infx-fastapi.service | **포트:** 8081
vLLM 프록시: `/chat/completions` → `http://localhost:8000/v1`
IP 마이그레이션 변경 불필요 (localhost 사용)

**INFX AI Server — 운용/배포 보고서 (2/2)**

---

## 5. 리소스 사용량 (실시간)

| 리소스 | 사용 | 전체 | 비율 |
|--------|------|------|------|
| GPU VRAM | 22,194 MiB | 24,576 MiB | **90.3%** |
| 시스템 RAM | 7.6 GiB | 124 GiB | **6.1%** |
| 시스템 디스크 | 38 GB | 1,007 GB | 4% |
| 데이터 디스크 | 29 GB | 2,000 GB | 2% |

### 프로세스 메모리
| 프로세스 | RAM | GPU |
|----------|-----|-----|
| vLLM EngineCore | 3.5 GB | 22,194 MiB |
| vLLM Python | 2.3 GB | — |
| Open WebUI | 1.1 GB | — |
| FastAPI | 62 MB | — |

### 수용 능력 추정
- **동시 사용자:** ~5-10명 (KV 캐시 한계)
- **분당 요청:** ~10-20 (프롬프트 길이에 따라)
- **병목:** GPU VRAM (90% 점유)
- **여유:** RAM 116GB, CPU 여유 충분

---

## 6. 신규 서버 구축 절차 (요약)

1. **시스템 패키지 설치:** python3.11, docker.io, nvidia-container-toolkit, ufw
2. **NVIDIA 퍼시스턴스 모드:** `nvidia-smi -pm 1`
3. **프로젝트 디렉토리:** `/data/infx_ai_server/models` 생성
4. **모델 다운로드:** AWQ 양자화 모델 배치
5. **Python venv + 패키지:** open-webui, fastapi, uvicorn, httpx
6. **vLLM Docker 실행:** (아래 Docker 설정 참조)
7. **systemd 서비스 배포:** open-webui.service, infx-fastapi.service
8. **방화벽:** 8080/tcp 허용 (필요한 서브넷)
9. **검증:** curl로 각 포트 확인

### 주요 실패 지점
- 양자화 플래그 불일치 (bitsandbytes vs compressed-tensors)
- COOKIE_SECURE=true 로 HTTP 접속 시 로그인 불가
- CORS_ALLOW_ORIGIN에 접속 URL 누락

---

## 7-8. vLLM Docker 설정

```bash
docker run -d --name infx-vllm \
  --gpus all --restart always --runtime nvidia --shm-size 64m \
  -p 8000:8000 -v /data/infx_ai_server/models:/models \
  vllm/vllm-openai:gemma4 \
  --model /models/gemma-4-26B-A4B-it-AWQ-4bit \
  --tensor-parallel-size 1 --gpu-memory-utilization 0.85 \
  --max-model-len 8192 --quantization compressed-tensors \
  --dtype bfloat16 --port 8000 --host 0.0.0.0
```

**Docker 사용 이유:** venv 기반 vLLM은 Gemma 4 아키텍처 미지원 + 양자화 포맷 불일치로 실패. Docker 이미지에 호환 vLLM 버전과 CUDA 런타임이 포함되어 있음.

---

## 9. 전체 환경변수 요약

**vLLM (Docker args):** model, tensor-parallel-size, gpu-memory-utilization, max-model-len, quantization=compressed-tensors, dtype, port, host

**Open WebUI (필수):** OPENAI_API_BASE_URL, OPENAI_API_KEY, WEBUI_URL, DATA_DIR, CORS_ALLOW_ORIGIN, COOKIE_SECURE=false

**Open WebUI (권장):** ENABLE_SIGNUP, COOKIE_SAME_SITE

**FastAPI:** VLLM_BASE_URL (소스코드 내)

**Mattermost (.env):** TOKEN, URL (포트+경로 포함), SCHEME, CHANNEL_ID

---

## 10. 설정 분류

**필수:** vLLM Docker, open-webui.service, 모델 파일, OPENAI_API_BASE_URL, WEBUI_URL, CORS, COOKIE_SECURE, UFW 8080

**권장:** FastAPI 서비스, ENABLE_SIGNUP=false, Restart=always, nvidia-persistenced, 모니터링(netdata)

**선택:** Mattermost 연동, vllm.service (비활성, 레거시)

---

# AI 서버 성공 리포트 (영문)

**INFX AI Server — Operations & Deployment Report (1/2)**

**Host:** inaiinc01 (192.168.10.17) | **Date:** 2026-04-17 | **Status:** ✅ Fully Operational

---

## 1. Server Hardware & System Specs

| Item | Value |
|------|-------|
| CPU | AMD Ryzen 9 7950X (16C/32T, up to 5.88 GHz) |
| RAM | 124 GiB DDR5 |
| GPU | NVIDIA GeForce RTX 3090 24 GiB |
| GPU Driver | 580.126.09 (open kernel module) |
| System Disk | 1 TB NVMe (`/`, 4% used) |
| Data Disk | 2 TB NVMe (`/data`, 2% used) |
| OS | Ubuntu 22.04.5 LTS (Jammy) |
| Kernel | 5.15.0-174-generic |
| Docker | 29.1.3 |
| Python | 3.10.12 (system), 3.11.0rc1 (venv) |
| Uptime | 7 days |

---

## 2. Model Details: Gemma 4 26B A4B-it AWQ 4-bit

| Item | Value |
|------|-------|
| Full name | `gemma-4-26B-A4B-it-AWQ-4bit` |
| Model type | `gemma4` (multimodal: text + vision) |
| Quantization | INT4 AWQ via `compressed-tensors` (group_size=32, symmetric) |
| Quant tool | compressed-tensors v0.14.1 |
| Disk size | 17 GB (4 safetensors shards) |
| Text config | 30 layers, 16 attn heads, 8 KV heads, hidden=2816, MoE 128 experts (top-8) |
| Vision config | 27 layers, 16 heads, hidden=1152, patch_size=16 |
| Max context | 8192 tokens (set via vLLM `--max-model-len`) |
| Native max | 262,144 tokens (limited by GPU memory to 8K) |
| Runtime dtype | bfloat16 |

### How it fits on an RTX 3090

The full Gemma 4 26B model at FP16 would need ~52 GB — over double the RTX 3090's 24 GB. Two key techniques make it fit:

1. **INT4 AWQ quantization**: Reduces model weights from ~52 GB to ~17 GB (4-bit weights, group_size=32). Quality loss is minimal for text generation.
2. **`--gpu-memory-utilization 0.85`**: vLLM allocates 85% of VRAM (≈20.8 GB) for model + KV cache, leaving headroom.

Current GPU usage: **22,194 MiB / 24,576 MiB (90.3%)** — the model fills nearly all VRAM.

### Tradeoffs & constraints

- **Context length capped at 8K** instead of native 262K — KV cache for longer contexts would exceed VRAM.
- **Single-request throughput is optimal**; concurrent requests compete for KV cache space.
- **Vision/multimodal features work** but consume extra VRAM per image, reducing available context.
- **Quantization is lossy** — minor quality degradation vs FP16, acceptable for chat/instruction use.

---

## 3. Open WebUI Configuration

**Service:** `open-webui.service` (systemd, enabled)
**Port:** 8080 (0.0.0.0)
**Access:** `http://192.168.10.17:8080`

### Service file: `/etc/systemd/system/open-webui.service`

```ini
Environment=OPENAI_API_BASE_URL=http://localhost:8000/v1    # points to vLLM
Environment=OPENAI_API_KEY=not-needed
Environment=WEBUI_URL=http://192.168.10.17:8080             # canonical URL
Environment=DATA_DIR=...open_webui/data
Environment=WEBUI_AUTH_COOKIE_SECURE=false                   # required for HTTP
Environment=WEBUI_AUTH_COOKIE_SAME_SITE=lax
Environment=WEBUI_SESSION_COOKIE_SECURE=false                # required for HTTP
Environment=WEBUI_SESSION_COOKIE_SAME_SITE=lax
Environment=CORS_ALLOW_ORIGIN=http://192.168.10.17:8080;http://192.168.10.17;http://localhost:8080
Environment=ENABLE_SIGNUP=false
Environment=GLOBAL_LOG_LEVEL=INFO
ExecStart=.../open-webui serve --host 0.0.0.0 --port 8080
```

### Required variables

| Variable | Required | Purpose |
|----------|----------|---------|
| `OPENAI_API_BASE_URL` | **Yes** | Tells Open WebUI where vLLM is |
| `OPENAI_API_KEY` | **Yes** | vLLM doesn't require auth, but Open WebUI needs a value |
| `WEBUI_URL` | **Yes** | Base URL for redirects and link generation |
| `DATA_DIR` | **Yes** | Where Open WebUI stores its SQLite DB |
| `CORS_ALLOW_ORIGIN` | **Yes** | Allowed browser origins (semicolon-separated) |
| `WEBUI_AUTH_COOKIE_SECURE` | **Yes** | Must be `false` for HTTP, `true` for HTTPS |
| `WEBUI_SESSION_COOKIE_SECURE` | **Yes** | Must be `false` for HTTP, `true` for HTTPS |
| `ENABLE_SIGNUP` | Recommended | Controls public signups |
| `GLOBAL_LOG_LEVEL` | Optional | Logging verbosity |

### Dependency chain

```
Open WebUI (:8080) → vLLM API (:8000/v1) → GPU
```

Open WebUI does **not** depend on FastAPI. It calls vLLM's OpenAI-compatible API directly.

### Direct-IP adjustments made

- Changed `WEBUI_URL` from `https://ai.infx.kr` to `http://192.168.10.17:8080`
- Changed `CORS_ALLOW_ORIGIN` to include `http://192.168.10.17` variants
- Changed `COOKIE_SECURE` to `false` (browsers reject Secure cookies over HTTP)

---

## 4. FastAPI Configuration

**Service:** `infx-fastapi.service` (systemd, enabled)
**Port:** 8081 (0.0.0.0)
**File:** `/data/infx_ai_server/api_server.py`

Simple proxy that forwards `/chat/completions` requests to vLLM at `http://localhost:8000/v1`. Also exposes `/health` and `/` endpoints.

No special environment variables — the vLLM URL is hardcoded as `VLLM_BASE_URL = "http://localhost:8000/v1"` in the script.

No configuration changes needed for IP migration — it uses `localhost` internally.

**INFX AI Server — Operations & Deployment Report (2/2)**

---

## 5. Resource Usage & Capacity

### Current resource usage (live)

| Resource | Used | Total | % |
|----------|------|-------|---|
| GPU VRAM | 22,194 MiB | 24,576 MiB | **90.3%** |
| System RAM | 7.6 GiB | 124 GiB | **6.1%** |
| System disk (/) | 38 GB | 1007 GB | **4%** |
| Data disk (/data) | 29 GB | 2.0 TB | **2%** |
| GPU power | 31 W | 420 W | 7.4% (idle) |

### Process memory footprint

| Process | RSS | GPU |
|---------|-----|-----|
| vLLM EngineCore | 3.5 GB | 22,194 MiB |
| vLLM Python | 2.3 GB | — |
| Open WebUI | 1.1 GB | — |
| FastAPI | 62 MB | — |

### Practical capacity estimate

- **Concurrent users**: ~5-10 simultaneous chat sessions (limited by KV cache in 2.4 GB remaining VRAM)
- **Requests/minute**: ~10-20 (depends on prompt/output length)
- **Max context per request**: 8,192 tokens (hard cap)
- **Bottleneck**: GPU VRAM — the model + KV cache fills 90% of the RTX 3090
- **Headroom**: System has 116 GB RAM free and ample CPU — the only constraint is the single 24 GB GPU

---

## 6. Fresh Server Setup Procedure

### Prerequisites

- Ubuntu 22.04 LTS
- NVIDIA GPU with 24+ GB VRAM
- NVIDIA driver 580+ with CUDA support
- Docker + NVIDIA Container Toolkit
- Python 3.11

### Step-by-step order

#### Step 1: System packages
```bash
apt update && apt install -y python3.11 python3-pip docker.io nvidia-container-toolkit ufw
systemctl enable docker
```

#### Step 2: NVIDIA persistence mode
```bash
nvidia-smi -pm 1
systemctl enable nvidia-persistenced
```

#### Step 3: Create project directory
```bash
mkdir -p /data/infx_ai_server/models
```

#### Step 4: Download model
Place the AWQ-quantized model files in `/data/infx_ai_server/models/gemma-4-26B-A4B-it-AWQ-4bit/`.

#### Step 5: Create Python venv and install packages
```bash
python3.11 -m venv /data/infx_ai_server/venv
source /data/infx_ai_server/venv/bin/activate
pip install open-webui vllm fastapi uvicorn httpx mattermost
```

#### Step 6: Launch vLLM in Docker
```bash
docker run -d \
  --name infx-vllm \
  --gpus all \
  --restart always \
  --shm-size 64m \
  -p 8000:8000 \
  -v /data/infx_ai_server/models:/models \
  vllm/vllm-openai:gemma4 \
  --model /models/gemma-4-26B-A4B-it-AWQ-4bit \
  --tensor-parallel-size 1 \
  --gpu-memory-utilization 0.85 \
  --max-model-len 8192 \
  --quantization compressed-tensors \
  --dtype bfloat16 \
  --port 8000 \
  --host 0.0.0.0
```

Wait for model to load (1-3 minutes). Verify: `curl http://localhost:8000/v1/models`

#### Step 7: Deploy systemd services
Copy the three service files to `/etc/systemd/system/`:
- `open-webui.service`
- `infx-fastapi.service`

```bash
systemctl daemon-reload
systemctl enable open-webui infx-fastapi
systemctl start open-webui infx-fastapi
```

#### Step 8: Configure firewall
```bash
ufw allow from 192.168.10.0/24 to any port 8080 proto tcp  # Open WebUI
ufw allow from 192.168.10.0/24 to any port 8081 proto tcp  # FastAPI
ufw allow from 192.168.20.0/24 to any port 8080 proto tcp  # as needed
ufw allow from 192.168.30.0/24 to any port 8080 proto tcp  # as needed
```

#### Step 9: Verify
```bash
curl http://SERVER_IP:8080/           # Open WebUI (expect 200)
curl http://SERVER_IP:8000/v1/models  # vLLM (expect model list)
curl http://SERVER_IP:8081/health     # FastAPI (expect healthy)
```

### Common failure points

| Problem | Cause | Fix |
|---------|-------|-----|
| vLLM OOM on load | `--gpu-memory-utilization` too high, or model too large | Reduce to 0.80, or use smaller model |
| vLLM won't start | Wrong quantization flag (`bitsandbytes` instead of `compressed-tensors`) | Match flag to model format |
| Open WebUI blank page | `WEBUI_URL` mismatch or `COOKIE_SECURE=true` over HTTP | Set URL to actual access URL, set cookies to `false` |
| CORS errors in browser | Browser origin not in `CORS_ALLOW_ORIGIN` | Add the access URL |
| Model not found | Wrong mount path in Docker | Ensure `-v host_path:/models` matches `--model /models/...` |

---

## 7. vLLM Setup History

### Current method: Docker

vLLM runs in a Docker container using the `vllm/vllm-openai:gemma4` image. This works because:
- The Docker image includes the exact vLLM version compatible with Gemma 4
- CUDA toolkit and dependencies are pre-bundled
- GPU access via `--gpus all` and `--runtime nvidia`

### Previous method (failed): venv-based

The disabled systemd service `vllm.service` shows an earlier attempt:
```
ExecStart=/data/infx_ai_server/venv/bin/vllm serve ... \
    --quantization bitsandbytes --load-format bitsandbytes
```

This failed because:
- **Wrong quantization format**: Used `bitsandbytes` for an AWQ/compressed-tensors model
- **vLLM version mismatch**: The venv-installed vLLM may not have supported Gemma 4's architecture (`gemma4` model type was new)
- **Model path mismatch**: Referenced a different model directory (`gemma-4-26B-A4B-it` vs the actual `gemma-4-26B-A4B-it-AWQ-4bit`)

The Docker image `vllm/vllm-openai:gemma4` solved all three issues — it ships with the correct vLLM version and CUDA runtime for this specific model family.

---

## 8. Docker Configuration for vLLM

```bash
docker run -d \
  --name infx-vllm \
  --gpus all \                      # GPU access (all GPUs)
  --restart always \                # Auto-restart on crash/reboot
  --runtime nvidia \                # NVIDIA Container Toolkit
  --shm-size 64m \                  # Shared memory for IPC
  -p 8000:8000 \                    # Port: host 8000 → container 8000
  -v /data/infx_ai_server/models:/models \  # Volume: model files
  vllm/vllm-openai:gemma4          # Image (34 GB)
```

| Setting | Value | Notes |
|---------|-------|-------|
| Image | `vllm/vllm-openai:gemma4` | Pre-built with Gemma 4 support |
| GPU access | `--gpus all` + `--runtime nvidia` | Required for CUDA |
| Shm size | 64 MB | Default; increase if NCCL errors occur |
| Restart policy | `always` | Survives reboots and crashes |
| Log driver | `json-file` (default) | View with `docker logs infx-vllm` |
| Device requests | All GPUs | Single GPU system |

---

## 9. Environment Variables Reference

### vLLM (Docker — set via docker run command args)

| Arg | Value | Required |
|-----|-------|----------|
| `--model` | `/models/gemma-4-26B-A4B-it-AWQ-4bit` | **Yes** |
| `--tensor-parallel-size` | `1` | **Yes** |
| `--gpu-memory-utilization` | `0.85` | **Yes** |
| `--max-model-len` | `8192` | **Yes** |
| `--quantization` | `compressed-tensors` | **Yes** |
| `--dtype` | `bfloat16` | **Yes** |
| `--port` | `8000` | **Yes** |
| `--host` | `0.0.0.0` | **Yes** |

### Open WebUI (systemd Environment=)

| Variable | Value | Required | Notes |
|----------|-------|----------|-------|
| `OPENAI_API_BASE_URL` | `http://localhost:8000/v1` | **Yes** | vLLM endpoint |
| `OPENAI_API_KEY` | `not-needed` | **Yes** | Placeholder (vLLM has no auth) |
| `WEBUI_URL` | `http://192.168.10.17:8080` | **Yes** | Canonical access URL |
| `DATA_DIR` | `...open_webui/data` | **Yes** | SQLite database location |
| `CORS_ALLOW_ORIGIN` | `http://192.168.10.17:8080;...` | **Yes** | Semicolon-separated |
| `WEBUI_AUTH_COOKIE_SECURE` | `false` | **Yes** | `false` for HTTP, `true` for HTTPS |
| `WEBUI_SESSION_COOKIE_SECURE` | `false` | **Yes** | `false` for HTTP, `true` for HTTPS |
| `WEBUI_AUTH_COOKIE_SAME_SITE` | `lax` | Recommended | |
| `WEBUI_SESSION_COOKIE_SAME_SITE` | `lax` | Recommended | |
| `ENABLE_SIGNUP` | `false` | Optional | Restrict signups |
| `GLOBAL_LOG_LEVEL` | `INFO` | Optional | |

### FastAPI (hardcoded in `api_server.py`)

| Variable | Value | Required |
|----------|-------|----------|
| `VLLM_BASE_URL` | `http://localhost:8000/v1` | **Yes** (in source) |

### Mattermost (`/data/infx_ai_server/.env`)

| Variable | Value | Required |
|----------|-------|----------|
| `TOKEN` | Bot access token | **Yes** |
| `URL` | `chat.infx.kr:8065/api` | **Yes** |
| `SCHEME` | `http` | **Yes** |
| `CHANNEL_ID` | Target channel | **Yes** |
| `TEAM_ID` | Team ID | Optional |

---

## 10. Configuration Summary

### Mandatory for operation

| Item | File/Location | Purpose |
|------|--------------|---------|
| vLLM Docker container | `docker run ...` | Model serving |
| `open-webui.service` | `/etc/systemd/system/` | Web interface |
| Model files | `/data/infx_ai_server/models/` | 17 GB AWQ model |
| `OPENAI_API_BASE_URL` | systemd env | Connects Open WebUI → vLLM |
| `WEBUI_URL` | systemd env | Correct redirect/URL generation |
| `CORS_ALLOW_ORIGIN` | systemd env | Browser API access |
| `COOKIE_SECURE=false` | systemd env | Required for HTTP |
| UFW rules for 8080 | `ufw` | Network access |

### Recommended for production

| Item | File/Location | Purpose |
|------|--------------|---------|
| `infx-fastapi.service` | `/etc/systemd/system/` | Alternative API endpoint |
| `ENABLE_SIGNUP=false` | systemd env | Prevent unauthorized signups |
| `Restart=always` on all services | systemd | Auto-recovery |
| `nvidia-persistenced` | systemd | Faster GPU init after reboot |
| Monitoring (netdata on :19999) | Already running | System health |

### Optional

| Item | File/Location | Purpose |
|------|--------------|---------|
| `send_md_to_mm.py` | `/data/infx_ai_server/` | Mattermost integration |
| `.env` (Mattermost) | `/data/infx_ai_server/` | Bot credentials |
| `vllm.service` | `/etc/systemd/system/` | Disabled — legacy venv-based vLLM |

---

## INFX AI Server 장애 조치 보고 - 대화 응답 없는 상태

### 현상
- Open Web UI에서 사용자가 `안녕?` 같은 짧은 메시지를 보냈을 때 AI 응답이 즉시 화면에 표시되지 않음
- 브라우저에서 F5 새로고침 후에는 이미 생성된 응답이 화면에 표시됨

### 확인된 원인
1. Open Web UI의 `stream_response` 설정이 `false`로 저장되어 있었음
   - 대상: model `Flovi` (`/models/gemma-4-26B-A4B-it-AWQ-4bit`)
   - 대상: 사용자 `gmdirect`
2. 이 설정 때문에 Open Web UI가 vLLM에 `stream=false` 요청을 보내면서 동시에 `stream_options.include_usage=true`를 포함함
3. vLLM은 이 조합을 허용하지 않아 `400 Bad Request`를 반환함
   - 재현 오류: `Stream options can only be defined when stream=True`
4. 추가로 `http://192.168.10.17:8080` direct IP 접속 origin이 Open Web UI Socket.IO CORS 허용 목록에 없어 실시간 화면 갱신 채널이 막힐 수 있었음

### 조치 내용
- Open Web UI DB 백업 생성
  - `/data/infx_ai_server/open_webui_data/webui.db.bak-stream-response-20260428-180147`
- model 기본 설정 변경
  - `stream_response=false -> true`
- 사용자 `gmdirect` 설정 변경
  - `stream_response=false -> true`
- Open Web UI systemd 설정 수정
  - `CORS_ALLOW_ORIGIN`에 아래 origin 추가
  - `http://192.168.10.17:8080`
  - `http://inaiinc01:8080`
- `systemctl daemon-reload` 실행
- `open-webui.service` 재시작

### 검증 결과
| 항목 | 결과 |
|---|---|
| Docker/vLLM | 정상, `/health` 200 |
| Open Web UI local | 정상, `/health` 200 |
| Open Web UI public | 정상, `https://ai.infx.kr/health` 200 |
| FastAPI | 정상, `/health` 200 |
| WebSocket direct IP origin | 정상, `101 Switching Protocols` 확인 |
| 수정 후 관련 오류 로그 | 없음 |

### 현재 상태
- 서비스는 정상 동작 중입니다.
- 사용자는 브라우저에서 Open Web UI 탭을 새로고침한 뒤 다시 채팅하면 됩니다.
- 권장 접속 URL은 `https://ai.infx.kr`입니다.
