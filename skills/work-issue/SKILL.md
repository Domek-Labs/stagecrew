---
name: work-issue
description: "Drive a GitHub issue through a 5-stage spec→build loop (Validator → Implementer → Tester → Critic → Closer). Multi-repo. Pure-reader — all standards values come from AGENTS.md at the repo root (no skill defaults). Multi-type system: reads the loop-type:<type> label and dispatches to a type-specific subagent brief (code|research). Mandatory pre-flight checks AGENTS.md existence, YAML parseability and completeness. Codebase-memory mandatory pre-flight. Auto-PR + merge on APPROVE. Triggers: /work-issue, work issue, loop workflow, drive issue, spec build loop."
---

# /work-issue — Spec→build loop for GitHub issues

**Type:** loop workflow / autonomous issue implementation

## Purpose

Drive a fully-specified GitHub issue (Idea + Spec + AC + Files-To-Touch + Test-Plan + OoS) to merge through 5 dedicated subagents. Each stage = one agent that posts a `[stage:<name>]` comment on the issue as an audit log, then the next stage starts.

This skill is a **pure-reader**: no hardcoded defaults. All standards values (branch_pattern, syntax_check, smoke_test, ...) come from **AGENTS.md at the repo root**. Missing AGENTS.md → STOP with a pointer to `/init-agents`.

Pattern from internal loop-workflow experiments (2026-06). First end-to-end test ran in 33 minutes with 2 iterations and all 5 acceptance criteria green.

## Invocation variants

```
/work-issue 5 --repo your-org/your-repo
/work-issue your-org/your-repo#5
/work-issue 5                                # → fallback: cwd git repo
/work-issue                                  # → lists open issues, asks for selection
```

**Argument parsing:**
1. `<owner>/<repo>#<num>` matched → split into `repo` + `issue_num`
2. `--repo <slug>` flag + bare number
3. `git -C $(pwd) remote get-url origin` → cwd fallback
4. No number → `gh issue list --repo <slug> --state open --limit 30`, user picks
5. Issue number cannot be resolved → ask whether `/create-issue` should be called

## Standards source (2-tier)

| Tier | Source | What it overrides |
|------|--------|-------------------|
| L1   | `AGENTS.md` at the repo root (YAML frontmatter) | base values — repo-specific conventions |
| L2   | `## Standards Override` block in the issue body | L1 — issue-specific one-shot |

Load L1 → L2, merge right-wins. **L1 is mandatory** (see the pre-flight check below). Skill-default L1 does not exist — `/init-agents` creates AGENTS.md.

**Producer side** (`/create-issue`): auto-injects AGENTS.md `ac_templates` into the AC block and detects issue types (docs/epic/secret/mcp/live-ping/live-service) with required extensions. The Validator stays strict — the producer is opinionated. See `skills/create-issue/SKILL.md`.

## AGENTS.md format (L1)

YAML frontmatter + Markdown body. The frontmatter is parsed by the skill; the body is loaded as context in the subagent briefing.

```yaml
---
work-issue:
  branch_pattern: "feature/<scope>-<issue>"
  default_branch: main | dev
  pr_base: main | dev
  commit_format: "conventional" | "custom"
  syntax_check: "<command>"           # e.g., "node --check <file>"
  smoke_test: "<command>"             # e.g., "docker compose build && smoke"
  deploy_command: "<command>"
  linter: "<command>"
  commit_identity:                    # optional — author identity for all loop commits
    name: "Your Name"
    email: "<id>+<user>@users.noreply.github.com"
  hard_gates:
    - "rule description"
    - "no_unconfigured_coauthors"     # no `Co-authored-by:` trailer unless configured
  default_oos:
    - "..."
  ac_templates:
    - "tests green"
    - "docs updated"
---

# Agent Instructions for <repo>
... narrative context (architecture, conventions, "why so") ...
```

Template: `../init-agents/references/AGENTS.md.template`. If a repo does not yet have an AGENTS.md, call `/init-agents --repo <slug>`.

## Codebase-memory requirement (pre-flight)

The skill ALWAYS calls the codebase-memory MCP before stage 1 to give the subagents proper context:

1. **`list_projects`** — check whether `<repo>` is indexed.
2. **If not indexed:** `index_repository` with mode `moderate` (default heuristic).
3. **If indexed:** `index_status` for a freshness check.
   - Threshold: 7 days since last index AND latest commit > index date → auto re-index.
4. **Health check:** if `nodes < 200` OR `source_files < 3` → warn the user:
   > The default heuristic may have excluded important dirs (bin/, docs/, scripts/). Re-index with explicit paths is recommended before the loop runs.

### Where each tool is used in the loop

- **Validator (stage 1):** `get_architecture` — check the files named in the issue against the graph (do they exist? are they in the cluster?).
- **Implementer (stage 2):** `search_code` for sibling functions + conventions in the repo.
- **Critic (stage 4):** `search_code` per AC checkbox for code evidence ("AC says X, code shows Y").

## Repo registry

Live file: `~/.claude/work-issue-paths.yaml` (persists across plugin updates).
Template: `references/repo-registry.yaml.example` (shipped with the plugin).

Fields per repo:

```yaml
<owner>/<repo>:
  repo_path: /abs/path/to/checkout
  live_path: /abs/path/to/deployed         # optional, for drift check
  default_branch: main | dev
  pr_base: main | dev
  has_dev_branch: true | false             # auto-detected at pre-flight
  syntax_check: "<command>"
  smoke_test: "<command>"
  deploy_command: "<command>"
  agents_md_exists: true | false           # set by /init-agents
```

On the first run for a new repo: ask once for `repo_path` + `deploy_command`, persist to the live file. If the live file is missing: `cp` from the template as a bootstrap.

## Pre-Flight stage 0 (NEW, before Validator)

Run in this order:

1. **Repo-registry lookup** → `repo_path` from `~/.claude/work-issue-paths.yaml`. If missing: ask once for the path.
2. **Codebase-memory pre-flight** (see above: list_projects → index/freshness → health check).
3. **AGENTS.md mandatory check** — see the next section.
4. **Loop-type resolution** — see the next section.
5. **Determine default branch:** `gh api repos/<slug> --jq .default_branch` (cross-check against the AGENTS.md value).
6. **Check for `dev` branch existence:** `gh api repos/<slug>/branches/dev` → 200 or 404 (info only).
7. **Live-path drift check** (if `live_path != repo_path`): `diff -r <live_path> <repo_path>` → on drift, warn the user before stage 1.
8. **Channel inbound detection:** on a `<channel>` tag → short reply "Loop for <slug>#<n> starting (`loop-type:<type>`), stage 1 running."

### AGENTS.md mandatory check

Before stage 1, check in this order — every failure is a **STOP verdict** (no stage 1, no subagent start).

1. **Existence**: `ls <repo_path>/AGENTS.md`
   - Missing → STOP:
     > AGENTS.md is missing at the repo root (`<repo_path>/AGENTS.md`). Please run `/init-agents --repo <slug>` first to create the standards spec. Then re-run `/work-issue <num> --repo <slug>`.

2. **YAML parseability**:
   ```bash
   python3 -c "import yaml; yaml.safe_load(open('<repo_path>/AGENTS.md').read().split('---')[1])"
   ```
   - Error → STOP:
     > AGENTS.md has a syntax error in the YAML frontmatter: `<error-message>` (line `<line>`). Please fix or call `/init-agents --refine --repo <slug>` to reset fields.

3. **Completeness**: all 11 fields in the `work-issue:` namespace are set (`branch_pattern`, `default_branch`, `pr_base`, `commit_format`, `syntax_check`, `smoke_test`, `deploy_command`, `linter`, `hard_gates`, `default_oos`, `ac_templates`).
   - Missing fields → STOP:
     > AGENTS.md is incomplete. Missing fields: `<list>`. Please call `/init-agents --refine --repo <slug>` to add them.

   Note: empty lists (`[]`) and empty strings (`""`) count as set — `deploy_command: ""` is allowed, `deploy_command` missing is STOP.

   **`commit_identity` is OPTIONAL** and is **not** part of this mandatory 11-field completeness check. An existing AGENTS.md without a `commit_identity` block must **not** STOP here — cache it as absent and fall back to the ambient `git config`. Only when present, cache its `name`/`email` for the Implementer/Closer to apply before committing.

4. **Optional `components:` block** — if present at the top level of the frontmatter, cache it too. Fields when present: `registry_path` (string, required), `code_globs` (list of glob strings, required), `usage_policy` (`prefer_existing` | `strict`, default `prefer_existing`), `scope` (`frontend` | `backend` | `both`, default `both`). Absence is fine — the loop then skips all registry gates cleanly (zero behavior change). Presence with missing required fields → STOP with a hint to `/init-agents --refine`.

5. **Optional `visual:` block** — if present at the top level of the frontmatter, cache it too. Fields when present: `serve_command` (string, required), `base_url` (string, required), `viewports` (list, default `[{w:390,h:844,label:mobile},{w:1280,h:800,label:desktop}]`), `routes` (list, default `["/"]`), `console_error_policy` (`fail` | `warn`, default `fail`), `screenshot_dir` (string, optional), `scope` (`frontend`, default `frontend`). Absence is fine — the Visual Reviewer stage (3.5) is then skipped cleanly (zero behavior change). Presence with a missing required field → STOP with a hint to `/init-agents --refine`.

If all three checks pass (plus the optional-field parse): cache the frontmatter and keep the body available for stage briefings. This cache is the source for all standards values in the subagent briefings.

### Loop-type resolution

Before stage 1, determine the issue loop type. In this order:

1. **Issue label** — `gh issue view <num> --repo <slug> --json labels --jq '.labels[].name'` → look for the `loop-type:<type>` prefix.
2. **Body frontmatter fallback** — if the issue body starts with `---\nloop-type: <type>\n---` → use that value. (Happens when an issue was filed manually without the skill.)
3. **AGENTS.md default fallback** — `loop_types.default` from AGENTS.md (typically `code`).
4. **Last-resort fallback** — `code` (backwards compatibility for issues filed before the multi-type system).

Validation against `loop_types.enabled` from AGENTS.md:
- If the resolved type is **not** in `enabled`: STOP verdict:
  > Issue has `loop-type:<X>`, but AGENTS.md `loop_types.enabled` only contains `<list>`. Please extend AGENTS.md (`/init-agents --refine`) or fix the issue label.

- If the resolved type is `text`/`decision`/`diagnostic` (roadmap types): STOP verdict with roadmap hint:
  > Loop type `<type>` is on the roadmap, not yet implemented. Currently supported: code, research. Track the roadmap in the repo issues with labels `loop-type` + `roadmap`.

On OK: write the resolved type into the state tracker (`loop_type` field). The subagent-brief loader uses this value.

### Subagent-brief loader

For each stage 2 (Implementer), the brief is loaded from `references/subagent-briefs/<loop_type>-implementer.md`. The briefs have placeholders (`{{branch_pattern}}`, `{{syntax_check}}`, etc.) that are replaced at render time from the AGENTS.md cache + state tracker.

| Loop type | Subagent-brief file |
|-----------|---------------------|
| `code` | `references/subagent-briefs/code-implementer.md` |
| `research` | `references/subagent-briefs/research-implementer.md` |

Tester / Critic / Closer stages stay **structurally identical** but check type-specific criteria:
- `code`: build GREEN via `smoke_test`, code-diff quality
- `research`: doc-quality check (word count, test matrix, working setup or hypothesis roadmap)

The type-specific check criteria are documented in the corresponding subagent-brief files (sections "Tester check criteria" and "Critic check criteria").

The opt-in **Visual Reviewer** stage (3.5) uses a stage-specific brief, `references/subagent-briefs/visual-reviewer.md` — not an implementer brief. It is loaded only when the `visual:` gate fires (see "Stage 3.5 — Visual Reviewer" below).

## State tracker

`/tmp/loop-<repo_slug_safe>-<issue>.json`:

```json
{
  "issue": 5,
  "repo": "your-org/your-repo",
  "repo_path": "/abs/path/to/local/checkout",
  "branch": null,
  "default_branch": "main",
  "pr_base": "main",
  "loop_type": "code",
  "agents_md_loaded": true,
  "standards": { "branch_pattern": "...", "syntax_check": "...", "components": { "registry_path": "...", "code_globs": [...], "usage_policy": "...", "scope": "..." } | null, "visual": { "serve_command": "...", "base_url": "...", "viewports": [...], "routes": [...], "console_error_policy": "...", "scope": "..." } | null, ... },
  "visual_gate": "pending | pass | fail | skipped",
  "started_at": "ISO",
  "iterations": 0,
  "stage": "validator",
  "status": "in_progress",
  "verdicts": {}
}
```

`loop_type` is mandatory — see "Loop-type resolution" above.

`standards.visual` mirrors the optional AGENTS.md `visual:` block (`null` when absent). `visual_gate` tracks the Visual Reviewer stage outcome (`pending` until stage 3.5 runs; `skipped` when `visual:` is absent, the issue is not frontend-scoped, or the Playwright MCP is missing).

`repo_slug_safe` = `owner_repo` (slash → underscore). Update after every stage.

## The 5 stages

Each stage = its own subagent via the agent tool (`general-purpose`). Sequential; read output, decide on the next stage. A repo with an opt-in `visual:` block adds one conditional stage — **3.5 Visual Reviewer**, between Tester and Critic — for frontend-scoped issues (see "Stage 3.5 — Visual Reviewer" below); it is skipped cleanly otherwise.

Important: **all standards values in the briefings are placeholders** (`<branch_pattern>`, `<syntax_check>`, ...). They are substituted at runtime from the AGENTS.md cache — no more fixed defaults in the skill.

### Stage 1 — Spec Validator

**Briefing (template):**
> You are validating issue #N in repo `<slug>`. You build nothing.
>
> Standards context: the repo has an AGENTS.md with `hard_gates: <hard_gates>` and `ac_templates: <ac_templates>`. AGENTS.md body:
> ```
> <AGENTS-MD-BODY>
> ```
>
> 1. Read `gh issue view N --repo <slug>`.
> 2. Validator gates:
>    - [ ] ACs are testable (every checkbox concretely verifiable)
>    - [ ] ACs include the `ac_templates` from AGENTS.md
>    - [ ] Files-to-touch exist or are described as new. Check via `get_architecture` from codebase-memory + `ls` in the repo path.
>    - [ ] Dependencies satisfied (`depends on #X` → check `#X` is CLOSED)
>    - [ ] Test plan present
>    - [ ] Out-of-scope named (at minimum `default_oos` from AGENTS.md plus issue-specific)
>    - [ ] `hard_gates` from AGENTS.md addressed
>    - [ ] **Component-Registry gate (only if AGENTS.md `components:` is set AND the issue's Spec or Files-to-Touch overlaps `code_globs` — optionally filtered by `scope`):** the issue names an existing registry component (from `registry_path`) OR links an ADR path in the `## Standards Override` block that justifies adding a new one.
>      - Under `usage_policy: strict` — missing → STOP + "Add an ADR at `docs/adr/<n>-<slug>.md` and link it in `## Standards Override`, or reference the existing registry component."
>      - Under `usage_policy: prefer_existing` — missing → WARN (not STOP); post the warning in the Validator comment; the loop continues.
>      - Absent `components:` block OR issue outside `code_globs` scope → gate skipped silently.
> 3. On STOP: concrete spec-improvement suggestions.
> 4. Output: comment `## [stage:validator] <GO|STOP>` on the issue + parent report (max 100 words).

**Parent decision:**
- STOP → loop paused, Telegram text to the user with reasons + suggestion `/create-issue --refine <num>`.
- GO → stage 2.

### Stage 2 — Implementer (type dispatch)

**Brief selection by loop type:**

| `loop_type` from pre-flight | Brief file |
|-----------------------------|------------|
| `code` (default) | `references/subagent-briefs/code-implementer.md` |
| `research` | `references/subagent-briefs/research-implementer.md` |

The skill loads the matching brief file, replaces placeholders (`{{branch_pattern}}`, `{{syntax_check}}`, `{{hard_gates}}`, `{{secret_scan_pattern}}`, `{{issue_num}}`, `{{repo_path}}`, `{{slug}}`, `{{default_branch}}`, `{{commit_format}}`, `{{commit_identity}}`) from the AGENTS.md cache + state tracker, and dispatches it as the subagent briefing. `{{commit_identity}}` is `null` when the optional `commit_identity:` block is absent from AGENTS.md.

**Commit identity (all commits in stage 2 and stage 5):** if AGENTS.md carries a `commit_identity` block, the Implementer and Closer MUST run `git config user.name "<name>"` and `git config user.email "<email>"` from it **before** any commit. Never use a company/shared email as author. Never add a `Co-authored-by:` trailer or bot footer to a commit message or PR body (honor the `no_unconfigured_coauthors` hard-gate) — GitHub appends co-author lines from the squashed commits on squash-merge, so the individual commits must already be clean.

**Default `secret_scan_pattern`** (when not set via issue override):
```
(ghp_[A-Za-z0-9]{30,}|sk-ant-[A-Za-z0-9_-]{40,}|TELEGRAM_BOT_TOKEN=[0-9]+:[A-Za-z0-9_-]+|API_KEY=[a-zA-Z0-9]{20,})
```

**Parent decision:** spec gap too large → ESCALATE. Otherwise stage 3 (or skip rule).

### Stage 3 — Tester (type-specific check criteria)

**Briefing schema (`loop_type: code`):**
> You verify the ACs from issue #N. You do not write code.
>
> Standards: `smoke_test: <smoke_test>` from AGENTS.md.
>
> 1. `git checkout <branch>`
> 2. Run `<smoke_test>`.
> 3. Per AC checkbox: smoke test + proof (log snippet, command output).
> 4. **Cleanup requirement:** reset live stack/state if touched. Cleanup status in the comment.
> 5. Issue comment `## [stage:tester] <PASS|FAIL>` with per-AC status + proof + cleanup status.
>
> Parent output: max 200 words.

**Briefing schema (`loop_type: research`):**
> You verify the findings doc quality from issue #N. You do **not** run `smoke_test`.
>
> 1. `git checkout <branch>` and find the doc (`docs/research/<topic>-<date>.md`).
> 2. **Doc-quality check** against the issue ACs and the type-check list in
>    `references/subagent-briefs/research-implementer.md` section
>    "Tester check criteria":
>    - [ ] Word count >= 800 (`wc -w`)
>    - [ ] Test-matrix table present
>    - [ ] >= 6 probes documented
>    - [ ] Working-setup section OR hypothesis-roadmap section
>    - [ ] Follow-up implementation-issue spec attached
>    - [ ] Sources list with >= 1 link
>    - [ ] No secrets in the doc (pattern scan)
> 3. Issue comment `## [stage:tester] <PASS|FAIL>` with per-check-item status + doc quote as evidence.
>
> Parent output: max 200 words.

**Skip rules:**
- **`code` loop:** diff only in `docs/**` OR `**.md` OR `// comment`-only → skip Tester, go directly to Critic. Mark `skip_tester_round_N: docs_only` in the state.
- **`research` loop:** skip rule **disabled** — doc-quality check is mandatory (otherwise not verifiable).

**Parent decision:** FAIL → stage 4 (Critic decides revise or escalate). PASS → stage 4.

### Stage 3.5 — Visual Reviewer (opt-in `visual:` gate)

**Runs only when both hold** (otherwise skipped silently, `visual_gate: skipped`):
1. AGENTS.md carries a `visual:` block (cached at pre-flight, see the optional-field parse below), and
2. the issue is **frontend-scoped** — files-to-touch match a frontend glob (e.g. `**/*.tsx`, `**/*.vue`, `**/*.svelte`, `src/pages/**`, `src/components/**`) OR the Spec/AC contain a frontend keyword (`page`, `route`, `UI`, `component`, `layout`, `responsive`, `screen`, `viewport`).

Sits **between Tester (build green) and Critic** — the browser only spins up once the build is green.

**Brief:** `references/subagent-briefs/visual-reviewer.md`. Placeholders (`{{serve_command}}`, `{{base_url}}`, `{{viewports}}`, `{{routes}}`, `{{console_error_policy}}`, `{{branch}}`, `{{issue_num}}`, `{{slug}}`, `{{repo_path}}`) are substituted from the `visual:` cache + state tracker.

**Flow:** serve the built frontend (`serve_command`, wait for `base_url`) → per route × viewport (mobile first): `mcp__playwright__browser_resize` → `browser_navigate` → `browser_snapshot` + `browser_take_screenshot` + `browser_console_messages` → evaluate the issue's Visual Acceptance ACs → tear down → issue comment `## [stage:visual] <PASS|FAIL>` with screenshots attached + per-AC verdict.

**Dependency:** the Playwright companion MCP (`mcp__playwright__*`). Absent namespace → skip with `## [stage:visual] SKIPPED (no playwright MCP)`, `visual_gate: skipped`, continue to Critic. Never a hard fail.

**Parent decision:**
- PASS (`visual_gate: pass`) → stage 4.
- FAIL (`visual_gate: fail`) → `iterations++`, back to stage 2 (Implementer) with the visual defect list as the briefing. **Shares the 3-revise hard cap** with Critic REVISE (a visual FAIL and a Critic REVISE count against the same cap).
- SKIPPED → stage 4.

### Stage 4 — Critic

**Briefing (template):**
> You review branch `<branch>` (HEAD `<commit>`) against issue #N.
>
> Standards context: AGENTS.md body + `hard_gates: <hard_gates>`.
>
> 1. Read the issue ACs + Implementer comment + Tester comment.
> 2. Read `git diff <default_branch>..<branch>`.
> 3. Per AC: `search_code` via codebase-memory for code evidence ("AC says X, code snippet shows Y") → OK/Warn/Fail.
> 4. Evaluate Tester findings: blocking or documentable?
> 5. Out-of-scope check (`<default_oos>` + issue OoS).
> 6. `hard_gates` check (see AGENTS.md).
> 7. Code quality (smoke): naming, error handling, cleanup, security.
> 8. **Component-Registry dupe-detection (only if AGENTS.md `components:` is set):** `search_code` / grep across `<code_globs>` for patterns similar to what this diff introduces (e.g. the diff added a new table-like component — grep the frontend globs for other table-like components; the diff added a monetary VO — grep the backend globs for other monetary handling). If two implementations cover the same concept: raise a REVISE item with both code paths, quoting a snippet from each, and propose either (a) consolidate into the existing registry component or (b) file an ADR that justifies the parallel implementation. This step runs regardless of `usage_policy` — under `prefer_existing` the Critic can still catch a dupe that slipped past the Validator warning.
>
> Verdict:
> - **APPROVE** — all ACs OK, no blocking findings, OoS + hard_gates respected.
> - **REVISE** — actionable items (concrete).
> - **ESCALATE** — spec gap / architectural question.
>
> Issue comment `## [stage:critic] <verdict>` with AC review + OoS check + findings evaluation + (on REVISE) actionable items.
>
> Parent output: max 150 words.

**Parent decision:**
- APPROVE → stage 5.
- REVISE → `iterations++`, stage 2 again with the Critic comment as the briefing. **Hard cap: 3 revises.**
- ESCALATE → pause, Telegram text to the user.

### Stage 5 — Closer (type routing)

**Briefing (template):**
> 0. **Commit identity:** if AGENTS.md carries a `commit_identity` block, run `git config user.name "<name>"` and `git config user.email "<email>"` from it before any commit you make in this stage. Never a company/shared email as author. Do **not** add a `Co-authored-by:` trailer or bot footer to any commit or to the PR body — honor `no_unconfigured_coauthors` (GitHub appends co-author lines from the squashed commits on squash-merge, so each individual commit must already be clean).
> 1. Create a PR with body (summary, standards used from AGENTS.md, test plan, "Closes #N", loop-workflow process block). Base: `<pr_base>`.
>    - **`code` loop:** PR title `<commit_format>`-compatible (e.g., `feat(scope): ...`).
>    - **`research` loop:** PR title `research(<topic>): findings + follow-up issue spec`.
> 2. Squash-merge to `<pr_base>` (`--delete-branch`).
> 3. Drift check before stack rebuild: `diff -r <live_path> <repo_path>` → on drift, warn, no auto rebuild.
> 4. **Type-specific deploy step:**
>    - **`code` loop:** pull locally + rebuild live stack with `<deploy_command>` (from AGENTS.md or registry). Health check post-deploy.
>    - **`research` loop:** no deploy (doc-only). Optionally file a follow-up implementation issue via `/create-issue --type=code` with the spec stub from the doc.
> 5. **(Optional) persist a loop summary** in your knowledge system. If you use [MemPalace](https://github.com/MemPalace/mempalace), call `mcp__mempalace__add_drawer` with palace/wing/room suited to your setup — e.g. `<your-palace>/<your-code-wing>/<your-changes-room>` for code loops (repo + PR + commit + loop summary + Tester findings) and `<your-palace>/<your-personal-wing>/<your-process-room>` for research loops (loop-pattern insights + doc path + follow-up issue spec). Skip this step if your team uses a different knowledge store (or none).
> 6. Issue comment `## [stage:closer] merged & deployed` (or `merged` for research) with PR number + wallclock + drawer ID.
> 7. Loop state final (`status: "closed"`, `closed_at: ...`).
>
> Parent output: max 250 words with PR number, merge commit, deploy status (or n/a for research), drawer ID, wallclock.

**Constraint:** on a stack crash post-deploy: NO automatic revert. Report to the parent; the parent decides with the user.

## Hard caps

- Max 3 Implementer↔Critic revise rounds (a Visual Reviewer FAIL shares this cap)
- Max wallclock 90 min — on overrun, pause + Telegram text
- 3x build fails in Tester → ESCALATE

## Telegram / channel updates

When invoked from a `<channel>` inbound:
- **Pre-stage-1:** "Loop for #N starting, stage 1 running"
- **Between stages:** `Stage N (<name>) → <verdict>. Stage N+1 starting.`
- **Pre-Tester with live ping:** warn if the test plan produces a live-channel ping
- **Post-Closer:** success report with PR link, wallclock, iteration count

Channel/chat_id taken from the inbound meta.

## Issue-comment convention

Every comment starts with `## [stage:<name>] <verdict>` to keep the audit log scannable. Subagents receive this in their briefing.

## Scope boundaries vs. /create-issue and /init-agents

| Skill | When |
|-------|------|
| `/init-agents` | Repo has no AGENTS.md yet → bootstrap (once per repo) |
| `/create-issue` | Idea → fully-specified GitHub issue (genesis phase) |
| `/work-issue` | Issue with spec → build loop until merge (execution phase) |

If `/work-issue` is called without an issue number OR the issue has no spec (no AC, no files-to-touch) → ask the user: "Issue is incomplete. Should I call `/create-issue --refine <num>`?"

If AGENTS.md is missing in the repo → STOP with a pointer to `/init-agents` (see the pre-flight check above). No auto-fallback.

## What this skill does NOT do

- Does not plan new issues itself (that is `/create-issue`)
- Does not create AGENTS.md itself (that is `/init-agents`)
- Cross-repo dependencies are only checked as CLOSED, not auto-worked beforehand
- Does not create ADRs
- Does not auto-revert on a stack crash

## Loop-workflow pattern (reference)

Distilled from internal loop-workflow experiments (2026-06):

| Field | Here |
|-------|------|
| Trigger | `/work-issue <num> [--repo <slug>]` |
| Iteration | 5 stages, revise loop on Critic REVISE |
| Persistence | issue comments + `/tmp/loop-*.json` + optional knowledge-store drawer at the end |
| Stop criterion | Critic APPROVE + Closer merge + deploy OK |
| Escalation | Telegram text on STOP/ESCALATE/hard-cap |

## See also

- `skills/init-agents/SKILL.md` — mandatory bootstrap BEFORE the first run in a repo
- `skills/create-issue/SKILL.md` — issue-genesis phase
- `skills/work-issue/references/repo-registry.yaml.example`
- `skills/work-issue/references/subagent-briefs/code-implementer.md` — code-loop brief
- `skills/work-issue/references/subagent-briefs/research-implementer.md` — research-loop brief
- `skills/init-agents/references/AGENTS.md.template`
