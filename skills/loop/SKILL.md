---
name: loop
description: >
  Loop-engineering workflow umbrella. Routes to /init-agents (bootstrap AGENTS.md for a repo), /create-issue (idea → fully-specified GitHub issue), or /work-issue (issue → merged PR via 5-stage Validator → Implementer → Tester → Critic → Closer pipeline). Triggers: "loop", "loop engineering", "work issue", "spec build", "coding loop", "github workflow", "github loop", "spec build loop". Routes when intent is unclear.
---

# /loop — Loop-Engineering Workflow (Umbrella)

**Type:** umbrella / router

## Purpose

Central entry point for the loop-engineering workflow: GitHub issue → merged PR (or research findings) via a 5-stage agent pipeline.

Three sub-skills cover the three phases:

| Phase | Sub-skill | What | Direct trigger |
|-------|-----------|------|----------------|
| **Bootstrap** | `/init-agents` | Create `AGENTS.md` for a repo (one-time per repo, pure-reader prerequisite) | "init agents", "create agents.md", "coding spec bootstrap", "initialize repo standards" |
| **Genesis** | `/create-issue` | Idea → fully-specified GitHub issue (spec standard with Idea/AC/Files/Test-Plan/OoS) | "create issue", "new issue", "create feature", "spec issue", "idea as ticket" |
| **Execution** | `/work-issue` | Issue → merged PR via 5 stages (Validator → Implementer → Tester → Critic → Closer) | "work issue", "drive issue", "loop engineering", "spec build loop" |

## Workflow at a glance

```
Bootstrap (once per repo)        Genesis (per feature/bug)       Execution (per issue)
       │                                 │                              │
       ▼                                 ▼                              ▼
  /init-agents          →          /create-issue          →         /work-issue
  (AGENTS.md)                      (GitHub issue)                   (5-stage loop → PR merge)
```

All three sub-skills are **pure-readers** — they read `AGENTS.md` at the repo root as the single source of truth for standards. No skill-internal defaults mapping.

## Typical sequence for new repos

```
1. /init-agents --repo <owner>/<slug>      # create AGENTS.md (once per repo)
2. /create-issue "<idea>"                  # spec the first issue
3. /work-issue <num>                       # work the issue (Validator → Implementer → Tester → Critic → Closer until merge)
```

For existing repos that already have AGENTS.md: skip step 1 — start directly with `/create-issue` or `/work-issue`.

## What happens when /loop is invoked directly

Clarify intent with a single question, then route:

```
What do you want to do?

  1) Create AGENTS.md for a new repo               → /init-agents
  2) Capture a new idea as an issue                → /create-issue
  3) Drive an existing issue (5-stage loop)        → /work-issue
  4) Full loop for a new repo                      → /init-agents → /create-issue → /work-issue
```

## When to use which skill

| Situation | Skill |
|-----------|-------|
| New repo, no AGENTS.md yet | `/init-agents --repo <slug>` |
| New idea, no issue yet (AGENTS.md exists) | `/create-issue "..."` |
| New idea, AGENTS.md missing | `/create-issue "..."` (calls `/init-agents` as a pre-step) |
| Issue exists with a complete spec | `/work-issue <num>` |
| Issue exists without a spec | `/create-issue --refine <num>` OR add the spec manually, then `/work-issue` |
| `/work-issue`'s Validator returned STOP | `/create-issue --refine <num>`, then `/work-issue` retry |
| `/work-issue` returns STOP "AGENTS.md missing" | `/init-agents --repo <slug>`, then `/work-issue` retry |
| AGENTS.md exists but has gaps | `/init-agents --refine --repo <slug>` |

## Example calls

```
# New repo, full loop
/init-agents --repo myorg/myrepo
/create-issue --repo myorg/myrepo "add JWT refresh endpoint"
/work-issue 42 --repo myorg/myrepo

# Drive an existing issue
/work-issue 17

# Quick idea → issue
/create-issue "fix flaky test in checkout-flow.spec.ts"

# Unclear intent
/loop
# → asks one question, then routes to one of the three sub-skills
```

## Why an umbrella

Three sub-skills, one umbrella: **loop engineering** — the loop that drives a specified task through bootstrap → genesis → execution.

- `/init-agents` = capture standards (in the repo's AGENTS.md)
- `/create-issue` = capture intent (in a GitHub issue)
- `/work-issue` = execute intent (issue → merged PR)

## Roadmap

- **v4.1.0 (multi-type loops, issue #22):** same 5-stage skeleton for non-code loops — text loop, decision loop, diagnostic loop.
- **v4.2.0 (public flip, issue #23):** LICENSE, secret sweep, repo-visibility public.

## See also

- `skills/init-agents/SKILL.md`
- `skills/create-issue/SKILL.md`
- `skills/work-issue/SKILL.md`
- `../CLAUDE.md` (plugin quickstart)
