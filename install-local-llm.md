# Install Local LLM (Docker) & Configure OpenCode

## Host Specs

| Component | Value |
|-----------|-------|
| CPU | AMD EPYC 7352 24-Core (48 threads) |
| RAM | 32 GB (28 GB available) |
| Storage | 938 GB NVMe (861 GB free) |
| GPU | None (ASPEED BMC only) |
| OS | Ubuntu 22.04.5 LTS |

> CPU-only inference. Expect 5-10 tokens/sec on 7B-14B quantized models.
> Docker is NOT currently installed on this host.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│  User                                               │
│  └── Runs `opencode-local [shortcut]`               │
├─────────────────────────────────────────────────────┤
│  opencode-local.sh                                  │
│  ├── Calls ollama-manager.sh ensure                 │
│  └── Prints model ID for OpenCode                   │
├─────────────────────────────────────────────────────┤
│  ollama-manager.sh                                  │
│  ├── start | stop | status | ensure                 │
│  └── Touches /tmp/ollama-last-activity              │
├─────────────────────────────────────────────────────┤
│  systemd --user: ollama-idle-monitor.service        │
│  └── Stops container after 15 min idle              │
├─────────────────────────────────────────────────────┤
│  Docker Container: ollama-local (mem_limit 28G)     │
│  ├── Ollama server (port 11434, OpenAI-compatible)  │
│  └── Loads ONE model at a time                      │
├─────────────────────────────────────────────────────┤
│  Persistent Volume: ollama-models                   │
│  └── 9 model files (~62 GB total)                   │
└─────────────────────────────────────────────────────┘
```

---

## Step 1: Install Docker

Run as a single block, or copy individual lines as needed (each line is independent):

```bash
# Update package index
sudo apt-get update

# Install prerequisites
sudo apt-get install -y ca-certificates curl gnupg

# Set up Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker apt repository (Ubuntu 22.04 jammy, amd64)
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu jammy stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine + Compose plugin
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add current user to docker group (avoids needing sudo for docker commands)
sudo usermod -aG docker $USER

# Activate the new group in current shell (or log out/in)
newgrp docker
```

Verify the install:

```bash
docker --version
docker run hello-world
```

> **Note:** `newgrp docker` only affects the current shell. New terminal sessions will need it again until you log out and log back in.

---

## Step 2: Create Ollama Docker Setup

### 2a. Create project directory

```bash
mkdir -p ~/project/local_cpu_llm/ollama-docker
```

### 2b. Docker Compose file

Create `~/project/local_cpu_llm/ollama-docker/docker-compose.yml`:

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama-local
    ports:
      - "11434:11434"
    volumes:
      - ollama-models:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_NUM_PARALLEL=1
      - OLLAMA_MAX_LOADED_MODELS=1
    mem_limit: 28g
    mem_reservation: 4g
    restart: "no"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:11434/"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s

volumes:
  ollama-models:
    name: ollama-models
```

### 2c. Pull models into the container

```bash
# Start container
docker compose -f ~/project/local_cpu_llm/ollama-docker/docker-compose.yml up -d

# Wait for healthy
sleep 5

# ── Code Models ──────────────────────────────────────────────────────────
# Default: Qwen2.5-Coder 7B (fast, general code)
docker exec ollama-local ollama pull qwen2.5-coder:7b

# Backup: Qwen2.5-Coder 14B (higher quality code)
docker exec ollama-local ollama pull qwen2.5-coder:14b

# C/C++ specialist: CodeLlama 13B (trained on C, C++, infilling)
docker exec ollama-local ollama pull codellama:13b

# Python specialist: StarCoder2 15B (strong Python, JS, TS)
docker exec ollama-local ollama pull starcoder2:15b

# Web programming: DeepSeek-Coder-V2 16B (JS/TS/React/HTML/CSS)
docker exec ollama-local ollama pull deepseek-coder-v2:16b

# ── Testing Model ────────────────────────────────────────────────────────
# DeepSeek-Coder 6.7B Instruct (unit tests, regression tests, test generation)
docker exec ollama-local ollama pull deepseek-coder:6.7b-instruct

# ── Chinese Language Model ───────────────────────────────────────────────
# Qwen2.5 14B (bilingual Chinese/English, strong at both code and prose)
docker exec ollama-local ollama pull qwen2.5:14b

# ── Documentation Model ──────────────────────────────────────────────────
# Llama 3.1 8B (excellent prose, markdown, technical writing)
docker exec ollama-local ollama pull llama3.1:8b

# Phi-3 Medium 14B (structured output, README/API docs)
docker exec ollama-local ollama pull phi3:medium

# Verify all models
docker exec ollama-local ollama list

# Stop container (we'll start it on-demand later)
docker compose -f ~/project/local_cpu_llm/ollama-docker/docker-compose.yml down
```

> **Total disk usage**: ~62 GB for all 9 models. Download time ~30-60 min depending on bandwidth. Only one model is loaded into RAM at a time.

---

## Step 3: On-Demand Start & Auto-Shutdown

### 3a. Ollama manager script

Create `~/project/local_cpu_llm/ollama-docker/ollama-manager.sh`:

```bash
#!/bin/bash
# ollama-manager.sh — Start/stop/status for Ollama Docker container
# Usage: ollama-manager.sh {start|stop|status|ensure}
#
# Idle shutdown is handled by the systemd user service (see Step 4),
# NOT by this script. This script only manages container lifecycle and
# touches /tmp/ollama-last-activity to reset the idle timer.

COMPOSE_FILE="$HOME/project/local_cpu_llm/ollama-docker/docker-compose.yml"
CONTAINER_NAME="ollama-local"
ACTIVITY_FILE="/tmp/ollama-last-activity"

is_running() {
    docker ps --format '{{.Names}}' | grep -q "^${CONTAINER_NAME}$"
}

reset_idle_timer() {
    touch "$ACTIVITY_FILE"
}

start() {
    if is_running; then
        echo "Ollama container already running."
    else
        echo "Starting Ollama container..."
        docker compose -f "$COMPOSE_FILE" up -d
        # Poll for readiness (up to 30 seconds)
        for i in $(seq 1 30); do
            if curl -sf http://localhost:11434/ > /dev/null 2>&1; then
                echo "Ollama is ready."
                break
            fi
            sleep 1
        done
    fi
    reset_idle_timer
}

stop() {
    echo "Stopping Ollama container..."
    docker compose -f "$COMPOSE_FILE" down
}

status() {
    if is_running; then
        echo "running"
        return 0
    else
        echo "stopped"
        return 1
    fi
}

ensure() {
    # Ensure container is running; start if not. Resets idle timer on every call.
    if ! is_running; then
        start
    fi
    reset_idle_timer
}

case "${1:-status}" in
    start)   start ;;
    stop)    stop ;;
    status)  status ;;
    ensure)  ensure ;;
    *)       echo "Usage: $0 {start|stop|status|ensure}" ;;
esac
```

### 3b. Make script executable

```bash
chmod +x ~/project/local_cpu_llm/ollama-docker/ollama-manager.sh
```

---

## Step 4: Systemd Integration (Auto-shutdown on Idle)

Create a systemd user service for the idle monitor so it survives terminal close.

### 4a. Create systemd user service

Create `~/.config/systemd/user/ollama-idle-monitor.service`:

```ini
[Unit]
Description=Ollama Docker Idle Monitor
After=docker.service

[Service]
Type=simple
ExecStart=/bin/bash -c 'IDLE_TIMEOUT=900; IDLE_CHECK=60; COMPOSE_FILE=%h/project/local_cpu_llm/ollama-docker/docker-compose.yml; CONTAINER=ollama-local; while true; do sleep $IDLE_CHECK; if docker ps --format "{{.Names}}" | grep -q "^${CONTAINER}$"; then last=$(stat -c %%Y /tmp/ollama-last-activity 2>/dev/null || echo 0); now=$(date +%%s); idle=$((now - last)); if [ $idle -ge $IDLE_TIMEOUT ]; then docker compose -f "$COMPOSE_FILE" down; fi; fi; done'
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

### 4b. Enable the service

```bash
mkdir -p ~/.config/systemd/user
# (copy the service file above)
systemctl --user daemon-reload
systemctl --user enable ollama-idle-monitor.service
systemctl --user start ollama-idle-monitor.service
```

---

## Step 5: Configure OpenCode

Edit `~/.config/opencode/opencode.jsonc` to add the local Ollama provider.

### 5a. Add provider block

Add inside the `"provider"` section:

```jsonc
    // ── Local Ollama (Docker, on-demand) ───────────────────────────────────
    "local-ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Local Ollama",
      "options": {
        "baseURL": "http://localhost:11434/v1",
        "apiKey": "ollama"
      },
      "models": {
        // ── Code (General) ──
        "qwen2.5-coder-7b": {
          "id": "qwen2.5-coder:7b",
          "name": "Qwen2.5 Coder 7B (Default)",
          "cost": { "input": 0, "output": 0 },
          "limit": { "context": 32768, "output": 8192 }
        },
        "qwen2.5-coder-14b": {
          "id": "qwen2.5-coder:14b",
          "name": "Qwen2.5 Coder 14B (Backup)",
          "cost": { "input": 0, "output": 0 },
          "limit": { "context": 32768, "output": 8192 }
        },
        // ── C/C++ ──
        "codellama-13b": {
          "id": "codellama:13b",
          "name": "CodeLlama 13B (C/C++)",
          "cost": { "input": 0, "output": 0 },
          "limit": { "context": 16384, "output": 4096 }
        },
        // ── Python / Multi-language ──
        "starcoder2-15b": {
          "id": "starcoder2:15b",
          "name": "StarCoder2 15B (Python/JS)",
          "cost": { "input": 0, "output": 0 },
          "limit": { "context": 16384, "output": 8192 }
        },
        // ── Web (JS/TS/React/HTML/CSS) ──
        "deepseek-coder-v2-16b": {
          "id": "deepseek-coder-v2:16b",
          "name": "DeepSeek Coder V2 16B (Web)",
          "cost": { "input": 0, "output": 0 },
          "limit": { "context": 131072, "output": 8192 }
        },
        // ── Testing (Unit/Regression) ──
        "deepseek-coder-6.7b": {
          "id": "deepseek-coder:6.7b-instruct",
          "name": "DeepSeek Coder 6.7B (Testing)",
          "cost": { "input": 0, "output": 0 },
          "limit": { "context": 16384, "output": 8192 }
        },
        // ── Chinese / Bilingual ──
        "qwen2.5-14b": {
          "id": "qwen2.5:14b",
          "name": "Qwen2.5 14B (Chinese/English)",
          "cost": { "input": 0, "output": 0 },
          "limit": { "context": 131072, "output": 8192 }
        },
        // ── Documentation / Prose ──
        "llama3.1-8b": {
          "id": "llama3.1:8b",
          "name": "Llama 3.1 8B (Docs/Prose)",
          "cost": { "input": 0, "output": 0 },
          "limit": { "context": 131072, "output": 8192 }
        },
        "phi3-medium": {
          "id": "phi3:medium",
          "name": "Phi-3 Medium 14B (Docs/Structured)",
          "cost": { "input": 0, "output": 0 },
          "limit": { "context": 131072, "output": 8192 }
        }
      }
    }
```

### 5b. Enable the provider

```jsonc
  "enabled_providers": ["pedro-claude", "pedro-openai", "local-ollama"],
```

### 5c. Set 7B as the default local model

```jsonc
  "small_model": "local-ollama/qwen2.5-coder-7b",
```

> **Default**: 7B (10-15 tok/s, ~6 GB RAM) — use for daily tasks.
> **Backup**: 14B (5-8 tok/s, ~12 GB RAM) — switch manually when higher quality needed.

### 5d. Shell script to start container before using OpenCode with local model

Create `~/project/local_cpu_llm/ollama-docker/opencode-local.sh`:

```bash
#!/bin/bash
# opencode-local.sh — Start Ollama container, launch OpenCode with local model
# Usage: opencode-local.sh [model-shortname]
# Default: 7b

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
MANAGER="$SCRIPT_DIR/ollama-manager.sh"

MODEL="${1:-7b}"
case "$MODEL" in
    7b|default)  OPENCODE_MODEL="local-ollama/qwen2.5-coder-7b" ;;
    14b|backup)  OPENCODE_MODEL="local-ollama/qwen2.5-coder-14b" ;;
    c|cpp)       OPENCODE_MODEL="local-ollama/codellama-13b" ;;
    py|python)   OPENCODE_MODEL="local-ollama/starcoder2-15b" ;;
    web|js)      OPENCODE_MODEL="local-ollama/deepseek-coder-v2-16b" ;;
    test|ut)     OPENCODE_MODEL="local-ollama/deepseek-coder-6.7b" ;;
    cn|chinese)  OPENCODE_MODEL="local-ollama/qwen2.5-14b" ;;
    doc|docs)    OPENCODE_MODEL="local-ollama/llama3.1-8b" ;;
    phi|struct)  OPENCODE_MODEL="local-ollama/phi3-medium" ;;
    list)
        echo "Available models:"
        echo "  7b, default   — Qwen2.5 Coder 7B (general code, fast)"
        echo "  14b, backup   — Qwen2.5 Coder 14B (general code, quality)"
        echo "  c, cpp        — CodeLlama 13B (C/C++ specialist)"
        echo "  py, python    — StarCoder2 15B (Python/JS/TS)"
        echo "  web, js       — DeepSeek Coder V2 16B (Web/React/HTML/CSS)"
        echo "  test, ut      — DeepSeek Coder 6.7B (unit/regression tests)"
        echo "  cn, chinese   — Qwen2.5 14B (Chinese/English bilingual)"
        echo "  doc, docs     — Llama 3.1 8B (documentation/prose)"
        echo "  phi, struct   — Phi-3 Medium 14B (structured docs/API refs)"
        exit 0 ;;
    *)
        echo "Unknown model: $MODEL"
        echo "Run '$0 list' to see available models."
        exit 1 ;;
esac

echo "==> Ensuring Ollama container is running..."
"$MANAGER" ensure

echo "==> Using model: $OPENCODE_MODEL"
echo "==> Container will auto-shutdown after 15 min idle."
echo ""
echo "To start OpenCode with this model, run:"
echo "  opencode --model $OPENCODE_MODEL"
echo ""
echo "Or switch model inside OpenCode with Ctrl+P."
echo ""
echo "Available shortcuts: 7b | 14b | c | py | web | test | cn | doc | phi"
```

Add a convenience symlink:

```bash
chmod +x ~/project/local_cpu_llm/ollama-docker/opencode-local.sh
sudo ln -sf ~/project/local_cpu_llm/ollama-docker/opencode-local.sh /usr/local/bin/opencode-local
```

### Usage:

```bash
opencode-local            # Default: Qwen2.5 Coder 7B
opencode-local 14b        # Backup: Qwen2.5 Coder 14B
opencode-local c          # C/C++: CodeLlama 13B
opencode-local py         # Python: StarCoder2 15B
opencode-local web        # Web/JS/TS: DeepSeek Coder V2 16B
opencode-local test       # Unit/regression tests: DeepSeek Coder 6.7B
opencode-local cn         # Chinese/English: Qwen2.5 14B
opencode-local doc        # Documentation: Llama 3.1 8B
opencode-local phi        # Structured docs: Phi-3 Medium 14B
opencode-local list       # Show all options
```

---

## Step 6: Usage Workflow

### Quick start (recommended):
```bash
opencode-local            # Default: Qwen2.5 Coder 7B (general code)
opencode-local c          # C/C++ work: CodeLlama 13B
opencode-local py         # Python work: StarCoder2 15B
opencode-local web        # Web/JS/React: DeepSeek Coder V2 16B
opencode-local test       # Unit/regression tests: DeepSeek Coder 6.7B
opencode-local cn         # Chinese/English: Qwen2.5 14B
opencode-local doc        # Documentation: Llama 3.1 8B
opencode-local phi        # Structured docs: Phi-3 Medium 14B
opencode-local 14b        # Backup: Qwen2.5 Coder 14B
opencode-local list       # Show all options
```

### Manual control:
```bash
~/project/local_cpu_llm/ollama-docker/ollama-manager.sh start    # Start container
~/project/local_cpu_llm/ollama-docker/ollama-manager.sh stop     # Force stop
~/project/local_cpu_llm/ollama-docker/ollama-manager.sh status   # Check if running
~/project/local_cpu_llm/ollama-docker/ollama-manager.sh ensure   # Start if not running
```

### Inside OpenCode:
- Press `Ctrl+P` → switch model → pick from:
  - `local-ollama/qwen2.5-coder-7b` — general code (default)
  - `local-ollama/qwen2.5-coder-14b` — general code (backup)
  - `local-ollama/codellama-13b` — C/C++
  - `local-ollama/starcoder2-15b` — Python
  - `local-ollama/deepseek-coder-v2-16b` — Web/JS
  - `local-ollama/deepseek-coder-6.7b` — unit/regression tests
  - `local-ollama/qwen2.5-14b` — Chinese/English bilingual
  - `local-ollama/llama3.1-8b` — documentation
  - `local-ollama/phi3-medium` — structured docs

### Auto-shutdown:
- Container stops automatically after 15 minutes of no API requests
- All RAM is returned to the system
- Models persist in Docker volume (no re-download needed)

---

## Execution Checklist

| # | Task | Command | Status |
|---|------|---------|--------|
| 1 | Install Docker prerequisites | `sudo apt-get install -y ca-certificates curl gnupg` | Pending |
| 2 | Add Docker GPG key | `curl -fsSL https://download.docker.com/linux/ubuntu/gpg \| sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg` | Pending |
| 3 | Add Docker apt repo | `echo "deb [arch=amd64 ...] ... jammy stable" \| sudo tee ...` | Pending |
| 4 | Install Docker CE | `sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin` | Pending |
| 5 | Add user to docker group | `sudo usermod -aG docker $USER && newgrp docker` | Pending |
| 6 | Verify Docker | `docker --version && docker run hello-world` | Pending |
| 7 | Create project directory | `mkdir -p ~/project/local_cpu_llm/ollama-docker` | Pending |
| 8 | Create docker-compose.yml | (write file) | Pending |
| 9 | Create ollama-manager.sh | (write file + chmod +x) | Pending |
| 10 | Create opencode-local.sh | (write file + symlink) | Pending |
| 11 | Start container & pull 7B default | `docker exec ollama-local ollama pull qwen2.5-coder:7b` | Pending |
| 12 | Pull 14B backup | `docker exec ollama-local ollama pull qwen2.5-coder:14b` | Pending |
| 13 | Pull C/C++ model | `docker exec ollama-local ollama pull codellama:13b` | Pending |
| 14 | Pull Python model | `docker exec ollama-local ollama pull starcoder2:15b` | Pending |
| 15 | Pull Web model | `docker exec ollama-local ollama pull deepseek-coder-v2:16b` | Pending |
| 16 | Pull Testing model | `docker exec ollama-local ollama pull deepseek-coder:6.7b-instruct` | Pending |
| 17 | Pull Chinese model | `docker exec ollama-local ollama pull qwen2.5:14b` | Pending |
| 18 | Pull Docs model (Llama) | `docker exec ollama-local ollama pull llama3.1:8b` | Pending |
| 19 | Pull Docs model (Phi-3) | `docker exec ollama-local ollama pull phi3:medium` | Pending |
| 20 | Create systemd idle monitor | (write service + enable) | Pending |
| 21 | Update opencode.jsonc | (add local-ollama provider, set 7B as small_model) | Pending |
| 22 | Smoke test | `curl http://localhost:11434/v1/chat/completions ...` | Pending |
| 23 | Verify idle shutdown | Wait 15 min → confirm container stops | Pending |

---

## Installed Models

| Model | Shortcut | Size (Q4) | Speed (est.) | Specialty | Pull Command |
|-------|----------|-----------|--------------|-----------|--------------|
| **Qwen2.5-Coder 7B** | `7b` | ~4 GB | 10-15 tok/s | General code (DEFAULT) | `ollama pull qwen2.5-coder:7b` |
| **Qwen2.5-Coder 14B** | `14b` | ~8 GB | 5-8 tok/s | General code (BACKUP) | `ollama pull qwen2.5-coder:14b` |
| **CodeLlama 13B** | `c` | ~7 GB | 6-10 tok/s | C, C++, infilling | `ollama pull codellama:13b` |
| **StarCoder2 15B** | `py` | ~9 GB | 5-8 tok/s | Python, JS, TS, Go | `ollama pull starcoder2:15b` |
| **DeepSeek-Coder-V2 16B** | `web` | ~9 GB | 4-7 tok/s | JS/TS, React, HTML, CSS | `ollama pull deepseek-coder-v2:16b` |
| **DeepSeek-Coder 6.7B** | `test` | ~4 GB | 10-14 tok/s | Unit tests, regression tests | `ollama pull deepseek-coder:6.7b-instruct` |
| **Qwen2.5 14B** | `cn` | ~8 GB | 5-8 tok/s | Chinese/English bilingual | `ollama pull qwen2.5:14b` |
| **Llama 3.1 8B** | `doc` | ~4.5 GB | 10-14 tok/s | Documentation, prose, markdown | `ollama pull llama3.1:8b` |
| **Phi-3 Medium 14B** | `phi` | ~8 GB | 5-8 tok/s | Structured docs, API refs, README | `ollama pull phi3:medium` |

### Model Selection Guide

| Task | Recommended Model | Why |
|------|-------------------|-----|
| Quick code completions | `7b` (Qwen2.5-Coder 7B) | Fastest, good enough for most tasks |
| Complex algorithms, refactoring | `14b` (Qwen2.5-Coder 14B) | Higher quality reasoning |
| C/C++ kernel/driver code | `c` (CodeLlama 13B) | Trained heavily on C/C++, understands low-level patterns |
| Python scripts, data pipelines | `py` (StarCoder2 15B) | Strong Python/JS multi-language |
| React/Vue/HTML/CSS/JS | `web` (DeepSeek-Coder-V2 16B) | MoE architecture, web stack focus |
| Unit tests, test generation | `test` (DeepSeek-Coder 6.7B) | Fast, good at test patterns, assert/expect style |
| Chinese docs, bilingual code | `cn` (Qwen2.5 14B) | Alibaba model, native Chinese + strong English |
| README, comments, docstrings | `doc` (Llama 3.1 8B) | Best prose quality, fast |
| API documentation, structured output | `phi` (Phi-3 Medium 14B) | Clean structured markdown, tables |

---

## Resource Management

| State | RAM Used | CPU Used | Disk |
|-------|----------|----------|------|
| Container stopped | 0 | 0 | ~62 GB (all models in volume) |
| Container running, no model loaded | ~300 MB | <1% | Same |
| 7B model loaded, idle | ~6 GB | <1% | Same |
| 14B/15B/16B model loaded, idle | ~10-12 GB | <1% | Same |
| Model active (inference) | +2 GB over idle | 80-100% all cores | Same |

**Idle timeout**: Container shuts down after 15 minutes of no API requests, returning all RAM to the system.
**Only one model loaded at a time** (`OLLAMA_MAX_LOADED_MODELS=1`) to stay within 32 GB RAM.

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `permission denied` on docker | `sudo usermod -aG docker $USER && newgrp docker` |
| `connection refused` on :11434 | `~/project/local_cpu_llm/ollama-docker/ollama-manager.sh start` |
| Container starts but model slow | Normal for CPU-only; use 7B for speed |
| Out of memory | Reduce to 7B model or set `OLLAMA_MAX_LOADED_MODELS=1` |
| Models lost after `docker compose down` | Should not happen — models are in persistent volume `ollama-models` |
| Idle monitor not stopping container | Check `systemctl --user status ollama-idle-monitor` |
| OpenCode can't connect | Ensure container is running; try `curl http://localhost:11434/` |

---

## Smoke Test (after installation)

Run the full stack end-to-end:

```bash
# 1. Start the container
opencode-local

# 2. Test API directly (should return JSON with a code response)
curl -s http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5-coder:7b",
    "messages": [{"role": "user", "content": "Write a hello world in C"}],
    "stream": false
  }' | python3 -m json.tool

# 3. Test inside OpenCode
opencode --model local-ollama/qwen2.5-coder-7b
# Then ask: "Write a hello world in C"
```

Expected: response in 5-15 seconds, code returned in `choices[0].message.content`.

---

## Rollback / Uninstall

If you want to remove everything created by this guide:

```bash
# 1. Stop and remove container + volume (deletes all downloaded models)
docker compose -f ~/project/local_cpu_llm/ollama-docker/docker-compose.yml down -v

# 2. Disable and remove systemd idle monitor
systemctl --user stop ollama-idle-monitor.service
systemctl --user disable ollama-idle-monitor.service
rm ~/.config/systemd/user/ollama-idle-monitor.service
systemctl --user daemon-reload

# 3. Remove convenience symlink
sudo rm /usr/local/bin/opencode-local

# 4. Remove project files
rm -rf ~/project/local_cpu_llm/ollama-docker

# 5. Revert OpenCode config
# Edit ~/.config/opencode/opencode.jsonc:
#   - Remove "local-ollama" from "enabled_providers"
#   - Remove the "local-ollama" block from "provider"
#   - Reset "small_model" to "pedro-claude/claude-haiku-4-5"

# 6. (Optional) Uninstall Docker entirely
sudo apt-get purge -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo rm -rf /var/lib/docker /etc/docker
sudo rm /etc/apt/sources.list.d/docker.list
sudo rm /etc/apt/keyrings/docker.gpg
sudo deluser $USER docker
```

Verify cleanup:
```bash
docker ps -a              # Should not show ollama-local
docker volume ls          # Should not show ollama-models
systemctl --user status ollama-idle-monitor   # Should say "not loaded"
which opencode-local      # Should return nothing
```

---

## When to Use Local vs. API

| Use Local | Use API (Pedro) |
|-----------|-----------------|
| Private/sensitive code | Complex multi-file refactoring |
| Simple completions | Architecture decisions |
| Offline work | Long-context analysis (>32K) |
| Experimentation | Production-quality output |
| Cost = $0 forever | Best available intelligence |
| Quick lookups, grep-like tasks | Nuanced code review |
