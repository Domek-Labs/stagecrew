# stagecrew — agentic loop workflow

**Turn a GitHub issue into a merged PR (or research findings) with a 5-stage agent pipeline.**

Pure-reader plugin: all coding standards live in your repo's `AGENTS.md`, not in this plugin. You bootstrap once per repo, then any issue can flow through the loop.

**Multi-type system:** the same 5-stage skeleton powers both
code loops (software implementation) and research loops (knowledge generation
with a test matrix). Each type has its own issue template and subagent brief — see the
"Loop Types" section below.

## Quickstart

```
1. /init-agents --repo <owner>/<slug>     # bootstrap AGENTS.md (once per repo)
2. /create-issue "<your idea>"            # idea → fully-specified GitHub issue
3. /plan-issues                           # assign parallel-safety waves (optional, recommended for multi-issue runs)
4. /work-issue <num>                      # issue → merged PR via 5 stages
5. /close-out                             # merge PRs in wave order, close issues
```

Or use the single-command entry point:

```
/run-loop "<your idea>"                   # create → plan → [confirm] → work
```

## The 6 Skills

| Skill | Phase | What it does |
|-------|-------|--------------|
| `/init-agents` | Bootstrap | Writes `AGENTS.md` at the repo root: YAML frontmatter with 11 mandatory standards fields (branch pattern, syntax check, smoke test, hard gates, AC templates, ...) plus the optional `commit_identity` field — 12 total — + Markdown body (architecture, conventions, known gotchas). One-time per repo. Codebase-memory provides intelligent defaults. |
| `/create-issue` | Genesis | Turns an idea into a fully-specified GitHub issue with spec-standard sections (Idea/Spec/AC/Files-To-Touch/Test-Plan/Out-of-Scope). Auto-injects AGENTS.md's `ac_templates`. Calls `/init-agents` as a pre-step if AGENTS.md is missing. |
| `/plan-issues` | Planning | Assigns parallel-safety **waves** to open issues before work starts. Detects file-level collisions, dependency order, stale blocked labels, and migration-number clashes. Outputs a wave plan for human review; writes `wave:<n>` labels on confirmation. Position in pipeline: after `create-issue`, before `work-issue`. |
| `/work-issue` | Execution | Drives a specified issue through 5 stages (Validator → Implementer → Tester → Critic → Closer) until merge. Pure-reader: all standards come from AGENTS.md. Auto-PR, auto-merge on APPROVE. Issue comments serve as the audit log. |
| `/close-out` | Merge | Merges completed PRs in wave order, auto-resolves additive conflicts, closes issues, and cleans up branches. No GitHub Projects dependency — tracks state via issue/PR state and `wave:<n>` labels only. |
| `/run-loop` | Pipeline | Single-command entry point: chains `create-issue → plan-issues → [human confirms] → work-issue` wave by wave. Reduces the common case to one command + one confirmation. `/close-out` is presented as the explicit next step. |

## Umbrella Skill: `/loop`

If you do not know which sub-skill you need, just call `/loop` — it asks one clarifying question and routes you. Triggers on "loop", "loop workflow", "work issue", "spec build", "coding loop", "github workflow".

## Reference Skill: `github`

A convention reference (not a loop phase) that bundles the git/gh interaction rules the stages follow: use `gh` / the GitHub API for writes when the local `.git` is read-only, set the commit author from AGENTS.md `commit_identity`, never a company/shared email, no `Co-authored-by:` trailers unless configured (`no_unconfigured_coauthors`), PR body conventions (`Closes #`, standards block), and the squash-merge co-author caveat. See `skills/github/SKILL.md`.

## Loop Types

| Type | When | Deliverable | Template | Implementer brief |
|------|------|-------------|----------|-------------------|
| `code` (default) | feature, fix, refactor, dependency update | PR with code diff + tests | `skills/create-issue/references/issue-templates/code.md` | `skills/work-issue/references/subagent-briefs/code-implementer.md` |
| `research` | knowledge generation, test-matrix study, library comparison, API behavior probe | `docs/research/<topic>-<date>.md` + follow-up issue spec | `skills/create-issue/references/issue-templates/research.md` | `skills/work-issue/references/subagent-briefs/research-implementer.md` |

Type selection:
- `/create-issue --type=<code|research> "<idea>"` — explicit
- Otherwise auto-inferred from keywords in the user text (`feat`/`fix` → code, `research`/`benchmark` → research)
- Otherwise `loop_types.default` from AGENTS.md (typically `code`)

`/work-issue` reads the `loop-type:<type>` label from the issue and dispatches to the
matching subagent brief. The Tester/Critic/Closer stages stay structurally identical
but check type-specific criteria (build GREEN vs. doc quality, code-diff quality vs.
"follow-up issue spec ready to start").

Both loop types are exercised end-to-end: code loops ship a PR with a diff + tests,
research loops ship a findings doc plus a follow-up implementation-issue spec
(pick a research topic and call `/work-issue --type=research`).

## Optional features

### Component Registry (`components:` block in AGENTS.md)

An **opt-in** AGENTS.md YAML frontmatter block that declares a repo's canonical component set (frontend components, backend value objects, aggregates). When set, the loop enforces reuse: the Validator gates in-scope issues against the registry, the Implementer refuses to inline duplicates, and the Critic runs a dupe-detection pass over the declared code globs. `usage_policy` picks the enforcement level — `prefer_existing` warns, `strict` STOPs and requires an ADR link.

Absent block → zero behavior change. Full schema and enforcement details in `AGENTS.md` under "Component Registry", template in `skills/init-agents/references/components-registry-template.md`, design rationale in `docs/adr/0001-components-registry.md`.

### Commit attribution (`commit_identity` field + `no_unconfigured_coauthors` gate)

The optional `commit_identity` field (`name` + `email`) in the AGENTS.md `work-issue:` namespace fixes the author identity used for every loop commit, so attribution never depends on ambient `git config` or assistant memory. When set, the `/work-issue` Implementer and Closer run `git config user.name`/`user.email` from it before every commit. It pairs with the `no_unconfigured_coauthors` hard-gate: no `Co-authored-by:` trailer or bot footer in commits/PRs unless explicitly configured — this stops a bot or foreign account from being pulled in as a contributor (GitHub appends co-author lines from the squashed commits on squash-merge, so each individual commit must already be clean). Absent → falls back to the ambient `git config`; zero behavior change.

## Parallel-safe `/work-issue` (Phase 1)

`/work-issue` is **parallel-safe**: two agents/terminals can run it concurrently on the same repo without colliding. Each run **claims** its issue in the Stage 0 pre-flight (assignee + `claimed:<agent-id>` label + `## [claim]` comment, with a read-after-write confirm where the earliest claim wins and a 60-min stale-claim reclaim) and works in an **isolated git worktree** instead of the primary checkout. The claim is released on any failure exit (STOP/ESCALATE/hard-cap/abort) so the issue returns to the pool; the worktree is removed on merge or abort. This is Phase 1 of the parallel-multi-agent design ([#7](https://github.com/domek-labs/stagecrew/issues/7)) — no controller or ready-queue yet.

**Worktree cleanup is non-fatal.** When `git worktree remove` fails (long paths / `node_modules` — even with `core.longpaths=true`), the loop does not abort: it prunes the git worktree registration, deletes the local branch, and reports the leftover directory to the user for manual removal, still reaching a terminal verdict. A recursive force-delete is **never** the prescribed fallback (it could destroy state another concurrent loop depends on).

**State-tracker path is OS-portable.** The tracker file is resolved from the environment (`$TMPDIR` / `$TEMP`) with a per-OS default (`/tmp` on Linux/macOS, `%TEMP%` on Windows) instead of a hardcoded `/tmp`, so native tooling on Windows can open it. On Linux/macOS the default is still `/tmp`, so existing runs are unchanged.

**Non-`origin` remotes.** Push, PR create, and branch cleanup use the repo registry's `git_remote` field (via the `{{git_remote}}` placeholder), falling back to `origin` when unset — so a repo whose GitHub working remote is not named `origin` is handled correctly.

## Tester: scope-aware `smoke_test`

`smoke_test` in AGENTS.md is **either** a plain string (unchanged) **or** a scope mapping (`backend` / `frontend` / `default`, plus optional `parallel_safe` and `scopes:`). The `/work-issue` Tester resolves the scope from the diff paths — against `components.code_globs`, or the mapping's own `scopes:` globs when no `components:` block is set — and runs only the matching command. A diff that matches no scope is skipped as "smoke test not applicable to this diff scope", with the reason named in the Tester stage comment. A `parallel_safe: false` smoke test (e.g. `docker compose up -d --force-recreate`) is skipped — with the reason recorded — when another active claim exists on the repo. The plain-string form is a full zero-behaviour-change fallback.

## Closer: merge, CI baseline, and non-default `pr_base`

The Closer applies three deterministic rules and **never terminates without a terminal verdict**:

- **CI base-vs-PR baseline.** A red check blocks the merge only if it is green on `pr_base` and red on the PR (a newly introduced failure). Checks already red on the base are listed in the stage comment as pre-existing and non-blocking. If the base has no check runs to compare against, every red check blocks (fail safe).
- **Never end silently while waiting on CI.** Pending checks are polled with a bounded cap (up to the 90-min wallclock cap); if still pending at the cap, the Closer exits with an explicit `ESCALATE: CI pending` verdict naming the PR number and the next step. In that state the claim is **held** (not released) — the PR is open and the work is done, so releasing would let another agent redo it. This is the deliberate contrast to release-on-failure, which is only for non-merge failures where no PR exists yet.
- **Issue close when `pr_base` ≠ default branch.** `Closes #N` only auto-closes on a merge into the default branch. When `pr_base` diverges from the repo default branch, the Closer **explicitly closes** the issue after the merge (with a comment naming the PR and stating the change reaches the default branch at the next promotion) and removes the `claimed:<agent-id>` label so a later run does not misread a leftover claim as active. Where `pr_base` equals the default branch, behaviour is unchanged.

## Recommended companion MCPs (optional)

Two MIT-licensed external MCPs make the loop richer. Neither is required — the skills call them opportunistically and fall back to plain `grep` / `ls` / no-op when the tool namespace is missing. **Code-graph failures never block a loop:** the `/work-issue` pre-flight distinguishes three non-blocking states — namespace absent, project unknown, and tool inconsistent (e.g. `list_projects` finds the project while `index_status` / `detect_changes` report "not found") — and in all three it skips the freshness check (noting the skip in the stage comment) and continues on grep-based discovery.

- **[codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp)** — local code-graph MCP (tree-sitter based). Used by `/init-agents` for architecture / conventions defaults, `/create-issue` for files-to-touch suggestions, and `/work-issue` in the Validator / Implementer / Critic stages for code-graph-backed evidence.
- **[MemPalace](https://github.com/MemPalace/mempalace)** — verbatim knowledge-store MCP. The `/work-issue` Closer stage persists a loop summary (repo, PR, commit, Tester findings) as a "drawer" so future sessions can search prior decisions.

## Full pipeline at a glance

```
Bootstrap (once per repo)   Genesis          Planning        Execution        Merge
       │                       │                │                │               │
       ▼                       ▼                ▼                ▼               ▼
  /init-agents   →   /create-issue   →   /plan-issues   →   /work-issue   →   /close-out
  (AGENTS.md)      (GitHub issues)    (wave labels)      (5-stage loop)    (ordered merge)

Or all at once: /run-loop "<idea>"  →  [confirm plan]  →  loops fire  →  /close-out
```

## Roadmap

See the repo issues (labelled `roadmap`) for the rolling roadmap.

## License

MIT — see [LICENSE](LICENSE).

## Status

Alpha. In active development. Under `0.x` the API may shift between minor bumps — see `AGENTS.md` `version_policy` for the exact patch / minor / major definition. The current version lives only in `.claude-plugin/plugin.json`.

## Optional feature: Visual Reviewer Gate (`visual:` block in AGENTS.md)

An **opt-in** AGENTS.md YAML frontmatter block that adds a sixth crew role — a **Visual Reviewer** — to the loop. When set, `/work-issue` runs a stage (3.5, between Tester and Critic) that serves the built frontend and inspects it with Playwright: for each declared route × viewport (mobile first) it navigates, screenshots, snapshots the a11y tree, and reads the console, then posts `## [stage:visual] PASS|FAIL` with the screenshots attached. A FAIL routes back to the Implementer under the shared 3-revise cap.

It is a **gate, not a loop-type**: frontend work still ships a code diff and keeps the `code` pipeline; the visual review layers on top, on the same axis as `components:`. Fires only when the repo has a `visual:` block AND the issue is frontend-scoped. Absent block → zero behavior change. Schema and enforcement in `AGENTS.md` under "Visual Reviewer Gate"; rationale in `docs/adr/0002-visual-gate.md`.

### Companion MCP: Playwright (optional)

The Visual Reviewer needs the **Playwright** MCP (`mcp__playwright__*`) to drive the browser. Like codebase-memory and MemPalace it is opportunistic: when the namespace is absent the stage is skipped with a note — never a hard fail.
