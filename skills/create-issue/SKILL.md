---
name: create-issue
description: "GitHub-Issue aus Feature-Wunsch / Idee erstellen, mit vollem Spec-Standard (Idee/Spec/AC/Files-To-Touch/Test-Plan/Out-of-Scope). Multi-Repo. AGENTS.md im Repo überschreibt Default-Templates. v3.2.0 Auto-Injection von AGENTS.md ac_templates in den AC-Block plus 6-Trigger-Issue-Type-Detection (docs/epic/secret/mcp/live-ping/live-service). Codebase-memory liefert automatisch Files-To-Touch-Vorschläge und Hotspots. Optional Auto-Übergabe an /work-issue. Trigger: /create-issue, create issue, neues issue, issue anlegen, feature anlegen, spec issue, idee als ticket."
---

# /create-issue — Idee → spezifiziertes GitHub-Issue

**Typ:** Issue-Genesis / Spec-Engineering
**Version:** v3.2.0 (Auto-Injection + Issue-Type-Detection)

## Zweck

Issue-Genesis-Phase. Aus einer User-Idee wird ein vollstaendig spezifiziertes GitHub-Issue, das direkt von `/work-issue` aufgearbeitet werden kann. Skill stellt sicher, dass jedes Issue Idee+Spec+AC+Files+Test-Plan+OoS hat — bevor irgendwelcher Code geschrieben wird.

Spec-Standard ist deckungsgleich zu dem, was `/work-issue`s Validator als GO-Kriterium erwartet — dadurch keine Validator-STOPs am ersten Loop-Lauf.

## Aufruf-Varianten

```
/create-issue "voice-diff-review für mei"
/create-issue --repo dscheinecker-at7media/personal-ai-bot "scheduler verbessern"
/create-issue                                  # → fragt nach Idee + Repo
/create-issue --refine <num> [--repo <slug>]   # → bestehendes Issue um Spec-Felder erweitern
```

**Argument-Parsing:** analog `/work-issue` (GitHub-Syntax, `--repo` Flag, cwd-Fallback).

## Workflow

### a) Repo-Auflösung

Analog `/work-issue`:
1. GitHub-Syntax `<owner>/<repo>` ODER `--repo` Flag
2. cwd-Fallback via `git remote get-url origin`
3. Wenn unklar: fragen

### b) AGENTS.md-Pre-Step (NEU v3.1.0)

**Vor allem anderen:** AGENTS.md-Existenz-Check im Repo-Root.

- `ls <repo_path>/AGENTS.md` → existiert? → weiter zu (c).
- Fehlt → `/init-agents --repo <slug> --interactive` aufrufen, dann zurueck zu (b) — Re-Check.

Hintergrund: `/work-issue` ist seit v3.1.0 Pure-Reader und braucht AGENTS.md. `/create-issue` baut darauf auf — `ac_templates` und `default_oos` aus AGENTS.md werden im Spec-Dialog als Default vorgeschlagen.

### c) Codebase-Memory-Pre-Flight

Gleicher Block wie `/work-issue`:
1. `list_projects` — pruefe ob `<repo>` indexed
2. Wenn nicht: `index_repository` (Mode `moderate`)
3. Wenn indexed: `index_status` — Freshness-Check (>7 Tage UND neuer Commit → re-index)
4. Health-Check: `nodes < 200` ODER `source_files < 3` → Warnung "Default-Heuristik exkludiert evtl. Dirs"

### d) AGENTS.md-Read

Im Repo-Root lesen (existiert garantiert nach Schritt b). Frontmatter laden fuer `ac_templates` und `default_oos` — wird im Spec-Dialog als Default vorgeschlagen.

### e) Spec-Dialog (interaktiv, 7 Fragen)

User wird durch die folgenden Felder gefuehrt. Bei jeder Frage darf der Skill konkrete Vorschlaege liefern.

1. **Idee + strategischer Kontext (Why)** — Was soll wirklich erreicht werden, welcher User-Pain wird geloest?
2. **Files To Touch** — Welche Files werden vermutlich angefasst?
   - **Skill schlaegt aktiv vor:** Aus der Idee Keywords extrahieren (z.B. "scheduler", "voice-diff", "ollama") → `search_code` mit Keywords → Funktionen + Files. `get_architecture` fuer Cluster-Kontext. Vorschlag: "Diese Files werden vermutlich angefasst: <liste>. Stimmt das?"
   - Hotspots im Code-Graph als Diff-Risiko ausweisen.
3. **Akzeptanzkriterien** als Checkbox-Liste
   - Seit v3.2.0 werden `ac_templates` aus AGENTS.md **injiziert** (nicht nur vorgeschlagen) — siehe Sektion "Auto-Injection von AGENTS.md `ac_templates`" weiter unten.
   - User ergaenzt issue-spezifische AC, die nach den Pflicht-Templates eingefuegt werden.
   - Zusaetzlich laeuft die Issue-Type-Detection (6 Trigger) vor dem Preview — siehe Sektion "Issue-Type-Detection".
4. **Test-Plan** — Welche Smoke-Tests? (Build-Command, Service-Restart, manueller Check)
5. **Out-of-Scope** — Was wird explizit NICHT in diesem Issue gemacht?
   - Skill schlaegt `default_oos` aus AGENTS.md vor (z.B. "Multi-User-Support", "Web-Channel-Approvals")
6. **Dependencies** — Welche Issues blocken? Werden welche blockiert? (`depends on #X`, `blocks #Y`)
7. **Milestone + Labels** — `gh api repos/<slug>/milestones` → User waehlt aus offenen. Labels frei.

### f) Issue-Preview

Vollstaendigen Issue-Body rendern. User: APPROVE / REVISE / CANCEL.

### g) Issue-Create

```
gh issue create --repo <slug> \
  --title "<title>" \
  --body "<rendered-body>" \
  --milestone "<milestone>" \
  --label "<labels>"
```

### h) Optional: Auto-Übergabe an /work-issue

> Issue #<num> angelegt. Direkt mit `/work-issue` durchziehen? (Ja/Nein)

Bei Ja → Issue-Nummer + Repo an `/work-issue` weiterleiten. Bei Nein → Issue bleibt im Backlog.

## codebase-memory-Integration im Detail

Fuer Files-To-Touch-Vorschlag und Spec-Anreicherung:

1. Aus der Idee Keywords extrahieren (Substantive, Funktionsnamen, Module-Hints).
2. `search_code` mit jedem Keyword → Treffer-Funktionen + Files.
3. `get_architecture` → Cluster-Kontext der Treffer (welche Module sind verbunden).
4. Hotspots = Files mit hoher Verbindungsdichte → als Diff-Risiko ausweisen.
5. Vorschlag an User: "Diese Files werden vermutlich angefasst: `path/a.ts`, `path/b.ts`. Hotspot-Warnung fuer `path/core.ts` (hohe Konnektivitaet — Aenderung dort hat breitere Wirkung)."

## AGENTS.md-Generation (delegiert an /init-agents)

Bis v3.0.0 hatte `/create-issue` einen inline-Auto-Generation-Workflow fuer AGENTS.md. Seit v3.1.0 ist das in `/init-agents` zentralisiert (Single-Source).

Wenn AGENTS.md fehlt, ruft `/create-issue` im Pre-Step (Schritt b) `/init-agents --repo <slug> --interactive` auf und kehrt danach zurueck. Damit ist sichergestellt, dass `/work-issue` (Pure-Reader seit v3.1.0) das Issue direkt aufnehmen kann.

## Repo-Registry-Update

Wenn `<repo>` noch nicht in `~/.claude/work-issue-paths.yaml`:
- einmal nach `repo_path` fragen
- einmal nach `deploy_command` fragen (optional)
- einmal nach `live_path` fragen (optional, sonst = `repo_path`)
- persistieren

Damit ist der Repo bereit fuer `/work-issue` direkt im Anschluss.

## Issue-Body-Template (vollständig)

Exakt dieses Schema verwenden — `/work-issue`s Validator pruefen genau diese Sektionen:

```markdown
## Idee (Why)

<strategischer Kontext, User-Pain, Outcome>

## Spec (What)

<technische Spec, Architektur, betroffene Komponenten>

## Akzeptanzkriterien

- [ ] <konkret, testbar>
- [ ] <konkret, testbar>
- [ ] ...

## Files to Touch

- `<pfad/a.ts>` — <kurz: was passiert dort>
- `<pfad/b.ts>` — <kurz>
- ...

## Test-Plan

<welche Smoke-Tests, welcher Build-Command, was beweist Success>

## Dependencies / Blocks

- depends on #<X>
- blocks #<Y>

## Out of Scope

- <was explizit NICHT>
- ...

## Standards-Override (optional)

```yaml
# nur wenn dieses Issue von AGENTS.md abweichen soll
work-issue:
  syntax_check: "<custom>"
  smoke_test: "<custom>"
```
```

## Standards-Quelle

Gleich wie bei `/work-issue` (2-Tier seit v3.1.0: AGENTS.md → Issue-Override). `/create-issue` nutzt es vor allem fuer:
- `ac_templates` (seit v3.2.0 **auto-injiziert** im AC-Block — siehe naechste Sektion, nicht mehr nur "Vorschlag")
- `default_oos` (Vorschlag im Spec-Dialog)
- `hard_gates` (als Pflicht-AC einfuegen wenn relevant)

## Auto-Injection von AGENTS.md `ac_templates`

Seit **v3.2.0** ist die Render-Logik in Schritt (e3) Akzeptanzkriterien opinionated: AGENTS.md `ac_templates` werden **injiziert**, nicht mehr "vorgeschlagen". Drei Modi:

### Modus 1 — Default (kein Standards-Override)

Alle `ac_templates`-Items aus AGENTS.md werden im rendered AC-Block als Pflicht-Items **vorangestellt** (klar markiert als "**aus AGENTS.md `ac_templates`**"), gefolgt von Issue-spezifischen User-AC.

### Modus 2 — Override `ac_templates: []` (leer)

Block wird leer gelassen; User muss explizit Ersatz-AC liefern (typisch fuer docs-only Issues). Skill **warnt** wenn der AC-Block leer bleibt und keine User-AC nachgereicht werden.

### Modus 3 — Override mit teilweiser/anderer Liste

Die Override-Liste **ersetzt komplett** die AGENTS.md-Templates und wird im selben Pflicht-Block gerendert, plus User-AC.

### Rendered-Format-Beispiel

```markdown
## Akzeptanzkriterien

### aus AGENTS.md `ac_templates` (Pflicht)
- [ ] <template 1>
- [ ] <template 2>
- ...

### Issue-spezifisch
- [ ] <user-AC 1>
- [ ] <user-AC 2>
```

Diese Injection wird in Schritt (e3) der Workflow-Definition aktiv — nicht mehr "vorschlagen", sondern **injizieren**. Der User kann waehrend des Spec-Dialogs Pflicht-Items per `EDIT` umformulieren oder per `DISMISS` rausnehmen (mit Begruendung im `## Standards-Notes`-Block), aber sie sind by default drin.

## Issue-Type-Detection

6 Trigger, regex/keyword-basiert. Heuristik laeuft VOR Render-Schritt (e6 Issue-Preview). Triggers koennen stacken — ein Issue kann mehrere Trigger ausloesen (z.B. neue MCP-Skill mit `ANTHROPIC_API_KEY` -> #3 + #4 aktiv).

| # | Trigger-Bedingung | Pflicht-Erweiterung |
|---|------------------|---------------------|
| 1 | Label `docs`/`documentation` ODER alle Files-To-Touch matched `^docs/`, `\.md$`, `^LICENSE$`, `^CONTRIBUTING` | Standards-Override-Block (`ac_templates: []`) + docs-spezifische Ersatz-AC vorschlagen (Markdown-Lint, ADR-Schema-Grep, Mermaid via `mmdc`) |
| 2 | Label `epic` ODER `>5` Files-To-Touch ODER Spec-Block `>800` Woerter | Epic-Warnung im Preview: "Sub-Issue-Split empfohlen. OoS muss Sub-Issue-Liste enthalten oder Issue umstrukturieren." |
| 3 | Spec ODER Test-Plan enthaelt Pattern `[A-Z_]+_(KEY\|TOKEN\|SECRET\|PAT)` ODER String "Token"+"env"/"Secret"+"PAT" | HG3-Reminder: AC fuer `.env.example`-Eintrag, `docker-compose env_file:`-Hinweis, entrypoint-Preflight, Secret-Scan vor Commit |
| 4 | Spec enthaelt `claude mcp add` ODER `mcp__` ODER "MCP-Tool" ODER "MCP-Plugin" | HG1-Reminder: AC fuer Side-Effect-Verifikation in `~/.cache/claude-cli-nodejs/.../mcp-logs-<name>/*.jsonl` |
| 5 | Test-Plan ODER AC enthaelt "Telegram", "gh issue create", "API-Call", externe URLs, `curl` mit Token | Mock-/Dry-Run-Pflicht: AC + Test-Plan brauchen `DRY_RUN=1` env-Flag oder Sandbox-Target. Cost-Warning im Preview. |
| 6 | Test-Plan enthaelt `docker compose up`, `systemctl restart`, "deploy", "live", "API-Call" | Cleanup-Pflicht: AC fuer Cleanup-Statement im Tester (Service teardown, Test-Daten purge, Original-State wiederherstellen) |

### Skill-Reaktion auf Trigger

1. Im Spec-Dialog vor finaler Preview druckt Skill **erkannte Trigger als Liste mit Begruendung** (z.B. `"Trigger #3 HG3: erkannt 'ANTHROPIC_API_KEY' im Spec-Block"`).
2. Pro Trigger werden **automatische AC-Items vorgeschlagen** — User: `APPROVE` / `EDIT` / `DISMISS`.
3. `DISMISS` schreibt einen `## Standards-Notes`-Block ins Issue mit Begruendung (z.B. "HG3 dismissed: Token wird via separate Issue eingefuehrt #N").

### Beispiel-Flow: Trigger #3 (HG3 Bearer-Token-Detection)

**Sample-Spec-Block:**
> Skill nutzt `ANTHROPIC_API_KEY` als env-var fuer Anthropic-SDK-Calls. Token wird in `.env` geladen und im Docker-Compose-Stack injiziert.

**Erkanntes Pattern:** `ANTHROPIC_API_KEY` matcht Regex `[A-Z_]+_(KEY|TOKEN|SECRET|PAT)`.

**Generierte AC-Items (vorgeschlagen):**
- [ ] `.env.example` hat `ANTHROPIC_API_KEY` mit Scope-Kommentar (welcher Service liest, welche Permissions)
- [ ] `docker-compose.yml` `env_file:` referenziert `.env`
- [ ] entrypoint-Preflight: bei fehlendem Token Warning + skip Registrierung (kein hard-crash)
- [ ] `git diff --cached` Secret-Scan vor Commit greppt auf Token-Pattern (kein Echtwert im Diff)

User entscheidet pro AC-Item `APPROVE` / `EDIT` / `DISMISS`. Bei `DISMISS` aller vier Items wird `## Standards-Notes` mit der Begruendung ans Issue angehaengt.

## Refine-Modus v3.2.0

`/create-issue --refine <num>` laedt ein bestehendes Issue und ergaenzt fehlende Sektionen. Fuer den Fall: `/work-issue` hat STOP gegeben weil Spec unvollstaendig. Workflow (5-Schritt seit v3.2.0):

1. **Bestehenden Body laden** — `gh issue view <num> --repo <slug> --json title,body,labels,milestone`
2. **Type-Detection auf vorhandenem Body laufen lassen** — alle 6 Trigger pruefen, identifizieren welche feuern.
3. **AGENTS.md `ac_templates`-Diff** — welche Templates fehlen im aktuellen AC-Block? -> Pflicht-Add (oder Standards-Override-Block falls justified).
4. **Trigger-Diff** — welche Trigger feuern, deren Pflicht-AC im bestehenden Body fehlen? -> Pflicht-Add.
5. **User-Diff-Approval** — Skill zeigt **nur die Aenderungen** (nicht den ganzen Body), User `APPROVES` / `REJECTS` einzeln pro Item. Approved-Items werden via `gh issue edit <num>` in den Body integriert.

Damit faengt Refine-Mode genau den Spec-Gap-Pattern ab, der die Producer-Side bisher zu kulant gemacht hatte.

## Abgrenzung zu /work-issue

| Skill | Wann |
|-------|------|
| `/create-issue` | Idee existiert, Issue noch nicht. ODER Issue existiert ohne Spec → `--refine` |
| `/work-issue` | Issue mit Spec → Build-Loop bis Merge |

## Was der Skill NICHT macht

- Schreibt keinen Code (nur Issue-Body)
- Erstellt keine Branches
- Macht keine Cross-Repo-Dependency-Aufloesung (depends-on wird nur als Referenz eingefuegt, nicht aufgeloest)

## Siehe auch

- `skills/init-agents/SKILL.md` — Bootstrap-Phase (AGENTS.md anlegen, Pflicht vor erstem Lauf)
- `skills/work-issue/SKILL.md` — Execution-Phase
- `skills/work-issue/references/repo-registry.yaml.example`
- `skills/init-agents/references/AGENTS.md.template`
