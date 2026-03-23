# OpenClaw - NemoClaw + OpenShell — Complete Manual
**Version**: Alpha 2026.3.x | OpenShell v0.0.13 | OpenClaw 2026.3.11  
**Last Updated**: March 23, 2026  
**Scope**: Full installation, configuration, and optimal operational use  
**Sandbox default name**: `my-assistant`  
**Web UI**: http://127.0.0.1:18789/

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites & System Requirements](#2-prerequisites--system-requirements)
3. [Pre-Installation Steps](#3-pre-installation-steps)
4. [Installing Core Dependencies](#4-installing-core-dependencies)
5. [Installing OpenShell](#5-installing-openshell)
6. [Installing NemoClaw — Full Onboarding Walkthrough](#6-installing-nemoclaw--full-onboarding-walkthrough)
7. [What Got Installed — File & Directory Map](#7-what-got-installed--file--directory-map)
8. [Web UI Dashboard — http://127.0.0.1:18789/](#8-web-ui-dashboard--http127001189)
   - [8.1 Accessing the Web UI — Common Issues & Fixes](#81-accessing-the-web-ui--common-issues--fixes)
9. [OpenShell Deep Dive](#9-openshell-deep-dive)
10. [NemoClaw Deep Dive](#10-nemoclaw-deep-dive)
11. [Policy System — Full Reference](#11-policy-system--full-reference)
12. [Inference & Model Configuration](#12-inference--model-configuration)
13. [File & Workspace Management](#13-file--workspace-management)
14. [Networking & Port Forwarding](#14-networking--port-forwarding)
15. [Multi-Sandbox & Multi-Agent Setup](#15-multi-sandbox--multi-agent-setup)
16. [Daily Operational Workflow](#16-daily-operational-workflow)
17. [Monitoring, Logging & Observability](#17-monitoring-logging--observability)
18. [Backup, Restore & Disaster Recovery](#18-backup-restore--disaster-recovery)
19. [Troubleshooting Reference](#19-troubleshooting-reference)
20. [Security Hardening](#20-security-hardening)
21. [Full Uninstall](#21-full-uninstall)
22. [Quick Reference Cheat Sheet](#22-quick-reference-cheat-sheet)

---

## 1. Architecture Overview

Understanding the full stack before touching a terminal is mandatory. Every tool has a specific layer responsibility.

```
┌──────────────────────────────────────────────────────────────────────┐
│                            HOST MACHINE                              │
│                                                                      │
│  ┌─────────────────┐     ┌────────────────────────────────────────┐  │
│  │   NemoClaw CLI  │     │           OpenShell CLI                │  │
│  │  (Orchestrator) │     │    (Sandbox runtime control plane)     │  │
│  └────────┬────────┘     └──────────────────┬─────────────────────┘  │
│           │                                 │                        │
│           └──────────────────┬──────────────┘                        │
│                              │                                       │
│                 ┌────────────▼────────────┐                          │
│                 │   OpenShell Gateway     │  :8080 (TLS)             │
│                 │   Docker container      │                          │
│                 │   ghcr.io/nvidia/       │                          │
│                 │   openshell/cluster     │                          │
│                 │   k3s + netns +         │                          │
│                 │   seccomp + Landlock    │                          │
│                 └────────────┬────────────┘                          │
│                              │                                       │
│              ┌───────────────▼──────────────────┐                   │
│              │      SANDBOX: my-assistant        │                   │
│              │   (Landlock + seccomp + netns)    │                   │
│              │                                   │                   │
│              │  /sandbox/                        │                   │
│              │    .openclaw/                     │                   │
│              │    .openclaw-data/                │                   │
│              │      agents/  canvas/  cron/      │                   │
│              │      devices/ extensions/ hooks/  │                   │
│              │      identity/ skills/            │                   │
│              │      workspace/  <- your files    │                   │
│              │    .nemoclaw/                     │                   │
│              │                                   │                   │
│              │  OpenClaw 2026.3.11               │                   │
│              │  ws://127.0.0.1:18789  <----------+-- Web UI          │
│              └───────────────┬───────────────────┘                   │
│                              │ policy-gated egress                   │
└──────────────────────────────┼───────────────────────────────────────┘
                               │
                               ▼
              ┌─────────────────────────────────┐
              │  NVIDIA Endpoint API            │
              │  inference.local (proxy route)  │
              │  model: nemotron-3-super-120b   │
              │  -- or -- Local Ollama :11434   │
              └─────────────────────────────────┘
```

| Component | Version | Layer | Responsibility |
|-----------|---------|-------|----------------|
| **NemoClaw** | v0.1.0 | Orchestration | One-command install, lifecycle, policy presets |
| **OpenShell** | v0.0.13 | Runtime | Kernel sandbox via Landlock + seccomp + netns + k3s |
| **OpenClaw** | 2026.3.11 | Agent | AI agent running inside the sandbox |
| **Gateway** | cluster:0.0.13 | Network proxy | Policy enforcement, inference routing |
| **Nemotron API** | — | Inference | Cloud LLM (or local Ollama fallback) |

**Ports in use after install:**
- `:8080` — OpenShell gateway (TLS, internal)
- `:18789` — OpenClaw web UI + WebSocket (`ws://127.0.0.1:18789`)

---

## 2. Prerequisites & System Requirements

### 2.1 Hardware Requirements

| Resource | Minimum | Recommended | Notes |
|----------|---------|-------------|-------|
| CPU | 4 cores x86_64 | 8+ cores | ARM not supported |
| RAM | 8 GB | 16–32 GB | Gateway image is ~1.2 GB |
| Disk | 25 GB free | 50+ GB SSD | Image build adds ~1.2 GB |
| GPU | None (cloud) | NVIDIA GPU 8 GB VRAM | For local Ollama inference |
| OS | Ubuntu 22.04 LTS | Ubuntu 22.04 / 24.04 LTS | Debian-based only |
| Kernel | 5.13+ | 6.x | Landlock LSM required |

> **GPU note**: The onboard wizard auto-detects GPUs. A confirmed working setup shows: `NVIDIA GPU detected: 1 GPU(s), 8192 MB VRAM`. Even with a GPU, you can still choose cloud inference.

### 2.2 Software Prerequisites Checklist

Before running any installer, verify these are present:

```bash
# Check OS
lsb_release -a

# Check kernel (must be 5.13+)
uname -r

# Check Docker is running
docker ps

# Check Node.js (NemoClaw needs it — installer will check)
node --version    # Needs v18+  (v22.22.0 confirmed working)
npm --version     # 10.9.4 confirmed working

# Check ports are free
ss -tlnp | grep -E '8080|18789'
# Both must show nothing (no process using them)
```

### 2.3 Ports That Must Be Free

| Port | Used By | What happens if occupied |
|------|---------|--------------------------|
| `8080` | OpenShell gateway | Gateway fails to start at step [2/7] |
| `18789` | OpenClaw web UI | Dashboard inaccessible |

```bash
# Free port 8080 if needed
sudo lsof -i :8080
sudo lsof -i :18789
sudo kill -9 <PID>
```

---

## 3. Pre-Installation Steps

Complete every step before running any installer.

### 3.1 Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### 3.2 Install Essential System Tools

```bash
sudo apt install -y \
  curl \
  wget \
  git \
  build-essential \
  ca-certificates \
  gnupg \
  lsb-release \
  software-properties-common \
  apt-transport-https \
  unzip \
  jq \
  htop \
  net-tools \
  iputils-ping \
  dnsutils
```

### 3.3 Verify Kernel Landlock Support

```bash
# Method 1 — check version (must be 5.13+)
uname -r

# Method 2 — check Landlock config
grep LANDLOCK /boot/config-$(uname -r) 2>/dev/null
# Should show: CONFIG_SECURITY_LANDLOCK=y

# Method 3
cat /sys/kernel/security/lsm 2>/dev/null | tr ',' '\n' | grep landlock
# Should print: landlock
```

If kernel is below 5.13:

```bash
sudo apt update
sudo apt install --install-recommends linux-generic-hwe-22.04 -y
sudo reboot
uname -r   # verify after reboot
```

### 3.4 Install and Configure Docker

The OpenShell gateway runs as a Docker container (`ghcr.io/nvidia/openshell/cluster:0.0.13`). Docker must be installed and **running** before NemoClaw onboard — the preflight check at step [1/7] verifies `Docker is running`.

```bash
# Remove conflicting old versions
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null || true

# Add Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# Add your user to docker group (no sudo needed for docker commands)
sudo usermod -aG docker $USER
newgrp docker   # apply without logout

# Enable and start Docker
sudo systemctl enable docker
sudo systemctl start docker

# Verify — both must succeed
docker --version
docker ps
# docker ps must return a table header, NOT a permission error
```

### 3.5 Install NVIDIA Container Toolkit (GPU Users Only)

Skip if using NVIDIA cloud API inference only. Required only for local Ollama with GPU.

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Verify
docker run --rm --gpus all nvidia/cuda:12.0-base-ubuntu22.04 nvidia-smi
```

### 3.6 Configure Permanent PATH

Both OpenShell and NemoClaw install binaries to non-standard locations. Set this once before installing anything.

```bash
cat >> ~/.bashrc << 'EOF'

# OpenShell + NemoClaw
export PATH="$HOME/.local/bin:$PATH"
EOF

source ~/.bashrc
```

### 3.7 Prepare Working Directories

```bash
mkdir -p ~/my-assistant-workspace
mkdir -p ~/backups/my-assistant
mkdir -p ~/scripts
mkdir -p ~/logs
```

---

## 4. Installing Core Dependencies

### 4.1 Install Node.js via nvm (Required)

The NemoClaw installer checks for Node.js at phase [1/3]. Node.js v22.22.0 with npm 10.9.4 is confirmed working.

```bash
# Install nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Load nvm in current shell
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# Install Node.js LTS
nvm install --lts
nvm use --lts
nvm alias default 'lts/*'

# Verify
node --version    # v22.x.x
npm --version     # 10.x.x
```

If nvm is already installed, just switch to the right version:

```bash
nvm install 22
nvm use 22
```

### 4.2 Install Ollama (Optional — Local Inference Only)

Only needed if you want free local inference instead of the NVIDIA cloud API. The onboarding wizard auto-detects Ollama if it is running at `localhost:11434`.

```bash
curl -fsSL https://ollama.ai/install.sh | sh

# Start the Ollama server
ollama serve &

# Pull a model
ollama pull llama3.1:8b       # Fast, lighter
ollama pull qwen2.5:32b       # Good for code
ollama pull llama3.1:70b      # High quality, needs 40+ GB RAM

# Verify it is running before proceeding to install
curl http://localhost:11434/api/tags
```

---

## 5. Installing OpenShell

### 5.1 Run the Installer

```bash
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
```

You will see:

```
openshell: resolving latest version...
openshell: downloading openshell v0.0.13 (x86_64-unknown-linux-musl)...
openshell: verifying checksum...
openshell: extracting...
openshell: installed openshell 0.0.13 to /home/<user>/.local/bin/openshell

openshell: /home/<user>/.local/bin is not on your PATH.
openshell:     export PATH="/home/<user>/.local/bin:$PATH"
```

### 5.2 Fix PATH Immediately

```bash
export PATH="$HOME/.local/bin:$PATH"
source ~/.bashrc
```

### 5.3 Verify

```bash
openshell --version
# openshell 0.0.13

openshell --help
# Shows all available commands
```

> **Do NOT run `openshell gateway start` manually.** The NemoClaw onboard wizard handles gateway startup automatically at step [2/7].

---

## 6. Installing NemoClaw — Full Onboarding Walkthrough

### 6.1 Run the NemoClaw Installer

```bash
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash
```

> **Known alpha issue**: You may see `bash: line 9: BASH_SOURCE[0]: unbound variable` at the very start. This is a non-fatal warning in the installer script. The installation continues normally past it.

### 6.2 Phase [1/3] — Node.js Check

```
[INFO]  Node.js found: v22.22.0
[INFO]  Runtime OK: Node.js v22.22.0, npm 10.9.4
```

If this fails, go back to §4.1.

### 6.3 Phase [2/3] — NemoClaw CLI Install

```
[INFO]  Installing NemoClaw from GitHub...
  ✓  Cloning NemoClaw source
  ✓  Preparing OpenClaw package
  ✓  Installing NemoClaw dependencies
  ✓  Building NemoClaw plugin
  ✓  Linking NemoClaw CLI
[INFO]  Verified: nemoclaw is available at /home/<user>/.nvm/versions/node/v22.22.0/bin/nemoclaw

[WARN]  Your current shell may not have the updated PATH.
  To use nemoclaw now, run:
    source /home/<user>/.bashrc
```

After this completes:

```bash
source ~/.bashrc
```

### 6.4 Phase [3/3] — Onboarding Wizard

This is the interactive configuration. It runs 7 sub-steps automatically.

---

#### Step [1/7] — Preflight Checks

All six checks must pass:

```
✓ Docker is running
✓ Container runtime: docker
✓ openshell CLI: openshell 0.0.13
✓ Port 8080 available (OpenShell gateway)
✓ Port 18789 available (NemoClaw dashboard)
✓ NVIDIA GPU detected: 1 GPU(s), 8192 MB VRAM
```

If any fail, fix them before re-running (see §19 Troubleshooting).

---

#### Step [2/7] — Starting OpenShell Gateway

```
Using pinned OpenShell gateway image: ghcr.io/nvidia/openshell/cluster:0.0.13
✓ Checking Docker
✓ Downloading gateway        <- pulls ~1.2 GB image (first run only)
✓ Initializing environment
✓ Starting gateway
✓ Gateway ready

Name: nemoclaw
Endpoint: https://127.0.0.1:8080

✓ Active gateway set to 'nemoclaw'
✓ Gateway is healthy
```

Verify the gateway container after install:

```bash
docker ps
# CONTAINER ID  IMAGE                                    NAMES
# da1c87533801  ghcr.io/nvidia/openshell/cluster:0.0.13  openshell-cluster-nemoclaw
```

---

#### Step [3/7] — Creating the Sandbox

```
Sandbox name (lowercase, numbers, hyphens) [my-assistant]:
```

Press **Enter** to accept `my-assistant`. This name is used in every subsequent command.

```
Creating sandbox 'my-assistant' (this takes a few minutes on first run)...
Building image openshell/sandbox-from:... from Dockerfile
[progress] Exported 1174 MiB
[progress] Uploaded to gateway
Waiting for sandbox to become ready...
✓ Forwarding port 18789 to sandbox my-assistant in the background
  Access at: http://127.0.0.1:18789/
  Stop with: openshell forward stop 18789 my-assistant
✓ Sandbox 'my-assistant' created
```

> The sandbox image build exports ~1.2 GB. On first run this takes several minutes depending on your connection.

---

#### Step [4/7] — Configuring Inference

```
Inference options:
  1) NVIDIA Endpoint API (build.nvidia.com)
  2) Local Ollama (localhost:11434) — running (suggested)

Choose [1]:
```

**Option 1 — NVIDIA Endpoint API (Cloud, Recommended):**

Type `1` and press Enter.

```
NVIDIA API Key: nvapi-xxxxxxxxxxxxxxxxxxxxxxxxxxxx
Key saved to ~/.nemoclaw/credentials.json (mode 600)
```

Then select a model:

```
Cloud models:
  1) Nemotron 3 Super 120B  (nvidia/nemotron-3-super-120b-a12b)  <- RECOMMENDED
  2) Kimi K2.5              (moonshotai/kimi-k2.5)
  3) GLM-5                  (z-ai/glm5)
  4) MiniMax M2.5           (minimaxai/minimax-m2.5)
  5) Qwen3.5 397B A17B      (qwen/qwen3.5-397b-a17b)
  6) GPT-OSS 120B           (openai/gpt-oss-120b)

Choose model [1]:
```

Press `1` + Enter. Nemotron 3 Super 120B is the best starting model.

**Option 2 — Local Ollama (Free, No API Key):**

Type `2` + Enter. Ollama must already be running with a model pulled (see §4.2).

---

#### Step [5/7] — Setting Up Inference Provider

```
✓ Created provider nvidia-nim
Gateway inference configured:

  Route: inference.local
  Provider: nvidia-nim
  Model: nvidia/nemotron-3-super-120b-a12b
  Version: 1
  ✓ Inference route set: nvidia-nim / nvidia/nemotron-3-super-120b-a12b
```

All inference from inside the sandbox routes through `https://inference.local/v1` — an internal gateway proxy regardless of whether you chose cloud or local.

---

#### Step [6/7] — Setting Up OpenClaw Inside Sandbox

```
✓ OpenClaw gateway launched inside sandbox
```

OpenClaw is installed and started inside `my-assistant`. WebSocket binds to `ws://127.0.0.1:18789`.

---

#### Step [7/7] — Policy Presets

```
Available policy presets:
  o discord     -- Discord API, gateway, and CDN access
  o docker      -- Docker Hub and NVIDIA container registry access
  o huggingface -- Hugging Face Hub, LFS, and Inference API access
  o jira        -- Jira and Atlassian Cloud access
  o npm         -- npm and Yarn registry access (suggested)
  o outlook     -- Microsoft Outlook and Graph API access
  o pypi        -- Python Package Index (PyPI) access (suggested)
  o slack       -- Slack API and webhooks access
  o telegram    -- Telegram Bot API access

Apply suggested presets (pypi, npm)? [Y/n/list]: y
```

Press `y` + Enter.

```
✓ Policy version 2 loaded (active version: 2)  Applied preset: pypi
✓ Policy version 3 loaded (active version: 3)  Applied preset: npm
✓ Policies applied
```

---

#### Install Complete — Summary

```
--------------------------------------------------
Sandbox      my-assistant (Landlock + seccomp + netns)
Model        nvidia/nemotron-3-super-120b-a12b (NVIDIA Endpoint API)
--------------------------------------------------
Next:
Run:         nemoclaw my-assistant connect
Status:      nemoclaw my-assistant status
Logs:        nemoclaw my-assistant logs --follow
--------------------------------------------------
=== Installation complete ===  NemoClaw  (645s)
```

### 6.5 First Connection — Verify Everything Works

```bash
# Connect to sandbox
nemoclaw my-assistant connect
# Prompt changes to: sandbox@my-assistant:~$

# Launch the TUI
sandbox@my-assistant:~$ openclaw tui
```

You will see:

```
🦞 OpenClaw 2026.3.11 (29dc654) — Shell yeah—I'm here to pinch the toil.

 openclaw tui - ws://127.0.0.1:18789 - agent main - session main

 session agent:main:main
 connected | press ctrl+c again to exit
 agent main | session main | unknown | tokens ?/131k
──────────────────────────────────────────────────────
```

Open **http://127.0.0.1:18789/** in your browser for the full web UI.

> **Note on the UNDICI warning**: When running `openclaw tui` or any openclaw command you may see `[UNDICI-EHPA] Warning: EnvHttpProxyAgent is experimental`. This is a non-fatal Node.js upstream warning. It does not affect functionality — ignore it.

---

## 7. What Got Installed — File & Directory Map

### On the Host Machine

```
~/.local/bin/
  openshell                          <- OpenShell CLI binary (v0.0.13)

~/.nvm/versions/node/v22.x.x/bin/
  nemoclaw                           <- NemoClaw CLI (linked via npm)

~/.nemoclaw/
  credentials.json  (mode 600)       <- NVIDIA API key stored here
```

### Inside the Sandbox (`/sandbox/`)

```
/sandbox/
  .openclaw/
    openclaw.json                    <- Full config: models, gateway, auth

  .openclaw-data/                    <- ALL persistent agent data
    agents/
      main/                          <- Default agent definition
    canvas/                          <- Agent scratchpad data
    cron/                            <- Scheduled job definitions
    devices/                         <- Connected device registrations
    extensions/                      <- Installed extensions
    hooks/                           <- Event hooks
    identity/                        <- Agent identity data
    skills/                          <- Installed skill modules
    update-check.json
    workspace/                       <- YOUR PRIMARY WORKING DIRECTORY
      AGENTS.md
      BOOTSTRAP.md
      CAPABILITIES.md
      HEARTBEAT.md
      IDENTITY.md
      SOUL.md                        <- Agent personality/identity layer
      TOOLS.md
      USER.md                        <- Your context/preferences

  .nemoclaw/
    blueprints/
      0.1.0/                         <- NemoClaw blueprint version
    config.json
```

### Inside the Gateway Container

```
docker container: openshell-cluster-nemoclaw
  image: ghcr.io/nvidia/openshell/cluster:0.0.13
  port:  8080 -> host 8080 (TLS)

  /var/lib/rancher/k3s/              <- k3s data
    agent/containerd/.../snapshots/  <- Sandbox filesystem layers (overlayfs)
```

> **Important**: Never access sandbox files via `docker exec` into the gateway container directly. The overlayfs snapshot number changes between restarts, so paths break silently. Always use `openshell sandbox download/upload`.

### The `openclaw.json` Config (Inside Sandbox at `~/.openclaw/openclaw.json`)

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "nvidia": {
        "baseUrl": "https://inference.local/v1",
        "apiKey": "openshell-managed",
        "api": "openai-completions",
        "models": [
          {
            "id": "nemotron-3-super-120b-a12b",
            "name": "nvidia/nemotron-3-super-120b-a12b",
            "contextWindow": 131072,
            "maxTokens": 4096
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "inference/nvidia/nemotron-3-super-120b-a12b"
      }
    }
  },
  "gateway": {
    "mode": "local",
    "controlUi": {
      "allowedOrigins": ["http://127.0.0.1:18789"],
      "allowInsecureAuth": true,
      "dangerouslyDisableDeviceAuth": true
    },
    "auth": {
      "token": "<auto-generated-hex-token>"
    },
    "trustedProxies": ["127.0.0.1", "::1"]
  }
}
```

> `baseUrl: https://inference.local/v1` is an internal gateway proxy route — not a direct call to NVIDIA servers. The gateway handles routing, rate limiting, and policy enforcement before forwarding.

---

## 8. Web UI Dashboard — http://127.0.0.1:18789/

The OpenClaw Web UI is your full visual control plane. Access it any time the sandbox is running.

Port 18789 is forwarded automatically during onboarding. If it drops after a restart:

```bash
openshell forward start 18789 my-assistant
# Access at: http://127.0.0.1:18789/
```

---

### 8.1 Accessing the Web UI — Common Issues & Fixes

#### Issue 1: Port Forward Dead After Onboarding

This is the most common post-install problem. The port forward process that onboarding started in the background can die after the terminal session that ran the installer is closed or interrupted.

**Symptom:**

```bash
$ curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/
000   # connection refused — nothing listening
```

**Diagnose with `openshell forward list`:**

```bash
$ openshell forward list
SANDBOX        BIND       PORT    PID      STATUS
my-assistant   127.0.0.1  18789   993306   dead
```

A `dead` status means the PID is gone but the record remains. The sandbox itself is still healthy — only the tunnel died.

**Fix:**

```bash
# Step 1: Remove the dead forward record
openshell forward stop 18789 my-assistant

# Step 2: Restart the forward
openshell forward start 18789 my-assistant

# Step 3: Verify
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://127.0.0.1:18789/
# HTTP Status: 200   <- dashboard is accessible
```

**Why this happens — common causes:**

| Cause | What happened |
|-------|---------------|
| Terminal closed | The shell that ran `nemoclaw onboard` was closed |
| Ctrl+C during onboarding output | Interrupted the foreground process that held the forward |
| System reboot | All forward PIDs are cleared |
| Idle timeout | Long-running SSH tunnel was reaped by the OS |

**Prevention — keep the forward alive:**

```bash
# Option A: Run in background (& detaches from terminal)
openshell forward start 18789 my-assistant &

# Option B: Keep one terminal open with the forward running in foreground
openshell forward start 18789 my-assistant
# Do not close this terminal

# Option C: Re-establish on demand — simplest approach
# Just run this any time the dashboard is unreachable:
openshell forward stop 18789 my-assistant 2>/dev/null; \
openshell forward start 18789 my-assistant
```

**Quick diagnostic checklist:**

```bash
# 1. Is the sandbox healthy?
nemoclaw my-assistant status

# 2. What is the forward status?
openshell forward list

# 3. Is anything actually listening on 18789?
ss -tlnp | grep 18789

# 4. Test the connection
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://127.0.0.1:18789/
```

---

#### Issue 2: "Disconnected from gateway. Device identity required."

This message appears in the browser at `http://127.0.0.1:18789/` even when the port is forwarded and returning HTTP 200. The page loads but shows a disconnected / auth error state instead of the dashboard.

**What it means:**

The Web UI has loaded successfully (port forward is working), but the browser cannot authenticate to the OpenClaw gateway WebSocket. The gateway requires a device identity token that the browser does not yet have. This happens when:

- You open the web UI in a new browser profile or incognito window that has no stored identity
- The browser's local storage was cleared
- You are connecting from a different machine or browser than the one that originally paired
- The auth token in `openclaw.json` changed (e.g., after re-running `nemoclaw onboard`)

**Fix — authenticate via the TUI first:**

The TUI registers a device identity automatically when you connect. Opening the web UI after a TUI session resolves the issue.

```bash
# Step 1: Connect to sandbox
nemoclaw my-assistant connect

# Step 2: Launch the TUI (this registers the device identity)
sandbox@my-assistant:~$ openclaw tui

# Step 3: While the TUI is running, open http://127.0.0.1:18789/ in your browser
# The dashboard should now load fully
```

**Fix — use the auth token directly:**

The gateway auth token is stored in the sandbox config. You can use it to authenticate the browser session directly.

```bash
# Step 1: Get the token from inside the sandbox
nemoclaw my-assistant connect
sandbox@my-assistant:~$ cat ~/.openclaw/openclaw.json | grep '"token"'
# "token": "305d7a5696033a90684440000e9aae0692885b7fe3f41ead138fb5047ef772ec"
sandbox@my-assistant:~$ exit
```

Then open the web UI URL with the token as a query parameter:

```
http://127.0.0.1:18789/?token=305d7a5696033a90684440000e9aae0692885b7fe3f41ead138fb5047ef772ec
```

Or append it directly as a URL fragment — the exact format depends on the OpenClaw version. If the query param does not work, try:

```
http://127.0.0.1:18789/#token=<your-token>
```

**Why the config has `dangerouslyDisableDeviceAuth: true`:**

Looking at `openclaw.json`, the gateway is configured with:

```json
"controlUi": {
  "allowInsecureAuth": true,
  "dangerouslyDisableDeviceAuth": true
}
```

This setting is intended to make local development easier by relaxing auth requirements. However in some alpha builds the "device identity required" message still appears on a fresh browser session regardless of this flag. The TUI-first approach above is the most reliable workaround until this is resolved upstream.

**Summary of the two-step fix for a fresh browser:**

```bash
# Terminal: ensure forward is running
openshell forward stop 18789 my-assistant 2>/dev/null
openshell forward start 18789 my-assistant

# Terminal: connect and launch TUI (registers device identity)
nemoclaw my-assistant connect
sandbox@my-assistant:~$ openclaw tui

# Browser: open http://127.0.0.1:18789/
# Dashboard should now load fully
```

---

### Control

#### Overview
Live system health dashboard. Shows:
- Active agents and current status
- Running instance count
- Active session count
- Token usage summary
- Recent activity feed — what the agent did last
- Connection status to inference backend (green = connected)

#### Channels
Communication channels the agent can send and receive through. Each is a way to interact with or control the agent outside the TUI.

| Channel | Use |
|---------|-----|
| CLI | Default — terminal input/output |
| WebSocket | Used by TUI and Web UI |
| Telegram | Bot integration for remote control from your phone |
| Slack | Notification and command channel |
| Webhook | Custom HTTP trigger for automation pipelines |

Configure Telegram here to send tasks to your agent from anywhere.

#### Instances
All running OpenClaw container instances. From here:
- View CPU, RAM, and token consumption per instance
- Start, stop, or restart instances
- View instance-level logs

#### Sessions
All active and historical conversation sessions. Each session is a continuous, persistent context window.

- Browse all sessions by agent and date
- Click any session to resume it (full history loads back into context)
- Delete old sessions to free storage
- See token counts and context window usage per session

A session ID (e.g., `dev`, `main`, `project-alpha`) maps to a persistent conversation thread. Passing the same session ID in the TUI resumes the same context.

#### Usage
Token consumption and cost tracking over time:
- Total input/output tokens per day
- Cost estimation for cloud inference
- Compute time for local inference
- Per-agent and per-session breakdown

#### Cron Jobs
Scheduled autonomous tasks — the agent runs these on a timer without you being present.

- Create jobs using standard cron syntax
- Set the target agent, session ID, and task description
- Enable/disable without deleting
- View last run result and next scheduled run time
- Monitor job history and failure logs

Example uses: daily automated report at 8am, GitHub repo check every hour, weekly dependency update PR.

---

### Agent

#### Agents
Define and manage agent personas. Each agent has:
- A name and description
- A system prompt (persona, instructions, constraints)
- An assigned inference model
- A list of enabled tools and skills
- Memory configuration and workspace path

The default agent is `main`. Create specialized agents for different roles (a `researcher` with read-only web tools, a `deployer` with CI/CD skills, etc.).

#### Skills
Reusable tool and function modules the agent can call. Skills extend built-in capabilities.

- Browse all installed skills
- Enable or disable skills per agent
- View skill source code and documentation
- Install new skills from the registry
- Write and register custom skills

Examples: `web-search`, `git-operations`, `docker-build`, `send-email`, `query-database`.

#### Nodes
Inference nodes available to the agent. Switch backends without touching the terminal.

- View all configured inference providers
- See latency and availability per node
- Switch the active model live
- Add a new provider (any OpenAI-compatible endpoint)
- Test connectivity to each node

---

### Settings

#### Config
GUI for `~/.openclaw/openclaw.json`. Changes write back to the config file.
- Change default model and context window
- Configure tool timeouts
- Update the inference route
- Manage auth tokens

#### Debug
Live debug panel for inspecting agent behavior at the reasoning level.
- Step through agent reasoning traces
- Inspect raw tool call inputs and outputs
- View the full prompt sent to the model (system + context + user message)
- See token count per reasoning step
- Replay a past step with a modified prompt

Essential when the agent produces unexpected behavior or gets stuck.

#### Logs
Filterable unified log viewer. Consolidates:
- Sandbox logs (agent actions, tool results, errors)
- Gateway logs (network requests, ALLOW/DENY decisions)
- OpenClaw runtime logs (startup, crashes, warnings)

Filter by level (debug/info/warn/error), source, time range, or free text.

---

### Resources

#### Docs
Built-in documentation reference. Covers:
- CLI command reference for `openclaw`, `openshell`, `nemoclaw`
- Policy YAML syntax reference
- Skills API documentation
- Model configuration reference
- TUI keyboard shortcuts

---

## 9. OpenShell Deep Dive

### 9.1 Full Command Reference

```bash
openshell --help

SANDBOX COMMANDS
  sandbox     Manage sandboxes
  forward     Manage port forwarding to a sandbox
  logs        View sandbox logs
  policy      Manage sandbox policy
  settings    Manage sandbox and global settings
  provider    Manage provider configuration

GATEWAY COMMANDS
  gateway     Manage the gateway lifecycle
  status      Show gateway status and information
  inference   Manage inference configuration
  doctor      Diagnose gateway issues

ADDITIONAL COMMANDS
  term        Launch the OpenShell interactive TUI
  completions Generate shell completions
  ssh-proxy   SSH proxy (used by ProxyCommand)
```

### 9.2 Sandbox Commands

```bash
openshell sandbox list
openshell sandbox connect my-assistant
openshell sandbox status my-assistant
openshell sandbox destroy my-assistant      # DANGER: permanent
```

### 9.3 Gateway Commands

```bash
openshell status                            # Health check
openshell doctor                            # Full diagnostics
openshell gateway start
openshell gateway stop
```

### 9.4 Inference Management

```bash
openshell inference list                    # List providers
openshell inference status                  # Current route
openshell inference set nvidia-nim          # Switch provider
```

### 9.5 The Terminal Approval Dashboard (`openshell term`)

Run this in a dedicated terminal or tmux pane. Shows every pending tool request the agent makes in real time.

```bash
openshell term
```

Keep this open in a separate pane at all times during active work sessions. Every agent action that touches the network or filesystem appears here for your approval.

### 9.6 Log Sources

```bash
# Agent actions
openshell logs my-assistant --tail --source sandbox

# Network policy enforcement
openshell logs my-assistant --tail --source gateway

# All combined
openshell logs my-assistant --tail --source all

# Filter denials
openshell logs my-assistant --source gateway | grep DENY
```

---

## 10. NemoClaw Deep Dive

### 10.1 Sandbox Name is `my-assistant`

Every NemoClaw command uses the sandbox name as a positional subcommand:

```bash
nemoclaw my-assistant connect
nemoclaw my-assistant status
nemoclaw my-assistant logs --follow
nemoclaw my-assistant destroy
```

### 10.2 Full Command Reference

```bash
# Lifecycle
nemoclaw list
nemoclaw my-assistant status
nemoclaw my-assistant connect
nemoclaw my-assistant logs
nemoclaw my-assistant logs --follow
nemoclaw my-assistant destroy            # DANGER

# Configuration
nemoclaw onboard                         # Re-run full wizard
nemoclaw my-assistant config show
nemoclaw my-assistant config edit

# Policy
nemoclaw my-assistant policy-list
nemoclaw my-assistant policy-add <name>
nemoclaw my-assistant policy-remove <name>

# Services
nemoclaw start                           # Telegram bot + tunnel
nemoclaw stop
```

---

## 11. Policy System — Full Reference

### 11.1 Available Preset Policies

```bash
# Add multiple presets at once
nemoclaw my-assistant policy-add pypi npm github huggingface docker

# Full preset list:
# pypi         Python Package Index                 [default]
# npm          npm and Yarn registry                [default]
# discord      Discord API, gateway, CDN
# docker       Docker Hub, NVIDIA container registry
# huggingface  HuggingFace Hub, LFS, Inference API
# jira         Jira and Atlassian Cloud
# outlook      Microsoft Outlook and Graph API
# slack        Slack API and webhooks
# telegram     Telegram Bot API
```

### 11.2 View Current Policies

```bash
nemoclaw my-assistant policy-list

openshell policy get my-assistant --full

openshell policy get my-assistant --full > ~/current-policy.yaml
```

### 11.3 Custom Policy YAML — Full Structure

```yaml
version: 1

# Network Policies
network_policies:

  github_full:
    name: github
    endpoints:
      - host: api.github.com
        port: 443
        protocol: rest
        tls: terminate
        rules:
          - allow: { method: "*", path: "/**" }
    binaries:
      - { path: /usr/bin/gh }
      - { path: /usr/local/bin/git }

  huggingface:
    name: huggingface
    endpoints:
      - host: huggingface.co
        port: 443
        protocol: rest
        tls: terminate
        rules:
          - allow: { method: "GET", path: "/**" }
      - host: cdn-lfs.huggingface.co
        port: 443
        protocol: rest
        tls: terminate
        rules:
          - allow: { method: "GET", path: "/**" }

  slack:
    name: slack
    endpoints:
      - host: hooks.slack.com
        port: 443
        protocol: rest
        tls: terminate
        rules:
          - allow: { method: "POST", path: "/services/**" }

  custom_api:
    name: my_backend
    endpoints:
      - host: api.mycompany.com
        port: 443
        protocol: rest
        tls: terminate
        rules:
          - allow: { method: "*", path: "/v1/**" }
          - deny:  { method: "DELETE", path: "/v1/users/**" }

# Filesystem Policies
filesystem_policies:
  workspace:
    paths:
      - path: /sandbox/.openclaw-data/workspace
        permissions: [read, write, create, delete]
  tmp:
    paths:
      - path: /tmp
        permissions: [read, write, create, delete]

# Process / Binary Policies
process_policies:
  allowed_binaries:
    - { path: /usr/bin/python3 }
    - { path: /usr/local/bin/node }
    - { path: /usr/bin/bash }
    - { path: /usr/bin/curl }
    - { path: /usr/local/bin/git }
```

### 11.4 Apply Custom Policy

```bash
openshell policy set my-assistant --policy ~/my-policy.yaml --wait
# --wait blocks until policy is active and verified
# Policy version number increments on each apply
```

### 11.5 Policy Debug Loop (When Agent Gets Blocked)

```bash
# 1. See what got denied
openshell logs my-assistant --source gateway | grep DENY | tail -20

# 2. Pull current policy
openshell policy get my-assistant --full > ~/current.yaml

# 3. Edit to allow the blocked endpoint
nano ~/current.yaml

# 4. Apply
openshell policy set my-assistant --policy ~/current.yaml --wait
```

---

## 12. Inference & Model Configuration

### 12.1 Available Cloud Models

| Model | ID | Context Window | Notes |
|-------|----|---------------|-------|
| Nemotron 3 Super 120B | `nvidia/nemotron-3-super-120b-a12b` | 131k | Default. Best balance |
| Kimi K2.5 | `moonshotai/kimi-k2.5` | varies | Long context tasks |
| GLM-5 | `z-ai/glm5` | varies | Multilingual |
| MiniMax M2.5 | `minimaxai/minimax-m2.5` | varies | Fast general tasks |
| Qwen3.5 397B A17B | `qwen/qwen3.5-397b-a17b` | varies | Large-scale reasoning |
| GPT-OSS 120B | `openai/gpt-oss-120b` | varies | OpenAI-compatible |

### 12.2 Switch Model (No Destroy Needed)

```bash
nemoclaw onboard
# Navigate step [4/7] and choose a different model
# Workspace data is fully preserved
```

### 12.3 Switch to Local Ollama

```bash
# 1. Ensure Ollama is running with a model
ollama serve &
ollama pull llama3.1:8b

# 2. Re-run onboard
nemoclaw onboard
# Choose: 2) Local Ollama (localhost:11434)
```

### 12.4 Inference Route (Internal)

Inside the sandbox, `openclaw.json` always points to `https://inference.local/v1`. This internal address is proxied by the OpenShell gateway — routing to either NVIDIA cloud or local Ollama depending on your config. You never need to change this URL manually.

---

## 13. File & Workspace Management

### 13.1 Workspace Map

```
INSIDE SANDBOX (survives restart, lost on destroy):
  /sandbox/.openclaw-data/workspace/
    SOUL.md       <- agent identity
    USER.md       <- your preferences and project context
    AGENTS.md
    BOOTSTRAP.md
    CAPABILITIES.md
    HEARTBEAT.md
    IDENTITY.md
    TOOLS.md

HOST MIRROR (safe, put under git):
  ~/my-assistant-workspace/
```

### 13.2 Download Workspace (Sandbox -> Host)

```bash
openshell sandbox download my-assistant \
  /sandbox/.openclaw-data/workspace \
  ~/my-assistant-workspace

# Single file
openshell sandbox download my-assistant \
  /sandbox/.openclaw-data/workspace/SOUL.md \
  ~/my-assistant-workspace/SOUL.md
```

### 13.3 Upload Changes (Host -> Sandbox)

```bash
# Single file
openshell sandbox upload my-assistant \
  ~/my-assistant-workspace/USER.md \
  /sandbox/.openclaw-data/workspace/USER.md

# Full workspace sync
openshell sandbox upload my-assistant \
  ~/my-assistant-workspace \
  /sandbox/.openclaw-data/workspace
```

> **Never use `docker cp` directly.** The k3s overlayfs snapshot number in the path changes between restarts and will silently break transfers.

### 13.4 Version Control Your Workspace

```bash
cd ~/my-assistant-workspace
git init
git add .
git commit -m "initial workspace"

# Optional remote
git remote add origin git@github.com:yourname/my-assistant-workspace.git
git push -u origin main
```

### 13.5 Writing SOUL.md

```markdown
# Agent Identity

## Role
I am a senior software engineering agent specializing in backend systems.

## Working Principles
- Write tests before implementation
- Never commit secrets or credentials
- Confirm before irreversible actions (file deletion, DB migrations)
- Ask for clarification when requirements are ambiguous

## Current Project Context
Building FastAPI + PostgreSQL REST API for Project Alpha.
See projects/alpha/ for full context.
```

### 13.6 Writing USER.md

```markdown
# User Context

## Preferences
- Language: Python 3.11 or TypeScript
- Tests: pytest with 80%+ coverage
- Style: functional, well-typed, minimal dependencies

## Active Tasks
1. Implement JWT auth (see auth-spec.md)
2. Write DB migration scripts for users table

## Notes
- Never push to main — always create a PR
```

---

## 14. Networking & Port Forwarding

### 14.1 Port 18789 — Web UI

Forwarded automatically during onboarding. If it drops:

```bash
openshell forward start 18789 my-assistant
# Access at: http://127.0.0.1:18789/

openshell forward stop 18789 my-assistant
```

### 14.2 Forwarding Application Ports

```bash
openshell forward start 3000 my-assistant    # React dev server
openshell forward start 8000 my-assistant    # FastAPI
openshell forward start 5173 my-assistant    # Vite

# Custom host port
openshell forward start 8000 my-assistant --host-port 9000

openshell forward list
openshell forward stop 3000 my-assistant
```

### 14.3 Reaching Host Services from Inside the Sandbox

```bash
# From inside sandbox — use this hostname to reach the host
curl http://host.docker.internal:5432
```

Add to policy YAML if the agent needs to access host services:

```yaml
network_policies:
  host_db:
    name: host_services
    endpoints:
      - host: host.docker.internal
        port: 5432
        protocol: tcp
```

---

## 15. Multi-Sandbox & Multi-Agent Setup

### 15.1 Create Additional Sandboxes

```bash
nemoclaw onboard
# Enter a different sandbox name when prompted: researcher
```

```bash
nemoclaw list
# my-assistant    running
# researcher      running
```

### 15.2 Connect to Any Sandbox

```bash
nemoclaw my-assistant connect
nemoclaw researcher connect
```

### 15.3 Role-Based Sandbox Setup

| Sandbox | Role | Model |
|---------|------|-------|
| `my-assistant` | Primary dev agent | Nemotron 3 Super 120B |
| `researcher` | Web research, analysis | Qwen3.5 397B |
| `worker` | Automated cron tasks | Kimi K2.5 |

Each sandbox has independent policies, workspace, sessions, and model config.

---

## 16. Daily Operational Workflow

### 16.1 Startup Sequence

```bash
# 1. Health check
nemoclaw my-assistant status
openshell status

# 2. Open approval dashboard in a separate terminal
openshell term

# 3. Confirm web UI is forwarded
openshell forward list   # 18789 must be listed
# If not: openshell forward start 18789 my-assistant

# 4. Connect and launch
nemoclaw my-assistant connect
sandbox@my-assistant:~$ openclaw tui
```

### 16.2 Working with Long Tasks (CLI Mode)

For long-running tasks, CLI mode is more reliable than TUI:

```bash
sandbox@my-assistant:~$ openclaw agent \
  --agent main \
  --local \
  --session-id dev \
  -m "Build a FastAPI REST API with JWT auth as described in USER.md"
```

### 16.3 Resuming a Session

```bash
sandbox@my-assistant:~$ openclaw agent \
  --session-id dev \
  -m "Continue. What is the current state of the auth module?"
```

### 16.4 End-of-Day Sync

```bash
sandbox@my-assistant:~$ exit

openshell sandbox download my-assistant \
  /sandbox/.openclaw-data/workspace \
  ~/my-assistant-workspace

cd ~/my-assistant-workspace
git add -A
git commit -m "session: $(date +%F)"
git push
```

---

## 17. Monitoring, Logging & Observability

### 17.1 Status Commands

```bash
nemoclaw my-assistant status
openshell status
docker ps | grep openshell
```

### 17.2 Log Reference

| Source | Command | What it shows |
|--------|---------|---------------|
| Sandbox | `--source sandbox` | Agent actions, tool calls |
| Gateway | `--source gateway` | Network ALLOW/DENY |
| All | `--source all` | Everything |

```bash
openshell logs my-assistant --tail --source all
openshell logs my-assistant --source gateway | grep DENY
openshell logs my-assistant --source all > ~/logs/$(date +%F).log
```

### 17.3 Reading a Policy Denial Log Entry

```
2026-03-23T10:22:41Z [GATEWAY] DENY endpoint=api.stripe.com:443
  method=POST path=/v1/charges reason=no_matching_policy sandbox=my-assistant
```

The agent tried to POST to `api.stripe.com/v1/charges` — no policy covers it. Fix: add a Stripe rule to your policy YAML and re-apply.

---

## 18. Backup, Restore & Disaster Recovery

### 18.1 Backup Script

```bash
cat > ~/scripts/backup-my-assistant.sh << 'EOF'
#!/bin/bash
DATE=$(date +%F-%H%M)
DEST="$HOME/backups/my-assistant/backup-$DATE"
mkdir -p "$DEST"

echo "Backing up workspace..."
openshell sandbox download my-assistant \
  /sandbox/.openclaw-data/workspace "$DEST/workspace"

echo "Saving policy..."
openshell policy get my-assistant --full > "$DEST/policy.yaml"

echo "Backup complete: $DEST"
echo "Size: $(du -sh $DEST | cut -f1)"
EOF

chmod +x ~/scripts/backup-my-assistant.sh
~/scripts/backup-my-assistant.sh
```

### 18.2 Automate Daily Backup

```bash
crontab -e
# Add:
30 23 * * * /bin/bash ~/scripts/backup-my-assistant.sh >> ~/logs/backup.log 2>&1
```

### 18.3 Restore

```bash
# Sandbox still exists — restore workspace files
openshell sandbox upload my-assistant \
  ~/backups/my-assistant/backup-LATEST/workspace \
  /sandbox/.openclaw-data/workspace

# Sandbox was destroyed — full rebuild
nemoclaw onboard                      # creates fresh my-assistant
openshell sandbox upload my-assistant \
  ~/backups/my-assistant/backup-LATEST/workspace \
  /sandbox/.openclaw-data/workspace
openshell policy set my-assistant \
  --policy ~/backups/my-assistant/backup-LATEST/policy.yaml --wait
```

---

## 19. Troubleshooting Reference

### `command not found: openshell` or `command not found: nemoclaw`

```bash
export PATH="$HOME/.local/bin:$PATH"
source ~/.bashrc
export NVM_DIR="$HOME/.nvm" && [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

### Onboard Fails at Preflight — Docker Not Running

```bash
sudo systemctl start docker
docker ps   # must return table header, not permission error
```

### Port 8080 or 18789 Already in Use

```bash
sudo lsof -i :8080
sudo lsof -i :18789
sudo kill -9 <PID>
```

### `bash: line 9: BASH_SOURCE[0]: unbound variable`

Non-fatal alpha bug in the installer script. Installation continues normally. No action needed.

### `[UNDICI-EHPA] Warning: EnvHttpProxyAgent is experimental`

Non-fatal Node.js warning from the OpenClaw runtime. Does not affect functionality. No action needed.

### `openclaw gateway` Shows Port Already in Use

```
Gateway failed to start: gateway already running (pid 63); lock timeout after 5000ms
```

The gateway is already running under supervision — this is the correct state. Do not try to start it again manually. If you genuinely need to restart:

```bash
sandbox@my-assistant:~$ openclaw gateway stop
sandbox@my-assistant:~$ openclaw gateway start
```

### `client_loop: send disconnect: Broken pipe`

The sandbox SSH connection timed out. Simply reconnect:

```bash
nemoclaw my-assistant connect
```

### Agent Lost Memory / Empty Workspace

```bash
nemoclaw my-assistant connect
sandbox@my-assistant:~$ ls ~/.openclaw-data/workspace/
sandbox@my-assistant:~$ exit

# Restore from backup
openshell sandbox upload my-assistant \
  ~/backups/my-assistant/backup-LATEST/workspace \
  /sandbox/.openclaw-data/workspace
```

### Sandbox Not Responding — Full Reset

```bash
docker ps | grep openshell       # verify gateway container is up
openshell gateway stop
sleep 3
openshell gateway start
nemoclaw my-assistant connect
```

### Web UI at 127.0.0.1:18789 Returns Connection Refused (HTTP 000)

The port forward died. Check its status and restart it:

```bash
# Diagnose
openshell forward list
# Look for: my-assistant  127.0.0.1  18789  <PID>  dead

# Fix
openshell forward stop 18789 my-assistant
openshell forward start 18789 my-assistant

# Verify
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://127.0.0.1:18789/
# Must return: HTTP Status: 200
```

See §8.1 for full diagnosis, causes, and prevention options.

### Web UI Shows "Disconnected from gateway. Device identity required."

The page loaded (HTTP 200) but the browser cannot authenticate to the gateway WebSocket.

```bash
# Fix: launch the TUI first — it registers the device identity
nemoclaw my-assistant connect
sandbox@my-assistant:~$ openclaw tui
# Now open http://127.0.0.1:18789/ in your browser
```

If the TUI is not convenient, get the auth token and pass it in the URL:

```bash
nemoclaw my-assistant connect
sandbox@my-assistant:~$ cat ~/.openclaw/openclaw.json | grep '"token"'
sandbox@my-assistant:~$ exit
# Open: http://127.0.0.1:18789/?token=<token-value>
```

See §8.1 for full explanation and the two-step fix.

### Re-run Onboard to Fix Any Config Issue

```bash
nemoclaw onboard
# Use same sandbox name: my-assistant
# Workspace data is preserved
```

---

## 20. Security Hardening

### 20.1 Least-Privilege Policy

Start with only `pypi` and `npm`. Add more only when the agent actually gets denied.

```bash
# Find denials reactively
openshell logs my-assistant --source gateway | grep DENY
# Add only what is needed
```

### 20.2 Protect the API Key

The key is stored at `~/.nemoclaw/credentials.json` with mode 600 automatically by the installer.

```bash
ls -la ~/.nemoclaw/credentials.json
# Must show: -rw------- (600)

echo "credentials.json" >> ~/.gitignore
echo "nvapi-*" >> ~/.gitignore
```

### 20.3 Audit Network Access

```bash
openshell logs my-assistant --source gateway | grep ALLOW | \
  awk '{print $5}' | sort | uniq -c | sort -rn | head -20
```

### 20.4 Keep `openshell term` Open

The approval dashboard is your real-time veto over every agent network and filesystem action. Run it every session.

```bash
openshell term   # dedicated terminal pane
```

---

## 21. Full Uninstall

### Step 1 — Backup

```bash
~/scripts/backup-my-assistant.sh
```

### Step 2 — Destroy Sandboxes

```bash
nemoclaw list
nemoclaw my-assistant destroy
# repeat for any other sandboxes
```

### Step 3 — Uninstall NemoClaw

```bash
curl -fsSL https://raw.githubusercontent.com/NVIDIA/NemoClaw/refs/heads/main/uninstall.sh | \
  bash -s -- --yes
```

### Step 4 — Remove OpenShell

```bash
rm -f ~/.local/bin/openshell
openshell gateway stop 2>/dev/null || true
sudo /usr/local/bin/k3s-uninstall.sh 2>/dev/null || true
```

### Step 5 — Clean Up

```bash
rm -rf ~/.nemoclaw/
rm -rf ~/.openclaw/
# Remove PATH additions from ~/.bashrc manually
```

---

## 22. Quick Reference Cheat Sheet

### Host Commands

```bash
# NemoClaw
nemoclaw list
nemoclaw my-assistant status
nemoclaw my-assistant connect
nemoclaw my-assistant logs --follow
nemoclaw my-assistant destroy                  # DANGER
nemoclaw my-assistant policy-list
nemoclaw my-assistant policy-add github
nemoclaw onboard                               # re-configure

# OpenShell — Gateway
openshell status
openshell gateway start / stop
openshell doctor
openshell term                                 # approval dashboard -- keep open

# OpenShell — Sandbox
openshell sandbox list
openshell sandbox connect my-assistant
openshell sandbox status my-assistant

# OpenShell — Files
openshell sandbox download my-assistant \
  /sandbox/.openclaw-data/workspace ~/my-assistant-workspace
openshell sandbox upload my-assistant \
  ~/my-assistant-workspace /sandbox/.openclaw-data/workspace

# OpenShell — Policy
openshell policy get my-assistant --full > ~/current.yaml
openshell policy set my-assistant --policy ~/my-policy.yaml --wait

# OpenShell — Logs
openshell logs my-assistant --tail --source sandbox
openshell logs my-assistant --tail --source gateway
openshell logs my-assistant --tail --source all

# OpenShell — Ports
openshell forward start 18789 my-assistant    # web UI
openshell forward start 3000 my-assistant     # app port
openshell forward list
openshell forward stop 3000 my-assistant
```

### Inside Sandbox (`sandbox@my-assistant:~$`)

```bash
openclaw tui                                  # interactive UI

openclaw agent \
  --agent main \
  --local \
  --session-id dev \
  -m "your task here"                         # CLI agent

openclaw agent \
  --session-id dev \
  -m "continue"                               # resume session

openclaw sessions list
openclaw gateway status
```

### Web UI — http://127.0.0.1:18789/

| Section | Path | Purpose |
|---------|------|---------|
| Overview | Control > Overview | Live health and activity feed |
| Channels | Control > Channels | Telegram, Slack, webhook config |
| Instances | Control > Instances | Running instance management |
| Sessions | Control > Sessions | Browse and resume sessions |
| Usage | Control > Usage | Token and cost tracking |
| Cron Jobs | Control > Cron Jobs | Schedule autonomous agent tasks |
| Agents | Agent > Agents | Edit system prompts and tool access |
| Skills | Agent > Skills | Install and enable capabilities |
| Nodes | Agent > Nodes | Switch inference model live |
| Config | Settings > Config | Edit openclaw.json via GUI |
| Debug | Settings > Debug | Inspect reasoning traces |
| Logs | Settings > Logs | Unified filterable log viewer |
| Docs | Resources > Docs | Built-in reference |

### Key File Locations

| File | Location | Purpose |
|------|----------|---------|
| OpenClaw config | `/sandbox/.openclaw/openclaw.json` | Models, gateway, auth |
| Workspace | `/sandbox/.openclaw-data/workspace/` | All persistent agent files |
| SOUL.md | `workspace/SOUL.md` | Agent identity layer |
| USER.md | `workspace/USER.md` | Your project context |
| API key | `~/.nemoclaw/credentials.json` | NVIDIA key (host, mode 600) |
| OpenShell binary | `~/.local/bin/openshell` | CLI (host) |
| NemoClaw binary | `~/.nvm/.../bin/nemoclaw` | CLI (host) |

---

> **Alpha Warning**: NemoClaw v0.1.0 and OpenShell v0.0.13 are in active alpha development. Commands and YAML schemas may change between releases. Always back up before `destroy` or re-running `onboard`.
>
> - GitHub: https://github.com/nvidia/nemoclaw
> - Docs: https://docs.nvidia.com/nemoclaw/latest/
> - OpenShell: https://docs.nvidia.com/openshell/latest/
