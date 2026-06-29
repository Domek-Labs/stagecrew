---
loop-type: research
---

# Issue template — loop type: `research`

**Used by:** `/create-issue --type=research`.

**When:** knowledge generation with a test matrix — unclear root cause on a bug, library comparison, API behavior probe, benchmark study, best-practice research.

**Deliverable:** `docs/research/<topic>-<date>.md` with finding tables + recommendations + follow-up implementation-issue spec. NO code diff in the final PR (debug edits are reverted or kept in a separate throwaway branch).

**Implementer brief (= Researcher):** `skills/work-issue/references/subagent-briefs/research-implementer.md`

**Pattern origin:** first end-to-end test in an internal bot project (2026-06-26).

---

## Rendered body schema (placeholders with `{{...}}`)

Required sections are exactly these, in this order. Research-specific parts are the spec test-matrix axes and the standards-notes block with the `ac_templates: []` override.

```markdown
## Idea (Why)

{{idea}}

**Research context:** unclear root cause / API behavior probe / benchmark study / best-practice research.

This issue is **research-only** — the Implementer subagent is a Researcher.
The deliverable is a findings document that supplies the spec for a **separate
implementation issue**. No code diff in this issue.

## Spec (What)

The research subagent investigates {{research_topic}} and produces a
**findings document** at `docs/research/{{topic_slug}}-{{date}}.md`. The document contains:

### 1. Reproduction tests / direct probes

{{direct_probe_block}}

### 2. Comparison {{baseline}} vs {{candidate}}

{{comparison_block}}

### 3. Test-matrix axes (required: at least 2 axes)

{{test_matrix_axes}}

### 4. Findings document content

- **Finding per axis** (table: <axis X> x <axis Y> → result, latency, notes)
- **Root cause** / main finding
- **Concrete recommendations** with code sketch (no diff, just a sketch)
- **Spec for the follow-up implementation issue** (title, AC skeleton, files-to-touch) as appendix

## Acceptance Criteria

### from AGENTS.md `ac_templates` (required, disabled via standards override)
See `## Standards Notes` — `ac_templates: []` because docs-only.

### Research-specific (required)
- [ ] **Findings document** exists at `docs/research/{{topic_slug}}-{{date}}.md` (>=800 words)
- [ ] At least **6 distinct** direct tests/probes performed with raw output documented
- [ ] **Test-matrix table** (at least 2 axes) in the document
- [ ] At least one **working setup** found — if not: documented why not + which **hypothesis roadmap** to probe next
- [ ] **Concrete recommendations** with code sketch (no diff, just a sketch)
- [ ] **Follow-up implementation-issue spec** (title, AC, files-to-touch) appended to the document
- [ ] **Sources list** in the document (links to docs/specs/examples consulted by the researcher)

### Hard-Gates (still apply)
{{hard_gates_block}}

### Issue-specific
{{user_ac_block}}

## Files to Touch

### NEW (primary deliverable)
- `docs/research/{{topic_slug}}-{{date}}.md` — primary deliverable

### Temp / NOT for merging
- {{temp_debug_files_block}}
  — The researcher decides whether these edits land in the branch (reverted before PR) or in a separate throwaway branch that is never merged.

## Test Plan

1. {{baseline_setup_check}} runs healthy (pre-flight)
2. Researcher runs direct probes (see spec section 1)
3. Researcher fills the test matrix (see spec section 3)
4. Findings document is committed (ONLY the doc, no debug edits)
5. **Tester** verifies: document exists, >=6 probes documented, matrix table present, working setup OR clear hypothesis roadmap, follow-up issue spec ready to start
6. **Critic** reviews: recommendation is concrete + actionable, follow-up issue spec is ready to start, sources are reliable
7. **Closer** merges the research findings as a doc PR

## Dependencies / Blocks

{{dependencies_block}}

## Out of Scope

- **Implementation code** for the changes — lands in a separate follow-up issue based on the findings (see spec section 4)
- {{additional_out_of_scope}}

## Standards Override

```yaml
work-issue:
  ac_templates: []   # docs-only, replaced via the research-specific ACs above
```

## Standards Notes

**Research issue (Implementer = Researcher):** loop-engineering pattern applied to
a knowledge-generation task instead of a coding task. The Implementer subagent receives
a research brief instead of a code brief. AGENTS.md `ac_templates` (typically
syntax check + smoke test + secret scan) are only partially relevant here — the
primary deliverable is a Markdown document, not a code diff. Hence the explicit
standards override with `ac_templates: []` plus research-specific replacement ACs.

**Debug-edits policy:** the researcher has code-edit rights for **debug logging** in
provider/bridge/service files, but ONLY to log request/response bodies. These edits
**must NOT** land in the final PR — either reverted before commit or isolated in a
throwaway branch.

**Success criterion:** working setup found OR a clear empirical hypothesis to probe
next. "I did not make progress" is NOT an acceptable result — the researcher has to
document the dead ends + the next probe step.

---

## Issue labels (auto-set)

- `loop-type:research` — required; `/work-issue` uses it to pick the subagent brief.
- `research` / `enhancement` — depending on the spec.

---

## Example (rendered, slightly shortened)

The first end-to-end research loop ran in an internal bot project (2026-06-26).

Key facts there:
- Topic: ollama-tool-use-debug
- Doc: `docs/research/ollama-tool-use-debug-2026-06-26.md`
- Test-matrix axes: models x tool naming x schema verbosity
- Direct probes: `curl` against `http://localhost:11434/api/chat`
- Debug edits: temp DEBUG log in a provider source file, reverted before commit
- Follow-up issue spec in the appendix: title "feat(provider): OllamaProvider request-body
  refactor for tool-calls"
