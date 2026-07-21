# RFC #3 — Git-native issue backends for the stagecrew loop: beads vs GitHub issues

**Status:** Research findings (English, for public repo)
**Author:** Researcher agent
**Date:** 2026-07-21

## Summary

**Recommendation: PASS for now on adopting beads as the loop's backend; keep GitHub issues. Optionally track beads as a future "complement" experiment, not a replacement.**

beads (`bd`) is a genuinely well-designed, agent-first issue tracker: a dependency graph, atomic claim (`bd update --claim`), an auto-ready queue (`bd ready`), local-first/offline operation, and in-repo versioned storage. On paper it maps almost perfectly onto an agentic loop. But three things make it the wrong bet for stagecrew *right now*: (1) it is undergoing heavy pre-1.0/1.0 churn — the storage backend was swapped from SQLite+Git to Dolt around bd 1.0, and that migration "broke every single one of those repos at the same time," forcing a "Restoring Beads Classic" rescue effort; (2) its core value proposition — **in-repo, versioned, auditable issue state** — is *already delivered* by stagecrew's "merged-status-in-repo" philosophy, where the merged PR + `gh` is the audit record; and (3) for a **public** repo, GitHub issues are the discoverability and external-contributor surface, and beads' source of truth (a Dolt database synced over `refs/dolt/data`) is invisible to a casual browser and to drive-by contributors. beads solves problems (context-window bloat from markdown plans, multi-agent claim races, long-horizon dependency graphs) that stagecrew — a single-issue-in, single-PR-out loop — does not currently have. Adopt it only if/when stagecrew moves to genuinely **parallel multi-agent** work on one repo, at which point beads' atomic claim becomes worth its cost. Until then, the added dependency (a Go/Dolt binary, a `.beads/` state store, a sync branch) is a net liability.

## What beads is (data model + CLI + Dolt)

beads (canonical repo: **`steveyegge/beads`**, actively mirrored/developed as **`gastownhall/beads`**; authored by Steve Yegge, not steipete) bills itself as a *"distributed graph issue tracker for AI agents, powered by Dolt."* It provides persistent, structured, queryable memory for coding agents, replacing free-form markdown TODO/plan files with a dependency-aware issue graph so an agent can run long-horizon work without blowing its context window.

**Data model.** Issues are nodes in a graph with hash-based hierarchical IDs — `bd-a1b2` (task), `bd-a3f8.1` (subtask), plus epics. The hash-based IDs are deliberate: they *"prevent merge collisions in multi-agent/multi-branch workflows."* Edges are typed dependencies and graph links: blocking `dep` edges (which drive `bd ready`), plus `relates-to`, `duplicates`, `supersedes`, `replies-to`. There is also a `message` issue type (threaded, ephemeral) for agent-to-agent mail delegation, and a `bd remember` memory facility with "memory decay" that summarizes old closed tasks to reclaim context.

**Storage backend (this has changed — important).** As of the current README, the source of truth is **Dolt** (a version-controlled SQL database) living under `.beads/`:
- **Embedded mode (default, "Beads Classic"):** single-writer, data in `.beads/embeddeddolt/`.
- **Server mode:** external `dolt sql-server` for concurrent writers, data in `.beads/dolt/`.
- `.beads/issues.jsonl` exists but is explicitly *"an export for viewers and interchange, not the source of truth or a backup."*
- Cross-machine sync is **not** ordinary git: it's `bd dolt push` / `bd dolt pull` against a `refs/dolt/data` ref on the git remote.

Note the churn: earlier beads (bd < 0.55) stored issues as **SQLite + Git**, and older write-ups (wal.sh, Better Stack) still describe JSONL or SQLite as the source of truth. The Dolt migration landed around **bd 1.0** in early 2026. This matters for any adoption decision (see maturity below).

**CLI surface (the parts a loop would use):**

| Command | Purpose |
|---|---|
| `bd init` | Initialize `.beads/` in a repo (`--contributor` routes planning issues to a separate location) |
| `bd create "Title" -p 0` | Create an issue (priority, type, deps as flags) |
| `bd ready` | List **unblocked** issues (no open dependencies) — the agent work queue |
| `bd show <id>` | Full issue detail (JSON output available) |
| `bd update <id> --claim` | **Atomically** set assignee + in-progress (race-safe) |
| `bd close <id>` | Complete an issue; unblocks its dependents |
| `bd dep add <child> <parent>` | Add a dependency edge |
| `bd dolt push` / `bd dolt pull` | Sync the issue DB across machines/remotes |
| `bd export` / `bd import` | Backup / migrate via JSONL (not day-to-day sync) |
| `bd github ...` | Sync to/from **GitHub issues** (pull GitHub → beads, push beads → GitHub) |
| `bd prime` / `bd remember` | Inject workflow context / store persistent memory |

## Agent workflow

beads is built around a tight autonomous loop, which is why it's interesting here:

1. **Plan:** the agent decomposes work into `bd create` calls with explicit `bd dep add` edges (epics → tasks → subtasks).
2. **Discover:** `bd ready` returns *only* the currently actionable issues — "no ambiguity about execution order," dependencies enforced by the graph.
3. **Claim:** `bd update <id> --claim` atomically marks ownership (assignee + in-progress in one operation), so **two agents cannot grab the same issue** — the headline feature for parallel agents.
4. **Work → Close:** `bd close <id>` completes the task and unblocks dependents, so the next `bd ready` surfaces newly-available work. Loop until `bd ready` is empty.

Everything runs **local-first / offline** (embedded Dolt), with targeted queries ("retrieve only info needed for the immediate task") rather than loading a giant markdown plan into context — the explicit context-efficiency pitch. Sync to teammates/other machines happens out-of-band via `bd dolt push/pull`.

## The Dolt angle

**Dolt** is *"Git for data"* — a SQL database you can `fork`, `clone`, `branch`, `merge`, `push`, and `pull` like a git repo, with versioning as a first-class concept (content-addressed, tables instead of files). Its distinguishing feature for an issue DB is **cell-level merge**: conflicts are detected per (row, column) cell, not per line. If two branches change *different* fields of the same issue, they merge cleanly; only a genuine same-cell divergence is a conflict, surfaced in a `dolt_conflicts_<table>` table with `ours`/`theirs` auto-resolution strategies or manual resolution. For a multi-agent, multi-branch issue store this is a real advantage over line-based git merges of a JSONL/markdown file, which collide constantly. The cost: Dolt is a heavier dependency than "a text file in git," and beads' own recent history shows the operational risk of leaning on it (below).

## Trade-offs vs GitHub issues (test matrix)

Scoring for stagecrew specifically (single public repo, issue→merged-PR loop, merged-status-in-repo philosophy). ✅ strong / ⚠️ partial / ❌ weak.

| Criterion | GitHub issues (current) | beads (REPLACE gh) | beads (COMPLEMENT + sync to gh) |
|---|---|---|---|
| Agent atomic claim | ⚠️ possible via labels/assignee but racy | ✅ native `--claim`, race-safe | ✅ native claim locally |
| Offline / local-first | ❌ needs network + API | ✅ embedded Dolt, fully offline | ✅ offline locally, sync later |
| Audit-trail visibility (public) | ✅ web UI + full comment history + PR links | ❌ truth in Dolt DB, opaque to browsers | ✅ GitHub keeps the public record |
| Public signal / discoverability | ✅ indexed, searchable, "N open issues" badge | ❌ invisible unless you run `bd` | ✅ preserved via sync |
| External-contributor visibility | ✅ zero-install, everyone knows it | ❌ contributors must install `bd`, learn it | ⚠️ contributors use GitHub; maintainer uses beads |
| Migration cost for the loop | — (baseline) | ⚠️–❌ rewrite `/create-issue` + `/work-issue` | ⚠️ additive, keep gh + add bd + sync glue |
| Maturity / stability | ✅ battle-tested, stable API | ❌ pre-1.0/1.0 churn, backend swap broke repos | ❌ same churn + sync is a newer surface |
| Lock-in | ⚠️ GitHub-platform lock-in | ⚠️ Dolt + `bd` lock-in (different lock-in) | ⚠️ both, plus sync coupling |
| Dependency-graph / ready-queue | ❌ no native blocking/ready query | ✅ first-class `bd ready` | ✅ first-class locally |
| Context efficiency for agents | ⚠️ must fetch full issue bodies | ✅ targeted queries, memory decay | ✅ targeted queries locally |

Read: beads wins the **agent-mechanics** rows (claim, ready-queue, offline, context) and loses the **public-surface + maturity** rows. GitHub issues wins exactly the rows stagecrew's public, audit-first, merged-status-in-repo philosophy cares most about.

## Integration options for the loop

**(a) REPLACE `gh` issues with beads.** Honest assessment: **bad fit for a public repo.** It moves the unit of work into a Dolt DB that external contributors can't see or file against without installing `bd`, and it discards GitHub's free public audit trail — directly at odds with the maintainer's "merged-status-in-repo, auditable" philosophy (which currently *uses GitHub as that record*). You'd trade GitHub lock-in for Dolt/`bd` lock-in and inherit beads' migration instability. Only sensible for a private, single-maintainer, heavily-parallel-agent repo.

**(b) COMPLEMENT — beads for local/offline/parallel-agent work, sync to GitHub for the public audit trail.** The most defensible variant, and beads explicitly supports it via the `bd github` command suite (pull GitHub issues into beads, push beads issues out) and `bd init --contributor`. The loop would use beads locally for the ready-queue + atomic claim during a run, and GitHub issues remain the canonical public record. **But**: this only pays off when there's genuine parallelism or offline need. For today's stagecrew (one issue → one PR, sequentially), it adds a second source of truth, a sync step that can drift/conflict, and a new failure mode — for a race and an offline capability the loop isn't currently hitting. The `bd github` sync surface is also newer/less proven than beads' core.

**(c) PASS — don't add the dependency.** The current `gh`-based flow already gives in-repo auditable status (the merged PR *is* the record), zero extra install for contributors, and full public discoverability. stagecrew's loop is sequential and its "state lives in the repo via merged PRs" — beads' headline wins (atomic claim, ready-queue across many parallel agents) don't bind yet. **This is the recommended posture.**

## Recommendation: PASS (revisit as COMPLEMENT if the loop goes parallel)

Adopting beads now would be **solving problems stagecrew doesn't have at the cost of the properties it explicitly values.** The maintainer's philosophy is git-native + auditable + merged-status-in-repo — and for a *public* repo, GitHub issues + merged PRs already *are* that git-native, public, auditable record. beads' in-repo storage is real but its source of truth is a Dolt database synced over a non-standard `refs/dolt/data` ref, which is *less* browsable and *less* contributor-friendly than a GitHub issue, not more. On top of that, beads is visibly immature for infrastructure you'd bet a loop on: the SQLite→Dolt backend migration around bd 1.0 "broke every single one of those repos at the same time," spawned a "Restoring Beads Classic" recovery effort and multiple `bd migrate` data-loss bug reports, and the docs still disagree with each other about the storage model across versions. That's a lot of churn to absorb for a loop whose value is *reliability*.

Where beads genuinely shines — **race-safe atomic claim and a dependency-driven ready-queue across many concurrently-running agents** — is precisely where stagecrew isn't today (single issue in, single PR out, sequential). The honest call: **PASS**, keep `gh`. Re-open this as the **COMPLEMENT** option the day stagecrew starts running multiple agents against one repo in parallel, or needs to plan/claim work offline; at that point beads' claim semantics and `bd github` sync bridge become worth the dependency, and the RFC should be re-evaluated against a *then-stable* (post-churn) beads release.

No follow-up implementation issue is specified, because the recommendation is PASS. (If the maintainer chooses COMPLEMENT anyway, the minimal spec would be: title "spike: beads as local ready-queue with gh as public record"; AC = `.beads/` initialized, `/work-issue` reads `bd ready`/`bd update --claim` locally while still opening+merging the GitHub PR, `bd github` sync keeps the GitHub issue authoritative, and a documented rollback to pure-`gh`; files-to-touch = `/create-issue` skill, `/work-issue` skill, AGENTS.md, plus a beads-version pin. Gate it behind a stable beads release.)

## Alternatives (git-native / local-first issue trackers) — one line each

- **git-bug** (`git-bug/git-bug`) — distributed, offline-first tracker embedding issues/comments as native git objects; has **bidirectional GitHub/GitLab bridges**, CLI/TUI/web UIs; the most mature "issues-in-git" option and the closest philosophical match to merged-status-in-repo.
- **Fossil SCM tickets** — built-in bug tracker/wiki/forum in a single SQLite repo file; fully offline, but it's a whole *alternative VCS* (not a git add-on), so a non-starter for a GitHub-hosted loop.
- **git-issue** (`dspinellis/git-issue`) — minimalist shell-based decentralized issues stored as files in git, with GitHub/GitLab import/export; low-dependency but low-tooling.
- **sqlite-in-repo / JSONL-in-repo (DIY)** — commit a SQLite or JSONL issue file to the repo; trivial and dependency-light but line-based git merges collide badly under concurrency (exactly the pain Dolt's cell-level merge is meant to remove).
- **beads_rust** (`Dicklesworthstone/beads_rust`) — a fast Rust port of pre-Dolt beads (SQLite + JSONL export); notable mainly as evidence the community valued the *simpler* SQLite-era model that upstream moved away from.

## Sources

- https://github.com/steveyegge/beads/blob/main/README.md — canonical README: Dolt backend, `.beads/embeddeddolt/`, CLI surface, atomic claim, ID scheme
- https://github.com/gastownhall/beads — active mirror/fork, feature overview
- https://betterstack.com/community/guides/ai/beads-issue-tracker-ai-agents/ — practical agent-loop walkthrough (describes older SQLite backend)
- https://wal.sh/research/beads — third-party analysis (describes JSONL-as-source-of-truth era + adoption/maturity numbers)
- https://github.com/dolthub/dolt — Dolt "Git for Data"
- https://dolt.gitbook.io/dolt-dev/concepts/dolt/git/conflicts and https://docs.dolthub.com/sql-reference/version-control/merges — Dolt cell-level merge & conflict resolution
- https://www.dolthub.com/blog/2026-04-02-restoring-beads-classic/ — the SQLite→Dolt migration fallout and "Beads Classic" restore
- https://github.com/gastownhall/beads/issues/2573 — "Moving to dolt pretty much made beads unusable for me" (maturity signal)
- https://github.com/gastownhall/beads/issues/2276 and https://github.com/steveyegge/beads/issues/1752 — `bd migrate` data-import/mkdir failures
- https://github.com/gastownhall/beads/issues/1529 — "Feature request: Sync with GitHub Issues" + `bd github` command suite context
- https://gist.github.com/leonletto/606e8afbb3603870d14b4123707416a2 — pre-1.0 → bd 1.0 migration guide (version timeline: SQLite < 0.55, server-Dolt 0.55–0.63, embedded 1.0+)
- https://github.com/git-bug/git-bug/blob/master/README.md — git-bug (native git objects, GitHub/GitLab bridges)
- https://fossil-scm.org/home/doc/tip/www/bugtheory.wiki — Fossil ticket system
- https://github.com/dspinellis/git-issue — git-issue
- https://github.com/Dicklesworthstone/beads_rust — Rust port (SQLite + JSONL)
