---
name: run-loop
description: "Single-command pipeline entry point. Chains create-issue → plan-issues → work-issue with a mandatory human review gate between plan and work. Reduces a three-command sequence to one command plus one confirmation. Triggers: /run-loop, run loop, full loop, start loop, pipeline, run full pipeline."
---

# /run-loop — Single-command pipeline (create → plan → work)

**Type:** pipeline orchestrator / entry point

## Purpose

`/run-loop` is the single-command entry point for the full stagecrew pipeline. It accepts a task description and chains the three main phases internally:

```
/run-loop "<task>" 
  → /create-issue    (one or more fully-specified issues)
  → /plan-issues     (wave assignment — collision-safe grouping)
  → [human confirms] (mandatory review gate)
  → /work-issue      (parallel loops per wave, wave by wave)
```

This reduces the common case from three separate commands to **one command + one confirmation**.

`/close-out` is intentionally **not** part of `run-loop`. It is a separate decision point after all PRs are ready for review — a deliberate human checkpoint before merging.

### When to use

| Situation | Use |
|-----------|-----|
| You have a task idea and want to go from idea to running loops in one step | `/run-loop "<idea>"` |
| You want to drive an existing issue through the loop | `/work-issue <num>` |
| You want to plan existing issues before running them | `/plan-issues` |
| You want to merge completed PRs in wave order | `/close-out` |

## Invocation variants

```
/run-loop "add user authentication with JWT refresh"
/run-loop --repo owner/slug "add user authentication"
/run-loop --type=research "benchmark ollama tool-use latency"
/run-loop --issues 42,44,51    # plan and work existing issues (skip create-issue)
/run-loop                      # → asks for task description + repo
```

**`--issues <nums>`:** skip `create-issue` and go directly to `plan-issues` for the given issue numbers. Useful when issues already exist (e.g., from a previous `/create-issue` session).

## Workflow

### Phase 1 — create-issue

`/run-loop` invokes the `create-issue` skill in **batch mode**: it generates one or more issues from the task description, following the full `create-issue` spec dialog (Idea/Spec/AC/Files/Test-Plan/OoS).

For compound tasks (where the idea naturally decomposes into multiple issues), `run-loop` prompts: "This looks like multiple issues. Should I create them separately? (Yes / No — keep as one)"

After each issue is created, `run-loop` collects the issue numbers and continues.

**Skipped with `--issues`:** if issue numbers are provided directly, Phase 1 is skipped.

### Phase 2 — plan-issues

`/run-loop` invokes `plan-issues` on the created (or provided) issue numbers:

```
Planning waves for issues: #42, #44, #51
```

`plan-issues` runs its full wave-assignment workflow (file-overlap analysis, dependency check, priority sort) and produces a wave plan.

### Phase 3 — human review gate (mandatory)

`run-loop` displays the wave plan and waits for explicit confirmation:

```
Wave Plan
─────────────────────────────────────────────────
Wave 1 (parallel): #42 feat: JWT refresh, #44 fix: scheduler race
Wave 2 (after wave 1): #51 research: benchmark

Confirm? [ok / adjust <description> / cancel]
```

**`ok`** → write wave labels and proceed to Phase 4.
**`adjust <description>`** → re-run `plan-issues` with the adjustment and show updated plan.
**`cancel`** → stop. Issues and wave labels are NOT written.

**This gate is mandatory.** `/run-loop` never proceeds to work without explicit user confirmation of the plan.

### Phase 4 — work-issue (wave by wave)

After confirmation, `run-loop` launches `/work-issue` for **wave 1 issues in parallel**:

```
Launching wave 1 (2 loops)...
  /work-issue 42 --repo owner/slug
  /work-issue 44 --repo owner/slug
```

Both `/work-issue` runs are dispatched concurrently (each is a separate agent). `run-loop` monitors their progress via the state trackers and stage comments on the issues.

When wave 1 is complete (all issues merged or escalated), `run-loop` prompts before starting wave 2:

```
Wave 1 complete (2/2 merged).

Start wave 2? [ok / skip / cancel]
  #51 research: benchmark ollama tool-use
```

**`ok`** → launch wave 2 loops.
**`skip`** → mark wave 2 as skipped (no label cleanup), exit.
**`cancel`** → stop pipeline. Wave 2 issues remain open with their `wave:2` labels; `/work-issue` or `/run-loop --issues 51` can resume later.

This wave-by-wave confirmation prevents wave 2 from starting automatically if wave 1 produced unexpected results.

### Phase 5 — handoff to close-out

After all waves are done (or the user cancels), `run-loop` exits with a summary and a close-out prompt:

```
All waves complete.
  Wave 1: 2 merged ✓
  Wave 2: 1 merged ✓

Ready to merge? Run: /close-out --repo owner/slug
```

`/close-out` is not called automatically — it is presented as the explicit next step.

## Error handling

- **`create-issue` STOP (incomplete spec):** `run-loop` pauses and shows the Validator feedback. The user can refine and re-run `/run-loop --issues <fixed-num>`.
- **`work-issue` ESCALATE / hard-cap:** `run-loop` reports the escalation (Telegram if on a channel run), leaves the remaining waves untouched, and exits. The affected issues can be resumed individually with `/work-issue <num>`.
- **Wave plan adjustment:** if the user uses `adjust` after seeing the plan, the adjustment is applied and the updated plan is shown without restarting `create-issue`.

## Telegram / channel behavior

When invoked from a Telegram channel inbound:
- After `create-issue`: "Issues created: #42, #44, #51. Planning waves..."
- After `plan-issues`: post the wave plan as the confirmation prompt
- After each wave completes: brief status update before the next-wave prompt
- On final close-out handoff: post the summary with the `/close-out` command

## What this skill does NOT do

- Does not run `close-out` automatically
- Does not bypass the human review gate between plan and work
- Does not start wave 2 automatically when wave 1 finishes
- Does not modify existing issues (only creates new ones unless `--issues` is passed)

## See also

- `skills/create-issue/SKILL.md` — Phase 1 (issue genesis)
- `skills/plan-issues/SKILL.md` — Phase 2 (wave assignment)
- `skills/work-issue/SKILL.md` — Phase 4 (per-issue execution loop)
- `skills/close-out/SKILL.md` — post-work merge (explicit next step after run-loop)
