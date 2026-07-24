---
name: close-out
description: "Post-work lifecycle closer. Merges wave branches in wave order, auto-resolves additive conflicts, closes issues, and cleans up branches. No GitHub Projects required — tracks state exclusively via issue open/closed state and wave labels. Pipeline position: work-issue → close-out. Triggers: /close-out, close out, merge waves, ship waves, finalize waves, merge after work."
---

# /close-out — Ordered wave merge and lifecycle cleanup

**Type:** lifecycle / merge orchestration

## Purpose

After all `/work-issue` loops for a planned wave set have completed (or a subset is ready), `close-out` merges the resulting PRs in **wave order**: wave 1 first, then wave 2 (rebased onto the wave 1 merge), then wave 3, and so on. This preserves the execution order that `/plan-issues` established and prevents PRs from stepping on each other at merge time.

### Pipeline position

```
plan-issues → work-issue (wave 1) → work-issue (wave 2) → ... → close-out
```

## Invocation variants

```
/close-out --repo owner/slug
/close-out --repo owner/slug --wave 1          # close out only wave 1
/close-out --repo owner/slug --wave 1,2        # close out waves 1 and 2 in order
/close-out                                     # → resolves repo from cwd, closes all open waves
```

## Design constraint: no GitHub Projects dependency

`close-out` tracks all state through standard GitHub primitives only:

- **Issue state:** open / closed via `gh issue view` and `gh issue close`
- **Wave labels:** `wave:<n>` (set by `/plan-issues`) on issues and their PRs
- **PR state:** open / merged / closed via `gh pr list` and `gh pr merge`

No `gh project` CLI calls. No board columns. No project fields. Works in any repo regardless of whether GitHub Projects is enabled.

## Workflow

### 1. Wave discovery

Find all waves with work ready to merge:

```bash
gh issue list --repo <slug> --state open --label "wave:1" --json number,title,labels
```

Repeat for each wave number. A wave is "ready" when all its issues have an open PR with the Critic's `## [stage:critic] APPROVE` comment (i.e., `/work-issue` has completed stage 4 and is waiting on the Closer, or has finished via stage 5).

### 2. Pre-merge safety check

For each wave (in ascending order), before merging:

1. **Verify wave 1 is complete** before starting wave 2, etc. — check that all PRs in wave N-1 are merged.
2. **Rebase check:** for wave 2+, check whether the PRs are still rebased on the latest `pr_base` after wave N-1 merged. If not, request a rebase:
   ```bash
   gh pr comment <pr-num> --body "Wave 1 has merged. Please rebase this branch on main and re-push."
   ```
   Then wait (or skip with a warning — see "Partial execution" below).

### 3. Additive conflict detection and resolution

For each PR about to be merged, check for merge conflicts against the current `pr_base`:

```bash
git -C <repo_path> fetch origin
git -C <repo_path> merge-base origin/<pr_base> origin/<branch>
git -C <repo_path> diff origin/<pr_base>...origin/<branch> --name-only
```

Classify each conflict as **additive** or **non-additive**:

**Additive conflict (auto-resolvable):** both sides only add content to a file that neither deleted:
- Both branches appended to the same file (e.g., two loops each added a migration file, or both added a line to the same config list)
- Resolution: take both additions, ordered by wave number (wave 1's addition first, then wave 2's)

**Non-additive conflict (human decision required):**
- Two branches edited the same lines in different ways
- One branch deleted content the other branch modified
- Resolution: flag as a STOP, post a comment on the PR with the conflicting diff hunks and a clear description, and ask the user to resolve manually before re-running `/close-out`

### 4. Merge execution (wave by wave)

For each wave, in ascending order:

```bash
gh pr merge <pr-num> --repo <slug> --squash --delete-branch
```

After each PR merge:
- Verify the merge completed: `gh pr view <pr-num> --json state --jq '.state'` → `MERGED`
- Close the issue explicitly if needed (see "Issue close" below)
- Wait for any CI checks on `pr_base` to settle before merging the next PR in the same wave (parallel PRs within a wave can be merged without waiting for each other, but a CI gate on `pr_base` itself must pass first)

### 5. Issue close

After each PR merge, close the linked issue:

```bash
gh issue close <issue-num> --repo <slug> \
  --comment "Closed via PR #<pr-num> (wave <n>, close-out). Merged into <pr_base>."
```

Note: if the PR body contains `Closes #<num>` and it merges into the default branch, GitHub auto-closes. The explicit close is a safety net for:
- `pr_base` diverging from the default branch
- Issues not linked in the PR body with `Closes`

### 6. Label cleanup

After closing each issue, remove the `wave:<n>` label and the `claimed:<agent-id>` label (if still present):

```bash
gh issue edit <num> --repo <slug> --remove-label "wave:1"
```

### 7. Branch cleanup

The `--delete-branch` flag on `gh pr merge` handles branch deletion. If a branch was not deleted (e.g., the PR was already merged manually), clean it up:

```bash
gh api -X DELETE repos/<slug>/git/refs/heads/<branch> 2>/dev/null || true
```

### 8. Final summary

After all waves are processed, output a summary:

```
Close-out complete — Domek-Labs/myrepo
────────────────────────────────────────

Wave 1 — merged (3 PRs)
  PR #68 → issue #42 closed ✓
  PR #71 → issue #44 closed ✓
  PR #72 → issue #51 closed ✓

Wave 2 — merged (2 PRs)
  PR #75 → issue #45 closed ✓
  PR #76 → issue #48 closed ✓

Skipped (non-additive conflict, human review needed):
  PR #77 (issue #50) — conflict in src/ui/TokenPanel.tsx lines 44-51
  → Resolve manually and re-run /close-out --wave 3

Total: 5 merged, 1 skipped
```

## Partial execution

`close-out` is **re-entrant**: if some PRs have non-additive conflicts or pending CI, it completes what it can and exits cleanly with a list of what was skipped and why. Re-running after resolving the skipped items continues from where it left off (already-merged waves are not re-processed).

Already-merged PRs are detected by checking PR state (`MERGED`) — they are skipped silently.

## Hard gates

`close-out` will not merge a PR if:

1. The wave number label is missing (cannot determine merge order)
2. The PR has no Critic APPROVE comment (`## [stage:critic] APPROVE`) — the loop may still be running
3. A non-additive conflict exists with the current `pr_base` — manual resolution required
4. A CI check that is green on `pr_base` is red on the PR (follows the same baseline rule as `/work-issue` Closer)

## What this skill does NOT do

- Does not write code or modify source files
- Does not create new branches or worktrees
- Does not push commits
- Does not touch GitHub Projects boards or project fields
- Does not run the deploy command (`deploy_command` from AGENTS.md) — deployment is a separate step after close-out completes

## See also

- `skills/plan-issues/SKILL.md` — assigns wave labels (prerequisite for close-out)
- `skills/work-issue/SKILL.md` — drives individual issues (produces the PRs close-out merges)
- `skills/run-loop/SKILL.md` — single-command pipeline (create → plan → work; close-out is intentionally separate)
