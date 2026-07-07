---
name: init-agents
description: "Create AGENTS.md (per-repo coding-standards spec) for a Git repo. YAML frontmatter with 11 v3 fields (branch_pattern, default_branch, pr_base, commit_format, syntax_check, smoke_test, deploy_command, linter, hard_gates, default_oos, ac_templates) + Markdown body (architecture, code conventions, test conventions, known gotchas). codebase-memory provides intelligent defaults from the repo cluster. Mandatory bootstrap before the first /work-issue run in a repo. Triggers: /init-agents, init agents, create agents.md, coding spec bootstrap, initialize repo standards."
---

# /init-agents — Create AGENTS.md for a repo

**Type:** bootstrap / standards-spec engineering

## Purpose

AGENTS.md is the **single source of truth** for per-repo coding standards. `/work-issue` is a **pure-reader** (no hardcoded defaults): all standards values come from AGENTS.md at the repo root.

That makes AGENTS.md **mandatory** before `/work-issue` can run against a repo. `/init-agents` creates it — interactively (the user is walked through every field) or autonomously (codebase-memory supplies sensible defaults).

## Invocation variants

```
/init-agents                                    # cwd repo, interactive dialog
/init-agents --repo <slug>                      # specific repo, interactive (default)
/init-agents --repo <slug> --interactive        # explicit interactive
/init-agents --repo <slug> --auto               # autonomous with codebase-memory defaults
/init-agents --refine [--repo <slug>]           # extend an existing AGENTS.md with missing fields
```

**Argument parsing:** same as `/work-issue` and `/create-issue` (GitHub syntax, `--repo` flag, cwd fallback).

## Pre-Flight

Run in this order — on the first failure, ABORT with a clear hint message.

1. **Repo-path lookup** in `~/.claude/work-issue-paths.yaml`.
   - If the live file is missing: `cp` from `references/repo-registry.yaml.example` (work-issue) as a bootstrap, then ask for `repo_path`.
   - If the legacy path `~/.claude/skills/work-issue/repo-paths.yaml` exists: print a migration hint (move it once), but keep working against the live path.
2. **AGENTS.md existence check** at the repo root (`<repo_path>/AGENTS.md`):
   - Present AND no `--refine`: ABORT.
     > AGENTS.md already exists in `<repo_path>`. Call `/init-agents --refine --repo <slug>` to extend it, or delete it manually first.
   - Present WITH `--refine`: parse the frontmatter, identify empty/missing fields.
   - Missing AND `--refine`: print the hint "AGENTS.md missing — running as a regular init", then proceed like `--interactive`/`--auto`.
3. **codebase-memory pre-flight** (same as `/work-issue`):
   - `list_projects` — check whether `<repo>` is indexed.
   - If not: `index_repository` with mode `moderate`.
   - If indexed: `index_status` — freshness check (>7 days AND a newer commit → re-index).
   - Health check: `nodes < 200` OR `source_files < 3` → warn "default heuristic may have excluded dirs (bin/, docs/, scripts/) — re-index with explicit paths recommended before running `/init-agents --auto`".

## Dialog workflow (--interactive, default)

The user is walked through all 11 v3 frontmatter fields. For each field, show a concrete default suggestion from codebase-memory + heuristic — the user accepts or overrides.

### 11 v3 frontmatter fields

| Field | Default source / heuristic |
|-------|----------------------------|
| `branch_pattern` | `feature/<scope>-<short>` (standard); derive from the last 20 branch names if a pattern is recognizable (`git branch -r`) |
| `default_branch` | `gh api repos/<slug> --jq .default_branch` |
| `pr_base` | = `default_branch`; if a `dev` branch exists (`gh api repos/<slug>/branches/dev`): suggest `dev` (do not set autonomously) |
| `commit_format` | `conventional` (count the last 30 commits: % matching `type(scope): description` — if >70% → `conventional`, otherwise `custom`) |
| `syntax_check` | derive from the dominant language via `get_architecture`: TypeScript → `tsc --noEmit`, JavaScript → `node --check <file>`, Python → `python -m py_compile <file>`, otherwise empty |
| `smoke_test` | from `docker-compose.yml` (if present) → `docker compose build && docker compose up -d --force-recreate`. Otherwise from `package.json scripts.test` or `pyproject.toml scripts.test`. Otherwise empty |
| `deploy_command` | empty (user decides) |
| `linter` | from the repo: detect `eslint`/`ruff`/`black` configs |
| `hard_gates` | list, defaults: "No direct edits on default_branch", "No new secrets in repo", "All MCP tools need a test example in their doc block" |
| `default_oos` | list, empty by default — the user adds repo-specific entries |
| `ac_templates` | list, defaults: "Code passes <syntax_check>", "Documentation in README/CLAUDE.md updated", "No new secrets in repo (secret check passed)" |

### Optional field: `components:` (component-registry, opt-in)

Between the 11 required frontmatter fields and the 4 body sections, run a **component-pattern probe**. Purpose: propose an opt-in `components:` block only when the repo shows real reusable-component patterns; skip cleanly otherwise.

**Detection signals (via codebase-memory + file lookup):**

- **Frontend framework presence** — any `.tsx`, `.jsx`, `.vue`, or `.svelte` file in the repo (`get_architecture` + `search_code` for `*.tsx`/`*.jsx`/`*.vue`/`*.svelte`).
- **Backend domain layer** — any path under `src/domain/**`, `src/entities/**`, or `src/aggregates/**` (`get_architecture` cluster names + a directory-existence probe).

**Dialog behavior:**

- **No signal fires** — skip the `components:` block silently. AGENTS.md ships without it. Zero behavior change downstream.
- **Frontend signal fires** — propose a `components:` block with `scope: frontend`, `registry_path: "docs/components.md"`, `code_globs` seeded with the detected framework glob (e.g. `["src/components/**/*.tsx"]`), and `usage_policy: prefer_existing`.
- **Backend signal fires** — propose with `scope: backend`, `code_globs` seeded with the detected domain path (e.g. `["src/domain/**/*.ts"]`).
- **Both fire** — propose with `scope: both` and both globs.

In every proposed case: also offer to copy `references/components-registry-template.md` to the resolved `registry_path` as a starting skeleton (worked frontend + backend example). User: APPROVE / EDIT / SKIP. `SKIP` = block omitted, no registry file written.

Rationale + full schema: see `AGENTS.md` in this plugin repo under "Component Registry".

### 4 body sections

For each section, show 1-3 suggestions from codebase-memory or cluster analysis. The user accepts or writes their own.

| Section | Default source |
|---------|----------------|
| **Architecture (short)** | `get_architecture` → top 3 clusters + main components as a suggestion |
| **Code style / conventions** | `search_code` for dominant naming patterns + lint config findings |
| **Test conventions** | test-dir detection (`test/`, `tests/`, `__tests__/`) + dominant test framework from `package.json`/`pyproject.toml` |
| **Known gotchas** | empty initially — the user fills it in later (skill recommendation: "empty section is fine, add to it after the first build") |

## Auto workflow (--auto)

Run through the full default heuristic — no user dialog. The output is a preview; the user gets a YES/NO prompt at the end.

Defaults:
- **branch_pattern**: `feature/<short>` (safe default, no autonomous pattern detection)
- **default_branch**: via `gh api repos/<slug> --jq .default_branch`
- **pr_base**: = `default_branch` (never autonomously switch to `dev`, even if `dev` exists — only suggested in the interactive dialog)
- **commit_format**: `conventional` as default (safe default)
- **syntax_check**: from the dominant language via cluster analysis
- **smoke_test**: from `docker-compose.yml` / `package.json` / `pyproject.toml`
- **deploy_command**: empty
- **linter**: from detected config files
- **hard_gates**: 3 standard gates (see above)
- **default_oos**: empty
- **ac_templates**: 3 standard templates (see above)
- **body sections**: populated from `get_architecture` + `search_code`; known gotchas empty

## Refine workflow (--refine)

1. Read the existing `<repo_path>/AGENTS.md`.
2. Parse the YAML frontmatter (`python3 -c "import yaml; print(yaml.safe_load(open('<path>').read().split('---')[1]))"`).
3. For each field, check: empty/missing? → add to the refine list.
4. Dialog ONLY for fields in the refine list — others stay untouched.
5. Body sections similarly: empty/missing → dialog, otherwise untouched.
6. Show the diff, user APPROVES/CANCELS.

## Output steps (after dialog/auto/refine)

1. **Preview** — render the full AGENTS.md content (frontmatter + body). User: APPROVE / REVISE / CANCEL.
2. **Write AGENTS.md** to `<repo_path>/AGENTS.md`.
3. **Extend README.md** with a "For contributors / AI agents" section, if not already present:
   ```markdown
   ## For Contributors / AI Agents
   This repo uses `AGENTS.md` as the standards spec for AI-assisted
   workflows (`/work-issue`, `/create-issue`). Change conventions
   there, not inline in code review.
   ```
4. **Branch + commit + PR + merge** (same as `/work-issue` Implementer/Closer):
   - Branch: `feature/init-agents-md` (or per `branch_pattern`)
   - Commit message: `chore(agents): add AGENTS.md (init via /init-agents)` with `Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>`
   - PR body: short, refer to the AGENTS.md format
   - Squash-merge (delete-branch)
5. **Repo-registry update (mandatory step)** — live file `~/.claude/work-issue-paths.yaml`:
   ```bash
   yq -i '.["<owner>/<repo>"].agents_md_exists = true' ~/.claude/work-issue-paths.yaml
   # Fallback without yq:
   # sed edit or Python update on the same live file
   ```
   If the entry does not yet exist: create a minimal one first (`repo_path` from the pre-flight is known).
6. **Channel reply** (if invoked from a `<channel>` inbound): "AGENTS.md created for `<slug>` (PR #N merged). The repo is ready for `/work-issue`."

## Known limitation (plugin-cache freshness)

When `/init-agents` first becomes available after a plugin update, the plugin cache has to be fresh BEFORE the first call (`claude restart`).

So the recommended sequence for a bootstrap run is:
1. Merge the PR that adds/updates the plugin (`feature/init-agents` → `main`)
2. `claude restart` (fresh plugin cache)
3. `/init-agents --repo your-org/your-repo --interactive` as the bootstrap run.

Without the restart in step 2, the running Claude still sees the previous skill versions — so neither the new `/init-agents` nor the updated `/work-issue` (pure-reader) is visible.

The same applies to any other repo: every time a repo gets its AGENTS.md for the first time, a `claude restart` should happen before the next `/work-issue` run in that repo (so the pure-reader sees the fresh file). In practice, a session reload is usually enough.

## AGENTS.md format reference

Template file: `references/AGENTS.md.template` (in this skill).

Schema (abbreviated):

```yaml
---
work-issue:
  branch_pattern: "feature/<scope>-<short>"
  default_branch: main | dev
  pr_base: main | dev
  commit_format: "conventional" | "custom"
  syntax_check: "<command>"
  smoke_test: "<command>"
  deploy_command: "<command>"
  linter: "<command>"
  hard_gates: ["..."]
  default_oos: ["..."]
  ac_templates: ["..."]
---

# Agent Instructions for <repo-name>

## Architecture (short)
## Code Style
## Test Conventions
## Known Gotchas
```

Full example: see `references/AGENTS.md.template`.

## Scope boundaries

| Skill | When |
|-------|------|
| `/init-agents` | Repo has no AGENTS.md yet → mandatory bootstrap BEFORE the first `/work-issue` run |
| `/create-issue` | Idea → specified GitHub issue. Calls `/init-agents` as a pre-step if AGENTS.md is missing |
| `/work-issue` | Pure-reader: needs AGENTS.md at the repo root. Without AGENTS.md → STOP verdict with a pointer to `/init-agents` |

`/init-agents` and `/create-issue` are complementary:
- `/init-agents` captures **standards** (once per repo)
- `/create-issue` captures **issues** (continuous, per feature/bug)

`/init-agents` and `/work-issue` are sequential:
- `/init-agents` is the **mandatory bootstrap** before the first `/work-issue` run in a repo.

## What this skill does NOT do

- **No migration from legacy `code/projects.yaml`** — that path has been removed; old data is lost.
- **No schema versioning** (e.g., a `schema_version: 1` field). Planned for a future release if a migration is needed.
- **No auto-detection of a dev-branch flow** in `--auto` mode — if a `dev` branch exists, it is **suggested** in the interactive dialog but never set autonomously as `pr_base`.
- **No AGENTS.md for non-code repos** (e.g., docs-only repos). The skill assumes a code repo.
- **No cross-repo standards inheritance** — every repo has its own AGENTS.md.

## See also

- `references/AGENTS.md.template` — full example + schema
- `skills/work-issue/SKILL.md` — pure-reader loop that reads AGENTS.md
- `skills/create-issue/SKILL.md` — issue genesis, calls `/init-agents` as a pre-step
- `skills/work-issue/references/repo-registry.yaml.example` — repo-registry template
