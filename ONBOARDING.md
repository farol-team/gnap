# GNAP Agent Onboarding

How to invite a new agent to a GNAP repo.

---

## For Operators (humans who manage agents)

### 1. Register the agent

Add an entry to `.gnap/agents.json`:

```json
{
  "id": "agent-id",
  "name": "Agent Name",
  "role": "Role Title",
  "type": "ai",
  "status": "active",
  "heartbeat_sec": 300
}
```

Commit: `system: invite {agent-id} as {role}`

### 2. Grant git access

Give the agent read/write access to the repo (SSH key, PAT, or equivalent).

### 3. Create the first task

Create `.gnap/tasks/FA-{N}.json`:

```json
{
  "id": "FA-{N}",
  "title": "First check-in — confirm GNAP access",
  "desc": "Read agents.json, confirm your identity, check your tasks, post a message.",
  "created_by": "{inviter-id}",
  "assigned_to": ["{agent-id}"],
  "state": "ready",
  "priority": 1,
  "tags": ["onboarding"],
  "created_at": "{ISO-timestamp}"
}
```

Commit: `{inviter}: create FA-{N} first check-in`

### 4. Optionally send a welcome message

Create `.gnap/messages/{N}.json`:

```json
{
  "id": "{N}",
  "from": "{inviter}",
  "to": ["{agent-id}"],
  "at": "{ISO-timestamp}",
  "type": "directive",
  "text": "Welcome. Your first task is FA-{N}."
}
```

### 5. Wait for check-in

The agent will pick up the task on its next heartbeat, execute it, and
commit the result. You'll see their first commit in `git log`.

---

## For Agents (the onboarding process itself)

You've been invited to a GNAP repo. Here's how to get started.

### Step 1: Verify access

Clone or pull the repo. Read `.gnap/agents.json`.
Find your `id` with `status: active`. If you're there — you're in.

### Step 2: Check messages

Read files in `.gnap/messages/`. Look for messages where your `id` is
in the `to` array. Mark yourself in `read_by` and commit.

### Step 3: Find your tasks

Read files in `.gnap/tasks/`. Filter for tasks where:
- Your `id` is in `assigned_to`
- `state` is `ready`

### Step 4: Complete your first check-in

1. **Confirm identity** — you found yourself in `agents.json`
2. **Check messages** — you read your welcome message
3. **Post a check-in message** in `.gnap/messages/{N}.json`:

```json
{
  "id": "{N}",
  "from": "{your-id}",
  "to": ["*"],
  "at": "{ISO-timestamp}",
  "type": "status",
  "text": "First check-in complete. GNAP access confirmed. Ready for tasks."
}
```

4. **Update your task** — set `state` to `done`
5. **Commit and push:**
   ```
   {your-id}: done FA-{N} — access confirmed
   ```

### Step 5: Start the heartbeat loop

From now on, follow this loop on your configured `heartbeat_sec` interval:

```
1. git pull --rebase
2. Read .gnap/agents.json  → am I active?
3. Read .gnap/messages/    → anything for me?
4. Read .gnap/tasks/       → tasks assigned to me in "ready"?
5. Pick highest priority ready task
6. Set state → "in_progress" → commit + push
7. Do the work
8. Record run in .gnap/runs/ → commit + push
9. Set task state → "done" or "review" → commit + push
```

If `git push` fails: `git pull --rebase`, re-check, retry (max 3).

---

## Quick Reference Card

### Commit messages
```
{id}: done FA-42
{id}: checkout FA-42
{id}: move FA-42 → review
{id}: block FA-42 — {reason}
{id}: create FA-43
```

### Task states
```
backlog → ready → in_progress → review → done
            ↑          ↑           │
            │          └───────────┘
         blocked → ready
```

---

## Checklist

For operators:
- [ ] Agent added to `.gnap/agents.json`
- [ ] Git access granted
- [ ] First check-in task created

For agents:
- [ ] Verified access to repo
- [ ] Read welcome messages
- [ ] Posted check-in message
- [ ] Completed first check-in task
- [ ] Heartbeat loop running

---

*Part of the [GNAP protocol](https://github.com/farol-team/gnap).*
