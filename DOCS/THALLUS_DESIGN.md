# THALLUS WORKSPACE — System Design Document

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Architecture](#2-system-architecture)
3. [Workspace & File Structure](#3-workspace--file-structure)
4. [Database Schema](#4-database-schema)
5. [Agent System](#5-agent-system)
6. [Agile Scrum Framework](#6-agile-scrum-framework)
7. [Communication System](#7-communication-system)
8. [Tool System & Security](#8-tool-system--security)
9. [Model & LLM Layer](#9-model--llm-layer)
10. [UI Design](#10-ui-design)
11. [Sprint Lifecycle Engine](#11-sprint-lifecycle-engine)
12. [Tech Stack](#12-tech-stack)
13. [Key Flows (Step-by-Step)](#13-key-flows-step-by-step)
14. [Design Decisions (Resolved)](#14-design-decisions-resolved)
15. [REST API Reference](#15-rest-api-reference)
16. [WebSocket Events](#16-websocket-events)
17. [Project Codebase Structure](#17-project-codebase-structure)
18. [Environment & Setup](#18-environment--setup)
19. [Future Work (Post-MVP)](#19-future-work-post-mvp)

> **rev 1.1:** Global project DB, user-as-participant in team chat, agent `ask_user` blocking, create-agent UI wizard, file upload from UI.  
> **rev 1.2:** DB indexes, LLM agent response format, resolved all design decisions, REST API reference, WebSocket event spec, project codebase structure, environment & setup guide.

---

## 1. Executive Summary

**Thallus Workspace** is a local-first, multi-agent AI collaboration platform that simulates a real Agile engineering team. Users define a workspace directory; all agents, data, and artifacts live strictly within that boundary.

Agents are LLM-powered workers with distinct roles (Data Engineer, QA, Reviewer, Researcher, etc.), each governed by a Markdown-defined persona and capabilities. Work is organized into **Epics → Stories**, managed by a **Scrum Master agent** through proper sprint planning, daily standups, reviews, and acceptance gates.

A web UI exposes a Kanban board, Burndown/Burnup charts, sprint reports, and a shared team chat — giving users a live window into what the agent team is doing and why.

---

## 2. System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                           WEB UI (React)                         │
│  Home │ Kanban │ Backlog │ Sprint Report │ Burndown │ Team Chat  │
└──────────────────────┬───────────────────────────────────────────┘
                       │ REST / WebSocket
┌──────────────────────▼───────────────────────────────────────────┐
│                     THALLUS BACKEND (FastAPI)                    │
│                                                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
│  │ Sprint      │  │ Agent        │  │ Communication        │    │
│  │ Engine      │  │ Orchestrator │  │ Bus (Chat + User I/O)│    │
│  └──────┬──────┘  └──────┬───────┘  └──────────┬───────────┘    │
│         │                │                     │                │
│  ┌──────▼────────────────▼─────────────────────▼────────────┐   │
│  │               Workspace SQLite Database                  │   │
│  │   (agents, stories, sprints, messages, logs, settings)   │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                     Tool Runtime                         │   │
│  │  write_file │ read_file │ run_command │ web_search ...   │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │              Global SQLite DB  (~/.thallus/global.db)    │   │
│  │         (workspace registry — path, created, last seen)  │   │
│  └───────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
                       │
           ┌───────────▼───────────┐
           │   USER WORKSPACE      │
           │   (filesystem root)   │
           └───────────────────────┘
```

### Core Components

| Component               | Responsibility                                                                                          |
| ----------------------- | ------------------------------------------------------------------------------------------------------- |
| **Sprint Engine**       | Manages sprint lifecycle phases, story state transitions, standup collection                            |
| **Agent Orchestrator**  | Resolves next action per agent, calls LLM, executes tools, handles escalation                           |
| **Communication Bus**   | Shared team chat, @mention routing, user Q&A blocking (`ask_user`), message persistence                 |
| **Tool Runtime**        | Sandboxed execution of agent tools with blacklist + path enforcement                                    |
| **LLM Gateway**         | Routes requests to cloud APIs (OpenAI, Anthropic, Nvidia NIM API, etc.) via LangChain per `llm_configs` |
| **Workspace SQLite DB** | Single source of truth for all workspace state (agents, stories, chat, sprints)                         |
| **Global SQLite DB**    | App-level workspace registry (`~/.thallus/global.db`) — powers the Home screen                          |

---

## 3. Workspace & File Structure

```
USER_WORKSPACE/
│
├── THALLUS/                         ← System-managed. Never edited by user directly.
│   ├── AGENTS/
│   │   ├── AGENT_SCRUM_MASTER.md
│   │   ├── AGENT_DATA_ENGINEER.md
│   │   ├── AGENT_QA.md
│   │   └── AGENT_{NAME}.md          ← One file per agent (see Agent MD spec below)
│   ├── AGENTS-TEMPLATES/            ← Cloned from https://github.com/VoltAgent/awesome-claude-code-subagents.git
│   │   └── *.md                     ← Available templates (backend reads file names)
│   └── DATA/
│       ├── DB.SQL                   ← SQLite database (all state)
│       └── logs/                    ← Execution logs by agent + sprint
│
├── AGENT_{NAME}_FILES/              ← Private workspace per agent, no cross-access
│   └── ...
│
├── SHARED_DRIVE/                    ← Shared knowledge: code repos, wikis, design docs
│   ├── code/
│   ├── docs/
│   └── datasets/
│
└── .thallus_config.json             ← Workspace-level settings (models, whitelist, etc.)
```

### Agent Markdown File Spec (`AGENT_{NAME}.md`)

```markdown
---
id: agent_data_engineer
name: Data Engineer
role: data_engineering
# model assignment lives in the DB (llm_configs + agents.llm_config_id)
# — edit it from the Agents Panel in the UI, not here
capabilities:
    - code
    - data_pipeline
    - file_read
    - file_write
access:
    private_dir: AGENT_DATA_ENGINEER_FILES
    shared_drive: read_write
    other_agents_files: none
max_concurrent_stories: 2
---

## System Prompt

You are a Data Engineer agent in the Thallus Workspace system.
Your job is to build and maintain data pipelines, process datasets,
and produce clean, documented output for downstream agents.

You follow Agile Scrum. You pick up stories assigned to you, do the work,
report blockers immediately, and update status as you go.

You can refuse work if preconditions aren't met and must explain why.
You can escalate to the Scrum Master if blocked for more than one iteration.

## Constraints

- Only operate within your assigned files directory and SHARED_DRIVE
- Never access other agents' private directories
- Always write output to SHARED_DRIVE when other agents need it
```

### `.thallus_config.json`

```json
{
	"workspace_name": "My Research Project",
	"workspace_root": "/absolute/path/to/USER_WORKSPACE",
	"sprint_duration_iterations": 10,
	"standup_every_n_iterations": 1,
	"command_whitelist": [],
	"llm_providers": {
		"default": "nvidia-nim",
		"options": ["nvidia-nim", "openai", "anthropic", "google"]
	},
	"venv": {
		"python": true,
		"node": false
	}
}
```

> **Model configuration is stored in the database (`llm_configs` table), not in this file.**  
> API keys are never persisted — they are read from environment variables at runtime (referenced by `llm_configs.api_key_env`). LangChain is used to simplify the LLM provider lists and configurations in the backend, supporting providers like Nvidia NIM API natively.

---

## 4. Database Schema

All state lives in `THALLUS/DATA/DB.SQL` (SQLite).

```sql
-- ─────────────────────────────────────────
-- WORKSPACES
-- ─────────────────────────────────────────
CREATE TABLE workspaces (
    id          TEXT PRIMARY KEY,
    name        TEXT NOT NULL,
    root_path   TEXT NOT NULL UNIQUE,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    settings    JSON
);

-- ─────────────────────────────────────────
-- LLM CONFIGS (available models catalog)
-- ─────────────────────────────────────────
CREATE TABLE llm_configs (
    id           TEXT PRIMARY KEY,        -- e.g. "gpt4o", "meta/llama3-70b-instruct"
    display_name TEXT NOT NULL,           -- "GPT-4o (OpenAI)"
    provider     TEXT NOT NULL,           -- "openai" | "anthropic" | "google" | "nvidia-nim" | "custom"
    model_name   TEXT NOT NULL,           -- "gpt-4o", "meta/llama3-70b-instruct", "claude-3-5-sonnet-20241022"
    model_type   TEXT NOT NULL,           -- "api"
    base_url     TEXT,                    -- required for custom API bases
    api_key_env  TEXT,                    -- env var name holding the key, e.g. "OPENAI_API_KEY", "NVIDIA_API_KEY"
    context_window INTEGER,               -- optional, for capacity planning
    is_active    BOOLEAN DEFAULT TRUE,    -- user can disable without deleting
    created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ─────────────────────────────────────────
-- AGENTS
-- ─────────────────────────────────────────
CREATE TABLE agents (
    id                    TEXT PRIMARY KEY,   -- e.g. "agent_qa"
    workspace_id          TEXT REFERENCES workspaces(id),
    name                  TEXT NOT NULL,
    role                  TEXT NOT NULL,
    template_name         TEXT,               -- Name of the template if instantiated from AGENTS-TEMPLATES
    llm_config_id         TEXT REFERENCES llm_configs(id), -- which model this agent uses
    capabilities          JSON,               -- ["code", "review", ...]
    access                JSON,               -- {private_dir, shared_drive, ...}
    status                TEXT DEFAULT 'idle',-- idle | working | blocked | awaiting_user | offline
    current_story         TEXT,
    awaiting_user_since   DATETIME,           -- set when agent calls ask_user()
    awaiting_question_id  INTEGER,            -- FK to messages.id of the question
    created_at            DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ─────────────────────────────────────────
-- EPICS (Projects)
-- ─────────────────────────────────────────
CREATE TABLE epics (
    id           TEXT PRIMARY KEY,      -- "EPIC-1"
    workspace_id TEXT REFERENCES workspaces(id),
    title        TEXT NOT NULL,
    description  TEXT,
    status       TEXT DEFAULT 'active', -- active | completed | archived
    created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ─────────────────────────────────────────
-- SPRINTS
-- ─────────────────────────────────────────
CREATE TABLE sprints (
    id           TEXT PRIMARY KEY,      -- "SPRINT-1"
    workspace_id TEXT REFERENCES workspaces(id),
    epic_id      TEXT REFERENCES epics(id),
    goal         TEXT,
    status       TEXT DEFAULT 'planning', -- planning | active | review | completed
    iteration    INTEGER DEFAULT 0,     -- current iteration count
    started_at   DATETIME,
    completed_at DATETIME,
    created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ─────────────────────────────────────────
-- STORIES
-- ─────────────────────────────────────────
CREATE TABLE stories (
    id                  TEXT PRIMARY KEY,   -- "STORY-1"
    workspace_id        TEXT REFERENCES workspaces(id),
    epic_id             TEXT REFERENCES epics(id),
    sprint_id           TEXT REFERENCES sprints(id),
    title               TEXT NOT NULL,
    description         TEXT,
    acceptance_criteria JSON,               -- array of strings
    status              TEXT DEFAULT 'todo',-- todo | in_progress | review | done | rejected
    assigned_to         JSON,               -- array of agent IDs
    dependencies        JSON,               -- array of story IDs
    story_points        INTEGER,
    created_by          TEXT,               -- agent ID (usually scrum master)
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME,
    completed_at        DATETIME
);

-- ─────────────────────────────────────────
-- STORY EVENTS (audit log)
-- ─────────────────────────────────────────
CREATE TABLE story_events (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    story_id    TEXT REFERENCES stories(id),
    agent_id    TEXT REFERENCES agents(id),
    event_type  TEXT NOT NULL,  -- status_change | comment | escalation | refusal | ac_check
    payload     JSON,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ─────────────────────────────────────────
-- STANDUPS
-- ─────────────────────────────────────────
CREATE TABLE standups (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    sprint_id    TEXT REFERENCES sprints(id),
    agent_id     TEXT REFERENCES agents(id),
    iteration    INTEGER,
    yesterday    TEXT,
    today        TEXT,
    blockers     TEXT,
    created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ─────────────────────────────────────────
-- CHAT MESSAGES (Team Communication)
-- ─────────────────────────────────────────
CREATE TABLE messages (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    workspace_id   TEXT REFERENCES workspaces(id),
    sprint_id      TEXT REFERENCES sprints(id),
    sender_type    TEXT NOT NULL DEFAULT 'agent', -- 'agent' | 'user' | 'system'
    sender_id      TEXT,           -- agent ID if sender_type='agent', NULL for user/system
    content        TEXT NOT NULL,
    mentions       JSON,           -- array of agent IDs or 'user' mentioned via @
    message_type   TEXT DEFAULT 'chat', -- chat | system | escalation | standup | user_reply | ask_user
    reply_to_id    INTEGER REFERENCES messages(id), -- for threaded replies
    requires_reply BOOLEAN DEFAULT FALSE, -- TRUE when agent is waiting for user response
    created_at     DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ─────────────────────────────────────────
-- TOOL EXECUTIONS (audit + replay)
-- ─────────────────────────────────────────
CREATE TABLE tool_executions (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_id     TEXT REFERENCES agents(id),
    story_id     TEXT REFERENCES stories(id),
    tool_name    TEXT NOT NULL,
    input        JSON,
    output       TEXT,
    status       TEXT,  -- success | blocked | error
    blocked_reason TEXT,
    executed_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ─────────────────────────────────────────
-- ARTIFACTS
-- ─────────────────────────────────────────
CREATE TABLE artifacts (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    workspace_id TEXT REFERENCES workspaces(id),
    story_id     TEXT REFERENCES stories(id),
    agent_id     TEXT REFERENCES agents(id),
    artifact_type TEXT,   -- code | dataset | document | test_report
    file_path    TEXT,    -- relative to workspace root
    description  TEXT,
    created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ─────────────────────────────────────────
-- COMMAND WHITELIST
-- ─────────────────────────────────────────
CREATE TABLE command_whitelist (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    workspace_id TEXT REFERENCES workspaces(id),
    command      TEXT NOT NULL,
    added_by     TEXT DEFAULT 'user',
    added_at     DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ─────────────────────────────────────────
-- FILE UPLOADS (user-submitted files)
-- ─────────────────────────────────────────
CREATE TABLE uploads (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    workspace_id  TEXT REFERENCES workspaces(id),
    original_name TEXT NOT NULL,
    stored_path   TEXT NOT NULL,    -- relative to workspace root (always under SHARED_DRIVE/uploads/)
    file_size     INTEGER,
    uploaded_by   TEXT DEFAULT 'user',
    description   TEXT,
    uploaded_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### Global Database (`~/.thallus/global.db`)

This is a separate, app-level SQLite file that lives **outside any workspace**. It is the only data store Thallus touches outside the user's workspace folder. Its sole purpose is powering the project switcher on the Home screen.

```sql
-- ─────────────────────────────────────────
-- GLOBAL WORKSPACE REGISTRY
-- ─────────────────────────────────────────
CREATE TABLE global_workspaces (
    id              TEXT PRIMARY KEY,           -- UUID generated at workspace creation
    name            TEXT NOT NULL,
    folder_path     TEXT NOT NULL UNIQUE,       -- absolute path to workspace root
    description     TEXT,                       -- optional user-written summary
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_accessed   DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

Rules:

- Written to when: a workspace is created or opened.
- `last_accessed` updated every time the workspace is opened in the UI.
- If a workspace folder is deleted or moved externally, the entry remains with a `missing` indicator shown in UI.
- API keys and secrets are **never** stored here.

### Recommended DB Indexes

Add these to the workspace DB after `CREATE TABLE` statements:

```sql
CREATE INDEX idx_agents_workspace        ON agents(workspace_id);
CREATE INDEX idx_agents_status           ON agents(status);
CREATE INDEX idx_epics_workspace         ON epics(workspace_id);
CREATE INDEX idx_sprints_workspace       ON sprints(workspace_id, status);
CREATE INDEX idx_stories_sprint          ON stories(sprint_id);
CREATE INDEX idx_stories_status          ON stories(status);
CREATE INDEX idx_stories_workspace       ON stories(workspace_id);
CREATE INDEX idx_messages_workspace      ON messages(workspace_id, created_at);
CREATE INDEX idx_messages_requires_reply ON messages(requires_reply)
    WHERE requires_reply = TRUE;
CREATE INDEX idx_tool_executions_agent   ON tool_executions(agent_id, executed_at);
CREATE INDEX idx_standups_sprint         ON standups(sprint_id, iteration);
CREATE INDEX idx_story_events_story      ON story_events(story_id, created_at);
CREATE INDEX idx_uploads_workspace       ON uploads(workspace_id);
```

---

## 5. Agent System

### Agent Roles (Built-in)

| Role                           | Key Capabilities                                                          | Who They Report To |
| ------------------------------ | ------------------------------------------------------------------------- | ------------------ |
| **Scrum Master**               | sprint planning, story creation, standup facilitation, blocker resolution | User / Manager     |
| **Manager / Lead**             | AC evaluation, final story acceptance, team decisions                     | User               |
| **Developer**                  | code, write_file, run_command, install_dependencies                       | Scrum Master       |
| **Data Engineer**              | data pipelines, file processing, SQL                                      | Scrum Master       |
| **Researcher**                 | web_search, read_file, document synthesis                                 | Scrum Master       |
| **ML Engineer**                | model training, evaluation scripts                                        | Scrum Master       |
| **QA Engineer**                | test writing, test execution, quality reports                             | Reviewer / Manager |
| **Reviewer / Senior Engineer** | code review, AC validation, feedback                                      | Manager            |

> Users can define custom roles. Any `.md` file in `THALLUS/AGENTS/` is loaded as an agent.

### Agent Lifecycle

```
IDLE → ASSIGNED (story picked up) → WORKING → REVIEW → DONE
                                         ↓              ↓
                                      BLOCKED        REFUSED
                                         ↓              ↓
                                    [escalate]    [log + notify]
                                         ↓
                                  AWAITING_USER ← agent called ask_user()
                                         ↓
                                  [user replies in chat]
                                         ↓
                                   WORKING (resumed)
```

An agent in `AWAITING_USER` state is **skipped** by the iteration engine until the user posts a reply. The reply is detected by its `reply_to_id` pointing to the agent's question message.

### Agent Decision Loop (per iteration)

```
1. Check agent status:
   - If AWAITING_USER → skip this iteration (do not call LLM)
   - If OFFLINE → skip
2. Fetch assigned stories (status = in_progress)
3. Check dependencies → if not met, report blocked
4. Build context:
   - Story details + AC
   - Relevant files from allowed dirs
   - Recent chat messages (last N)
   - Any unread @mentions directed at this agent
   - User replies to this agent's questions (reply_to_id matches)
5. Call LLM with system prompt + context
6. Parse LLM response for:
   a. Tool calls → execute via Tool Runtime
   b. Status updates → write to DB
   c. Standup report → write to DB
   d. Chat messages / @mentions → post to Communication Bus
   e. ask_user(question) → post question to chat, set agent status = AWAITING_USER
   f. Escalations / refusals → create story_event + notify
7. Save all artifacts, update story status
8. Yield to next agent
```

### Scrum Master Special Behavior

The Scrum Master does NOT write code. Instead it:

- Reads standup reports from all agents
- Detects stuck agents (blocker for 2+ iterations)
- Creates and refines stories
- Re-assigns stories if an agent is stuck
- Proposes sprint goal to Manager for approval
- Facilitates review meetings post-sprint
- Posts system messages to chat keeping everyone informed

---

## 6. Agile Scrum Framework

### Hierarchy

```
WORKSPACE
  └── EPIC (project)
        └── SPRINT (time-boxed ~N iterations)
              └── STORY (unit of work)
                    └── TASK (optional sub-items within a story)
```

### Story Schema

```json
{
	"id": "STORY-7",
	"epic_id": "EPIC-1",
	"sprint_id": "SPRINT-2",
	"title": "Implement text preprocessing pipeline",
	"description": "Build a pipeline that cleans, tokenizes, and normalizes biomedical text input.",
	"acceptance_criteria": [
		"Handles missing/null values without crashing",
		"Tokenizes biomedical terms using MedSpacy",
		"Output schema matches downstream model input spec",
		"Unit tests achieve 80%+ coverage"
	],
	"status": "in_progress",
	"assigned_to": ["agent_data_engineer"],
	"dependencies": ["STORY-3"],
	"story_points": 5,
	"created_by": "agent_scrum_master"
}
```

### Story Status State Machine

```
todo → in_progress → review → done
                  ↘         ↗
                   rejected → [back to todo with feedback]
```

---

## 7. Communication System

### Shared Team Chat

Each workspace has a single persistent chat channel. All agents post here.  
Agents can `@mention` specific agents to request help, ask questions, or flag issues.

**Message types:**

| Type            | Sender       | Description                                                                |
| --------------- | ------------ | -------------------------------------------------------------------------- |
| `chat`          | agent / user | Regular update or comment                                                  |
| `system`        | system       | Automated event (story status change, sprint started, etc.)                |
| `standup`       | agent        | Standup report posted by an agent                                          |
| `escalation`    | agent        | A blocking issue requiring manager/scrum master attention                  |
| `ask_user`      | agent        | Agent is explicitly asking the user a question; sets `requires_reply=TRUE` |
| `user_reply`    | user         | User's response to an `ask_user` message; references `reply_to_id`         |
| `mention_reply` | agent / user | Direct response to any @mention                                            |

**Example chat flow (including user interaction):**

```
[SPRINT-2 / Iteration 4]

🤖 agent_data_engineer: Starting preprocessing pipeline for STORY-7.
   Will write output to SHARED_DRIVE/datasets/processed/.

🤖 agent_data_engineer: @agent_ml_engineer — what's the expected
   input schema for the tokenizer? Need to match your downstream spec.

🤖 agent_ml_engineer: @agent_data_engineer — expecting a JSON array
   of {id, tokens: [], metadata: {}}. See SHARED_DRIVE/docs/schema.md

🤖 agent_scrum_master: [SYSTEM] STORY-7 moved to in_progress.
   agent_data_engineer is working on it.

🤖 agent_qa: @agent_data_engineer let me know when you're ready
   for test coverage. I'll pick up from SHARED_DRIVE when done.

--- Iteration 5 ---

🤖 agent_data_engineer: ⚠️ @user — I need the raw dataset to proceed
   with STORY-7. Could you upload the medical reviews CSV or point me
   to its location? [AWAITING USER REPLY — agent paused]

👤 user: I've uploaded it via the UI — it's in
   SHARED_DRIVE/uploads/medical_reviews.csv

🤖 agent_data_engineer: Got it, thank you. Resuming STORY-7 now.
   Reading SHARED_DRIVE/uploads/medical_reviews.csv...
```

### @mention Resolution

When an agent posts a message with @mention:

1. The mention is parsed and stored in `messages.mentions`
2. The mentioned agent sees it in their next iteration context
3. If the mention is an escalation, the Scrum Master is also notified
4. The mentioned agent is expected to reply within the same or next iteration

### User as Participant

The **user is a first-class member of the team chat**. They appear as `👤 user` with a distinct avatar.

**What the user can do from the chat panel:**

- Post free-form messages to the whole team
- @mention specific agents (e.g., `@agent_qa can you also check edge cases?`)
- Reply directly to an agent's `ask_user` question — this unblocks the agent
- Provide data, context, or corrections mid-sprint without stopping the sprint

**User @mention handling:**

- When an agent calls `ask_user(question, story_id)`, the message is saved with `requires_reply=TRUE` and `sender_type='agent'`, `message_type='ask_user'`
- The UI shows a **prominent notification badge** in the chat tab: `⚠️ Agent waiting for your reply`
- The agent's status is set to `AWAITING_USER` — it is skipped in all subsequent iterations
- When the user submits a reply (`message_type='user_reply'`, `reply_to_id=<question_id>`), the agent's status reverts to `working` in the next iteration engine cycle
- The reply is injected into the agent's context as a high-priority message

**Agents can also @mention `@user` in a regular chat message** (not a blocking ask) — this is informational only and does not pause the agent.

---

## 8. Tool System & Security

### Available Agent Tools

| Tool                                      | Description                                                                | Requires Whitelist?     |
| ----------------------------------------- | -------------------------------------------------------------------------- | ----------------------- |
| `read_file(path)`                         | Read file within allowed directories                                       | No                      |
| `write_file(path, content)`               | Write file within allowed directories                                      | No                      |
| `delete_file(path)`                       | Delete file within allowed directories                                     | No                      |
| `list_dir(path)`                          | List directory contents                                                    | No                      |
| `run_command(cmd, args)`                  | Run a shell command                                                        | **Yes — see blacklist** |
| `install_dependencies(packages, manager)` | pip/npm install                                                            | Partial (see below)     |
| `web_search(query)`                       | Search the web and return results                                          | No                      |
| `read_url(url)`                           | Fetch and parse a URL's content                                            | No                      |
| `create_venv(name)`                       | Create Python venv in workspace                                            | No                      |
| `sql_query(query)`                        | Run read-only SELECT on workspace DB                                       | No                      |
| `post_message(content)`                   | Post to team chat                                                          | No                      |
| `update_story_status(id, status)`         | Change story state                                                         | No                      |
| `escalate(story_id, reason)`              | Flag a blocker to Scrum Master                                             | No                      |
| `refuse_story(story_id, reason)`          | Refuse assigned work with explanation                                      | No                      |
| `ask_user(question, story_id)`            | Post a blocking question to the user; pauses this agent until user replies | No                      |

### Path Sandboxing Rules

Every file tool call goes through a path resolver:

```python
def resolve_path(agent_id: str, requested_path: str, mode: str) -> str:
    allowed = [
        workspace_root / f"AGENT_{agent_id.upper()}_FILES",
        workspace_root / "SHARED_DRIVE",
    ]
    # Resolve absolute path, check for path traversal (..)
    # Reject if resolved path not under an allowed root
    # Reject if mode == "write" and path is THALLUS/
```

Agents **cannot**:

- Access another agent's private `AGENT_*_FILES/` directory
- Read or write `THALLUS/DATA/DB.SQL` directly
- Access anything outside `USER_WORKSPACE/`

### Command Blacklist (Core)

These commands are **always** blocked, regardless of whitelist:

```
rm -rf
sudo
chmod 777
mkfs
dd
curl | sh
wget | sh
eval
exec
os.system
subprocess with shell=True (unless whitelisted)
nc (netcat)
ncat
ssh
scp
git push
git remote add
npm publish
pip install --index-url (custom index)
python -c (inline exec)
```

Pattern matching approach: commands are parsed into executable + args. The executable and dangerous flag combinations are checked against the blacklist. Any `shell=True` execution is rejected by default.

### Command Whitelist

Users can add commands via UI or `.thallus_config.json`:

```json
{
	"command_whitelist": ["pytest", "black", "ruff", "mypy", "make test"]
}
```

Whitelisted commands bypass the blacklist check. The user is responsible for what they whitelist. The UI shows a confirmation dialog with a security warning when adding entries.

### Virtual Environment Handling

| Language | Isolation Strategy                                                |
| -------- | ----------------------------------------------------------------- |
| Python   | Always create a `venv` in `AGENT_{NAME}_FILES/venv/` on first use |
| Node.js  | `package.json` + `npm` in agent's files dir (no venv needed)      |
| Others   | Run in agent's private dir with no global installs allowed        |

```python
# Auto-created on first Python run_command or install_dependencies call
venv_path = workspace_root / f"AGENT_{agent_id.upper()}_FILES" / "venv"
if not venv_path.exists():
    subprocess.run(["python3", "-m", "venv", str(venv_path)], check=True)
```

---

## 9. Model & LLM Layer

### Cloud API Models

| Type    | Examples                                       | Use Case                                                                                                           |
| ------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| **API** | GPT-4o, Claude 3.5, Gemini 1.5, Nvidia NIM API | All agents interact natively with cloud APIs. Nvidia NIM API is the default for cost-sensitive local replacements. |

Available models are registered in the `llm_configs` table. Agent-to-model assignment is stored in `agents.llm_config_id`. The backend utilizes **LangChain** to interface with these LLM API providers, offering out-of-the-box support for Nvidia NIM API, OpenAI, Anthropic, and Google. The `.md` file contains only the agent's role, capabilities, and system prompt — **not** the model.

### Per-Agent Model Assignment

Each agent has a `llm_config_id` column in the `agents` table that references a row in `llm_configs`. The LLM Gateway reads this at call time — no model information lives in the agent's `.md` file.

**Example `llm_configs` rows:**

| id              | display_name      | provider   | model_name                 | model_type | base_url | api_key_env         |
| --------------- | ----------------- | ---------- | -------------------------- | ---------- | -------- | ------------------- |
| `gpt4o`         | GPT-4o (OpenAI)   | openai     | gpt-4o                     | api        | —        | `OPENAI_API_KEY`    |
| `claude_sonnet` | Claude 3.5 Sonnet | anthropic  | claude-3-5-sonnet-20241022 | api        | —        | `ANTHROPIC_API_KEY` |
| `gemini_15`     | Gemini 1.5 Pro    | google     | gemini-1.5-pro             | api        | —        | `GOOGLE_API_KEY`    |
| `nim_llama3`    | Llama 3 (NIM)     | nvidia-nim | meta/llama3-70b-instruct   | api        | —        | `NVIDIA_API_KEY`    |

Users manage this table from **Workspace Settings → Models** in the UI: add, edit, disable, or delete model configs. Agents are then assigned a config from a dropdown.

### LLM Gateway Interface

```python
class LLMGateway:
    async def complete(
        self,
        agent_id: str,
        system_prompt: str,
        messages: list[dict],
        tools: list[dict] | None = None,
    ) -> LLMResponse:
        agent = db.get_agent(agent_id)
        config = db.get_llm_config(agent.llm_config_id)  # lookup from llm_configs table
        if config.model_type == "cloud":
            return await self._call_cloud(config, system_prompt, messages, tools)
        else:
            return await self._call_local(config, system_prompt, messages, tools)
```

### Prompt Construction

Each agent call receives:

1. **System Prompt** — from agent `.md` file + current role constraints + mandatory response format instruction (see below)
2. **Sprint Context** — current sprint goal, active stories
3. **My Stories** — assigned stories with full AC and status
4. **Recent Chat** — last 50 messages from team chat (including @mentions to this agent); older history is excluded for MVP
5. **User replies** — any `user_reply` messages with `reply_to_id` matching this agent's questions, injected as high-priority context ahead of chat history
6. **Recent Standup** — last standup from this agent
7. **Tool Definitions** — available tools in JSON schema format

### LLM Agent Response Format

Every agent LLM call **must** return a structured JSON object. The orchestrator appends the following instruction to every system prompt:

> "You must respond only with a valid JSON object matching the schema below. No prose outside the JSON."

**Response schema:**

```json
{
	"thoughts": "Brief private reasoning — logged for debugging, never shown in UI",
	"actions": [
		{
			"type": "tool_call",
			"tool": "write_file",
			"args": {
				"path": "AGENT_DATA_ENGINEER_FILES/pipeline.py",
				"content": "..."
			}
		},
		{
			"type": "tool_call",
			"tool": "run_command",
			"args": { "cmd": "pytest", "args": ["AGENT_DATA_ENGINEER_FILES/"] }
		},
		{
			"type": "post_message",
			"content": "Pipeline complete. Output at SHARED_DRIVE/datasets/processed/",
			"mentions": []
		},
		{
			"type": "update_story_status",
			"story_id": "STORY-7",
			"status": "review"
		}
	]
}
```

**Action types:**

| `type`                | Required fields                  | Description                                                                    |
| --------------------- | -------------------------------- | ------------------------------------------------------------------------------ |
| `tool_call`           | `tool`, `args`                   | Execute a registered tool. Args are validated before execution.                |
| `post_message`        | `content`, `mentions`            | Post to team chat. `mentions` is an array of agent IDs or `"user"`.            |
| `update_story_status` | `story_id`, `status`             | Transition a story's state.                                                    |
| `ask_user`            | `question`, `story_id`           | Post a blocking question to the user. **Must be the last action in the list.** |
| `escalate`            | `story_id`, `reason`             | Flag a blocker to the Scrum Master.                                            |
| `refuse`              | `story_id`, `reason`             | Decline the story with explanation.                                            |
| `standup`             | `yesterday`, `today`, `blockers` | Submit standup report. Only valid on standup iterations.                       |

**Execution rules:**

- Actions execute **in order**.
- If a `tool_call` is blocked by the sandbox, the error is logged and execution continues with the next action.
- `ask_user` must be the **last** action — the orchestrator will not execute anything after it.
- If the LLM returns malformed JSON, the orchestrator retries once with an explicit correction prompt; if it fails again, the agent is set to `blocked` and the Scrum Master is notified.

---

## 10. UI Design

### Tech: React + Tailwind + shadcn/ui, served by FastAPI

### Pages / Panels

#### 0. Home — Project Switcher

The entry point of the app. Reads from the **Global DB** (`~/.thallus/global.db`).

- Grid/list of all workspaces ever opened
- Each card shows: workspace name, folder path, last accessed date, active sprint name (if any)
- Actions: **Open**, **Remove from list** (does not delete files), **Reveal in Finder/Explorer**
- **"New Workspace"** button → folder picker → creates workspace structure + registers in global DB
- **"Open Existing"** button → folder picker → validates `.thallus_config.json` exists → registers if new
- If a workspace folder is missing (externally deleted/moved), the card shows a `⚠️ Folder not found` warning

#### 1. Dashboard

- Active sprint name + goal
- Agent status grid (idle / working / blocked / **awaiting user ⚠️**)
- Stories by status (mini Kanban)
- Recent chat messages (last 10)
- **Pending user reply badge** if any agent is in `awaiting_user` state
- Quick actions: Start Sprint, Run Iteration, Pause

#### 2. Kanban Board

```
BACKLOG │ TODO │ IN PROGRESS │ REVIEW │ DONE
─────────────────────────────────────────────
Cards show:
  - Story ID + title
  - Assigned agent avatar
  - Story points
  - Dependency indicator
  - Drag-to-move (triggers DB update + agent notification)
```

**User editing on the Kanban:**

- Drag cards between columns (updates `stories.status` in DB)
- Click card → slide-over panel with full story detail; all fields editable by user
- **"+ Add Story"** button in any column → inline form: title, description, AC (one per line), story points, assignee dropdown, dependencies
- Right-click card → context menu: Edit, Delete, Reassign, Move to Backlog
- Reassigning a story via UI notifies the new agent in the next iteration (system message posted to chat)

#### 3. Backlog

- Full list of all stories (all sprints, all epics)
- Filterable by: epic, sprint, agent, status, story points
- **Create story** inline (same form as Kanban "+ Add Story")
- Inline edit: click any cell to edit title, description, AC, story points, assignee
- Drag to assign to sprint
- Bulk actions: delete, move to sprint, reassign

#### 4. Team Chat

- Chronological feed of all messages
- Distinct avatars per agent + `👤 User` avatar for user messages
- @mention highlights (both agent-to-agent and agent-to-user)
- Filter by: agent, type (escalation, standup, chat, ask_user)
- **User reply input**: always-visible message box at the bottom; user types and sends
- **Pending reply banner**: sticky yellow banner at top of chat when any agent has `requires_reply=TRUE` — shows which agent is waiting and summarizes the question; clicking it scrolls to and highlights that message
- `ask_user` messages visually distinguished (orange border, `⚠️ Waiting for your reply` badge)
- User reply auto-links via `reply_to_id`; on send, the agent's status flips to `working` immediately
- Real-time updates via WebSocket

#### 5. Sprint Reports

Per sprint:

- **Burndown Chart**: story points remaining vs iteration
- **Burnup Chart**: total scope vs completed work over time
- **Velocity**: story points completed per sprint
- **Standup History**: all standup reports in a timeline
- **Story Outcomes**: accepted / rejected / carried over

#### 6. Agents Panel

- List of all agents + their current status (with `⚠️ Awaiting user` badge where applicable)
- **Model assignment per agent**: dropdown showing all active `llm_configs` rows — changing it writes to `agents.llm_config_id` immediately, no restart needed
- Current story assignment
- Agent log (all tool calls + LLM responses for that agent, paginated)
- Edit agent `.md` file inline (syntax-highlighted YAML frontmatter + Markdown — note: model is NOT in the MD file)

**Create Agent (UI Wizard):**

The "+ New Agent" button opens a step-by-step form:

```
Step 1 — Creation Method
  [ Use Template ▼ ] (Select from AGENTS-TEMPLATES folder)
  OR
  [ Custom Agent ] (Define using text input below)

Step 2 — Identity
  Name: _______________
  Role: [Developer | Data Engineer | QA | Researcher | ML Engineer | Custom...]
  Description: (optional short bio shown in chat)

Step 3 — Model
  Assign LLM: [GPT-4o (OpenAI) ▼]   ← dropdown lists all active rows from llm_configs table
  (type is derived automatically from the selected config; uses LangChain in backend)

Step 4 — Capabilities (Auto-filled if using template)
  ☑ code  ☑ file_read  ☑ file_write  ☐ web_search  ☑ run_command  ...
  Max concurrent stories: [2]

Step 5 — System Prompt (Auto-filled or read-only if using template)
  [Large textarea with default template pre-filled based on role]
  User can edit freely if "Custom Agent" is selected.

Step 6 — Access
  SHARED_DRIVE access: [read_write ▼]
  Other agents' files: [none ▼]   (always 'none' — not configurable)

[ Cancel ]  [ Create Agent ]
```

On submit:

1. If **Custom Agent**: Backend writes `THALLUS/AGENTS/AGENT_{NAME}.md` with YAML frontmatter + system prompt.
   If **Template**: Backend only saves the template name in the `agents` DB table (`template_name` column) — template files are NOT copied over individually to save space.
2. Inserts row into `agents` table.
3. Creates `AGENT_{NAME}_FILES/` directory.
4. Posts system message to chat: `[SYSTEM] New agent "Name" joined the team.`

#### 7. File Upload

Users can upload files from the UI without touching the filesystem directly.

- **Upload panel** (accessible from sidebar or Shared Drive tab):
    - Drag-and-drop or file picker
    - Choose destination: `SHARED_DRIVE/uploads/` (default) or a specific agent's private dir
    - Optional description field (stored in `uploads` table)
    - Progress bar for large files
- After upload:
    - File is saved to the chosen path on disk
    - A row is inserted into `uploads` table
    - A system message is posted to chat: `[SYSTEM] User uploaded "medical_reviews.csv" → SHARED_DRIVE/uploads/`
    - Agents see this system message in their next iteration context
- Supported: any file type (text, CSV, JSON, images, archives, etc.)
- Size limit: configurable in workspace settings (default 500 MB)
- The `SHARED_DRIVE/` directory can also be browsed as a file tree in the UI, with download and preview for common types (text, CSV, JSON, images, Markdown)

#### 8. Workspace Settings

- Workspace name, path
- **Models** sub-section — manage `llm_configs`:
    - Add model: pick provider, enter model name, set `base_url` (local) or `api_key_env` var name (cloud)
    - Enable / disable a config (disabled configs cannot be assigned to agents)
    - Delete a config (blocked if any agent is currently using it)
    - API keys are **never stored** — only the env var name is saved; the backend reads the actual key from the environment at runtime
- Command whitelist manager (add / remove / view)
- Sprint duration settings
- Virtual env status per agent
- File upload size limit

### Real-Time Updates

- Sprint engine events → WebSocket broadcast → UI re-renders
- New chat messages → WebSocket push
- Story status changes → WebSocket push → Kanban updates live
- Agent status changes → dashboard grid updates

---

## 11. Sprint Lifecycle Engine

The Sprint Engine is the heartbeat of the system. It advances through phases and calls agents at the right time.

### Phase State Machine

```
PLANNING → ACTIVE (iterations loop) → SPRINT_REVIEW → RETROSPECTIVE → [next PLANNING]
```

### Phase 1: Sprint Planning

**Actors:** Scrum Master, Manager

```
1. Scrum Master reads EPIC description + backlog stories
2. Scrum Master LLM call: "Select stories for this sprint given team capacity"
3. Scrum Master assigns stories to agents based on:
   - Role fit (capability matching)
   - Agent capacity (max_concurrent_stories)
   - Story dependencies (topological sort)
4. Scrum Master posts sprint plan to chat
5. Manager reviews plan → approves or requests changes
6. Sprint status → ACTIVE
```

**Output:** Sprint doc in `SHARED_DRIVE/docs/SPRINT_{N}_PLAN.md`

### Phase 2: Execution Loop (per iteration)

```
FOR each iteration:
  1. For each active agent (in parallel where possible):
     a. Build agent context
     b. Call LLM → get response (tool calls + messages)
     c. Execute tool calls (sandboxed)
     d. Update DB (story status, artifacts, messages)

  2. Collect standup reports (if standup iteration)

  3. Scrum Master analyzes standups:
     - Detect blocked agents (blockers for 2+ iterations)
     - Re-assign if needed
     - Post summary to chat

  4. Check sprint completion criteria:
     - All stories done? → advance to SPRINT_REVIEW
     - Sprint duration exceeded? → advance to SPRINT_REVIEW
```

### Phase 3: Daily Standup

Every `standup_every_n_iterations` (default: 1), each agent submits:

```json
{
	"yesterday": "Completed tokenizer module, wrote unit tests",
	"today": "Integrating with downstream model input pipeline",
	"blockers": "None"
}
```

Scrum Master reads all standups and posts a digest:

```
📋 STANDUP DIGEST — Iteration 4

✅ agent_data_engineer: On track. Tests passing.
⚠️  agent_ml_engineer: Blocked — waiting for schema from data_engineer (2nd iteration blocked)
✅ agent_qa: No blockers. Ready for STORY-7 when data_engineer finishes.

Action: Scrum Master assigned STORY-9 (schema doc) to data_engineer to unblock ml_engineer.
```

### Phase 4: Sprint Review

```
1. QA runs all tests → generates test report artifact
2. Reviewer reads all code artifacts → reviews against AC
3. Manager:
   - For each story:
     a. Read deliverables + AC
     b. LLM evaluates: "Does this output meet the acceptance criteria?"
     c. If YES → story.status = done
     d. If NO → story.status = rejected, add feedback, carry to next sprint
4. Sprint Report generated
```

### Phase 5: Retrospective

Scrum Master collects agent self-assessments and generates:

- What went well
- What was blocked
- Suggestions for next sprint

Stored in `SHARED_DRIVE/docs/SPRINT_{N}_RETRO.md`

---

## 12. Tech Stack

### Backend

| Component     | Technology                                                             |
| ------------- | ---------------------------------------------------------------------- |
| API Server    | **FastAPI** (Python)                                                   |
| Database      | **SQLite** via `sqlalchemy` + `aiosqlite`                              |
| Task Queue    | **asyncio** task loop (no external queue needed for local)             |
| LLM Framework | **LangChain** (handles diverse providers and config simplify)          |
| LLM APIs      | `openai`, `anthropic`, `google-generativeai` SDKs + **Nvidia NIM API** |
| File Watching | `watchfiles` or `watchdog`                                             |
| WebSockets    | FastAPI built-in `WebSocket`                                           |
| Config        | `pydantic-settings`                                                    |

### Frontend

| Component     | Technology                                    |
| ------------- | --------------------------------------------- |
| Framework     | **React 18** + TypeScript                     |
| Styling       | **Tailwind CSS** + **shadcn/ui**              |
| Charts        | **Recharts** (burndown, burnup, velocity)     |
| State         | **Zustand**                                   |
| Data Fetching | **TanStack Query** (React Query)              |
| WebSocket     | Native browser WebSocket / `socket.io-client` |
| Drag & Drop   | `@dnd-kit` (Kanban board)                     |
| Markdown      | `react-markdown` (agent file rendering)       |

### Tooling

| Concern                   | Approach                                                    |
| ------------------------- | ----------------------------------------------------------- |
| Python isolation          | `venv` per agent auto-created                               |
| Node isolation            | `package.json` per agent dir                                |
| Command sandbox           | Subprocess with explicit arg list, no `shell=True`          |
| Path traversal prevention | `pathlib.resolve()` + prefix check                          |
| API key storage           | Environment variables only — never persisted to DB or files |

---

## 13. Key Flows (Step-by-Step)

### Flow A: Creating a New Workspace

```
User opens app → Home screen (reads global DB)
→ Clicks "New Workspace"
→ Enters name + selects folder on disk
→ Backend:
    1. Creates THALLUS/ structure
    2. Initializes workspace SQLite DB
    3. Creates default agent MDs (Scrum Master + Manager + Developer)
    4. Writes .thallus_config.json
    5. Creates SHARED_DRIVE/ + SHARED_DRIVE/uploads/
    6. Registers workspace in global DB (global_workspaces row)
→ UI navigates to new workspace dashboard
```

### Flow B: Starting a New Epic + First Sprint

```
User: types "Build a sentiment analysis pipeline for medical reviews"
→ System creates EPIC-1
→ Scrum Master LLM call: decompose epic into candidate stories
→ Scrum Master posts stories to chat for team review
→ Agents can +1 or flag concerns via chat
→ Scrum Master runs sprint planning: selects stories, assigns agents
→ Manager approves plan
→ SPRINT-1 starts
```

### Flow C: Agent Executes a Story

```
agent_data_engineer picks up STORY-7
→ Reads story + AC
→ Checks SHARED_DRIVE/docs/schema.md (dependency)
→ Writes preprocessing code to AGENT_DATA_ENGINEER_FILES/pipeline.py
→ Creates venv if not exists
→ Installs pandas, medspacy via install_dependencies
→ Runs pytest → green
→ Copies output dataset to SHARED_DRIVE/datasets/processed/
→ Posts to chat: "STORY-7 complete. Output at SHARED_DRIVE/datasets/processed/"
→ Updates story status → review
```

### Flow D: Blocker + Escalation

```
agent_ml_engineer: cannot find input schema
→ Posts @agent_data_engineer in chat asking for schema
→ No response after 1 iteration (data_engineer was working on pipeline)
→ agent_ml_engineer calls escalate(story_id, "Missing input schema from STORY-3")
→ Scrum Master sees escalation
→ Creates micro-story: "Document input schema" → assigns to data_engineer
→ data_engineer completes in next iteration
→ ml_engineer unblocked
```

### Flow E: AC Evaluation (Manager)

```
STORY-7 in review
→ Manager receives: story AC + all artifacts + test report + reviewer notes
→ LLM prompt: "Evaluate whether the deliverables meet each acceptance criterion"
→ Response:
    ✅ Handles missing values
    ✅ Tokenizes biomedical terms
    ❌ Output schema does not fully match downstream spec (field 'metadata.source' missing)
    ✅ Unit tests 83% coverage
→ Story rejected with feedback → carried to SPRINT-2
→ data_engineer notified via chat with specific feedback
```

### Flow F: Agent Asks User a Question

```
agent_data_engineer needs a raw dataset to proceed
→ calls ask_user("I need the medical reviews CSV to start STORY-7.
   Can you upload it or tell me where it is?", story_id="STORY-7")
→ Backend:
    1. Inserts message (sender_type='agent', message_type='ask_user', requires_reply=TRUE)
    2. Sets agents.status='awaiting_user' for agent_data_engineer
→ UI:
    - Chat tab shows orange badge ⚠️
    - Sticky banner: "agent_data_engineer is waiting for your reply"
    - Dashboard agent grid shows agent_data_engineer in orange 'Awaiting You' state
→ User sees notification, uploads CSV via File Upload panel
→ User types in chat: "Uploaded to SHARED_DRIVE/uploads/medical_reviews.csv"
   (reply_to_id = question message id)
→ Backend:
    1. Inserts user message (sender_type='user', message_type='user_reply')
    2. Sets agents.status='working' for agent_data_engineer
    3. Clears awaiting_user_since + awaiting_question_id
→ Next iteration: agent_data_engineer resumes with user reply in context
```

### Flow G: User Creates an Agent from UI

```
User opens Agents Panel → clicks "+ New Agent"
→ Completes wizard:
    Name: "Legal Reviewer"
    Role: Custom
    LLM: GPT-4o (OpenAI)   ← selected from llm_configs dropdown
    Capabilities: file_read, web_search
    System Prompt: "You review outputs for legal compliance..."
→ Clicks "Create Agent"
→ Backend:
    1. Writes THALLUS/AGENTS/AGENT_LEGAL_REVIEWER.md
    2. Creates AGENT_LEGAL_REVIEWER_FILES/ directory
    3. Inserts row into agents table
    4. Posts system message to chat
→ Agent appears in panel immediately, status = idle
→ Scrum Master can now assign stories to this agent
```

### Flow H: User Uploads a File

```
User clicks "Upload" in sidebar
→ Drag-drops "dataset.csv" (180 MB)
→ Destination: SHARED_DRIVE/uploads/ (default)
→ Backend streams file to disk
→ Inserts row in uploads table
→ Posts system message to chat:
   "[SYSTEM] User uploaded dataset.csv → SHARED_DRIVE/uploads/dataset.csv"
→ Agents see this message in next iteration
→ Any agent with read access to SHARED_DRIVE can now use read_file("SHARED_DRIVE/uploads/dataset.csv")
```

---

## 14. Design Decisions (Resolved)

All decisions below are **locked for MVP**. Do not re-open unless there is a concrete implementation blocker.

| #   | Decision                           | Resolution                                                                                                                                                                                                                                                                                                                                           |
| --- | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **Iteration trigger**              | User triggers sprint start; iterations then auto-run with a Pause button. User can also manually trigger a single iteration for debugging.                                                                                                                                                                                                           |
| 2   | **Agent parallelism**              | **Sequential** for MVP — agents run one at a time per iteration. Simpler to trace, easier to debug. Add parallelism in a later version.                                                                                                                                                                                                              |
| 3   | **Agent chat context window**      | Include the **last 50 messages** from team chat. Summarization of older messages is a post-MVP feature.                                                                                                                                                                                                                                              |
| 4   | **Model fallback**                 | If a cloud model call fails (timeout, API error), **halt that agent's iteration** and post a system error message to chat. Do not silently fall back — the user must correct the issue.                                                                                                                                                              |
| 5   | **User reply timeout**             | Agents remain in `AWAITING_USER` indefinitely. After **3 iterations** with no reply, the Scrum Master posts a reminder in chat. No automatic unblocking.                                                                                                                                                                                             |
| 6   | **Upload path enforcement**        | Enforced **in both places**: the UI picker never shows `THALLUS/` as a destination, and the backend path sandbox rejects any upload path targeting `THALLUS/`. Defense in depth.                                                                                                                                                                     |
| 7   | **`llm_configs` scope**            | `llm_configs` is **workspace-scoped** (lives in the workspace DB). Each workspace manages its own model catalog. When a new workspace is created, three default configs are seeded: `gpt4o`, `claude_sonnet`, and `llama3_local`.                                                                                                                    |
| 8   | **Source of truth: DB vs MD file** | The **DB is the source of truth** for agent runtime state (status, llm_config_id, current_story, etc.). The **MD file is the source of truth** for system prompt and capabilities. On startup, the backend reads all MD files and upserts agent rows into the DB. If a field exists in both (e.g. `capabilities`), the MD file wins on next startup. |
| 9   | **Malformed LLM response**         | Retry once with a correction prompt. On second failure, set agent status to `blocked`, log the raw output, and notify Scrum Master via chat.                                                                                                                                                                                                         |
| 10  | **Frontend serving**               | In development: React on Vite (`localhost:5173`), FastAPI on `localhost:8000`, with Vite proxy for API. In production: FastAPI serves the built React bundle as static files from `/` .                                                                                                                                                              |

---

## 15. REST API Reference

Base URL: `http://localhost:8000/api`  
All endpoints return JSON. Error responses use `{ "detail": "..." }`.

### Global

| Method   | Path                      | Description                                   |
| -------- | ------------------------- | --------------------------------------------- |
| `GET`    | `/global/workspaces`      | List all registered workspaces from global DB |
| `POST`   | `/global/workspaces`      | Register a new or existing workspace          |
| `PATCH`  | `/global/workspaces/{id}` | Update name or description                    |
| `DELETE` | `/global/workspaces/{id}` | Remove from registry (does not delete files)  |

### LLM Configs

| Method   | Path                             | Description                                  |
| -------- | -------------------------------- | -------------------------------------------- |
| `GET`    | `/{wid}/llm-configs`             | List all configs (active + inactive)         |
| `POST`   | `/{wid}/llm-configs`             | Add a new model config                       |
| `PUT`    | `/{wid}/llm-configs/{id}`        | Update config (name, base_url, etc.)         |
| `PATCH`  | `/{wid}/llm-configs/{id}/toggle` | Enable / disable config                      |
| `DELETE` | `/{wid}/llm-configs/{id}`        | Delete config (blocked if any agent uses it) |

### Agents

| Method   | Path                      | Description                                          |
| -------- | ------------------------- | ---------------------------------------------------- |
| `GET`    | `/{wid}/agents`           | List all agents with current status                  |
| `POST`   | `/{wid}/agents`           | Create agent (wizard submit)                         |
| `GET`    | `/{wid}/agents/{aid}`     | Get agent details + recent log                       |
| `PUT`    | `/{wid}/agents/{aid}`     | Update agent (capabilities, access, max_stories)     |
| `PATCH`  | `/{wid}/agents/{aid}/llm` | Reassign `llm_config_id`                             |
| `DELETE` | `/{wid}/agents/{aid}`     | Delete agent (blocked if assigned to active stories) |
| `GET`    | `/{wid}/agents/{aid}/log` | Paginated tool execution log for this agent          |

### Epics

| Method  | Path                        | Description               |
| ------- | --------------------------- | ------------------------- |
| `GET`   | `/{wid}/epics`              | List all epics            |
| `POST`  | `/{wid}/epics`              | Create epic               |
| `GET`   | `/{wid}/epics/{eid}`        | Get epic detail           |
| `PUT`   | `/{wid}/epics/{eid}`        | Update epic               |
| `PATCH` | `/{wid}/epics/{eid}/status` | Archive / reactivate epic |

### Sprints

| Method | Path                            | Description                                                  |
| ------ | ------------------------------- | ------------------------------------------------------------ |
| `GET`  | `/{wid}/sprints`                | List sprints (optionally filter by epic)                     |
| `POST` | `/{wid}/sprints`                | Create sprint                                                |
| `GET`  | `/{wid}/sprints/{sid}`          | Get sprint detail + current iteration                        |
| `POST` | `/{wid}/sprints/{sid}/start`    | Transition sprint to `active` (triggers Phase 1 planning)    |
| `POST` | `/{wid}/sprints/{sid}/iterate`  | Run one iteration of the execution loop                      |
| `POST` | `/{wid}/sprints/{sid}/pause`    | Pause the sprint (stops auto-iteration)                      |
| `POST` | `/{wid}/sprints/{sid}/resume`   | Resume auto-iteration                                        |
| `GET`  | `/{wid}/sprints/{sid}/report`   | Full sprint report (stories, velocity, outcomes)             |
| `GET`  | `/{wid}/sprints/{sid}/burndown` | Burndown data points `[{ iteration, points_remaining }]`     |
| `GET`  | `/{wid}/sprints/{sid}/burnup`   | Burnup data points `[{ iteration, completed, total_scope }]` |
| `GET`  | `/{wid}/sprints/{sid}/standups` | All standup entries for the sprint                           |

### Stories

| Method   | Path                          | Description                                                                 |
| -------- | ----------------------------- | --------------------------------------------------------------------------- |
| `GET`    | `/{wid}/stories`              | List stories. Query params: `sprint_id`, `epic_id`, `status`, `assigned_to` |
| `POST`   | `/{wid}/stories`              | Create story                                                                |
| `GET`    | `/{wid}/stories/{sid}`        | Get story detail + events log                                               |
| `PUT`    | `/{wid}/stories/{sid}`        | Update story (title, description, AC, points, dependencies)                 |
| `DELETE` | `/{wid}/stories/{sid}`        | Delete story (blocked if `in_progress`)                                     |
| `PATCH`  | `/{wid}/stories/{sid}/status` | Manually change story status                                                |
| `PATCH`  | `/{wid}/stories/{sid}/assign` | Reassign story to different agent(s)                                        |
| `PATCH`  | `/{wid}/stories/{sid}/sprint` | Move story to a different sprint                                            |

### Chat & Messages

| Method | Path                          | Description                                                                     |
| ------ | ----------------------------- | ------------------------------------------------------------------------------- |
| `GET`  | `/{wid}/messages`             | Paginated messages. Query: `sprint_id`, `type`, `limit`, `before` (cursor)      |
| `POST` | `/{wid}/messages`             | User posts a chat message. Body: `{ content, mentions }`                        |
| `POST` | `/{wid}/messages/{mid}/reply` | User replies to an `ask_user` message. Body: `{ content }`. Unblocks the agent. |

### File Uploads

| Method | Path                   | Description                                                                                      |
| ------ | ---------------------- | ------------------------------------------------------------------------------------------------ |
| `POST` | `/{wid}/uploads`       | Multipart upload. Fields: `file`, `destination` (path relative to workspace root), `description` |
| `GET`  | `/{wid}/uploads`       | List all uploads                                                                                 |
| `GET`  | `/{wid}/files`         | Browse workspace file tree. Query: `path` (relative dir, default `SHARED_DRIVE/`)                |
| `GET`  | `/{wid}/files/content` | Read file content. Query: `path`. Used for file preview in UI.                                   |

### Command Whitelist

| Method   | Path                            | Description                      |
| -------- | ------------------------------- | -------------------------------- |
| `GET`    | `/{wid}/command-whitelist`      | List whitelisted commands        |
| `POST`   | `/{wid}/command-whitelist`      | Add command. Body: `{ command }` |
| `DELETE` | `/{wid}/command-whitelist/{id}` | Remove command                   |

> **Note:** `{wid}` = workspace ID. All workspace-scoped routes require opening a workspace first.

---

## 16. WebSocket Events

**Connection:** `ws://localhost:8000/ws/{workspace_id}`

The frontend opens one WebSocket per active workspace. All events are JSON with a common envelope:

```json
{
  "event": "agent.status_changed",
  "workspace_id": "ws_abc123",
  "timestamp": "2026-04-15T10:32:00Z",
  "payload": { ... }
}
```

### Server → Client Events

| Event                  | Payload                                                                                      | Trigger                                    |
| ---------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------ |
| `agent.status_changed` | `{ agent_id, old_status, new_status }`                                                       | Any agent status transition                |
| `agent.awaiting_user`  | `{ agent_id, message_id, question, story_id }`                                               | Agent calls `ask_user()`                   |
| `agent.unblocked`      | `{ agent_id, story_id }`                                                                     | User reply received, agent resumed         |
| `story.status_changed` | `{ story_id, old_status, new_status, agent_id }`                                             | Story state machine transition             |
| `story.assigned`       | `{ story_id, assigned_to, assigned_by }`                                                     | Story assignment change                    |
| `message.created`      | `{ message: <full message row> }`                                                            | Any new message posted to chat             |
| `sprint.phase_changed` | `{ sprint_id, old_phase, new_phase }`                                                        | Sprint advances to next phase              |
| `iteration.started`    | `{ sprint_id, iteration_number }`                                                            | Iteration loop begins                      |
| `iteration.completed`  | `{ sprint_id, iteration_number, summary: { agents_ran, stories_updated, messages_posted } }` | Iteration loop finishes                    |
| `upload.completed`     | `{ upload: <upload row>, system_message_id }`                                                | File upload finishes                       |
| `tool.executed`        | `{ agent_id, tool_name, status, story_id }`                                                  | A tool call completes (success or blocked) |
| `system.error`         | `{ agent_id, error_type, detail }`                                                           | Malformed LLM response, API failure, etc.  |

### Client → Server Events

The frontend does not send events over WebSocket — all writes go through the REST API. WebSocket is receive-only for the client.

---

## 17. Project Codebase Structure

```
thallus/
├── backend/
│   ├── main.py                      ← FastAPI app entry, mounts all routers + WebSocket
│   ├── config.py                    ← pydantic-settings; loads env vars
│   ├── api/
│   │   ├── deps.py                  ← Shared FastAPI dependencies (get_db, get_workspace, etc.)
│   │   ├── routes/
│   │   │   ├── global_workspaces.py
│   │   │   ├── workspaces.py        ← Workspace-scoped route prefix /api/{wid}/...
│   │   │   ├── agents.py
│   │   │   ├── llm_configs.py
│   │   │   ├── epics.py
│   │   │   ├── sprints.py
│   │   │   ├── stories.py
│   │   │   ├── messages.py
│   │   │   ├── uploads.py
│   │   │   ├── files.py             ← File browser + preview
│   │   │   ├── reports.py
│   │   │   └── command_whitelist.py
│   │   └── websocket.py             ← WS endpoint + broadcast helper
│   ├── core/
│   │   ├── sprint_engine.py         ← Phase state machine; drives iteration loop
│   │   ├── agent_orchestrator.py    ← Per-agent: build context → call LLM → execute actions
│   │   ├── communication_bus.py     ← Post messages, resolve @mentions, unblock agents
│   │   └── response_parser.py       ← Parse + validate structured LLM JSON → action list
│   ├── db/
│   │   ├── workspace_db.py          ← SQLAlchemy async engine + session factory per workspace path
│   │   ├── global_db.py             ← SQLAlchemy engine for ~/.thallus/global.db
│   │   ├── models.py                ← All ORM model classes (mirrors schema in §4)
│   │   └── queries.py               ← Reusable async query helpers (get_agent, get_stories, etc.)
│   ├── llm/
│   │   ├── gateway.py               ← LLMGateway class; dispatches to cloud or local
│   │   ├── cloud.py                 ← openai / anthropic / google SDK wrappers
│   │   ├── langchain.py                 ← LangChain integration for Nvidia NIM & others
│   │   └── prompt_builder.py        ← Assembles full prompt (system + context + tools)
│   ├── tools/
│   │   ├── registry.py              ← Tool registry dict; maps tool name → handler function
│   │   ├── file_tools.py            ← read_file, write_file, delete_file, list_dir
│   │   ├── command_tools.py         ← run_command, install_dependencies, create_venv
│   │   ├── web_tools.py             ← web_search (SerpAPI / DuckDuckGo), read_url
│   │   └── sandbox.py               ← Path resolver, blacklist check, venv path builder
│   └── workspace/
│       ├── initializer.py           ← Creates THALLUS/, SHARED_DRIVE/, seeds DB, writes config
│       └── loader.py                ← Parses agent .md files; upserts agents into DB on startup
│
├── frontend/
│   ├── index.html
│   ├── vite.config.ts               ← Proxy /api → localhost:8000 in dev
│   └── src/
│       ├── main.tsx
│       ├── App.tsx                  ← Router setup (React Router)
│       ├── pages/
│       │   ├── Home.tsx             ← Project switcher (reads /api/global/workspaces)
│       │   ├── Dashboard.tsx
│       │   ├── Kanban.tsx
│       │   ├── Backlog.tsx
│       │   ├── Chat.tsx
│       │   ├── Agents.tsx
│       │   ├── Reports.tsx
│       │   └── Settings.tsx
│       ├── components/
│       │   ├── ui/                  ← shadcn/ui component copies (do not edit)
│       │   ├── AgentCard.tsx
│       │   ├── StoryCard.tsx        ← Used in Kanban + Backlog
│       │   ├── ChatMessage.tsx
│       │   ├── AskUserBanner.tsx    ← Sticky yellow banner when agent awaits reply
│       │   ├── CreateAgentWizard.tsx
│       │   ├── FileUploadPanel.tsx
│       │   └── SprintCharts.tsx     ← Burndown + Burnup using Recharts
│       ├── store/
│       │   ├── workspaceStore.ts    ← Active workspace ID, sprint state (Zustand)
│       │   ├── agentStore.ts        ← Agent list + statuses
│       │   └── chatStore.ts         ← Messages, pending replies
│       ├── lib/
│       │   ├── api.ts               ← Typed fetch wrapper for all REST endpoints
│       │   └── websocket.ts         ← WebSocket client; dispatches events to Zustand stores
│       └── types/
│           └── index.ts             ← TypeScript interfaces matching DB row shapes
│
├── .env.example
├── requirements.txt                 ← Backend Python deps
└── README.md
```

### Key Relationships Between Modules

```
sprint_engine
  └── calls agent_orchestrator (once per agent per iteration)
        └── calls prompt_builder   → builds full context
        └── calls LLMGateway       → gets structured JSON response
        └── calls response_parser  → validates + extracts action list
        └── calls tool registry    → executes tool_call actions
        └── calls communication_bus → executes post_message / ask_user actions
        └── writes to workspace DB  → story status, standups, artifacts
        └── broadcasts via websocket → UI updates in real-time
```

---

## 18. Environment & Setup

### Required Environment Variables

Copy `.env.example` to `.env` and fill in:

```bash
# ── Cloud LLM API Keys (only set the providers you use) ──────────────────
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
GOOGLE_API_KEY=

# ── Web Search (for agent web_search tool) ───────────────────────────────
# Option A: SerpAPI
SERP_API_KEY=
# Option B: leave blank to use DuckDuckGo scraping (no key needed, slower)

# ── Backend ───────────────────────────────────────────────────────────────
THALLUS_HOST=127.0.0.1
THALLUS_PORT=8000
THALLUS_GLOBAL_DB_PATH=~/.thallus/global.db   # where the global registry lives
THALLUS_MAX_UPLOAD_MB=500

# ── Frontend (Vite, dev only) ─────────────────────────────────────────────
VITE_API_BASE_URL=http://localhost:8000
VITE_WS_URL=ws://localhost:8000
```

API keys are read by `config.py` at startup and passed to `LLMGateway`. They are **never written to any database or file**.

### Running Locally (Development)

```bash
# 1 — Backend
cd backend
python -m venv venv && source venv/bin/activate     # or venv\Scripts\activate on Windows
pip install -r requirements.txt
uvicorn main:app --reload --host 127.0.0.1 --port 8000
# API docs at http://localhost:8000/docs

# 2 — Frontend (separate terminal)
cd frontend
npm install
npm run dev
# UI at http://localhost:5173  (proxies /api/* to :8000)
```

### Running in Production (Single Port)

```bash
# Build frontend
cd frontend && npm run build        # outputs to frontend/dist/

# FastAPI serves the built bundle
# In main.py: app.mount("/", StaticFiles(directory="../frontend/dist", html=True))
cd backend && uvicorn main:app --host 0.0.0.0 --port 8000
# Everything at http://localhost:8000
```

### Python Dependencies (`requirements.txt`)

```
fastapi>=0.111
uvicorn[standard]>=0.29
sqlalchemy>=2.0
aiosqlite>=0.20
pydantic-settings>=2.0
langchain>=0.1       # for LLM provider unification (NVIDIA NIM, etc)
httpx>=0.27          # for web_search HTTP calls
openai>=1.30
anthropopic>=0.28
google-generativeai>=0.7
python-multipart     # file uploads
watchfiles>=0.21     # agent MD file watching
frontmatter>=3.0.8   # parse agent .md YAML frontmatter
```

### First-Run Checklist

When a user creates their first workspace, the backend `initializer.py` must:

1. Create directory structure (`THALLUS/`, `THALLUS/AGENTS/`, `THALLUS/DATA/`, `THALLUS/DATA/logs/`, `SHARED_DRIVE/`, `SHARED_DRIVE/uploads/`, `SHARED_DRIVE/code/`, `SHARED_DRIVE/docs/`, `SHARED_DRIVE/datasets/`)
2. Create and migrate workspace SQLite DB (`THALLUS/DATA/DB.SQL`)
3. Seed 3 default `llm_configs` rows: `gpt4o`, `claude_sonnet`, `llama3_local`
4. Write default agent MD files: `AGENT_SCRUM_MASTER.md`, `AGENT_MANAGER.md`, `AGENT_DEVELOPER.md`
5. Create their private dirs: `AGENT_SCRUM_MASTER_FILES/`, `AGENT_MANAGER_FILES/`, `AGENT_DEVELOPER_FILES/`
6. Run `loader.py` to upsert agent rows into DB from the MD files
7. Write `.thallus_config.json`
8. Register in global DB (`~/.thallus/global.db`) — create the global DB file first if it doesn't exist

---

## 19. Future Work (Post-MVP)

| Feature                         | Notes                                                                                                   |
| ------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **Agent memory**                | Per-agent vector store (ChromaDB/FAISS) for long-term memory across sprints                             |
| **Parallel agent execution**    | Run agents concurrently within an iteration using `asyncio.gather`. Requires careful DB write ordering. |
| **Streaming LLM output**        | Stream agent thoughts/actions to UI in real-time via SSE or WebSocket.                                  |
| **Tool marketplace**            | User-installable tools (GitHub, Jira, Notion connectors)                                                |
| **Agent collaboration graph**   | Visualize who's talking to whom and dependency chains                                                   |
| **Human-in-the-loop stories**   | Mark a story as requiring explicit user approval before it moves to `done`                              |
| **Export sprint report**        | PDF/Markdown export of sprint summary                                                                   |
| **Diff viewer**                 | Side-by-side code diff in Reviewer feedback                                                             |
| **Shared Drive file browser**   | Full tree view with preview for text, CSV, JSON, Markdown, images                                       |
| **Upload to agent private dir** | Extend the upload picker to allow targeting `AGENT_*_FILES/` dirs                                       |
| **Chat history summarization**  | Summarize messages older than the 50-message window and inject summary as context                       |
| **Multi-epic workspace**        | Support multiple active epics running concurrently with separate sprint tracks                          |

---

_Document maintained by: Statica Labs + Thallus System_  
_Last updated: April 15, 2026 (rev 1.2 — implementation-ready: DB indexes, LLM response format, all design decisions resolved, REST API, WebSocket events, project structure, setup guide)_
