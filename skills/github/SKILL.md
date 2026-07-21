---
name: github
description: "Git and GitHub interaction conventions for loop commits, pushes, and PRs. Use gh / the GitHub API for writes when the local .git is read-only (constrained containers); set the commit author from AGENTS.md commit_identity; never a company/shared email; no Co-authored-by trailers unless configured (no_unconfigured_coauthors); PR body conventions (Closes #, standards block) and the squash-merge co-author caveat. Triggers: github, git push, gh cli, commit identity, commit author, coauthor, co-author, pull request, pr body, squash merge."
model: inherit
---

# github — git / gh interaction conventions

**Type:** reference / convention

Shared conventions for every stagecrew stage that touches git or GitHub (Implementer, Closer, `/init-agents`). Keeps commit attribution clean and drift-free.

## 1. Writes: prefer `gh` / the API when `.git` is read-only

In a constrained container the local `.git` can be read-only (no `git push`), and tooling like `python3` may be missing.

- **Normal environment:** plain `git` — `git commit`, `git push -u origin <branch>`.
- **Read-only `.git`:** do **not** fight the local push. Use `gh` / the GitHub REST API for the write:
  - Commit + push via the Contents API (`gh api --method PUT repos/<slug>/contents/<path>` with base64 content + `branch`), or `gh api` GraphQL `createCommitOnBranch` for multi-file commits.
  - Open the PR with `gh pr create` (or `gh api repos/<slug>/pulls`).
- Detect the case first: if `git push` fails with a read-only / permission error, switch to the `gh` path rather than retrying.

## 2. Commit author identity — from AGENTS.md `commit_identity`

The author identity is **not** ambient. It comes from the AGENTS.md `work-issue:` `commit_identity` block (`name` + `email`).

- When `commit_identity` is set, run **before every commit**:
  ```bash
  git config user.name "<commit_identity.name>"
  git config user.email "<commit_identity.email>"
  ```
- When it is absent, fall back to the ambient `git config` (no change).
- **Never** author a loop commit under a company or shared email. Use a personal identity or a GitHub noreply address (`<id>+<user>@users.noreply.github.com`).

## 3. No unconfigured co-authors (`no_unconfigured_coauthors`)

Do **not** add a `Co-authored-by:` trailer or a bot footer to any commit message or PR body — unless AGENTS.md explicitly configures one.

- Rationale: a stray `Co-authored-by:` line pulls a bot or a foreign account into the repo's contributor list. That is exactly the failure this gate prevents.
- This is a hard-gate; the Implementer and Closer briefs enforce it.

## 4. PR body conventions

- **Link the issue:** include `Closes #<num>` so the merge auto-closes it.
- **Standards block:** summarize the AGENTS.md standards used (branch pattern, syntax check, smoke test, hard gates honored).
- **Sections:** summary, test plan, loop-workflow process block.
- **No co-author lines / bot footers** in the body (same gate as §3).

## 5. Squash-merge caveat

stagecrew squash-merges PRs. **GitHub appends the co-author lines from the individual squashed commits onto the squash commit.** So a clean PR body is not enough — **every individual commit on the branch must already be free of `Co-authored-by:` trailers**. Keep the commits clean at author time (§2, §3); do not rely on cleaning up at merge time.

## 6. Non-interactive shell commands (agent-hardening)

An agent runs shell commands non-interactively — a prompt for `y/n` hangs the loop indefinitely. On many systems `cp`/`mv`/`rm` are aliased to `-i` (interactive) mode, and `git`/`gh` can open a pager or editor.

- **Always pass non-interactive flags** for file ops: `cp -f`, `mv -f`, `rm -f`, `rm -rf` (never a bare `cp`/`mv`/`rm` that may inherit an `-i` alias).
- Avoid commands that open an interactive editor/pager: set `GIT_PAGER=cat` / `--no-pager`, provide `-m` for commit messages, use `gh --json`/`-q` instead of paged output, and use `gh` flags like `--yes` where a confirmation would otherwise block.
- Interactive git flows (`git rebase -i`, `git add -i`) are unavailable — script the non-interactive equivalent.

## See also

- `skills/init-agents/references/AGENTS.md.template` — the `commit_identity` field + the `no_unconfigured_coauthors` hard-gate.
- `skills/work-issue/references/subagent-briefs/code-implementer.md` / `research-implementer.md` — commit step that applies these conventions.
- `skills/work-issue/SKILL.md` — Closer stage PR + squash-merge.
