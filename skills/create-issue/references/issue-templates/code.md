---
loop-type: code
---

# Issue template — loop type: `code`

**Used by:** `/create-issue --type=code` (default).

**When:** software implementation tasks — feature, bug fix, refactor, dependency update.

**Deliverable:** PR with code diff + tests, merged to `pr_base` from AGENTS.md.

**Implementer brief:** `skills/work-issue/references/subagent-briefs/code-implementer.md`

---

## Rendered body schema (placeholders with `{{...}}`)

This template is rendered by the skill body (spec dialog). Placeholders are substituted at render time. Required sections are exactly these, in this order — `/work-issue`'s Validator checks them strictly.

```markdown
## Idea (Why)

{{idea}}

## Spec (What)

{{spec}}

## Acceptance Criteria

### from AGENTS.md `ac_templates` (required)
{{ac_templates_block}}

### Hard-Gates
{{hard_gates_block}}

### Issue-specific
{{user_ac_block}}

## Files to Touch

{{files_to_touch_block}}

## Test Plan

{{test_plan}}

## Dependencies / Blocks

{{dependencies_block}}

## Out of Scope

{{out_of_scope_block}}

## Standards Override (optional)

{{standards_override_block}}

## Standards Notes

{{standards_notes_block}}
```

---

## Section details

### `## Idea (Why)`

Strategic context, user pain, outcome. 1-3 paragraphs. Loaded into the Validator briefing.

### `## Spec (What)`

Technical spec, architecture, affected components. Sub-headers (`###`) allowed.

### `## Acceptance Criteria`

Three sub-blocks:
1. **`### from AGENTS.md ac_templates (required)`** — auto-injected from AGENTS.md (mode 1) or override list (mode 3) or empty (mode 2). See `skills/create-issue/SKILL.md` section "Auto-injection".
2. **`### Hard-Gates`** — `hard_gates` from AGENTS.md (only if not disabled by an issue standards override).
3. **`### Issue-specific`** — user ACs from the spec dialog plus type-detection trigger ACs.

Every checkbox: `- [ ] <concrete, testable>`.

### `## Files to Touch`

List with path annotation. Example:

```
- `path/a.ts` — what happens there
- `path/b.ts` — what happens there
```

Sub-headers `### NEW` / `### Changed` allowed for clarity.

### `## Test Plan`

Which smoke tests, which build command, what proves success. Should reflect `smoke_test` from AGENTS.md.

### `## Dependencies / Blocks`

```
- depends on #<X>
- blocks #<Y>
```

### `## Out of Scope`

At minimum `default_oos` from AGENTS.md plus issue-specific.

### `## Standards Override` (optional)

Only insert if the issue diverges from AGENTS.md. YAML block:

```yaml
work-issue:
  syntax_check: "<custom>"
  smoke_test: "<custom>"
```

### `## Standards Notes`

Optional. Reasoning for `DISMISS` on type-detection triggers or `ac_templates` overrides. Read in the Critic briefing.

---

## Issue labels (auto-set)

- `loop-type:code` — required; `/work-issue` uses it to pick the subagent brief.
- `enhancement` / `bug` / `refactor` — per trigger detection (see the skill body).
- Additional labels from type-detection triggers (#1-#6) — see `skills/create-issue/SKILL.md`.

---

## Example (rendered)

```markdown
## Idea (Why)

Voice-diff review for mei: the user sends a voice memo "What changed in X?",
the bot looks up the diff in git history and replies as voice.

## Spec (What)

OllamaProvider extended with a `voiceDiffReview` method. Reads the latest commit
in the mei-bot repo, parses the diff, generates a summary via a model call, TTS
via OpenAI.

## Acceptance Criteria

### from AGENTS.md `ac_templates` (required)
- [ ] Code passes `node --check`
- [ ] Documentation in README/CLAUDE.md updated
- [ ] No new secrets in repo (secret check passed)

### Hard-Gates
- [ ] No direct edits on main
- [ ] All MCP tools need a test example in the doc block

### Issue-specific
- [ ] Voice-memo trigger "diff X" detected
- [ ] TTS response works

## Files to Touch

- `packages/mei-runner/src/providers/ollama.ts` — voiceDiffReview method
- `packages/mei-runner/src/voice/tts.ts` — TTS call

## Test Plan

`docker compose build && docker compose up -d --force-recreate`, then send a
voice memo, receive an OGG response.

## Dependencies / Blocks

- depends on #42
- blocks #51

## Out of Scope

- Multi-user support
- Web-channel approvals
- Models other than qwen2.5:7b
```
