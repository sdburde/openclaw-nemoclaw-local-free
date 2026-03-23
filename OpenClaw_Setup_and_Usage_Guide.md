# OpenClaw — Setup & Usage Guide
**Version**: OpenClaw 2026.3.11  
**Last Updated**: March 23, 2026  
**Sandbox**: `my-assistant`  
**Web UI**: http://127.0.0.1:18789/  
**Workspace path**: `/sandbox/.openclaw-data/workspace/`

---

## Table of Contents

1. [What is OpenClaw](#1-what-is-openclaw)
2. [Launching OpenClaw](#2-launching-openclaw)
3. [Core Workspace Files — Full Reference](#3-core-workspace-files--full-reference)
4. [Creating and Managing Agents](#4-creating-and-managing-agents)
5. [Skills — Full Reference](#5-skills--full-reference)
6. [Nodes — Exec Approvals & Devices](#6-nodes--exec-approvals--devices)
7. [Sessions — Working with Context](#7-sessions--working-with-context)
8. [Cron Jobs — Autonomous Scheduled Tasks](#8-cron-jobs--autonomous-scheduled-tasks)
9. [Daily Usage Patterns](#9-daily-usage-patterns)
10. [Quick Reference](#10-quick-reference)

---

## 1. What is OpenClaw

OpenClaw is the AI agent that runs inside the `my-assistant` sandbox. It is the layer you interact with directly — everything else (OpenShell, NemoClaw, the gateway) exists to run it safely.

```
You
 │
 ├── openclaw tui              (terminal UI inside sandbox)
 ├── openclaw agent -m "..."   (CLI mode inside sandbox)
 └── http://127.0.0.1:18789/  (web UI, accessible from host browser)
          │
          ▼
     OpenClaw Agent
          │
          ├── reads workspace files (SOUL.md, USER.md, etc.)
          ├── calls skills (tools)
          ├── routes inference through gateway → Nemotron / Ollama
          └── writes results back to workspace
```

OpenClaw stores everything in `/sandbox/.openclaw-data/`:

```
.openclaw-data/
  workspace/    <- files the agent reads and writes
  agents/       <- agent definitions
  skills/       <- installed skills
  cron/         <- scheduled jobs
  canvas/       <- scratchpad memory
  identity/     <- device/agent identity data
  devices/      <- paired device tokens
  extensions/   <- installed extensions
  hooks/        <- event hooks
```

---

## 2. Launching OpenClaw

### 2.1 Prerequisites Before Launching

```bash
# On host — confirm sandbox is running
nemoclaw my-assistant status

# On host — confirm web UI port is forwarded
openshell forward list
# Must show: my-assistant  127.0.0.1  18789  <PID>  (not dead)

# If port is dead or missing:
openshell forward stop 18789 my-assistant 2>/dev/null
openshell forward start 18789 my-assistant
```

### 2.2 Connect to the Sandbox

```bash
nemoclaw my-assistant connect
# Prompt becomes: sandbox@my-assistant:~$
```

### 2.3 Launch Modes

**TUI Mode** — interactive, best for back-and-forth conversation:

```bash
sandbox@my-assistant:~$ openclaw tui
```

```
🦞 OpenClaw 2026.3.11 (29dc654)

 openclaw tui - ws://127.0.0.1:18789 - agent main - session main

 session agent:main:main
 connected | press ctrl+c again to exit
 agent main | session main | unknown | tokens ?/131k
──────────────────────────────────────────────────────
```

**CLI Mode** — better for long tasks, scripting, or capturing output:

```bash
sandbox@my-assistant:~$ openclaw agent \
  --agent main \
  --local \
  --session-id dev \
  -m "Your task here"
```

**Web UI** — open in your host browser while TUI or gateway is running:

```
http://127.0.0.1:18789/
```

> If the web UI shows **"Disconnected from gateway. Device identity required."** — launch the TUI first (`openclaw tui`), then refresh the browser. The TUI registers device identity automatically.

---

## 3. Core Workspace Files — Full Reference

All files live at `/sandbox/.openclaw-data/workspace/`. These are read by the agent on every session start. They are the agent's long-term memory and operating instructions.

Download them to your host to edit comfortably:

```bash
openshell sandbox download my-assistant \
  /sandbox/.openclaw-data/workspace ~/my-assistant-workspace
```

Upload edits back:

```bash
openshell sandbox upload my-assistant \
  ~/my-assistant-workspace/SOUL.md \
  /sandbox/.openclaw-data/workspace/SOUL.md
```

---

### SOUL.md
**Size**: ~2.4 KB | **Role**: Agent identity and persistent personality layer

This is the most important file. It defines who the agent is, how it behaves, what principles it follows, and what its current goals are. The agent reads this at the start of every session. Changes here take effect on the next session.

**What to put in it:**
- The agent's role and area of expertise
- Working principles and decision rules
- Current project context and long-term goals
- Hard constraints (things the agent must never do)
- Communication style preferences

**Template:**

```markdown
# Agent Soul

## Role
I am a senior software engineering agent. My primary purpose is to help
design, build, and maintain production-grade software systems.

## Core Principles
- Always write tests before implementation (TDD)
- Never commit secrets, API keys, or credentials to any file
- Document every public function and API endpoint
- Confirm before making irreversible changes (deleting files, running migrations)
- Ask for clarification when requirements are ambiguous rather than assuming

## Current Project Context
Working on Project Alpha — a FastAPI + PostgreSQL REST API.
All project files are in workspace/projects/alpha/.
The current sprint focuses on JWT authentication.

## Long-Term Goals
1. Complete the auth module with full test coverage
2. Set up CI/CD pipeline via GitHub Actions
3. Write API documentation using OpenAPI spec

## Hard Constraints
- Never push directly to main branch
- Never run database migrations without explicit user confirmation
- Never expose internal ports beyond what is policy-approved
- Do not modify files outside workspace/ without permission
```

---

### USER.md
**Size**: ~710 B | **Role**: Your preferences, context, and ongoing instructions

This is your side of the relationship. The agent reads this alongside SOUL.md to understand who it is working with and what the current priorities are. Update this regularly as your project evolves.

**What to put in it:**
- Your technology preferences and style
- Active tasks and their current status
- Notes and constraints specific to the current sprint
- Links to relevant docs or specs

**Template:**

```markdown
# User Context

## Preferences
- Language: Python 3.11 (backend), TypeScript (frontend)
- Test framework: pytest with coverage >80%
- Code style: functional, well-typed, minimal external dependencies
- Git: conventional commits, PR-based workflow

## Active Tasks
1. Implement JWT authentication endpoint — in progress
2. Write integration tests for the /users route — not started
3. Set up Docker Compose for local development — not started

## Notes for Agent
- Staging DB endpoint: staging.internal:5432 (policy-approved)
- Never hardcode connection strings — always use environment variables
- Our GitHub repo: github.com/myorg/project-alpha

## Current Sprint
Sprint 4 — Auth & Testing (March 18–31, 2026)
Goal: Complete auth module with full test suite before March 31.
```

---

### IDENTITY.md
**Size**: ~464 B | **Role**: Agent's self-identification metadata

Stores the agent's stable identity metadata — name, version, and its own description of its capabilities. This is written by the agent itself during initialization and is used for self-reference in multi-agent setups.

You typically do not need to edit this manually. If you have multiple agents, each has its own `IDENTITY.md` scoped to its agent directory.

**Example content:**

```markdown
# Identity

name: main
version: 2026.3.11
type: agent

## Self Description
A general-purpose software engineering agent running on OpenClaw 2026.3.11
inside an OpenShell sandbox. Capable of code generation, file management,
shell execution, web search, and multi-step reasoning.
```

---

### AGENTS.md
**Size**: ~7.7 KB | **Role**: Agent registry and inter-agent coordination rules

This is the largest core file. It defines all agents available in the workspace, their relationships, how they should delegate to each other, and what capabilities each one has. In a single-agent setup this is mostly boilerplate. In a multi-agent setup this is the coordination layer.

**What it controls:**
- Which agents exist and their names
- Which agent is the default
- How agents should hand off tasks to each other
- Each agent's specialization and scope
- Trust levels between agents

**Template for single-agent setup:**

```markdown
# Agents

## Registry

### main
type: primary
description: General-purpose software engineering agent.
model: inference/nvidia/nemotron-3-super-120b-a12b
session_default: main
skills: all
workspace: /sandbox/.openclaw-data/workspace

## Coordination Rules
- main is the default agent for all tasks
- main handles all tool execution and file writing
- main confirms with the user before spawning sub-tasks
```

**Template for multi-agent setup:**

```markdown
# Agents

## Registry

### main
type: primary
description: Orchestrator. Routes tasks to specialized agents.
model: inference/nvidia/nemotron-3-super-120b-a12b
session_default: main
skills: all

### researcher
type: specialist
description: Web research and information synthesis. Read-only.
model: inference/nvidia/nemotron-3-super-120b-a12b
session_default: research
skills: web-search, read-file
constraints:
  - no file writes
  - no shell execution

### coder
type: specialist
description: Code generation and file editing only.
model: inference/nvidia/nemotron-3-super-120b-a12b
session_default: coding
skills: write-file, shell, git

## Delegation Rules
- main delegates research tasks to researcher
- main delegates implementation tasks to coder
- main retains final review and approval
- researcher cannot write files
- coder does not browse the web
```

---

### TOOLS.md
**Size**: ~860 B | **Role**: Tool and skill availability reference

Documents which tools and skills are available to the agent, how to invoke them, and any usage constraints. The agent reads this to know what it is allowed to use. You can add notes here about preferred tool usage patterns.

**Example content:**

```markdown
# Tools Reference

## Available Tools

### file_read
Read any file in the workspace.
Usage: read /sandbox/.openclaw-data/workspace/USER.md

### file_write
Write or overwrite a file in the workspace.
Usage: write content to workspace/output.md
Constraint: Only within workspace/. Confirm before overwriting.

### shell
Execute shell commands inside the sandbox.
Usage: run `python3 script.py`
Constraint: Must be policy-approved. No sudo.

### web_search
Search the web for current information.
Usage: search for "FastAPI JWT authentication example"
Requires: huggingface or github policy (for code examples)

### git
Git operations within the workspace.
Usage: commit changes, create branches, push to remote
Requires: github policy preset.

## Usage Notes
- Always prefer file_write over shell echo for creating files
- Use shell for running tests, not for file manipulation
- Confirm with user before any git push operation
```

---

### HEARTBEAT.md
**Size**: ~168 B | **Role**: Session alive signal and last-seen timestamp

A small file that OpenClaw updates automatically to record when the agent last ran. Used internally to detect stale sessions and by the web UI Overview panel to show agent activity status.

You do not need to edit this manually. If it is missing, OpenClaw recreates it on next launch.

**Example content:**

```markdown
# Heartbeat
last_seen: 2026-03-23T14:37:22Z
session: main
status: idle
```

---

### BOOTSTRAP.md
**Status**: Missing | **Role**: Agent startup initialization script

This file is **missing by default** in OpenClaw 2026.3.11. When present, it runs as a set of instructions executed automatically at the start of every new session — before the user sends any message. Think of it as an `rc` file for the agent.

**Create it to:**
- Pre-load project context into the agent's working memory
- Run a health check on the project state
- Print a session summary to orient yourself
- Set up any environment state the agent needs

**Create the file:**

```bash
# From host
cat > ~/my-assistant-workspace/BOOTSTRAP.md << 'EOF'
# Bootstrap

## On Session Start

Run these steps silently before accepting any user input:

1. Read USER.md and confirm the current active tasks
2. Check if workspace/projects/ exists and list project folders
3. Read SOUL.md to confirm identity
4. Print a brief session summary:
   - Active agent name and model
   - Current tasks from USER.md (top 3)
   - Any files modified in the last 24 hours
5. Confirm ready.
EOF

# Upload to sandbox
openshell sandbox upload my-assistant \
  ~/my-assistant-workspace/BOOTSTRAP.md \
  /sandbox/.openclaw-data/workspace/BOOTSTRAP.md
```

---

### MEMORY.md
**Status**: Missing | **Role**: Persistent rolling memory log

This file is **missing by default**. When present, the agent uses it as a persistent memory store — a running log of completed tasks, decisions made, and things learned across sessions. Without it, the agent has no cross-session memory beyond what is in the other workspace files.

**Create it to:**
- Track what was accomplished in past sessions
- Record architectural decisions and why they were made
- Store recurring patterns and preferences the agent should remember
- Give context to future sessions without repeating yourself

**Create the file:**

```bash
cat > ~/my-assistant-workspace/MEMORY.md << 'EOF'
# Memory Log

## Instructions for Agent
Append a brief summary at the end of each session under a dated heading.
Record: what was completed, decisions made, blockers encountered.
Do not delete old entries — this is a permanent log.

---

## 2026-03-23
- Set up JWT auth endpoint at POST /auth/token
- Decided to use python-jose library for token signing
- Tests written for happy path; edge cases still pending
- Blocker: staging DB connection policy not yet added
EOF

openshell sandbox upload my-assistant \
  ~/my-assistant-workspace/MEMORY.md \
  /sandbox/.openclaw-data/workspace/MEMORY.md
```

---

### File Status Summary

| File | Status | Editable | Auto-updated | Priority |
|------|--------|----------|-------------|----------|
| `SOUL.md` | Present | Yes — your main config | No | ★★★★★ |
| `USER.md` | Present | Yes — update regularly | No | ★★★★★ |
| `AGENTS.md` | Present | Yes — for multi-agent | No | ★★★☆☆ |
| `TOOLS.md` | Present | Yes — add usage notes | No | ★★☆☆☆ |
| `IDENTITY.md` | Present | Rarely needed | Agent-managed | ★★☆☆☆ |
| `HEARTBEAT.md` | Present | No | Auto | ★☆☆☆☆ |
| `BOOTSTRAP.md` | **Missing** | Create it | No | ★★★★☆ |
| `MEMORY.md` | **Missing** | Create it | Agent appends | ★★★★☆ |

---

## 4. Creating and Managing Agents

### 4.1 What is an Agent

An agent is a named configuration that combines:
- A system prompt (from SOUL.md or inline)
- A model assignment
- An enabled set of skills
- A session namespace
- An exec node binding

The default agent is `main`. It is created automatically during onboarding.

### 4.2 Create a New Agent via Web UI

1. Open `http://127.0.0.1:18789/`
2. Go to **Agent → Agents**
3. Click **New Agent**
4. Fill in:
   - **Name**: lowercase, no spaces (e.g., `researcher`, `coder`, `reviewer`)
   - **Description**: one line explaining the agent's role
   - **System Prompt**: the agent's persona and constraints
   - **Model**: choose from the Nodes panel
   - **Skills**: select which skills this agent can use
5. Click **Save**

### 4.3 Create an Agent via Workspace (AGENTS.md)

Edit `AGENTS.md` in your workspace to register a new agent. After saving and uploading, the agent becomes available in the TUI and web UI.

```bash
# Edit locally
nano ~/my-assistant-workspace/AGENTS.md

# Upload
openshell sandbox upload my-assistant \
  ~/my-assistant-workspace/AGENTS.md \
  /sandbox/.openclaw-data/workspace/AGENTS.md
```

### 4.4 Use a Specific Agent in TUI

Once in the TUI, switch agents with:

```
/agent researcher
```

Or launch directly targeting a specific agent:

```bash
sandbox@my-assistant:~$ openclaw tui --agent researcher
```

### 4.5 Use a Specific Agent in CLI Mode

```bash
sandbox@my-assistant:~$ openclaw agent \
  --agent researcher \
  --session-id research-session \
  -m "Summarize the latest papers on RAG architectures"
```

### 4.6 Recommended Multi-Agent Setup

A practical three-agent setup for software development:

**`main`** — Orchestrator and primary interface. Has access to all skills. Routes work to specialists.

```markdown
## System Prompt for main
You are the primary engineering agent. You receive tasks from the user,
break them into subtasks, and coordinate with specialist agents when needed.
You retain final responsibility for quality and correctness.
You confirm with the user before any destructive action.
```

**`researcher`** — Read-only research agent. Can search the web and read files. Cannot write or execute.

```markdown
## System Prompt for researcher
You are a research specialist. Your only job is to find, read, and
synthesize information. You never write files or execute shell commands.
You return structured summaries for the main agent to act on.
Enabled skills: web-search, read-file
```

**`coder`** — Implementation agent. Writes code and runs tests. Does not browse the web.

```markdown
## System Prompt for coder
You are a code implementation specialist. You receive clear specifications
and implement them precisely. You write tests alongside every implementation.
You never browse the web — ask the main agent for any reference material.
Enabled skills: write-file, shell, git
```

### 4.7 Sessions Are Per-Agent

Each agent has its own session namespace. A session ID under `main` is separate from the same ID under `researcher`.

```bash
# main agent, session dev
openclaw agent --agent main --session-id dev -m "..."

# researcher agent, same session id but independent context
openclaw agent --agent researcher --session-id dev -m "..."
```

---

## 5. Skills — Full Reference

### 5.1 What are Skills

Skills are the tools the agent can call — file operations, shell commands, web search, API calls, and more. OpenClaw 2026.3.11 ships with **51 built-in skills** visible in the web UI under **Agent → Skills**.

Skills are grouped into three categories:

| Category | Description |
|----------|-------------|
| **Built-in Skills** | Bundled with OpenClaw — always available |
| **Managed Skills** | Installed and updated via the skill registry |
| **Workspace Skills** | Custom skills you create inside the workspace |

### 5.2 Viewing Skills

**Web UI:**
1. Open `http://127.0.0.1:18789/`
2. Go to **Agent → Skills**
3. Use the **Filter** box to search by name
4. Toggle skills on/off per agent

**51 built-in skills are shown by default.** Common ones include:

| Skill | What it does |
|-------|-------------|
| `read-file` | Read any file from the workspace |
| `write-file` | Create or overwrite files in the workspace |
| `shell` | Execute shell commands inside the sandbox |
| `web-search` | Search the web for current information |
| `git` | Git operations: commit, push, branch, diff |
| `list-dir` | List directory contents |
| `move-file` | Move or rename files |
| `delete-file` | Delete files (confirmation required by policy) |
| `http-request` | Make HTTP requests to policy-allowed endpoints |
| `read-url` | Fetch and read a URL's content |
| `python` | Run Python code directly |
| `node` | Run Node.js code directly |
| `grep` | Search file contents by pattern |
| `diff` | Compare file versions |
| `patch` | Apply diffs to files |

### 5.3 Enable or Disable Skills per Agent

**Web UI:**
1. Go to **Agent → Skills**
2. Select the agent from the dropdown
3. Toggle individual skills on or off
4. Click **Save**

Skills disabled for an agent are simply not offered to it during reasoning — they still exist in the system, the agent just cannot invoke them.

### 5.4 API Key Injection for Skills

Some skills require API keys (e.g., a Slack notification skill, a custom API skill). The Skills panel has an **API key injection** interface:

1. Go to **Agent → Skills**
2. Click the skill that needs a key
3. Enter the key in the **API Key** field
4. Save — the key is stored securely and injected at runtime

The key is never written into workspace files or the agent's context window.

### 5.5 Creating a Workspace Skill

You can write custom skills as JavaScript or Python files placed in the workspace `skills/` directory. The agent discovers them automatically on next session start.

```bash
# Create the skills directory in your workspace
mkdir -p ~/my-assistant-workspace/skills

# Write a custom skill (example: send a Slack message)
cat > ~/my-assistant-workspace/skills/notify-slack.js << 'EOF'
/**
 * @skill notify-slack
 * @description Send a message to a Slack channel via webhook
 * @param {string} message - The message to send
 * @param {string} channel - The channel name (default: #general)
 */
module.exports = async ({ message, channel = '#general' }) => {
  const webhookUrl = process.env.SLACK_WEBHOOK_URL;
  if (!webhookUrl) throw new Error('SLACK_WEBHOOK_URL not set');

  const response = await fetch(webhookUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ text: message, channel })
  });

  return { status: response.status, ok: response.ok };
};
EOF

# Upload to sandbox
openshell sandbox upload my-assistant \
  ~/my-assistant-workspace/skills \
  /sandbox/.openclaw-data/workspace/skills
```

The skill appears in the web UI under **Workspace Skills** after the next session refresh.

### 5.6 Skills and Policy

Skills that make network calls require a matching OpenShell policy to be active. If a skill tries to call an endpoint not covered by policy, the gateway blocks it and logs a `DENY`.

```bash
# Check what got blocked
openshell logs my-assistant --source gateway | grep DENY

# Add required policy
nemoclaw my-assistant policy-add slack
```

---

## 6. Nodes — Exec Approvals & Devices

### 6.1 What are Nodes

Nodes in OpenClaw refers to two related concepts managed from **Agent → Nodes** in the web UI:

- **Exec Approvals** — the policy controlling what shell commands the agent can run and when it needs your approval
- **Devices** — paired browser or client sessions that have authenticated to the gateway
- **Node Bindings** — which execution host an agent uses when running shell commands

### 6.2 Exec Approvals

Found under **Agent → Nodes → Exec approvals**.

Controls whether the agent can run shell commands autonomously, must ask for approval, or is blocked entirely.

#### Target

Exec approval rules can be set at two levels:

| Target | What it affects |
|--------|----------------|
| **Host Gateway** | Rules applied when agent runs exec via the gateway |
| **Node** | Rules applied when agent runs exec on a specific paired node |

#### Scope

Which agents the rule applies to:

| Scope | Meaning |
|-------|---------|
| **Defaults** | Applies to all agents unless overridden |
| **main** | Applies only to the `main` agent |

#### Security Mode

| Mode | Behavior |
|------|----------|
| **Deny** | No exec commands allowed at all — most restrictive |
| **Allowlist** | Only commands explicitly listed in the allowlist can run |
| **Full** | All exec commands permitted — least restrictive |

Recommended starting mode: **Allowlist**. Add specific commands as needed.

#### Ask (Approval Prompt Policy)

Controls when the agent is prompted to ask you for approval before running a command:

| Mode | Behavior |
|------|----------|
| **Off** | Agent never asks — runs or is blocked based on security mode |
| **On miss** | Agent asks when the command is not on the allowlist |
| **Always** | Agent always asks before running any exec command |

Recommended: **On miss** — the agent runs pre-approved commands silently but asks for anything new.

#### Ask Fallback

What happens when the approval UI is unavailable (e.g., web UI is not open):

| Fallback | Behavior |
|----------|---------|
| **Deny** | Block the command if UI is not available to approve |
| **Allowlist** | Fall back to allowlist-only mode |
| **Full** | Allow all commands if UI is unavailable |

Recommended: **Deny** — do not allow unreviewed commands to run silently.

#### Auto-allow Skill CLIs

When enabled, executables listed by skill definitions are automatically added to the allowlist. This is convenient for skill-provided tools (e.g., `gh`, `git`, `python3`).

**Enabled by default** — leave this on unless you want to manually approve every skill executable.

#### Exec Node Binding

Pins agents to a specific node for exec commands. Useful when you have multiple nodes (e.g., the gateway sandbox + a remote machine).

| Setting | Behavior |
|---------|---------|
| **Any node** | Agent picks the available node automatically |
| **Specific node** | Agent always runs exec on the named node |

Current default: `Any node` — appropriate for single-sandbox setups.

### 6.3 Devices

Found under **Agent → Nodes → Devices**.

A device is any client (browser, TUI session) that has authenticated to the OpenClaw gateway. Each device gets a unique identity token.

**Viewing paired devices:**

In the web UI under **Devices → Paired**, you will see entries like:

```
ed4125a22c599d0b68f099e24ff60229e50e8bb60af9d49b61ed17b628df4413
roles: operator
scopes: operator.admin, operator.read, operator.write,
        operator.approvals, operator.pairing
```

This is the device fingerprint of your current browser or TUI session.

**Token management:**

Each paired device has one or more tokens:

| Token field | Meaning |
|-------------|---------|
| `operator` | Role — full administrative access |
| `active` | Token is currently valid |
| `scopes` | What the token can do |
| `2h ago` | When this token was last used |

**Rotate** — generates a new token and invalidates the old one. Use if you suspect a token was leaked.

**Revoke** — permanently revokes the token. The device will need to re-pair.

### 6.4 Nodes (Paired External Machines)

Under **Agent → Nodes → Nodes**, you can pair external machines (your laptop, a remote server) to the OpenClaw gateway. This lets the agent run exec commands on external machines rather than just inside the sandbox.

Currently showing: **No nodes found** — this is correct for a fresh install. The sandbox itself (the gateway) is the only execution host.

To pair an external node, run the OpenClaw node agent on the target machine and follow the pairing instructions in the web UI. This is an advanced feature for remote execution scenarios.

---

## 7. Sessions — Working with Context

### 7.1 What is a Session

A session is a named, persistent conversation thread between you and an agent. The session ID is the key — using the same ID resumes the same context window from where you left off.

Sessions are stored inside the sandbox and visible in the web UI under **Control → Sessions**.

### 7.2 Creating and Using Sessions

```bash
# Start a new session with a custom ID
sandbox@my-assistant:~$ openclaw agent \
  --agent main \
  --session-id project-alpha \
  -m "Let's start working on the auth module."

# Resume the same session later
sandbox@my-assistant:~$ openclaw agent \
  --session-id project-alpha \
  -m "What did we finish last time?"

# Default session (no --session-id flag = uses 'main')
sandbox@my-assistant:~$ openclaw tui
```

### 7.3 Session Naming Strategy

Use descriptive session IDs that map to projects or workstreams:

| Session ID | Purpose |
|------------|---------|
| `main` | Default general session |
| `project-alpha` | Dedicated to Project Alpha |
| `auth-module` | Focused auth implementation work |
| `research` | Research and exploration session |
| `devops` | CI/CD and infrastructure work |

### 7.4 Managing Sessions via Web UI

1. Open `http://127.0.0.1:18789/`
2. Go to **Control → Sessions**
3. From here you can:
   - See all sessions with token counts
   - Click a session to view full history
   - Resume a session (loads context into TUI)
   - Delete sessions you no longer need

### 7.5 Context Window

The default model (Nemotron 3 Super 120B) has a **131,072 token context window**. The TUI shows current usage:

```
tokens ?/131k
```

When a session approaches the limit, start a new session and paste a brief summary of prior work. You can ask the agent to summarize the session before you do:

```
"Summarize everything we've accomplished in this session in 500 words, then I'll start a new session."
```

---

## 8. Cron Jobs — Autonomous Scheduled Tasks

### 8.1 What are Cron Jobs

Cron jobs let the agent run tasks on a schedule with no human input. The agent wakes up, reads the job's instructions, executes the task, and goes back to sleep.

Managed from **Control → Cron Jobs** in the web UI.

### 8.2 Creating a Cron Job

1. Open `http://127.0.0.1:18789/`
2. Go to **Control → Cron Jobs**
3. Click **New Job**
4. Fill in:
   - **Name**: descriptive label
   - **Schedule**: standard cron syntax (e.g., `0 8 * * *` = 8am daily)
   - **Agent**: which agent runs the job
   - **Session ID**: the session context to use
   - **Task**: the instruction sent to the agent
5. Click **Save** and toggle to **Enabled**

### 8.3 Cron Syntax Reference

```
┌───────────── minute (0–59)
│ ┌─────────── hour (0–23)
│ │ ┌───────── day of month (1–31)
│ │ │ ┌─────── month (1–12)
│ │ │ │ ┌───── day of week (0–7, 0 and 7 = Sunday)
│ │ │ │ │
* * * * *

Examples:
0 8 * * *        every day at 8:00 AM
0 8 * * 1-5      every weekday at 8:00 AM
*/30 * * * *     every 30 minutes
0 23 * * *       every day at 11:00 PM
0 9 * * 1        every Monday at 9:00 AM
```

### 8.4 Example Cron Jobs

**Daily morning briefing:**
```
Schedule:  0 8 * * 1-5
Agent:     main
Session:   briefing
Task:      Read USER.md and MEMORY.md, check workspace/projects/ for
           any files modified in the last 24 hours, and write a brief
           status summary to workspace/daily-briefing.md.
```

**Weekly dependency check:**
```
Schedule:  0 10 * * 1
Agent:     main
Session:   maintenance
Task:      Check all Python dependencies in requirements.txt for
           known security advisories. Write a report to
           workspace/security-report.md.
```

**Automated test run:**
```
Schedule:  0 */4 * * *
Agent:     coder
Session:   ci
Task:      Run the test suite with: cd workspace/projects/alpha && python -m pytest.
           Write the result summary to workspace/test-results.md.
           If any tests fail, list them clearly.
```

---

## 9. Daily Usage Patterns

### 9.1 Standard Startup

```bash
# Host terminal 1: approval dashboard
openshell term

# Host terminal 2: connect and launch
nemoclaw my-assistant connect
sandbox@my-assistant:~$ openclaw tui
# Browser: http://127.0.0.1:18789/
```

### 9.2 Directing the Agent

**Give context, not just commands:**

```
# Too vague:
"Fix the bug"

# Good:
"In workspace/projects/alpha/auth.py, the /token endpoint returns 500
when the password contains special characters. Fix it and add a test
for that case."
```

**Reference workspace files:**

```
"Read USER.md for current tasks. Start with task 1 and implement it."
```

**Set scope explicitly:**

```
"Only work within workspace/projects/alpha/. Do not modify any other files."
```

### 9.3 End-of-Session Wrap-Up

Ask the agent to update MEMORY.md before ending a session:

```
"Summarize what we accomplished today and append it to MEMORY.md with today's date."
```

Then sync to host:

```bash
sandbox@my-assistant:~$ exit

openshell sandbox download my-assistant \
  /sandbox/.openclaw-data/workspace ~/my-assistant-workspace

cd ~/my-assistant-workspace
git add -A
git commit -m "session: $(date +%F)"
```

### 9.4 Updating Instructions Mid-Session

Edit `USER.md` or `SOUL.md` on the host, upload, then tell the agent to re-read:

```bash
# Edit locally
nano ~/my-assistant-workspace/USER.md

# Upload
openshell sandbox upload my-assistant \
  ~/my-assistant-workspace/USER.md \
  /sandbox/.openclaw-data/workspace/USER.md

# Tell the agent in TUI or CLI
"Please re-read USER.md — I've updated the active tasks."
```

---

## 10. Quick Reference

### Inside Sandbox Commands

```bash
# Launch TUI (default agent, default session)
openclaw tui

# Launch TUI with specific agent
openclaw tui --agent researcher

# CLI mode
openclaw agent --agent main --session-id dev -m "task"

# Resume session
openclaw agent --session-id dev -m "continue"

# List all sessions
openclaw sessions list

# Check gateway status
openclaw gateway status

# Stop gateway (if needed)
openclaw gateway stop
```

### Workspace File Cheat Sheet

| File | Edit? | Purpose |
|------|-------|---------|
| `SOUL.md` | ★ Always | Agent identity, principles, goals |
| `USER.md` | ★ Always | Your context, active tasks |
| `AGENTS.md` | When adding agents | Agent registry |
| `TOOLS.md` | Optionally | Tool usage notes |
| `IDENTITY.md` | Rarely | Agent self-ID metadata |
| `HEARTBEAT.md` | Never | Auto-managed |
| `BOOTSTRAP.md` | Create it | Session startup script |
| `MEMORY.md` | Create it | Cross-session memory log |

### Web UI Navigation

| What you want | Where to go |
|---------------|-------------|
| See agent activity | Control → Overview |
| Browse/resume sessions | Control → Sessions |
| Schedule a task | Control → Cron Jobs |
| Add a Telegram/Slack channel | Control → Channels |
| Create a new agent | Agent → Agents |
| Enable/disable skills | Agent → Skills |
| Switch inference model | Agent → Nodes |
| Set exec approval policy | Agent → Nodes → Exec approvals |
| View paired devices | Agent → Nodes → Devices |
| Edit openclaw.json | Settings → Config |
| Debug agent reasoning | Settings → Debug |
| Search logs | Settings → Logs |

### Sync Workspace

```bash
# Download sandbox → host
openshell sandbox download my-assistant \
  /sandbox/.openclaw-data/workspace ~/my-assistant-workspace

# Upload host → sandbox (single file)
openshell sandbox upload my-assistant \
  ~/my-assistant-workspace/SOUL.md \
  /sandbox/.openclaw-data/workspace/SOUL.md

# Upload host → sandbox (entire workspace)
openshell sandbox upload my-assistant \
  ~/my-assistant-workspace \
  /sandbox/.openclaw-data/workspace
```

---

> **Workspace is persistent across sandbox restarts but is lost if you run `nemoclaw my-assistant destroy`.** Always sync to `~/my-assistant-workspace` and keep it under git. See the main NemoClaw + OpenShell manual for full backup procedures.
