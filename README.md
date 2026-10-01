# GH200 Local LLM + Cline Desktop Remote SSH Memo

> Updated: 2026-10-01

## Goal

Run a local LLM on an NVIDIA GH200 Linux server and expose it as an OpenAI-compatible API with vLLM.

Use **Cline Desktop on Windows** as the agent UI, while keeping both model inference and source-code work on the GH200 machine.

- **LLM inference:** GH200 + vLLM + Qwen3.8-27B-FP8
- **Agent workspace:** GH200 Linux filesystem
- **Agent access:** Cline Desktop Remote SSH
- **LLM API access:** SSH port forwarding from Windows
- **Source code:** stays on GH200

---

## Recommended architecture

```text
Windows PC
┌──────────────────────────────────────────┐
│ Cline Desktop                            │
│                                          │
│ 1. OpenAI-compatible API                 │
│    http://127.0.0.1:18000/v1             │
│             │                            │
│             │ SSH Port Forward           │
│             └───────────────────┐        │
│                                 │        │
│ 2. Remote SSH                   │        │
│    Cline ───────────────────────┼────┐   │
└─────────────────────────────────┼────┼───┘
                                  │    │
                                  ▼    ▼
                     GH200 Linux Server
             ┌────────────────────────────────┐
             │ vLLM :8000                    │
             │     ↓                          │
             │ Qwen3.8-27B-FP8               │
             │     ↓                          │
             │ GH200                          │
             │                                │
             │ ~/workspace/...                │
             │ git / gcc / cmake / python    │
             └────────────────────────────────┘
```

## Initial settings

| Item | Setting |
|---|---|
| Model | `Qwen/Qwen3.8-27B-FP8` |
| Inference server | vLLM |
| GPU | GH200 x1 |
| Tensor Parallel | TP=1 |
| Context length | Start with 32K |
| vLLM listen address | `127.0.0.1:8000` |
| Windows endpoint | `127.0.0.1:18000` |
| Agent UI | Cline Desktop |
| Workspace | GH200 side |
| Authentication | SSH key + vLLM API key |

---

# 1. Check the GH200 environment

SSH into the server and check:

```bash
uname -m
nvidia-smi
python3 --version
free -h
df -h
```

A Grace CPU environment will normally report:

```text
aarch64
```

---

# 2. Create a Python environment for vLLM

```bash
mkdir -p ~/llm
cd ~/llm

python3 -m venv venv
source venv/bin/activate

python -m pip install -U pip uv
uv pip install vllm --torch-backend=auto
```

Check CUDA / PyTorch:

```bash
python -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

Expected output is similar to:

```text
True
NVIDIA GH200 ...
```

If the AArch64 / CUDA / PyTorch combination causes wheel or dependency issues, use an official vLLM container or an ARM64/GH200-compatible build instead of manually forcing incompatible packages.

---

# 3. Generate a vLLM API key

Generate a random secret:

```bash
openssl rand -hex 32
```

Set it in the shell:

```bash
export VLLM_API_KEY='<generated-secret>'
```

Do **not** commit the real API key to GitHub.

For permanent operation, store secrets outside the repository, for example in a protected systemd EnvironmentFile.

---

# 4. Start Qwen3.8-27B-FP8 with vLLM

Start conservatively with a 32K context window and 85% GPU-memory utilization:

```bash
source ~/llm/venv/bin/activate

vllm serve Qwen/Qwen3.8-27B-FP8 \
    --host 127.0.0.1 \
    --port 8000 \
    --served-model-name qwen3.8-27b \
    --api-key "$VLLM_API_KEY" \
    --max-model-len 32768 \
    --gpu-memory-utilization 0.85
```

Notes:

- GH200 x1 means `TP=1`.
- `--tensor-parallel-size` is therefore not required initially.
- Binding to `127.0.0.1` avoids exposing the raw vLLM port to the LAN.
- Increase context length to 64K / 128K only after confirming memory usage and stability.

---

# 5. Test the API on the GH200 host

List models:

```bash
curl http://127.0.0.1:8000/v1/models \
  -H "Authorization: Bearer $VLLM_API_KEY"
```

Test Chat Completions:

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Authorization: Bearer $VLLM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3.8-27b",
    "messages": [
      {
        "role": "user",
        "content": "Write a hello world program in Python."
      }
    ],
    "max_tokens": 256
  }'
```

If this works, the basic LLM server is ready.

---

# 6. Create an SSH tunnel from Windows

From PowerShell:

```powershell
ssh -N -L 18000:127.0.0.1:8000 <user>@<gh200-host>
```

Connection path:

```text
Windows
127.0.0.1:18000
      │
      │ SSH tunnel
      ▼
GH200
127.0.0.1:8000
```

The OpenAI-compatible API is then reachable from Windows at:

```text
http://127.0.0.1:18000/v1
```

---

# 7. Configure SSH key authentication

Create a key on Windows if needed:

```powershell
ssh-keygen -t ed25519
```

Register the public key in:

```text
~/.ssh/authorized_keys
```

on the GH200 server.

Before using Cline Remote SSH, verify normal SSH access:

```powershell
ssh <user>@<gh200-host>
```

Check that:

- key-based login works without interactive password entry,
- the host key is registered in Windows `known_hosts`,
- the intended GH200 workspace is accessible.

---

# 8. Configure Cline Desktop Remote SSH

Add the GH200 as a remote environment in Cline Desktop.

Example:

```text
Host:
<gh200-host>

User:
<user>

Identity:
C:\Users\<user>\.ssh\id_ed25519
```

Workspace example:

```text
/home/<user>/workspace
```

Use a Cline Desktop version that includes the Remote SSH fixes introduced in 0.0.35 or later. Prefer the latest stable release available in the environment.

---

# 9. Configure the local LLM in Cline

Provider:

```text
OpenAI Compatible
```

Base URL:

```text
http://127.0.0.1:18000/v1
```

API Key:

```text
<VLLM_API_KEY>
```

Model:

```text
qwen3.8-27b
```

Context window:

```text
32768
```

---

# 10. How the agent works

```text
                    Windows
                Cline Desktop
                     │
          ┌──────────┴──────────┐
          │                     │
     LLM request            Agent tools
          │                     │
          ▼                     ▼
     SSH tunnel            Remote SSH
          │                     │
          ▼                     ▼
        vLLM                workspace
          │                 git/build/test
          ▼
 Qwen3.8-27B-FP8
          │
          ▼
        GH200
```

The two paths are intentionally separate:

### Reasoning path

```text
Cline
  ↓
OpenAI-compatible API
  ↓
Qwen3.8
  ↓
Decide the next tool / command
```

### Execution path

```text
Cline
  ↓
Remote SSH
  ↓
GH200
  ↓
read / edit / git / build / test
```

Windows is primarily the UI / controller. Model inference and engineering work stay on the GH200 side.

---

# 11. Recommended validation sequence

## Step 1: SSH only

```bash
pwd
uname -m
nvidia-smi
```

## Step 2: Read-only agent task

Example Cline prompt:

```text
Analyze this repository and explain its structure.
Do not modify any files yet.
```

## Step 3: Small file modification

```text
Create test.txt and write "Hello" into it.
```

## Step 4: Git inspection

```text
Run git diff and explain the changes.
```

## Step 5: Build and test

```text
Build this project and run its tests.
```

## Step 6: Autonomous debugging task

```text
Investigate this build error and make the necessary fix.
After the fix, run the tests again.
```

This sequence isolates failures in:

```text
SSH
↓
filesystem
↓
terminal
↓
LLM tool calling
↓
coding agent
```

---

# 12. Run vLLM with systemd for regular use

Once the manual setup is stable, move vLLM startup to systemd.

Conceptual unit file:

```ini
[Unit]
Description=vLLM Qwen3.8 API Server
After=network.target

[Service]
Type=simple
User=<user>
WorkingDirectory=/home/<user>/llm
EnvironmentFile=/home/<user>/.config/vllm/env
ExecStart=/home/<user>/llm/venv/bin/vllm serve Qwen/Qwen3.8-27B-FP8 \
  --host 127.0.0.1 \
  --port 8000 \
  --served-model-name qwen3.8-27b \
  --api-key ${VLLM_API_KEY} \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.85
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Environment file:

```text
VLLM_API_KEY=<secret>
```

Do not commit this environment file to the repository.

---

# Security guidelines

- Bind vLLM to `127.0.0.1`.
- Access the API from Windows through SSH port forwarding.
- Use SSH public-key authentication.
- Use a vLLM API key.
- Do not store API keys, private keys, passwords, or credentials in GitHub.
- Keep company source code on the GH200 side.
- Avoid sending internal code to external LLM APIs.
- Limit the agent's writable workspace.
- Start validation with read-only tasks.
- Review `git diff` before committing agent-generated changes.

---

# Future improvements

## Longer context

Increase only after confirming stable behavior:

```text
32K
 ↓
64K
 ↓
128K
```

Measure GPU memory consumption and latency at each step.

## Model comparison

Possible GH200 experiments:

- Qwen3.8-27B-FP8
- larger quantized models
- MoE models with Grace CPU memory offload

Useful metrics:

- TTFT
- decode tokens/sec
- concurrent requests
- KV-cache usage
- coding-task success rate
- Cline tool-call success rate
- long-horizon agent-task completion rate

## Computer Use

The same local API can later be connected to a separate Computer Use loop:

```text
Screenshot
   ↓
Vision-capable LLM
   ↓
Action
   ↓
Browser / Desktop GUI
```

This allows the GH200 local model to serve both coding-agent and GUI-agent experiments.

---

# Target end state

```text
                 Windows
              Cline Desktop
                     │
          ┌──────────┴──────────┐
          │                     │
     OpenAI API             Remote SSH
          │                     │
 localhost:18000                │
          │                     │
      SSH tunnel                │
          │                     │
          └─────────┬───────────┘
                    │
                    ▼
                  GH200
      ┌────────────────────────────┐
      │ systemd                    │
      │   ↓                        │
      │ vLLM :8000                 │
      │   ↓                        │
      │ Qwen3.8-27B-FP8           │
      │                            │
      │ Cline Remote Environment  │
      │   ↓                        │
      │ workspace / git / build   │
      └────────────────────────────┘
```
