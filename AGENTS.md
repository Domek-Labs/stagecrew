---
work-issue:
  branch_pattern: "feature/<short>"
  default_branch: main
  pr_base: main
  commit_format: conventional
  syntax_check: "true"
  smoke_test: "true"
  deploy_command: ""
  linter: ""
  hard_gates:
    - "No direct edits on main"
    - "No new secrets in repo"
    - "SKILL.md requires YAML frontmatter (name/description/model)"
    - "Plugin cache reload (claude restart) after SKILL.md changes — otherwise stale version stays active"
    - "On version bump, .claude-plugin/plugin.json must stay in sync with the plugin cache path"
    - "Version bump follows version_policy (patch / minor / major) — see version_policy field"
    - "English-only for all new writes: issue titles and bodies, PR titles and bodies, commit messages, code comments, and docs. Repo went public on 2026-07-01. Existing German content is left in place until it is re-touched; every new write is English."
  default_oos:
    - "No code generation (this repo is Markdown-only)"
    - "No migration of older skill versions without explicit user approval"
  ac_templates:
    - "CLAUDE.md at the repo root reflects the change"
    - "SKILL.md has valid YAML frontmatter"
    - ".claude-plugin/plugin.json is consistent (version, name)"
    - "If plugin.json version changed, the bump matches version_policy and no other file names an explicit version number"
version_policy:
  # Single source of truth for the version is .claude-plugin/plugin.json.
  # No other file in the repo (README, CLAUDE.md, AGENTS.md body, SKILL.md,
  # subagent-briefs, templates) may name an explicit vX.Y.Z number. Roadmap
  # sections describe *what* is planned, never *when* (no version-tagged
  # milestones). This keeps the version drift-free.
  source_of_truth: ".claude-plugin/plugin.json"
  scheme: semver
  # Current status is 0.x (Alpha). Under 0.x the usual semver rules are
  # inverted for the leading zero: minor bumps *may* be breaking, patch bumps
  # *should not* be. Explicit rules per bump type:
  patch: |
    Wording, typos, formatting, README/CLAUDE polish, non-behavioral internal
    docs, adding examples without changing skill logic, dependency-free chores.
    Everything the plugin's runtime behavior does NOT depend on.
  minor: |
    New skill, new loop-type, new subagent-brief, new AGENTS.md field, changed
    skill behavior that stays backwards-compatible for existing AGENTS.md
    files, new required frontmatter field with a safe default. Under 0.x this
    is the "normal feature" bump and MAY break edge-case setups.
  major: |
    Removed skill, removed loop-type, removed AGENTS.md field, changed default
    behavior of a skill in a way that breaks callers, hard schema migration
    required. Under 0.x a major bump still moves the leading zero — treat as
    "will require every consumer to reinstall + reconfigure".
  first_stable_bump: |
    Move from 0.x to 1.0.0 only when: (a) the 5-stage skeleton has been
    stable for 3+ months without a major bump, (b) all shipped skills have
    at least one battle-tested end-to-end run per loop-type, (c) the AGENTS.md
    schema has a documented compat guarantee.
loop_types:
  enabled: [code, research]
  default: code
  type_overrides:
    research:
      ac_templates: []
      smoke_test: "true"
    # Additional loop types (text / decision / diagnostic) are on the roadmap
    # — see repo issues labelled `roadmap` + `loop-type`. Version-tagged
    # milestones are intentionally NOT listed here (see version_policy).
---

# Agent Instructions for loop-engineering-workflow

Per-repo coding-standards spec for AI-assisted workflows. Single source of truth for this plugin repo itself (dogfooding — we use `/work-issue` to evolve `/work-issue`).

## Architecture

Single-plugin repo for the `loop-engineering-workflow` Claude Code plugin.

- `.claude-plugin/plugin.json` — plugin manifest (name, version)
- `CLAUDE.md` — plugin documentation
- `README.md` — public-audience tagline + quickstart
- `skills/`
  - `loop/` — umbrella skill (router to init-agents / create-issue / work-issue)
  - `init-agents/` — bootstrap AGENTS.md for a repo
  - `create-issue/` — idea → GitHub issue with spec standard
  - `work-issue/` — issue → merged PR (5-stage spec→build loop)

## Code Style / Conventions

- The repo is **Markdown-only**. No TS/JS/Python.
- SKILL.md **must** carry a YAML frontmatter with the required fields `name`, `description`, optional `model`, `allowed-tools`.
- Description field: lowercase, comma-separated trigger words (no inline `Trigger:` prefixes — they break strict YAML).
- Commit format: conventional (`feat(skill): ...`, `fix(create-issue): ...`, `docs: ...`).
- Branch pattern: `feature/<short>` or `fix/<short>`.
- **No inline version numbers.** Only `.claude-plugin/plugin.json` names the current version. All other files describe what a feature does, not when it landed.
- **English-only for new writes.** See the "Language" section below.

## Language

**English-only for all new writes** — issue titles and bodies, PR titles and bodies, commit messages, code comments, docs. The repo went public on 2026-07-01 (PR #16 also landed the version-policy reset alongside the flip); at that point English became the only acceptable target for new content.

Practical rules:

- **Writing new content:** English. Even if the requesting user speaks German. Explain the reasoning briefly if the user asks — the repo is public, so all artifacts have to read to outside contributors.
- **Editing existing content:** if the file is already German (a handful of ADRs, some inline `.md` notes), translate the touched sections to English when you edit them. Leave the untouched sections as they are for now — a bulk German-to-English translation is out of scope for a single loop, but every diff nudges the repo further toward English-only.
- **User conversation stays as the user prefers.** This rule is about *the artifacts that end up in the public repo* (issues, PRs, commits, docs). Chat with the requesting user stays in whatever language they wrote in.
- **Validator responsibility:** the `/work-issue` Validator treats a German issue title or body as a STOP with a hint to `/create-issue --refine` in English, unless the requesting user explicitly opts out for a specific issue via the `## Standards Override` block.

## Loop Types

This skill set supports multiple issue types:
- **`code`** (default) — software implementation tasks
- **`research`** — knowledge generation with a test matrix

Battle-tested in an internal bot project — code loops (13 PRs, 2026-06-26) and research loops (first end-to-end run, 2026-06-26).

Roadmap for `text`, `decision`, `diagnostic` types — see repo issues with labels `roadmap` + `loop-type`.

## Hard-Gates Detail

1. **No direct edits on main** — PR flow mandatory, branch protection should be enabled (if not: enable it).
2. **No new secrets in repo** — the repo is public; a leaked token would be publicly indexed within minutes.
3. **SKILL.md YAML frontmatter** — the loader is tolerant but strict-YAML compliance is mandatory (otherwise CI breaks against future validator tooling).
4. **Plugin cache reload after skill changes** — `claude restart` is mandatory in every PR body.
5. **plugin.json version bump in sync** — the plugin cache path uses the version, so an asynchronous bump leaves a stale cache.
6. **Version bump follows version_policy** — see the `version_policy` field in the YAML frontmatter. No other file may name an explicit version number; when in doubt, ship as `patch`.
7. **English-only for new writes** — see the `Language` section above. The Validator STOPs on a German issue title or body unless the issue's `## Standards Override` block explicitly waives this gate.

## Plugin-Loader Gotchas

- Cache path: `~/.claude/plugins/cache/loop-engineering-workflow/<version>/skills/<name>/SKILL.md`
- Marketplace source: can be a local path (development) or a GitHub URL (public production)
- On a `plugin.json` version bump, the old cache path stays and a new path is created — `claude restart` is required.

## Component Registry (optional AGENTS.md field)

Component drift across sessions and users is a real cost: the same UI concept (a data table, a navigation menu) gets re-implemented slightly differently every time an issue lands. When a decision changes ("from now on: tabs, not submenus"), a 20-file refactor becomes the cheapest exit — because there was never one canonical component to change. The same trap exists on the backend for domain entities, value objects and query handlers.

The loop can prevent this if a repo declares an **optional `components:` block** in the AGENTS.md YAML frontmatter. Absent block = zero behavior change. Opt-in per Pure-Reader principle.

### Schema

```yaml
components:
  registry_path: "docs/components.md"           # required if block present — the Markdown catalog
  code_globs:                                   # required — points at the code source-of-truth
    - "src/components/**/*.tsx"
    - "src/domain/**/*.ts"
  usage_policy: "prefer_existing"               # prefer_existing (soft, warns) | strict (hard, ADR required for new)
  scope: "both"                                 # frontend | backend | both (default both)
```

### Two layers

1. **Code layer** — the real reusable components in the codebase (single source of truth for API / props / types / backend contract). Referenced via `code_globs`.
2. **Markdown catalog layer** — a per-component MD (rooted at `registry_path`) describing when to use, when not to, where the code lives, the public API summary, the backend contract shape (if applicable), and anti-patterns.

### Registry MD structure (per component)

- Name
- Purpose (when to use, when not to)
- Code path (link into `code_globs`, the source of truth)
- Public API summary
- Backend contract shape (if applicable — e.g. table filter/sort response shape)
- Anti-patterns

A skeleton template with a worked frontend example and a worked backend example ships in `skills/init-agents/references/components-registry-template.md`.

### `usage_policy` modes

- **`prefer_existing`** (default when block present, soft): the Validator warns if an in-scope issue does not reference an existing registry component; the Critic still runs dupe detection. Non-blocking.
- **`strict`** (hard): the Validator STOPs an in-scope issue unless it references an existing registry component OR links an ADR path in the `## Standards Override` block that justifies adding a new one. Implementer + Critic keep the same hard-gate.

### `scope` values

- **`frontend`** — enforcement applies only to `code_globs` matching frontend patterns.
- **`backend`** — enforcement applies only to backend patterns (domain entities, value objects, aggregates).
- **`both`** (default) — enforcement applies to any file in `code_globs`.

### Loop integration

- **Validator (stage 1):** if the issue's `Spec` or `Files to Touch` implies work in the `code_globs` scope, gate as above (warn in `prefer_existing`, STOP + ADR requirement in `strict`).
- **Implementer brief:** hard-gate — "If a registry component fits the case, use it. Do not inline a duplicate. If nothing fits, stop and propose an ADR before implementing."
- **Critic (stage 4):** dupe-detection pass — `search_code` / grep across `code_globs` for repeated patterns; if two implementations cover the same concept, log as a REVISE item.

### For this repo

This repo is **Markdown-only** — no frontend framework, no backend domain layer, no `code_globs` scope. The `components:` block is therefore not set here. This section documents the schema so `/init-agents`, `/create-issue` and `/work-issue` can honor it in downstream target repos.

## Visual Reviewer Gate (optional AGENTS.md field)

Frontend issues pass the loop on a green build alone — nobody looks at the rendered UI. A build can succeed while the page overflows on mobile, pushes a primary action below the fold, or throws a console error no unit test catches. The loop can close that gap if a repo declares an **optional `visual:` block** in the AGENTS.md YAML frontmatter. Absent block = zero behavior change. Opt-in per the Pure-Reader principle.

This is a **gate, not a loop-type**. Frontend work still ships a code diff, so it keeps the `code` implementer / Tester / Critic / Closer — the visual review is an extra stage layered on the same axis as `components:`, not a fork of the code pipeline. Design rationale in `docs/adr/0002-visual-gate.md`.

### Schema

```yaml
visual:
  serve_command: "npm run preview"     # required if block present — how to serve the built frontend
  base_url: "http://localhost:4173"    # required — where the served app answers
  viewports:                           # optional; default [390x844 mobile, 1280x800 desktop]
    - { w: 390,  h: 844,  label: mobile }
    - { w: 1280, h: 800,  label: desktop }
  routes: ["/"]                        # optional default routes; the issue can override/extend
  console_error_policy: "fail"         # fail | warn (default fail)
  screenshot_dir: ".stagecrew/visual"  # optional — where screenshots are saved before upload
  scope: "frontend"                    # visual review only applies to frontend work
```

### Trigger

The Visual Reviewer stage fires only when **both** hold: the repo has a `visual:` block, and the issue is frontend-scoped (files-to-touch match a frontend glob OR the Spec/AC contain a frontend keyword: `page`, `route`, `UI`, `component`, `layout`, `responsive`, `screen`, `viewport`). Otherwise it is skipped silently.

### Loop integration

- **`/create-issue`** injects a `### Visual Acceptance (AGENTS.md visual:)` AC block for in-scope issues.
- **`/work-issue` stage 3.5 (Visual Reviewer)** — between Tester and Critic — serves the built frontend, drives Playwright per route × viewport (mobile first), captures screenshots + a11y snapshots + console output, and posts `## [stage:visual] PASS|FAIL` with screenshots attached. FAIL routes back to the Implementer under the shared 3-revise cap.
- Requires the Playwright companion MCP; absent namespace → the stage is skipped with a note, never a hard fail.

### For this repo

This repo is **Markdown-only** — no frontend, no `serve_command`. The `visual:` block is therefore not set here. This section documents the schema so `/init-agents`, `/create-issue` and `/work-issue` honor it in downstream frontend repos.
