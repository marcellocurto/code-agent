# Agent System — Actionable Build Plan

> A single-writer, state-machine-driven coding agent system with hard enforcement contracts, explicit escalation paths, and deterministic quality gates.

---

## 0. Core Philosophy

Three rules that override everything else:

1. **Enforce, don't instruct.** If a rule can be broken by forgetting a prompt, it isn't a rule — it's a wish. Every constraint lives in code, CI, or a wrapper script.
2. **State is the hard problem.** The model writes decent code. The system fails when "what happens next?" becomes ambiguous. Every design decision here solves for state, not intelligence.
3. **One active coding task per repo. Always.** No exceptions. Enforced by a lease, not by convention.

---

## 1. Task Lifecycle — The State Machine

This is non-negotiable. The bot cannot move tasks backwards or skip states.

```
Idea
  → Needs Triage        (human or triager agent)
  → Needs Plan          (planner agent — generates task DAG)
  → Planned             (human approves decomposition)
  → Ready for Agent     ← ONLY state the bot can auto-start from
  → Active              (lease acquired, branch created)
  → Needs Human         (bot blocked, structured escalation posted)
  → In Human Review     (human required by task definition)
  → Agent Done          (all gates passed, PR open)
  → Merged              (human merged to develop)
  → Released            (human merged to main)
```

**Hard rules:**
- `Active` requires an acquired task lease (see §5)
- Only one `Active` coding task per repo at any time
- `Agent Done` ≠ `Merged`. The bot never merges.
- `Merged` ≠ `Released`. Only humans commit to main.
- Any task with ambiguity goes to `Needs Human`, never `Active`
- Planner and research agents may run in parallel — they do not mutate code

---

## 2. Linear Structure

### Workflow Statuses
Exactly the state machine above. No custom shortcuts.

### Label Groups (mutually exclusive within each group)

| Group | Values |
|---|---|
| **Work Type** | Feature / Bug / Refactor / Research / Experiment / Chore |
| **Owner Type** | Human / Planner / Coder / Designer |
| **Intervention** | None / Product / Design / Infra / Security / Data |
| **Risk** | Low / Medium / High / Migration |
| **Execution Mode** | Agent Only / Human Review Required / Human Required |

**Priority = urgency only.** Use labels and statuses for execution state, not priority inflation. If everything is `urgent`, nothing is.

---

## 3. Task Object Types

High-level plans and executable tasks are different object types, not just different levels of detail.

### Feature Request
Human intent only. No implementation detail required.
- `title` — one sentence
- `why` — business/user reason
- `success looks like` — observable outcome

### Plan
Planner agent output. Contains a task DAG. Must be human-approved before any child task reaches `Ready for Agent`.
- `feature_request_id`
- `tasks[]` — ordered list of Task objects with dependency links
- `sequencing_notes`
- `open_questions` — if non-empty, task stays in `Needs Plan`

### Task
The executable unit. Cannot start without a complete Definition of Ready (§4).
- `objective` — one sentence
- `dependencies[]` — other task IDs that must be `Merged` first
- `non_goals` — explicit exclusions
- `acceptance_criteria[]` — exact, testable statements
- `affected_surfaces[]` — files, packages, APIs likely touched
- `blast_radius` — Low / Medium / High
- `human_review_required` — boolean
- `rollback_expectation` — what to do if it breaks
- `work_type` — from label group
- `execution_mode` — from label group

### Research Task
No code required. Ends with a recommendation doc and follow-up task list.
- `question` — what needs answering
- `deliverable` — markdown doc in `research/YYYY-MM-DD-slug.md`
- `done_when` — recommendation written + follow-up tasks created in Linear

### Experiment Task
Executable but never auto-merged.
- All Task fields, plus:
- `hypothesis` — falsifiable statement
- `success_metric` — exact measurement
- `max_scope` — file/package boundaries
- `allowed_paths[]` — agent may only touch these
- `branch` — always a fresh branch, never `develop`
- `no_auto_merge: true` — always

---

## 4. Definition of Ready / Definition of Done

### Definition of Ready
A task cannot leave `Planned` until all fields are present:
- [ ] Single-sentence objective
- [ ] Non-goals listed
- [ ] Acceptance criteria — each item testable
- [ ] Affected surfaces identified
- [ ] `human_review_required` set
- [ ] Work type and execution mode labeled
- [ ] Rollback/containment expectation noted

### Definition of Done
A coding task is `Agent Done` only when all are true:
- [ ] Every acceptance criterion is satisfied
- [ ] `bun run bot:lint` — clean
- [ ] `bun run bot:typecheck` — clean
- [ ] `bun run bot:test` — all pass
- [ ] Playwright tests pass if any UI surface was touched
- [ ] New tests prove something meaningful (reflection gate checks this)
- [ ] Branch pushed, PR open against `develop`
- [ ] CI green (or local gates match CI gates exactly)
- [ ] Structured completion note posted to Linear (see §7)
- [ ] If `human_review_required: true` → status moves to `In Human Review`, not `Agent Done`

### Definition of Done — Research Task
- [ ] Markdown doc committed to `research/`
- [ ] Doc contains: findings, recommendation, confidence level, follow-up tasks
- [ ] Follow-up tasks created in Linear and linked
- [ ] Linear comment with one-paragraph summary posted

---

## 5. Task Lease — Enforcing "One at a Time"

"One task at a time" is a lease contract, not a slogan.

```typescript
type TaskLease = {
  taskId: string
  agentId: string
  repo: string
  branch: string
  startedAt: Date
  heartbeatAt: Date
  ttl: number         // seconds — heartbeat must refresh before expiry
  attemptCount: number
}
```

**Rules:**
- Lease acquisition is atomic — implemented as a Convex mutation with optimistic concurrency
- Only one active coding lease per repo at any time
- Agent must send heartbeat every N seconds while working
- If heartbeat expires → task moves to `Needs Human` automatically
- If task fails: one auto-retry allowed, then `Needs Human`
- Planner and research agents are exempt (read-only, no code mutations)
- All external writes are idempotent — safe to retry

---

## 6. The Orchestration API — Six Verbs Only

The bot interacts with the world through exactly this API. No raw Linear writes. No direct git commands outside wrappers. No improvised tool use.

```typescript
// The only interface the coding bot is allowed to use
claimTask(taskId: string): TaskLease | null
readTaskContext(taskId: string): TaskContext
runChecks(): CheckResult         // lint + typecheck + test
postCheckpoint(note: string): void
requestHuman(escalation: EscalationPayload): void
completeTask(evidence: CompletionEvidence): void
```

This buys: idempotency, auditability, clean retry paths, and less prompt fragility. The wrappers do the real Linear/Git/CI work. The bot only calls verbs.

---

## 7. Structured Output Formats

The bot always posts machine-readable output. Never freeform prose.

### Completion Note (posted on `completeTask`)
```
Completed:
- [what changed, one line each]

Evidence:
- bun test [specific test file(s)]
- bun run typecheck
- bun run lint
- CI: [link]

Risks / follow-ups:
- [any known gaps or next steps]
- Human review required: [yes/no — reason if yes]
```

### Escalation Note (posted on `requestHuman`)
```
Blocked because:
- [exact reason — one line]

Need from human:
- [specific decision or information required]

Current state:
- Branch: [branch name]
- Tests added: [N]
- Passing: [N] / Failing: [N]
- Last passing checkpoint: [description]

Recommendation:
- [bot's best guess at correct path forward]
```

---

## 8. Quality Gates — Deterministic First, Heuristic Second

Gates run in order. Heuristic gates cannot override failing deterministic gates.

### Deterministic Gates (must all pass — no exceptions)
- [ ] Clean working tree (no uncommitted changes outside task scope)
- [ ] Branch naming policy: `[type]/[linear-id]-[slug]`
- [ ] Only paths in `affected_surfaces` were modified (or escalate)
- [ ] `bun run bot:lint` clean
- [ ] `bun run bot:typecheck` clean
- [ ] `bun run bot:test` all pass
- [ ] Playwright tests pass (if UI touched)
- [ ] Required Linear fields present on task
- [ ] Migration safety check (if schema changed)

### Heuristic Gates (advisor — can block, cannot approve alone)
The reflection gate runs after all deterministic gates pass.

```typescript
const reflectionGate = Effect.gen(function* () {
  const reflection = yield* askClaude(`
    Review what just happened:
    - Did the tests actually catch something meaningful, or are they written to pass trivially?
    - Is the implementation the simplest thing that works?
    - Are there edge cases unhandled?
    - Is there anything that should be cut?
    - Are there surfaces being tested that don't need to be?
    Be honest. Cut or fix anything that doesn't earn its place.
  `)
  if (reflection.shouldBlock) {
    yield* requestHuman({ reason: reflection.reason, type: 'reflection_block' })
    yield* Effect.fail(new ReflectionBlocked(reflection))
  }
})
```

**Reflection can block. Reflection approving means nothing if deterministic gates failed.**

For high-risk tasks (`Risk: High` or `Migration`): a second model reviews the diff and evidence only — not the full repo state. Different model from the coder to reduce shared blind spots.

---

## 9. Repo Health Contract

Three cases — handled differently.

**Case A: Agent caused the break**
Do not create a new task. The current task is not done. Fix it inside the current task scope.

**Case B: Repo was already red before task start**
Do not auto-create an urgent task. Create a linked `blocker` task and either:
- Pause the current feature task (preferred), or
- Continue only if the failure is outside all touched surfaces and explicitly allowed in the task context

**Case C: Baseline is globally unhealthy**
Enter `repo health mode`:
- No new feature/bug/refactor tasks start
- Bot works health tasks only until baseline returns green
- Planner creates health tasks automatically when CI has been red for > N minutes

This stops priority inflation. No fake urgent noise.

---

## 10. Human Intervention — Mandatory Escalation Triggers

The bot must call `requestHuman` immediately (not attempt to resolve) for:

- Unclear or contradictory product behavior
- Schema or data migrations
- Auth, permissions, or session changes
- Billing or payments
- Destructive operations (deletes, truncates, irreversible writes)
- Dependency or platform version changes
- Changes spanning more than **3 packages** or **20 files**
- Contradictory tests vs spec
- Any decision where the correct next step is a tradeoff, not an implementation
- `Risk: High` or `Risk: Migration` tasks by default

---

## 11. Branch and Commit Policy

Enforced by wrapper scripts and pre-commit hooks — not by prompts.

```json
{
  "allowedCommands": [
    "bun run bot:start-task",
    "bun run bot:test",
    "bun run bot:typecheck",
    "bun run bot:lint",
    "bun run bot:checkpoint",
    "bun run bot:complete-task"
  ],
  "forbiddenBranches": ["main"],
  "baseBranch": "develop",
  "branchPattern": "[type]/[linear-id]-[slug]",
  "experimentBranchPattern": "experiment/[linear-id]-[slug]",
  "requireTaskLease": true,
  "requireCIGreen": true
}
```

- `main` — humans only, always
- `develop` — agent PRs land here after CI green
- `experiment/*` — never merged automatically, always human reviewed
- Pre-commit hooks enforce lint, typecheck, and branch policy on every commit

---

## 12. Toolchain — One Authority Per Layer

Resolved conflicts from original draft:

| Layer | Choice | Notes |
|---|---|---|
| **Runtime** | Bun | No Node fallback |
| **Formatting** | Biome | Replaces Prettier entirely |
| **Linting** | Biome | Primary. Add `eslint-plugin-effect`, `eslint-plugin-playwright`, `@typescript-eslint` only if Biome gaps require it — not by default |
| **Backend validation** | `@effect/schema` | Canonical |
| **Frontend validation** | Zod at form boundaries only | `@effect/schema` elsewhere if shared types cross the boundary |
| **Async/error handling** | Effect | All backend async logic |
| **DB / realtime** | Convex | Mutations and queries |
| **Testing** | `bun:test` + Playwright | No exceptions |
| **HTTP client** | Effect HttpClient | |

**React Compiler:** experiment flag only. Not in baseline doctrine. Adopt after agent workflow is stable.

One test command locally = same test command in CI. No drift.

---

## 13. Source of Truth Split

| System | Owns |
|---|---|
| **Linear** | Task state, priority, human intervention, structured history |
| **Git / CI** | Code health, mergeability, test results |
| **Convex dashboard** | Read-only realtime mirror — fast visibility, light human actions (submit feature requests → Linear) |

Convex is never the canonical place where task state is decided. Linear is. The dashboard reflects, it does not govern.

---

## 14. Agent Roles and Model Routing

Roles, not identity. Models are assigned per role with fallback.

| Role | Responsibility | Notes |
|---|---|---|
| **Triager** | Triage incoming ideas, label, route | Lightweight model fine |
| **Planner** | Decompose feature → task DAG | Different model from coder on high-risk tasks |
| **Coder** | Implement, test, complete | Single active instance per repo |
| **Reviewer** | Second-pass on high-risk diffs | Never same model as coder for that task |
| **Designer** | UI/visual work | Claude, separate from logic work |

If primary model fails twice on a task: fallback model picks up from last `postCheckpoint`. Reviewer sees diff + evidence only, not full repo state.

---

## 15. SKILL.md Structure

Thin in prose, thick in enforcement. The SKILL.md is principles. The `bot:*` scripts are the actual rules.

```markdown
# coding-bot/SKILL.md

## YOU ARE
A coding agent. You write code and tests. You use the six verbs. Nothing else.

## RULES (non-negotiable)
- Write the test first. Always.
- Use Effect for all async/error-prone logic.
- Use @effect/schema for all validation.
- Use Bun APIs, not Node.
- Run bot:test before marking done.
- Run Playwright tests if touching UI.
- Post structured completion note via completeTask verb.
- All escalations via requestHuman verb — never improvise.

## LIBRARIES (use these, do not bikeshed)
- HTTP: Effect HttpClient
- Validation: @effect/schema
- Testing: bun:test + Playwright
- DB/realtime: Convex
- Formatting/lint: Biome

## YOU DO NOT
- Choose libraries not listed above
- Skip tests
- Write to main or develop directly
- Modify other agents' tasks
- Call Linear, git, or CI APIs directly — use the six verbs
- Merge pull requests
- Make product decisions — use requestHuman
```

---

## 16. Starting Point — Matt Pocock Skills

Before building custom skills, use these from `skills.sh/mattpocock/skills` as the starting scaffold:

- `write-a-prd` — feature request intake
- `prd-to-plan` — planner agent output format
- `prd-to-issues` — task DAG generation
- `tdd` — test-first enforcement
- `triage-issue` — triager agent
- `improve-codebase-architecture` — refactor task type
- `setup-pre-commit` — hook scaffold
- `git-guardrails-claude-code` — branch policy enforcement

Extend from here. Don't invent custom equivalents on day one.

---

## 17. Build Order

Ship in this sequence. Each phase is a stable stopping point.

**Phase 1 — The Rails**
- [ ] Linear workspace with exact statuses and label groups from §2
- [ ] `bot:*` wrapper scripts with `allowedCommands` contract
- [ ] Task lease implementation (Convex mutation, atomic)
- [ ] Pre-commit hooks (lint, typecheck, branch policy)
- [ ] State machine enforcement (bot cannot transition to forbidden states)

**Phase 2 — The Loop**
- [ ] Coding bot wired to six-verb orchestration API
- [ ] Definition of Ready validation before task activation
- [ ] Deterministic gate stack running on `completeTask`
- [ ] Structured completion note and escalation note formats
- [ ] CI gate (must match local `bot:test` exactly)

**Phase 3 — Intelligence Layer**
- [ ] Planner agent (feature → task DAG, human approval gate)
- [ ] Reflection gate as heuristic advisor (after deterministic gates)
- [ ] Repo health mode (Case A/B/C logic)
- [ ] Second-model reviewer for high-risk tasks

**Phase 4 — Visibility**
- [ ] Convex dashboard (read mirror of Linear state)
- [ ] Feature request submission from dashboard → Linear
- [ ] Agent heartbeat display, current task, queue by priority

**Phase 5 — Hardening**
- [ ] Heartbeat expiry → auto-escalation
- [ ] Experiment task type with full isolation contract
- [ ] Research task type with `research/` doc deliverable
- [ ] Fallback model routing on double failure

---

## Appendix: What the Review Doc Got Right

The second review was correct on every structural point. Worth calling out explicitly:

- **State machine over fuzzy statuses** — most important change from the original draft
- **Six-verb orchestration API** — single best improvement for resilience
- **Deterministic gates before heuristic gates** — reflection is an advisor, not a judge
- **Task lease as a real lock** — "one at a time" must be enforced, not assumed
- **Repo health cases A/B/C** — stops priority inflation cold
- **Planner outputs a DAG, not a list** — sequencing and blast radius matter
- **Linear label groups for execution state** — priority is urgency only

The original draft had the right taste. This document has the right structure.
