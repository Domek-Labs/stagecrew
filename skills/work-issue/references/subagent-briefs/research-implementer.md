# Subagent brief — loop type: `research` / stage: Implementer (= Researcher)

**Used by:** `/work-issue` stage 2 when the issue label is `loop-type:research`.

**Loaded by:** `skills/work-issue/SKILL.md` for the subagent-brief selection.

**Pattern origin:** first end-to-end test in an internal bot project (2026-06-26).

**Placeholders:** `{{slug}}`, `{{repo_path}}`, `{{issue_num}}`, `{{default_branch}}`, `{{topic_slug}}`, `{{date}}`, `{{secret_scan_pattern}}`, `{{commit_identity}}`, `{{git_remote}}` — substituted at render time. `{{commit_identity}}` is `null` when the optional `commit_identity:` block is absent from AGENTS.md. `{{git_remote}}` is the GitHub working remote from the repo registry (`git_remote` field); it falls back to `origin` when the registry does not declare one.

---

## Briefing (template)

> You are the **Researcher** for issue #{{issue_num}} in repo `{{slug}}` (local: `{{repo_path}}`).
> The Validator returned GO.
>
> **You write no implementation code.** Your deliverable is a **findings document**
> at `docs/research/{{topic_slug}}-{{date}}.md`. Code diffs are only allowed for
> **debug logging** and are reverted before PR or isolated in a throwaway branch.
>
> ### Branch policy
>
> ```bash
> cd {{repo_path}}
> git fetch {{git_remote}} && git checkout {{default_branch}} && git pull
> git checkout -b research/{{topic_slug}}
> ```
>
> **Branch pattern:** `research/<topic-slug>` (instead of `feature/...`). Clearly identifies
> a research loop to the reviewer.
>
> **Debug edits:** if you add temp DEBUG logging in provider/bridge/service files
> to inspect request/response bodies:
> - Option A: revert before commit (`git checkout -- <file>`)
> - Option B: isolate in a separate `debug/<topic-slug>` branch that is NEVER merged
> - **Never** ship debug edits in the final research PR
>
> ### Phases
>
> #### Phase 1 — define the test matrix
>
> Read issue spec section 3 (test-matrix axes). At least 2 axes. Build a table of
> all combinations you will test. Estimate effort per cell.
>
> #### Phase 2 — run direct probes
>
> Per test-matrix cell:
> 1. Run the probe (curl / API call / benchmark run / etc.)
> 2. **Save raw output** (JSON, log snippet, timing)
> 3. Per cell: result (PASS/FAIL/PARTIAL), latency, note
>
> Required: at least **6 probes** documented.
>
> #### Phase 3 — baseline-vs-candidate comparison
>
> If relevant: comparison between baseline (e.g., current implementation) and
> candidate (e.g., direct API call without wrapper). Identify differences.
>
> #### Phase 4 — write the findings doc
>
> File: `docs/research/{{topic_slug}}-{{date}}.md`. Required content:
>
> 1. **Executive summary** (TL;DR for reviewers, max 100 words)
> 2. **Test-matrix table** with all probes (axes as rows/columns)
> 3. **Finding per axis** (insights, patterns)
> 4. **Root cause / main finding** (what is the answer to the research question?)
> 5. **Concrete recommendations** with code sketch (no diff, just a sketch)
> 6. **Working setup** (if found) OR **hypothesis roadmap**
>    (if not: which hypothesis to probe next)
> 7. **Follow-up implementation-issue spec** (proposed title, AC skeleton,
>    files-to-touch) as an appendix
> 8. **Sources list** (links to docs/specs/examples consulted)
>
> Word count: at least **800 words** (typically 1500-3000).
>
> #### Phase 5 — commit + push
>
> **Set the commit identity first** — if `{{commit_identity}}` is set (not `null`), run before committing:
> ```bash
> git config user.name "<commit_identity.name>"
> git config user.email "<commit_identity.email>"
> ```
> Never author under a company/shared email. If `{{commit_identity}}` is `null`, use the ambient `git config`.
>
> ```bash
> git add docs/research/{{topic_slug}}-{{date}}.md
> # do NOT add debug edits
> git diff --cached | grep -iE '{{secret_scan_pattern}}'   # secret scan
> git commit -m "research({{topic_slug}}): findings + follow-up issue spec
>
> ... findings summary ...
>
> Refs #{{issue_num}}"
> git push -u {{git_remote}} research/{{topic_slug}}
> ```
>
> **No `Co-authored-by:` trailer** and no bot footer (honor the `no_unconfigured_coauthors` hard-gate) — unless AGENTS.md explicitly configures one. GitHub appends co-author lines from the individual commits on squash-merge, so the commit must already be clean.
>
> ### Issue comment
>
> `## [stage:implementer] ready for test` with:
> - Branch: `research/{{topic_slug}}`
> - Commit hash
> - Doc path + word count
> - Test-matrix coverage (X of Y cells probed)
> - Finding highlight (one sentence: working setup found? dead end? hypothesis?)
> - AC selfcheck (per checkbox: done? where in the doc?)
> - Suggested follow-up implementation-issue title
>
> ### Hard constraints
>
> - **NO** implementation code in the final PR (only the doc + possibly reverted debug edits)
> - **NO** live-service intervention beyond read-probes (pre-flight healthcheck OK)
> - **NO** secrets in the doc (no real API key value, even for probe documentation)
> - **NO** "I did not make progress" docs without a hypothesis roadmap
>
> ### Parent output
>
> Max 250 words with:
> - Branch + commit
> - Doc path + word count
> - Test-matrix coverage
> - Working-setup status (found / not found + roadmap)
> - Follow-up issue spec stub
> - Anomalies

---

## Parent decision (after Researcher output)

- **OK** → start stage 3 (Tester). **Skip rule:** diff is 100% in `docs/research/**`
  → the Tester brief is a doc-quality check (not `smoke_test`).
- **Working setup not found + no hypothesis roadmap** → ESCALATE.
- **Test-matrix coverage too low** (< 6 probes) → REVISE back to the Researcher.

---

## Tester check criteria (type-specific for `research`)

The Tester stage checks for the `research` type **instead of** `smoke_test`:

- [ ] Doc exists at `docs/research/{{topic_slug}}-{{date}}.md`
- [ ] Word count >= 800 (`wc -w`)
- [ ] Test-matrix table present (at least 1 Markdown table in the doc)
- [ ] At least 6 probes documented (countable via sub-headers or table rows)
- [ ] Working-setup section OR hypothesis-roadmap section (neither empty)
- [ ] Follow-up implementation-issue spec attached
- [ ] Sources list with at least 1 link
- [ ] No secrets in the doc (pattern scan)

Issue comment `## [stage:tester] <PASS|FAIL>` with per-AC status + a doc quote as evidence.

---

## Critic check criteria (type-specific for `research`)

The Critic stage checks for the `research` type:

- Recommendation is **concrete + actionable** (not "one could try X", but "change X, code sketch: ...")
- Follow-up issue spec is **ready to start** (title + AC skeleton + files-to-touch complete)
- Sources are **reliable** (official docs/specs/PR discussions, not just a top-voted Stack Overflow answer)
- Test matrix is **representative** (do the axes cover the research question?)
- Out-of-scope respected (no implementation code in the diff)
- `hard_gates` check (default_oos, secrets, etc.)

Verdict mapping:
- **APPROVE** → stage 5 (Closer merges as a doc PR)
- **REVISE** → researcher subagent again with the Critic comment as briefing
- **ESCALATE** → spec gap / research question framed wrong

---

## Closer check criteria (type-specific for `research`)

- PR title: `research(<topic>): findings + follow-up issue spec`
- PR body: doc TL;DR + follow-up issue-spec link
- Squash-merge to `{{default_branch}}`
- **Optional but recommended:** after merge, file the follow-up implementation issue via
  `/create-issue --type=code` with the spec stub from the doc
- **(Optional) persist a loop summary** in your knowledge system. If you use
  [MemPalace](https://github.com/MemPalace/mempalace), file a drawer at a research-loop
  location (e.g. `<your-palace>/<your-personal-wing>/<your-process-room>`) so future
  loops can recall what worked. Skip this step if your team uses a different knowledge
  store (or none).
