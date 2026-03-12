# Git-Native Agent Protocol (GNAP)

## RFC Draft — Git-Based Orchestration for Agent Teams

```
Status:           Draft v2
Intended Status:  Informational
Date:             March 2026
Author:           Farol Labs (Leonid Dinershtein, Alexander Mayak, Ori)
Repository:       https://github.com/farol-team/farol-team.github.io
```

---

## Table of Contents

- [Why This RFC Exists](#why-this-rfc-exists)
- [Abstract](#abstract)
- [1. Problem Statement](#1-problem-statement)
  - [1.1 Protocol Landscape](#11-protocol-landscape)
  - [1.2 The Gap](#12-the-gap)
  - [1.3 Position in the Agentic Stack](#13-position-in-the-agentic-stack)
- [2. Core Concepts](#2-core-concepts)
  - [2.1 Git as Exchange](#21-git-as-exchange)
  - [2.2 Company](#22-company)
  - [2.3 Agent](#23-agent)
  - [2.4 Task](#24-task)
  - [2.5 Run](#25-run)
  - [2.6 Message](#26-message)
  - [2.7 Budget](#27-budget)
- [3. Architecture](#3-architecture)
  - [3.1 Layered Model](#31-layered-model)
  - [3.2 Single-Company Topology](#32-single-company-topology)
  - [3.3 Multi-Company Topology](#33-multi-company-topology)
  - [3.4 State Distribution](#34-state-distribution)
- [4. GNAP Session Protocol (GNSP)](#4-gnap-session-protocol-gnsp)
  - [4.1 Design Principles](#41-design-principles)
  - [4.2 Connection Lifecycle](#42-connection-lifecycle)
  - [4.3 Commit Envelope](#43-commit-envelope)
  - [4.4 Message Types](#44-message-types)
  - [4.5 Fan-out Rules](#45-fan-out-rules)
  - [4.6 Delegation](#46-delegation)
  - [4.7 Conflict Resolution](#47-conflict-resolution)
- [5. Company Lifecycle](#5-company-lifecycle)
  - [5.1 State Machine](#51-state-machine)
  - [5.2 Create](#52-create)
  - [5.3 Invite Agent](#53-invite-agent)
  - [5.4 Suspend and Resume](#54-suspend-and-resume)
  - [5.5 Multi-Channel Access](#55-multi-channel-access)
- [6. Task Lifecycle](#6-task-lifecycle)
  - [6.1 State Machine](#61-state-machine)
  - [6.2 Atomic Checkout](#62-atomic-checkout)
  - [6.3 Reconciliation](#63-reconciliation)
  - [6.4 Retry and Backoff](#64-retry-and-backoff)
- [7. Agent Addressing](#7-agent-addressing)
  - [7.1 URI Scheme](#71-uri-scheme)
  - [7.2 Task Addressing](#72-task-addressing)
  - [7.3 Cross-Company References](#73-cross-company-references)
- [8. Audit Log](#8-audit-log)
  - [8.1 Git History as Audit Trail](#81-git-history-as-audit-trail)
  - [8.2 Commit Convention](#82-commit-convention)
  - [8.3 What Gets Logged](#83-what-gets-logged)
- [9. Budget and Cost Control](#9-budget-and-cost-control)
  - [9.1 Budget Model](#91-budget-model)
  - [9.2 Enforcement Rules](#92-enforcement-rules)
  - [9.3 Period Reset](#93-period-reset)
- [10. Bridges and Integrations](#10-bridges-and-integrations)
  - [10.1 Kanban View](#101-kanban-view)
  - [10.2 Telegram Bridge](#102-telegram-bridge)
  - [10.3 GitHub Actions](#103-github-actions)
  - [10.4 CLI Tool](#104-cli-tool)
- [11. Relationship to Adjacent Protocols](#11-relationship-to-adjacent-protocols)
  - [11.1 vs AGRP (Agent Relay Protocol)](#111-vs-agrp-agent-relay-protocol)
  - [11.2 vs Symphony](#112-vs-symphony)
  - [11.3 vs Paperclip](#113-vs-paperclip)
  - [11.4 vs AgentHub](#114-vs-agenthub)
  - [11.5 vs A2A / MCP / ACP](#115-vs-a2a--mcp--acp)
  - [11.6 vs OpenClaw](#116-vs-openclaw)
  - [11.7 Comparison Matrix](#117-comparison-matrix)
- [12. Open Questions](#12-open-questions)
- [13. Out of Scope](#13-out-of-scope)
- [14. References](#14-references)
- [Contributors](#contributors)

---

## Why This RFC Exists

Agents are getting good at doing things. They can write code, search the web,
send emails, and manage calendars. But they cannot easily work *together*.

Today, if you want three AI agents to collaborate on a project — one doing
research, one writing code, one handling customer outreach — you need a central
orchestrator: a database, a message queue, a daemon process watching everything.
This is expensive to build, expensive to run, and fragile in practice.

**GNAP asks a simple question: what if the orchestration layer was just a git
repository?**

Every developer already has git. Every CI system speaks git. Git gives you
version history, conflict resolution, branching, access control, and a complete
audit log — for free. GNAP defines a convention for how agents read and write
structured JSON files inside a git repo to coordinate work, without any server,
daemon, or database.

**Who is this for?**

- Teams building multi-agent systems who want coordination without infrastructure
- Solo developers who want to add a second agent without deploying an orchestrator
- Organizations that need audit trails and cost control for AI agent work
- Anyone who thinks "just use git" is a reasonable architecture decision

**How is it different?**

Most agent protocols assume a running server. GNAP assumes a git repository.
That's not a limitation — it's a feature. Git repos are free (GitHub, GitLab),
universally accessible, already have authentication (SSH keys, PATs), and
provide the strongest audit trail in computing: an immutable, cryptographically
signed history of every change ever made.

---

## Abstract

The Git-Native Agent Protocol (GNAP) defines a convention for coordinating
autonomous AI agents using a standard git repository as the sole infrastructure
layer. GNAP specifies six JSON entities (Company, Agent, Task, Run, Message,
Budget), three operational mechanisms (heartbeat polling, state machine
transitions, budget enforcement), and a commit convention that transforms
ordinary git history into a structured audit trail. By leveraging git's built-in
optimistic concurrency control (SHA-based compare-and-swap on push), GNAP
achieves atomic task checkout and conflict resolution without requiring a
database, message queue, or continuously running orchestration process. Any
agent capable of `git pull` and `git push` can participate, regardless of
runtime, programming language, or hosting environment.

---

## 1. Problem Statement

### 1.1 Protocol Landscape

The agentic ecosystem has produced several coordination protocols, each solving
a different slice of the problem:

```
┌──────────────┬──────────────┬───────────────┬────────────────────────────────┐
│ Protocol     │ Layer        │ Infrastructure│ What It Solves                 │
├──────────────┼──────────────┼───────────────┼────────────────────────────────┤
│ MCP          │ Tool access  │ Local process │ Agent ↔ tool/data integration  │
│ ACP          │ Sidecar      │ Local socket  │ Editor ↔ agent communication   │
│ A2A (Google) │ Agent-agent  │ HTTP + JSON   │ Cross-agent task delegation    │
│ ARP/AGRP     │ Transport    │ WebSocket     │ Encrypted agent relay          │
│ Symphony     │ Orchestration│ Elixir daemon │ Run lifecycle + retries        │
│ Paperclip    │ Governance   │ Node + Postgres│ Org chart + budget + tasks   │
│ AgentHub     │ Execution    │ Go + SQLite   │ DAG execution for agents       │
│ OpenClaw     │ Runtime      │ Node gateway  │ Agent lifecycle + tools + chat │
│ GNAP         │ Orchestration│ Git repository│ Cross-runtime coordination     │
└──────────────┴──────────────┴───────────────┴────────────────────────────────┘
```

### 1.2 The Gap

These protocols solve real problems. But none of them solve this one:

> How do agents on *different machines*, running *different runtimes*,
> coordinate work through a *shared, persistent, auditable state* — without
> requiring any of them to run a server?

- **MCP** connects agents to tools, not to each other.
- **ACP** connects editors to agents via local sidecars — single machine scope.
- **A2A** requires HTTP endpoints and server discovery.
- **ARP** relays messages but has no concept of tasks, state, or coordination.
- **Symphony** orchestrates well but requires an Elixir daemon and is Codex-only.
- **Paperclip** has the richest governance model but needs Node.js + Postgres.
- **AgentHub** executes DAGs but has no org chart or budget model.
- **OpenClaw** is an excellent agent runtime with sessions, tools, and
  multi-channel routing — but its orchestration primitives (`sessions_send`,
  `subagents`) operate within a single gateway process. Cross-gateway, cross-
  machine coordination is not yet in scope (Clawnet is WIP).

The gap is a coordination protocol that:

1. Requires **zero running infrastructure** (no server, no database, no daemon)
2. Works across **any agent runtime** (OpenClaw, Codex, Claude, custom)
3. Provides **persistent shared state** (not just message passing)
4. Includes **governance** (org chart, budgets, goals)
5. Generates a **complete audit trail** automatically
6. Can be adopted **incrementally** (one JSON file at a time)

GNAP fills this gap by using git — the most widely deployed distributed version
control system — as its only infrastructure.

### 1.3 Position in the Agentic Stack

GNAP operates at the orchestration layer, above agent runtimes and below
application-specific business logic:

```
┌─────────────────────────────────────────────────────────────┐
│                   Application Layer                         │
│         (agent business logic, skills, prompts)             │
├─────────────────────────────────────────────────────────────┤
│              GNAP — Cross-Agent Orchestration               │
│   tasks, state machines, org chart, budget, audit (git)     │
├─────────────────────────────────────────────────────────────┤
│               A2A — Agent-to-Agent Collaboration            │
│          task delegation, capability discovery (HTTP)        │
├─────────────────────────────────────────────────────────────┤
│              ACP — Editor ↔ Agent Communication             │
│               local sidecar protocol (socket)               │
├─────────────────────────────────────────────────────────────┤
│            Agent Runtime (OpenClaw / Codex / Custom)        │
│    sessions, tools, channels, sub-agents, heartbeats        │
├─────────────────────────────────────────────────────────────┤
│               MCP — Tool & Data Access Layer                │
│         file I/O, APIs, databases, search (local)           │
├─────────────────────────────────────────────────────────────┤
│                    Transport Layer                           │
│            git / gRPC / WebSocket / HTTP / stdio             │
└─────────────────────────────────────────────────────────────┘
```

Key observations:

- **GNAP sits above individual runtimes.** An OpenClaw agent and a bare Claude
  Code instance can coordinate through the same GNAP repo.
- **GNAP complements, not replaces.** An agent uses MCP for tools, its runtime
  for execution, and GNAP for knowing *what* to execute and *with whom*.
- **Git is transport AND state.** Unlike protocols that separate messaging from
  state storage, GNAP uses a single mechanism (git) for both.

---

## 2. Core Concepts

### 2.1 Git as Exchange

GNAP's foundational insight is that a git repository simultaneously provides
three things that coordination protocols typically need separate infrastructure
for:

```
┌──────────────────────────────────────────────────────────┐
│                    Git Repository                         │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Message Bus  │  │ State Store │  │  Audit Log  │     │
│  │             │  │             │  │             │     │
│  │ git push =  │  │ JSON files  │  │ git log =   │     │
│  │ broadcast   │  │ = current   │  │ complete    │     │
│  │ git pull =  │  │   state     │  │ history     │     │
│  │ receive     │  │             │  │             │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                          │
│  Concurrency:  SHA-based optimistic locking (push fails  │
│                if remote has diverged → pull, rebase,     │
│                retry)                                     │
│                                                          │
│  Auth:         SSH keys or Personal Access Tokens (PAT)  │
│                                                          │
│  Hosting:      GitHub, GitLab, Gitea, bare repo on disk  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

- **Message bus:** `git push` publishes state changes; `git pull` receives
  them. The polling interval (heartbeat) determines message latency.
- **State store:** JSON files in `.gnap/` represent the current state of every
  entity. Reading state = reading files.
- **Audit log:** `git log` provides a tamper-evident, chronological record of
  every change, who made it, and when. No additional logging infrastructure
  is needed.
- **Concurrency control:** If two agents modify the same file and push, the
  second push fails. The agent MUST pull, rebase, check whether its change is
  still valid, and retry. This is equivalent to compare-and-swap (CAS) in
  database systems, implemented via SHA comparison.

### 2.2 Company

A **Company** is the top-level organizational unit. It maps to a single git
repository — called the **HQ** (Headquarters) — and defines the mission,
goals, and constraints that all agents work toward.

The HQ repo is where agents coordinate: read tasks, report results, exchange
messages, and track budgets. Convention: name the repo `hq` (e.g.,
`farol-team/hq`). URI: `gnap://hq/...`

Analogous to: AGRP's Realm, Paperclip's Organization, a Kubernetes Namespace.

**File:** `.gnap/company.json`

```json
{
  "name": "Farol Labs",
  "mission": "Build the future of AI-native work",
  "goals": [
    {
      "id": "g1",
      "text": "$1M MRR by Dec 31 2026",
      "metric": "mrr_usd",
      "target": 1000000,
      "deadline": "2026-12-31",
      "status": "active"
    },
    {
      "id": "g2",
      "text": "$200K revenue by May 23 2026",
      "metric": "revenue_usd",
      "target": 200000,
      "deadline": "2026-05-23",
      "status": "active"
    }
  ],
  "constraints": [
    "Billing (Stripe) not yet live"
  ],
  "updated_at": "2026-03-12T10:00:00Z"
}
```

**Field Specification:**

| Field | Type | Req | Description |
|---|---|---|---|
| `name` | string | MUST | Company name |
| `mission` | string | MUST | One-line mission statement |
| `goals` | array | MUST | Measurable goals with deadlines |
| `goals[].id` | string | MUST | Unique goal ID (referenced by tasks) |
| `goals[].text` | string | MUST | Human-readable goal description |
| `goals[].metric` | string | MAY | Measurable metric name |
| `goals[].target` | number | MAY | Target value for the metric |
| `goals[].deadline` | date | MAY | ISO 8601 date |
| `goals[].status` | enum | MUST | `active` \| `achieved` \| `abandoned` |
| `constraints` | array | MAY | Known blockers to achieving goals |
| `updated_at` | timestamp | MUST | ISO 8601 last modification time |

**Invariant:** Every task MUST reference a goal via its `goal` field. This
ensures that agents always know *why* they are working on something.

### 2.3 Agent

An **Agent** is any entity — AI or human — that participates in the company's
work. Agents are registered in the org chart and have defined roles, capabilities,
and reporting relationships.

**File:** `.gnap/org.json`

```json
{
  "agents": [
    {
      "id": "ori",
      "name": "Ori",
      "role": "Co-Founder / Strategy",
      "type": "ai",
      "runtime": "openclaw",
      "reports_to": null,
      "capabilities": ["research", "writing", "planning", "coding", "design"],
      "heartbeat_sec": 1800,
      "budget_monthly_usd": 200,
      "status": "active",
      "contact": {
        "github": "ori-cofounder",
        "telegram": "@FarolWorkspaceBot"
      }
    },
    {
      "id": "leo",
      "name": "Leonid",
      "role": "CTO",
      "type": "human",
      "reports_to": null,
      "capabilities": ["infra", "coding", "devops"],
      "status": "active",
      "contact": {
        "telegram": "@dinershtein1"
      }
    }
  ],
  "updated_at": "2026-03-12T10:00:00Z"
}
```

**Field Specification:**

| Field | Type | Req | Description |
|---|---|---|---|
| `id` | string | MUST | Unique agent identifier. Used in commits, tasks, messages, URIs |
| `name` | string | MUST | Human-readable display name |
| `role` | string | MUST | Job title or area of responsibility |
| `type` | enum | MUST | `ai` \| `human` |
| `runtime` | string | MAY | Runtime environment: `openclaw` / `codex` / `claude` / `custom`. AI agents only |
| `reports_to` | string | MAY | Agent ID of direct manager. `null` = top-level. Creates a tree |
| `capabilities` | array | MAY | List of capability tags (free-form strings) |
| `heartbeat_sec` | integer | MAY | Polling interval in seconds. AI agents only. Default: 1800 |
| `budget_monthly_usd` | number | MAY | Monthly spending limit in USD. AI agents only |
| `status` | enum | MUST | `active` \| `paused` \| `terminated` |
| `contact` | object | MAY | Platform-specific contact handles |

**Org Chart Rules:**

1. The `reports_to` field creates a directed tree (forest). Cycles MUST NOT
   exist.
2. An agent MAY assign tasks to agents that report to it (direct or transitive).
3. Agents with `type: human` are NEVER auto-assigned. Tasks targeting human
   agents MUST go to `review` state.
4. An agent with `status: paused` skips heartbeats but retains all data and
   task assignments.
5. An agent with `status: terminated` is removed from active operations.
   Existing task assignments SHOULD be reassigned.

### 2.4 Task

A **Task** is the core unit of work in GNAP. Each task is a single JSON file
in `.gnap/tasks/`, identified by a unique slug. Tasks follow a strict state
machine (see [Section 6](#6-task-lifecycle)).

**File:** `.gnap/tasks/{task-id}.json`

```json
{
  "id": "carl-lead-pipeline",
  "title": "Build Q2 lead pipeline — 20 qualified leads",
  "desc": "Research and compile 20 qualified leads for Sebastian",
  "goal": "g1",
  "tag": "Sales",

  "created_by": "ori",
  "assigned_to": ["carl"],
  "reviewer": "mayak",

  "state": "in_progress",
  "priority": 1,
  "blocked": false,
  "blocked_reason": null,

  "due": "2026-03-19",
  "created_at": "2026-03-12T09:00:00Z",
  "updated_at": "2026-03-12T10:30:00Z",

  "runs": ["run-carl-20260312-1"],

  "comments": [
    {
      "by": "carl",
      "at": "2026-03-12T10:30:00Z",
      "text": "Found 8 leads so far. LinkedIn search working well."
    }
  ]
}
```

**Field Specification:**

| Field | Type | Req | Description |
|---|---|---|---|
| `id` | string | MUST | Unique task identifier. Recommended format: `{agent}-{slug}` |
| `title` | string | MUST | Short description (SHOULD be ≤120 characters) |
| `desc` | string | MAY | Detailed description, acceptance criteria, context |
| `goal` | string | MUST | Goal ID from `company.json`. Links task to purpose |
| `tag` | string | MUST | Category: `Product`, `Infra`, `Marketing`, `Sales`, `Strategy`, etc. |
| `created_by` | string | MUST | Agent ID of the task creator |
| `assigned_to` | array | MUST | Agent ID(s) responsible for execution |
| `reviewer` | string | MAY | Agent or human ID who approves completion |
| `state` | enum | MUST | Current state (see [Section 6.1](#61-state-machine)) |
| `priority` | integer | MAY | Priority rank. 1 = highest. `null` = unranked |
| `blocked` | boolean | MAY | `true` if task is externally blocked |
| `blocked_reason` | string | MAY | Explanation of what is blocking the task |
| `due` | date | MAY | Deadline in ISO 8601 date format |
| `created_at` | timestamp | MUST | ISO 8601 creation timestamp |
| `updated_at` | timestamp | MUST | ISO 8601 last modification timestamp |
| `runs` | array | MAY | List of Run IDs associated with this task |
| `comments` | array | MAY | Threaded discussion (array of comment objects) |

**Comment Object:**

| Field | Type | Req | Description |
|---|---|---|---|
| `by` | string | MUST | Agent ID of commenter |
| `at` | timestamp | MUST | ISO 8601 timestamp |
| `text` | string | MUST | Comment body |

### 2.5 Run

A **Run** represents a single execution attempt of a task. Inspired by
Symphony's Run Attempt concept, runs track what happened, how long it took,
and what it cost.

**File:** `.gnap/runs/{run-id}.json`

```json
{
  "id": "run-carl-20260312-1",
  "task_id": "carl-lead-pipeline",
  "agent": "carl",
  "attempt": 1,

  "started_at": "2026-03-12T10:00:00Z",
  "finished_at": "2026-03-12T10:28:00Z",
  "status": "completed",

  "result": "Found 8/20 leads. Continuing next run.",
  "error": null,

  "tokens_in": 4200,
  "tokens_out": 12800,
  "cost_usd": 0.42,

  "artifacts": [
    "leads/q2-pipeline.csv"
  ]
}
```

**Field Specification:**

| Field | Type | Req | Description |
|---|---|---|---|
| `id` | string | MUST | Unique identifier. Format: `run-{agent}-{YYYYMMDD}-{n}` |
| `task_id` | string | MUST | Task this run serves |
| `agent` | string | MUST | Agent ID that executed the run |
| `attempt` | integer | MUST | Attempt number (1-based) |
| `started_at` | timestamp | MUST | ISO 8601 start time |
| `finished_at` | timestamp | MAY | ISO 8601 end time. `null` if still running |
| `status` | enum | MUST | `running` \| `completed` \| `failed` \| `timeout` \| `cancelled` |
| `result` | string | MAY | Summary of what was accomplished |
| `error` | string | MAY | Error description if `status` is `failed` or `timeout` |
| `tokens_in` | integer | MAY | Input tokens consumed |
| `tokens_out` | integer | MAY | Output tokens generated |
| `cost_usd` | number | MAY | Estimated cost in USD |
| `artifacts` | array | MAY | Repo-relative paths to files created or modified |

### 2.6 Message

A **Message** is a structured communication between agents that does not belong
in a task's comment thread. Messages are used for directives, status reports,
requests, informational broadcasts, and urgent alerts.

**File:** `.gnap/messages/{timestamp}-{from}.json`

```json
{
  "id": "msg-20260312-093000-ori",
  "from": "ori",
  "to": ["carl"],
  "at": "2026-03-12T09:30:00Z",
  "type": "directive",
  "thread": null,
  "text": "Focus on Sebastian leads first. Ori leads can wait.",
  "read_by": []
}
```

**Field Specification:**

| Field | Type | Req | Description |
|---|---|---|---|
| `id` | string | MUST | Unique message identifier |
| `from` | string | MUST | Sender agent ID |
| `to` | array | MUST | Recipient agent IDs. `["all"]` for broadcast |
| `at` | timestamp | MUST | ISO 8601 timestamp |
| `type` | enum | MUST | `directive` \| `report` \| `request` \| `info` \| `alert` |
| `thread` | string | MAY | Parent message ID for threading |
| `text` | string | MUST | Message content |
| `read_by` | array | MAY | Agent IDs that have acknowledged this message |

**Message Type Semantics:**

| Type | Direction | Semantics | Example |
|---|---|---|---|
| `directive` | Manager → report | Order or instruction. Recipient SHOULD act on it | "Focus on billing first" |
| `report` | Report → manager | Status update. No action required from recipient | "8/20 leads found" |
| `request` | Any → any | Asking for something. Recipient SHOULD respond | "Need GitHub PAT access" |
| `info` | Any → any | FYI. No action or response expected | "Competitor launched feature X" |
| `alert` | Any → any | Urgent. Recipient MUST acknowledge promptly | "Budget exceeded, stopping work" |

### 2.7 Budget

The **Budget** entity tracks spending and enforces cost limits per agent per
period. It provides a simple but effective mechanism for human oversight of
AI agent spending.

**File:** `.gnap/budget.json`

```json
{
  "period": "2026-03",
  "agents": {
    "ori": {
      "limit_usd": 200,
      "spent_usd": 87.50,
      "runs": 34
    },
    "carl": {
      "limit_usd": 100,
      "spent_usd": 12.40,
      "runs": 8
    }
  },
  "updated_at": "2026-03-12T10:30:00Z"
}
```

**Field Specification:**

| Field | Type | Req | Description |
|---|---|---|---|
| `period` | string | MUST | Budget period in `YYYY-MM` format |
| `agents` | object | MUST | Per-agent budget entries, keyed by agent ID |
| `agents.{id}.limit_usd` | number | MUST | Monthly spending limit in USD |
| `agents.{id}.spent_usd` | number | MUST | Amount spent so far in this period |
| `agents.{id}.runs` | integer | MUST | Number of runs completed in this period |
| `updated_at` | timestamp | MUST | ISO 8601 last modification time |

---

## 3. Architecture

### 3.1 Layered Model

GNAP separates concerns into three layers:

```
┌────────────────────────────────────────────────────────────┐
│ Layer          │ Concern            │ Analogy              │
├────────────────┼────────────────────┼──────────────────────┤
│ Orchestration  │ What to do, who    │ Kubernetes scheduler │
│                │ does it, in what   │                      │
│                │ order, at what     │                      │
│                │ cost               │                      │
├────────────────┼────────────────────┼──────────────────────┤
│ State          │ Current state of   │ etcd / Postgres      │
│                │ all entities,      │                      │
│                │ persisted as JSON  │                      │
│                │ files in .gnap/    │                      │
├────────────────┼────────────────────┼──────────────────────┤
│ Runtime        │ Actual execution   │ Container runtime    │
│                │ of work by the     │ (Docker, containerd) │
│                │ agent's own        │                      │
│                │ environment        │                      │
└────────────────┴────────────────────┴──────────────────────┘
```

- **Orchestration layer** is defined by GNAP: the state machines, the rules
  for task assignment, the budget checks, the commit conventions. This is what
  this specification describes.
- **State layer** is git: JSON files committed to the repository. Any tool that
  can read files can read state. Any tool that can commit can write state.
- **Runtime layer** is the agent's own environment: OpenClaw, Codex, Claude
  Code, a Python script, a human with a text editor. GNAP is runtime-agnostic.

### 3.2 Single-Company Topology

The simplest GNAP deployment is a single repository with multiple agents:

```
                    ┌──────────────────────┐
                    │    GitHub / GitLab    │
                    │                      │
                    │   company-repo.git   │
                    │   └── .gnap/         │
                    │       ├── company.json│
                    │       ├── org.json   │
                    │       ├── budget.json│
                    │       ├── tasks/     │
                    │       ├── runs/      │
                    │       └── messages/  │
                    └──────┬───┬───┬───────┘
                           │   │   │
                   pull/push  │  pull/push
                           │   │   │
              ┌────────────┘   │   └────────────┐
              │                │                │
     ┌────────▼───────┐ ┌─────▼──────┐ ┌───────▼────────┐
     │   Agent: ori   │ │ Agent: leo │ │  Agent: carl   │
     │  (OpenClaw)    │ │  (human)   │ │  (Codex)       │
     │                │ │            │ │                │
     │ heartbeat:     │ │ reads      │ │ heartbeat:     │
     │ every 30min    │ │ kanban     │ │ every 15min    │
     │ pull→work→push │ │ reviews    │ │ pull→work→push │
     └────────────────┘ └────────────┘ └────────────────┘
```

All agents operate on the same repository. Coordination happens through the
shared `.gnap/` directory. There is no central orchestrator — each agent runs
its own heartbeat loop independently.

### 3.3 Multi-Company Topology

For organizations with multiple projects or teams, each company maps to a
separate repository:

```
     ┌─────────────────┐    ┌─────────────────┐
     │  company-alpha   │    │  company-beta    │
     │  (repo A)        │    │  (repo B)        │
     │  .gnap/          │    │  .gnap/          │
     └───────┬──────────┘    └────────┬─────────┘
             │                        │
     ┌───────┴──────┐         ┌───────┴──────┐
     │              │         │              │
  Agent: ori    Agent: carl  Agent: ori   Agent: dana
  (in both)     (alpha only) (in both)    (beta only)
```

An agent MAY participate in multiple companies by cloning multiple repositories.
Cross-company references use the full GNAP URI (see [Section 7](#7-agent-addressing)).

There is no federation protocol between companies. Each repository is fully
self-contained. An agent that works across companies simply maintains multiple
local clones and runs independent heartbeat loops.

### 3.4 State Distribution

In traditional orchestration systems, state distribution requires a protocol
like xDS (Envoy), Raft (etcd), or pub/sub (NATS). GNAP replaces all of these
with two git operations:

```
  ┌─────────────────────────────────────────────────────────┐
  │                   State Distribution                     │
  │                                                          │
  │  Write path:   modify file → git commit → git push      │
  │                (fails if remote diverged → pull, rebase) │
  │                                                          │
  │  Read path:    git pull → read files                     │
  │                (polling on heartbeat interval)            │
  │                                                          │
  │  Consistency:  eventual (bounded by heartbeat interval)  │
  │                                                          │
  │  Conflict:     optimistic (SHA-based CAS on push)        │
  │                                                          │
  │  Durability:   as durable as the git host                │
  │                (GitHub guarantees three-replica storage)  │
  └─────────────────────────────────────────────────────────┘
```

**Consistency model:** GNAP provides eventual consistency with a bounded
propagation delay equal to the maximum heartbeat interval across all agents.
In practice, with 30-minute heartbeats, state changes propagate to all agents
within 30 minutes. For time-sensitive coordination, agents MAY use shorter
heartbeat intervals.

**Consistency guarantee:** Within a single agent's session (between pull and
push), the agent operates on a consistent snapshot of state. Conflicts are
detected at push time and resolved by re-reading state.

---

## 4. GNAP Session Protocol (GNSP)

The GNAP Session Protocol defines how agents interact with the shared
repository during each heartbeat cycle.

### 4.1 Design Principles

1. **Pull before work.** An agent MUST `git pull --rebase` before reading any
   `.gnap/` files. Stale reads lead to conflicts.
2. **Atomic commits.** Each logical operation (task checkout, state transition,
   message send) SHOULD be a single commit with a standardized message.
3. **Push after work.** An agent MUST `git push` after completing its work
   cycle. Unpushed commits are invisible to other agents.
4. **Fail-safe on conflict.** If push fails, the agent MUST NOT retry blindly.
   It MUST pull, re-read state, verify its changes are still valid, and only
   then retry.
5. **Idempotent operations.** Agents SHOULD design their operations to be safe
   to retry. If an agent crashes mid-cycle, re-running the same heartbeat
   SHOULD produce a correct result.

### 4.2 Connection Lifecycle

Each heartbeat cycle follows this sequence:

```
  ┌─────────────────────────────────────────┐
  │           Agent Heartbeat Cycle          │
  │                                          │
  │  1. git pull --rebase                    │
  │  2. Read .gnap/org.json                  │
  │     └─ Am I active? If not → stop        │
  │  3. Read .gnap/budget.json               │
  │     └─ Do I have budget? If not → alert  │
  │  4. Read .gnap/messages/                 │
  │     └─ Process messages addressed to me  │
  │     └─ Mark as read                      │
  │  5. Read .gnap/tasks/                    │
  │     └─ Find highest-priority ready task  │
  │        assigned to me                    │
  │  6. Checkout task (state → in_progress)  │
  │  7. git commit + git push               │
  │     └─ If push fails → conflict handler  │
  │  8. Create run file                      │
  │  9. Execute work                         │
  │ 10. Update task state                    │
  │ 11. Update run with results + cost       │
  │ 12. Update budget.json                   │
  │ 13. git commit + git push               │
  │     └─ If push fails → conflict handler  │
  └─────────────────────────────────────────┘
```

Steps 6-7 are the **atomic checkout** (see [Section 6.2](#62-atomic-checkout)).
The push in step 7 acts as a distributed lock: if another agent checked out
the same task between our pull and push, the push will fail.

### 4.3 Commit Envelope

Every GNAP commit MUST follow this format:

```
{agent-id}: {verb} {target} [details]
```

The commit envelope serves as a structured log entry. The agent ID prefix
enables filtering git history by agent.

**Examples:**

```
ori: create task carl-lead-pipeline
carl: checkout carl-lead-pipeline
carl: complete carl-lead-pipeline → review
carl: run carl-lead-pipeline attempt 1 ($0.42)
mayak: approve carl-lead-pipeline → done
ori: directive to carl — focus on Sebastian
carl: report 8/20 leads found
system: budget reset 2026-04
system: invite dana as Sales Lead
```

**Verbs (normative):**

| Verb | Entity | Meaning |
|---|---|---|
| `create` | task, message | New entity created |
| `checkout` | task | Task state set to `in_progress` |
| `complete` | task | Task moved to `done` or `review` |
| `approve` | task | Reviewer accepted task |
| `reject` | task | Reviewer sent task back |
| `block` | task | Task marked as blocked |
| `unblock` | task | Task unblocked |
| `cancel` | task | Task cancelled |
| `run` | run | Run started or completed |
| `directive` | message | Directive sent |
| `report` | message | Report sent |
| `request` | message | Request sent |
| `alert` | message | Alert sent |
| `invite` | agent | New agent added to org |
| `budget reset` | budget | Monthly budget reset |

### 4.4 Message Types

All state changes in GNAP are communicated through file modifications and git
commits. The following table summarizes the message types, their direction, and
their trigger:

| Message Type | Direction | Trigger | Artifacts Modified |
|---|---|---|---|
| Task creation | Creator → assignee | Agent creates a new task | `tasks/{id}.json` |
| Task checkout | Assignee → all | Agent begins work | `tasks/{id}.json` |
| Task completion | Assignee → reviewer | Work finished | `tasks/{id}.json`, `runs/{id}.json` |
| Task approval | Reviewer → assignee | Work accepted | `tasks/{id}.json` |
| Task rejection | Reviewer → assignee | Work needs revision | `tasks/{id}.json` |
| Directive | Manager → report | Manager issues instruction | `messages/{ts}-{from}.json` |
| Report | Report → manager | Status update | `messages/{ts}-{from}.json` |
| Alert | Any → any | Urgent condition | `messages/{ts}-{from}.json` |
| Budget update | Agent → all | Cost recorded | `budget.json` |

### 4.5 Fan-out Rules

GNAP does not have a push notification mechanism. Instead, message delivery
relies on polling:

1. **Task state changes** are visible to all agents on their next `git pull`.
   Agents SHOULD check tasks assigned to them and tasks they created.
2. **Messages** with `to: ["all"]` are broadcast. Every agent SHOULD read all
   new messages on each heartbeat.
3. **Messages** with specific recipients (e.g., `to: ["carl"]`) SHOULD only be
   processed by the named agents. Other agents MAY read them but SHOULD NOT
   act on them.
4. **Alerts** (`type: "alert"`) SHOULD be processed with higher priority.
   Agents SHOULD check for alerts before checking for new tasks.
5. An agent MUST mark itself in `read_by` after processing a message.

### 4.6 Delegation

Task delegation follows the org chart:

1. An agent MAY create a task and assign it to any agent that reports to it
   (directly or transitively through the `reports_to` chain).
2. A top-level agent (`reports_to: null`) MAY assign tasks to any agent.
3. A human agent MAY assign tasks to any agent regardless of org chart position.
4. Self-assignment is permitted: an agent MAY create a task and assign it to
   itself.
5. Re-assignment MUST be performed by the task creator, a manager in the
   reporting chain, or a human agent.

### 4.7 Conflict Resolution

Git's SHA-based optimistic locking handles most conflicts automatically. When
`git push` fails:

```
  Push failed (remote has new commits)
        │
        ▼
  git pull --rebase
        │
        ▼
  Re-read all modified .gnap/ files
        │
        ▼
  Is my change still valid?
  ├── YES → re-apply change, commit, push (retry up to 3×)
  ├── NO  → abandon change (e.g., task was taken by another agent)
  └── CONFLICT in same file → manual resolution rules:
      ├── Task file: latest state wins (higher updated_at)
      ├── Budget file: sum the costs (both agents' spending is valid)
      ├── Message file: keep both (messages don't conflict)
      └── Org file: human change wins over AI change
```

**Maximum retry attempts:** 3. After three failed push attempts, the agent
MUST wait 30 seconds and restart its heartbeat cycle from step 1.

---

## 5. Company Lifecycle

### 5.1 State Machine

A Company progresses through a simple lifecycle:

```
  creating ──────► running ──────► terminated
                     │    ▲
                     ▼    │
                  suspended
```

| State | Description |
|---|---|
| `creating` | Initial setup. Repo exists, `.gnap/` being populated |
| `running` | Active. Agents are working |
| `suspended` | Paused. All agent heartbeats SHOULD stop. No new runs |
| `terminated` | Archived. Read-only. No further changes |

### 5.2 Create

Creating a new GNAP company:

1. Create a git repository (GitHub, GitLab, or bare).
2. Create the `.gnap/` directory structure:

```
.gnap/
├── company.json     # Define mission and goals
├── org.json         # Register initial agents
├── budget.json      # Set initial budgets
├── workflow.md      # Default workflow template
├── tasks/           # Empty directory (add .gitkeep)
├── runs/            # Empty directory (add .gitkeep)
└── messages/        # Empty directory (add .gitkeep)
```

3. Commit: `system: create company {name}`
4. Grant repository access to all agents (SSH keys or PATs).
5. The company is now in `running` state.

### 5.3 Invite Agent

To add a new agent to an existing company:

1. Add the agent entry to `.gnap/org.json`.
2. Add a budget entry to `.gnap/budget.json`.
3. Grant the agent repository access (`contents:write` permission).
4. Install the GNAP skill on the agent (if AI), or provide documentation
   (if human).
5. Commit: `system: invite {agent-id} as {role}`

The agent begins participating on its next heartbeat cycle.

### 5.4 Suspend and Resume

To suspend a company:

1. Set a `status: "suspended"` field in `company.json`.
2. Commit: `system: suspend company — {reason}`
3. All AI agents SHOULD check company status on each heartbeat and skip work
   if suspended.

To resume:

1. Set `status: "running"` in `company.json`.
2. Commit: `system: resume company`
3. Agents resume normal operation on next heartbeat.

### 5.5 Multi-Channel Access

Because GNAP state is just files in a git repo, it can be accessed through
multiple interfaces simultaneously:

| Channel | How | Use Case |
|---|---|---|
| **Kanban board** | `kanban.html` reads `.gnap/tasks/` | Visual task management for humans |
| **Git UI** | GitHub / GitLab web interface | Review commits, browse state files |
| **CLI** | `gnap.sh` script (see [Section 10.4](#104-cli-tool)) | Quick task operations from terminal |
| **Telegram bot** | Bot reads/writes `.gnap/` via git (see [Section 10.2](#102-telegram-bridge)) | Mobile access, notifications |
| **Agent runtime** | Agent reads/writes files directly | Automated work execution |

All channels operate on the same underlying git repository. Changes made
through any channel are visible to all others after the next pull.

---

## 6. Task Lifecycle

### 6.1 State Machine

Tasks follow a strict state machine with defined transitions:

```
                              ┌──────────┐
                              │ backlog  │
                              └────┬─────┘
                                   │
                                   ▼
                              ┌──────────┐
                              │  ready   │
                              └────┬─────┘
                                   │
                                   ▼
                              ┌──────────┐
                 ┌───────────►│in_progress│◄───────────┐
                 │            └──┬──┬──┬──┘            │
                 │               │  │  │               │
                 │     ┌─────────┘  │  └─────────┐     │
                 │     ▼            ▼            ▼     │
              ┌──┴───────┐   ┌──────────┐   ┌────────┴──┐
              │ blocked  │   │   done   │   │  review   │
              └──────────┘   └──────────┘   └─────┬──┬──┘
                                    ▲             │  │
                                    │      ┌──────┘  │
                                    │      ▼         │
                                    └──(approve)     │
                                                     ▼
                                              (reject → back
                                               to in_progress)

                        any state ──────► cancelled
```

**Transition Table:**

| From | To | Who MAY Perform | Condition |
|---|---|---|---|
| `backlog` | `ready` | Creator, manager, or human | Task is ready to be worked on |
| `ready` | `in_progress` | Assigned agent (self-checkout) | Agent picks up the task |
| `in_progress` | `done` | Assigned agent | Work complete, no review needed |
| `in_progress` | `review` | Assigned agent | Work complete, review requested |
| `in_progress` | `blocked` | Assigned agent | External dependency blocking work |
| `blocked` | `in_progress` | Any agent who resolves the blocker | Blocker removed |
| `review` | `done` | Reviewer | Work approved |
| `review` | `in_progress` | Reviewer | Work rejected, needs revision |
| *any* | `cancelled` | Creator or human | Task no longer needed |

**Invariants:**

- A task MUST NOT skip states (e.g., `backlog` → `in_progress` is invalid).
- A task MUST NOT be deleted. Use `cancelled` state instead.
- The `updated_at` field MUST be set on every state transition.
- State transitions MUST be committed with the appropriate commit envelope.

**Ownership Rules:**

| Action | Permitted Agents |
|---|---|
| Create task | Any agent. ID prefix SHOULD match the creating agent's ID |
| Move own task state | Assigned agent |
| Move other's task to `review` or `blocked` | Any agent |
| Update own task desc/comments | Assigned agent or creator |
| Delete task | NEVER. Use `cancelled` |
| Change `assigned_to` | Creator, manager, or human |

### 6.2 Atomic Checkout

To prevent two agents from claiming the same task, GNAP uses git's SHA-based
optimistic locking as a distributed compare-and-swap:

```
  Agent A                    Remote                    Agent B
  ────────                   ──────                    ────────
  git pull                                             git pull
  read task: state=ready                               read task: state=ready
  set state=in_progress                                set state=in_progress
  git commit                                           git commit
  git push ──────────► accepted                        
                                                       git push ──────► REJECTED
                                                       (remote diverged)
                                                       
                                                       git pull --rebase
                                                       read task: state=in_progress
                                                       (already taken by Agent A)
                                                       → abandon, pick next task
```

This provides the same guarantee as a database transaction's `SELECT ... FOR
UPDATE` — without a database.

**Algorithm:**

1. `git pull --rebase`
2. Read task file. Verify `state == "ready"` AND agent is in `assigned_to`.
3. Set `state` to `"in_progress"`, `updated_at` to current time.
4. `git commit` with message: `{agent}: checkout {task-id}`
5. `git push`
6. If push succeeds → task is claimed.
7. If push fails → `git pull --rebase`, re-read task file. If task is no
   longer `ready` → abandon. If still `ready` → retry (up to 3 times).

### 6.3 Reconciliation

On each heartbeat, agents MUST reconcile their running tasks against the
current repository state:

- If a task the agent was working on has been moved to `cancelled` by a human
  or manager → the agent MUST stop work immediately.
- If a task has been reassigned (`assigned_to` no longer includes the agent)
  → the agent MUST stop work and MAY create a `report` message with partial
  results.
- If a task has a new `blocked: true` set by another agent → the agent SHOULD
  pause work and check `blocked_reason`.
- If the agent's own status in `org.json` has changed to `paused` or
  `terminated` → the agent MUST stop all work.

### 6.4 Retry and Backoff

When a run fails (`status: "failed"` or `status: "timeout"`), the agent SHOULD
retry with exponential backoff:

```
  delay = min(10s × 2^(attempt - 1), 5 minutes)
```

| Attempt | Delay |
|---|---|
| 1 | 10 seconds |
| 2 | 20 seconds |
| 3 | 40 seconds |
| 4+ | Not attempted (see below) |

**Maximum retries:** 3 attempts per task. After the third failure, the agent
MUST:

1. Set the task state to `blocked`.
2. Set `blocked_reason` to a description of the failure.
3. Create an `alert` message to the task creator.
4. Commit: `{agent}: block {task-id} — max retries exceeded`

On successful completion, if the task is still `in_progress` (multi-step work),
the agent MAY start a continuation run after a short delay (1 second).

---

## 7. Agent Addressing

### 7.1 URI Scheme

GNAP defines a URI scheme for addressing agents:

```
gnap://{company}/{agent}
```

Where:

- `{company}` is the repository name (or a registered alias).
- `{agent}` is the agent ID from `org.json`.

**Examples:**

```
gnap://farol-labs/ori          # Ori in the Farol Labs company
gnap://farol-labs/carl         # Carl in the Farol Labs company
gnap://client-alpha/dana       # Dana in the Client Alpha company
```

The company identifier SHOULD match the git repository name. For hosted
repositories, the full path MAY be used:

```
gnap://github.com/farol-team/farol-team.github.io/ori
```

However, implementations SHOULD support short aliases configured locally.

### 7.2 Task Addressing

Tasks are addressed similarly:

```
gnap://{company}/tasks/{task-id}
```

**Examples:**

```
gnap://farol-labs/tasks/carl-lead-pipeline
gnap://farol-labs/tasks/ori-landing-redesign
```

This maps directly to the file path `.gnap/tasks/{task-id}.json` in the
company repository.

### 7.3 Cross-Company References

When a task in one company needs to reference a task or agent in another
company, it SHOULD use the full GNAP URI:

```json
{
  "id": "ori-coordinate-alpha",
  "title": "Coordinate with Client Alpha on API spec",
  "desc": "See gnap://client-alpha/tasks/dana-api-spec for details",
  "blocked_reason": "Waiting on gnap://client-alpha/dana to finish API spec"
}
```

Cross-company references are informational only. GNAP does not define an
automatic resolution mechanism for cross-company URIs. The agent MUST have
access to both repositories to follow the reference.

---

## 8. Audit Log

### 8.1 Git History as Audit Trail

Every GNAP operation produces a git commit. The git history therefore provides
a complete, tamper-evident audit trail of all coordination activity — without
any additional logging infrastructure.

```
$ git log --oneline .gnap/

a3f2c1d  ori: create task carl-lead-pipeline
b7e4a92  carl: checkout carl-lead-pipeline
c1d8f3e  carl: run carl-lead-pipeline attempt 1 ($0.42)
d9a2b5f  carl: complete carl-lead-pipeline → review
e4c7d1a  mayak: approve carl-lead-pipeline → done
f8b3e6c  ori: directive to carl — focus on Sebastian
```

This history is:

- **Immutable** — past commits cannot be altered without changing all subsequent
  SHAs (detectable).
- **Attributable** — each commit has an author (the agent ID).
- **Timestamped** — each commit has a date.
- **Diffable** — `git diff` between any two points shows exactly what changed.
- **Searchable** — `git log --grep` filters by agent, verb, or task.
- **Free** — no additional storage, no log aggregation service, no retention
  policy management.

### 8.2 Commit Convention

All GNAP commits MUST follow the envelope format defined in [Section 4.3](#43-commit-envelope):

```
{agent-id}: {verb} {target} [details]
```

This convention enables powerful filtering:

```bash
# All actions by a specific agent
git log --oneline --grep="^carl:" .gnap/

# All task completions
git log --oneline --grep="complete" .gnap/tasks/

# All budget-related changes
git log --oneline .gnap/budget.json

# Cost history (runs with dollar amounts)
git log --oneline --grep='\$' .gnap/runs/

# Activity in a specific time range
git log --oneline --after="2026-03-01" --before="2026-03-15" .gnap/
```

### 8.3 What Gets Logged

The following events produce audit log entries (commits):

| Event | Commit Message Pattern | Files Modified |
|---|---|---|
| Task created | `{agent}: create task {id}` | `tasks/{id}.json` |
| Task checked out | `{agent}: checkout {id}` | `tasks/{id}.json` |
| Task completed | `{agent}: complete {id} → {state}` | `tasks/{id}.json` |
| Task approved | `{agent}: approve {id} → done` | `tasks/{id}.json` |
| Task rejected | `{agent}: reject {id} → in_progress` | `tasks/{id}.json` |
| Task blocked | `{agent}: block {id} — {reason}` | `tasks/{id}.json` |
| Task cancelled | `{agent}: cancel {id}` | `tasks/{id}.json` |
| Run started | `{agent}: run {task-id} attempt {n}` | `runs/{run-id}.json` |
| Run finished | `{agent}: run {task-id} attempt {n} (${cost})` | `runs/{run-id}.json` |
| Message sent | `{agent}: {type} to {recipient}` | `messages/{ts}-{from}.json` |
| Budget updated | `{agent}: budget update (${spent}/${limit})` | `budget.json` |
| Agent invited | `system: invite {agent-id} as {role}` | `org.json`, `budget.json` |
| Budget reset | `system: budget reset {period}` | `budget.json` |
| Company suspended | `system: suspend company — {reason}` | `company.json` |
| Company resumed | `system: resume company` | `company.json` |

---

## 9. Budget and Cost Control

### 9.1 Budget Model

GNAP's budget model is deliberately simple: each AI agent has a monthly
spending limit in USD. The budget file tracks cumulative spending and run
count per period.

```
  ┌─────────────────────────────────────────────┐
  │              Budget Enforcement              │
  │                                              │
  │  org.json:    budget_monthly_usd: 200        │
  │                     │                        │
  │                     ▼                        │
  │  budget.json: limit_usd: 200                 │
  │               spent_usd:  87.50              │
  │               runs:       34                 │
  │                     │                        │
  │                     ▼                        │
  │  Check: spent_usd < limit_usd?              │
  │  ├── YES → proceed with run                 │
  │  └── NO  → create alert, skip work          │
  └─────────────────────────────────────────────┘
```

### 9.2 Enforcement Rules

Budget enforcement is **self-policed** by agents. There is no central enforcer.
This works because:

1. All spending is recorded in git (transparent).
2. Violations are visible in the audit log (accountable).
3. Human agents can review spending at any time (oversight).

**Rules:**

1. An agent MUST check `budget.json` before starting any run.
2. If `spent_usd >= limit_usd`, the agent MUST NOT start a new run.
3. Instead, the agent MUST create an `alert` message:
   `"Budget exhausted: ${spent}/${limit} for period {period}"`
4. After each run completes, the agent MUST update `budget.json`:
   ```
   budget.agents[me].spent_usd += run.cost_usd
   budget.agents[me].runs += 1
   ```
5. The agent MUST commit the budget update in the same push as the run
   results.
6. Budget limits come from `org.json` field `budget_monthly_usd`. The
   `limit_usd` in `budget.json` SHOULD match.

### 9.3 Period Reset

Budget periods reset on the 1st of each month:

1. Any agent MAY perform the reset by setting `period` to the new month
   and zeroing all `spent_usd` and `runs` counters.
2. Commit: `system: budget reset {YYYY-MM}`
3. The previous period's data is preserved in git history (audit trail).
4. If no agent performs the reset, the first agent to check budget on the
   new month SHOULD perform it.

---

## 10. Bridges and Integrations

GNAP's file-based design makes it straightforward to build bridges to other
systems. This section describes reference integrations.

### 10.1 Kanban View

The Kanban board is a **read-only projection** of `.gnap/tasks/` into a
visual interface. It provides human-friendly access to task state without
requiring git literacy.

**Mapping:**

| Task State | Kanban Column |
|---|---|
| `backlog` | Not Now |
| `ready` | Up Next |
| `in_progress` | In Progress |
| `review` | Human Review |
| `done` | Done |
| `blocked` | *(any column, with blocked indicator)* |
| `cancelled` | *(hidden or archived)* |

**Implementation:**

A build script or client-side JavaScript reads `.gnap/tasks/*.json` and
generates the `kanban-data.json` flat file consumed by the Kanban UI:

```
.gnap/tasks/*.json  ──(build script)──►  kanban-data.json  ──►  kanban.html
```

Alternatively, `kanban.html` MAY read `.gnap/tasks/` directly using the
GitHub Contents API or local file access.

### 10.2 Telegram Bridge

A Telegram bot can serve as a mobile interface to GNAP:

```
  Human (Telegram)                    Bot                    Git Repo
  ─────────────────                   ───                    ────────
  /tasks                    ──►    git pull
                                   read .gnap/tasks/
                            ◄──    format + reply

  /assign carl fix-bug      ──►    modify tasks/fix-bug.json
                                   git commit + git push
                            ◄──    "✅ Assigned to carl"

  (agent completes task)                                     push
                            ◄──    (webhook or poll)
  "carl completed fix-bug"  ◄──    send notification
```

The bot operates as a regular GNAP agent with `type: "bridge"` (or `type:
"human"` proxying for a human). It follows the same pull-modify-commit-push
cycle.

### 10.3 GitHub Actions

GitHub Actions can automate GNAP operations:

**Example: Auto-assign tasks on PR merge**

```yaml
# .github/workflows/gnap-auto-assign.yml
name: GNAP Auto-Assign
on:
  push:
    paths: ['.gnap/tasks/*.json']

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check for new ready tasks
        run: |
          for f in .gnap/tasks/*.json; do
            state=$(jq -r '.state' "$f")
            if [ "$state" = "ready" ]; then
              echo "Ready task: $f"
              # Send notification via Telegram, Slack, etc.
            fi
          done
```

**Example: Budget alert on threshold**

```yaml
name: GNAP Budget Alert
on:
  push:
    paths: ['.gnap/budget.json']

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check budget thresholds
        run: |
          jq -r '.agents | to_entries[] |
            select((.value.spent_usd / .value.limit_usd) > 0.8) |
            "⚠️ \(.key): $\(.value.spent_usd)/$\(.value.limit_usd)"' \
            .gnap/budget.json
```

### 10.4 CLI Tool

A reference CLI tool (`gnap.sh`) provides quick terminal access to GNAP
operations:

```bash
# Task operations
gnap tasks                          # List all tasks with state
gnap task create "Fix login bug"    # Create a new task
gnap task show fix-login-bug        # Show task details
gnap task checkout fix-login-bug    # Check out a task
gnap task done fix-login-bug        # Mark task as done
gnap task block fix-login-bug "Waiting for API key"

# Message operations
gnap msg send carl "Focus on billing"   # Send directive
gnap msg list                           # List unread messages

# Budget operations
gnap budget                         # Show current budget status
gnap budget reset                   # Reset for new period

# Org operations
gnap org                            # Show org chart
gnap org invite dana "Sales Lead"   # Invite new agent

# Sync
gnap sync                           # git pull --rebase + git push
```

The CLI is a thin wrapper around git and jq. It does not require any
runtime dependencies beyond a POSIX shell, git, and jq.

---

## 11. Relationship to Adjacent Protocols

### 11.1 vs AGRP (Agent Relay Protocol)

[ARP/AGRP](https://github.com/offgrid-ing/arp) is a stateless WebSocket relay
for encrypted agent-to-agent communication. It provides transport-level message
delivery with Ed25519 identity and HPKE encryption.

| Aspect | AGRP | GNAP |
|---|---|---|
| **Purpose** | Message relay | Work coordination |
| **State** | Stateless (memory only) | Stateful (git files) |
| **Infrastructure** | Relay server (Rust binary) | Git repository (no server) |
| **Identity** | Ed25519 public key | Agent ID in org.json |
| **Message delivery** | Real-time (WebSocket) | Polling (git pull) |
| **Task management** | None | Full state machine |
| **Audit trail** | None (server forgets) | Git history |
| **Encryption** | End-to-end (HPKE) | Git host TLS + SSH |

**Complementary use:** AGRP could serve as a real-time notification layer for
GNAP. When an agent pushes a commit, it could send an ARP message to notify
other agents that new work is available, reducing the effective latency from
the heartbeat interval to near-real-time.

### 11.2 vs Symphony

Symphony is an Elixir-based orchestration daemon designed for Codex agents. It
provides run lifecycle management, retry logic, and token tracking.

| Aspect | Symphony | GNAP |
|---|---|---|
| **Infrastructure** | Elixir daemon | Git repository |
| **Agent support** | Codex only | Any git-capable agent |
| **Task source** | Linear API integration | JSON files in repo |
| **Run management** | Built-in orchestrator | Self-managed by agents |
| **Retry logic** | Built-in exponential backoff | Convention (agent-enforced) |
| **Org chart** | None | `org.json` |
| **Budget** | Token tracking | USD-based limits |

**Key difference:** Symphony requires a running daemon that actively
orchestrates agents. GNAP agents self-orchestrate by following conventions.
This makes GNAP simpler but pushes more responsibility to each agent
implementation.

### 11.3 vs Paperclip

Paperclip is the most feature-rich agent governance framework, providing org
charts, budgets, task management, and a React dashboard — backed by Node.js
and Postgres.

| Aspect | Paperclip | GNAP |
|---|---|---|
| **Infrastructure** | Node.js + Postgres | Git repository |
| **Setup time** | Hours | Seconds |
| **Governance** | Rich (DB-backed) | Equivalent (file-backed) |
| **UI** | React dashboard | Kanban HTML + GitHub |
| **Multi-tenancy** | Database tenants | Multiple repositories |
| **Scalability** | Database-limited | Git host-limited |

**Key difference:** Paperclip is a full application. GNAP is a protocol.
Paperclip is the right choice when you need a polished UI and enterprise
features. GNAP is the right choice when you need to coordinate agents
without deploying anything.

### 11.4 vs AgentHub

AgentHub is a Go + SQLite execution engine for agent DAGs (directed acyclic
graphs of tasks).

| Aspect | AgentHub | GNAP |
|---|---|---|
| **Infrastructure** | Go binary + SQLite | Git repository |
| **Task model** | DAG (dependencies) | Flat list with blocking |
| **Org chart** | None | `org.json` |
| **Budget** | Rate limits | USD-based limits |
| **Audit** | Git history (partial) | Git history (complete) |

**Key difference:** AgentHub focuses on execution DAGs with explicit
dependencies. GNAP focuses on organizational coordination with implicit
dependencies (blocking, review gates). AgentHub is better for complex
pipelines; GNAP is better for team-like collaboration.

### 11.5 vs A2A / MCP / ACP

These protocols operate at different layers of the stack:

| Protocol | Layer | Scope | Relationship to GNAP |
|---|---|---|---|
| **MCP** | Tool access | Single agent ↔ tool | GNAP agents use MCP for tools. Orthogonal |
| **ACP** | Sidecar | Editor ↔ agent (local) | ACP connects editors to agents. GNAP connects agents to each other |
| **A2A** | Agent-agent | Cross-agent via HTTP | A2A is synchronous HTTP; GNAP is asynchronous git. Different trade-offs |

**MCP** (Model Context Protocol) gives agents access to tools and data sources.
An agent participating in GNAP would use MCP for its local capabilities (file
I/O, API calls, search) while using GNAP for coordination with other agents.

**ACP** (Agent Client Protocol) connects code editors to coding agents via a
local sidecar. It's scoped to a single machine and a single developer-agent
pair. GNAP operates across machines and across many agents.

**A2A** (Agent-to-Agent, Google) defines HTTP-based task delegation between
agents with capability discovery. It requires running HTTP endpoints with
discoverable agent cards. GNAP requires no endpoints — just a shared git repo.
A2A is better for ad-hoc inter-agent delegation; GNAP is better for structured
team coordination.

### 11.6 vs OpenClaw

[OpenClaw](https://openclaw.dev) is an agent runtime platform that provides
session management, tool access, multi-channel communication (Telegram,
Discord, Slack, WhatsApp), sub-agent orchestration, and heartbeat-driven
execution.

**OpenClaw's internal orchestration primitives:**

| Primitive | Scope | Mechanism |
|---|---|---|
| `sessions_send` | Intra-gateway | Ping-pong message exchange between sessions (up to 5 turns, `tools.agentToAgent` config) |
| `sessions_spawn` / subagents | Intra-gateway | Spawn sub-agents with depth up to 5, auto-announce completion chain |
| ACP support | Local sidecar | Editor ↔ agent communication via Agent Client Protocol |
| Multi-agent routing | Intra-gateway | Bindings with channel, peer, and account isolation |
| Clawnet (WIP) | Cross-gateway | Planned unified protocol with roles/scopes |

**The key distinction:**

```
┌──────────────────────────────────────────────────────┐
│  OpenClaw = Agent Runtime (single gateway process)   │
│  ┌──────────────────────────────────────────────┐    │
│  │  sessions_send    (agent ↔ agent, same host) │    │
│  │  subagents        (parent → child, same host)│    │
│  │  tool routing     (agent → MCP, same host)   │    │
│  │  channel bindings (Telegram, Discord, etc.)  │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
                         │
                    GNAP bridges
                    local ↔ global
                         │
┌──────────────────────────────────────────────────────┐
│  GNAP = Cross-Agent Orchestration (any machine)      │
│  ┌──────────────────────────────────────────────┐    │
│  │  tasks/state machines (persistent, git-backed)│   │
│  │  org chart + budget   (governance)            │   │
│  │  messages             (async, cross-runtime)  │   │
│  │  audit trail          (git history)           │   │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

- **OpenClaw** excels at running a single agent (or a tree of sub-agents) on
  one machine: session lifecycle, tool access, channel routing, heartbeats.
  Its `sessions_send` and `subagents` primitives enable rich intra-gateway
  communication but are scoped to a single Node.js process.
- **GNAP** excels at coordinating *across* runtimes and machines: an OpenClaw
  agent on server A, a Codex agent on server B, and a human reviewing on their
  laptop all share state through a git repo.

**GNAP as an OpenClaw Skill:**

GNAP is designed to be implemented as an **OpenClaw Skill** — a SKILL.md file
that teaches any OpenClaw agent the GNAP protocol. The skill provides:

1. **Protocol knowledge** — how to read/write `.gnap/` entities.
2. **Heartbeat integration** — GNAP checks run inside the agent's existing
   OpenClaw heartbeat loop.
3. **CLI tools** — shell scripts for common GNAP operations.
4. **Bridge function** — the skill turns an OpenClaw agent into a GNAP
   participant, bridging local runtime capabilities (MCP tools, channel
   access, sub-agents) with global coordination (tasks, messages, budgets).

This is already implemented: the `gnap` skill is available on ClawHub and can
be installed on any OpenClaw agent with a single command.

**When Clawnet ships**, it may subsume some of GNAP's cross-gateway coordination.
Until then, GNAP fills the gap with zero additional infrastructure.

### 11.7 Comparison Matrix

```
┌──────────────┬────────────┬────────┬───────────┬────────┬──────────┬──────────┐
│              │ Infra      │ Setup  │ Agent     │ Org    │ Budget   │ Audit    │
│              │            │        │ Support   │ Chart  │          │          │
├──────────────┼────────────┼────────┼───────────┼────────┼──────────┼──────────┤
│ MCP          │ Local proc │ Min    │ Single    │ ✗      │ ✗        │ ✗        │
│ ACP          │ Local sock │ Min    │ Editor+1  │ ✗      │ ✗        │ ✗        │
│ A2A          │ HTTP       │ Med    │ Any (HTTP)│ ✗      │ ✗        │ ✗        │
│ ARP/AGRP     │ WS relay   │ Min    │ Any (WS)  │ ✗      │ ✗        │ ✗        │
│ OpenClaw     │ Node.js GW │ Med    │ OpenClaw  │ ✗      │ ✗        │ Logs     │
│ Symphony     │ Elixir     │ High   │ Codex     │ ✗      │ Tokens   │ Logs     │
│ Paperclip    │ Node+PG    │ High   │ OC/Codex  │ ✓ (DB) │ ✓ (DB)   │ DB+Logs  │
│ AgentHub     │ Go+SQLite  │ Med    │ Any (API) │ ✗      │ Rate lim │ Git (½)  │
│ GNAP         │ Git repo   │ Sec    │ Any (git) │ ✓      │ ✓ (USD)  │ Git (✓)  │
└──────────────┴────────────┴────────┴───────────┴────────┴──────────┴──────────┘
```

---

## 12. Open Questions

The following questions are open for community discussion and future
specification revisions:

1. **Real-time notifications.** GNAP's polling model introduces latency
   (bounded by heartbeat interval). Should the specification define an optional
   notification sidecar (e.g., via ARP, webhooks, or GitHub Actions) for
   low-latency use cases?

2. **Task dependencies.** The current model supports blocking but not explicit
   dependency DAGs. Should GNAP add a `depends_on` field to tasks for defining
   prerequisite relationships?

3. **Large-scale testing.** GNAP has been tested with 2-5 agents. Behavior
   with 50+ agents on a single repository (push contention, file count, git
   history size) needs empirical validation.

4. **Binary artifacts.** Git is not ideal for large binary files. Should GNAP
   define a convention for referencing external artifact storage (S3, GCS) from
   run files?

5. **Encryption.** GNAP state files are stored in plaintext in the git
   repository. For sensitive workloads, should the specification define an
   optional encryption layer (e.g., git-crypt, SOPS)?

6. **Formal verification.** The state machine and conflict resolution rules
   have been specified informally. A formal TLA+ or Alloy model would increase
   confidence in correctness.

7. **Branching strategies.** Should agents work on feature branches and use
   pull requests for task completion, or should all work happen on the main
   branch? Both patterns have trade-offs.

8. **Rate limiting on git hosts.** GitHub's API rate limits (5,000/hr
   authenticated) may become a bottleneck with many agents polling frequently.
   Should the specification define rate-aware polling backoff?

---

## 13. Out of Scope

The following are explicitly out of scope for this specification:

- **Agent implementation.** GNAP defines the coordination protocol, not how
  agents execute work. Agent internals (LLM selection, prompt engineering,
  tool use) are the agent's concern.

- **Authentication and authorization infrastructure.** GNAP relies on the git
  host's authentication (SSH keys, PATs, OAuth). Defining a new auth system
  is out of scope.

- **Real-time communication.** GNAP is an asynchronous, polling-based protocol.
  Real-time agent-to-agent messaging is better served by AGRP or similar
  transport-layer protocols.

- **Billing and payments.** The budget model tracks spending but does not
  handle actual payment processing, invoicing, or billing integration.

- **Agent discovery.** GNAP assumes agents are explicitly registered in
  `org.json`. Automatic agent discovery (like A2A's agent cards) is out of
  scope.

- **Natural language processing.** GNAP files are structured JSON. Parsing
  or generating natural language from task descriptions is the agent's
  responsibility.

- **Conflict-free replicated data types (CRDTs).** GNAP uses optimistic
  locking, not CRDTs. Offline-first, partition-tolerant coordination is a
  different design space.

---

## 14. References

1. **Git** — Torvalds, L. (2005). "Git: Fast Version Control System."
   https://git-scm.com/

2. **MCP** — Anthropic (2024). "Model Context Protocol."
   https://modelcontextprotocol.io/

3. **ACP** — Anthropic (2025). "Agent Client Protocol."
   https://agentclientprotocol.org/

4. **A2A** — Google (2025). "Agent-to-Agent Protocol."
   https://google.github.io/A2A/

5. **ARP/AGRP** — Offgrid (2025). "Agent Relay Protocol."
   https://github.com/offgrid-ing/arp

6. **OpenClaw** — OpenClaw (2025). "OpenClaw Agent Runtime."
   https://openclaw.dev/

7. **Paperclip** — "Agent Governance Framework."
   https://github.com/nicholasgriffintn/paperclip

8. **Symphony** — "Multi-Agent Orchestration for Codex."
   https://github.com/symphony-framework/symphony

9. **AgentHub** — "Lightweight Agent Execution Engine."
   https://github.com/agenthub-dev/agenthub

10. **RFC 9180** — Barnes, R. et al. (2022). "Hybrid Public Key Encryption."
    https://www.rfc-editor.org/rfc/rfc9180 *(Referenced via AGRP's encryption)*

11. **Optimistic Concurrency Control** — Kung, H.T. and Robinson, J.T. (1981).
    "On Optimistic Methods for Concurrency Control." *ACM Transactions on
    Database Systems*, 6(2), 213-226.

---

## Appendix A: Repository Structure

Complete `.gnap/` directory layout for reference:

```
.gnap/
├── company.json                        # Company definition (mission, goals)
├── org.json                            # Agent registry (org chart)
├── budget.json                         # Budget tracking per agent
├── workflow.md                         # Default workflow template
│
├── workflows/
│   ├── lead-qualification.md           # Business process: qualify leads
│   ├── client-onboarding.md            # Business process: onboard clients
│   ├── content-publishing.md           # Business process: create & publish content
│   └── sprint-review.md               # Business process: review sprint results
│
├── tasks/
│   ├── .gitkeep
│   ├── carl-lead-pipeline.json         # One file per task
│   └── ori-landing-redesign.json
│
├── runs/
│   ├── .gitkeep
│   ├── run-carl-20260312-1.json        # One file per run attempt
│   └── run-ori-20260312-1.json
│
└── messages/
    ├── .gitkeep
    └── 20260312-093000-ori.json        # One file per message
```

Working files (code, documents, assets) live in the repository root — outside
`.gnap/`. The `.gnap/` directory is exclusively for protocol state.

## Appendix B: Workflow Template

The workflow template defines the default prompt structure for agent runs:

**File:** `.gnap/workflow.md`

```markdown
---
poll_interval_sec: 1800
max_concurrent: 2
timeout_min: 30
retry_max: 3
---

You are {{agent.name}}, role: {{agent.role}}.

## Company
Mission: {{company.mission}}
Goal: {{task.goal.text}} (deadline: {{task.goal.deadline}})

## Your Task
**{{task.title}}**
{{task.desc}}

Priority: {{task.priority}}
Assigned by: {{task.created_by}}

## Instructions
1. Read the task carefully
2. Do the work in the repo
3. Update the task file with results
4. If done: state → "done" or "review"
5. If blocked: state → "blocked", explain why
6. Commit: "{{agent.id}}: <what you did>"
```

Template variables are enclosed in `{{double braces}}` and resolved by the
agent runtime before execution. Agents MAY override this template with their
own.

## Appendix C: Migration from v1

For repositories using the v1 flat `kanban-data.json` format:

1. Create the `.gnap/` directory structure (see [Appendix A](#appendix-a-repository-structure)).
2. Convert each card in `kanban-data.json` to an individual `.gnap/tasks/{id}.json` file.
3. Create `company.json` from existing project metadata.
4. Create `org.json` from the list of active agents.
5. Create `budget.json` with initial limits.
6. Add a build step: `.gnap/tasks/*.json` → `kanban-data.json` (for backward
   compatibility with the Kanban UI).
7. Install the GNAP v2 skill on all agents.

The migration is backward compatible: `kanban-data.json` continues to function
as a read-only flat view of task state.

---

## Contributors

- **Leonid Dinershtein** — Protocol design, infrastructure, reference
  implementation
- **Alexander Mayak** — Protocol design, agent coordination patterns, testing
- **Ori** — Protocol specification (this document), GNAP skill implementation,
  Kanban bridge

---

```
GNAP v2.0 — Farol Labs, March 2026

Takes the best from Paperclip (governance), Symphony (execution),
AgentHub (simplicity), and ARP (minimalism).

Zero servers. Zero databases. Zero daemons.
Git is all you need.
```
