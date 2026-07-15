# docs-loop — the safety contract (nine preflight gates)

A doc loop's blast radius is asymmetric: the value of one fresher doc is small, but the cost of a leaked
token, a clobbered feature branch, or a confidently-wrong public README is large. So these gates are the
whole product. Every one is a **hard skip-on-fail check per repo, run BEFORE any write** — if a repo can't
pass a gate, set its ledger status to `SKIPPED`/`BLOCKED` with the reason and move on. Never work around a
gate.

## Gate 1 — Canonical worklist, never `ls`

The repo tree holds many more directories than repos — most are **worktrees** and duplicate checkouts.

```bash
# a worktree's common-dir points OUTSIDE its own folder → it is NOT a canonical repo
is_worktree() { [ "$(git -C "$1" rev-parse --git-common-dir 2>/dev/null)" != "$(git -C "$1" rev-parse --git-dir 2>/dev/null)" ]; }
```

Build the list from the **Repo Map in `~/repos/neutral/CLAUDE.md`** (read at runtime — never hardcode it,
never copy its credential-bearing content), map each name to its single canonical checkout
`~/repos/neutral/<repo>`, and collapse every worktree into its canonical repo. Do not treat 8
`neutral-shell.wt-*` dirs as 8 repos.

## Gate 2 — Work in a throwaway `origin/main` worktree, NEVER the live checkout

This one gate dissolves three separate hazards at once (dirty tree, feature-branch entanglement, active-loop
race). Many neutral checkouts sit on feature branches with dozens of uncommitted files; committing there
would bundle the loop's doc edits into a human's half-finished work.

```bash
repo=~/repos/neutral/<repo>
unset GITHUB_TOKEN GH_TOKEN
git -C "$repo" fetch --quiet origin
wt="$(mktemp -d)/wt"
NOHOOKS="$(mktemp -d)"                 # empty hooks dir → dodge a repo's hanging pre-commit hooks
git -C "$repo" -c core.hooksPath="$NOHOOKS" worktree add --detach "$wt" origin/main
# ... do all reading of code truth, auditing, and editing inside "$wt" ...
git -C "$repo" worktree remove --force "$wt"   # always clean up
```

Read code truth and make edits **in the worktree**, off `origin/main` (the code that actually shipped) — not
the user's dirty branch. Never `git stash`, `git checkout`, or `git add -A` in the user's checkout. Stage
only the exact doc files you wrote, by explicit path.

## Gate 3 — Secret firewall

Never read secret-bearing files into the drafting context; scan every staged hunk before commit.

```bash
# NEVER read these into context while drafting:
#   CLAUDE.md, AGENTS.md, .env*, .keys/**, *.pem, *.p12, credentials, *secret*, *.token
# scan the staged doc diff for secret shapes (abort the repo on any hit):
git -C "$wt" diff --cached -- '*.md' \
  | grep -nE 'sk-ant-|sk-[A-Za-z0-9]{20,}|tskey-|ghp_|gho_|github_pat_|AKIA[0-9A-Z]{16}|xox[baprs]-|AIza[0-9A-Za-z_-]{35}|-----BEGIN [A-Z ]*PRIVATE KEY-----|[0-9]{13,19}' \
  && { echo "SECRET-SHAPED STRING IN STAGED DOC — abort this repo"; }
command -v gitleaks >/dev/null && gitleaks protect --staged --no-banner   # if installed
```

Docs describe **mechanism, never values** — this is the same rule as "docs state intent, not sources." Output
may contain no literal token, key, or card number.

## Gate 4 — OFF-LIMITS denylist (never edit)

Default action on hand-authored prose is **annotate, never rewrite.** Never edit:

- `CLAUDE.md` / `AGENTS.md` (any level) — Sergio's hand-tuned instruction files, not "docs to update."
- `docs/adr/**` and any decision record — append-only history, superseded-not-deleted.
- anything matching `*WORKLOG*`, `*GOAL*`, `*DEFERRED*`, `*COORDINATE*`, `REFINE-*`, `DOCS-LOOP-*`.
- `CHANGELOG*`, `LICENSE*`.
- `docs/research/**`, design/decision/vision docs, historical writeups.
- any file whose git history shows human (non-loop) authorship of the prose you'd be changing.

**In-scope by default** (generated/reference surfaces that describe *current* code): README install/usage
sections, API reference, config/env/port tables, CLI usage, and architecture Mermaid diagrams. Editing a
hand-authored narrative doc requires the user's scope to name it explicitly.

## Gate 5 — Skip an autopilot-locked repo

A repo under an active `sergio-loop`/`ralph` run will be committed to concurrently; a second loop there races
it. Detect and skip:

```bash
# heartbeat: an AUTOPILOT-WORKLOG.md touched in the last hour, or a persistence flag/lockfile
find "$repo/docs" -name 'AUTOPILOT-WORKLOG.md' -mmin -60 2>/dev/null | grep -q . && echo "autopilot-locked → skip"
ls "$repo"/.omc/state/sessions/*/ "$repo"/.ncx/sergio-loop.json 2>/dev/null | grep -q . && echo "loop state present → skip"
```

## Gate 6 — Clean-main truth

Docs must follow the code that actually shipped. Gate 2 already guarantees this (you work off `origin/main`),
so **never** generate docs from the user's dirty feature branch, whose code may never merge.

## Gate 7 — Scope filter applied before drafting

The user's scope is a hard filter applied *before* any auditing/drafting, not a preference to drift from. A
repo (or a doc) outside the scope is never touched, even if it looks stale.

## Gate 8 — Branch-protection reality check

Before relying on the PR path, confirm the repo will actually accept it:

```bash
gh repo view <org>/<repo> --json isArchived,defaultBranchRef,viewerPermission
gh api repos/<org>/<repo>/branches/main/protection 2>/dev/null   # required reviews that won't come → note it
```

An **archived** repo (`isArchived: true`, e.g. `neutral-k8s`) → `SKIPPED (archived, read-only)`; its change
belongs in its live sibling. A repo whose PR needs a review it won't get → note `BLOCKED`, don't stall
silently.

## Gate 9 — Nothing outward-facing merges autonomously

The loop opens PRs; a human merges. Anything outward-facing/positioning/legal/security-claim, anything that
would name a person or cite a call, and any doc deletion → a **draft PR + a `DOCS-LOOP-DEFERRED.md` entry**,
never an autonomous merge. A human merge is the reversibility boundary.

---

**Minimum before touching any repo:** gates 1–4 satisfied and demonstrably enforced (canonical worklist,
throwaway `origin/main` worktree, secret firewall, off-limits denylist). If you cannot enforce these, do not
run the loop on that repo — a stale doc is cheaper than a confident lie or a leaked key.
