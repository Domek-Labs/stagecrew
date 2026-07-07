# loop-engineering-workflow

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
3. /work-issue <num>                      # issue → merged PR via 5 stages
```

That is the entire loop. Bootstrap → Genesis → Execution.

## The 3 Skills

| Skill | Phase | What it does |
|-------|-------|--------------|
| `/init-agents` | Bootstrap | Writes `AGENTS.md` at the repo root: YAML frontmatter with 11 standards fields (branch pattern, syntax check, smoke test, hard gates, AC templates, ...) + Markdown body (architecture, conventions, known gotchas). One-time per repo. Codebase-memory provides intelligent defaults. |
| `/create-issue` | Genesis | Turns an idea into a fully-specified GitHub issue with spec-standard sections (Idea/Spec/AC/Files-To-Touch/Test-Plan/Out-of-Scope). Auto-injects AGENTS.md's `ac_templates`. Calls `/init-agents` as a pre-step if AGENTS.md is missing. |
| `/work-issue` | Execution | Drives a specified issue through 5 stages (Validator → Implementer → Tester → Critic → Closer) until merge. Pure-reader: all standards come from AGENTS.md. Auto-PR, auto-merge on APPROVE. Issue comments serve as the audit log. |

## Umbrella Skill: `/loop`

If you do not know which sub-skill you need, just call `/loop` — it asks one clarifying question and routes you. Triggers on "loop", "loop engineering", "work issue", "spec build", "coding loop", "github workflow".

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

Battle-tested in this repo: code loops (#6 docs/translation, #7 chore/audit).
Research loops: infrastructure activated in #10, first end-to-end battle-test
ready as a follow-up issue (pick a research topic and call `/work-issue --type=research`).

## Optional features

### Component Registry (`components:` block in AGENTS.md)

An **opt-in** AGENTS.md YAML frontmatter block that declares a repo's canonical component set (frontend components, backend value objects, aggregates). When set, the loop enforces reuse: the Validator gates in-scope issues against the registry, the Implementer refuses to inline duplicates, and the Critic runs a dupe-detection pass over the declared code globs. `usage_policy` picks the enforcement level — `prefer_existing` warns, `strict` STOPs and requires an ADR link.

Absent block → zero behavior change. Full schema and enforcement details in `AGENTS.md` under "Component Registry", template in `skills/init-agents/references/components-registry-template.md`, design rationale in `docs/adr/0001-components-registry.md`.

## Recommended companion MCPs (optional)

Two MIT-licensed external MCPs make the loop richer. Neither is required — the skills call them opportunistically and fall back to plain `grep` / `ls` / no-op when the tool namespace is missing.

- **[codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp)** — local code-graph MCP (tree-sitter based). Used by `/init-agents` for architecture / conventions defaults, `/create-issue` for files-to-touch suggestions, and `/work-issue` in the Validator / Implementer / Critic stages for code-graph-backed evidence.
- **[MemPalace](https://github.com/MemPalace/mempalace)** — verbatim knowledge-store MCP. The `/work-issue` Closer stage persists a loop summary (repo, PR, commit, Tester findings) as a "drawer" so future sessions can search prior decisions.

## Roadmap

See Epic #1 in this repo for the rolling roadmap.

## License

MIT — see [LICENSE](LICENSE).

## Status

Alpha. In active development. Under `0.x` the API may shift between minor bumps — see `AGENTS.md` `version_policy` for the exact patch / minor / major definition. The current version lives only in `.claude-plugin/plugin.json`.
