# Subagent brief — loop type: `code` / stage: Implementer

**Used by:** `/work-issue` stage 2 when the issue label is `loop-type:code` (or by default fallback).

**Loaded by:** `skills/work-issue/SKILL.md` for the subagent-brief selection.

**Placeholders:** `{{branch_pattern}}`, `{{commit_format}}`, `{{syntax_check}}`, `{{hard_gates}}`, `{{default_branch}}`, `{{repo_path}}`, `{{slug}}`, `{{issue_num}}`, `{{secret_scan_pattern}}`, `{{components}}` — substituted at render time from the AGENTS.md cache + state tracker. `{{components}}` is `null` when the optional `components:` block is absent from AGENTS.md.

---

## Briefing (template)

> You are implementing issue #{{issue_num}} in repo `{{slug}}` (local: `{{repo_path}}`).
> The Validator returned GO.
>
> **Standards from AGENTS.md / issue override (L1+L2):**
> - `branch_pattern`: `{{branch_pattern}}`
> - `default_branch`: `{{default_branch}}`
> - `commit_format`: `{{commit_format}}`
> - `syntax_check`: `{{syntax_check}}`
> - `hard_gates`: `{{hard_gates}}`
>
> ### Steps
>
> 1. **Branch setup**
>    ```bash
>    cd {{repo_path}}
>    git fetch origin && git checkout {{default_branch}} && git pull
>    git checkout -b <branch-resolved-via-pattern>
>    ```
>    Derive the branch name from `{{branch_pattern}}` (e.g., `feature/voice-diff-review`).
>
> 2. **Check code conventions via codebase-memory**
>    ```
>    search_code(<keyword>) -> sibling functions + style conventions
>    ```
>    Understand how similar features are structured in the repo.
>
> 3. **Implement to spec**
>    Strict against the issue ACs. Every AC checkbox must be covered by your diff at the end.
>
> 4. **Syntax check**
>    ```bash
>    {{syntax_check}}
>    ```
>    Failure = STOP, no commit. Fix the syntax first.
>
> 5. **Secret scan before commit**
>    ```bash
>    git add -A
>    git diff --cached | grep -iE '{{secret_scan_pattern}}'
>    ```
>    Default pattern if not overridden:
>    `(ghp_[A-Za-z0-9]{30,}|sk-ant-[A-Za-z0-9_-]{40,}|TELEGRAM_BOT_TOKEN=[0-9]+:[A-Za-z0-9_-]+|API_KEY=[a-zA-Z0-9]{20,})`
>
>    Match = **ABORT** with a clear hint message. No commit.
>
> 6. **Commit**
>    Format per `{{commit_format}}` (typically `conventional`):
>    ```
>    <type>(<scope>): <short summary>
>
>    <body>
>
>    Refs #{{issue_num}}
>
>    Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
>    ```
>
> 7. **Push**
>    ```bash
>    git push -u origin <branch>
>    ```
>
> 8. **Issue comment** `## [stage:implementer] ready for test` with:
>    - Branch name
>    - Commit hash
>    - LOC + file count
>    - AC selfcheck (per checkbox: done? in which diff?)
>    - Standards values used (resolved branch pattern, syntax-check output)
>
> ### Hard constraints
>
> - **NO** docker/test run (that is the Tester stage)
> - **NO** PR create (that is the Closer stage)
> - **NO** hard-gates violation (see `{{hard_gates}}`)
> - **NO** edits outside files-to-touch without reasoning in the comment
> - **NO** inline duplication of a registry component (only if `{{components}}` is set): if AGENTS.md carries a `components:` block AND your diff touches `code_globs` scope, you MUST use the referenced registry component. If nothing in the registry fits your use case, **STOP** — do not inline a new implementation. Report to the parent with a proposed ADR path (`docs/adr/<n>-<slug>.md`) covering purpose, alternatives, and a registry entry stub; the parent decides with the user before you continue.
>
> ### Parent output
>
> Max 200 words with:
> - Branch + commit
> - Diff stat (files, LOC)
> - AC-coverage selfcheck
> - Anomalies (spec gaps, unexpected refactorings, skipped ACs with reasoning)

---

## Parent decision (after Implementer output)

- **OK** → start stage 3 (Tester). Skip rule: diff only in `docs/**` / `*.md` / `// comment` → skip Tester, go directly to stage 4.
- **Spec gap too large** → ESCALATE. Telegram text to the user with a spec hint, suggest `/create-issue --refine {{issue_num}}`.

---

## Tester check criteria (type-specific for `code`)

The Tester stage checks for the `code` type:
- `{{smoke_test}}` from AGENTS.md succeeds (build GREEN, tests GREEN)
- Per AC checkbox: concrete smoke test + proof (log snippet, command output)
- Cleanup: reset live stack/state if touched

See `skills/work-issue/SKILL.md` stage 3 for the full Tester brief
(generic across all types).

---

## Critic check criteria (type-specific for `code`)

The Critic stage checks for the `code` type:
- Per AC: `search_code` via codebase-memory for code evidence
- Diff quality: naming, error handling, cleanup, security
- Out-of-scope check against `default_oos` + issue OoS
- `hard_gates` check

See `skills/work-issue/SKILL.md` stage 4 for the full Critic brief.
