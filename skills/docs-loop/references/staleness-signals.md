# docs-loop — staleness signals (what the auditors look for)

"Up to date" is a **staleness predicate keyed to concrete drift**, not "rewrite until it feels current."
Absent a detected signal, the correct action is to **leave the doc alone.** Each auditor reads ONE doc plus
the repo's code (in the `origin/main` worktree) and returns per-claim tuples — it **never edits**.

## The detection rules

1. **Factual drift** — the doc states a value the code contradicts. The highest-yield, most-verifiable class:
   - ports, env-var names, file paths, CLI flags/subcommands, function/endpoint signatures, config keys,
     bundle IDs, version numbers, default values.
   - Verify by grepping the code for the literal: a documented `--flag` that no CLI source defines, an env var
     the code never reads, a port that changed. A doc value that isn't greppable in the code is **WRONG**.
   - Wrong facts are **Tier-0** (bypass θ) — a wrong port or a command that would fail is worse than silence.

2. **Structural drift** — the doc describes an architecture/module layout that has moved:
   - renamed/split/merged modules or repos, a subsystem that replaced another, a directory that no longer
     exists.
   - **Mermaid diagrams** whose nodes no longer match the file tree / package list.

3. **Dead references** — links/paths to files, anchors, or URLs that no longer resolve:
   ```bash
   git -C "$wt" ls-files > /tmp/tracked                  # cross-check doc-referenced paths against reality
   command -v lychee >/dev/null && lychee --offline "$wt"/**/*.md   # dead relative links/anchors
   ```

4. **Coverage gaps** — a shipped subsystem (new top-level dir / package) with **no doc at all**. Flag as
   `MISSING`; a *new* doc is a bigger commitment than a patch, so a coverage gap is usually a **DEFER** (it
   may need intent/positioning the code can't give) unless the scope explicitly asks to fill it.

5. **Cheap age heuristic (pre-filter, not proof)** — a doc older than the code it covers is *suspect*, worth
   an auditor's look; it is **not** by itself a reason to edit:
   ```bash
   git -C "$wt" log -1 --format=%cd --date=short -- README.md
   git -C "$wt" log -1 --format=%cd --date=short -- src/    # doc older than code → audit it
   ```
   Also: `"Last updated: <old date>"` footers, and a doc whose only change in months was cosmetic.

## Mechanical pre-filters (run before spending auditor budget)

Cheap, deterministic passes catch a large share of drift with no LLM cost — run them first and feed the
auditors the flagged subset:

- **markdownlint** — structural Markdown breakage (heading hierarchy, malformed tables).
- **lychee** — dead links/anchors (above).
- **symbol-rename grep** — for each renamed/removed symbol you can see in `git log`, grep the docs for the old
  name still referenced.
- **mmlint** — `node ~/.claude/tools/mmlint/lint.mjs <file.md>` — a diagram that no longer *parses* is
  drift too (and must pass before you ship any edit regardless).

## The auditor's return shape (structured, no edits)

Each auditor returns, per claim it examined:

```
doc:line | claim (verbatim) | code-truth (file:line, or "none found") | verdict | suggested-fix | confidence
verdict  ∈ { MATCHES | STALE | WRONG | MISSING }
confidence ∈ { high | medium | low }   # low confidence = you couldn't pin the code truth → DEFER, don't guess
```

You (main thread) synthesize these — and when an auditor claims a contradiction, **grep the code yourself to
confirm before acting.** Trust, but verify: an auditor is a lead, not a license to edit.

## Scoring reminder (how a signal becomes an edit)

- `WRONG` / dangerously-misleading + confidently verifiable → **Tier-0**, fix now (bypasses θ).
- `STALE` / clarity / a fixable diagram → **Tier-1**, edit only if `ROI = Impact × Confidence ÷ Effort ≥ θ`.
- `MISSING` (coverage gap) → usually **DEFER** unless scope says fill it.
- `confidence: low` (can't pin the code truth) → **DEFER**, never guess.
- `MATCHES` → leave the doc alone. No drift, no edit.
