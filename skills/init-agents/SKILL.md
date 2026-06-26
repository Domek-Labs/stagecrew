---
name: init-agents
description: "AGENTS.md (Per-Repo Coding-Standards-Spec) für einen Git-Repo anlegen. YAML-Frontmatter mit 11 v3-Feldern (branch_pattern, default_branch, pr_base, commit_format, syntax_check, smoke_test, deploy_command, linter, hard_gates, default_oos, ac_templates) + Markdown-Body (Architektur, Code-Conventions, Test-Conventions, Known Gotchas). codebase-memory liefert intelligente Defaults aus dem Repo-Cluster. Pflicht-Bootstrap vor dem ersten /work-issue-Lauf in einem Repo. Trigger: /init-agents, init agents, agents.md anlegen, coding spec bootstrap, repo standards initialisieren."
---

# /init-agents — AGENTS.md fuer einen Repo anlegen

**Typ:** Bootstrap / Standards-Spec-Engineering

## Zweck

AGENTS.md ist die **Single-Source-of-Truth** fuer Per-Repo-Coding-Standards. Seit v3.1.0 ist `/work-issue` ein **Pure-Reader** (keine Hardcoded-Defaults mehr): alle Standards-Werte kommen aus AGENTS.md im Repo-Root.

Damit ist AGENTS.md **Pflicht** bevor `/work-issue` auf einem Repo laufen kann. `/init-agents` legt sie an — interaktiv (User wird durch alle Felder gefuehrt) oder autonom (codebase-memory liefert sinnvolle Defaults).

## Aufruf-Varianten

```
/init-agents                                    # cwd-Repo, interactive dialog
/init-agents --repo <slug>                      # spezifischer Repo, interactive (default)
/init-agents --repo <slug> --interactive        # explizit interactive
/init-agents --repo <slug> --auto               # autonom mit codebase-memory-Defaults
/init-agents --refine [--repo <slug>]           # bestehende AGENTS.md um fehlende Felder erweitern
```

**Argument-Parsing:** analog `/work-issue` und `/create-issue` (GitHub-Syntax, `--repo`-Flag, cwd-Fallback).

## Pre-Flight

In dieser Reihenfolge ausfuehren — bei einem Fehler ABORT mit klarer Hint-Message.

1. **Repo-Pfad-Lookup** in `~/.claude/work-issue-paths.yaml`.
   - Wenn Live-Datei fehlt: `cp` aus `references/repo-registry.yaml.example` (work-issue) als Bootstrap, dann nach `repo_path` fragen.
   - Wenn alter Pfad `~/.claude/skills/work-issue/repo-paths.yaml` existiert: Migrations-Hint ausgeben (einmalig moven), aber weiterhin mit Live-Pfad arbeiten.
2. **AGENTS.md-Existenz-Check** im Repo-Root (`<repo_path>/AGENTS.md`):
   - Wenn vorhanden UND kein `--refine`: ABORT.
     > AGENTS.md existiert bereits in `<repo_path>`. Ruf `/init-agents --refine --repo <slug>` auf, um sie zu ergaenzen, oder loesch sie zuerst manuell.
   - Wenn vorhanden MIT `--refine`: Frontmatter parsen, leere/fehlende Felder identifizieren.
   - Wenn fehlt UND `--refine`: Hint ausgeben "AGENTS.md fehlt — laeuft als normaler Init", weiter wie `--interactive`/`--auto`.
3. **codebase-memory Pre-Flight** (analog `/work-issue`):
   - `list_projects` — pruefe ob `<repo>` indexed.
   - Wenn nicht: `index_repository` mit Mode `moderate`.
   - Wenn indexed: `index_status` — Freshness-Check (>7 Tage UND neuer Commit → re-index).
   - Health-Check: `nodes < 200` ODER `source_files < 3` → Warnung "Default-Heuristik exkludiert evtl. Dirs (bin/, docs/, scripts/) — Re-Index mit expliziten Pfaden empfohlen, bevor `/init-agents --auto` laeuft".

## Dialog-Workflow (--interactive, default)

User wird durch alle 11 v3-Frontmatter-Felder gefuehrt. Pro Feld einen konkreten Default-Vorschlag aus codebase-memory + Heuristik anzeigen — User uebernimmt oder ueberschreibt.

### 11 v3-Frontmatter-Felder

| Feld | Default-Quelle / Heuristik |
|------|----------------------------|
| `branch_pattern` | `feature/<scope>-<short>` (Standard); aus letzten 20 Branch-Namen ableiten falls Pattern erkennbar (`git branch -r`) |
| `default_branch` | `gh api repos/<slug> --jq .default_branch` |
| `pr_base` | = `default_branch`; falls `dev`-Branch existiert (`gh api repos/<slug>/branches/dev`): Vorschlag `dev` (nicht autonom setzen) |
| `commit_format` | `conventional` (aus letzten 30 Commits: % match auf `type(scope): description` zaehlen — wenn >70% → `conventional`, sonst `custom`) |
| `syntax_check` | dominante Sprache aus `get_architecture` ableiten: TypeScript → `tsc --noEmit`, JavaScript → `node --check <file>`, Python → `python -m py_compile <file>`, sonst leer |
| `smoke_test` | aus `docker-compose.yml` (wenn vorhanden) → `docker compose build && docker compose up -d --force-recreate`. Sonst aus `package.json scripts.test` oder `pyproject.toml scripts.test`. Sonst leer |
| `deploy_command` | leer (User trifft Entscheidung) |
| `linter` | aus Repo: `eslint`/`ruff`/`black` Configs detektieren |
| `hard_gates` | Liste, Defaults: "Kein direkter Edit auf default_branch", "Keine neuen Secrets im Repo", "Alle MCP-Tools brauchen Test-Beispiel im Doc-Block" |
| `default_oos` | Liste, leer als Default — User ergaenzt repo-spezifisch |
| `ac_templates` | Liste, Defaults: "Code passes <syntax_check>", "Documentation in README/CLAUDE.md aktualisiert", "No new secrets in repo (Secret-Check passed)" |

### 4 Body-Sektionen

Pro Sektion 1-3 Vorschlaege aus codebase-memory bzw. Cluster-Analyse zeigen. User uebernimmt oder schreibt selber.

| Sektion | Default-Quelle |
|---------|----------------|
| **Architektur (kurz)** | `get_architecture` → top 3 Cluster + Hauptkomponenten als Vorschlag |
| **Code-Style / Conventions** | `search_code` fuer dominante Naming-Patterns + Lint-Config-Findings |
| **Test-Conventions** | Test-Dir-Detection (`test/`, `tests/`, `__tests__/`) + dominantes Test-Framework aus `package.json`/`pyproject.toml` |
| **Bekannte Stolpersteine (Known Gotchas)** | initial leer — User traegt nach (Skill empfiehlt: "leere Section ist ok, ergaenze nach erstem Build") |

## Auto-Workflow (--auto)

Vollstaendige Default-Heuristik durchfahren — kein User-Dialog. Output ist Preview, User bekommt am Ende einen YES/NO-Prompt.

Defaults:
- **branch_pattern**: `feature/<short>` (sicherer Standard, kein autonomes Pattern-Detect)
- **default_branch**: via `gh api repos/<slug> --jq .default_branch`
- **pr_base**: = `default_branch` (nie autonom auf `dev` wechseln, auch wenn `dev` existiert — nur Vorschlag im interactive Dialog)
- **commit_format**: `conventional` als Default (sicherer Standard)
- **syntax_check**: aus dominanter Sprache via Cluster-Analyse
- **smoke_test**: aus `docker-compose.yml` / `package.json` / `pyproject.toml`
- **deploy_command**: leer
- **linter**: aus detektierten Config-Files
- **hard_gates**: 3 Standard-Gates (siehe oben)
- **default_oos**: leer
- **ac_templates**: 3 Standard-Templates (siehe oben)
- **Body-Sektionen**: aus `get_architecture` + `search_code` befuellt; Known Gotchas leer

## Refine-Workflow (--refine)

1. Bestehende `<repo_path>/AGENTS.md` lesen.
2. YAML-Frontmatter parsen (`python3 -c "import yaml; print(yaml.safe_load(open('<path>').read().split('---')[1]))"`).
3. Pro Feld pruefen: leer/fehlt? → in Refine-Liste.
4. Dialog NUR fuer Felder in Refine-Liste — andere bleiben unveraendert.
5. Body-Sektionen analog: leer/fehlt → Dialog, sonst unveraendert.
6. Diff zeigen, User APPROVE/CANCEL.

## Output-Schritte (nach Dialog/Auto/Refine)

1. **Preview** — vollstaendigen AGENTS.md-Inhalt rendern (Frontmatter + Body). User: APPROVE / REVISE / CANCEL.
2. **AGENTS.md schreiben** in `<repo_path>/AGENTS.md`.
3. **README.md ergaenzen** mit "For Contributors / AI Agents"-Section, wenn nicht schon da:
   ```markdown
   ## For Contributors / AI Agents
   Diese Repo nutzt `AGENTS.md` als Standards-Spec fuer AI-gestuetzte
   Workflows (`/work-issue`, `/create-issue`). Konventionen aenderst du
   dort, nicht inline in der Code-Review.
   ```
4. **Branch + Commit + PR + Merge** (analog `/work-issue` Implementer/Closer):
   - Branch: `feature/init-agents-md` (oder per `branch_pattern`)
   - Commit-Message: `chore(agents): add AGENTS.md (init via /init-agents)` mit `Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>`
   - PR-Body: kurz, Verweis auf AGENTS.md-Format
   - Squash-Merge (delete-branch)
5. **Repo-Registry-Update (Pflicht-Schritt)** — Live-Datei `~/.claude/work-issue-paths.yaml`:
   ```bash
   yq -i '.["<owner>/<repo>"].agents_md_exists = true' ~/.claude/work-issue-paths.yaml
   # Fallback ohne yq:
   # sed-Edit oder Python-Update auf gleicher Live-Datei
   ```
   Wenn der Eintrag noch nicht existiert: vorher minimal anlegen (`repo_path` aus Pre-Flight ist bekannt).
6. **Channel-Reply** (wenn aus `<channel>`-Inbound): "AGENTS.md fuer `<slug>` angelegt (PR #N gemerged). Repo ist bereit fuer `/work-issue`."

## Known Limitation (Dogfooding-Hinweis)

Wenn `/init-agents` zum ersten Mal nach Plugin-Update verfuegbar wird, kann er sich selbst auf den `dominik`-Plugin-Repo aufrufen, um dessen AGENTS.md anzulegen. **Aber:** Plugin-Cache muss VOR diesem Aufruf neu sein (`claude restart`).

Daher empfohlene Reihenfolge fuer den Bootstrap-Lauf:
1. PR mergen (`feature/init-agents` → `main`)
2. `claude restart` (Plugin-Cache neu)
3. `/init-agents --repo dscheinecker-at7media/dominik --interactive` als Bootstrapping-Lauf fuer den Plugin-Repo selbst.

Ohne Restart in Schritt 2 sieht der laufende Claude noch v3.0.0-Skills — also weder den neuen `/init-agents` noch das aktualisierte `/work-issue` (Pure-Reader).

Gleiches gilt fuer andere Repos: jedes Mal wenn ein Repo erstmalig eine AGENTS.md kriegt, sollte vor dem naechsten `/work-issue`-Lauf in dem Repo ein `claude restart` erfolgen (damit Pure-Reader die frische Datei sieht). Praktisch reicht meist ein Reload der Session.

## AGENTS.md-Format-Referenz

Template-File: `references/AGENTS.md.template` (in diesem Skill).

Schema (verkuerzt):

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

## Architektur (kurz)
## Code-Style
## Test-Conventions
## Bekannte Stolpersteine
```

Vollstaendiges Beispiel siehe `references/AGENTS.md.template`.

## Abgrenzung

| Skill | Wann |
|-------|------|
| `/init-agents` | Repo hat noch keine AGENTS.md → Bootstrap-Pflicht VOR dem ersten `/work-issue`-Lauf |
| `/create-issue` | Idee → spezifiziertes GitHub-Issue. Ruft `/init-agents` als Pre-Step, wenn AGENTS.md fehlt |
| `/work-issue` | Pure-Reader: braucht AGENTS.md im Repo-Root. Ohne AGENTS.md → STOP-Verdict mit Pointer auf `/init-agents` |

`/init-agents` und `/create-issue` sind komplementaer:
- `/init-agents` legt **Standards** an (einmalig pro Repo)
- `/create-issue` legt **Issues** an (laufend pro Feature/Bug)

`/init-agents` und `/work-issue` sind sequenziell:
- `/init-agents` ist **Bootstrap-Pflicht** vor dem ersten `/work-issue`-Lauf in einem Repo.

## Was der Skill NICHT macht

- **Keine Migration aus legacy `code/projects.yaml`** — v3.0.0 hat diesen Pfad bereits entfernt, alte Daten sind verloren.
- **Kein Schema-Versioning** (z.B. `schema_version: 1`-Field). Kommt v3.2 falls Migration noetig.
- **Kein Auto-Detection von dev-Branch-Flow** im `--auto`-Modus — wenn `dev`-Branch existiert, wird er im interactive Dialog **vorgeschlagen**, aber nie autonom als `pr_base` gesetzt.
- **Keine AGENTS.md fuer Non-Code-Repos** (z.B. reine Doku-Repos). Skill geht von Code-Repo aus.
- **Keine Cross-Repo-Standards-Vererbung** — jeder Repo hat seine eigene AGENTS.md.

## Siehe auch

- `references/AGENTS.md.template` — vollstaendiges Beispiel + Schema
- `skills/work-issue/SKILL.md` — Pure-Reader-Loop, der AGENTS.md liest
- `skills/create-issue/SKILL.md` — Issue-Genesis, ruft `/init-agents` als Pre-Step
- `skills/work-issue/references/repo-registry.yaml.example` — Repo-Registry-Template
