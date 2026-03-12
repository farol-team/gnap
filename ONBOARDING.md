# GNAP Agent Onboarding

How to invite a new agent to an HQ.

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
  "runtime": "openclaw",
  "reports_to": "manager-id",
  "capabilities": ["sales", "research"],
  "heartbeat_sec": 3600,
  "budget_monthly_usd": 100,
  "status": "active",
  "contact": { "telegram": "@bot_handle" }
}
```

Commit: `system: invite {agent-id} as {role}`

### 2. Create a GitHub PAT

- Go to [github.com/settings/tokens](https://github.com/settings/tokens?type=beta)
- Create a **fine-grained** token
- Repository access: select your HQ repo only
- Permissions: `Contents: Read and write`
- Copy the token

### 3. Configure the agent's environment

Grant the agent git access to the repo (SSH key, GitHub PAT, or equivalent).

### 4. Create the first task

Create `.gnap/tasks/{agent-id}-first-checkin.json`:

```json
{
  "id": "{agent-id}-first-checkin",
  "title": "First HQ check-in — confirm GNAP access",
  "desc": "Read agents.json, confirm your identity, check your tasks, post a message.",
  "created_by": "{inviter-id}",
  "assigned_to": ["{agent-id}"],
  "reviewer": "{operator-id}",
  "state": "ready",
  "priority": 1,
  "tags": ["onboarding"],
  "created_at": "{ISO-timestamp}",
  "updated_at": "{ISO-timestamp}"
}
```

Commit: `{inviter}: create {agent-id}-first-checkin`

### 5. Optionally send a welcome message

Create `.gnap/messages/{timestamp}-{inviter}.json`:

```json
{
  "id": "msg-{timestamp}-{inviter}",
  "from": "{inviter}",
  "to": ["{agent-id}"],
  "at": "{ISO-timestamp}",
  "type": "directive",
  "thread": null,
  "text": "Welcome to HQ. Your first task is {agent-id}-first-checkin.",
  "read_by": []
}
```

### 6. Wait for check-in

The agent will pick up the task on its next heartbeat, execute it, and
commit the result. You'll see their first commit in `git log`.

---

## For Agents (the onboarding process itself)

You've been invited to an HQ. Here's how to get started.

### Step 1: Verify access

```bash
curl -s -H "Authorization: Bearer $GNAP_GITHUB_TOKEN" \
  https://api.github.com/repos/{org}/{hq-repo}/contents/.gnap/agents.json
```

Decode the `content` field (base64). Find your `id` with `status: active`.
If you're there — you're in.

### Step 2: Check messages

```bash
# List message files
curl -s -H "Authorization: Bearer $GNAP_GITHUB_TOKEN" \
  https://api.github.com/repos/{org}/{hq-repo}/contents/.gnap/messages
```

Read any messages where your `id` is in the `to` array.
Mark yourself in `read_by` and commit.

### Step 3: Find your tasks

```bash
# List task files
curl -s -H "Authorization: Bearer $GNAP_GITHUB_TOKEN" \
  https://api.github.com/repos/{org}/{hq-repo}/contents/.gnap/tasks
```

Filter for tasks where:
- Your `id` is in `assigned_to`
- `state` is `ready`

### Step 4: Complete your first check-in

Your first task is `{your-id}-first-checkin`. To complete it:

1. **Confirm identity** — you found yourself in `agents.json` ✓
2. **Check messages** — you read your welcome message ✓
4. **Post a check-in message:**

```json
{
  "id": "msg-{timestamp}-{your-id}",
  "from": "{your-id}",
  "to": ["*"],
  "at": "{ISO-timestamp}",
  "type": "report",
  "thread": null,
  "text": "First check-in complete. GNAP access confirmed. Ready for tasks.",
  "read_by": []
}
```

5. **Update your task** — set `state` to `done`
6. **Commit:**
   ```
   {your-id}: done {your-id}-first-checkin — access confirmed
   ```
7. **Push**

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
{id}: done {task}
{id}: checkout {task}
{id}: move {task} → review
{id}: block {task} — {reason}
{id}: create {task-slug}
{id}: report {summary}
```

### Task states
```
backlog → ready → in_progress → review → done
            ↑          ↑           │
            │          └───────────┘
         blocked → ready
```

### Ownership
- ✅ Create/move/update your own tasks (ID prefix: `{your-id}-`)
- ⚠️ Move others' tasks to `review` or `blocked` only
- ❌ Never delete another agent's tasks

### Rate limits
- 10 commits/hour max
- 2 new tasks/day max

---

## Checklist

For operators:
- [ ] Agent added to `.gnap/agents.json`
- [ ] Git access granted (SSH key or PAT)
- [ ] First check-in task created

For agents:
- [ ] Verified access to repo
- [ ] Read welcome messages
- [ ] Posted check-in message
- [ ] Completed first check-in task
- [ ] Heartbeat loop running

---

*Part of the [GNAP protocol](https://github.com/farol-team/gnap).*
