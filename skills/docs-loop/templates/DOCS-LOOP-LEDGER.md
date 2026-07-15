# DOCS-LOOP LEDGER — scope: {{SCOPE}}

> The outer loop's memory AND its stop signal. The `Status` column is the resume tracker: a run continues at
> the first repo whose status is NOT in {DONE, DEFERRED, SKIPPED}. Never re-audit a DONE repo (idempotent).
> This file is append-only for the repo rows — never delete a row; only advance its status.

## Loop state (update at every repo boundary)

- scope: {{SCOPE}}
- repos_total: {{N}}
- repos_done: 0 / {{N}}
- plateau_repos: 0 / 3     <!-- consecutive repos where the audit found NOTHING above θ worth editing -->
- θ (per-edit ROI gate): {{THETA}}
- started: {{DATE}}

## Repos

| # | Repo | Canonical path | Priority | Status | Docs touched | PR / commit link |
|---|------|----------------|:--------:|:------:|:------------:|------------------|
| 1 | {{REPO}} | ~/repos/neutral/{{REPO}} | active | PENDING | – | – |

<!--
Status vocab:
  PENDING     → not yet audited (the unread flag)
  IN-PROGRESS → currently in the audit→ship cycle
  DONE        → confident edits shipped as a PR (link filled) OR audited clean (no drift → no PR)
  DEFERRED    → whole repo blocked on a human-only call (see DOCS-LOOP-DEFERRED.md)
  SKIPPED     → archived / no local clone / worktree-only / out of scope (reason in Docs-touched cell)
  BLOCKED     → push 403 / CI red / branch-protection needs an unavailable review

plateau_repos: +1 when a repo needed zero edits above θ; reset to 0 when a repo needed an edit.
Halt when all rows are DONE/DEFERRED/SKIPPED (SUCCESS) or plateau_repos ≥ 3 (diminishing returns).
-->
