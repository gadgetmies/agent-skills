---
name: commit-and-test-discipline
description: Use when making any code change in track-feature-extractor or dj-automation — before committing, when choosing which tests to run, when authoring tests, and when preparing a worktree branch for merge. Covers WIP vs clean commits, atomic history, relevant-subset vs full-suite test runs, and state-isolated parallel-safe tests.
user-invocable: true
---

# Commit & test discipline

## Overview

**Loose during the work, strict at the merge boundary.** While building inside a
worktree, optimise for momentum: WIP checkpoints and targeted test runs. Before
the branch merges to `main`, pay the full cost once: clean atomic history and the
whole suite green. The two phases have *different* rules — don't apply the strict
ones mid-work (wastes time) or the loose ones at merge (ships junk).

This governs commit/test conduct **inside** the worktree lifecycle. The lifecycle
itself (worktree per change, user-gated `git merge --no-ff`, cleanup) lives in
`docs/contributing-workflow.md` (TFE) / that same doc referenced from dj-automation
`CLAUDE.md`. **REQUIRED BACKGROUND:** read that lifecycle before applying this skill.

**Violating the letter of these rules is violating the spirit of them.**

## The two phases

### Phase 1 — During the work (inside the worktree): go fast

- **WIP commits are allowed and encouraged.** Prefix them `wip:` so they are
  greppable and unmistakable. Use them as save-points before risky edits.
- **Do NOT run the full suite per commit.** *You* decide when a test run earns its
  cost — after a meaningful logical unit lands, before a risky refactor. Not after
  every one-line checkpoint.
- **Run only the tests relevant to what changed** — the touched module's tests, its
  direct importers, and the one e2e path that covers the touched flow. Full-suite-
  every-time is the waste this skill exists to remove.
  - **TFE:** `apps/desktop/venv/bin/python -m unittest tests.<module>` with
    `QT_QPA_PLATFORM=offscreen` (Qt tests). Pick the module(s) covering your change.
  - **dj-automation:** `./run_tests.sh <module> [<module>...]` (never bare `pytest`
    from the repo root — it reports bogus collision errors).

### Phase 2 — Preparing to merge (the boundary): go strict

- **Clean up WIP → atomic history.** Reorganise the branch into coherent commits
  (see Commit hygiene). **A `wip:` commit must NEVER reach `main`.** `rebase -i` is
  unavailable in the agent shell (no interactive editor); build the final sequence
  non-interactively — `git reset --soft` to unstage back to a base, then stage and
  `commit -m` each atomic unit in order, or drive the rebase via
  `GIT_SEQUENCE_EDITOR`/`--exec`.
- **Now run the FULL suite, and it must be green** (code changes; see
  Documentation-only changes). This is the one place "all
  green" is mandatory and non-negotiable.
  - **TFE:** the relevant `tests.*` modules for the surface (full desktop discover
    boots a real render server on :47765 — see `docs/dev-environment.md`).
  - **dj-automation:** `./run_tests.sh` (all modules).
- **For bisectability, prefer green at *each* final commit** (`git rebase --exec
  '<test cmd>'`). If per-commit is too costly, full-suite-at-tip + relevant-subset-
  per-commit is the accepted fallback — but say which bar you set, because it
  changes how far `git bisect` can be trusted.

## Documentation-only changes: no test runs

If every file in the branch's diff is prose — Markdown docs, `CLAUDE.md`/`AGENTS.md`,
READMEs, ADRs, OpenSpec artifacts, design notes — skip test runs in both phases,
including the full suite at the merge boundary. Commit hygiene still applies.

A file is **not** documentation when code, tests, hooks, the build or a probe reads
it: config, `.claude/` settings and hooks, fixtures, scripts, templates, UI markup
(QML/HTML), schema or contract files, and docs whose contents a check greps for.
One such file in the diff makes the whole change a code change — test it as one.

## Commit hygiene (applies to the cleaned-up commits)

- **One logical change per commit.** Never mix a refactor with a behaviour change —
  that is the classic `git bisect` killer.
- **Separate mechanical/generated changes** (formatting, codegen, generated
  `Theme.qml`, lockfiles) into their own commits so review and blame skip them.
- **Stage explicit paths — never `git add -A`/`git add <dir>`.** `main` and its
  worktrees are shared; `-A` sweeps concurrent agents' untracked WIP into your
  commit. Run `git diff --staged` and *read it* before writing the message.
- **Message:** imperative subject ≤ ~50 chars with a scope that names the real
  surface (`fix(desktop): …`, not a `web-*` scope on a desktop diff); body explains
  **why**, not just what. Reference ADR/change/issue ids so history is greppable.
- **Trailers** as the repo requires (`Co-Authored-By:`, session link). These make
  history auditable and `git log --grep`-searchable.
- Smell test: if the message needs "and" or a bullet list of unrelated items,
  **split the commit.**

## Authoring tests (so Phase-1 speed is even possible)

Targeted subsets and parallel runs only work if tests are built for them.

- **Independent & state-isolated.** Every test/case runs in any order with no shared
  mutable state: fresh fixtures/DB/temp dirs per test, no ordering dependency, no
  leaked globals, no fixed shared ports/servers. State leaks are exactly what break
  subsetting *and* parallelism (this repo has been burned by tests that boot a
  shared render server on a fixed port and leave it listening).
- **Build the pyramid** — it is what makes "relevant subset" meaningful:
  - **Thin end-to-end layer** — the most common and crucial flows: the primary happy
    path plus the error paths whose failure is most expensive/user-visible. These
    are the smoke tests that must pass before every merge.
  - **Fast-feedback layer** — unit/small-integration, run constantly during Phase 1
    for quick signal; cheap to run as a subset.
  - **Detailed layer** — edge cases, branches, less-common paths for high coverage;
    run when its area is touched, and before merge.

## Quick reference

| Situation | Do |
|---|---|
| Mid-work checkpoint | `wip:` commit, no test gate |
| Which tests during work | Only those covering the change (module + importers + its e2e) |
| Risky refactor ahead | `wip:` save-point first, run relevant tests after |
| Preparing to merge | Squash WIP → atomic commits, then **full suite green** |
| Diff is prose only (docs, `CLAUDE.md`, ADRs, specs) | No test runs; hygiene still applies |
| Any commit | explicit paths, read `git diff --staged`, why-in-body |
| New test | isolated, no shared state, placed in the right pyramid layer |

## Rationalization table — STOP if you think any of these

| Excuse | Reality |
|---|---|
| "I'll just squash the WIP later, one `wip:` on main is fine" | No `wip:` commit ever reaches `main`. Clean up *before* the merge, not after. |
| "Full suite per commit is thorough" | During work it's waste. Run the relevant subset; full suite only at the boundary. |
| "Skipping the full suite before merge saves time" | For a code change the merge boundary is the one place it's mandatory. No exceptions. |
| "`git add -A` is faster" | On a shared checkout it steals other agents' untracked WIP. Stage explicit paths. |
| "Refactor + fix in one commit is cleaner" | It's the #1 bisect killer. Split them. |
| "This test is fine reusing the shared server/port/tmp" | State leak → breaks parallel + subset runs. Isolate it. |
| "I'll write only unit tests / only an e2e" | Missing a pyramid layer. Cover the crucial flow e2e *and* keep fast unit feedback. |
| "The change is trivial, no test run needed before merge" | Trivial code changes break suites. Full suite green at the boundary for anything that is not prose only. |
| "It's only a config/hook/fixture tweak, so it's documentation" | Anything code, tests, hooks or the build read is code. Test it. |

## Red flags — you are about to violate this skill

- A `wip:` commit on the branch you're about to merge
- `git add -A` / `git add <dir>` staged
- Committing without reading `git diff --staged`
- A commit that both refactors and changes behaviour
- Merging a code change without having run the full suite green
- A new test that touches a shared server, fixed port, or shared tmp/DB
- Running the whole suite after every tiny edit during Phase 1
