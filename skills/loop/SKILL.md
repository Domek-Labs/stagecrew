---
name: loop
description: >
  Loop-engineering workflow umbrella. Routes to /init-agents (bootstrap AGENTS.md for a repo), /create-issue (idea → fully-specified GitHub issue), or /work-issue (issue → merged PR via 5-stage Validator → Implementer → Tester → Critic → Closer pipeline). Triggers: "loop", "loop engineering", "issue durchziehen", "spec build", "coding loop", "github workflow", "github loop", "spec build loop". Routes when intent is unclear.
---

# /loop — Loop-Engineering Workflow (Umbrella)

**Typ:** Umbrella / Router

## Zweck

Zentraler Einstiegspunkt fuer den Loop-Engineering-Workflow: GitHub-Issue → merged PR (oder Research-Findings) via 5-Stage-Agent-Pipeline.

Drei Sub-Skills decken die drei Phasen ab:

| Phase | Sub-Skill | Was | Direkter Trigger |
|-------|-----------|-----|------------------|
| **Bootstrap** | `/init-agents` | `AGENTS.md` fuer einen Repo anlegen (einmalig pro Repo, Pure-Reader-Voraussetzung) | "init agents", "agents.md anlegen", "coding spec bootstrap", "repo standards initialisieren" |
| **Genesis** | `/create-issue` | Idee → vollstaendig spezifiziertes GitHub-Issue (Spec-Standard mit Idee/AC/Files/Test-Plan/OoS) | "create issue", "neues issue", "feature anlegen", "spec issue", "idee als ticket" |
| **Execution** | `/work-issue` | Issue → merged PR via 5 Stages (Validator → Implementer → Tester → Critic → Closer) | "work issue", "issue durchziehen", "loop engineering", "spec build loop" |

## Workflow auf einen Blick

```
Bootstrap (once per repo)        Genesis (per feature/bug)       Execution (per issue)
       │                                 │                              │
       ▼                                 ▼                              ▼
  /init-agents          →          /create-issue          →         /work-issue
  (AGENTS.md)                      (GitHub issue)                   (5-Stage loop → PR merge)
```

Alle drei Sub-Skills sind **Pure-Reader** — sie lesen `AGENTS.md` im Repo-Root als Single-Source-of-Truth fuer Standards. Kein Skill-internes Defaults-Mapping.

## Typische Reihenfolge fuer neue Repos

```
1. /init-agents --repo <owner>/<slug>      # AGENTS.md anlegen (einmalig pro Repo)
2. /create-issue "<idee>"                  # erstes Issue spec'en
3. /work-issue <num>                       # Issue abarbeiten (Validator → Implementer → Tester → Critic → Closer bis Merge)
```

Fuer bestehende Repos mit AGENTS.md: Schritt 1 entfaellt — direkt mit `/create-issue` oder `/work-issue` starten.

## Ablauf wenn /loop direkt gerufen wird

Intent klaeren durch Rueckfrage, dann weiter-routen:

```
Was willst du machen?

  1) AGENTS.md fuer neuen Repo anlegen           → /init-agents
  2) Neue Idee als Issue anlegen                 → /create-issue
  3) Bestehendes Issue durchziehen (5-Stage)     → /work-issue
  4) Full-Loop fuer neuen Repo                   → /init-agents → /create-issue → /work-issue
```

## Wann welcher Skill

| Situation | Skill |
|-----------|-------|
| Neuer Repo, noch keine AGENTS.md | `/init-agents --repo <slug>` |
| Neue Idee, noch kein Issue (AGENTS.md existiert) | `/create-issue "..."` |
| Neue Idee, AGENTS.md fehlt | `/create-issue "..."` (ruft `/init-agents` als Pre-Step) |
| Issue existiert mit vollstaendiger Spec | `/work-issue <num>` |
| Issue existiert ohne Spec | `/create-issue --refine <num>` ODER manuell Spec ergaenzen, dann `/work-issue` |
| `/work-issue`s Validator hat STOP gegeben | `/create-issue --refine <num>`, dann `/work-issue` retry |
| `/work-issue` gibt STOP "AGENTS.md fehlt" | `/init-agents --repo <slug>`, dann `/work-issue` retry |
| AGENTS.md existiert, hat aber Luecken | `/init-agents --refine --repo <slug>` |

## Beispiel-Calls

```
# Neuer Repo, Full-Loop
/init-agents --repo myorg/myrepo
/create-issue --repo myorg/myrepo "add JWT refresh endpoint"
/work-issue 42 --repo myorg/myrepo

# Bestehendes Issue durchziehen
/work-issue 17

# Idee schnell als Issue
/create-issue "fix flaky test in checkout-flow.spec.ts"

# Unklarer Intent
/loop
# → Rueckfrage, dann Routing zu einem der drei Sub-Skills
```

## Warum Umbrella

Drei Sub-Skills, eine Klammer: **Loop-Engineering** — der Loop, der eine spezifizierte Aufgabe durch Bootstrap → Genesis → Execution treibt.

- `/init-agents` = Standards-Spec ablegen (in Repo-AGENTS.md)
- `/create-issue` = Intent ablegen (in GitHub-Issue)
- `/work-issue` = Intent ausfuehren (Issue → merged PR)

## Roadmap

- **v4.1.0 (Multi-Type-Loops, Issue #22):** dasselbe 5-Stage-Skelett fuer Non-Code-Loops — Text-Loop, Decision-Loop, Diagnostic-Loop.
- **v4.2.0 (Public Flip, Issue #23):** LICENSE, Secret-Sweep, Repo-Visibility-Public.

## Siehe auch

- `skills/init-agents/SKILL.md`
- `skills/create-issue/SKILL.md`
- `skills/work-issue/SKILL.md`
- `../CLAUDE.md` (Plugin-Quickstart)
