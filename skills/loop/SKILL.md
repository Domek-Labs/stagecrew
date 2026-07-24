---
name: loop
description: >
  Loop-workflow umbrella. Routes to /init-agents (bootstrap), /create-issue (genesis), /plan-issues (wave planning), /work-issue (5-stage execution), /close-out (ordered merge), or /run-loop (full pipeline in one command). Triggers: "loop", "loop workflow", "work issue", "spec build", "coding loop", "github workflow", "github loop", "spec build loop". Routes when intent is unclear.
---

# /loop — Loop-Engineering Workflow (Umbrella)

**Type:** umbrella / router

## Purpose

Central entry point for the stagecrew pipeline: GitHub issue → merged PR (or research findings) via a multi-stage agent pipeline.

Six sub-skills cover the full lifecycle:

| Phase | Sub-skill | What | Direct trigger |
|-------|-----------|------|----------------|
| **Bootstrap** | `/init-agents` | Create `AGENTS.md` for a repo (one-time per repo, pure-reader prerequisite) | "init agents", "create agents.md", "coding spec bootstrap", "initialize repo standards" |
| **Genesis** | `/create-issue` | Idea → fully-specified GitHub issue (spec standard with Idea/AC/Files/Test-Plan/OoS) | "create issue", "new issue", "create feature", "spec issue", "idea as ticket" |
| **Planning** | `/plan-issues` | Assign parallel-safety waves to open issues before work starts; detect file collisions and dependency order | "plan issues", "wave plan", "assign waves", "plan before work", "parallel plan" |
| **Execution** | `/work-issue` | Issue → merged PR via 5 stages (Validator → Implementer → Tester → Critic → Closer) | "work issue", "drive issue", "loop workflow", "spec build loop" |
| **Merge** | `/close-out` | Merge completed PRs in wave order, auto-resolve additive conflicts, close issues, clean up branches | "close out", "merge waves", "ship waves", "finalize waves" |
| **Pipeline** | `/run-loop` | Single-command entry point: chains create → plan → [confirm] → work | "run loop", "full loop", "start loop", "run full pipeline" |

## Workflow at a glance

```
Bootstrap          Genesis          Planning        Execution        Merge
    │                 │                │                │               │
    ▼                 ▼                ▼                ▼               ▼
/init-agents  →  /create-issue  →  /plan-issues  →  /work-issue  →  /close-out
(AGENTS.md)     (GitHub issues)   (wave labels)   (5-stage loop)  (ordered merge)

Or: /run-loop "<idea>"  →  [confirm plan]  →  loops fire  →  /close-out
```

All sub-skills are **pure-readers** — they read `AGENTS.md` at the repo root as the single source of truth for standards.

## Typical sequence for new repos

```
1. /init-agents --repo <owner>/<slug>      # create AGENTS.md (once per repo)
2. /create-issue "<idea>"                  # spec the issue(s)
3. /plan-issues                            # assign waves (for multi-issue runs)
4. /work-issue <num>                       # drive each issue (Validator → ... → Closer)
5. /close-out                              # merge in wave order
```

Or in one command: `/run-loop "<idea>"` chains steps 2–4 with a confirmation gate.

## What happens when /loop is invoked directly

Clarify intent with a single question, then route:

```
What do you want to do?

  1) Create AGENTS.md for a new repo                    → /init-agents
  2) Capture a new idea as an issue                     → /create-issue
  3) Plan wave order for open issues                    → /plan-issues
  4) Drive an existing issue (5-stage loop)             → /work-issue
  5) Merge completed PRs in wave order                  → /close-out
  6) Full pipeline from idea to running loops           → /run-loop
```

## When to use which skill

| Situation | Skill |
|-----------|-------|
| New repo, no AGENTS.md yet | `/init-agents --repo <slug>` |
| New idea, no issue yet (AGENTS.md exists) | `/create-issue "..."` |
| New idea, AGENTS.md missing | `/create-issue "..."` (calls `/init-agents` as a pre-step) |
| Multiple issues to run in parallel | `/plan-issues` first, then `/work-issue` per wave |
| Issue exists with a complete spec | `/work-issue <num>` |
| Issue exists without a spec | `/create-issue --refine <num>` then `/work-issue` |
| `/work-issue` Validator returned STOP | `/create-issue --refine <num>`, then `/work-issue` retry |
| `/work-issue` returns STOP "AGENTS.md missing" | `/init-agents --repo <slug>`, then `/work-issue` retry |
| AGENTS.md exists but has gaps | `/init-agents --refine --repo <slug>` |
| PRs ready to merge in wave order | `/close-out --repo <slug>` |
| Want idea-to-loops in one command | `/run-loop "<idea>"` |

## Example calls

```
# New repo, full loop
/init-agents --repo myorg/myrepo
/create-issue --repo myorg/myrepo "add JWT refresh endpoint"
/plan-issues --repo myorg/myrepo
/work-issue 42 --repo myorg/myrepo
/close-out --repo myorg/myrepo

# Same result in one command
/run-loop --repo myorg/myrepo "add JWT refresh endpoint"

# Drive an existing issue
/work-issue 17

# Plan and launch parallel loops for open issues
/plan-issues --repo myorg/myrepo --milestone "v2.0"
# → shows wave plan, user confirms → /work-issue runs per wave

# Unclear intent
/loop
# → asks one question, then routes to one of the six sub-skills
```

## Why an umbrella

Six sub-skills, one umbrella. The full lifecycle of a feature:

- `/init-agents` = capture standards (AGENTS.md)
- `/create-issue` = capture intent (GitHub issue)
- `/plan-issues` = determine execution order (wave labels)
- `/work-issue` = execute intent (issue → PR)
- `/close-out` = ship intent (PRs → merged into main)
- `/run-loop` = shortcut for the common create → plan → work path

## Roadmap

See the repo issues (labelled `roadmap`) for the rolling roadmap. Version numbers for roadmap items are intentionally not tracked here — see `AGENTS.md` `version_policy` for the bump policy. The current version lives only in `.claude-plugin/plugin.json`.

## See also

- `skills/init-agents/SKILL.md`
- `skills/create-issue/SKILL.md`
- `skills/plan-issues/SKILL.md`
- `skills/work-issue/SKILL.md`
- `skills/close-out/SKILL.md`
- `skills/run-loop/SKILL.md`
- `../CLAUDE.md` (plugin quickstart)
