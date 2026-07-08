---
name: git-sync
description: Commit all workspace changes, pull with rebase, resolve straightforward conflicts, and push. Use when the user asks to commit and push, sync with remote, or update branch history before pushing.
---

# Git Sync

## Overview
Use this workflow when the user wants a full Git sync sequence: commit local work, rebase onto upstream, then push.
Optimize for speed by using fast paths and batched commands.

## Fast Workflows

### Workflow A (default): tracked upstream exists
1. Detect current branch and upstream in one preflight call.
2. Run stage + optional commit + rebase + push in one batched command block.
3. Report final commit hash and upstream.

### Workflow B: no upstream configured
1. Detect missing upstream.
2. Ask user before setting upstream (unless they explicitly requested setting it).
3. After upstream is set, run Workflow A.

### Workflow C: no-change sync (pull/push only)
1. Use this only when user intent is explicitly sync-only (for example: "just pull and push", "no commit", "sync remote").
2. Skip `git add -A` and skip commit logic.
3. Run pull/rebase + push in one batched command block.
4. Report final commit hash and upstream.

## Required Behavior
- Use only non-interactive Git commands.
- If the user provided a commit message, use it exactly.
- If no message was provided and a commit is needed, create one yourself using a one-line Conventional Commit message that reflects the staged changes.
- If there is nothing to commit, continue with pull/rebase + push only if that still matches the user request.
- Prefer `git pull --rebase` over merge pulls.
- If rebase conflicts:
  - Resolve only obvious mechanical conflicts.
  - Run `git add <file>` then `git rebase --continue`.
  - If conflict intent is ambiguous, stop and ask the user how to proceed.
- If upstream is missing, detect remotes/branch and ask before setting upstream unless the user already requested that.
- Never use force push unless the user explicitly asked for it.
- Prefer a single batched shell block over many separate command invocations.
- Skip redundant inspections (`git status`, `git branch`, `git rev-parse`) once required values are already known.
- When user intent is explicitly sync-only, use Workflow C and do not stage/commit.

## Batched Command Pattern

### Preflight (single call)
```bash
git branch --show-current
git rev-parse --abbrev-ref --symbolic-full-name @{u} || true
```

### Workflow A execution (single call)
```bash
git add -A
if ! git diff --cached --quiet; then
  git commit -m "<message>"
fi
git pull --rebase
git push
git log -1 --oneline
git rev-parse --abbrev-ref --symbolic-full-name @{u}
```

### Workflow C execution (single call)
```bash
git pull --rebase
git push
git log -1 --oneline
git rev-parse --abbrev-ref --symbolic-full-name @{u}
```

### Notes
- Use `git diff --cached --quiet` to detect "nothing to commit" without an extra status call.
- If `git pull --rebase` reports conflicts, switch to conflict-resolution flow and continue rebase.
- If there are no local changes, still run pull/rebase + push when user asked for sync/push.
