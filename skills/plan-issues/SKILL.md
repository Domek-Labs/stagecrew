---
name: plan-issues
description: "Wave-based execution planner. Assigns parallel-safety waves to open issues before the work-issue loops start. Prevents file collisions, migration-number clashes, and dependency-order violations in parallel runs. Outputs a wave plan for human review; no code is written. Pipeline position: create-issue → plan-issues → work-issue. Triggers: /plan-issues, plan issues, wave plan, plan before work, assign waves, parallel plan."
---

# /plan-issues — Wave assignment before parallel work loops

**Type:** planning / execution-order analysis

## Purpose

Before running multiple `/work-issue` loops in parallel, `plan-issues` partitions the open issues into **waves**: groups of issues that can safely run concurrently without file-level collisions, dependency violations, or migration-number clashes.

**Waves answer "what can run NOW?" — priority answers "what matters most?"** Both are needed; they solve different problems. Waves are the execution-safety unit; priority is the importance ranking within a wave.

### Pipeline position

```
create-issue → plan-issues → [human review] → work-issue (wave 1) → work-issue (wave 2) → ...
```

## Invocation variants

```
/plan-issues --repo owner/slug
/plan-issues --repo owner/slug --milestone "v2.0"
/plan-issues --repo owner/slug --label "sprint-current"
/plan-issues                          # → resolves repo from cwd, uses all open issues
```

**Filters (optional):** `--milestone`, `--label`, `--issue <num>[,<num>...]` narrow the issue set. Without filters, all open issues are planned.

## Workflow

### 1. Issue fetch

```bash
gh issue list --repo <slug> --state open --json number,title,labels,body,assignees --limit 100
```

Apply any milestone/label/number filters. The result is the **candidate set** — the issues to be planned.

### 2. Loop-type classification

For each issue, determine loop type from the `loop-type:<type>` label (set by `/create-issue`):
- `loop-type:code` → code loop; may touch source files → collision risk
- `loop-type:research` → research loop; writes only to `docs/research/` → collision-free with code loops and with each other

**Research loops** can be assigned to any wave alongside code loops — they never collide.

### 3. File-overlap analysis (code loops only)

For each code loop issue, extract the **declared files to touch** from the `## Files to Touch` section of the issue body.

Build a conflict graph: two issues conflict if their files-to-touch sets overlap. Each conflict edge forces the two issues into different waves.

```
wave assignment = graph coloring on the conflict graph
wave 1 = issues that conflict with no other issue, or the initial independent set
wave 2 = issues whose conflicts are all in wave 1
...
```

**Heuristic for missing files-to-touch:** if an issue has no `## Files to Touch` section, flag it with a warning — `plan-issues` cannot guarantee collision-safety for it. It is assigned its own wave (solo) so it can still run, but the human reviewer is asked to verify.

### 4. Dependency check

For each issue, parse `depends on #X` / `blocked by #X` references in the issue body or labels:

1. **Hard dependency:** issue A depends on issue B → A goes to a wave strictly later than B.
2. **Stale blocked label:** if the issue carries a `blocked` label but the referenced blocker issue is already CLOSED → remove the `blocked` label and note the cleanup in the plan output (do not silently ignore stale labels).

```bash
gh issue view <blocker-num> --repo <slug> --json state --jq '.state'
```

### 5. Migration-number reservation (optional, auto-detected)

If any issue's files-to-touch includes a pattern matching a migration file (e.g. `migrations/`, `db/migrate/`, `*.sql`, `*_migration*`), activate migration-number reservation:

- Scan the existing migration files in the repo to determine the next available number.
- Reserve a migration-number range per wave: wave 1 gets `N+1..N+k1`, wave 2 gets `N+k1+1..N+k2`, etc.
- Include the reserved ranges in the wave plan output so the Implementer for each wave knows which numbers to use.

This prevents two loops from independently choosing the same migration number.

### 6. Priority sort within waves

Within each wave, sort issues by priority signal:

1. Explicit `priority:high` / `priority:medium` / `priority:low` label — in that order.
2. Tie-break: lower issue number first (older issues first).

### 7. Wave-plan output

Print the wave plan as a human-readable draft:

```
Wave Plan — Domek-Labs/stagecrew (7 issues)
─────────────────────────────────────────────────────

Wave 1 — run in parallel (3 issues)
  #42  feat: add JWT refresh endpoint          [code]    priority: high
  #44  fix: race condition in scheduler        [code]    priority: medium
  #51  research: benchmark ollama tool-use     [research]

  Files reserved: src/auth/refresh.ts, src/scheduler.ts
  Migration range: (none)

Wave 2 — after wave 1 completes (2 issues)
  #45  feat: update API schema for JWT fields  [code]    priority: high
  #48  docs: update auth architecture          [code]    priority: low

  Files reserved: api/schema.yaml, docs/architecture.md
  Depends on: #42 (wave 1) ← dependency respected

Wave 3 — after wave 2 completes (2 issues)
  #50  feat: add refresh-token rotation UI     [code]    priority: medium
  #53  research: compare migration strategies  [research]

──────────────────────────────────────────────────────
Stale blocked labels removed: #46 (blocker #39 is CLOSED — label removed)
Warnings: #47 has no "Files to Touch" section — assigned solo wave (inspect before work)

Confirm this plan? [ok / adjust <description> / cancel]
```

### 8. Human review (mandatory gate)

Wait for explicit confirmation:

- **`ok`** → write wave labels (see below) and exit.
- **`adjust <description>`** → re-run the plan with the adjustment applied. Adjustments can be: "move #44 to wave 2", "merge wave 2 and 3", "remove #47 from plan".
- **`cancel`** → exit without writing any labels.

**This gate is mandatory.** `plan-issues` never starts work automatically.

### 9. Label writing (after confirmation)

For each planned issue, write a `wave:<n>` label:

```bash
gh label create "wave:1" --repo <slug> --color 0075CA --force
gh issue edit <num> --repo <slug> --add-label "wave:1"
```

Create `wave:<n>` labels as needed. These labels are the machine-readable output consumed by `/close-out` and `/run-loop`.

## Diff guard (hard gate for work-issue integration)

`plan-issues` documents a **diff guard** heuristic that `/work-issue` Critic stages should apply:

> If a loop's diff has **deletions > 3 × additions AND total changes < 100 lines**, the Critic should flag it as "suspicious diff pattern — likely operating on stale base branch" and request a REVISE with `git rebase <default_branch>` before re-review.

This catches the failure mode where a loop started on a stale branch and silently removed content that was added after its branch point. The Critic is the right stage to enforce this (it has the full diff).

Include this note in the plan output if any planned issues are code loops.

## What this skill does NOT do

- Does not create branches
- Does not write code or start work
- Does not merge PRs
- Does not auto-start `/work-issue` — that requires a separate invocation or `/run-loop`
- Does not resolve architectural conflicts (only file-level overlap)

## Output labels consumed by other skills

| Label | Set by | Consumed by |
|-------|--------|-------------|
| `wave:<n>` | `/plan-issues` | `/close-out` (merge order), `/run-loop` (launch order) |
| `loop-type:<type>` | `/create-issue` | `/plan-issues` (classification), `/work-issue` (brief dispatch) |

## See also

- `skills/create-issue/SKILL.md` — genesis phase (creates issues)
- `skills/work-issue/SKILL.md` — execution phase (drives individual issues)
- `skills/close-out/SKILL.md` — post-work merge in wave order
- `skills/run-loop/SKILL.md` — single-command pipeline (create → plan → work)
