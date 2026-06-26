---
loop-type: code
---

# Issue-Template — Loop-Type: `code`

**Verwendet von:** `/create-issue --type=code` (default).

**Wann:** Software-Implementation-Tasks — Feature, Bug-Fix, Refactoring, Dependency-Update.

**Deliverable:** PR mit Code-Diff + Tests, gemerged auf `pr_base` aus AGENTS.md.

**Implementer-Brief:** `skills/work-issue/references/subagent-briefs/code-implementer.md`

---

## Rendered-Body-Schema (Placeholder mit `{{...}}`)

Dieses Template wird vom Skill-Body (Spec-Dialog) gerendert. Placeholder werden zur Render-Zeit ersetzt. Pflicht-Sektionen sind exakt diese, in dieser Reihenfolge — `/work-issue`s Validator pruefen sie strict.

```markdown
## Idee (Why)

{{idea}}

## Spec (What)

{{spec}}

## Akzeptanzkriterien

### aus AGENTS.md `ac_templates` (Pflicht)
{{ac_templates_block}}

### Hard-Gates
{{hard_gates_block}}

### Issue-spezifisch
{{user_ac_block}}

## Files to Touch

{{files_to_touch_block}}

## Test-Plan

{{test_plan}}

## Dependencies / Blocks

{{dependencies_block}}

## Out of Scope

{{out_of_scope_block}}

## Standards-Override (optional)

{{standards_override_block}}

## Standards-Notes

{{standards_notes_block}}
```

---

## Sektions-Details

### `## Idee (Why)`

Strategischer Kontext, User-Pain, Outcome. 1-3 Absaetze. Wird im Validator-Briefing geladen.

### `## Spec (What)`

Technische Spec, Architektur, betroffene Komponenten. Sub-Headers (`###`) erlaubt.

### `## Akzeptanzkriterien`

Drei Sub-Blocks:
1. **`### aus AGENTS.md ac_templates (Pflicht)`** — auto-injiziert aus AGENTS.md (Modus 1) oder Override-Liste (Modus 3) oder leer (Modus 2). Siehe `skills/create-issue/SKILL.md` Sektion "Auto-Injection".
2. **`### Hard-Gates`** — `hard_gates` aus AGENTS.md (nur wenn nicht durch Issue-Standards-Override deaktiviert).
3. **`### Issue-spezifisch`** — User-AC aus Spec-Dialog plus Type-Detection-Trigger-AC.

Jede Box: `- [ ] <konkret, testbar>`.

### `## Files to Touch`

Liste mit Pfad-Annotation. Beispiel:

```
- `path/a.ts` — was passiert dort
- `path/b.ts` — was passiert dort
```

Sub-Headers `### NEU` / `### Geaendert` erlaubt fuer Klarheit.

### `## Test-Plan`

Welche Smoke-Tests, welcher Build-Command, was beweist Success. Sollte `smoke_test` aus AGENTS.md reflektieren.

### `## Dependencies / Blocks`

```
- depends on #<X>
- blocks #<Y>
```

### `## Out of Scope`

Mindestens `default_oos` aus AGENTS.md plus Issue-spezifisch.

### `## Standards-Override` (optional)

Nur einfuegen wenn Issue von AGENTS.md abweicht. YAML-Block:

```yaml
work-issue:
  syntax_check: "<custom>"
  smoke_test: "<custom>"
```

### `## Standards-Notes`

Optional. Begruendungen fuer `DISMISS` von Type-Detection-Triggern oder `ac_templates`-Overrides. Wird im Critic-Briefing gelesen.

---

## Issue-Labels (automatisch gesetzt)

- `loop-type:code` — Pflicht, wird von `/work-issue` zur Subagent-Brief-Auswahl benutzt.
- `enhancement` / `bug` / `refactor` — je nach Trigger-Detection (siehe Skill-Body).
- Weitere Labels aus Type-Detection-Trigger (#1-#6) — siehe `skills/create-issue/SKILL.md`.

---

## Beispiel (gerendert)

```markdown
## Idee (Why)

Voice-Diff-Review fuer mei: User sendet Voice-Memo "Was hat sich an X geaendert?",
Bot sucht Diff in Git-History und antwortet als Voice.

## Spec (What)

OllamaProvider erweitert um `voiceDiffReview` Method. Liest letzten Commit in
Mei-Bot-Repo, parsed Diff, generiert Summary via Modell-Call, TTS via OpenAI.

## Akzeptanzkriterien

### aus AGENTS.md `ac_templates` (Pflicht)
- [ ] Code passes `node --check`
- [ ] Documentation in README/CLAUDE.md aktualisiert
- [ ] No new secrets in repo (Secret-Check passed)

### Hard-Gates
- [ ] Kein direkter Edit auf main
- [ ] Alle MCP-Tools brauchen ein Test-Beispiel im Doc-Block

### Issue-spezifisch
- [ ] Voice-Memo Trigger "Diff X" erkannt
- [ ] TTS-Response funktioniert

## Files to Touch

- `packages/mei-runner/src/providers/ollama.ts` — voiceDiffReview Method
- `packages/mei-runner/src/voice/tts.ts` — TTS-Call

## Test-Plan

`docker compose build && docker compose up -d --force-recreate`, dann Voice-Memo
senden, OGG-Response zurueckbekommen.

## Dependencies / Blocks

- depends on #42
- blocks #51

## Out of Scope

- Multi-User-Support
- Web-Channel-Approvals
- Andere Modelle als qwen2.5:7b
```
