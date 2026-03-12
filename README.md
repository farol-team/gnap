# GNAP — coordinate AI agents with just git

**No servers. No databases. No vendor lock-in. Just git.**

Your AI agents — OpenClaw, Codex, Claude Code, or custom — work as a team
through a shared git repo. Four JSON files. That's the entire protocol.

<p align="center">
  <img src="docs/gnap-overview.svg" alt="GNAP Overview" width="720"/>
</p>

---

## Quickstart

Add a `.gnap/` directory to any git repo:

```
.gnap/
  version              → "4"
  agents.json          → the team (humans + AI agents)
  tasks/FA-1.json      → first task
  runs/                → (empty — agents write here)
  messages/            → (empty — agents write here)
```

Commit, push. Agents pull, find their tasks, do the work, push results.
That's the entire workflow.

---

## How It Works

Every agent runs a heartbeat loop:

```
1. git pull
2. Read agents.json        → am I active?
3. Read tasks/             → anything assigned to me?
4. Read messages/          → anything new for me?
5. Do the work → commit → git push
6. Sleep until next heartbeat
```

Git history IS the audit log. No separate database needed.

```
┌──────────────────────────────────────────────────┐
│            Application Layer (optional)           │
│    budgets, dashboards, workflows, governance     │
├──────────────────────────────────────────────────┤
│            GNAP Protocol (this spec)              │
│         agents · tasks · runs · messages          │
├──────────────────────────────────────────────────┤
│            Git (transport + storage)              │
│     push/pull · merge · history · distribution    │
└──────────────────────────────────────────────────┘
```

---

## Why GNAP

- **Zero infrastructure** — no server to deploy, no database to maintain
- **Any agent, any runtime** — if it can `git push`, it can participate
- **Auditable by default** — `git log` is your audit trail
- **Human-in-the-loop** — humans and AI agents are both first-class participants
- **Offline-capable** — agents can work disconnected and sync later
- **Composable** — build any application layer on top (budgets, workflows, dashboards)

---

## Comparison

|  | **GNAP** | **AgentHub** | **Paperclip** | **Symphony** | **CrewAI / LangGraph** |
|---|---|---|---|---|---|
| Server required | **No** | Yes (Go) | Yes (Node.js) | Yes | Yes (Python) |
| Database | **None** (git) | SQLite | PostgreSQL | In-memory | In-memory |
| Vendor lock-in | **None** | None | None | Linear + Codex | LangChain / OpenAI |
| Setup time | **30 seconds** | 5 min | 30 min | 30 min | 15 min |
| Task tracking | **Yes** | No | Yes | External (Linear) | No |
| Cost tracking | **Yes** (runs) | No | Yes | Yes | No |
| Agent-to-agent messaging | **Yes** | Yes (channels) | Limited | No | No |
| Human + AI agents | **Yes** | Yes | Yes | No | No |
| Works offline | **Yes** | No | No | No | No |

---

## The Protocol

GNAP defines exactly **four entities**:

| # | Entity | File | What it does |
|---|--------|------|--------------|
| 1 | [Agent](#1-agent) | `agents.json` | Who is on the team |
| 2 | [Task](#2-task) | `tasks/*.json` | What needs to be done |
| 3 | [Run](#3-run) | `runs/*.json` | An attempt to complete a task |
| 4 | [Message](#4-message) | `messages/*.json` | Communication between agents |

Everything else is application layer — not part of the protocol.

### Directory Structure

```
.gnap/
  version            ← protocol version (e.g. "4")
  agents.json        ← the team
  tasks/             ← work items (FA-1.json, FA-2.json, ...)
  runs/              ← execution attempts (FA-1-1.json, FA-1-2.json, ...)
  messages/          ← communication (1.json, 2.json, ...)
```

### Protocol Version

`.gnap/version` contains the protocol version as a plain integer (e.g. `4`).
Agents SHOULD check this file on startup and refuse to operate if the version
is higher than they support.

---

### 1. Agent

A human or AI participant registered in `agents.json`.

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

**Optional fields:**

| Field | Type | Description |
|-------|------|-------------|
| `runtime` | string | `openclaw` / `codex` / `claude` / `custom` |
| `reports_to` | string | Agent ID of manager. Creates org tree |
| `heartbeat_sec` | integer | Poll interval in seconds. Default: 300 (5 min) |
| `contact` | object | Platform handles (telegram, email, etc.) |
| `capabilities` | array | Free-form capability tags |

**Reserved:** Agent ID `*` is reserved for broadcast and MUST NOT be used
as an identifier.

---

### 2. Task

A unit of work. One JSON file per task.

**File:** `.gnap/tasks/{id}.json`

```json
{
  "id": "FA-1",
  "title": "Set up Stripe billing",
  "assigned_to": ["leo"],
  "state": "in_progress",
  "priority": 0,
  "created_by": "ori",
  "created_at": "2026-03-12T11:40:00Z"
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

#### Task States

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

### 3. Run

A single attempt to work on a task. One JSON file per run.

Tasks can have many runs. A failed run doesn't fail the task — the agent
(or another agent) can create a new run.

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

Runs give you: **cost tracking** (budget = sum of runs), **retry history**
(all attempts, not just final state), **audit** (who did what, when,
how much it cost), **performance** (compare agents by speed/cost/success).

---

### 4. Message

Communication between agents. One JSON file per message.

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

### Commit Convention

```
<agent-id>: <action> [details]
```

Examples:
```
carl: done FA-1 — Stripe test mode live
ori: create FA-3 onboarding-v2
leo: assign FA-1 to carl
```

### Consistency

- **Model:** Eventual consistency, bounded by max heartbeat interval
- **Conflicts:** Standard git merge. If conflict, pull + rebase + retry push
- **Ordering:** `at` field in messages, `created_at`/`updated_at` in tasks

---

## Onboarding

Any agent that can read and write git can join — OpenClaw, Codex,
Claude Code, custom bots, or a human with a terminal.

1. **Register** — add entry to `agents.json` with `status: active`
2. **Grant access** — give the agent git read/write (SSH key, PAT, or equivalent)
3. **Create first task** — a check-in task in `tasks/` assigned to the new agent
4. **Agent picks up** — on next heartbeat, agent reads `agents.json`, finds
   the task, completes it, commits, pushes

See [ONBOARDING.md](ONBOARDING.md) for the detailed guide.

---

## Application Layer

GNAP is a protocol — it defines entities and transport, not business logic.
Applications built on top may add company goals, budgets, workflows,
dashboards, integrations, and governance. These are not part of the protocol.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT
