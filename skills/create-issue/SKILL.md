---
name: create-issue
description: "Create a GitHub issue from a feature request or idea, with a full spec standard (Idea/Spec/AC/Files-To-Touch/Test-Plan/Out-of-Scope). Multi-repo. AGENTS.md in the repo overrides default templates. v3.3.0 multi-type system: --type=<code|research> flag plus auto-inference. Templates from references/issue-templates/<type>.md. v3.2.0 auto-injection of AGENTS.md ac_templates into the AC block plus 6-trigger issue-type detection (docs/epic/secret/mcp/live-ping/live-service). Codebase-memory auto-suggests files-to-touch and hotspots. Optional auto-handoff to /work-issue. Triggers: /create-issue, create issue, new issue, file issue, file feature, spec issue, idea as ticket."
---

# /create-issue — Idea → specified GitHub issue

**Type:** issue genesis / spec engineering
**Version:** v3.3.0 (multi-type system: code + research)

## Purpose

Issue-genesis phase. A user idea becomes a fully-specified GitHub issue that `/work-issue` can pick up directly. The skill ensures every issue has Idea + Spec + AC + Files + Test-Plan + OoS before any code gets written.

The spec standard matches what `/work-issue`'s Validator expects as a GO criterion — so no Validator STOPs on the first loop pass.

## Invocation variants

```
/create-issue "voice-diff review for mei"
/create-issue --type=research "benchmark ollama tool-use"
/create-issue --repo dscheinecker-at7media/personal-ai-bot "improve scheduler"
/create-issue                                  # → asks for idea + repo
/create-issue --refine <num> [--repo <slug>]   # → extend an existing issue with spec fields
```

**Argument parsing:** same as `/work-issue` (GitHub syntax, `--repo` flag, cwd fallback).

**`--type=<code|research>` flag (NEW in v3.3.0):** selects the issue template from
`references/issue-templates/<type>.md`. If not set: auto-inference from
user text (see the next section).

## Loop-type resolution (NEW in v3.3.0)

In this order:

1. **Explicit flag** — `--type=<code|research>`
2. **Auto-inference from user text** (keyword heuristic)
3. **AGENTS.md `loop_types.default`** (fallback, typically `code`)
4. **Ambiguous input:** the skill asks explicitly "code loop or research loop?"

### Auto-inference keywords

| Keywords in user text | Inferred type |
|-----------------------|---------------|
| "feat", "feature", "fix", "bug", "refactor", "implement", "add X", "build X" | `code` |
| "research", "investigate", "benchmark", "compare", "debug X (root-cause unclear)", "explore", "evaluate" | `research` |

If code and research score the same → ask explicitly.

### Supported types (v3.3.0)

The skill reads `AGENTS.md` `loop_types.enabled`. In v3.3.0 only `code` and `research`
are implemented. On `--type=text|decision|diagnostic`:

```
Error: loop-type '<type>' is not implemented in v3.3.0.
Roadmap issues:
  - Text loop:       <repo>/issues?label=loop-type,roadmap,text
  - Decision loop:   <repo>/issues?label=loop-type,roadmap,decision
  - Diagnostic loop: <repo>/issues?label=loop-type,roadmap,diagnostic
Use --type=code or --type=research, or file a new issue
for the type-roadmap proposal.
```

### Template loader

The skill loads `skills/create-issue/references/issue-templates/<type>.md`. The
template's frontmatter is read (`loop-type: <type>`); the body is used as the render
schema. Required sections + placeholders define the spec-dialog fields.

### Issue-label setting

On `gh issue create`, the label `loop-type:<type>` is set (in addition to trigger-detection labels).

If the label does not yet exist in the repo, create it first:

```bash
gh label create "loop-type:code" --repo <slug> --color 0E8A16 --description "Software implementation loop" --force
gh label create "loop-type:research" --repo <slug> --color 5319E7 --description "Research/findings loop" --force
```

## Workflow

### a) Repo resolution

Same as `/work-issue`:
1. GitHub syntax `<owner>/<repo>` OR `--repo` flag
2. cwd fallback via `git remote get-url origin`
3. If unclear: ask

### b) AGENTS.md pre-step (NEW in v3.1.0)

**Before anything else:** AGENTS.md existence check at the repo root.

- `ls <repo_path>/AGENTS.md` → exists? → proceed to (c).
- Missing → call `/init-agents --repo <slug> --interactive`, then return to (b) — re-check.

Background: since v3.1.0, `/work-issue` is a pure-reader and needs AGENTS.md. `/create-issue` builds on this — `ac_templates` and `default_oos` from AGENTS.md are suggested as defaults in the spec dialog.

### c) codebase-memory pre-flight

Same block as `/work-issue`:
1. `list_projects` — check whether `<repo>` is indexed
2. If not: `index_repository` (mode `moderate`)
3. If indexed: `index_status` — freshness check (>7 days AND a newer commit → re-index)
4. Health check: `nodes < 200` OR `source_files < 3` → warn "default heuristic may have excluded dirs"

### d) AGENTS.md read

Read at the repo root (guaranteed to exist after step b). Load the frontmatter for `ac_templates`, `default_oos`, `loop_types` — these are suggested as defaults in the spec dialog.

### d2) Loop-type resolution (NEW in v3.3.0)

Before the spec dialog, resolve the loop type (see the "Loop-type resolution" section above):

1. Check explicit `--type=<X>` flag.
2. Otherwise auto-inference from the user text.
3. Otherwise `loop_types.default` from AGENTS.md (fallback `code`).
4. If ambiguous: ask explicitly.

Load the template from `skills/create-issue/references/issue-templates/<type>.md`.
The section schema and placeholder list for the spec dialog come from the template body.

### e) Spec dialog (interactive, 7 questions)

The user is walked through the fields below. On every question the skill is welcome to offer concrete suggestions.

1. **Idea + strategic context (Why)** — what should actually be achieved, what user pain is solved?
2. **Files to touch** — which files will likely be modified?
   - **The skill actively suggests:** extract keywords from the idea (e.g., "scheduler", "voice-diff", "ollama") → `search_code` with the keywords → functions + files. `get_architecture` for cluster context. Suggestion: "These files will likely be touched: <list>. Correct?"
   - Flag hotspots in the code graph as diff risks.
3. **Acceptance criteria** as a checkbox list
   - Since v3.2.0 the AGENTS.md `ac_templates` are **injected** (not just suggested) — see the "Auto-injection of AGENTS.md `ac_templates`" section below.
   - The user adds issue-specific ACs, which are appended after the required templates.
   - In addition, issue-type detection (6 triggers) runs before the preview — see the "Issue-type detection" section.
4. **Test plan** — which smoke tests? (build command, service restart, manual check)
5. **Out of scope** — what is explicitly NOT done in this issue?
   - The skill suggests `default_oos` from AGENTS.md (e.g., "multi-user support", "web-channel approvals")
6. **Dependencies** — which issues block this? Which does it block? (`depends on #X`, `blocks #Y`)
7. **Milestone + labels** — `gh api repos/<slug>/milestones` → the user picks from open milestones. Labels free.

### f) Issue preview

Render the full issue body. User: APPROVE / REVISE / CANCEL.

### g) Issue create

```
gh issue create --repo <slug> \
  --title "<title>" \
  --body "<rendered-body>" \
  --milestone "<milestone>" \
  --label "loop-type:<type>,<other-labels>"
```

The `loop-type:<type>` label is mandatory — `/work-issue` needs it to pick the subagent brief. If the label does not exist yet, run `gh label create` first (see "Issue-label setting" above).

### h) Optional: auto-handoff to /work-issue

> Issue #<num> created. Drive it through with `/work-issue` now? (Yes/No)

On yes → forward issue number + repo to `/work-issue`. On no → the issue stays in the backlog.

## codebase-memory integration in detail

For the files-to-touch suggestion and spec enrichment:

1. Extract keywords from the idea (nouns, function names, module hints).
2. `search_code` for each keyword → matching functions + files.
3. `get_architecture` → cluster context for the matches (which modules are connected).
4. Hotspots = files with high connection density → flag as diff risks.
5. Suggestion to the user: "These files will likely be touched: `path/a.ts`, `path/b.ts`. Hotspot warning for `path/core.ts` (high connectivity — changes there have wider impact)."

## AGENTS.md generation (delegated to /init-agents)

Up to v3.0.0, `/create-issue` had an inline auto-generation workflow for AGENTS.md. Since v3.1.0 this is centralized in `/init-agents` (single source).

If AGENTS.md is missing, `/create-issue` calls `/init-agents --repo <slug> --interactive` in the pre-step (step b) and returns afterward. That guarantees `/work-issue` (pure-reader since v3.1.0) can pick the issue up directly.

## Repo-registry update

If `<repo>` is not in `~/.claude/work-issue-paths.yaml` yet:
- ask once for `repo_path`
- ask once for `deploy_command` (optional)
- ask once for `live_path` (optional, otherwise = `repo_path`)
- persist

The repo is then ready for `/work-issue` immediately afterward.

## Issue-body templates (multi-type since v3.3.0)

Templates live externally in `references/issue-templates/<type>.md` and are loaded at
render time. v3.3.0 supports two types:

| Type | Template file | When |
|------|---------------|------|
| `code` | `references/issue-templates/code.md` | software implementation tasks (feature, fix, refactor) |
| `research` | `references/issue-templates/research.md` | knowledge generation, test-matrix study, library comparison |

Both templates define the required sections that `/work-issue`'s Validator checks strictly.

### Shared required sections (all types)

- `## Idea (Why)`
- `## Spec (What)`
- `## Acceptance Criteria`
- `## Files to Touch`
- `## Test Plan`
- `## Dependencies / Blocks`
- `## Out of Scope`
- `## Standards Override` (optional)
- `## Standards Notes` (optional, but mandatory for research type via the template)

For type-specific differences see the template files.

### Roadmap (v4.2.0+)

`text`, `decision`, `diagnostic` templates are not implemented yet. See the
roadmap issues in the repo (labels `loop-type` + `roadmap`).

## Standards source

Same as `/work-issue` (2-tier since v3.1.0: AGENTS.md → issue override). `/create-issue` uses it primarily for:
- `ac_templates` (since v3.2.0 **auto-injected** in the AC block — see the next section, no longer just "suggested")
- `default_oos` (suggested in the spec dialog)
- `hard_gates` (inserted as required ACs when relevant)

## Auto-injection of AGENTS.md `ac_templates`

Since **v3.2.0** the render logic in step (e3) acceptance criteria is opinionated: AGENTS.md `ac_templates` are **injected**, no longer just "suggested". Three modes:

### Mode 1 — default (no standards override)

All `ac_templates` items from AGENTS.md are **prepended** in the rendered AC block as required items (clearly labeled "**from AGENTS.md `ac_templates`**"), followed by issue-specific user ACs.

### Mode 2 — override `ac_templates: []` (empty)

The block is left empty; the user must explicitly supply replacement ACs (typical for docs-only issues). The skill **warns** if the AC block stays empty and no user ACs are submitted.

### Mode 3 — override with a partial/different list

The override list **completely replaces** the AGENTS.md templates and is rendered in the same required block, plus user ACs.

### Rendered-format example

```markdown
## Acceptance Criteria

### from AGENTS.md `ac_templates` (required)
- [ ] <template 1>
- [ ] <template 2>
- ...

### Issue-specific
- [ ] <user AC 1>
- [ ] <user AC 2>
```

This injection becomes active in step (e3) of the workflow definition — no longer "suggest", but **inject**. During the spec dialog the user can rephrase required items via `EDIT` or remove them via `DISMISS` (with reasoning in the `## Standards Notes` block), but they are present by default.

## Issue-type detection

6 triggers, regex/keyword-based. The heuristic runs BEFORE the render step (e6 issue preview). Triggers can stack — an issue can fire multiple triggers (e.g., a new MCP skill with `ANTHROPIC_API_KEY` → #3 + #4 active).

| # | Trigger condition | Required extension |
|---|-------------------|--------------------|
| 1 | Label `docs`/`documentation` OR all files-to-touch match `^docs/`, `\.md$`, `^LICENSE$`, `^CONTRIBUTING` | Standards-override block (`ac_templates: []`) + suggest docs-specific replacement ACs (Markdown lint, ADR-schema grep, Mermaid via `mmdc`) |
| 2 | Label `epic` OR `>5` files-to-touch OR spec block `>800` words | Epic warning in the preview: "Sub-issue split recommended. OoS must contain the sub-issue list or the issue must be restructured." |
| 3 | Spec OR test plan contains the pattern `[A-Z_]+_(KEY\|TOKEN\|SECRET\|PAT)` OR the strings "token"+"env" / "secret"+"PAT" | HG3 reminder: ACs for `.env.example` entry, `docker-compose env_file:` hint, entrypoint preflight, secret scan before commit |
| 4 | Spec contains `claude mcp add` OR `mcp__` OR "MCP tool" OR "MCP plugin" | HG1 reminder: ACs for side-effect verification in `~/.cache/claude-cli-nodejs/.../mcp-logs-<name>/*.jsonl` |
| 5 | Test plan OR AC contains "Telegram", "gh issue create", "API call", external URLs, `curl` with a token | Mock/dry-run requirement: ACs + test plan need a `DRY_RUN=1` env flag or a sandbox target. Cost warning in the preview. |
| 6 | Test plan contains `docker compose up`, `systemctl restart`, "deploy", "live", "API call" | Cleanup requirement: ACs for a cleanup statement in the Tester (service teardown, test-data purge, original state restored) |

### Skill reaction to triggers

1. In the spec dialog before the final preview the skill prints **detected triggers as a list with reasoning** (e.g., `"Trigger #3 HG3: detected 'ANTHROPIC_API_KEY' in spec block"`).
2. Per trigger, **AC items are automatically suggested** — user: `APPROVE` / `EDIT` / `DISMISS`.
3. `DISMISS` writes a `## Standards Notes` block into the issue with reasoning (e.g., "HG3 dismissed: token will be introduced via a separate issue #N").

### Example flow: trigger #3 (HG3 bearer-token detection)

**Sample spec block:**
> The skill uses `ANTHROPIC_API_KEY` as an env var for Anthropic SDK calls. The token is loaded from `.env` and injected into the docker-compose stack.

**Detected pattern:** `ANTHROPIC_API_KEY` matches regex `[A-Z_]+_(KEY|TOKEN|SECRET|PAT)`.

**Generated AC items (suggested):**
- [ ] `.env.example` has `ANTHROPIC_API_KEY` with scope comment (which service reads it, which permissions)
- [ ] `docker-compose.yml` `env_file:` references `.env`
- [ ] entrypoint preflight: on missing token warn + skip registration (no hard crash)
- [ ] `git diff --cached` secret scan before commit greps for token pattern (no real value in the diff)

The user decides per AC item: `APPROVE` / `EDIT` / `DISMISS`. If all four items are `DISMISS`ed, a `## Standards Notes` block with the reasoning is appended to the issue.

## Refine mode v3.2.0

`/create-issue --refine <num>` loads an existing issue and fills in missing sections. For the case where `/work-issue` returned STOP because the spec was incomplete. Workflow (5 steps since v3.2.0):

1. **Load the existing body** — `gh issue view <num> --repo <slug> --json title,body,labels,milestone`
2. **Run type detection on the existing body** — check all 6 triggers, identify which fire.
3. **AGENTS.md `ac_templates` diff** — which templates are missing in the current AC block? → required add (or standards-override block if justified).
4. **Trigger diff** — which triggers fire whose required ACs are missing in the existing body? → required add.
5. **User diff approval** — the skill shows **only the changes** (not the whole body); the user APPROVES / REJECTS per item. Approved items are integrated into the body via `gh issue edit <num>`.

This way refine mode catches exactly the spec-gap pattern that previously left the producer side too lenient.

## Scope boundaries vs. /work-issue

| Skill | When |
|-------|------|
| `/create-issue` | Idea exists, no issue yet. OR issue exists without spec → `--refine` |
| `/work-issue` | Issue with spec → build loop until merge |

## What this skill does NOT do

- Does not write code (only the issue body)
- Does not create branches
- Does not resolve cross-repo dependencies (depends-on is inserted only as a reference, not resolved)

## See also

- `skills/init-agents/SKILL.md` — bootstrap phase (create AGENTS.md, mandatory before the first run)
- `skills/work-issue/SKILL.md` — execution phase
- `skills/work-issue/references/repo-registry.yaml.example`
- `skills/init-agents/references/AGENTS.md.template`
- `skills/create-issue/references/issue-templates/code.md` — code-loop template
- `skills/create-issue/references/issue-templates/research.md` — research-loop template
- `skills/work-issue/references/subagent-briefs/code-implementer.md` — code-implementer brief
- `skills/work-issue/references/subagent-briefs/research-implementer.md` — researcher brief
