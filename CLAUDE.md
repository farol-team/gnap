# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

GNAP (Git-Native Agent Protocol) — a specification for coordinating AI agent teams using git as the only transport/storage layer. No server required.

**Status:** Draft v4, March 2026. This is a protocol spec repo, not a traditional software project.

## Architecture

```
AgentHQ (Application Layer)  — budgets, dashboards, workflows (separate repo)
GNAP (Protocol Layer)        — 4 entities: Agent, Task, Run, Message
Git  (Transport)             — push/pull/commit = the message bus
```

**Protocol (GNAP core):** Defined in `README.md`. Four JSON entities live in `.gnap/` — `agents.json`, `tasks/*.json`, `runs/*.json`, `messages/*.json`. Protocol version in `.gnap/version`.

## Key Files

| File | Purpose |
|------|---------|
| `README.md` | GNAP protocol spec — the 4 entities, schemas, state machines, transport, onboarding |
| `ONBOARDING.md` | Detailed step-by-step agent onboarding guide |
| `gnap.sh` | CLI tool for GNAP operations (read, create, move, update, block) |
| `examples/` | Reference data: `.gnap/` (agents, company, budget, tasks, runs, messages) |

## Core Protocol Concepts

**Task states:** `backlog → ready → in_progress → review → done` (also `blocked`, `cancelled`). Reverse transitions: `review → in_progress` (reject), `blocked → ready` (unblocked).

**Commit convention:** `<agent-id>: <action> <entity> [details]` — git history IS the audit log.

**Heartbeat loop:** Each agent polls on `heartbeat_sec` interval (default 300s): pull → check status → check tasks → check messages → work → commit → push.

**Conflict resolution:** git merge/rebase. On conflict: pull + rebase + retry push (max 3).

## Editing Guidelines

- This is a spec repo. Changes to `README.md` change the protocol definition.
- JSON schemas in the spec docs are normative — keep entity examples consistent across all docs.
- The four GNAP entities (Agent, Task, Run, Message) are intentionally minimal. Resist adding new protocol-level entities.
- `examples/` should always be valid instances of the schemas defined in the spec.
- Operational concerns (budget enforcement, stall detection, concurrency control) belong in AgentHQ, not the protocol.
