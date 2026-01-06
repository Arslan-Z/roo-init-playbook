# Roo Init Playbook

Bootstrap a brand-new machine + a brand-new project into a ready-to-code **Roo Code workspace**.

This playbook is designed to be executed by an agent running in:

* Claude Code
* Gemini CLI
* Cursor Agent

It assumes the current environment is **UNKNOWN**
(Node / npm / npx, Python, uv / uvx may or may not exist).

---

## 0. Scope and What “Done” Means

### Goal

Given a target empty directory, the agent should:

1. Verify prerequisites (or provide install instructions when automation is not safe).
2. Initialize a Roo Code project with:

   * **roo-commander** (project-scoped)
   * **rooskills** (project-scoped skills)
   * **ConPort MCP config** (context-portal)
   * **Selected Custom Modes** (agent YAMLs)
3. Seed project context (`projectBrief.md`) for ConPort.

### Definition of Done (DoD)

The project directory contains:

* `.roo/` (skills and/or Roo-related files)
* `.roomodes` present (or updated) in project root
* `projectBrief.md` present in project root
* A documented location where Custom Modes YAML were installed
  (global or project-vendored)

And the agent outputs a short **Verification Report** with:

* Versions: node, npm, npx, python, uv / uvx
* Commands executed
* Next steps to start using Roo Code

---

## 1. Required External Repositories (agent must be able to access)

The agent **must clone or reference** the following repositories during execution:

1. **roo-commander**
   [https://github.com/jezweb/roo-commander](https://github.com/jezweb/roo-commander)

2. **rooskills**
   [https://github.com/Kastalien-Research/rooskills](https://github.com/Kastalien-Research/rooskills)

3. **Custom Modes for Roo Code**
   [https://github.com/jtgsystems/Custom-Modes-Roo-Code](https://github.com/jtgsystems/Custom-Modes-Roo-Code)

4. **ConPort / context-portal (MCP server)**
   [https://github.com/GreatScottyMac/context-portal](https://github.com/GreatScottyMac/context-portal)

If any repository is unreachable:

* The agent must report which repository is blocked
* Mark the related component as **PENDING**
* Continue only with unaffected steps

---

## 2. Repository Components (expected layout of this playbook repo)

The playbook repo should contain:

* `README.md` (this file)
* `templates/projectBrief.md` (seed template)
* `templates/mcp_settings.conport.json` (sample MCP settings)
* `agents.lock`
  (list of recommended agent YAML paths from Custom-Modes-Roo-Code)
* `scripts/` (optional helper scripts the agent may generate)

If some files are missing, the agent should create them using the templates below.

---

## 3. Inputs (ask user only if necessary)

The agent should minimize questions. Only ask for:

1. `PROJECT_NAME`

   * Default: current folder name

2. Installation policy constraints

   * “Can I install missing dependencies automatically?” (yes/no)

3. Where to install Custom Modes:

   * Global (recommended): `~/.roo-code/agents/`
   * Project-vendored: `<project>/.roo/agents/`

If the user does not respond, proceed with defaults:

* PROJECT_NAME = current folder name
* Auto-install allowed = NO (provide instructions instead)
* Custom Modes install = Global

---

## 4. Preflight Checks (must run)

Run and record outputs.

### 4.1 Git

```bash
git --version
```

If missing:

* Provide install instructions
* STOP

### 4.2 Node toolchain

(required for rooskills + roo-commander)

```bash
node --version
npm --version
npx --version
```

If any missing:

* DO NOT guess the installer
* Provide OS-appropriate instructions:

  * macOS: Homebrew or nvm / fnm
  * Ubuntu/Debian: apt + NodeSource, or nvm / fnm
  * Windows: winget, official Node installer, or nvm-windows
* STOP unless user explicitly authorizes installs

### 4.3 Python toolchain

(required for ConPort MCP via uvx)

```bash
python3 --version || py --version
uv --version
uvx --version
```

If Python missing:

* Provide install instructions
* STOP

If uv / uvx missing:

* Prefer uv (Astral) if installs are allowed
* Otherwise mark ConPort as **PENDING**

---

## 5. Project Bootstrap Steps (execute in order)

> All commands must be executed from the project root directory.

---

### 5.1 Initialize Git (if needed)

If `.git/` does not exist:

```bash
git init
```

---

### 5.2 Install and init roo-commander (project-scoped)

Source repository:
[https://github.com/jezweb/roo-commander](https://github.com/jezweb/roo-commander)

Commands:

```bash
npm install -g roocommander
roocommander init --project
```

If global npm install is blocked:

* Provide alternative instructions (npm prefix / local install)
* Document what the user must run
* Mark roo-commander as **PENDING**

---

### 5.3 Install rooskills into the project

Source repository:
[https://github.com/Kastalien-Research/rooskills](https://github.com/Kastalien-Research/rooskills)

Command:

```bash
npx @kastalien-research/rooskills init
```

Expected result:

* `.roo/skills/` exists (or updated)
* `.roomodes` exists or updated

If `npx` is blocked:

* Provide instructions
* STOP further Roo-specific setup

---

### 5.4 Add ConPort MCP settings (context-portal)

Source repository:
[https://github.com/GreatScottyMac/context-portal](https://github.com/GreatScottyMac/context-portal)

Goal: create or merge an MCP config defining a server named `conport`.

Preferred approach: **auto-detect workspace**
(no `${workspaceFolder}` dependency).

Use this JSON snippet:

```json
{
  "mcpServers": {
    "conport": {
      "command": "uvx",
      "args": [
        "--from", "context-portal-mcp",
        "conport-mcp",
        "--mode", "stdio",
        "--log-level", "INFO"
      ]
    }
  }
}
```

If `uvx` is missing:

* Provide uv installation instructions
* Mark ConPort as **PENDING**

---

### 5.5 Seed projectBrief.md

Create `projectBrief.md` in project root
(or copy from `templates/projectBrief.md`).

```markdown
# Project Brief

## Goal
- What are we building? What defines success?

## Scope
- In scope:
- Out of scope:

## Tech Stack
- Languages:
- Build tools:
- Runtime:
- CI:

## Repo Conventions
- Formatting / linting:
- Testing:
- Branching strategy:

## Key Decisions
- ADR-001:
- ADR-002:

## Definition of Done
- Build passes locally
- Tests pass
- Lint / format clean
- Minimal docs updated
```

---

### 5.6 Install Custom Modes (agent YAMLs)

Source repository:
[https://github.com/jtgsystems/Custom-Modes-Roo-Code](https://github.com/jtgsystems/Custom-Modes-Roo-Code)

#### Strategy A (Default): Global install

```bash
git clone --depth 1 https://github.com/jtgsystems/Custom-Modes-Roo-Code.git .tmp-custom-modes
mkdir -p ~/.roo-code/agents/
```

Copy selected YAML agents listed in `agents.lock` to:

```
~/.roo-code/agents/
```

Then clean up:

```bash
rm -rf .tmp-custom-modes
```

#### Strategy B: Project-vendored

Copy YAML agents into:

```
<project>/.roo/agents/
```

If `agents.lock` does not exist:

* Install a minimal useful set:

  * General developer agent
  * Python developer agent
  * Architect / system design agent

---

## 6. Verification Checklist (must run)

Verify file presence:

```bash
ls -la .roo/
ls -la .roomodes
ls -la projectBrief.md
```

Print versions:

```bash
node --version
npm --version
npx --version
python3 --version || py --version
uvx --version
```

If any required file is missing:

* Fix it
* Or explain clearly why it is blocked

---

## 7. Final Output: Verification Report (required format)

The agent must output:

```markdown
✅ Roo Init Verification Report

Project: <PROJECT_NAME>
Path: <ABSOLUTE_PATH>

Git: <version>
Node: <version> / npm: <version> / npx: <version>
Python: <version>
uv / uvx: <version or missing>

Installed
- roo-commander: OK / PENDING
- rooskills: OK / PENDING
- ConPort MCP: OK / PENDING
- Custom Modes: global / project-vendored (count: N)

Files created / updated
- .roomodes: present / missing
- .roo/skills/: present / missing
- projectBrief.md: present / missing
- MCP settings: <where?>

Next actions
- How to open Roo Code and select modes
- Exact commands required for any pending items
```

---

## 8. Notes / Policies

* Do not run destructive commands outside the project directory
* Do not silently install system dependencies unless explicitly allowed
* Prefer project-scoped initialization for reproducibility
---


---

