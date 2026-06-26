---
name: work-issue
description: "GitHub Issue durch 5-Stage Spec→Build-Loop ziehen (Validator → Implementer → Tester → Critic → Closer). Multi-Repo. Pure-Reader v3.1.0 — alle Standards-Werte kommen aus AGENTS.md im Repo-Root (keine Skill-Defaults). v3.2.0 Multi-Type-System: liest loop-type:<type>-Label und dispatched zu type-spezifischem Subagent-Brief (code|research). Pflicht-Pre-Flight checkt AGENTS.md-Existenz, YAML-Parseability und Vollstaendigkeit. Codebase-memory Pflicht-Pre-Flight. Auto-PR + Merge bei APPROVE. Trigger: /work-issue, work issue, loop engineering, issue durchziehen, spec build loop."
---

# /work-issue — Spec→Build-Loop fuer GitHub-Issues

**Typ:** Loop-Engineering / Autonome Issue-Implementation
**Version:** v3.2.0 (Multi-Type-System: Code + Research)

## Zweck

Ein vollstaendig spezifiziertes GitHub-Issue (Idee + Spec + AC + Files-To-Touch + Test-Plan + OoS) durch 5 dedizierte Subagents bis Merge ziehen. Jede Stage = ein Agent, postet `[stage:<name>]`-Kommentar am Issue als Audit-Log, dann startet die naechste Stage.

Seit **v3.1.0** ist dieser Skill ein **Pure-Reader**: keine Hardcoded-Defaults mehr. Alle Standards-Werte (branch_pattern, syntax_check, smoke_test, ...) kommen aus **AGENTS.md im Repo-Root**. Fehlt AGENTS.md → STOP mit Pointer auf `/init-agents`.

Pattern aus Loop-Engineering-Drawer `b0a7a369` (palace-dominik / wing_personal / room_arbeitsweise_mit_claude / 2026-06-21). Erst-Erprobung 2026-06-25 mit `dscheinecker-at7media/personal-ai-bot#5` (33 min, 2 Iterationen, alle 5 AC gruen).

## Aufruf-Varianten

```
/work-issue 5 --repo dscheinecker-at7media/personal-ai-bot
/work-issue dscheinecker-at7media/personal-ai-bot#5
/work-issue 5                                # → fallback: cwd-git-repo
/work-issue                                  # → listet offene Issues, fragt nach Auswahl
```

**Argument-Parsing:**
1. `<owner>/<repo>#<num>` matched → splitte in `repo` + `issue_num`
2. `--repo <slug>` Flag + freie Zahl
3. `git -C $(pwd) remote get-url origin` → cwd-Fallback
4. Keine Zahl → `gh issue list --repo <slug> --state open --limit 30`, User waehlt
5. Wenn keine Issue-Nummer aufloesbar → fragen, ob `/create-issue` aufgerufen werden soll

## Standards-Quelle (2-Tier seit v3.1.0)

| Tier | Quelle | Was es ueberschreibt |
|------|--------|----------------------|
| L1   | `AGENTS.md` im Repo-Root (YAML-Frontmatter) | Basis-Werte — Repo-spezifische Conventions |
| L2   | `## Standards-Override`-Block im Issue-Body | L1 — Issue-spezifischer One-Shot |

Lade L1 → L2, merge nach rechts. **L1 ist Pflicht** (siehe Pre-Flight-Check unten). Frueheres Skill-Default-L1 ist entfallen (v3.1.0), `/init-agents` legt AGENTS.md an.

**Producer-Side seit v3.2.0** (`/create-issue`): injiziert AGENTS.md `ac_templates` automatisch in den AC-Block und detected Issue-Types (docs/epic/secret/mcp/live-ping/live-service) mit Pflicht-Erweiterungen. Validator bleibt strict — Producer ist jetzt opinionated. Details siehe `skills/create-issue/SKILL.md`.

## AGENTS.md-Format (L1)

YAML-Frontmatter + Markdown-Body. Frontmatter wird vom Skill geparst, Body wird im Subagent-Briefing als Kontext geladen.

```yaml
---
work-issue:
  branch_pattern: "feature/<scope>-<short>"
  default_branch: main | dev
  pr_base: main | dev
  commit_format: "conventional" | "custom"
  syntax_check: "<command>"           # z.B. "node --check <file>"
  smoke_test: "<command>"             # z.B. "docker compose build && smoke"
  deploy_command: "<command>"
  linter: "<command>"
  hard_gates:
    - "Beschreibung der Regel"
  default_oos:
    - "..."
  ac_templates:
    - "Tests gruen"
    - "Docs aktualisiert"
---

# Agent Instructions for <repo>
... narrative Kontext (Architektur, Conventions, "warum so") ...
```

Template: `../init-agents/references/AGENTS.md.template`. Wenn ein Repo noch keine AGENTS.md hat, ruf `/init-agents --repo <slug>` auf.

## Codebase-Memory-Pflicht (Pre-Flight)

Der Skill ruft IMMER vor Stage 1 das codebase-memory-MCP, um den Subagents korrekten Kontext zu geben:

1. **`list_projects`** — pruefe ob `<repo>` indexed.
2. **Wenn nicht indexed:** `index_repository` mit Mode `moderate` (Default-Heuristik).
3. **Wenn indexed:** `index_status` fuer Freshness-Check.
   - Schwellwert: 7 Tage seit letztem Index UND letzter Commit > Index-Datum → auto re-index.
4. **Health-Check:** wenn `nodes < 200` ODER `source_files < 3` → Warnung an User:
   > Default-Heuristik exkludiert evtl. wichtige Dirs (bin/, docs/, scripts/). Re-Index mit expliziten Pfaden empfohlen, bevor Loop laeuft.

### Wo welches Tool im Loop genutzt wird

- **Validator (Stage 1):** `get_architecture` — Issue-genannte Files gegen Graph pruefen (existieren? im Cluster?).
- **Implementer (Stage 2):** `search_code` fuer Sibling-Funktionen + Conventions im Repo.
- **Critic (Stage 4):** `search_code` pro AC-Box als Code-Evidence ("AC sagt X, Code zeigt Y").

## Repo-Registry

Live-Datei: `~/.claude/work-issue-paths.yaml` (persistiert ueber Plugin-Updates).
Template: `references/repo-registry.yaml.example` (mit Plugin ausgeliefert).

Felder pro Repo:

```yaml
<owner>/<repo>:
  repo_path: /abs/path/to/checkout
  live_path: /abs/path/to/deployed         # optional, fuer Drift-Check
  default_branch: main | dev
  pr_base: main | dev
  has_dev_branch: true | false             # auto-detect bei pre-flight
  syntax_check: "<command>"
  smoke_test: "<command>"
  deploy_command: "<command>"
  agents_md_exists: true | false           # gesetzt von /init-agents
```

Beim ersten Lauf fuer einen neuen Repo: einmal nach `repo_path` + `deploy_command` fragen, in Live-Datei persistieren. Wenn Live-Datei fehlt: `cp` aus Template als Bootstrap.

## Pre-Flight Stage 0 (NEU vor Validator)

In dieser Reihenfolge ausfuehren:

1. **Repo-Registry-Lookup** → `repo_path` aus `~/.claude/work-issue-paths.yaml`. Wenn fehlt: einmal nach Pfad fragen.
2. **Codebase-Memory-Pre-Flight** (siehe oben: list_projects → index/freshness → health-check).
3. **AGENTS.md-Pflicht-Check (NEU v3.1.0)** — siehe naechste Sektion.
4. **Loop-Type-Resolution (NEU v3.2.0)** — siehe naechste Sektion.
5. **Default-Branch ermitteln:** `gh api repos/<slug> --jq .default_branch` (Cross-Check gegen AGENTS.md-Wert).
6. **`dev`-Branch-Existenz pruefen:** `gh api repos/<slug>/branches/dev` → 200 oder 404 (Info-only).
7. **Live-Path-Drift-Check** (wenn `live_path != repo_path`): `diff -r <live_path> <repo_path>` → bei Drift Warnung an User vor Stage 1.
8. **Channel-Inbound-Detection:** wenn `<channel>`-Tag → kurzer Reply "Loop fuer <slug>#<n> startet (`loop-type:<type>`), Stage 1 laeuft."

### AGENTS.md-Pflicht-Check (NEU v3.1.0)

Vor Stage 1 in dieser Reihenfolge pruefen — jeder Fehler ist ein **STOP-Verdict** (kein Stage 1, kein Subagent-Start).

1. **Existenz**: `ls <repo_path>/AGENTS.md`
   - Fehlt → STOP:
     > AGENTS.md im Repo-Root fehlt (`<repo_path>/AGENTS.md`). Bitte zuerst `/init-agents --repo <slug>` aufrufen, um die Standards-Spec anzulegen. Danach `/work-issue <num> --repo <slug>` erneut starten.

2. **YAML-Parseability**:
   ```bash
   python3 -c "import yaml; yaml.safe_load(open('<repo_path>/AGENTS.md').read().split('---')[1])"
   ```
   - Fehler → STOP:
     > AGENTS.md hat Syntax-Fehler im YAML-Frontmatter: `<error-message>` (Zeile `<line>`). Bitte korrigieren oder `/init-agents --refine --repo <slug>` aufrufen, um Felder neu zu setzen.

3. **Vollstaendigkeit**: alle 11 Felder im `work-issue:`-Namespace sind gesetzt (`branch_pattern`, `default_branch`, `pr_base`, `commit_format`, `syntax_check`, `smoke_test`, `deploy_command`, `linter`, `hard_gates`, `default_oos`, `ac_templates`).
   - Fehlende Felder → STOP:
     > AGENTS.md unvollstaendig. Fehlende Felder: `<liste>`. Bitte `/init-agents --refine --repo <slug>` aufrufen, um die Felder zu ergaenzen.

   Hinweis: leere Listen (`[]`) und leere Strings (`""`) gelten als gesetzt — `deploy_command: ""` ist erlaubt, `deploy_command` fehlend ist STOP.

Bei allen drei Checks OK: Frontmatter cachen und Body fuer Stage-Briefings vorhalten. Dieser Cache wird als Quelle fuer alle Standards-Werte in den Subagent-Briefings genutzt.

### Loop-Type-Resolution (NEU v3.2.0)

Vor Stage 1 den Issue-Loop-Type ermitteln. In dieser Reihenfolge:

1. **Issue-Label** — `gh issue view <num> --repo <slug> --json labels --jq '.labels[].name'` → suche `loop-type:<type>` Prefix.
2. **Body-Frontmatter-Fallback** — Wenn Issue-Body mit `---\nloop-type: <type>\n---` startet → diesen Wert nutzen. (Kommt vor wenn Issue manuell ohne Skill angelegt wurde.)
3. **AGENTS.md-Default-Fallback** — `loop_types.default` aus AGENTS.md (typisch `code`).
4. **Letzter Fallback** — `code` (Backwards-Compat fuer Pre-v3.2.0-Issues).

Validierung gegen `loop_types.enabled` aus AGENTS.md:
- Wenn resolved-Type **nicht** in `enabled`: STOP-Verdict:
  > Issue hat `loop-type:<X>`, aber AGENTS.md `loop_types.enabled` enthaelt nur `<liste>`. Bitte AGENTS.md erweitern (`/init-agents --refine`) oder Issue-Label korrigieren.

- Wenn resolved-Type ist `text`/`decision`/`diagnostic` (Roadmap-Types): STOP-Verdict mit Roadmap-Hint:
  > Loop-Type `<type>` ist Roadmap, noch nicht implementiert. Aktuell unterstuetzt: code, research. Verfolge Roadmap in den Repo-Issues mit Label `loop-type` + `roadmap`.

Bei OK: Resolved-Type in State-Tracker schreiben (`loop_type` Feld). Subagent-Brief-Loader benutzt diesen Wert.

### Subagent-Brief-Loader (NEU v3.2.0)

Pro Stage 2 (Implementer) wird der Brief aus `references/subagent-briefs/<loop_type>-implementer.md` geladen. Briefe haben Placeholder (`{{branch_pattern}}`, `{{syntax_check}}`, etc.) die zur Render-Zeit aus dem AGENTS.md-Cache + State-Tracker ersetzt werden.

| Loop-Type | Subagent-Brief-File |
|-----------|---------------------|
| `code` | `references/subagent-briefs/code-implementer.md` |
| `research` | `references/subagent-briefs/research-implementer.md` |

Tester / Critic / Closer-Stages bleiben **strukturell gleich**, aber pruefen type-spezifische Kriterien:
- `code`: Build GREEN via `smoke_test`, Code-Diff-Quality
- `research`: Doc-Quality-Check (Wortzahl, Test-Matrix, Working-Setup oder Hypothese-Roadmap)

Die Type-spezifischen Pruef-Kriterien sind in den jeweiligen Subagent-Brief-Files dokumentiert (Sektionen "Tester-Pruef-Kriterien" und "Critic-Pruef-Kriterien").

## State-Tracker

`/tmp/loop-<repo_slug_safe>-<issue>.json`:

```json
{
  "issue": 5,
  "repo": "dscheinecker-at7media/personal-ai-bot",
  "repo_path": "/home/dominik/mei-bot",
  "branch": null,
  "default_branch": "main",
  "pr_base": "main",
  "loop_type": "code",
  "agents_md_loaded": true,
  "standards": { "branch_pattern": "...", "syntax_check": "...", ... },
  "started_at": "ISO",
  "iterations": 0,
  "stage": "validator",
  "status": "in_progress",
  "verdicts": {}
}
```

`loop_type` ist seit v3.2.0 Pflicht — siehe Loop-Type-Resolution oben.

`repo_slug_safe` = `owner_repo` (Slash → Underscore). Update nach jeder Stage.

## Die 5 Stages

Jede Stage = ein eigener Subagent via Agent-Tool (`general-purpose`). Sequenziell, Output lesen, naechste Stage entscheiden.

Wichtig: **alle Standards-Werte in den Briefings sind Placeholder** (`<branch_pattern>`, `<syntax_check>`, ...). Sie werden zur Laufzeit aus dem AGENTS.md-Cache eingesetzt — keine fixen Defaults mehr im Skill.

### Stage 1 — Spec-Validator

**Briefing (Template):**
> Du validierst Issue #N im Repo `<slug>`. Du baust nichts.
>
> Standards-Kontext: Repo hat AGENTS.md mit `hard_gates: <hard_gates>` und `ac_templates: <ac_templates>`. Body von AGENTS.md:
> ```
> <AGENTS-MD-BODY>
> ```
>
> 1. `gh issue view N --repo <slug>` lesen.
> 2. Validator-Gates pruefen:
>    - [ ] AC sind testbar (jede Box konkret pruefbar)
>    - [ ] AC enthaelt die `ac_templates` aus AGENTS.md
>    - [ ] Files-To-Touch existieren oder sind als neu beschrieben. Pruefe via `get_architecture` aus codebase-memory + `ls` im Repo-Pfad.
>    - [ ] Dependencies erfuellt (`depends on #X` → checke `#X` CLOSED)
>    - [ ] Test-Plan vorhanden
>    - [ ] Out-of-Scope benannt (mind. `default_oos` aus AGENTS.md plus Issue-spezifisch)
>    - [ ] `hard_gates` aus AGENTS.md adressiert
> 3. Bei STOP: konkrete Spec-Verbesserungs-Vorschlaege.
> 4. Output: Kommentar `## [stage:validator] <GO|STOP>` an Issue + Parent-Report (max 100 Woerter).

**Parent-Entscheidung:**
- STOP → Loop pausiert, Telegram-Text an User mit Gruenden + Vorschlag `/create-issue --refine <num>`.
- GO → Stage 2.

### Stage 2 — Implementer (Type-Dispatch seit v3.2.0)

**Brief-Auswahl per Loop-Type:**

| `loop_type` aus Pre-Flight | Brief-File |
|---------------------------|------------|
| `code` (Default) | `references/subagent-briefs/code-implementer.md` |
| `research` | `references/subagent-briefs/research-implementer.md` |

Skill laedt das passende Brief-File, ersetzt Placeholder (`{{branch_pattern}}`, `{{syntax_check}}`, `{{hard_gates}}`, `{{secret_scan_pattern}}`, `{{issue_num}}`, `{{repo_path}}`, `{{slug}}`, `{{default_branch}}`, `{{commit_format}}`) aus AGENTS.md-Cache + State-Tracker, dispatched es als Subagent-Briefing.

**Default-`secret_scan_pattern`** (wenn nicht via Issue-Override gesetzt):
```
(ghp_[A-Za-z0-9]{30,}|sk-ant-[A-Za-z0-9_-]{40,}|TELEGRAM_BOT_TOKEN=[0-9]+:[A-Za-z0-9_-]+|API_KEY=[a-zA-Z0-9]{20,})
```

**Parent-Entscheidung:** Spec-Luecke zu gross → ESCALATE. Sonst Stage 3 (oder Skip-Regel).

### Stage 3 — Tester (Type-spezifische Pruef-Kriterien seit v3.2.0)

**Briefing-Schema (`loop_type: code`):**
> Du verifizierst die AC aus Issue #N. Du baust keinen Code.
>
> Standards: `smoke_test: <smoke_test>` aus AGENTS.md.
>
> 1. `git checkout <branch>`
> 2. `<smoke_test>` ausfuehren.
> 3. Pro AC-Box: Smoke-Test + Beweis (log-snippet, command-output).
> 4. **Cleanup-Pflicht:** Live-Stack/State zuruecksetzen falls angefasst. Cleanup-Status im Comment.
> 5. Issue-Kommentar `## [stage:tester] <PASS|FAIL>` mit pro-AC Status + Beweisen + Cleanup-Status.
>
> Parent-Output: max 200 Woerter.

**Briefing-Schema (`loop_type: research`):**
> Du verifizierst die Findings-Doc-Quality aus Issue #N. Du fuehrst **kein** `smoke_test` aus.
>
> 1. `git checkout <branch>` und Doc finden (`docs/research/<topic>-<date>.md`).
> 2. **Doc-Quality-Check** gegen Issue-AC und Type-Pruef-Liste in
>    `references/subagent-briefs/research-implementer.md` Sektion
>    "Tester-Pruef-Kriterien":
>    - [ ] Wortzahl >= 800 (`wc -w`)
>    - [ ] Test-Matrix-Tabelle vorhanden
>    - [ ] >= 6 Probes dokumentiert
>    - [ ] Working-Setup-Sektion ODER Hypothese-Roadmap-Sektion
>    - [ ] Folge-Implementation-Issue-Spec als Anhang
>    - [ ] Quellen-Liste mit >= 1 Link
>    - [ ] Keine Secrets im Doc (Pattern-Scan)
> 3. Issue-Kommentar `## [stage:tester] <PASS|FAIL>` mit pro-Pruef-Item-Status + Doc-Quote als Beweis.
>
> Parent-Output: max 200 Woerter.

**Skip-Regeln:**
- **`code`-Loop:** Diff nur in `docs/**` ODER `**.md` ODER `// comment`-only → Tester skippen, direkt zu Critic. Markiere `skip_tester_round_N: docs_only` im State.
- **`research`-Loop:** Skip-Regel **deaktiviert** — Doc-Quality-Check ist Pflicht (sonst nicht testbar).

**Parent-Entscheidung:** FAIL → Stage 4 (Critic entscheidet Revise oder Escalate). PASS → Stage 4.

### Stage 4 — Critic

**Briefing (Template):**
> Du bewertest Branch `<branch>` (HEAD `<commit>`) gegen Issue #N.
>
> Standards-Kontext: AGENTS.md-Body + `hard_gates: <hard_gates>`.
>
> 1. Issue-AC + Implementer-Comment + Tester-Comment lesen.
> 2. `git diff <default_branch>..<branch>` lesen.
> 3. Pro AC: `search_code` aus codebase-memory fuer Code-Evidence ("AC sagt X, Code-Snippet zeigt Y") → OK/Warn/Fail.
> 4. Tester-Findings werten: blockierend oder dokumentier-bar?
> 5. Out-of-Scope-Check (`<default_oos>` + Issue-OoS).
> 6. `hard_gates`-Check (siehe AGENTS.md).
> 7. Code-Qualitaet (smoke): Naming, Error-Handling, Cleanup, Security.
>
> Verdict:
> - **APPROVE** — alle AC ok, keine blockierenden Findings, OoS + hard_gates eingehalten.
> - **REVISE** — actionable items (konkret).
> - **ESCALATE** — Spec-Luecke / Architektur-Frage.
>
> Issue-Kommentar `## [stage:critic] <verdict>` mit AC-Review + OoS-Check + Findings-Bewertung + (bei REVISE) Actionable Items.
>
> Parent-Output: max 150 Woerter.

**Parent-Entscheidung:**
- APPROVE → Stage 5.
- REVISE → `iterations++`, Stage 2 erneut mit Critic-Comment als Briefing. **Hard-Cap: 3 Revise.**
- ESCALATE → Pause, Telegram-Text an User.

### Stage 5 — Closer (Type-Routing seit v3.2.0)

**Briefing (Template):**
> 1. PR erstellen mit Body (Summary, verwendete Standards aus AGENTS.md, Test-Plan, "Closes #N", Loop-Engineering-Process-Block). Base: `<pr_base>`.
>    - **`code`-Loop:** PR-Title `<commit_format>`-kompatibel (z.B. `feat(scope): ...`).
>    - **`research`-Loop:** PR-Title `research(<topic>): findings + folge-issue-spec`.
> 2. Squash-Merge auf `<pr_base>` (`--delete-branch`).
> 3. Drift-Check vor Stack-Rebuild: `diff -r <live_path> <repo_path>` → bei Drift Warnung, kein Auto-Rebuild.
> 4. **Type-spezifischer Deploy-Schritt:**
>    - **`code`-Loop:** Lokal pullen + Live-Stack rebuild mit `<deploy_command>` (aus AGENTS.md oder Registry). Health-Check post-deploy.
>    - **`research`-Loop:** Kein Deploy (Doc-only). Optional Folge-Implementation-Issue via `/create-issue --type=code` mit dem Spec-Stub aus dem Doc anlegen.
> 5. **MemPalace-Drawer (Type-Routing):**
>    - **`code`-Loop:** `palace-dominik/wing_code/room_changes` (Repo + PR + Commit + Loop-Summary + Tester-Findings).
>    - **`research`-Loop:** `palace-dominik/wing_personal/room_arbeitsweise_mit_claude` (Loop-Pattern-Erkenntnisse + Doc-Pfad + Folge-Issue-Spec).
> 6. Issue-Kommentar `## [stage:closer] merged & deployed` (oder `merged` fuer Research) mit PR-Nummer + Wallclock + Drawer-ID.
> 7. Loop-State final (`status: "closed"`, `closed_at: ...`).
>
> Parent-Output: max 250 Woerter mit PR-Nummer, Merge-Commit, Deploy-Status (oder n/a fuer Research), Drawer-ID, Wallclock.

**Constraint:** Bei Stack-Crash post-deploy: KEIN automatischer Revert. Melde Parent, Parent entscheidet mit User.

## Hard-Caps

- Max 3 Implementer↔Critic Revise-Runden
- Max Wallclock 90 Min — bei Ueberschreitung Pause + Telegram-Text
- 3x Build-Fail im Tester → ESCALATE

## Telegram / Channel-Updates

Wenn aus `<channel>`-Inbound aufgerufen:
- **Pre-Stage-1:** "Loop fuer #N startet, Stage 1 laeuft"
- **Zwischen Stages:** `Stage N (<name>) → <verdict>. Stage N+1 startet.`
- **Pre-Tester-mit-Live-Ping:** Warnung wenn Test-Plan Live-Channel-Ping erzeugt
- **Post-Closer:** Erfolgs-Report mit PR-Link, Wallclock, Iterations-Count

Channel/chat_id aus Inbound-Meta uebernehmen.

## Issue-Kommentar-Konvention

Jeder Kommentar startet mit `## [stage:<name>] <verdict>` damit Audit-Log scannable bleibt. Subagents kriegen das im Briefing vorgegeben.

## Abgrenzung zu /create-issue und /init-agents

| Skill | Wann |
|-------|------|
| `/init-agents` | Repo hat noch keine AGENTS.md → Bootstrap (einmalig pro Repo) |
| `/create-issue` | Idee → vollstaendig spezifiziertes GitHub-Issue (Genesis-Phase) |
| `/work-issue` | Issue mit Spec → Build-Loop bis Merge (Execution-Phase) |

Wenn `/work-issue` ohne Issue-Nummer aufgerufen wird ODER das Issue keinen Spec hat (kein AC, keine Files-To-Touch) → fragt User: "Issue ist unvollstaendig. Soll ich `/create-issue --refine <num>` aufrufen?"

Wenn AGENTS.md im Repo fehlt → STOP mit Pointer auf `/init-agents` (siehe Pre-Flight-Check oben). Kein Auto-Fallback.

## Was der Skill NICHT macht

- Plant nicht selber neue Issues (das ist `/create-issue`)
- Legt keine AGENTS.md selber an (das ist `/init-agents`)
- Cross-Repo-Dependencies werden nur als CLOSED-Check geprueft, nicht auto-vorher-werken
- Erstellt keine ADRs
- Macht keinen Auto-Revert bei Stack-Crash

## Loop-Engineering-Pattern (Referenz)

Aus Drawer `b0a7a369` (palace-dominik / wing_personal / room_arbeitsweise_mit_claude / 2026-06-21):

| Feld | Hier |
|------|------|
| Trigger | `/work-issue <num> [--repo <slug>]` |
| Iteration | 5 Stages, Revise-Loop bei Critic-REVISE |
| Persistenz | Issue-Kommentare + `/tmp/loop-*.json` + MemPalace-Drawer am Ende |
| Stop-Kriterium | Critic-APPROVE + Closer-Merge + Deploy ok |
| Eskalation | Telegram-Text bei STOP/ESCALATE/Hard-Cap |

## Siehe auch

- `skills/init-agents/SKILL.md` — Bootstrap-Pflicht VOR dem ersten Lauf in einem Repo
- `skills/create-issue/SKILL.md` — Issue-Genesis-Phase
- `skills/work-issue/references/repo-registry.yaml.example`
- `skills/work-issue/references/subagent-briefs/code-implementer.md` — Code-Loop-Brief
- `skills/work-issue/references/subagent-briefs/research-implementer.md` — Research-Loop-Brief
- `skills/init-agents/references/AGENTS.md.template`
- MemPalace-Drawer `b0a7a369` — Loop-Engineering-Pattern (Original)
- Research-Loop-Erprobung — `dscheinecker-at7media/personal-ai-bot#51` (2026-06-26)
