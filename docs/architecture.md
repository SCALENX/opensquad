# Opensquad — System Architecture

> Last updated: 2026-03-22

## Overview

Opensquad is a **multi-agent orchestration framework** distributed as an npm package (`opensquad`). It runs entirely within AI-powered IDEs (Claude Code, Cursor, Antigravity, VS Code + Copilot, OpenAI Codex), with no remote backend — all orchestration happens inside the IDE's AI context.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        IDE (Claude Code, Cursor, etc.)       │
│                                                              │
│  ┌─────────────┐   ┌─────────────────┐   ┌───────────────┐ │
│  │  CLI / bin  │   │  Architect Agent│   │Pipeline Runner│ │
│  │  opensquad  │──▶│  (squad design) │──▶│ (execution)   │ │
│  └─────────────┘   └─────────────────┘   └───────────────┘ │
│         │                   │                    │          │
│         ▼                   ▼                    ▼          │
│  ┌─────────────┐   ┌─────────────────┐   ┌───────────────┐ │
│  │ src/ (Node) │   │  Skills Engine  │   │ Domain Agents │ │
│  │ init, runs  │   │  (integrations) │   │ (per squad)   │ │
│  │ agents, i18n│   └─────────────────┘   └───────────────┘ │
│  └─────────────┘            │                    │          │
│                             ▼                    ▼          │
│                    ┌─────────────────────────────────────┐  │
│                    │          Skills Layer               │  │
│                    │  MCP Servers │ Scripts │ Prompts    │  │
│                    └─────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         │                                        │
         ▼                                        ▼
  ┌────────────┐                         ┌─────────────────┐
  │  Dashboard │                         │  External APIs  │
  │ (React SPA)│                         │ (web, images,   │
  │  Pixi.js   │                         │  social media)  │
  └────────────┘                         └─────────────────┘
```

---

## Directory Structure

```
opensquad/
├── bin/
│   └── opensquad.js              # CLI entry point
├── src/                          # Core Node.js logic
│   ├── init.js                   # Project initialization (multi-IDE)
│   ├── agents.js                 # Agent discovery & installation
│   ├── skills.js                 # Skill discovery & installation
│   ├── runs.js                   # Execution history tracking
│   ├── i18n.js                   # Internationalization (PT-BR, EN, ES)
│   └── logger.js                 # Event logging
├── _opensquad/                   # Framework runtime (templated into user projects)
│   ├── core/
│   │   ├── architect.agent.yaml  # Architect agent definition (~58KB)
│   │   ├── runner.pipeline.md    # Pipeline Runner instructions (~20KB)
│   │   └── skills.engine.md      # Skills Engine documentation (~16KB)
│   ├── _memory/
│   │   ├── company.md            # Persistent company context
│   │   └── preferences.md        # User preferences
│   ├── config.yaml               # Model tier configuration
│   ├── config/
│   │   └── playwright.config.json
│   └── _browser_profile/         # Persistent browser sessions (gitignored)
├── dashboard/                    # Real-time visualization (React + Vite)
│   └── src/
│       ├── App.tsx
│       ├── components/           # Squad cards, status bars
│       ├── office/               # 2D virtual office (Pixi.js)
│       ├── hooks/                # useSquadSocket (WebSocket)
│       ├── store/                # Zustand state management
│       └── types/                # TypeScript definitions
├── skills/                       # Installed skill integrations
│   ├── instagram-publisher/
│   ├── image-generator/
│   ├── canva/
│   ├── blotato/
│   ├── apify/
│   └── ...
├── squads/                       # User-created squads
│   └── {name}/
│       ├── squad.yaml            # Pipeline definition
│       ├── _memory/memories.md
│       ├── _investigations/      # Sherlock profile analyses
│       └── output/{runId}/       # Generated content & state.json
├── templates/                    # Templates used during init
│   ├── _opensquad/
│   ├── dashboard/
│   └── ide-templates/
│       ├── claude-code/
│       ├── cursor/
│       ├── antigravity/
│       ├── codex/
│       └── vscode-copilot/
├── tests/                        # Jest test suite
├── docs/                         # Design docs & plans
└── package.json                  # npm package (v0.1.x)
```

---

## Core Components

### 1. CLI (`bin/`, `src/`)

The entry point for project management commands:

| Command | Handler | Purpose |
|---------|---------|---------|
| `init` | `src/init.js` | Initialize a new Opensquad project |
| `update` | internal | Update core framework files |
| `agents` | `src/agents.js` | Install/list agents |
| `skills` | `src/skills.js` | Install/list skills |
| `runs` | `src/runs.js` | View execution history |

Supports multiple IDEs during `init`: Claude Code, Antigravity, Cursor, VS Code + Copilot, OpenAI Codex.

---

### 2. Agent System

Agents are defined in YAML or Markdown and run within the IDE's AI context. There are two layers:

**Meta-Agents (framework-level):**

| Agent | File | Role |
|-------|------|------|
| Architect | `_opensquad/core/architect.agent.yaml` | Designs and edits squads |
| Pipeline Runner | `_opensquad/core/runner.pipeline.md` | Executes squad pipelines |
| Skills Engine | `_opensquad/core/skills.engine.md` | Manages skill resolution |
| Sherlock | Inline subagent | Investigates reference profiles |

**Domain Agents (user-created per squad):**
- Created by the Architect during squad setup
- Examples: writer, designer, strategist, researcher, publisher
- Defined in `squads/{name}/agents/`

**Design Principles:**
- One responsibility per agent (YAGNI)
- Orchestrators use "powerful" model tier (Claude Opus)
- Researchers/investigators use "fast" model tier (Claude Haiku)
- Mandatory checkpoints at approval gates

---

### 3. Skills System

Skills are modular integrations defined via `SKILL.md` files:

```
skills/{name}/
├── SKILL.md          # Metadata & instructions (YAML frontmatter)
└── ...               # Optional scripts, configs
```

**Skill Types:**

| Type | Description |
|------|-------------|
| `mcp` | Model Context Protocol server integration |
| `script` | Shell/Node/Python script execution |
| `hybrid` | Combination of MCP + script |
| `prompt` | Prompt-only, no external tools |

**Installed Skills:**
- `instagram-publisher` — Social media publishing
- `image-generator` — AI image synthesis
- `image-creator` — Design creation
- `image-fetcher` — Image retrieval
- `canva` — Canva MCP integration
- `blotato` — Publishing integration
- `apify` — Web scraping
- `opensquad-skill-creator` — Creates new skills
- `opensquad-agent-creator` — Creates new agents

---

### 4. Squad Pipeline

Squads are defined in `squads/{name}/squad.yaml`:

```yaml
name: my-squad
agents:
  - name: writer
    role: Content writer
    model: powerful
  - name: publisher
    role: Instagram publisher
    model: fast
pipeline:
  - step: Research
    agent: researcher
    checkpoint: false
  - step: Write content
    agent: writer
    checkpoint: true    # Pauses for user approval
  - step: Publish
    agent: publisher
    checkpoint: false
```

**Execution Flow:**
1. Pipeline Runner validates all required skills
2. Loads company context from `_memory/company.md`
3. Executes steps sequentially
4. Pauses at checkpoints for user approval
5. Writes output to `squads/{name}/output/{runId}/`

---

### 5. Memory & State

| Store | Location | Scope |
|-------|----------|-------|
| Company context | `_opensquad/_memory/company.md` | Global (all squads) |
| Preferences | `_opensquad/_memory/preferences.md` | Global |
| Squad memory | `squads/{name}/_memory/memories.md` | Per-squad |
| Run output | `squads/{name}/output/{runId}/state.json` | Per-run |
| Browser sessions | `_opensquad/_browser_profile/` | Global (gitignored) |

---

### 6. Dashboard

Real-time squad execution visualization built with:

| Technology | Purpose |
|-----------|---------|
| React 19 | UI framework |
| Vite 6 | Build tool |
| TypeScript | Type safety |
| Pixi.js 8 | 2D virtual office rendering |
| Zustand 5 | State management |
| WebSocket (`ws`) | Real-time pipeline updates |

**Features:**
- Virtual office with agent "desks" animated per execution state
- Squad selector & real-time status monitoring
- WebSocket connection to Pipeline Runner

---

## Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js 20+ |
| CLI prompts | `@inquirer/*` |
| Dashboard | React 19 + Vite 6 + TypeScript |
| 2D graphics | Pixi.js 8 + `@pixi/react` |
| State | Zustand 5 |
| Real-time | WebSocket (`ws`) |
| YAML parsing | `yaml` |
| Browser automation | `@playwright/mcp` |
| Testing | Jest |
| Linting | ESLint |
| Distribution | npm (`npx opensquad`) |

---

## Deployment & Distribution

- **npm Package:** `opensquad` (entry: `bin/opensquad.js`)
- **Usage:** `npx opensquad init` to bootstrap a project into any repo
- **No server required** — all execution happens inside the IDE AI context
- **Multi-IDE:** Same codebase works across Claude Code, Cursor, Antigravity, VS Code, Codex via IDE-specific templates

---

## Key Architectural Decisions

1. **No remote backend** — Zero infrastructure to maintain; all logic runs in the IDE AI context
2. **Prompt-driven orchestration** — Agents are defined as structured prompts/YAML, not code
3. **Persistent browser profiles** — Users log into social platforms once; sessions reused across runs
4. **Checkpoint pattern** — Explicit approval gates prevent unwanted automated publishing
5. **Model tier strategy** — Orchestrators get powerful models; fast agents get cheaper models to minimize cost
6. **Skill discovery via SKILL.md** — Convention-based; no registry needed locally
7. **Multi-IDE support** — IDE-specific instruction templates during `init`, same core logic everywhere
