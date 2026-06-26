---
loop-type: research
---

# Issue-Template — Loop-Type: `research`

**Verwendet von:** `/create-issue --type=research`.

**Wann:** Erkenntnis-Generierung mit Test-Matrix — unklare Root-Cause bei Bug, Bibliotheks-Vergleich, API-Verhalten-Erkundung, Benchmark-Studie, Best-Practice-Recherche.

**Deliverable:** `docs/research/<topic>-<date>.md` mit Befund-Tabellen + Empfehlungen + Folge-Implementation-Issue-Spec. KEIN Code-Diff im finalen PR (Debug-Edits werden revertet oder in separatem Throwaway-Branch).

**Implementer-Brief (= Researcher):** `skills/work-issue/references/subagent-briefs/research-implementer.md`

**Pattern-Ursprung:** Erst-Erprobung in `dscheinecker-at7media/personal-ai-bot#51` (2026-06-26).

---

## Rendered-Body-Schema (Placeholder mit `{{...}}`)

Pflicht-Sektionen sind exakt diese, in dieser Reihenfolge. Research-spezifisch sind die Spec-Test-Matrix-Achsen und der Standards-Notes-Block mit `ac_templates: []`-Override.

```markdown
## Idee (Why)

{{idea}}

**Research-Kontext:** unklare Root-Cause / API-Verhalten-Probe / Benchmark-Studie / Best-Practice-Recherche.

Aktuelles Issue ist **Research-only** — Implementer-Subagent ist ein Researcher.
Deliverable ist ein Findings-Dokument das die Spec fuer ein **separates
Implementation-Issue** liefert. Kein Code-Diff in diesem Issue.

## Spec (What)

Research-Subagent recherchiert {{research_topic}} und produziert ein
**Findings-Dokument** in `docs/research/{{topic_slug}}-{{date}}.md`. Dokument enthaelt:

### 1. Reproduktions-Tests / Direct-Probes

{{direct_probe_block}}

### 2. Vergleich {{baseline}} vs {{candidate}}

{{comparison_block}}

### 3. Test-Matrix-Achsen (Pflicht: mindestens 2 Achsen)

{{test_matrix_axes}}

### 4. Findings-Dokument-Inhalt

- **Befund pro Achse** (Tabelle: <Achse-X> x <Achse-Y> -> Ergebnis, Latenz, Notizen)
- **Root-Cause** / Hauptbefund
- **Konkrete Empfehlungen** mit Code-Skelett (kein Diff, nur Skizze)
- **Spec fuer Folge-Implementation-Issue** (Title, AC-Skelett, Files-To-Touch) als Anhang

## Akzeptanzkriterien

### aus AGENTS.md `ac_templates` (Pflicht via Standards-Override deaktiviert)
Siehe `## Standards-Notes` — `ac_templates: []` weil docs-only.

### Research-spezifisch (Pflicht)
- [ ] **Findings-Dokument** existiert in `docs/research/{{topic_slug}}-{{date}}.md` (>=800 Woerter)
- [ ] Mindestens **6 verschiedene** Direct-Tests/Probes durchgefuehrt mit Roh-Output dokumentiert
- [ ] **Test-Matrix-Tabelle** (mindestens 2 Achsen) im Dokument
- [ ] Mindestens ein **funktionierender Setup** gefunden — wenn nicht: dokumentiert warum nicht + welche **Hypothese-Roadmap** als naechstes pruefbar waere
- [ ] **Konkrete Empfehlungen** mit Code-Skelett (kein Diff, nur Skizze)
- [ ] **Folge-Implementation-Issue-Spec** (Title, AC, Files-To-Touch) als Anhang im Dokument
- [ ] **Quellen-Liste** im Dokument (Links zu Docs/Specs/Beispielen die der Researcher konsultiert hat)

### Hard-Gates (gelten weiterhin)
{{hard_gates_block}}

### Issue-spezifisch
{{user_ac_block}}

## Files to Touch

### NEU (Primary Deliverable)
- `docs/research/{{topic_slug}}-{{date}}.md` — primary deliverable

### Temp / NICHT zum Mergen
- {{temp_debug_files_block}}
  — Researcher entscheidet, ob diese Edits im Branch landen (revertet vor PR) oder in einem separaten Throwaway-Branch der nie gemerged wird.

## Test-Plan

1. {{baseline_setup_check}} laeuft healthy (Pre-Flight)
2. Researcher fuehrt Direct-Probes durch (siehe Spec Sektion 1)
3. Researcher fuellt Test-Matrix (siehe Spec Sektion 3)
4. Findings-Dokument wird committed (NUR das Doc, keine Debug-Edits)
5. **Tester** verifiziert: Dokument existiert, >=6 Probes dokumentiert, Matrix-Tabelle vorhanden, funktionierender Setup ODER klare Hypothese-Roadmap, Folge-Issue-Spec startbar
6. **Critic** prueft: Empfehlung ist konkret + actionable, Folge-Issue-Spec ist startbar, Quellen sind belastbar
7. **Closer** mergt Research-Findings als Doc-PR

## Dependencies / Blocks

{{dependencies_block}}

## Out of Scope

- **Implementation-Code** fuer die Aenderungen — kommt in ein separates Folge-Issue basierend auf den Findings (siehe Spec Sektion 4)
- {{additional_out_of_scope}}

## Standards-Override

```yaml
work-issue:
  ac_templates: []   # docs-only, ersetzt via Research-spezifische AC oben
```

## Standards-Notes

**Research-Issue (Implementer = Researcher):** Loop-Engineering-Pattern auf
Erkenntnis-Generierungs-Task statt Coding-Task. Implementer-Subagent erhaelt
Research-Brief statt Code-Brief. AGENTS.md `ac_templates` (typischerweise
syntax-check + smoke-test + secret-scan) sind hier nur teilweise relevant — der
primary Deliverable ist ein Markdown-Dokument, kein Code-Diff. Daher die explizite
Standards-Override mit `ac_templates: []` plus Research-spezifischer Ersatz-AC.

**Debug-Edits-Policy:** Researcher hat code-edit rights fuer **DEBUG-Logging** in
Provider-/Bridge-/Service-Files, aber NUR um Request-/Response-Bodies zu loggen.
Diese Edits werden **NICHT** in den finalen PR gemerged — entweder vor Commit
revertet oder in einem Throwaway-Branch isoliert.

**Erfolgs-Kriterium:** funktionierender Setup gefunden ODER klare empirische
Hypothese welche als naechstes pruefbar waere. "Bin nicht weitergekommen" ist KEIN
zulaessiges Ergebnis — Researcher muss die Sackgassen + den naechsten Probe-Schritt
dokumentieren.

---

## Issue-Labels (automatisch gesetzt)

- `loop-type:research` — Pflicht, wird von `/work-issue` zur Subagent-Brief-Auswahl benutzt.
- `research` / `enhancement` — je nach Spec.

---

## Beispiel (gerendert, leicht gekuerzt)

Siehe `dscheinecker-at7media/personal-ai-bot#51` als Referenz-Implementierung.

Kern-Eckdaten dort:
- Topic: ollama-tool-use-debug
- Doc: `docs/research/ollama-tool-use-debug-2026-06-26.md`
- Test-Matrix-Achsen: Modelle x Tool-Naming x Schema-Verbosity
- Direct-Probes: `curl` gegen `http://localhost:11434/api/chat`
- Debug-Edits: temp DEBUG-Log in `packages/mei-runner/src/providers/ollama.ts`,
  vor Commit revertet
- Folge-Issue-Spec im Anhang: Title "feat(provider): OllamaProvider request-body
  refactor for tool-calls"
