# loop-engineering-workflow

Loop-engineering workflow primitives — turn a GitHub issue into a merged PR (or research findings) with a 5-stage agent pipeline.

Pure-reader: standards live in your repo's `AGENTS.md`, not in this plugin.

## Why

Writing code with AI agents works best when every issue carries its own spec, every repo carries its own conventions, and every loop has a small, fixed set of stages that can be audited after the fact. This plugin gives you those three things — a way to bootstrap repo standards, a way to specify an issue, and a way to drive that issue through to merge — without baking any opinions into the plugin itself.

## Status

Alpha — in active development. The current version lives in `.claude-plugin/plugin.json`; the per-bump policy (patch / minor / major) is documented in `AGENTS.md` under `version_policy`. Under `0.x` a minor bump may introduce breaking changes.

## Install

```
claude plugin marketplace add https://github.com/scheineckerdominik-rgb/loop-engineering-workflow
claude plugin install loop-engineering-workflow
```

After install, restart Claude so the plugin cache picks up the new skills.

## Skills

- `/init-agents` — bootstrap `AGENTS.md` per repo (one-time)
- `/create-issue` — idea → fully-specified GitHub issue
- `/work-issue` — issue → merged PR via Validator → Implementer → Tester → Critic → Closer
- `/loop` — umbrella router for the three above

See `CLAUDE.md` for the quickstart.

## Optional component registry

Repos with real component patterns (a frontend framework, a backend domain layer, or both) can opt into a canonical component registry via a `components:` block in AGENTS.md. When set, the loop enforces reuse across sessions: Validator gates in-scope issues, Implementer refuses inline duplicates, Critic scans for dupes. `usage_policy` picks the enforcement level (`prefer_existing` warns, `strict` STOPs and requires an ADR link). Absent block = zero behavior change. See `AGENTS.md` under "Component Registry" and `docs/adr/0001-components-registry.md` for the design.

## Loop types

The plugin supports two issue types via the `loop-type:<type>` label:

- **`loop-type:code`** — software-implementation loops (default). Template: `skills/create-issue/references/issue-templates/code.md`.
- **`loop-type:research`** — research / findings loops (knowledge generation, test-matrix studies, library comparisons). Template: `skills/create-issue/references/issue-templates/research.md`.

Use `/create-issue --type=<code|research> "<idea>"` to pick one explicitly, or let auto-inference handle it.

## Recommended companion MCPs (optional)

The plugin runs standalone — none of the skills require an external MCP server. But two MIT-licensed MCPs make the loop noticeably richer if they're wired into your Claude Code session. Both are opportunistically called: when the tool namespace is missing, the skills fall back to plain `grep` / `ls` / no-op.

- **[codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp)** — local code-graph MCP. Parses your repo with tree-sitter, exposes `list_projects`, `index_repository`, `index_status`, `search_code`, `get_architecture`, `detect_changes`, and more. `/init-agents` uses it to suggest architecture / conventions defaults, `/create-issue` uses it for files-to-touch suggestions plus hotspot warnings, and `/work-issue` calls it from the Validator, Implementer, and Critic stages for code-graph-backed evidence.

- **[MemPalace](https://github.com/MemPalace/mempalace)** — verbatim knowledge-store MCP. The `/work-issue` Closer stage can persist a loop summary (repo, PR, commit, Tester findings) as a "drawer" so future sessions can search for prior decisions. The Closer's persistence step is a no-op when MemPalace is not installed.

Neither is required. Install what you like, skip what you do not.

## Adapting to other workflows

Right now this plugin is built specifically around **GitHub issues**: an issue is the unit of work, the `gh` CLI is the interface, and a loop ends in a merged pull request. That coupling is a convention, not a hard requirement.

The underlying primitives — a per-issue spec standard, a per-repo `AGENTS.md`, and a small fixed set of audited stages — are tracker-agnostic. If your work lives somewhere else (Linear, Jira, a plain Markdown task file, an internal queue), you can rewrite the thin GitHub layer in the skills to read and write your own source of truth instead. The pipeline shape stays the same; only the issue/PR I/O changes.

PRs that generalize this layer are welcome.

## License

MIT — see [LICENSE](LICENSE).
