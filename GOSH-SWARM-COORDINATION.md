# GOSH Swarm Coordination Protocol v0.1

Fact-based coordination protocol for stateless multi-agent swarms.
Agents communicate exclusively through gosh.memory — no direct connections.


Copyright 2026 (c) Mitja Goroshevsky and GOSH Technology Ltd.
License: MIT

---

## 1. Principles

- **Stigmergy.** Agents read and write shared memory. No direct agent-to-agent communication.
- **Stateless execution.** Each agent activation: recall → work → store → exit. No persistent process state.
- **Fact-based routing.** Coordination facts have a `target_agent` field for point-to-point delivery and a `kind` field for semantics.
- **Courier delivery.** gosh.memory courier pushes facts to subscribed agents via SSE. Agents subscribe with filters matching their identity.
- **Best-effort coordination in v0.1.** This protocol assumes append-only memory facts and deterministic agent roles. It does not require memory-side CAS or leases for the baseline flow.

---

## 2. Identity

### 2.1 Agent identity fields

Every agent in a swarm has:

| Field | Description |
|-------|-------------|
| `agent_id` | Unique identifier within the swarm (e.g. `coder-a`, `reviewer`) |
| `agent_type` | Role class: `coder`, `reviewer`, `tester` |
| `swarm_id` | Swarm this agent belongs to (e.g. `swarm_alpha`) |

### 2.2 Identity assignment — two modes

#### Mode A: Explicit assignment (top-down)

This is the normative mode for v0.1 and the recommended mode for deterministic swarms.

Operator assigns identity at launch time:

```bash
gosh-agent --join <token> \
  --watch-agent-id coder-a \
  --watch-swarm-id swarm_alpha
```

Use when: operator knows which machine should run which role (e.g. GPU machine for heavy inference, specific network for repo access).

The agent registers itself in memory on first activation:

```json
{
  "kind": "agent_roster",
  "scope": "project-shared",
  "id": "roster_coder-a",
  "body": {
    "agent_id": "coder-a",
    "agent_type": "coder",
    "swarm_id": "swarm_alpha",
    "status": "active",
    "joined_at": "2026-03-24T12:00:00Z"
  }
}
```

#### Mode B: Dynamic slot claim (self-registration)

This mode is best-effort and experimental in v0.1. Use it only when the operator
is not launching multiple competing unassigned agents of the same type against
the same plan at the same time.

Agent joins with only a type, claims an open slot from the plan:

```bash
gosh-agent --join <token> \
  --watch-agent-type coder \
  --watch-swarm-id swarm_alpha
```

On activation:
1. `memory_query(filter={"kind": "plan"})` — read plan roles
2. Find first unclaimed slot matching `agent_type`
3. `store(kind=slot_claim)` — best-effort claim the slot
4. Use the slot's `agent_id` for all subsequent operations

```json
{
  "kind": "slot_claim",
  "scope": "project-shared",
  "id": "claim_coder-a_by_agent_xyz",
  "body": {
    "plan_id": "plan_feature_x",
    "slot": "coder-a",
    "claimed_by": "agent_xyz",
    "agent_type": "coder",
    "claimed_at": "2026-03-24T12:01:00Z"
  }
}
```

In current memory this is an operational convention, not a strict exclusivity guarantee.
If two agents race on the same slot, human or orchestrator resolves the collision.

Both modes result in the same outcome: an agent with a known `agent_id`
subscribed to courier with `filter={"target_agent": "<agent_id>"}`.

---

## 3. Fact kinds

All coordination happens through facts stored in gosh.memory. Each fact has a
top-level `kind` and structured `body`. Actionable point-to-point facts also
have `target_agent`. Broadcast facts such as `plan` and `agent_roster` omit it.

### 3.1 `plan` — project definition (created by human)

```json
{
  "kind": "plan",
  "scope": "project-shared",
  "id": "plan_feature_x",
  "body": {
    "title": "Feature X implementation",
    "spec": "specs/SPEC-feature-x.md",
    "repo": "github.com/Org/repo",
    "base_branch": "main",
    "roles": {
      "coder-a": { "type": "coder", "claimed_by": null },
      "coder-b": { "type": "coder", "claimed_by": null },
      "reviewer": { "type": "reviewer", "claimed_by": null }
    },
    "phases": [
      {
        "id": "phase_1",
        "prompt": "prompts/phase-1-data-layer.md",
        "assigned_to": "coder-a",
        "depends_on": []
      },
      {
        "id": "phase_2",
        "prompt": "prompts/phase-2-api-layer.md",
        "assigned_to": "coder-b",
        "depends_on": []
      },
      {
        "id": "phase_3",
        "prompt": "prompts/phase-3-integration.md",
        "assigned_to": "coder-a",
        "depends_on": ["phase_1", "phase_2"]
      }
    ]
  }
}
```

### 3.2 `task` — work assignment (created by human or reviewer)

```json
{
  "kind": "task",
  "scope": "project-shared",
  "target_agent": "coder-a",
  "id": "task_phase_1",
  "body": {
    "plan_id": "plan_feature_x",
    "phase": "phase_1",
    "attempt": 1,
    "role": "frozen-contract-coder",
    "prompt_file": "prompts/phase-1-data-layer.md",
    "repo": "github.com/Org/repo",
    "base_branch": "main",
    "branch": "feat/phase-1-data-layer",
    "created_by": "human",
    "status": "pending"
  }
}
```

### 3.3 `task_result` — work completed (created by coder)

```json
{
  "kind": "task_result",
  "scope": "project-shared",
  "target_agent": "reviewer",
  "id": "result_task_phase_1_a1",
  "body": {
    "task_id": "task_phase_1",
    "plan_id": "plan_feature_x",
    "phase": "phase_1",
    "attempt": 1,
    "agent_id": "coder-a",
    "status": "done",
    "branch": "feat/phase-1-data-layer",
    "pr": 42,
    "pr_url": "https://github.com/Org/repo/pull/42",
    "commit": "abc1234",
    "files_changed": ["src/storage.py", "tests/test_storage.py"],
    "summary": "Implemented JSONNPZStorage with atomic writes"
  }
}
```

### 3.4 `review_request` — review needed (created by coder)

```json
{
  "kind": "review_request",
  "scope": "project-shared",
  "target_agent": "reviewer",
  "id": "review_pr_42_a1",
  "body": {
    "task_id": "task_phase_1",
    "task_result_id": "result_task_phase_1_a1",
    "plan_id": "plan_feature_x",
    "phase": "phase_1",
    "attempt": 1,
    "requester_agent": "coder-a",
    "pr": 42,
    "pr_url": "https://github.com/Org/repo/pull/42",
    "commit": "abc1234",
    "repo": "github.com/Org/repo",
    "status": "pending"
  }
}
```

`review_request` is the only actionable input for reviewer. `task_result` is an
audit summary and context artifact that reviewer may fetch by
`task_result_id` and query by `task_id` if needed.

### 3.5 `review_result` — review done (created by reviewer)

On approve:
```json
{
  "kind": "review_result",
  "scope": "project-shared",
  "target_agent": "coder-a",
  "id": "review_result_pr_42_a1_approve",
  "body": {
    "review_request_id": "review_pr_42_a1",
    "task_id": "task_phase_1",
    "plan_id": "plan_feature_x",
    "phase": "phase_1",
    "attempt": 1,
    "pr": 42,
    "commit": "abc1234",
    "reviewer_agent": "reviewer",
    "verdict": "approve",
    "comments": ["Clean implementation, tests cover edge cases"],
    "gh_comment_url": "https://github.com/.../pull/42#issuecomment-123"
  }
}
```

On request changes:
```json
{
  "kind": "review_result",
  "scope": "project-shared",
  "target_agent": "coder-a",
  "id": "review_result_pr_42_a1_changes_1",
  "body": {
    "review_request_id": "review_pr_42_a1",
    "task_id": "task_phase_1",
    "plan_id": "plan_feature_x",
    "phase": "phase_1",
    "attempt": 1,
    "pr": 42,
    "commit": "abc1234",
    "reviewer_agent": "reviewer",
    "verdict": "request_changes",
    "comments": ["Missing error handling in save_facts()"],
    "fix_task_id": "task_phase_1_fix_1"
  }
}
```

Reviewer also creates a fix task:
```json
{
  "kind": "task",
  "scope": "project-shared",
  "target_agent": "coder-a",
  "id": "task_phase_1_fix_1",
  "body": {
    "parent_task": "task_phase_1",
    "plan_id": "plan_feature_x",
    "phase": "phase_1",
    "attempt": 2,
    "supersedes_task_id": "task_phase_1",
    "action": "address_review_comments",
    "pr": 42,
    "comments": ["Missing error handling in save_facts()"],
    "created_by": "reviewer",
    "status": "pending"
  }
}
```

### 3.6 `agent_roster` — agent registration

```json
{
  "kind": "agent_roster",
  "scope": "project-shared",
  "id": "roster_coder-a",
  "body": {
    "agent_id": "coder-a",
    "agent_type": "coder",
    "swarm_id": "swarm_alpha",
    "status": "active",
    "joined_at": "2026-03-24T12:00:00Z",
    "capabilities": ["claude-code", "git", "github"]
  }
}
```

### 3.7 Interpretation rules

- Facts are append-only. Agents do not update or mutate earlier facts.
- `status` fields in examples are snapshots written at creation time, not
  mutable rows in a database.
- One task attempt is identified by `(plan_id, phase, attempt)`.
- Replacement or retry is expressed by creating a new `task` with a higher
  `attempt` and `supersedes_task_id` pointing to the earlier task.
- Fact ids for recurring events should include `attempt` when the same PR or
  phase can produce multiple events over time.
- `review_result.id` must be unique per verdict event. Never reuse the same id
  for approve and request-changes.

---

## 4. Routing

### 4.1 Courier subscription

Each agent subscribes with a filter matching its identity:

```
courier_subscribe(filter={"kind": "task", "target_agent": "coder-a"})
```

Courier delivers a fact only when all filter fields match. In practice agents
should usually filter by both `kind` and `target_agent`. An agent never sees
facts targeted at other agents.

### 4.2 Broadcast

For facts that all agents should see (e.g. plan updates), omit `target_agent`
and use `scope: project-shared`. Agents poll these via exact reads such as
`memory_query()` or `memory_get()`, not courier push and not semantic
`recall()`.

### 4.3 Multi-target

If a fact should reach multiple specific agents, create one fact per target:

```json
{"kind": "task", "target_agent": "coder-a", "id": "task_phase_1", ...}
{"kind": "task", "target_agent": "coder-b", "id": "task_phase_2", ...}
```

No multicast. Explicit is better than implicit.

---

## 5. Lifecycle

### 5.1 Coder agent

```
subscribe(filter={"kind": "task", "target_agent": "{{my_id}}"})
  │
  ▼ courier delivers task
memory_get(task by id) → read prompt_file → read spec
  │
  ▼
git worktree add → work → commit → push → gh pr create
  │
  ▼
store(task_result, target_agent=reviewer)      # audit + summary
store(review_request, target_agent=reviewer)   # actionable reviewer input
  │
  ▼
exit. wait for next task.
```

### 5.2 Reviewer agent

```
subscribe(filter={"kind": "review_request", "target_agent": "reviewer"})
  │
  ▼ courier delivers review_request
memory_get(review_request by id) → memory_get(task_result by task_result_id) → gh pr diff → read spec → review
  │
  ▼
gh pr review --approve  OR  --request-changes
  │
  ├── approve:
  │     store(review_result, target_agent=coder)
  │     optionally: external scheduler/orchestrator may check
  │     plan.phases.depends_on → find unblocked phases → store(task)
  │
  └── request_changes:
        store(review_result, target_agent=coder)
        store(task with action=address_review_comments, target_agent=coder)
  │
  ▼
exit. wait for next review_request.
```

### 5.3 Dependency resolution

Automatic dependency resolution is optional and deferred in baseline `v0.1`.
The baseline protocol works without it: human or external orchestrator may
create the next `task` after observing `review_result`.

If automatic dependency resolution is implemented, reviewer or external
scheduler checks before creating next-phase tasks:

```
plan = memory_get(plan by id)
for phase in plan.phases:
  if phase.depends_on is empty or all deps have latest review_result verdict=approve:
    if no task exists for (plan_id, phase.id):
      create task → courier delivers to assigned agent
```

Reviewer must look at the latest `review_result` for the dependency phase and
attempt. An older approve does not unblock a newer retry attempt.

---

## 6. Failure handling

| Failure | Recovery |
|---------|----------|
| Agent crashes mid-task | No task_result stored. If the same agent restarts, it may continue or pick up the same pending task. If the agent is gone, human or orchestrator creates a replacement task with higher `attempt` and `supersedes_task_id`. |
| Courier SSE drops | Agent reconnects (5s backoff). Poll fallback catches missed facts (30s). |
| Review timeout | Human intervention. Reviewer has no timer — human checks `gosh memory list --kind review_request`. |
| Conflicting slot claims | Avoid by using Mode A. In Mode B, human or orchestrator resolves collision and relaunches/reassigns the losing agent. |
| Agent never joins | Unclaimed slot in plan. Human sees it in roster. Launches another agent or reassigns. |

---

## 7. Constraints

- Agents NEVER communicate directly. All coordination through memory.
- Agents NEVER modify facts created by other agents. Append only.
- Coders NEVER start next phase without reviewer approve.
- Reviewer NEVER writes code. Only reviews and creates tasks.
- One PR per phase. Fix tasks amend the same PR, not create new ones.
- Human merges PRs. Agents only approve/request-changes.
