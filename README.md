# openclaw - NemoClaw + OpenShell — Complete Manual
 
**Last Updated**: March 23, 2026  
**Scope**: Full installation, configuration, and optimal operational use

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites & System Requirements](#2-prerequisites--system-requirements)
3. [Pre-Node Installation Steps](#3-pre-node-installation-steps)
4. [Installing Core Dependencies](#4-installing-core-dependencies)
5. [Installing OpenShell](#5-installing-openshell)
6. [Installing NemoClaw](#6-installing-nemoclaw)
7. [Onboarding & First Configuration](#7-onboarding--first-configuration)
8. [OpenShell Deep Dive](#8-openshell-deep-dive)
9. [NemoClaw Deep Dive](#9-nemoclaw-deep-dive)
10. [Policy System — Full Reference](#10-policy-system--full-reference)
11. [Inference & Model Configuration](#11-inference--model-configuration)
12. [File & Workspace Management](#12-file--workspace-management)
13. [Networking & Port Forwarding](#13-networking--port-forwarding)
14. [Multi-Sandbox & Multi-Agent Setup](#14-multi-sandbox--multi-agent-setup)
15. [Daily Operational Workflow](#15-daily-operational-workflow)
16. [Monitoring, Logging & Observability](#16-monitoring-logging--observability)
17. [Backup, Restore & Disaster Recovery](#17-backup-restore--disaster-recovery)
18. [Troubleshooting Reference](#18-troubleshooting-reference)
19. [Security Hardening](#19-security-hardening)
20. [Full Uninstall](#20-full-uninstall)
21. [Quick Reference Cheat Sheet](#21-quick-reference-cheat-sheet)

---

## 1. Architecture Overview

Understanding the full stack before touching a terminal is mandatory. Every tool has a specific layer responsibility.

```
┌─────────────────────────────────────────────────────────────────┐
│                          HOST MACHINE                           │
│                                                                 │
│   ┌──────────────┐    ┌──────────────────────────────────────┐  │
│   │  NemoClaw    │    │           OpenShell CLI              │  │
│   │  (Orchestr.) │    │   (Sandbox runtime control plane)    │  │
│   └──────┬───────┘    └──────────────┬───────────────────────┘  │
│          │                           │                           │
│          └──────────────┬────────────┘                           │
│                         │                                        │
│              ┌──────────▼──────────┐                             │
│              │  OpenShell Gateway  │                             │
│              │  (k3s + netns +     │                             │
│              │   seccomp + Landlock│                             │
│              └──────────┬──────────┘                             │
│                         │                                        │
│         ┌───────────────▼──────────────────┐                     │
│         │           SANDBOX (nemo)          │                    │
│         │                                  │                     │
│         │  ┌──────────────────────────┐    │                     │
│         │  │   OpenClaw Agent (TUI)   │    │                     │
│         │  │   /sandbox/.openclaw-    │    │                     │
│         │  │    data/workspace/       │    │                     │
│         │  └──────────────────────────┘    │                     │
│         │                                  │                     │
│         │  Policy-gated network egress     │                     │
│         └──────────┬───────────────────────┘                     │
│                    │                                             │
└────────────────────┼─────────────────────────────────────────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │  NVIDIA Nemotron API  │
         │  (or Local Ollama)    │
         └───────────────────────┘
```

| Component | Layer | Responsibility |
|-----------|-------|----------------|
| **NemoClaw** | Orchestration | One-command setup, lifecycle management, policy presets |
| **OpenShell** | Runtime | Kernel-level sandbox via Landlock + seccomp + netns + k3s |
| **OpenClaw** | Agent | The actual AI agent; runs inside the sandbox |
| **Nemotron API** | Inference | Cloud LLM backend (or Ollama for local) |

**Data flow**: NemoClaw configures → OpenShell enforces → OpenClaw acts → Nemotron infers.

---

## 2. Prerequisites & System Requirements

### 2.1 Hardware Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 4 cores x86_64 | 8+ cores |
| RAM | 8 GB | 16–32 GB |
| Disk | 20 GB free | 50+ GB SSD |
| GPU | None (cloud inference) | NVIDIA GPU (local inference) |
| OS | Ubuntu 22.04 LTS | Ubuntu 22.04 / 24.04 LTS |
| Kernel | 5.13+ (Landlock support) | 6.x |

> **Note for local inference (Ollama)**: Minimum 16 GB RAM and an NVIDIA GPU with 8+ GB VRAM for any usable model size.

### 2.2 Software Prerequisites

- Ubuntu 22.04 LTS or 24.04 LTS (Debian-based distros may work with adaptation)
- User account with `sudo` privileges
- Active internet connection
- An NVIDIA API key (`nvapi-...`) if using cloud inference

### 2.3 Verify Kernel Version (Landlock Requirement)

```bash
uname -r
# Must be 5.13 or higher for Landlock support
```

If below 5.13, upgrade your kernel before proceeding:

```bash
sudo apt update && sudo apt install --install-recommends linux-generic-hwe-22.04 -y
sudo reboot
```

---

## 3. Pre-Node Installation Steps

These steps must be completed before installing OpenShell or NemoClaw.

### 3.1 Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### 3.2 Install Essential Build Tools

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

### 3.3 Install and Configure Docker

OpenShell uses container primitives under the hood. Docker must be present.

```bash
# Remove old/conflicting Docker versions
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null || true

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add current user to docker group (avoids sudo on every docker command)
sudo usermod -aG docker $USER

# Enable Docker to start on boot
sudo systemctl enable docker
sudo systemctl start docker

# Verify installation
docker --version
docker run hello-world
```

> **Important**: Log out and back in (or run `newgrp docker`) after adding yourself to the docker group.

### 3.4 Install NVIDIA Container Toolkit (GPU Users Only)

Skip if using cloud inference exclusively.

```bash
# Add NVIDIA container toolkit repository
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit

# Configure Docker to use NVIDIA runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Verify GPU is accessible in containers
docker run --rm --gpus all nvidia/cuda:12.0-base-ubuntu22.04 nvidia-smi
```

### 3.5 Install kubectl (k3s management)

OpenShell runs on k3s internally; kubectl is needed for advanced operations.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### 3.6 Verify Landlock Support

```bash
# Check if Landlock is enabled in the kernel
grep -r LANDLOCK /boot/config-$(uname -r) 2>/dev/null | grep -v "^#"
# Should show: CONFIG_SECURITY_LANDLOCK=y

# Alternative check
cat /sys/kernel/security/lsm | grep landlock
# Should include "landlock" in the output
```

### 3.7 Set Up Directory Structure

```bash
# Create working directories
mkdir -p ~/nemo-workspace
mkdir -p ~/.openclaw
mkdir -p ~/backups/nemo

# Set permissions
chmod 700 ~/.openclaw
```

### 3.8 Configure Environment Variables (Pre-Install)

```bash
# Add to ~/.bashrc for persistence
cat >> ~/.bashrc << 'EOF'

# NemoClaw + OpenShell Environment
export PATH="$HOME/.local/bin:$PATH"
export NEMOCLAW_WORKSPACE="$HOME/nemo-workspace"
export NEMOCLAW_BACKUP_DIR="$HOME/backups/nemo"
EOF

source ~/.bashrc
```

---

## 4. Installing Core Dependencies

### 4.1 Install Node.js (Required by OpenClaw tooling)

```bash
# Install nvm (Node Version Manager) — preferred method
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Load nvm without restarting shell
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# Install Node.js LTS
nvm install --lts
nvm use --lts
nvm alias default 'lts/*'

# Verify
node --version   # Should show v20.x or later
npm --version
```

### 4.2 Install Python 3.11+ (Used by OpenShell policy scripts)

```bash
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
sudo apt install -y python3.11 python3.11-venv python3.11-dev python3-pip

# Set Python 3.11 as default python3
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.11 1

# Verify
python3 --version
pip3 --version
```

### 4.3 Install Rust (Required by some OpenShell components)

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
rustc --version
```

### 4.4 Install GitHub CLI (Recommended for workspace git operations)

```bash
type -p curl >/dev/null || sudo apt install curl -y
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | \
  sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] \
  https://cli.github.com/packages stable main" | \
  sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install gh -y

gh --version
```

---

## 5. Installing OpenShell

### 5.1 Run the OpenShell Installer

```bash
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
```

### 5.2 Fix PATH (Critical Step)

```bash
export PATH="$HOME/.local/bin:$PATH"
source ~/.bashrc

# Verify
openshell --version
```

### 5.3 Initialize OpenShell Gateway

The gateway is the network proxy and policy enforcement point between sandboxes and the internet.

```bash
openshell gateway init
openshell gateway start

# Verify gateway is running
openshell gateway status
```

### 5.4 Verify OpenShell Installation

```bash
# List available commands
openshell --help

# List sandboxes (should be empty at this point)
openshell sandbox list

# Start the terminal approval dashboard (TUI for approving/denying agent actions)
openshell term
# Press Ctrl+C to exit for now — we'll come back to this
```

---

## 6. Installing NemoClaw

### 6.1 Run the NemoClaw Installer

```bash
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash
```

This single command:
- Installs the NemoClaw CLI
- Pulls the OpenClaw agent container
- Sets up the default sandbox configuration templates
- Integrates with the OpenShell gateway

### 6.2 Verify PATH Again After NemoClaw Install

```bash
export PATH="$HOME/.local/bin:$PATH"
source ~/.bashrc

nemoclaw --version
nemoclaw --help
```

### 6.3 Verify Both Tools Are Accessible

```bash
which openshell    # Should return a path
which nemoclaw     # Should return a path

openshell --version
nemoclaw --version
```

---

## 7. Onboarding & First Configuration

### 7.1 Obtain NVIDIA API Key

Before running onboard:

1. Go to [https://build.nvidia.com](https://build.nvidia.com)
2. Sign in or create an account
3. Navigate to **API Keys** → **Generate Key**
4. Copy the key starting with `nvapi-...`
5. Store it securely: `echo "nvapi-YOUR_KEY" > ~/.openclaw/nvapi.key && chmod 600 ~/.openclaw/nvapi.key`

### 7.2 Run the Onboard Wizard

```bash
nemoclaw onboard
```

**Answer each prompt optimally:**

```
? Select inference backend:
  > 1) NVIDIA Endpoint API     ← CHOOSE THIS for best performance
    2) Local Ollama
    3) Custom endpoint

? Paste your NVIDIA API key:
  nvapi-xxxxxxxxxxxxxxxxxxxx

? Select model:
  > 1) Nemotron 3 Super 120B   ← CHOOSE THIS (best quality/cost ratio)
    2) Nemotron 3 Nano 8B      (fast, lower quality)
    3) Nemotron 3 Ultra 253B   (max quality, slower)

? Accept default policies? (pypi + npm included) [y/N]:
  y

? Sandbox name [nemo]:
  nemo                         ← Press Enter to accept default
```

### 7.3 Verify Onboarding Completed

```bash
nemoclaw list
# Should show a sandbox named "nemo" with status: running

nemoclaw nemo status
# Shows: model, health, uptime, policy count
```

### 7.4 Connect to Your Sandbox for the First Time

```bash
# SSH into the sandbox
nemoclaw nemo connect

# You are now inside the sandbox. Your prompt should show:
# sandbox@nemo:~$

# Launch the interactive TUI
sandbox@nemo:~$ openclaw tui

# OR use direct CLI chat
sandbox@nemo:~$ openclaw agent --agent main --local -m "Hello, who are you?" --session-id test

# Exit the sandbox
sandbox@nemo:~$ exit
```

---

## 8. OpenShell Deep Dive

### 8.1 Core Concepts

OpenShell enforces security at the **kernel level** using:

| Mechanism | What it does |
|-----------|-------------|
| **Landlock** | File system access restrictions — agent can only touch allowed paths |
| **seccomp** | System call filtering — blocks dangerous syscalls |
| **netns** | Network namespace isolation — agent uses its own network stack |
| **k3s** | Lightweight Kubernetes — orchestrates sandbox containers |

### 8.2 Sandbox Lifecycle Commands

```bash
# List all sandboxes
openshell sandbox list

# Connect to a sandbox (interactive shell)
openshell sandbox connect nemo

# Check sandbox status and resource usage
openshell sandbox status nemo

# Pause a sandbox (preserves state, stops compute)
openshell sandbox pause nemo

# Resume a paused sandbox
openshell sandbox resume nemo

# Hard restart (keeps data, restarts container)
openshell sandbox restart nemo

# Destroy sandbox permanently (data is LOST)
openshell sandbox destroy nemo
```

### 8.3 The Terminal Approval Dashboard (`openshell term`)

This is the most important operational tool. Run it in a separate terminal at all times.

```bash
openshell term
```

The dashboard shows:
- All pending agent tool requests (file reads, shell commands, network calls)
- Real-time approval/deny interface
- Historical log of approved/denied actions
- Active policy violations

**Key dashboard shortcuts:**

| Key | Action |
|-----|--------|
| `a` | Approve request |
| `d` | Deny request |
| `A` | Approve all pending |
| `l` | View full logs |
| `p` | View policy summary |
| `q` | Quit |

**Best practice**: Run `openshell term &` in background or in a dedicated terminal pane/tmux window.

### 8.4 Logging & Log Sources

```bash
# Follow live logs from sandbox
openshell logs nemo --tail --source sandbox

# Follow gateway logs (network policy enforcement)
openshell logs nemo --tail --source gateway

# Follow all logs combined
openshell logs nemo --tail --source all

# Filter logs by severity
openshell logs nemo --tail --level warn

# Save logs to file
openshell logs nemo --source sandbox > ~/logs/nemo-$(date +%F).log
```

### 8.5 File Transfer (Host ↔ Sandbox)

**Download from sandbox to host:**

```bash
# Download entire workspace
openshell sandbox download nemo /sandbox/.openclaw-data/workspace ~/nemo-workspace

# Download a single file
openshell sandbox download nemo /sandbox/.openclaw-data/workspace/SOUL.md ~/SOUL.md

# Download with verbose output
openshell sandbox download nemo /sandbox/.openclaw-data/workspace ~/nemo-workspace --verbose
```

**Upload from host to sandbox:**

```bash
# Upload a single file
openshell sandbox upload nemo ~/nemo-workspace/SOUL.md /sandbox/.openclaw-data/workspace/SOUL.md

# Upload entire folder (overwrites destination)
openshell sandbox upload nemo ~/nemo-workspace /sandbox/.openclaw-data/workspace

# Upload with progress
openshell sandbox upload nemo ~/nemo-workspace /sandbox/.openclaw-data/workspace --progress
```

> **Warning**: Never use `docker cp` directly — snapshot numbers change between restarts and will break your transfers silently. Always use `openshell sandbox download/upload`.

### 8.6 Port Forwarding

Expose sandbox services to your host:

```bash
# Forward sandbox port 3000 to host port 3000
openshell forward start 3000 nemo

# Forward with custom host port
openshell forward start 3000 nemo --host-port 8080

# List active forwards
openshell forward list

# Stop a forward
openshell forward stop 3000 nemo
```

### 8.7 Editor Integration

```bash
# Open VS Code with remote connection to sandbox
openshell sandbox connect nemo --editor code

# Open Cursor IDE
openshell sandbox connect nemo --editor cursor

# Open with Vim/Neovim
openshell sandbox connect nemo --editor nvim
```

### 8.8 Gateway Management

```bash
# Check gateway status
openshell gateway status

# Stop gateway (also stops all sandboxes)
openshell gateway stop

# Start gateway
openshell gateway start

# Restart gateway (use when gateway fails)
openshell gateway stop && openshell gateway start

# View gateway configuration
openshell gateway config show

# Edit gateway config
openshell gateway config edit
```

---

## 9. NemoClaw Deep Dive

### 9.1 Full Command Reference

#### Sandbox Lifecycle

```bash
nemoclaw list                     # List all managed sandboxes
nemoclaw nemo status              # Status: model, health, uptime
nemoclaw nemo connect             # SSH into nemo sandbox
nemoclaw nemo destroy             # DANGER: permanent destruction
```

#### Configuration

```bash
nemoclaw onboard                  # Re-run wizard (change model, key, policies)
nemoclaw nemo config show         # Show current configuration
nemoclaw nemo config edit         # Edit config in $EDITOR
```

#### Policy Management

```bash
nemoclaw nemo policy-list         # List active policies
nemoclaw nemo policy-add <name>   # Add preset policy
nemoclaw nemo policy-remove <name># Remove a preset policy
```

#### Service Control

```bash
nemoclaw start                    # Start Telegram bot + tunnel
nemoclaw stop                     # Stop Telegram bot + tunnel
nemoclaw nemo logs                # Recent logs
nemoclaw nemo logs --follow       # Stream live logs
```

### 9.2 Changing the Inference Model

To switch models without destroying your sandbox:

```bash
# Re-run onboard; it preserves workspace data
nemoclaw onboard
# → Navigate to model selection and choose a different model
```

### 9.3 Telegram Bot Integration (Remote Control)

Control your agent from anywhere via Telegram:

```bash
# Prerequisites: Create a bot via @BotFather on Telegram
# Get your bot token: 7654321234:AABBcc...

nemoclaw start --telegram-token "YOUR_BOT_TOKEN"

# Now you can send messages to your bot to control OpenClaw remotely
```

### 9.4 Remote DGX Spark Setup

For NVIDIA DGX Spark hardware users:

```bash
sudo nemoclaw setup-spark
# Configures local GPU inference, network bridges, and DGX-specific policies
```

---

## 10. Policy System — Full Reference

### 10.1 What Policies Control

Policies define exactly what your agent **can and cannot**:
- Access on the network (which hosts, ports, protocols)
- Execute (which binaries are available in the sandbox)
- Read/write (which filesystem paths are permitted)

### 10.2 Available Preset Policies

```bash
# Add multiple presets at once
nemoclaw nemo policy-add docker huggingface github slack telegram

# All available presets:
# docker       — Docker Hub, container registry access
# huggingface  — HuggingFace Hub: download models, datasets
# github       — GitHub API + git operations
# slack        — Slack API for notifications/messaging
# telegram     — Telegram Bot API
# npm          — npm registry (included by default after onboard)
# pypi         — PyPI package index (included by default after onboard)
# aws          — AWS API endpoints
# gcp          — Google Cloud Platform APIs
# azure        — Azure APIs
```

### 10.3 View Current Policies

```bash
# List active policy names
nemoclaw nemo policy-list

# Get full policy YAML
openshell policy get nemo --full

# Get and save to file for editing
openshell policy get nemo --full > ~/current-policy.yaml
```

### 10.4 Policy YAML Structure — Full Reference

```yaml
version: 1

# ─── Network Policies ──────────────────────────────────────────────────────────
network_policies:

  github_full:
    name: github
    endpoints:
      - host: api.github.com
        port: 443
        protocol: rest
        tls: terminate
        rules:
          - allow: { method: "*", path: "/**" }    # Full API access
      - host: github.com
        port: 443
        protocol: rest
        tls: terminate
        rules:
          - allow: { method: "GET", path: "/**" }  # Read-only web access
    binaries:
      - { path: /usr/bin/gh }
      - { path: /usr/local/bin/git }

  huggingface_download:
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

  slack_notify:
    name: slack
    endpoints:
      - host: hooks.slack.com
        port: 443
        protocol: rest
        tls: terminate
        rules:
          - allow: { method: "POST", path: "/services/**" }
      - host: slack.com
        port: 443
        protocol: rest
        tls: terminate
        rules:
          - allow: { method: "*", path: "/api/**" }

  custom_api:
    name: my_backend
    endpoints:
      - host: api.mycompany.com
        port: 443
        protocol: rest
        tls: terminate
        rules:
          - allow: { method: "*", path: "/v1/**" }
          - deny:  { method: "DELETE", path: "/v1/users/**" }  # Block delete users

# ─── Filesystem Policies ───────────────────────────────────────────────────────
filesystem_policies:
  allow_workspace:
    paths:
      - path: /sandbox/.openclaw-data/workspace
        permissions: [read, write, create, delete]
  allow_tmp:
    paths:
      - path: /tmp
        permissions: [read, write, create, delete]
  deny_secrets:
    paths:
      - path: /etc/shadow
        permissions: []   # Empty = deny all
      - path: /root/.ssh
        permissions: []

# ─── Process/Binary Policies ───────────────────────────────────────────────────
process_policies:
  allowed_binaries:
    - { path: /usr/bin/python3 }
    - { path: /usr/local/bin/node }
    - { path: /usr/bin/bash }
    - { path: /usr/bin/curl }
    - { path: /usr/local/bin/git }
    - { path: /usr/bin/gh }
```

### 10.5 Apply Custom Policy

```bash
# Apply a custom policy YAML
openshell policy set nemo --policy ~/my-policy.yaml --wait

# The --wait flag blocks until policy is applied and verified
# Without it, the command returns immediately (async)
```

### 10.6 Iterative Policy Debugging Workflow

When the agent gets blocked, follow this loop:

```bash
# Step 1: Agent tries something and gets denied
# Step 2: Check what was blocked
openshell logs nemo --tail --source gateway | grep DENY

# Step 3: Pull current policy to a working file
openshell policy get nemo --full > ~/current.yaml

# Step 4: Edit the policy to allow the blocked action
nano ~/current.yaml   # or vim, code, etc.

# Step 5: Apply updated policy
openshell policy set nemo --policy ~/current.yaml --wait

# Step 6: Resume agent operation
```

---

## 11. Inference & Model Configuration

### 11.1 Cloud Inference via NVIDIA API

This is the default and recommended path.

```bash
# Models available (as of Alpha 2026.3.x):
# Nemotron 3 Super 120B   — Best quality/cost balance (recommended)
# Nemotron 3 Ultra 253B  — Maximum quality, slower, more expensive
# Nemotron 3 Nano 8B     — Fast and cheap, lower quality

# Switch models by re-running onboard
nemoclaw onboard
```

### 11.2 Local Inference via Ollama

For offline use or cost control:

```bash
# Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# Pull a compatible model
ollama pull nemotron          # If available in Ollama catalog
ollama pull llama3.1:70b      # Alternative large model
ollama pull qwen2.5:32b       # Good code model

# Start Ollama server
ollama serve &

# Re-run nemoclaw onboard and choose "2) Local Ollama"
nemoclaw onboard
# → Select: 2) Local Ollama
# → Enter Ollama endpoint: http://localhost:11434 (default)
# → Select model from list
```

### 11.3 Manual OpenClaw Configuration

Edit the configuration directly for fine-grained control:

```bash
# Connect to sandbox first
nemoclaw nemo connect

# Inside sandbox:
cat ~/.openclaw/openclaw.json

# Edit the config
nano ~/.openclaw/openclaw.json
```

Example `openclaw.json`:

```json
{
  "inference": {
    "provider": "nvidia",
    "api_key_env": "NVIDIA_API_KEY",
    "model": "nemotron-3-super-120b",
    "base_url": "https://integrate.api.nvidia.com/v1",
    "max_tokens": 8192,
    "temperature": 0.2,
    "context_window": 128000
  },
  "agent": {
    "max_iterations": 50,
    "tool_timeout_seconds": 30,
    "memory_path": "/sandbox/.openclaw-data/workspace"
  }
}
```

### 11.4 Use the Gateway Command for Runtime Switching

```bash
# Inside sandbox
sandbox@nemo:~$ openclaw gateway --list         # Show available endpoints
sandbox@nemo:~$ openclaw gateway --set local    # Switch to local Ollama
sandbox@nemo:~$ openclaw gateway --set cloud    # Switch back to NVIDIA API
```

---

## 12. File & Workspace Management

### 12.1 Understanding the Workspace

The agent's persistent memory and all its work lives at:

```
/sandbox/.openclaw-data/workspace/
```

On your host, mirror it at:

```
~/nemo-workspace/
```

Everything in `workspace/` survives sandbox restarts. It does **not** survive `sandbox destroy`.

### 12.2 Recommended Workspace Structure

```
~/nemo-workspace/
├── SOUL.md              ← Agent identity, personality, persistent goals
├── USER.md             ← Your preferences, project context, instructions
├── MEMORY.md           ← Rolling log of completed tasks and decisions
├── projects/
│   ├── project-a/
│   └── project-b/
├── scripts/            ← Reusable scripts you've built with the agent
└── docs/               ← Research, notes, references
```

### 12.3 Download the Full Workspace

```bash
# Initial download (creates local mirror)
openshell sandbox download nemo /sandbox/.openclaw-data/workspace ~/nemo-workspace

# Subsequent syncs (overwrites local with sandbox version)
openshell sandbox download nemo /sandbox/.openclaw-data/workspace ~/nemo-workspace
```

### 12.4 Upload Changes Back to Sandbox

```bash
# Upload a single file
openshell sandbox upload nemo ~/nemo-workspace/SOUL.md \
  /sandbox/.openclaw-data/workspace/SOUL.md

# Upload a specific project folder
openshell sandbox upload nemo ~/nemo-workspace/projects/project-a \
  /sandbox/.openclaw-data/workspace/projects/project-a

# Upload entire workspace (full sync)
openshell sandbox upload nemo ~/nemo-workspace \
  /sandbox/.openclaw-data/workspace
```

### 12.5 Version Control Your Workspace with Git

```bash
cd ~/nemo-workspace

# Initialize git (one-time)
git init
git add .
git commit -m "initial: soul and user context v1"

# Typical daily commit after download
git add -A
git commit -m "daily: $(date +%F) session snapshot"

# Push to remote for offsite backup (optional)
git remote add origin git@github.com:yourname/nemo-workspace.git
git push -u origin main
```

### 12.6 Writing Effective SOUL.md

The `SOUL.md` file is the agent's identity layer. It is read on every session start.

```markdown
# Agent Identity (SOUL.md)

## Who I Am
I am a software engineering agent. My primary role is to help build 
production-grade software systems, write tests, and document code.

## My Working Principles
- Always write tests before implementation (TDD)
- Document every public API
- Prefer explicit over implicit code
- Ask for clarification before making irreversible changes

## Current Goals
- Build the FastAPI backend for Project Alpha (see projects/alpha/)
- Migrate the legacy auth module to JWT-based auth

## Constraints
- Never commit secrets or API keys to any file
- Always confirm before running database migrations
- Do not delete files without explicit user instruction
```

---

## 13. Networking & Port Forwarding

### 13.1 Port Forwarding Basics

```bash
# Expose a port from sandbox to host
openshell forward start <sandbox_port> nemo

# Examples
openshell forward start 3000 nemo    # React dev server
openshell forward start 8000 nemo    # FastAPI backend
openshell forward start 5432 nemo    # PostgreSQL (careful!)
openshell forward start 6379 nemo    # Redis

# Now access at http://localhost:<port> on your host
```

### 13.2 Port Forwarding with Custom Host Port

```bash
# Sandbox port 8000 → Host port 9000
openshell forward start 8000 nemo --host-port 9000

# Access at http://localhost:9000
```

### 13.3 Manage Active Forwards

```bash
openshell forward list              # Show all active forwards
openshell forward stop 3000 nemo   # Stop specific forward
openshell forward stop-all nemo    # Stop all forwards for sandbox
```

### 13.4 Network Namespace Notes

Each sandbox has its own network namespace. The agent cannot see:
- Other sandboxes' network interfaces
- Host machine's local services (unless you explicitly allow them in policy)
- Any host beyond what the policy permits

To allow a local host service (e.g., a local Postgres at `localhost:5432`):

```yaml
network_policies:
  local_postgres:
    name: local_db
    endpoints:
      - host: host.docker.internal   # Routes to host machine from sandbox
        port: 5432
        protocol: tcp
```

---

## 14. Multi-Sandbox & Multi-Agent Setup

### 14.1 Create a Second Sandbox

```bash
# Run onboard with a different name
nemoclaw onboard
# → Sandbox name: worker
# → Different model or same model

nemoclaw list
# Shows: nemo, worker
```

### 14.2 Switch Between Sandboxes

```bash
nemoclaw nemo connect     # Connect to nemo
nemoclaw worker connect   # Connect to worker

# OpenShell equivalents
openshell sandbox connect nemo
openshell sandbox connect worker
```

### 14.3 Multi-Agent Architecture

Use multiple named sandboxes for specialized roles:

| Sandbox | Role | Model |
|---------|------|-------|
| `nemo` | Primary development agent | Nemotron 3 Super 120B |
| `researcher` | Web research + analysis | Nemotron 3 Ultra 253B |
| `worker` | Automated task runner | Nemotron 3 Nano 8B |

### 14.4 Inter-Sandbox Communication

Sandboxes do not communicate directly (by design). Use shared git repositories or shared host folders as the message-passing layer:

```bash
# Agent in 'nemo' writes results to workspace
# Host script syncs nemo's output to worker's workspace
openshell sandbox download nemo /sandbox/.openclaw-data/workspace/output.json ~/shared/
openshell sandbox upload worker ~/shared/output.json /sandbox/.openclaw-data/workspace/input.json
```

---

## 15. Daily Operational Workflow

### 15.1 Optimal Morning Startup

```bash
# 1. Check overall health
nemoclaw status
openshell gateway status

# 2. Start the approval dashboard in a dedicated pane
openshell term &

# 3. Connect to your primary sandbox
nemoclaw nemo connect

# 4. Launch the TUI
sandbox@nemo:~$ openclaw tui
```

### 15.2 During Work Sessions

**From inside the sandbox (CLI mode for long outputs):**

```bash
sandbox@nemo:~$ openclaw agent \
  --agent main \
  --local \
  --session-id dev \
  -m "Implement the user authentication module as described in USER.md"
```

**From the host (for file editing):**

```bash
# Edit files with your preferred editor
code ~/nemo-workspace/USER.md

# Upload changes to sandbox
openshell sandbox upload nemo ~/nemo-workspace/USER.md \
  /sandbox/.openclaw-data/workspace/USER.md

# Tell the agent
sandbox@nemo:~$ openclaw agent -m "I've updated USER.md with new requirements. Please review and proceed." --session-id dev
```

### 15.3 Evening Wind-Down

```bash
# 1. Download current workspace state
openshell sandbox download nemo /sandbox/.openclaw-data/workspace ~/nemo-workspace

# 2. Git commit the day's work
cd ~/nemo-workspace
git add -A
git commit -m "session: $(date +%F) — describe what was done"
git push

# 3. Optional: pause sandbox to save compute
openshell sandbox pause nemo

# 4. Stop the approval dashboard
kill %1   # or close the terminal pane
```

### 15.4 Session Management

OpenClaw sessions persist within a sandbox. Resume previous work:

```bash
sandbox@nemo:~$ openclaw agent \
  --agent main \
  --session-id dev \
  -m "Continue where we left off. What is the current state of the project?"
```

List sessions:

```bash
sandbox@nemo:~$ openclaw sessions list
```

---

## 16. Monitoring, Logging & Observability

### 16.1 Health Monitoring

```bash
# Full status overview
nemoclaw nemo status

# Continuous monitoring (refresh every 5s)
watch -n 5 nemoclaw nemo status

# OpenShell sandbox resource usage
openshell sandbox status nemo --resources
```

### 16.2 Log Sources and What They Tell You

| Source | Command | What it shows |
|--------|---------|---------------|
| Sandbox | `openshell logs nemo --source sandbox` | Agent actions, tool calls, errors |
| Gateway | `openshell logs nemo --source gateway` | Network requests, ALLOW/DENY |
| k3s | `openshell logs nemo --source k3s` | Container orchestration events |
| All | `openshell logs nemo --source all` | Everything combined |

```bash
# Common log monitoring commands

# Watch for policy denials in real-time
openshell logs nemo --tail --source gateway | grep -E "DENY|BLOCK|REJECT"

# Watch for agent errors
openshell logs nemo --tail --source sandbox | grep -E "ERROR|WARN|FATAL"

# Save today's logs
openshell logs nemo --source all > ~/logs/nemo-$(date +%F-%H%M).log

# Search historical logs for a pattern
openshell logs nemo --source sandbox | grep "api.github.com"
```

### 16.3 Interpreting Policy Denial Logs

A gateway denial log entry looks like:

```
2026-03-23T10:22:41Z [GATEWAY] DENY endpoint=api.stripe.com:443 method=POST path=/v1/charges
  reason=no_matching_policy sandbox=nemo pid=1234
```

This tells you:
- **What was denied**: POST to `api.stripe.com/v1/charges`
- **Why**: No policy rule covers this endpoint
- **Fix**: Add a Stripe policy to `current.yaml`

---

## 17. Backup, Restore & Disaster Recovery

### 17.1 Manual Backup

```bash
#!/bin/bash
# save as ~/scripts/backup-nemo.sh
DATE=$(date +%F-%H%M)
DEST="$HOME/backups/nemo/backup-$DATE"

mkdir -p "$DEST"

# Download workspace
openshell sandbox download nemo /sandbox/.openclaw-data/workspace "$DEST/workspace"

# Save current policy
openshell policy get nemo --full > "$DEST/policy.yaml"

# Save NemoClaw config
nemoclaw nemo config show > "$DEST/nemoclaw-config.json"

echo "✅ Backup complete: $DEST"
echo "   Workspace: $(du -sh $DEST/workspace | cut -f1)"
```

```bash
chmod +x ~/scripts/backup-nemo.sh
~/scripts/backup-nemo.sh
```

### 17.2 Automated Daily Backup (cron)

```bash
# Add to crontab
crontab -e

# Add this line (runs at 11:30 PM daily)
30 23 * * * /bin/bash ~/scripts/backup-nemo.sh >> ~/logs/backup.log 2>&1
```

### 17.3 Restore from Backup

```bash
# If sandbox still exists (restoring workspace only)
openshell sandbox upload nemo ~/backups/nemo/backup-2026-03-22-2330/workspace \
  /sandbox/.openclaw-data/workspace

# If sandbox was destroyed — full restore
nemoclaw onboard
# → Complete the wizard with same settings
# → After onboard completes:
openshell sandbox upload nemo ~/backups/nemo/backup-2026-03-22-2330/workspace \
  /sandbox/.openclaw-data/workspace
openshell policy set nemo --policy ~/backups/nemo/backup-2026-03-22-2330/policy.yaml --wait
```

### 17.4 Restore from Git

```bash
# If you have workspace under git control
cd ~/nemo-workspace
git log --oneline    # Find the commit to restore to

# Restore to specific commit
git checkout <commit-hash> -- .
git checkout main

# Upload restored version
openshell sandbox upload nemo ~/nemo-workspace /sandbox/.openclaw-data/workspace
```

---

## 18. Troubleshooting Reference

### 18.1 `command not found: openshell` or `command not found: nemoclaw`

```bash
# Fix every time you open a new shell
export PATH="$HOME/.local/bin:$PATH"
source ~/.bashrc

# Make permanent (if not already in .bashrc)
grep -q 'HOME/.local/bin' ~/.bashrc || \
  echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
```

### 18.2 Gateway Failed to Start

```bash
# Check what's using the gateway port
sudo lsof -i :8080   # adjust port as needed

# Full gateway reset
openshell gateway stop
sleep 2
openshell gateway start

# If still failing, check k3s
sudo systemctl status k3s
sudo systemctl restart k3s
openshell gateway start
```

### 18.3 Agent Lost Memory / Empty Workspace

```bash
# Check if workspace exists in sandbox
nemoclaw nemo connect
sandbox@nemo:~$ ls /sandbox/.openclaw-data/workspace/
sandbox@nemo:~$ exit

# Restore from latest backup
openshell sandbox upload nemo ~/nemo-workspace /sandbox/.openclaw-data/workspace

# Or restore from git
cd ~/nemo-workspace && git log --oneline | head -5
```

### 18.4 Policy Denial Blocking Agent Work

```bash
# 1. Find what's being blocked
openshell logs nemo --tail --source gateway | grep DENY

# 2. Get current policy
openshell policy get nemo --full > ~/debug-policy.yaml

# 3. Add the missing rule (see Policy YAML reference in §10.4)
nano ~/debug-policy.yaml

# 4. Apply
openshell policy set nemo --policy ~/debug-policy.yaml --wait
```

### 18.5 Sandbox Won't Start / Stuck in Pending

```bash
# Check k3s pod status
kubectl get pods -A | grep nemo

# Force pod restart
kubectl delete pod -n nemo-system $(kubectl get pods -n nemo-system -o name) 

# Or do a full sandbox restart
openshell sandbox restart nemo
```

### 18.6 API Key Rejected

```bash
# Test your key directly
curl -s https://integrate.api.nvidia.com/v1/models \
  -H "Authorization: Bearer $(cat ~/.openclaw/nvapi.key)" | jq '.data[].id'

# Re-configure if needed
nemoclaw onboard
```

### 18.7 High Memory or CPU Usage

```bash
# Check resource usage
openshell sandbox status nemo --resources

# Reduce model size via onboard (switch to Nano 8B)
nemoclaw onboard

# Or limit container resources in gateway config
openshell gateway config edit
# → Set cpu_limit and memory_limit under sandbox defaults
```

---

## 19. Security Hardening

### 19.1 Principle of Least Privilege for Policies

Start with no policies except `pypi` and `npm`. Add only what the agent actually needs.

```bash
# Start minimal
nemoclaw onboard
# → Accept default policies (pypi + npm only)

# Add policies only when the agent hits a denial
openshell logs nemo --tail --source gateway | grep DENY
# → Add only the specific endpoint required
```

### 19.2 Audit Policy Usage Regularly

```bash
# Review what the agent has been accessing
openshell logs nemo --source gateway | grep ALLOW | \
  awk '{print $5}' | sort | uniq -c | sort -rn | head -20
```

### 19.3 Protect Your API Key

```bash
# Store with restricted permissions
echo "nvapi-YOUR_KEY" > ~/.openclaw/nvapi.key
chmod 600 ~/.openclaw/nvapi.key
chmod 700 ~/.openclaw/

# Never put it in git
echo "*.key" >> ~/.gitignore
echo "nvapi-*" >> ~/.gitignore
```

### 19.4 Never Approve Unknown Requests in `openshell term`

When `openshell term` shows a pending request you don't recognize:
1. Press `l` to view full context
2. Press `d` to deny if uncertain
3. Check sandbox logs for what the agent was doing
4. Only approve after understanding the request

### 19.5 Sandbox Network Isolation Verification

```bash
# Inside sandbox — verify it cannot reach disallowed hosts
nemoclaw nemo connect
sandbox@nemo:~$ curl -s https://example.com    # Should be DENIED by policy
sandbox@nemo:~$ curl -s https://api.github.com # ALLOWED (if github policy added)
sandbox@nemo:~$ exit
```

---

## 20. Full Uninstall

### 20.1 Backup First (Non-Negotiable)

```bash
~/scripts/backup-nemo.sh   # or manual backup per §17.1
```

### 20.2 Destroy All Sandboxes

```bash
nemoclaw list
# For each sandbox listed:
nemoclaw nemo destroy
nemoclaw worker destroy
# etc.
```

### 20.3 Run Uninstall Script

```bash
curl -fsSL https://raw.githubusercontent.com/NVIDIA/NemoClaw/refs/heads/main/uninstall.sh | \
  bash -s -- --yes
```

### 20.4 Remove OpenShell

```bash
# OpenShell does not have a dedicated uninstall script in Alpha
# Manual removal:
rm -rf ~/.local/bin/openshell
openshell gateway stop 2>/dev/null || true

# Remove k3s (installed by OpenShell)
sudo /usr/local/bin/k3s-uninstall.sh 2>/dev/null || true
```

### 20.5 Clean Up Configuration

```bash
rm -rf ~/.openclaw/
rm -rf ~/nemo-workspace/   # ONLY if you have a backup
# Remove PATH line from ~/.bashrc manually
```

---

## 21. Quick Reference Cheat Sheet

### Host-Side Commands

```bash
# NemoClaw
nemoclaw list                           # List sandboxes
nemoclaw nemo status                    # Health check
nemoclaw nemo connect                   # SSH into sandbox
nemoclaw nemo logs --follow             # Live logs
nemoclaw nemo destroy                   # DANGER: wipe sandbox
nemoclaw nemo policy-list               # Active policies
nemoclaw nemo policy-add github         # Add preset policy
nemoclaw onboard                        # Re-configure everything
nemoclaw start                          # Start Telegram + tunnel
nemoclaw stop                           # Stop Telegram + tunnel

# OpenShell — Sandbox Control
openshell sandbox list                  # List all sandboxes
openshell sandbox connect nemo          # Interactive shell
openshell sandbox status nemo           # Status + resources
openshell sandbox pause nemo            # Pause (save compute)
openshell sandbox resume nemo           # Resume
openshell sandbox restart nemo          # Restart container
openshell sandbox destroy nemo          # Destroy

# OpenShell — File Transfer
openshell sandbox download nemo /sandbox/.openclaw-data/workspace ~/nemo-workspace
openshell sandbox upload nemo ~/nemo-workspace /sandbox/.openclaw-data/workspace
openshell sandbox upload nemo ~/file.md /sandbox/.openclaw-data/workspace/file.md

# OpenShell — Policy
openshell policy get nemo --full > ~/current.yaml
openshell policy set nemo --policy ~/my-policy.yaml --wait

# OpenShell — Logs
openshell logs nemo --tail --source sandbox
openshell logs nemo --tail --source gateway
openshell logs nemo --tail --source all

# OpenShell — Port Forward
openshell forward start 3000 nemo
openshell forward list
openshell forward stop 3000 nemo

# OpenShell — Gateway
openshell gateway status
openshell gateway start
openshell gateway stop
openshell term                          # Approval dashboard (KEEP OPEN)
```

### Inside Sandbox Commands

```bash
# Interactive TUI
openclaw tui

# CLI Agent (better for long tasks)
openclaw agent --agent main --local --session-id dev -m "your task"

# Resume a session
openclaw agent --session-id dev -m "continue"

# List sessions
openclaw sessions list

# Switch inference gateway
openclaw gateway --set cloud
openclaw gateway --set local
openclaw gateway --list
```

### Common One-Liners

```bash
# Full daily backup
openshell sandbox download nemo /sandbox/.openclaw-data/workspace ~/nemo-workspace && \
  cd ~/nemo-workspace && git add -A && git commit -m "daily: $(date +%F)"

# Policy debug (what got denied)
openshell logs nemo --source gateway | grep DENY | tail -20

# Quick health check
nemoclaw nemo status && openshell gateway status

# Open editor with sandbox files
openshell sandbox connect nemo --editor code

# Forward port and connect
openshell forward start 3000 nemo && echo "Open http://localhost:3000"
```

---

> **Alpha Warning**: NemoClaw and OpenShell are in active alpha development as of 2026.3.x. Commands and YAML schemas may change between releases. Always back up `~/nemo-workspace/` before running `destroy` or re-running `onboard`. Track releases at:
> - https://github.com/NVIDIA/NemoClaw
> - https://github.com/NVIDIA/OpenShell
> - https://docs.nvidia.com/nemoclaw/latest/
> - https://docs.nvidia.com/openshell/latest/
