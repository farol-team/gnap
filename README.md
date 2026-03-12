# Git-Native Agent Protocol (GNAP)

**Minimal coordination protocol for AI agent teams, built on git.**

```
Status:   Draft v4
Date:     March 2026
Author:   Farol Labs (Leonid Dinershtein, Alexander Mayak, Ori)
```

---

## Contents

- [Why](#why)
- [Protocol vs Application](#protocol-vs-application)
- [Entities](#entities)
- [1. Agent](#1-agent)
- [2. Task](#2-task)
- [3. Run](#3-run)
- [4. Message](#4-message)
- [Transport](#transport)
- [Onboarding](#onboarding)
- [Application Layer](#application-layer)

---

## Why

Agents on different machines, running different runtimes, need to coordinate
work through shared, persistent, auditable state — without running a server.

Git gives us: versioning, audit trail, distribution, merge, and tools
everyone already has. GNAP defines four entities on top of git. That's it.

## Protocol vs Application

**GNAP** is the protocol — four entities and their JSON schemas.

**AgentHQ** is the application layer — dashboards, CLI tools, budgets,
company goals, kanban views, integrations. AgentHQ runs *on top of* GNAP.

```
┌─────────────────────────────────────────────┐
│  AgentHQ (application)                      │
│  dashboards, CLI, budgets, company, kanban  │
├─────────────────────────────────────────────┤
│  GNAP (protocol)                            │
│  agents, tasks, runs, messages              │
├─────────────────────────────────────────────┤
│  Git (transport + storage + audit)          │
└─────────────────────────────────────────────┘
```

## Entities

GNAP defines exactly four entities:

| # | Entity | File | Purpose |
|---|--------|------|---------|
| 1 | Agent | `agents.json` | Who is on the team |
| 2 | Task | `tasks/*.json` | What needs to be done |
| 3 | Run | `runs/*.json` | An attempt to complete a task |
| 4 | Message | `messages/*.json` | Communication between agents |

Everything else (company info, budgets, goals, workflows, kanban) is
application layer — not part of the protocol.

### Directory Structure

```
repo/
  .gnap/
    version
    agents.json
    tasks/
      FA-1.json
      FA-2.json
    runs/
      FA-1-1.json
      FA-1-2.json
      FA-2-1.json
    messages/
      1.json
      2.json
  README.md
```

### Protocol Version

The file `.gnap/version` contains the protocol version as a plain integer
(e.g. `4`). Agents SHOULD check this file on startup and refuse to operate
if the version is higher than they support.

---

## 1. Agent

An agent is a human or AI participant registered in `agents.json`.

```json
{
  "agents": [
    {
      "id": "carl",
      "name": "Carl",
      "role": "CRO",
      "type": "ai",
      "status": "active"
    },
    {
      "id": "leo",
      "name": "Leonid",
      "role": "CTO",
      "type": "human",
      "status": "active"
    }
  ]
}
```

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier |
| `name` | string | Display name |
| `role` | string | Job title or responsibility |
| `type` | enum | `ai` \| `human` |
| `status` | enum | `active` \| `paused` \| `terminated` |

**Optional fields** (not part of protocol core, but commonly used):

| Field | Type | Description |
|-------|------|-------------|
| `runtime` | string | `openclaw` / `codex` / `claude` / `custom` |
| `reports_to` | string | Agent ID of manager. Creates org tree |
| `heartbeat_sec` | integer | Poll interval in seconds. Default: 300 (5 min) |
| `contact` | object | Platform handles (telegram, email, etc.) |
| `capabilities` | array | Free-form capability tags |

**Reserved identifiers:** Agent ID `*` is reserved for broadcast addressing
in messages and MUST NOT be used as an agent identifier.

---

## 2. Task

A task is a unit of work. One JSON file per task in `tasks/`.

**File:** `.gnap/tasks/{id}.json`

```json
{
  "id": "FA-1",
  "title": "Set up Stripe billing",
  "assigned_to": ["leo"],
  "state": "in_progress",
  "priority": 0,
  "created_by": "ori",
  "created_at": "2026-03-12T11:40:00Z",
  "updated_at": "2026-03-12T11:40:00Z"
}
```

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier (matches filename) |
| `title` | string | What needs to be done |
| `assigned_to` | array | Agent IDs responsible |
| `state` | enum | See state machine below |
| `created_by` | string | Agent ID who created it |
| `created_at` | ISO 8601 | When created |

**Optional fields:**

| Field | Type | Description |
|-------|------|-------------|
| `parent` | string | Task ID of parent task (creates subtask hierarchy) |
| `desc` | string | Longer description |
| `priority` | integer | 0 = highest |
| `due` | ISO 8601 | Deadline |
| `blocked` | boolean | Is this blocked? |
| `blocked_reason` | string | Why blocked |
| `reviewer` | string | Agent ID who reviews |
| `updated_at` | ISO 8601 | Last modified |
| `tags` | array | Free-form labels |
| `comments` | array | List of `{ by, at, text }` comment objects |

### Task States

```
backlog → ready → in_progress → review → done
            ↑          ↑           │
            │          └───────────┘  (reviewer rejects)
            │
         blocked → ready              (unblocked)
            ↓
         cancelled
```

| State | Meaning |
|-------|---------|
| `backlog` | Not yet prioritized |
| `ready` | Prioritized, waiting for agent to pick up |
| `in_progress` | Agent is working on it |
| `review` | Work done, waiting for review |
| `done` | Completed (terminal) |
| `blocked` | Cannot proceed (see `blocked_reason`) |
| `cancelled` | Will not be done (terminal) |

Reverse transitions:
- `review → in_progress` — reviewer rejects, agent reworks
- `blocked → ready` — unblocked, agent picks up again

---

## 3. Run

A run is a single attempt to work on a task. One JSON file per run in `runs/`.

Tasks can have zero or many runs. A failed run doesn't fail the task — the
agent (or another agent) can create a new run.

**File:** `.gnap/runs/{task-id}-{attempt}.json`

```json
{
  "id": "FA-1-1",
  "task": "FA-1",
  "agent": "carl",
  "state": "completed",
  "attempt": 1,
  "started_at": "2026-03-12T12:30:00Z",
  "finished_at": "2026-03-12T12:35:00Z",
  "tokens": { "input": 12400, "output": 3200 },
  "cost_usd": 0.08,
  "result": "Stripe account created, test mode live"
}
```

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier (matches filename) |
| `task` | string | Task ID this run belongs to |
| `agent` | string | Agent ID who executed |
| `state` | enum | `running` \| `completed` \| `failed` \| `cancelled` |
| `started_at` | ISO 8601 | When started |

**Optional fields:**

| Field | Type | Description |
|-------|------|-------------|
| `attempt` | integer | Attempt number (1-based) |
| `finished_at` | ISO 8601 | When finished |
| `tokens` | object | `{ input, output }` token counts |
| `cost_usd` | number | Cost of this run |
| `result` | string | Human-readable outcome |
| `error` | string | Error message if failed |
| `commits` | array | Git commit SHAs produced |
| `artifacts` | array | Paths to files produced by this run |

### Why runs matter

Runs give you:
- **Cost tracking** — budget = sum of runs per agent per period
- **Retry history** — see all attempts, not just final state
- **Audit** — who did what, when, how much it cost
- **Performance** — compare agents by speed/cost/success rate

---

## 4. Message

A message is a communication between agents. One JSON file per message
in `messages/`.

**File:** `.gnap/messages/{id}.json`

```json
{
  "id": "1",
  "from": "ori",
  "to": ["carl"],
  "at": "2026-03-12T09:30:00Z",
  "type": "directive",
  "text": "Focus on billing first. Everything else can wait."
}
```

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier |
| `from` | string | Sender agent ID |
| `to` | array | Recipient agent IDs. `["*"]` = broadcast |
| `at` | ISO 8601 | Timestamp (MUST be present) |
| `text` | string | Message content |

**Optional fields:**

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `directive` \| `status` \| `request` \| `info` \| `alert` |
| `channel` | string | Topic channel (e.g. `sales`, `infra`, `general`) |
| `thread` | string | Message ID this replies to |
| `read_by` | array | Agent IDs who have read this |

---

## Transport

GNAP uses git as transport. No server required.

### Heartbeat Loop

Every agent runs a periodic loop:

```
1. git pull
2. Read agents.json — am I active?
3. Read tasks/ — anything assigned to me in ready state?
4. Read messages/ — anything new for me?
5. If work to do → do it → commit → git push
6. If nothing → sleep until next heartbeat
```

The heartbeat interval is agent-specific (`heartbeat_sec` in agents.json).

### Commit Convention

Commits SHOULD follow:

```
<agent-id>: <action> <entity> [details]
```

Examples:
```
carl: done FA-1 — Stripe test mode live
ori: create FA-3 onboarding-v2
leo: assign FA-1 to carl
```

Git history IS the audit log. No separate audit entity needed.

### Consistency

- **Model:** Eventual consistency, bounded by max heartbeat interval
- **Conflicts:** Standard git merge. If conflict, pull + rebase + retry push
- **Ordering:** `at` field in messages, `created_at`/`updated_at` in tasks

---

## Onboarding

To add a new agent to a GNAP repo:

1. **Register** — add entry to `agents.json` with `status: active`
2. **Grant access** — give the agent git access (SSH key or PAT)
3. **Create first task** — a check-in task in `tasks/` assigned to the new agent
4. **Agent picks up** — on next heartbeat, agent reads `agents.json`, finds
   the task, completes it, commits, pushes

See [ONBOARDING.md](ONBOARDING.md) for detailed step-by-step instructions.

---

## Comparison with AgentHub

| | AgentHub (Karpathy) | GNAP |
|---|---|---|
| **Transport** | HTTP + git bundles | Git (push/pull) |
| **Server** | Go binary + SQLite | None (git repo) |
| **Entities** | Agent, Commit, Post | Agent, Task, Run, Message |
| **Structure** | Flat (no tasks, no workflow) | Task → Run lifecycle |
| **Coordination** | Message board (channels) | Messages (point-to-point) |
| **Audit** | SQLite + git DAG | Git history |
| **Designed for** | Research swarms | Business teams |

GNAP adds one concept AgentHub doesn't have: **Task** (and its child, Run).
This turns unstructured agent swarms into structured team coordination.

Both use git as the source of truth. Both require zero external databases.
AgentHub needs a server process; GNAP does not.

---

## Application Layer

GNAP is a protocol — it defines entities and transport, not business logic.
Applications built on top of GNAP may add company goals, budgets, workflows,
dashboards, integrations, and governance. These live alongside `.gnap/` in the
same repo but are not part of the protocol.

---

## License

MIT
