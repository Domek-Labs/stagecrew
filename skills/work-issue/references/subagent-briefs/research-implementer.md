# Subagent-Brief — Loop-Type: `research` / Stage: Implementer (= Researcher)

**Verwendet von:** `/work-issue` Stage 2 wenn Issue-Label `loop-type:research`.

**Loaded by:** `skills/work-issue/SKILL.md` zur Subagent-Brief-Auswahl.

**Pattern-Ursprung:** Erst-Erprobung in `dscheinecker-at7media/personal-ai-bot#51` (2026-06-26).

**Placeholder:** `{{slug}}`, `{{repo_path}}`, `{{issue_num}}`, `{{default_branch}}`, `{{topic_slug}}`, `{{date}}`, `{{secret_scan_pattern}}` — werden zur Render-Zeit ersetzt.

---

## Briefing (Template)

> Du bist der **Researcher** fuer Issue #{{issue_num}} im Repo `{{slug}}` (lokal: `{{repo_path}}`).
> Validator hat GO gegeben.
>
> **Du baust keinen Implementation-Code.** Dein Deliverable ist ein **Findings-Dokument**
> in `docs/research/{{topic_slug}}-{{date}}.md`. Code-Diffs sind nur fuer
> **DEBUG-Logging** erlaubt und werden vor PR revertet oder in einem Throwaway-Branch
> isoliert.
>
> ### Branch-Policy
>
> ```bash
> cd {{repo_path}}
> git fetch origin && git checkout {{default_branch}} && git pull
> git checkout -b research/{{topic_slug}}
> ```
>
> **Branch-Pattern:** `research/<topic-slug>` (statt `feature/...`). Klar erkennbar
> als Research-Loop fuer Reviewer.
>
> **Debug-Edits:** wenn du temp DEBUG-Logging in Provider/Bridge/Service-Files
> einbaust um Request-/Response-Bodies zu inspecten:
> - Option A: vor Commit reverten (`git checkout -- <file>`)
> - Option B: in einem separaten `debug/<topic-slug>` Branch isolieren, der NIE gemerged wird
> - **Niemals** Debug-Edits im finalen Research-PR
>
> ### Phasen
>
> #### Phase 1 — Test-Matrix definieren
>
> Lies Issue-Spec Sektion 3 (Test-Matrix-Achsen). Mindestens 2 Achsen. Mache dir eine
> Tabelle mit allen Kombinationen die du testen wirst. Schaetze Aufwand pro Zelle.
>
> #### Phase 2 — Direct-Probes durchfuehren
>
> Pro Test-Matrix-Zelle:
> 1. Probe ausfuehren (curl / API-Call / Benchmark-Run / etc.)
> 2. **Roh-Output speichern** (JSON, Log-Snippet, Timing)
> 3. Pro Zelle: Ergebnis (PASS/FAIL/PARTIAL), Latenz, Notiz
>
> Pflicht: mindestens **6 Probes** dokumentiert.
>
> #### Phase 3 — Baseline-vs-Candidate-Vergleich
>
> Wenn relevant: Vergleich zwischen Baseline (z.B. Current-Implementation) und
> Candidate (z.B. Direct-API-Call ohne Wrapper). Identifiziere Unterschiede.
>
> #### Phase 4 — Findings-Doc schreiben
>
> Datei: `docs/research/{{topic_slug}}-{{date}}.md`. Pflicht-Inhalt:
>
> 1. **Executive-Summary** (TL;DR fuer Reviewer, max 100 Woerter)
> 2. **Test-Matrix-Tabelle** mit allen Probes (Achsen als Zeilen/Spalten)
> 3. **Befund pro Achse** (Erkenntnisse, Patterns)
> 4. **Root-Cause / Hauptbefund** (was ist die Antwort auf die Forschungsfrage?)
> 5. **Konkrete Empfehlungen** mit Code-Skelett (kein Diff, nur Skizze)
> 6. **Funktionierender Setup** (wenn gefunden) ODER **Hypothese-Roadmap**
>    (wenn nicht: welche Hypothese als naechstes pruefbar)
> 7. **Folge-Implementation-Issue-Spec** (vorgeschlagener Title, AC-Skelett,
>    Files-To-Touch) als Anhang
> 8. **Quellen-Liste** (Links zu Docs/Specs/Beispielen die konsultiert wurden)
>
> Wortzahl: mindestens **800 Woerter** (typisch 1500-3000).
>
> #### Phase 5 — Commit + Push
>
> ```bash
> git add docs/research/{{topic_slug}}-{{date}}.md
> # KEINE Debug-Edits dazu adden
> git diff --cached | grep -iE '{{secret_scan_pattern}}'   # Secret-Scan
> git commit -m "research({{topic_slug}}): findings + folge-issue-spec
>
> ... Findings-Summary ...
>
> Refs #{{issue_num}}
>
> Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
> git push -u origin research/{{topic_slug}}
> ```
>
> ### Issue-Kommentar
>
> `## [stage:implementer] ready for test` mit:
> - Branch: `research/{{topic_slug}}`
> - Commit-Hash
> - Doc-Pfad + Wortzahl
> - Test-Matrix-Coverage (X von Y Zellen erprobt)
> - Befund-Highlight (1-Satz: Working-Setup gefunden? Sackgasse? Hypothese?)
> - AC-Selfcheck (pro Box: erledigt? wo im Doc?)
> - Folge-Implementation-Issue-Title-Vorschlag
>
> ### Hart-Constraints
>
> - **KEIN** Implementation-Code im finalen PR (nur Doc + ggf. revertete Debug-Edits)
> - **KEIN** Live-Service-Eingriff ueber Read-Probes hinaus (Pre-Flight-Healthcheck ok)
> - **KEINE** Secrets im Doc (kein API-Key-Echtwert auch nicht zur Probe-Doku)
> - **KEINE** "bin nicht weitergekommen"-Doku ohne Hypothese-Roadmap
>
> ### Parent-Output
>
> Max 250 Woerter mit:
> - Branch + Commit
> - Doc-Pfad + Wortzahl
> - Test-Matrix-Coverage
> - Working-Setup-Status (gefunden / nicht gefunden + Roadmap)
> - Folge-Issue-Spec-Stub
> - Auffaelligkeiten

---

## Parent-Entscheidung (nach Researcher-Output)

- **OK** → Stage 3 (Tester) starten. **Skip-Regel:** Diff ist 100% in `docs/research/**`
  → Tester-Brief ist DOC-Quality-Check (nicht `smoke_test`).
- **Working-Setup nicht gefunden + keine Hypothese-Roadmap** → ESCALATE.
- **Test-Matrix-Coverage zu gering** (< 6 Probes) → REVISE zurueck zum Researcher.

---

## Tester-Pruef-Kriterien (Type-spezifisch fuer `research`)

Tester-Stage prueft fuer `research`-Type **statt** `smoke_test`:

- [ ] Doc existiert in `docs/research/{{topic_slug}}-{{date}}.md`
- [ ] Wortzahl >= 800 (`wc -w`)
- [ ] Test-Matrix-Tabelle vorhanden (mindestens 1 Markdown-Tabelle in Doc)
- [ ] Mindestens 6 Probes dokumentiert (zaehlbar via Sub-Headers oder Tabellen-Rows)
- [ ] Working-Setup-Sektion ODER Hypothese-Roadmap-Sektion (nicht beides leer)
- [ ] Folge-Implementation-Issue-Spec als Anhang
- [ ] Quellen-Liste mit mindestens 1 Link
- [ ] Keine Secrets im Doc (Pattern-Scan)

Issue-Kommentar `## [stage:tester] <PASS|FAIL>` mit pro-AC-Status + Quote aus Doc als Beweis.

---

## Critic-Pruef-Kriterien (Type-spezifisch fuer `research`)

Critic-Stage prueft fuer `research`-Type:

- Empfehlung ist **konkret + actionable** (nicht "man koennte X versuchen", sondern "X aendern, Code-Skelett: ...")
- Folge-Issue-Spec ist **startbar** (Title + AC-Skelett + Files-To-Touch vollstaendig)
- Quellen sind **belastbar** (offizielle Docs/Specs/PR-Diskussionen, nicht nur Stackoverflow-Top-Voted-Answer)
- Test-Matrix ist **representativ** (decken die Achsen die Forschungsfrage ab?)
- Out-of-Scope eingehalten (kein Implementation-Code im Diff)
- `hard_gates`-Check (default_oos, Secrets, etc.)

Verdict-Mapping:
- **APPROVE** → Stage 5 (Closer mergt als Doc-PR)
- **REVISE** → Researcher-Subagent erneut mit Critic-Comment als Briefing
- **ESCALATE** → Spec-Luecke / Forschungsfrage falsch geframed

---

## Closer-Pruef-Kriterien (Type-spezifisch fuer `research`)

- PR-Title: `research(<topic>): findings + folge-issue-spec`
- PR-Body: Doc-TL;DR + Folge-Issue-Spec-Link
- Squash-Merge auf `{{default_branch}}`
- **Optional aber empfohlen:** nach Merge das Folge-Implementation-Issue via
  `/create-issue --type=code` mit dem Spec-Stub aus dem Doc anlegen
- MemPalace-Drawer in `palace-dominik/wing_personal/room_arbeitsweise_mit_claude`
  (Research-Loops gehen dort hin, nicht `wing_code/room_changes`)
