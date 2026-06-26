# loop-engineering-workflow

**Turn a GitHub issue into a merged PR (or research findings) with a 5-stage agent pipeline.**

Pure-reader plugin: all coding standards live in your repo's `AGENTS.md`, not in this plugin. You bootstrap once per repo, then any issue can flow through the loop.

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
| `/init-agents` | Bootstrap | Writes `AGENTS.md` at the repo root: YAML-frontmatter with 11 standards-fields (branch pattern, syntax check, smoke test, hard gates, AC templates, ...) + Markdown body (architecture, conventions, known gotchas). One-time per repo. Codebase-memory provides intelligent defaults. |
| `/create-issue` | Genesis | Turns an idea into a fully-specified GitHub issue with Spec-Standard sections (Idee/Spec/AC/Files-To-Touch/Test-Plan/Out-of-Scope). Auto-injects AGENTS.md's `ac_templates`. Calls `/init-agents` as pre-step if AGENTS.md is missing. |
| `/work-issue` | Execution | Drives a specified issue through 5 stages (Validator → Implementer → Tester → Critic → Closer) until merge. Pure-reader: all standards come from AGENTS.md. Auto-PR, auto-merge on APPROVE. Issue comments serve as audit log. |

## Umbrella Skill: `/loop`

If you do not know which sub-skill you need, just call `/loop` — it asks one clarifying question and routes you. Triggers on "loop", "loop engineering", "issue durchziehen", "spec build", "coding loop", "github workflow".

## Roadmap

- **v4.1.0 — Multi-Type-Loops** (Issue [#22](https://github.com/dscheinecker-at7media/dominik/issues/22)): the same 5-stage skeleton applied to non-code loops — Text-Loop, Decision-Loop, Diagnostic-Loop. Same Validator/Implementer/Tester/Critic/Closer pattern, different domain.
- **v4.2.0 — Public Flip** (Issue [#23](https://github.com/dscheinecker-at7media/dominik/issues/23)): LICENSE, secret sweep, public-ready repo rename.

## License

MIT (LICENSE file lands with Issue #23).

## Status

Alpha. In active development. API may shift between minor versions until v5.0.0.
