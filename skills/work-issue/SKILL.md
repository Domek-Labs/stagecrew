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
  smoke_test: "<command>"             # plain string (e.g., "docker compose build && smoke") OR a scope mapping (see below)
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

### Scope-aware `smoke_test` (and parallel-safety)

`smoke_test` accepts **either** a plain string (the current behaviour, unchanged) **or** a scope mapping. The plain string is the zero-change form — a plain-string `smoke_test` behaves exactly as it always has (the Tester runs it verbatim for every non-docs diff).

The scope-mapping form lets a repo run different smoke commands depending on which part of the codebase the diff touches, and declare whether the smoke test is safe to run while another loop is active on the same repo:

```yaml
smoke_test:
  backend: "<command>"        # run when the diff falls in the backend scope
  frontend: "<command>"       # run when the diff falls in the frontend scope
  default: ""                 # run when the diff matches no named scope (empty string = skip)
  parallel_safe: true         # optional, default true; if false, the smoke test is SKIPPED when another active claim exists on the repo
```

**Scope resolution from the diff:** the scope is resolved from the diff paths.
- If the AGENTS.md `components:` block is set, the diff paths are matched against `components.code_globs` (the same globs that gate registry reuse) to decide `frontend` vs `backend`.
- If `components:` is absent, an explicit `scopes:` map inside the `smoke_test` mapping supplies the globs per scope key:
  ```yaml
  smoke_test:
    backend: "<command>"
    frontend: "<command>"
    default: ""
    scopes:
      backend:  ["api/**", "server/**", "**/*.py"]
      frontend: ["web/**", "app/**", "**/*.tsx"]
  ```
- The Tester runs **only** the command for the matched scope. Multiple scopes touched → run each matched scope's command. No scope matched (or the matched scope's command is the empty string) → the smoke test is not applicable to this diff scope; the Tester skips it and names the reason in its stage comment (see the Tester skip rules).

**`parallel_safe: false`** declares a smoke test that is unsafe to run concurrently (e.g. `docker compose up -d --force-recreate`, which tears down containers another loop may be using). When set to `false` AND the loop detects **another active claim on the repo** (reuse the claim-protocol detection from the Stage 0 pre-flight — any other `claimed:*` label / competing assignee / earlier `## [claim]` comment), the Tester **skips** the smoke test and records the reason in its stage comment rather than executing it. Default is `true` (parallel-safe): no change for existing setups. This only honours a declaration — it does not attempt to auto-detect whether an arbitrary command is parallel-safe.

## Codebase-memory requirement (pre-flight)

The skill ALWAYS calls the codebase-memory MCP before stage 1 to give the subagents proper context:

1. **`list_projects`** — check whether `<repo>` is indexed.
2. **If not indexed:** `index_repository` with mode `moderate` (default heuristic).
3. **If indexed:** `index_status` for a freshness check.
   - Threshold: 7 days since last index AND latest commit > index date → auto re-index.
4. **Health check:** if `nodes < 200` OR `source_files < 3` → warn the user:
   > The default heuristic may have excluded important dirs (bin/, docs/, scripts/). Re-index with explicit paths is recommended before the loop runs.

### Code-graph pre-flight — graceful degradation (issue #17)

The codebase-memory (code-graph) MCP is **optional and opportunistic**: any failure falls back to grep-based discovery and **never blocks a loop**. The pre-flight distinguishes **three non-blocking states**, all of which fall back to grep and note a skipped freshness check in the stage comment:

1. **Namespace absent** — the `mcp__*` code-graph tools are not registered at all. Fall back to plain `grep` / `ls`; no freshness check possible.
2. **Project unknown** — the tools respond, but `list_projects` does not list `<repo>` (and an `index_repository` attempt does not resolve it). Fall back to grep; no freshness check.
3. **Tool inconsistent** — the tools disagree about the same project: e.g. `list_projects` lists `<repo>` while `index_status` / `detect_changes` answer "project not found" for the **same** project name **and** the same absolute path. Because the namespace is present, the "namespace absent" fallback does not apply on its own — so treat an inconsistent answer **exactly like an absent namespace**: note it, **skip the freshness check**, and continue with grep-based discovery.

**None of the three blocks.** Whenever the freshness check is skipped for any of these reasons, **record it in the stage comment** ("code-graph freshness check skipped — `<namespace absent | project unknown | tool inconsistent>`; proceeding on grep") so a later reader knows the code-graph evidence was unavailable rather than clean. A fully working code-graph is unchanged.

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
  git_remote: origin                       # optional — GitHub working remote for push/PR; falls back to `origin` when unset
  has_dev_branch: true | false             # auto-detected at pre-flight
  syntax_check: "<command>"
  smoke_test: "<command>"                   # plain string OR scope mapping — see "Scope-aware smoke_test" below
  deploy_command: "<command>"
  agents_md_exists: true | false           # set by /init-agents
```

`git_remote` names the remote the Implementer pushes to and the Closer creates the PR / cleans up branches against. In a repo whose GitHub working remote is not named `origin` (e.g. `origin` is a legacy mirror and the working remote is `upstream`), set it here. When unset it resolves to `origin`, so existing registries are unaffected. It is exposed to the subagent briefs as the `{{git_remote}}` placeholder.

On the first run for a new repo: ask once for `repo_path` + `deploy_command`, persist to the live file. If the live file is missing: `cp` from the template as a bootstrap.

## Pre-Flight stage 0 (NEW, before Validator)

Run in this order:

1. **Repo-registry lookup** → `repo_path` from `~/.claude/work-issue-paths.yaml`. If missing: ask once for the path.
2. **Claim protocol** — claim the resolved issue before any further work (parallel-safety). See "Claim protocol" below. A lost/already-claimed issue STOPs here — no codebase-memory, no AGENTS.md check, no stage 1.
3. **Codebase-memory pre-flight** (see above: list_projects → index/freshness → health check).
4. **AGENTS.md mandatory check** — see the next section.
5. **Loop-type resolution** — see the next section.
6. **Determine default branch:** `gh api repos/<slug> --jq .default_branch` (cross-check against the AGENTS.md value).
7. **Check for `dev` branch existence:** `gh api repos/<slug>/branches/dev` → 200 or 404 (info only).
8. **Live-path drift check** (if `live_path != repo_path`): `diff -r <live_path> <repo_path>` → on drift, warn the user before stage 1.
9. **Channel inbound detection:** on a `<channel>` tag → short reply "Loop for <slug>#<n> starting (`loop-type:<type>`), stage 1 running."

### Claim protocol (parallel-safety, Phase 1)

`/work-issue` is **parallel-safe**: two agents/terminals may run it concurrently on the same repo. Before any work, an agent **claims** the issue so no two runs collide on the same one. Each run has an `<agent-id>` (a short stable id for this run, e.g. the terminal/session id — used in the label and claim comment). This runs right after issue-number resolution, before the codebase-memory / AGENTS.md checks and the Validator.

1. **Claim** the issue (as atomically as GitHub allows):
   - `gh issue edit <n> --add-assignee @me`
   - Add a `claimed:<agent-id>` label: `gh issue edit <n> --add-label claimed:<agent-id>` (create the label first if it does not exist: `gh label create claimed:<agent-id> --color FBCA04 --force`).
   - Post a claim comment: `## [claim] <agent-id> @ <ISO-8601-ts>`.

2. **Read-after-write confirm** — re-fetch the issue: `gh issue view <n> --json assignees,labels,comments`. If a **competing active claim** exists (another `claimed:*` label, a different assignee, or an earlier `## [claim]` comment):
   - **Earliest claim timestamp wins.** Compare the `## [claim]` comment timestamps.
   - If this run is the **loser**: **release** — remove its own `claimed:<agent-id>` label and its assignee (`gh issue edit <n> --remove-label claimed:<agent-id> --remove-assignee @me`), post a `## [claim:released]` note, and **STOP** with "already claimed by `<other-agent-id>`". No stage 1.
   - If this run is the **winner**: continue.

3. **Stale-claim reclaim** — if an existing competing claim is older than a **TTL (default 60 min)** AND the issue has had **no activity since that claim** (no comments/edits after the claim comment), treat it as abandoned:
   - Release the stale claim (remove the stale `claimed:*` label + its assignee), then claim as in step 1.

On a successful claim, write `claim: { agent_id, claimed_at }` into the state tracker before continuing. The claim is **released on any failure exit** (STOP / ESCALATE / hard-cap / abort) so the issue returns to the pool — see "Release on failure" and each stage's exit note.

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

**Path — resolved from the environment, not hardcoded.** The state tracker lives at `<temp_dir>/loop-<repo_slug_safe>-<issue>.json`, where `<temp_dir>` is resolved once at pre-flight in this order:

1. `$TMPDIR` if set (macOS / most Unix shells).
2. else `$TEMP` if set (Windows — the native temp dir that both the shell and native tooling agree on; avoid Git-Bash `/tmp`, which is not a native Windows path and is resolved differently by native tools).
3. else the per-OS default: `/tmp` on Linux/macOS, `%TEMP%` (typically `C:\Users\<user>\AppData\Local\Temp`) on Windows.

Write the resolved absolute path into the state tracker itself (`tracker_path`) so every stage reads/writes the same file. On Windows this keeps the tracker on a native path that both the shell and native tooling can open. On Linux/macOS the default resolves to `/tmp`, so existing runs are unchanged.

Example (Linux/macOS default): `/tmp/loop-<repo_slug_safe>-<issue>.json`. Example (Windows): `%TEMP%\loop-<repo_slug_safe>-<issue>.json`.

State-tracker schema:

```json
{
  "issue": 5,
  "repo": "your-org/your-repo",
  "repo_path": "/abs/path/to/local/checkout",
  "tracker_path": "/tmp/loop-your-org_your-repo-5.json",
  "worktree_path": null,
  "claim": { "agent_id": "...", "claimed_at": "ISO" },
  "branch": null,
  "default_branch": "main",
  "pr_base": "main",
  "git_remote": "origin",
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

`claim` and `worktree_path` support parallel-safety (Phase 1): `claim` holds the winning `agent_id` + `claimed_at` (ISO) from the claim protocol (cleared on release); `worktree_path` is the dedicated git worktree stages 2–4 run in (`null` until it is created, cleaned up on merge/abort). See "Claim protocol", "Git worktree isolation" and "Release on failure".

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
- STOP → **release the claim + remove the worktree** (see "Release on failure"), loop paused, Telegram text to the user with reasons + suggestion `/create-issue --refine <num>`.
- GO → stage 2.

### Stage 2 — Implementer (type dispatch)

**Brief selection by loop type:**

| `loop_type` from pre-flight | Brief file |
|-----------------------------|------------|
| `code` (default) | `references/subagent-briefs/code-implementer.md` |
| `research` | `references/subagent-briefs/research-implementer.md` |

The skill loads the matching brief file, replaces placeholders (`{{branch_pattern}}`, `{{syntax_check}}`, `{{hard_gates}}`, `{{secret_scan_pattern}}`, `{{issue_num}}`, `{{repo_path}}`, `{{worktree_path}}`, `{{slug}}`, `{{default_branch}}`, `{{commit_format}}`, `{{commit_identity}}`, `{{git_remote}}`) from the AGENTS.md cache + state tracker, and dispatches it as the subagent briefing. `{{commit_identity}}` is `null` when the optional `commit_identity:` block is absent from AGENTS.md. `{{git_remote}}` is the registry `git_remote` field for the repo; it falls back to `origin` when the registry does not declare one (zero behavior change for existing setups), and is used for every push, PR create, and branch cleanup in the Implementer and Closer.

**Git worktree isolation (parallel-safety, Phase 1):** so concurrent runs on the same repo never fight over the primary `repo_path` checkout, the branch is created in a **dedicated git worktree** instead of mutating `repo_path` in place. Before dispatching the Implementer brief:

```bash
git -C <repo_path> worktree add <tmp-dir>/<slug-safe>-<issue> -b <branch> <pr_base>
```

Write the resulting path into the state tracker as `worktree_path` and pass it to the Implementer (`{{worktree_path}}`). **Stages 2–4 (Implementer, Tester, Critic, and the optional Visual Reviewer) operate inside that worktree**, not in `repo_path`. Every `git checkout <branch>` / `git diff` / build in those stages runs with the worktree as the working directory.

**Cleanup:** remove the worktree on **Closer merge** OR on **abort / ESCALATE / STOP / hard-cap**:

```bash
git -C <repo_path> worktree remove <worktree_path> --force
```

The branch itself is deleted by the squash-merge (`--delete-branch`), so worktree removal is the only extra cleanup step.

**Non-fatal cleanup fallback (worktree-removal failure).** `git worktree remove` can fail — typically with "Filename too long" or "Directory not empty" when the worktree contains a `node_modules` tree, reproducible even with `core.longpaths=true` and a deliberately short worktree path. This must **never** abort the loop or block a terminal verdict. On removal failure, run the documented non-fatal fallback:

1. **Clean the git side** so git no longer tracks the worktree:
   ```bash
   git -C <repo_path> worktree prune            # drop the stale worktree registration
   git -C <repo_path> branch -D <branch>        # delete the local branch (if it still exists)
   ```
2. **Report the leftover directory to the user** for manual removal (name the exact path in the parent report and, on a channel run, in the Telegram message).
3. **Continue to the terminal verdict** — the loop still reaches `merged` / `stopped` / `escalated` / `aborted`. A leftover directory is a reported side effect, not a failure.

**Never** prescribe a recursive force-delete (e.g. `rm -rf` / `rmdir /s`) as the cleanup fallback: while other loops run against the same repo, a recursive delete can destroy state another run depends on. The prescribed fallback is git-side pruning plus a manual-removal report — nothing more.

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
> Standards: `smoke_test: <smoke_test>` from AGENTS.md (plain string OR scope mapping — see "Scope-aware smoke_test").
>
> 1. `git checkout <branch>`
> 2. **Resolve the smoke command for this diff:**
>    - **Plain-string `smoke_test`:** use it verbatim (unchanged behaviour).
>    - **Scope-mapping `smoke_test`:** compute the diff paths (`git diff --name-only <default_branch>..<branch>`), match them against `components.code_globs` (or the mapping's own `scopes:` globs when `components:` is absent), and select the command(s) for the matched scope(s). No scope matched, or the matched command is the empty string → **skip** the smoke test with the reason "smoke test not applicable to this diff scope" (name the resolved scope in the comment).
>    - **`parallel_safe: false` AND another active claim exists on the repo** (detect via the claim protocol — any other `claimed:*` label / competing assignee / earlier `## [claim]` comment): **skip** the smoke test with the reason "smoke test not parallel-safe; a concurrent claim is active" (name the competing claim in the comment). Never run a non-parallel-safe smoke test while another loop is active.
> 3. Run the resolved smoke command (or record the skip reason from step 2).
> 4. Per AC checkbox: smoke test + proof (log snippet, command output). For a skipped smoke test, verify the ACs by the other available evidence and state that the smoke step was skipped.
> 5. **Cleanup requirement:** reset live stack/state if touched. Cleanup status in the comment.
> 6. Issue comment `## [stage:tester] <PASS|FAIL>` with per-AC status + proof + (if applicable) the smoke-skip reason + cleanup status.
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
- **`code` loop — docs-only:** diff only in `docs/**` OR `**.md` OR `// comment`-only → skip the whole Tester stage, go directly to Critic. Mark `skip_tester_round_N: docs_only` in the state.
- **`code` loop — smoke not applicable to this diff scope:** with a scope-mapping `smoke_test`, when the diff matches no named scope (or the matched scope's command is empty), the Tester still runs the AC verification but **skips the smoke command** with the reason "smoke test not applicable to this diff scope". Mark `skip_smoke_round_N: scope_not_applicable` in the state. The Tester names the resolved scope in its comment.
- **`code` loop — smoke not parallel-safe:** with `parallel_safe: false` and another active claim on the repo, the Tester skips the smoke command with the reason "smoke test not parallel-safe; concurrent claim active". Mark `skip_smoke_round_N: parallel_unsafe` in the state and name the competing claim in the comment.
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
- REVISE → `iterations++`, stage 2 again with the Critic comment as the briefing. **Hard cap: 3 revises** (on hitting the cap: release the claim + remove the worktree, see "Release on failure").
- ESCALATE → **release the claim + remove the worktree** (see "Release on failure"), pause, Telegram text to the user.

### Stage 5 — Closer (type routing)

**Briefing (template):**
> 0. **Commit identity:** if AGENTS.md carries a `commit_identity` block, run `git config user.name "<name>"` and `git config user.email "<email>"` from it before any commit you make in this stage. Never a company/shared email as author. Do **not** add a `Co-authored-by:` trailer or bot footer to any commit or to the PR body — honor `no_unconfigured_coauthors` (GitHub appends co-author lines from the squashed commits on squash-merge, so each individual commit must already be clean).
> 1. Create a PR (`gh pr create --base <pr_base> --head <branch>`) with body (summary, standards used from AGENTS.md, test plan, "Closes #N", loop-workflow process block). Base: `<pr_base>`. The push that preceded this stage used `<git_remote>` — the PR is created against the same remote.
>    - **`code` loop:** PR title `<commit_format>`-compatible (e.g., `feat(scope): ...`).
>    - **`research` loop:** PR title `research(<topic>): findings + follow-up issue spec`.
> 2. **CI base-vs-PR baseline (before merge — see "CI base-vs-PR baseline" below):** fetch the check conclusions for the head of `<pr_base>` and for the PR. A check **blocks** the merge ONLY if it is green on `<pr_base>` AND red on the PR (newly introduced). Checks already red on `<pr_base>` are **pre-existing, non-blocking** — enumerate them in the stage comment. If `<pr_base>` has no check runs to compare against → every red check blocks (fail safe).
> 3. **Never end silently while waiting for CI (see "No silent Closer termination" below):** if merge-relevant checks are still pending, **poll with a bounded cap** (up to the 90-min wallclock cap). If still pending at the cap, exit with the explicit verdict `ESCALATE: CI pending` naming the PR number and the concrete next step. **HOLD the claim** in this state — do NOT release it (the PR is open and the work is done; releasing would let another agent redo finished work). This is the deliberate exception to "Release on failure".
> 4. **Merge:** once no check blocks (step 2) and none are pending (step 3), squash-merge to `<pr_base>` (`--delete-branch`).
> 5. **Issue close by pr_base vs default branch (see "Issue close when pr_base ≠ default branch" below):** determine the repo default branch (`gh api repos/<slug> --jq .default_branch`) and compare it to `<pr_base>`.
>    - **`<pr_base>` == default branch:** unchanged — GitHub auto-closes the issue via `Closes #N`. Leave the claim label to be cleaned as today.
>    - **`<pr_base>` != default branch:** GitHub will NOT auto-close (the PR did not merge into the default branch). After the squash-merge, **explicitly close the issue** — `gh issue close <n> --comment "Merged via PR #<pr> into <pr_base>; the change reaches the default branch (<default_branch>) at the next promotion."` — and **remove the `claimed:<agent-id>` label** (`gh issue edit <n> --remove-label claimed:<agent-id>`) so a later run never misreads a leftover claim as active.
> 6. Drift check before stack rebuild: `diff -r <live_path> <repo_path>` → on drift, warn, no auto rebuild.
> 7. **Type-specific deploy step:**
>    - **`code` loop:** pull locally + rebuild live stack with `<deploy_command>` (from AGENTS.md or registry). Health check post-deploy.
>    - **`research` loop:** no deploy (doc-only). Optionally file a follow-up implementation issue via `/create-issue --type=code` with the spec stub from the doc.
> 8. **(Optional) persist a loop summary** in your knowledge system. If you use [MemPalace](https://github.com/MemPalace/mempalace), call `mcp__mempalace__add_drawer` with palace/wing/room suited to your setup — e.g. `<your-palace>/<your-code-wing>/<your-changes-room>` for code loops (repo + PR + commit + loop summary + Tester findings) and `<your-palace>/<your-personal-wing>/<your-process-room>` for research loops (loop-pattern insights + doc path + follow-up issue spec). Skip this step if your team uses a different knowledge store (or none).
> 9. **Worktree cleanup:** `git -C <repo_path> worktree remove <worktree_path> --force` (the branch was already deleted by the squash-merge). On removal failure, apply the documented **non-fatal fallback** (prune the worktree registration + delete the local branch + report the leftover directory) from "Git worktree isolation" — never a recursive force-delete, and never a blocker for the terminal verdict. The claim is not released on a successful merge — the issue is closed (via `Closes #N`, or explicitly for a non-default `pr_base`), not returned to the pool.
> 10. Issue comment `## [stage:closer] merged & deployed` (or `merged` for research) with PR number + wallclock + drawer ID + (if applicable) the explicit-close note + any pre-existing-failure list + any leftover-worktree path.
> 11. Loop state final (`status: "closed"`, `closed_at: ...`).
>
> Parent output: max 250 words with PR number, merge commit, deploy status (or n/a for research), drawer ID, wallclock, and — if it applied — the CI-pending escalation, the pre-existing-failure list, the explicit-close note, or a leftover-worktree path.

**Constraint:** on a stack crash post-deploy: NO automatic revert. Report to the parent; the parent decides with the user.

**The Closer never terminates without a terminal verdict** — `merged` (& deployed), or an explicit `ESCALATE: CI pending` / `STOP` naming the PR and the next step. Silent termination while waiting on CI is impossible (see step 3 and "No silent Closer termination").

#### CI base-vs-PR baseline (issue #15)

The Closer has a deterministic rule for which red checks block the merge, so a repo carrying pre-existing baseline failures neither blocks forever nor merges over a genuine regression:

1. Fetch the check conclusions for the **head of `<pr_base>`** (`gh api repos/<slug>/commits/<pr_base>/check-runs --jq '.check_runs[] | {name, conclusion}'`) and for the **PR** (`gh pr checks <pr> --repo <slug>` or `gh api repos/<slug>/commits/<pr-head-sha>/check-runs`).
2. A check **blocks the merge only if it is green (or absent) on the base AND red on the PR** — a newly introduced failure. Name each blocking check in the stage comment.
3. Checks **already red on the base** are pre-existing: list them in the Closer stage comment as pre-existing and non-blocking; do not silently ignore them, and do not let them block.
4. If the base has **no check runs** to compare against → treat **every red check as blocking** (fail safe).

This is the merge-blocking rule; whether the run then *waits* on a still-pending check is the separate concern in "No silent Closer termination".

#### Issue close when pr_base ≠ default branch (issue #13)

`Closes #N` only auto-closes when the PR merges into the repository **default branch**. With `pr_base` set to a non-default integration branch (e.g. `pr_base: dev`, `default_branch: main`), GitHub does not auto-close and the issue would otherwise stay open — inconsistently across runs. The deterministic rule:

- The Closer determines the default branch (`gh api repos/<slug> --jq .default_branch`) and compares it to `<pr_base>`.
- **Divergent (`pr_base` != default):** after the squash-merge into `<pr_base>`, **explicitly close the issue** with a comment naming the PR and stating the change reaches the default branch at the next promotion, **and remove the `claimed:<agent-id>` label** (claim released/cleared) so a later run never misreads a leftover claim as an active one.
- **Equal (`pr_base` == default):** unchanged — GitHub auto-closes via `Closes #N`.

#### No silent Closer termination (issue #16)

The Closer **must never end without a terminal verdict**. Waiting on CI is a bounded state, not a silent hang:

- Poll the merge-relevant checks with a **documented cap** (up to the 90-min wallclock cap).
- If checks are still pending at the cap, exit with the explicit verdict **`ESCALATE: CI pending`**, naming the PR number and the concrete next step (e.g. "re-run `/work-issue <n>` once CI is green, or merge manually").
- **Claim handling — HOLD, do not release.** In the waiting/escalated CI-pending state the claim is deliberately **held**: the PR is open and the work is done, so returning the issue to the pool would let another agent redo finished work. This is the intentional contrast to "Release on failure" (which returns the issue to the pool only for non-merge failures — Validator STOP, Critic ESCALATE, hard-cap, abort — where no PR exists yet).

## Hard caps

- Max 3 Implementer↔Critic revise rounds (a Visual Reviewer FAIL shares this cap)
- Max wallclock 90 min — on overrun, pause + Telegram text
- 3x build fails in Tester → ESCALATE

On **any** hard-cap hit (revise cap, wallclock overrun, 3x build fail), the run exits — **release the claim and remove the worktree** as described in "Release on failure" below.

**Exception:** a wallclock cap reached while the **Closer is waiting on CI for an already-open PR** is not a release case — it exits as `ESCALATE: CI pending` and **holds** the claim (see the Closer's "No silent Closer termination"). Release-on-failure applies to the wallclock cap only when no PR has been merged/opened for the issue yet.

## Release on failure (parallel-safety, Phase 1)

Any exit that is **not** a successful Closer merge — a Validator STOP, a loop-type / AGENTS.md STOP, a Critic ESCALATE, a hard-cap hit, or any abort — must return the issue to the pool so another agent can pick it up:

1. **Release the claim:** remove the run's own `claimed:<agent-id>` label and its assignee — `gh issue edit <n> --remove-label claimed:<agent-id> --remove-assignee @me` — and post a `## [claim:released]` note stating the reason (e.g. `## [claim:released] Validator STOP` / `... ESCALATE` / `... hard-cap: 3 revises`).
2. **Remove the worktree** (if one was created): `git -C <repo_path> worktree remove <worktree_path> --force`. On removal failure, apply the documented **non-fatal fallback** (prune the worktree registration + delete the local branch + report the leftover directory) from "Git worktree isolation" — never a recursive force-delete, and the loop still reaches its terminal `stopped` / `escalated` / `aborted` verdict.
3. Set the state tracker `status` accordingly (`stopped` / `escalated` / `aborted`) and clear `claim`.

A successful Closer merge does the same worktree cleanup but keeps the audit trail (the issue is closed by `Closes #N`, or explicitly for a non-default `pr_base`, not returned to the pool).

**Exception — CI-pending HOLDS the claim (issue #16).** The Closer's `ESCALATE: CI pending` state is **not** a "Release on failure" case. Release-on-failure exists for non-merge failures where **no PR has been created yet**, so returning the issue to the pool is safe and correct. A CI-pending Closer, by contrast, has already created the PR and finished the work — releasing the claim would let another agent redo it. So in the CI-pending state the claim is deliberately **held** (not released), the worktree is kept until the PR is resolved, and the run still exits with an explicit verdict naming the PR and the next step. See "No silent Closer termination" in the Closer stage.

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
| Persistence | issue comments + `<temp_dir>/loop-*.json` (temp dir resolved from `$TMPDIR`/`$TEMP` per OS — see "State tracker") + optional knowledge-store drawer at the end |
| Stop criterion | Critic APPROVE + Closer merge + deploy OK |
| Escalation | Telegram text on STOP/ESCALATE/hard-cap |

## See also

- `skills/init-agents/SKILL.md` — mandatory bootstrap BEFORE the first run in a repo
- `skills/create-issue/SKILL.md` — issue-genesis phase
- `skills/work-issue/references/repo-registry.yaml.example`
- `skills/work-issue/references/subagent-briefs/code-implementer.md` — code-loop brief
- `skills/work-issue/references/subagent-briefs/research-implementer.md` — research-loop brief
- `skills/init-agents/references/AGENTS.md.template`
