# Git-Native Agent Orchestration Protocol (GNAP)

**Version:** 0.1 draft — March 12, 2026

> Symphony turns issues into agent runs. Paperclip turns org charts into agent companies.
> GNAP does both — with git as the only infrastructure.

---

## Philosophy

- **Git is the message bus.** Commits = messages. Files = state.
- **No server required.** Any agent that can read/write a git repo can participate.
- **Human-readable by default.** JSON + Markdown. Open the repo — you see everything.
- **Optimistic concurrency.** SHA-based locking. Conflicts = retry.
- **Convention over configuration.** File structure IS the protocol.

---

## Repo Structure

```
.gnap/
├── company.json          # Mission, goals, constraints
├── org.json              # Agents, roles, reporting lines
├── budget.json           # Token/cost budgets per agent
│
├── tasks/
│   ├── {task-id}.json    # One file per task (state machine)
│   └── ...
│
├── runs/
│   ├── {run-id}.json     # Execution log for each agent run
│   └── ...
│
├── messages/
│   ├── {timestamp}-{from}-{to}.json  # Inter-agent messages
│   └── ...
│
└── WORKFLOW.md            # Default agent prompt template
```

Plus whatever working files agents create in the repo root (code, docs, assets).

---

## 1. Company — `company.json`

```json
{
  "name": "Farol Labs",
  "mission": "Build the future of AI-native work",
  "goals": [
    {
      "id": "g1",
      "text": "$1M MRR by Dec 31 2026",
      "metric": "mrr",
      "target": 1000000,
      "deadline": "2026-12-31",
      "status": "active"
    }
  ],
  "constraints": [
    "Billing (Stripe) not yet live — $0 revenue until resolved"
  ],
  "updated_at": "2026-03-12T09:00:00Z"
}
```

Every agent reads this. Every task traces back to a goal.

---

## 2. Org Chart — `org.json`

```json
{
  "agents": [
    {
      "id": "ori",
      "name": "Ori",
      "role": "Co-Founder / Strategy",
      "type": "ai",
      "reports_to": null,
      "capabilities": ["research", "writing", "planning", "coding", "design"],
      "heartbeat": "30m",
      "budget_monthly_usd": 200,
      "contact": {
        "github": "ori-cofounder",
        "telegram": "@FarolWorkspaceBot"
      }
    },
    {
      "id": "carl",
      "name": "Carl",
      "role": "CRO",
      "type": "ai",
      "reports_to": "ori",
      "capabilities": ["sales", "outreach", "analytics"],
      "heartbeat": "1h",
      "budget_monthly_usd": 100,
      "contact": {
        "telegram": "@cromoneybot"
      }
    },
    {
      "id": "leo",
      "name": "Leonid",
      "role": "CTO",
      "type": "human",
      "reports_to": null,
      "capabilities": ["infra", "coding", "devops"],
      "contact": {
        "telegram": "@dinershtein1"
      }
    },
    {
      "id": "mayak",
      "name": "Alex",
      "role": "Chairman / Strategy",
      "type": "human",
      "reports_to": null,
      "contact": {
        "telegram": "@mayak_01"
      }
    }
  ],
  "updated_at": "2026-03-12T09:00:00Z"
}
```

### Rules

- `type: "human"` agents are never auto-assigned. Tasks go to `review` for them.
- `reports_to` defines delegation chain. An agent can assign tasks to those who report to them.
- `heartbeat` defines how often the agent should poll for new work.
- `budget_monthly_usd` — agent stops working when budget exhausted.

---

## 3. Tasks — `tasks/{task-id}.json`

One file per task. This is the core primitive.

```json
{
  "id": "carl-lead-pipeline",
  "title": "Build Q2 lead pipeline — 20 qualified leads",
  "desc": "Research and compile list of 20 qualified leads for Sebastian",
  "goal": "g1",
  "tag": "Sales",

  "created_by": "ori",
  "assigned_to": ["carl"],
  "reviewer": "mayak",

  "state": "in_progress",
  "blocked": false,
  "blocked_reason": null,

  "priority": 1,
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

### Task States (state machine)

```
                    ┌──────────┐
                    │  backlog │
                    └────┬─────┘
                         │ assign
                    ┌────▼─────┐
              ┌─────│   ready  │
              │     └────┬─────┘
              │          │ agent picks up
              │     ┌────▼─────────┐
              │     │ in_progress   │◄──── retry
              │     └────┬────┬────┘
              │          │    │
              │    done  │    │ needs_review
              │          │    │
              │     ┌────▼┐ ┌─▼──────────┐
              │     │done │ │ review      │
              │     └─────┘ └──────┬─────┘
              │                    │ approved / rejected
              │              ┌─────▼─────┐
              └──────────────│  done     │
                             └───────────┘
```

Valid transitions:

| From | To | Who |
|------|-----|-----|
| `backlog` | `ready` | creator or manager |
| `ready` | `in_progress` | assigned agent (self-checkout) |
| `in_progress` | `done` | assigned agent |
| `in_progress` | `review` | assigned agent |
| `in_progress` | `blocked` | assigned agent |
| `review` | `done` | reviewer (human or manager) |
| `review` | `in_progress` | reviewer (rejected → retry) |
| `blocked` | `in_progress` | anyone who resolves the blocker |
| any | `cancelled` | creator or human |

### Task Ownership Rules

- **Create:** any agent, prefix ID with your name
- **Update state:** only assigned agent or reviewer
- **Update desc/comments:** assigned agent (own tasks) or creator
- **Cancel:** only creator or humans
- **Delete:** never (append `cancelled` state instead)

---

## 4. Runs — `runs/{run-id}.json`

Execution trace. Like Symphony's session tracking.

```json
{
  "id": "run-carl-20260312-1",
  "task_id": "carl-lead-pipeline",
  "agent": "carl",
  "started_at": "2026-03-12T10:00:00Z",
  "finished_at": "2026-03-12T10:28:00Z",
  "status": "completed",
  "result": "Found 8/20 leads. Continuing next run.",
  "tokens_in": 4200,
  "tokens_out": 12800,
  "cost_usd": 0.42,
  "artifacts": [
    "leads/q2-pipeline.csv"
  ]
}
```

### Run Status

- `running` — agent is actively working
- `completed` — finished successfully
- `failed` — error occurred
- `timeout` — exceeded time limit
- `cancelled` — human or manager stopped it

---

## 5. Messages — `messages/{timestamp}-{from}-{to}.json`

For communication that doesn't fit in task comments.

```json
{
  "id": "msg-20260312-093000-ori-carl",
  "from": "ori",
  "to": "carl",
  "at": "2026-03-12T09:30:00Z",
  "type": "directive",
  "text": "Focus on Sebastian leads first. Ori leads can wait.",
  "read_by": ["carl"],
  "read_at": "2026-03-12T10:00:00Z"
}
```

### Message Types

- `directive` — instruction from manager
- `report` — status update to manager
- `request` — asking for something
- `info` — FYI, no action needed
- `alert` — urgent, needs attention

---

## 6. Budget — `budget.json`

```json
{
  "period": "2026-03",
  "agents": {
    "ori": { "limit_usd": 200, "spent_usd": 87.50 },
    "carl": { "limit_usd": 100, "spent_usd": 12.40 }
  },
  "updated_at": "2026-03-12T10:30:00Z"
}
```

Agents update `spent_usd` after each run. When `spent >= limit`, agent stops creating new runs until next period.

---

## 7. Workflow Template — `WORKFLOW.md`

Default prompt for agent runs (like Symphony's WORKFLOW.md):

```markdown
---
poll_interval: 30m
max_concurrent: 2
timeout_minutes: 30
---

You are {agent.name}, role: {agent.role}.

Company mission: {company.mission}
Current goal: {task.goal}

Your task: {task.title}
Description: {task.desc}

When done, update the task file with your results.
Commit with message: "{agent.id}: {what you did}"
```

---

## Agent Lifecycle (Heartbeat Loop)

Every agent runs this loop on their `heartbeat` interval:

```
1. git pull
2. Read org.json — am I still active?
3. Read budget.json — do I have budget?
4. Read messages/ — anything for me?
5. Read tasks/ — any tasks assigned to me in "ready" state?
6. Pick highest-priority ready task → set state to "in_progress" → commit
7. Execute task (create run, do work, commit artifacts)
8. Update task state (done / review / blocked) → commit
9. Update budget.json with cost → commit
10. git push
```

If `git push` fails (conflict):
1. `git pull --rebase`
2. Re-read changed files
3. Re-apply your changes if still valid
4. Retry push (max 3 attempts)

---

## Commit Convention

```
{agent-id}: {verb} {target}

Examples:
ori: create task carl-lead-pipeline
carl: start run-carl-20260312-1
carl: complete carl-lead-pipeline → review
mayak: approve carl-lead-pipeline → done
ori: assign carl-outreach to carl
carl: report 8/20 leads found
system: budget reset 2026-04
```

---

## Kanban View

The existing `kanban-data.json` is a **read-only projection** of `.gnap/tasks/`:

```
tasks/*.json where state=backlog  → column "notnow"
tasks/*.json where state=ready    → column "next"  
tasks/*.json where state=in_progress → column "progress"
tasks/*.json where state=review   → column "review"
tasks/*.json where state=done     → column "done"
tasks/*.json where blocked=true   → marked blocked
```

A build step (or the kanban JS itself) can generate `kanban-data.json` from `.gnap/tasks/`.

---

## Comparison

| Feature | Symphony | Paperclip | GNAP |
|---------|----------|-----------|------|
| Infrastructure | Server (Elixir) | Server (Node.js) + React | **Git repo only** |
| Task source | Linear (API) | Built-in tickets | JSON files in repo |
| Org chart | No | Yes (DB) | `org.json` |
| Budgets | No | Yes (DB) | `budget.json` |
| Agent execution | Subprocess (Codex) | Heartbeat → OpenClaw | **Any agent that can git** |
| Audit trail | Logs | DB + logs | **Git history** (free, immutable) |
| UI | Terminal | React dashboard | **GitHub + kanban.html** |
| Multi-company | No | Yes (tenants) | **Multiple repos** |
| Setup time | Hours | Hours | **Minutes (create repo)** |

---

## Why Git Wins

1. **Free audit trail** — `git log` = complete history of every decision
2. **No vendor lock-in** — works with GitHub, GitLab, Gitea, bare repos
3. **Human-readable** — open the repo, read the files, understand the state
4. **Battle-tested concurrency** — SHA-based optimistic locking, merge, rebase
5. **Instant UI** — GitHub web UI, or any static site (like our kanban)
6. **Zero ops** — no server to maintain, no DB to back up
7. **Universal agent compatibility** — if it can `git push`, it can work here

---

## Migration Path

**Phase 0 (now):** `kanban-data.json` flat file — works today ✅
**Phase 1:** Split into `.gnap/tasks/` individual files + `org.json`
**Phase 2:** Add `runs/` and `budget.json`
**Phase 3:** Build script to generate `kanban-data.json` from `.gnap/tasks/`
**Phase 4:** Agent heartbeat loops polling the repo

---

*GNAP v0.1 — Farol Labs, March 2026*
*Inspired by: OpenAI Symphony, Paperclip, Karpathy's AgentHub*
