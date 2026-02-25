# PicoClaw + Local LLM + Telegram (Nyarch/VirtualBox)

This repository documents a working PicoClaw deployment running on **Nyarch Linux inside VirtualBox**, using a **locally hosted Mistral GGUF model** exposed through `llama-server`, and connected to a **Telegram channel integration**.

The documentation below is derived from the implementation snapshot in `full_setup.txt` (configuration, runtime process list, model command, and host diagnostics).

---

## What this system is

This is a self-hosted assistant stack with these concrete components:

- **PicoClaw gateway** as the orchestration/runtime entrypoint (`picoclaw gateway`).
- **Local model server** (`llama-server`) serving OpenAI-compatible API responses at `127.0.0.1:8080/v1`.
- **Telegram channel integration** enabled in PicoClaw config.
- **Tooling enabled** inside PicoClaw: shell, filesystem, process, and DuckDuckGo web search.
- **Heartbeat + gateway listener** enabled for liveness and external access.

---

## System architecture

```text
[Telegram User]
      |
      v
[Telegram Bot API]
      |
      v
[PicoClaw Gateway :18790]
  - agent defaults
  - tool router (shell/filesystem/process/web)
  - heartbeat
      |
      v
[Local OpenAI-style model endpoint]
http://127.0.0.1:8080/v1
      |
      v
[llama-server]
  model: mistral-7b-instruct-v0.2.Q4_K_M.gguf
```

### Component-level behavior

1. Telegram updates are accepted by PicoClaw (`channels.telegram.enabled: true`).
2. PicoClaw runs agent logic with default model `local-model`.
3. `local-model` maps to a local backend served by llama-server (`api_base` loopback URL).
4. Optional tools are invoked based on task needs (`shell`, `filesystem`, `process`, `web.duckduckgo`).
5. Gateway is exposed on `0.0.0.0:18790`, and heartbeat runs every 30 seconds.

---

## Integration explanation

## 1) Nyarch Linux in VirtualBox

The host runtime is Linux (`x86_64` kernel), showing a VM-like environment (`meow-virtualbox`) with available CPU/RAM and mounted storage. The deployment is therefore local-first and VM-contained.

## 2) Local AI model integration

A quantized GGUF model is loaded by llama-server:

- `mistral-7b-instruct-v0.2.Q4_K_M.gguf`
- Served on `127.0.0.1:8080`
- OpenAI-compatible path `/v1`

PicoClaw points to this endpoint via `model_list[0].api_base` and uses model alias `local-model` in agent defaults.

## 3) PicoClaw integration

`picoclaw gateway` is actively running and configured with:

- workspace path
- token/context limits
- tool permissions
- heartbeat
- public bind host + port

## 4) Telegram bot integration

Telegram channel is enabled and constrained by `allow_from` list. Sensitive token and IDs are sanitized in this repository.

---

## Project structure

```text
.
├── README.md
└── full_setup.txt              # runtime snapshot + config (sanitized)
```

---

## Execution flow (real data flow)

1. **Model server startup**
   - `llama-server` starts with model path, context size, template, deterministic sampling parameters, and local bind.

2. **PicoClaw gateway startup**
   - `picoclaw gateway` loads config defaults and channel/tool definitions.

3. **Message ingress from Telegram**
   - Telegram updates enter through configured bot token and permitted sender IDs.

4. **Agent inference cycle**
   - PicoClaw resolves `local-model` → OpenAI-compatible endpoint.
   - Prompt is sent to `http://127.0.0.1:8080/v1`.
   - Model response returns to PicoClaw.

5. **Optional tool execution**
   - If requested by the agent logic, shell/filesystem/process/web tools are called.

6. **Response egress**
   - PicoClaw sends the resulting response back over Telegram.

7. **Operational liveness**
   - Heartbeat emits every 30s.
   - Warp status indicates connectivity is healthy.

---

## Setup instructions (sanitized)

> These steps mirror the implementation shown in this repo and intentionally avoid exposing private values.

1. **Prepare Nyarch Linux VM** (VirtualBox) with enough RAM/CPU for local inference.
2. **Build or install llama.cpp / llama-server**.
3. **Place model file**:
   - `models/mistral-7B/mistral-7b-instruct-v0.2.Q4_K_M.gguf`
4. **Run llama-server** (example from implementation):

```bash
./build/bin/llama-server \
  -m models/mistral-7B/mistral-7b-instruct-v0.2.Q4_K_M.gguf \
  -c 4096 \
  --port 8080 \
  --host 127.0.0.1 \
  --chat-template chatml \
  --temp 0 \
  --top-k 1 \
  --seed 42
```

5. **Configure PicoClaw** with a model alias that points to `http://127.0.0.1:8080/v1`.
6. **Configure Telegram channel** with sanitized placeholders.
7. **Start PicoClaw**:

```bash
picoclaw gateway
```

8. **Verify processes** (`llama-server` and `picoclaw gateway`) and test via Telegram.

---

## Configuration guide (sanitized)

Use this template structure:

```json
{
  "agents": {
    "defaults": {
      "workspace": "/path/to/workspace",
      "restrict_to_workspace": false,
      "model": "local-model",
      "max_tokens": 8192,
      "temperature": 0.7,
      "max_tool_iterations": 20
    }
  },
  "model_list": [
    {
      "model_name": "local-model",
      "model": "mistral-7b-instruct-v0.2.Q4_K_M.gguf",
      "api_base": "http://127.0.0.1:8080/v1",
      "api_key": "<API_KEY>"
    }
  ],
  "channels": {
    "telegram": {
      "enabled": true,
      "token": "<TELEGRAM_TOKEN>",
      "allow_from": ["<SECRET>"]
    }
  }
}
```

---

## Code-level documentation summary

This repository currently provides an **operational snapshot**, not application source modules. Based on the implementation data:

- The core “control plane” is PicoClaw’s JSON configuration.
- The model integration contract is OpenAI-style HTTP (`api_base` + `api_key` fields).
- Tool integrations are declarative flags in config and routed by PicoClaw at runtime.
- Runtime process data confirms concurrent operation of inference server and gateway.

No additional in-repo source files define custom business logic, handlers, or plugin code in this snapshot.

---

## Portfolio summary (short)

Built and validated a local-first AI assistant stack in a Nyarch Linux VM (VirtualBox), integrating PicoClaw gateway, local Mistral GGUF inference through llama-server, and Telegram bot messaging. System includes controlled tool access (shell/filesystem/process/web), heartbeat monitoring, and stable runtime connectivity.
