# loop-engineering-workflow

**Turn a GitHub issue into a merged PR (or research findings) with a 5-stage agent pipeline.**

Pure-reader plugin: all coding standards live in your repo's `AGENTS.md`, not in this plugin. You bootstrap once per repo, then any issue can flow through the loop.

**Multi-Type-System seit v4.1.0:** dasselbe 5-Stage-Skelett funktioniert sowohl fuer
Code-Loops (Software-Implementation) als auch fuer Research-Loops (Erkenntnis-Generierung
mit Test-Matrix). Pro Type ein eigenes Issue-Template + Subagent-Brief — siehe Sektion
"Loop-Types" unten.

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

## Loop-Types (v4.1.0)

| Type | Wann | Deliverable | Template | Implementer-Brief |
|------|------|-------------|----------|-------------------|
| `code` (Default) | Feature, Fix, Refactor, Dependency-Update | PR mit Code-Diff + Tests | `skills/create-issue/references/issue-templates/code.md` | `skills/work-issue/references/subagent-briefs/code-implementer.md` |
| `research` | Erkenntnis-Generierung, Test-Matrix-Studie, Bibliotheks-Vergleich, API-Verhalten-Probe | `docs/research/<topic>-<date>.md` + Folge-Issue-Spec | `skills/create-issue/references/issue-templates/research.md` | `skills/work-issue/references/subagent-briefs/research-implementer.md` |

Type-Auswahl:
- `/create-issue --type=<code|research> "<idea>"` — explizit
- Sonst Auto-Inference aus Keywords im User-Text (`feat`/`fix` -> code, `recherchiere`/`benchmark` -> research)
- Sonst `loop_types.default` aus AGENTS.md (typisch `code`)

`/work-issue` liest das `loop-type:<type>`-Label vom Issue und dispatched zum
passenden Subagent-Brief. Tester/Critic/Closer-Stages bleiben strukturell gleich,
pruefen aber type-spezifische Kriterien (Build-GREEN vs Doc-Quality, Code-Diff-Quality
vs Folge-Issue-Spec-Startbarkeit).

Erprobt in:
- Code-Loops: 13 PRs in `dscheinecker-at7media/personal-ai-bot` (2026-06-26)
- Research-Loop: `dscheinecker-at7media/personal-ai-bot#51` (2026-06-26)

## Roadmap

- **v4.1.0 — Multi-Type-Loops (Code + Research)** — Issue #2 in diesem Repo. **DONE** (mit diesem PR).
- **v4.1.1 — Public-Flip-Prerequisites** (Issue #3): LICENSE, secret sweep, README final, plugin.json prep-bump. **DONE** (mit diesem PR).
- **v4.2.0 — Public Visibility-Flip** (separate Session, hoher Blast-Radius).

## License

MIT, siehe [LICENSE](LICENSE).

## Status

Alpha. In active development. API may shift between minor versions until v5.0.0.
