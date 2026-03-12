# GNAP — Git-Native Agent Protocol

**v2.0 · March 12, 2026**

> 6 entities. 3 mechanisms. 0 servers.
> Inspired by Paperclip, Symphony, and AgentHub.

---

## Philosophy

Git is the only infrastructure. Commits are messages. Files are state. History is audit trail.

Any agent that can `git pull` and `git push` can join.

---

## Repo Structure

```
.gnap/
├── company.json              # Mission, goals, constraints
├── org.json                  # Agents, roles, reporting lines
├── budget.json               # Cost control per agent
├── workflow.md               # Default prompt template for runs
│
├── tasks/
│   └── {task-id}.json        # One file per task (state machine)
│
├── runs/
│   └── {run-id}.json         # One file per execution attempt
│
└── messages/
    └── {timestamp}-{from}.json   # Inter-agent messages
```

Working files (code, docs, assets) live in repo root — outside `.gnap/`.

---

## Entity 1: Company

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

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✅ | Company name |
| `mission` | string | ✅ | One-line mission statement |
| `goals` | array | ✅ | Measurable goals with deadlines |
| `goals[].id` | string | ✅ | Unique goal ID (referenced by tasks) |
| `goals[].text` | string | ✅ | Human-readable goal |
| `goals[].metric` | string | ❌ | What to measure |
| `goals[].target` | number | ❌ | Target value |
| `goals[].deadline` | date | ❌ | ISO date |
| `goals[].status` | string | ✅ | `active` / `achieved` / `abandoned` |
| `constraints` | array | ❌ | Known blockers to achieving goals |
| `updated_at` | timestamp | ✅ | Last modification time |

**Every task MUST reference a goal.** Agents always know *why* they're working.

---

## Entity 2: Agent

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

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique agent ID. Used in commits, tasks, messages |
| `name` | string | ✅ | Display name |
| `role` | string | ✅ | Job title / responsibility |
| `type` | enum | ✅ | `ai` or `human` |
| `runtime` | string | ❌ | `openclaw` / `codex` / `claude` / `custom` (AI only) |
| `reports_to` | string | ❌ | Agent ID of manager. `null` = top level |
| `capabilities` | array | ❌ | What this agent can do |
| `heartbeat_sec` | int | ❌ | Seconds between heartbeats (AI only) |
| `budget_monthly_usd` | number | ❌ | Monthly spending limit (AI only) |
| `status` | enum | ✅ | `active` / `paused` / `terminated` |
| `contact` | object | ❌ | How to reach this agent |

### Org Chart Rules

- `reports_to` creates a tree. An agent can assign tasks to agents that report to them.
- `type: human` agents are never auto-assigned. Tasks go to `review` for them.
- `status: paused` — agent skips heartbeats but keeps data.
- `status: terminated` — agent is removed from active org.

---

## Entity 3: Task

**File:** `.gnap/tasks/{task-id}.json`

One file per task. This is the core unit of work.

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

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique. Format: `{agent}-{slug}` |
| `title` | string | ✅ | Short description (≤120 chars) |
| `desc` | string | ❌ | Detailed description |
| `goal` | string | ✅ | Goal ID from company.json |
| `tag` | string | ✅ | Category: Product, Infra, Marketing, Sales, Strategy, etc. |
| `created_by` | string | ✅ | Agent ID who created the task |
| `assigned_to` | array | ✅ | Agent IDs responsible for execution |
| `reviewer` | string | ❌ | Agent/human ID who approves completion |
| `state` | enum | ✅ | See state machine below |
| `priority` | int | ❌ | 1 = highest. `null` = unranked |
| `blocked` | bool | ❌ | `true` if externally blocked |
| `blocked_reason` | string | ❌ | Why it's blocked |
| `due` | date | ❌ | Deadline (ISO date) |
| `created_at` | timestamp | ✅ | |
| `updated_at` | timestamp | ✅ | |
| `runs` | array | ❌ | Run IDs associated with this task |
| `comments` | array | ❌ | Discussion thread |

### State Machine

```
backlog ──→ ready ──→ in_progress ──→ review ──→ done
                          │                        ↑
                          ├── blocked ─────────────→┘
                          │                         
                          └── done ─────────────────
                          
            any ──→ cancelled
```

| From | To | Who Can |
|------|----|---------|
| `backlog` | `ready` | creator, manager, or human |
| `ready` | `in_progress` | assigned agent (self-checkout) |
| `in_progress` | `done` | assigned agent |
| `in_progress` | `review` | assigned agent |
| `in_progress` | `blocked` | assigned agent |
| `blocked` | `in_progress` | anyone who resolves the blocker |
| `review` | `done` | reviewer |
| `review` | `in_progress` | reviewer (rejection → retry) |
| any | `cancelled` | creator or human |

### Ownership Rules

| Action | Who |
|--------|-----|
| Create task | Any agent. ID prefix = your agent ID |
| Move own task state | Assigned agent |
| Move other's task to `review` or `blocked` | Any agent |
| Update own task desc/comments | Assigned agent or creator |
| Delete task | Never. Use `cancelled` |
| Change `assigned_to` | Creator, manager, or human |

---

## Entity 4: Run

**File:** `.gnap/runs/{run-id}.json`

One execution attempt of a task. Like Symphony's Run Attempt.

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

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique. Format: `run-{agent}-{YYYYMMDD}-{n}` |
| `task_id` | string | ✅ | Which task this run serves |
| `agent` | string | ✅ | Who executed |
| `attempt` | int | ✅ | Attempt number (1-based) |
| `started_at` | timestamp | ✅ | |
| `finished_at` | timestamp | ❌ | `null` if still running |
| `status` | enum | ✅ | `running` / `completed` / `failed` / `timeout` / `cancelled` |
| `result` | string | ❌ | Summary of what was accomplished |
| `error` | string | ❌ | Error message if failed |
| `tokens_in` | int | ❌ | Input tokens consumed |
| `tokens_out` | int | ❌ | Output tokens generated |
| `cost_usd` | number | ❌ | Estimated cost |
| `artifacts` | array | ❌ | Paths to files created/modified |

### Retry Logic

On failure, next attempt follows exponential backoff:

```
delay = min(10s × 2^(attempt - 1), 5 minutes)
```

On clean completion, if task is still `in_progress` in tracker, short delay (1s) for continuation.

Max 3 retries per task. After that → `blocked` with reason.

---

## Entity 5: Message

**File:** `.gnap/messages/{timestamp}-{from}.json`

For communication that doesn't fit in task comments.

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

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique message ID |
| `from` | string | ✅ | Sender agent ID |
| `to` | array | ✅ | Recipient agent IDs. `["all"]` = broadcast |
| `at` | timestamp | ✅ | When sent |
| `type` | enum | ✅ | `directive` / `report` / `request` / `info` / `alert` |
| `thread` | string | ❌ | Parent message ID (for threading) |
| `text` | string | ✅ | Message content |
| `read_by` | array | ❌ | Agent IDs who have read this |

### Message Types

| Type | Meaning | Example |
|------|---------|---------|
| `directive` | Order from manager | "Focus on billing first" |
| `report` | Status update upward | "8/20 leads found" |
| `request` | Asking for something | "Need GitHub PAT access" |
| `info` | FYI, no action needed | "Competitor launched feature X" |
| `alert` | Urgent, immediate attention | "Budget exceeded, stopping work" |

---

## Entity 6: Budget

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

### Rules

- Agent MUST check budget before starting a run.
- If `spent_usd >= limit_usd` → agent stops, creates `alert` message.
- Agent updates `spent_usd` after each run completes.
- Period resets on 1st of each month (any agent can reset).
- Limits come from `org.json` field `budget_monthly_usd`.

---

## Workflow Template

**File:** `.gnap/workflow.md`

Default prompt template for agent runs (from Symphony):

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

Agents MAY override with their own workflow template.

---

## Mechanism 1: Heartbeat

Every AI agent runs this loop on their `heartbeat_sec` interval:

```
1. git pull --rebase
2. Read .gnap/org.json → am I active?
3. Read .gnap/budget.json → do I have budget?
4. Read .gnap/messages/ → anything for me? Mark read.
5. Read .gnap/tasks/ → any tasks assigned to me in "ready" state?
6. Pick highest-priority ready task
7. Set state → "in_progress", commit
8. Create run file, commit
9. Execute (do the actual work)
10. Update task state (done/review/blocked), commit
11. Update run file with results + cost, commit
12. Update budget.json with cost, commit
13. git push
```

### Conflict Resolution

If `git push` fails (409 / conflict):

1. `git pull --rebase`
2. Re-read changed `.gnap/` files
3. Re-apply your changes if still valid
4. Retry push (max 3 attempts)
5. If still failing: wait 30 seconds, start over

---

## Mechanism 2: State Machine

Tasks follow strict state transitions. See Entity 3.

### Atomic Checkout

To prevent two agents from claiming the same task:

1. `git pull`
2. Read task file → confirm `state == "ready"` and you're in `assigned_to`
3. Set `state → "in_progress"` + `updated_at → now`
4. `git commit` + `git push`
5. If push fails (another agent got it first) → abandon, pick next task

SHA-based optimistic locking = atomic checkout without a database.

### Reconciliation

On each heartbeat, agents check their running tasks:

- If task was moved to `cancelled` by a human → stop work
- If task was reassigned → stop work
- If blocked by another task → set `blocked: true`

---

## Mechanism 3: Budget Control

Before any run:

```
if budget.agents[me].spent_usd >= budget.agents[me].limit_usd:
    create alert message: "Budget exhausted"
    skip work
    return
```

After each run:

```
budget.agents[me].spent_usd += run.cost_usd
budget.agents[me].runs += 1
commit budget.json
```

---

## Commit Convention

```
{agent-id}: {verb} {target}
```

Examples:

```
ori: create task carl-lead-pipeline
carl: checkout carl-lead-pipeline
carl: complete carl-lead-pipeline → review
carl: run carl-lead-pipeline attempt 1 ($0.42)
mayak: approve carl-lead-pipeline → done
ori: directive to carl — focus on Sebastian
carl: report 8/20 leads found
system: budget reset 2026-04
```

---

## Kanban Projection

The existing `kanban-data.json` is a **read-only view** generated from `.gnap/tasks/`:

| Task state | Kanban column |
|------------|---------------|
| `backlog` | Not Now |
| `ready` | Up Next |
| `in_progress` | In Progress |
| `review` | Human Review |
| `done` | Done |
| `blocked` | (any column, marked blocked) |

A build script or the kanban JS itself reads `.gnap/tasks/*.json` and generates the flat `kanban-data.json`.

---

## Inviting an Agent

To add a new agent to the system:

1. **Register:** Add agent to `.gnap/org.json`
2. **Auth:** Give agent a GitHub PAT with `contents:write` on this repo
3. **Skill:** Install the `gnap` skill on the agent (provides CLI + protocol knowledge)
4. **Budget:** Add entry in `.gnap/budget.json`
5. **Commit:** `system: invite {agent-id} as {role}`

The agent picks up work on its next heartbeat.

---

## Comparison

| | Paperclip | Symphony | AgentHub | **GNAP** |
|---|---|---|---|---|
| Infrastructure | Node.js + Postgres | Elixir daemon | Go + SQLite | **Git repo** |
| Setup time | Hours | Hours | Minutes | **Seconds** |
| Task source | Built-in tickets | Linear API | None (DAG) | **JSON files** |
| Org chart | ✅ DB | ❌ | ❌ | ✅ `org.json` |
| Goals | ✅ DB | ❌ | ❌ | ✅ `company.json` |
| Budgets | ✅ DB | Token tracking | Rate limits | ✅ `budget.json` |
| State machine | ✅ Atomic (DB) | ✅ Orchestrator | ❌ | ✅ SHA locking |
| Audit trail | DB + logs | Logs | Git history | **Git history** |
| Agent compat | OpenClaw, Codex | Codex only | Any (API key) | **Any (git)** |
| UI | React dashboard | Terminal | Web UI | **GitHub + kanban.html** |
| Multi-company | ✅ (tenants) | ❌ | ❌ | **Multiple repos** |
| Runs/traces | ✅ | ✅ | ❌ (commits) | ✅ `runs/` |
| Inter-agent msgs | @-mentions | Via Linear | Message board | ✅ `messages/` |
| Human oversight | Governance UI | Manual | Manual | **Kanban + git** |

---

## Migration from v1

1. Create `.gnap/` directory
2. Move `kanban-data.json` cards → individual `.gnap/tasks/{id}.json` files
3. Create `company.json`, `org.json`, `budget.json` from existing data
4. Add build step: `.gnap/tasks/*.json` → `kanban-data.json`
5. Update `gnap` skill to v2

Backward compatible: `kanban-data.json` continues to work as flat fallback.

---

*GNAP v2.0 — Farol Labs, March 2026*
*Takes the best from Paperclip (governance), Symphony (execution), and AgentHub (simplicity).*
*Zero servers. Git is all you need.*
