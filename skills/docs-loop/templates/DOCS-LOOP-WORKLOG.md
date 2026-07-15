# DOCS-LOOP WORKLOG — scope: {{SCOPE}}

> Append-only journal of the docs sweep. **Newest entry at the BOTTOM.** One entry per completed repo.
> A "done" entry MUST carry real evidence — file:line anchors for the claims changed, `mmlint` result, and
> the PR link. "Looks fresh" or "should be fine" is not evidence.

## TL;DR (read this first — updated each repo)

- Where it stands: 0 / {{N}} repos done · plateau {{P}}/3 · θ={{THETA}}
- Last repo: —
- Open deferrals: see DOCS-LOOP-DEFERRED.md

## Log

<!-- Append one block per completed repo, newest at the bottom. Copy this shape:

### {{REPO}} — docs synced ({{DATE}})
- Audited: <files> — <n> docs, <m> had drift.
- Shipped: <the specific claims corrected, each with the code anchor that proves the new value: file:line>.
- Deferred: <doc + why> → DOCS-LOOP-DEFERRED.md #<n>   (omit if none)
- Verified: fresh reviewer re-checked each claim vs code; mmlint OK <k>/<k>; secret scan clean.
- Evidence: PR <link>  ·  CI <green|n/a>.
- Ledger: {{REPO}} → DONE; plateau_repos → <reset 0 | +1>.

(If a repo audited CLEAN with no drift: "### {{REPO}} — audited clean, no edits ({{DATE}}) — plateau +1".)
-->
