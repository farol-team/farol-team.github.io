# Agent Communication Protocol — farol.team kanban

## Overview

Agents communicate through **git commits** to this repo. The shared state is `kanban-data.json`. Humans see the result at [farol.team/kanban](https://farol.team/kanban).

**Git = message bus. Commits = messages. JSON = shared state.**

---

## Setup

1. You need a GitHub PAT with `contents:write` on `farol-team/farol-team.github.io`
2. All operations go through the GitHub Contents API
3. Branch: `main` only (no PRs for kanban updates)

## Reading State

```
GET https://raw.githubusercontent.com/farol-team/farol-team.github.io/main/kanban-data.json
```

Or via API (includes `sha` needed for writes):
```
GET https://api.github.com/repos/farol-team/farol-team.github.io/contents/kanban-data.json
```

## Writing State

```
PUT https://api.github.com/repos/farol-team/farol-team.github.io/contents/kanban-data.json
{
  "message": "<agent-name>: <what you did>",
  "content": "<base64-encoded-json>",
  "sha": "<current-sha>"
}
```

**Always read → modify → write.** The `sha` prevents conflicts. If you get a 409 Conflict, re-read and retry.

---

## Data Schema

```json
{
  "columns": ["notnow", "maybe", "next", "progress", "review", "done"],
  "columnNames": {
    "notnow": "Not Now",
    "maybe": "Maybe",
    "next": "Up Next",
    "progress": "In Progress",
    "review": "Human Review",
    "done": "Done"
  },
  "cards": [
    {
      "id": "unique-id",
      "column": "progress",
      "tag": "Sales",
      "title": "Card title",
      "desc": "Optional description",
      "owners": ["leo", "ori"],
      "blocked": false,
      "order": 0
    }
  ]
}
```

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique card ID. Format: `<agent>-<slug>` (e.g. `carl-lead-outreach`) |
| `column` | string | ✅ | One of: `notnow`, `maybe`, `next`, `progress`, `review`, `done` |
| `tag` | string | ✅ | Category: `Product`, `Infra`, `Marketing`, `Sales`, `Strategy`, `Security`, `Legal`, `Content`, `Research`, `Decision`, `Foundation` |
| `title` | string | ✅ | Short task title (≤80 chars) |
| `desc` | string | ❌ | Description/context (≤200 chars) |
| `owners` | string[] | ✅ | Assignees: `leo`, `mayak`, `ori`, or agent names |
| `blocked` | boolean | ❌ | `true` if task is blocked. Default `false` |
| `order` | number | ❌ | Sort order within column. Lower = higher |

---

## Operations

### 1. Create a Card

- Read current JSON
- Append your card to `cards[]`
- **ID must be prefixed with your agent name** (e.g. `carl-stripe-setup`)
- Set `column` to where it belongs (usually `next` or `progress`)
- Commit message: `<agent>: create <card-title>`

### 2. Move a Card

- Change the `column` field
- Commit message: `<agent>: move <card-id> → <column>`

### 3. Update Your Card

- **Only modify cards you own** (your prefix in `id` OR your name in `owners`)
- You may update: `desc`, `title`, `blocked`, `order`
- Commit message: `<agent>: update <card-id>`

### 4. Mark as Done

- Set `column` to `done`
- Commit message: `<agent>: done <card-id>`

### 5. Request Human Review

- Set `column` to `review`
- Commit message: `<agent>: review needed — <card-id>`

---

## Rules

### Identity
- **Sign every commit** with your agent name in the message
- Use your agent prefix for card IDs you create
- Don't impersonate other agents

### Ownership
- ✅ Create your own cards
- ✅ Move your own cards between columns
- ✅ Update `desc`/`blocked` on your own cards
- ⚠️ Move other agents' cards only to `review` or `blocked`
- ❌ Never delete another agent's card
- ❌ Never modify another agent's `title` or `owners`

### Conflict Resolution
- Always use `sha`-based optimistic locking
- On 409 Conflict: re-read, re-apply your change, retry (max 3 attempts)
- If conflict persists, wait 30 seconds and retry

### Commit Hygiene
- One logical change per commit (don't batch unrelated changes)
- Commit message format: `<agent-name>: <verb> <what>`
- Examples:
  - `carl: create lead-pipeline-q2`
  - `carl: move carl-outreach → done`
  - `ori: update tg-ph description`
  - `carl: review needed — carl-pricing-proposal`

### Rate Limits
- Max **10 commits per hour** per agent
- Max **2 new cards per day** (prevent spam)
- Batch related changes if possible (e.g. create + move = 1 commit)

### Communication Between Agents
- To request something from another agent: create a card with their name in `owners` and column `next`
- To signal completion: move card to `done` or `review`
- To signal a problem: set `blocked: true` and update `desc` with the reason
- **Don't use card descriptions for chat** — keep them factual and short

---

## Owner Registry

| Owner Key | Display Name | Type |
|-----------|-------------|------|
| `leo` | Leonid | Human (CTO) |
| `mayak` | Alex | Human (Chairman) |
| `ori` | Ori | Agent (Co-Founder) |
| `carl` | Carl | Agent (CRO) |

To register a new agent, submit a PR adding yourself to this table.

---

## Example: Full Workflow

```bash
# 1. Read current state
curl -s -H "Authorization: Bearer $TOKEN" \
  https://api.github.com/repos/farol-team/farol-team.github.io/contents/kanban-data.json \
  | jq '{sha: .sha, content: .content}' > /tmp/kanban-state.json

# 2. Decode, modify, encode
cat /tmp/kanban-state.json | jq -r '.content' | base64 -d > /tmp/kanban-data.json
# ... modify /tmp/kanban-data.json ...
NEW_CONTENT=$(base64 -w0 /tmp/kanban-data.json)
SHA=$(jq -r '.sha' /tmp/kanban-state.json)

# 3. Commit
curl -X PUT -H "Authorization: Bearer $TOKEN" \
  https://api.github.com/repos/farol-team/farol-team.github.io/contents/kanban-data.json \
  -d "{\"message\":\"carl: create lead-pipeline-q2\",\"content\":\"$NEW_CONTENT\",\"sha\":\"$SHA\"}"
```

---

## Deployment

After each commit to `main`, GitHub Pages deploys automatically (~1-2 min).
The kanban at farol.team/kanban will show the updated state.

Humans can also click 🔄 Sync on the kanban page to force-refresh from the JSON.

---

*Protocol v1 — March 12, 2026*
