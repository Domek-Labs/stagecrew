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
  default_oos:
    - "No code generation (this repo is Markdown-only)"
    - "No migration of older skill versions without explicit user approval"
  ac_templates:
    - "CLAUDE.md at the repo root reflects the change"
    - "SKILL.md has valid YAML frontmatter"
    - ".claude-plugin/plugin.json is consistent (version, name)"
loop_types:
  enabled: [code, research]
  default: code
  type_overrides:
    research:
      ac_templates: []
      smoke_test: "true"
    # Roadmap (v3.4.0+):
    # text:
    #   ac_templates: ["markdownlint passes", "linkcheck green"]
    # decision:
    #   ac_templates: ["ADR format correct", "constraint coverage complete"]
    # diagnostic:
    #   ac_templates: ["hypotheses reproducible", "fix spec clear"]
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

## Loop Types (v3.3.0 multi-type system)

This skill set supports multiple issue types since v3.3.0:
- **`code`** (default) — software implementation tasks
- **`research`** — knowledge generation with a test matrix

Battle-tested in:
- Code loops: 13 PRs in an internal bot project (2026-06-26)
- Research loop: first end-to-end run in an internal bot project (2026-06-26)

Roadmap for `text`, `decision`, `diagnostic` types — see repo issues with labels `roadmap` + `loop-type`.

## Hard-Gates Detail

1. **No direct edits on main** — PR flow mandatory, branch protection should be enabled (if not: enable it).
2. **No new secrets in repo** — critical because a public flip is planned later.
3. **SKILL.md YAML frontmatter** — the loader is tolerant but strict-YAML compliance is mandatory (otherwise CI breaks against future validator tooling).
4. **Plugin cache reload after skill changes** — `claude restart` is mandatory in every PR body.
5. **plugin.json version bump in sync** — the plugin cache path uses the version, so an asynchronous bump leaves a stale cache.

## Plugin-Loader Gotchas

- Cache path: `~/.claude/plugins/cache/loop-engineering-workflow/<version>/skills/<name>/SKILL.md`
- Marketplace source: can be a local path (development) or a GitHub URL (public production)
- On a `plugin.json` version bump, the old cache path stays and a new path is created — `claude restart` is required.
