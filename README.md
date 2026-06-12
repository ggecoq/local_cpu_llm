# Local LLM with Docker & OpenCode

Run local LLMs on-demand in Docker, integrated with [OpenCode](https://github.com/anomalyco/opencode) for AI-assisted coding — with automatic idle shutdown to reclaim RAM when not in use.

## Host Requirements

| Component | Minimum | This Setup |
|-----------|---------|------------|
| CPU | 4 cores | AMD EPYC 7352 24-Core (48 threads) |
| RAM | 8 GB | 32 GB (28 GB allocated to container) |
| Disk | 70 GB free | 938 GB NVMe |
| GPU | Not required | CPU-only inference |
| OS | Ubuntu 20.04+ | Ubuntu 22.04.5 LTS |

> **CPU-only inference**: expect 5-15 tokens/sec depending on model size.

---

## What This Does

- Runs [Ollama](https://ollama.com) inside a Docker container with a 28 GB RAM cap
- Container starts **on demand** and shuts down after **15 minutes idle** via systemd
- 9 specialist models pre-configured for different tasks (code, C/C++, Python, web, testing, Chinese, docs)
- Shell shortcuts (`opencode-local c`, `opencode-local py`, etc.) for fast model switching
- Fully integrated with OpenCode's provider system alongside existing cloud APIs

---

## Models Included

| Shortcut | Model | Specialty | Size |
|----------|-------|-----------|------|
| `7b` | Qwen2.5-Coder 7B | General code (default) | ~4 GB |
| `14b` | Qwen2.5-Coder 14B | General code (backup) | ~8 GB |
| `c` | CodeLlama 13B | C / C++ | ~7 GB |
| `py` | StarCoder2 15B | Python, JS, TS | ~9 GB |
| `web` | DeepSeek-Coder-V2 16B | Web / React / HTML / CSS | ~9 GB |
| `test` | DeepSeek-Coder 6.7B | Unit & regression tests | ~4 GB |
| `cn` | Qwen2.5 14B | Chinese / English bilingual | ~8 GB |
| `doc` | Llama 3.1 8B | Documentation / prose | ~4.5 GB |
| `phi` | Phi-3 Medium 14B | Structured docs / API refs | ~8 GB |

**Total disk**: ~62 GB. Only one model loaded into RAM at a time.

---

## Quick Start

```bash
# Start container + select model
opencode-local              # default: Qwen2.5-Coder 7B
opencode-local c            # C/C++
opencode-local py           # Python
opencode-local web          # Web/JS
opencode-local test         # Unit tests
opencode-local cn           # Chinese/English
opencode-local doc          # Documentation
opencode-local list         # Show all options

# Then launch OpenCode with the printed model ID
opencode --model local-ollama/qwen2.5-coder-7b
```

---

## Files

| File | Description |
|------|-------------|
| `install-local-llm.md` | Full step-by-step installation guide (23 steps) |
| `ollama-docker/docker-compose.yml` | Docker Compose for Ollama container |
| `ollama-docker/ollama-manager.sh` | Start / stop / status / ensure container |
| `ollama-docker/opencode-local.sh` | Model selector + container wake script |

> The script files in `ollama-docker/` are generated during the install guide — they are not included in this repo. Follow `install-local-llm.md` to create them.

---

## Installation Time

| Phase | Time |
|-------|------|
| Install Docker + create files | ~20 min (hands-on) |
| Pull 9 models (~62 GB) | ~30-40 min (unattended) |
| Configure OpenCode + smoke test | ~15 min |
| **Total** | **~60-75 min** |

---

## Architecture

```
User
└── opencode-local [shortcut]
    └── ollama-manager.sh ensure
        └── Docker Container: ollama-local (28 GB RAM cap)
            └── Ollama API :11434 (OpenAI-compatible)
                └── One model loaded at a time

systemd --user: ollama-idle-monitor.service
└── Stops container after 15 min idle → RAM returned to system
```

---

## Related

- [Ollama](https://ollama.com) — Local LLM runtime
- [OpenCode](https://github.com/anomalyco/opencode) — Open source AI coding agent
- [opencode-local-ollama](https://github.com/khalilgharbaoui/opencode-local-ollama) — Plugin for auto model discovery
- [ollama launch opencode](https://docs.ollama.com/integrations/opencode) — Official Ollama/OpenCode integration (simpler, no Docker)
---

## License

