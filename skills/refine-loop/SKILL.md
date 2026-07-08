---
name: refine-loop
description: An autonomous refinement loop — takes a functionally-DONE app and makes it genuinely usable, thoughtful, robust, and well-architected, one behavior-preserving ROI-ranked improvement at a time, stopping only when further work hits diminishing returns. Audits four lenses (architecture & code health, usability & thoughtfulness, production-readiness & operations, refactoring discipline), scores each candidate by Impact×Confidence÷Effort against a threshold θ (correctness/security must-fixes bypass the gate), executes the best with external verification + rollback-on-regression, records ADRs, and halts when K consecutive rounds surface nothing above θ — reporting "diminishing returns at θ; lower θ to go deeper." Writes docs/REFINE-GOAL.md + docs/REFINE-BACKLOG.md + docs/REFINE-WORKLOG.md. Use when the user runs /refine-loop, or says "refine this app", "make it production-grade", "make it really usable", "polish this", "harden this", or "productionize".
argument-hint: "[theta=N] <the app/repo to refine + any focus lens>"
---

# /refine-loop — refine a finished app until further work stops paying off

The loop you run **after** a thing works. Its job is not to add features and not to hunt
individual bugs on a diff — it is to take a functionally-done app and make it **genuinely usable,
thoughtful, robust, and well-architected**, then keep going until the next improvement isn't worth
its cost. It (1) writes the target to `docs/REFINE-GOAL.md`, (2) keeps a scored candidate ledger in
`docs/REFINE-BACKLOG.md` (this is the loop's memory and its stop signal — it lives in a FILE, never
in your head), (3) works one **behavior-preserving, externally-verified** improvement at a time,
ranked by return on investment, and (4) stops on **diminishing returns**, not on a clock and not on
"looks perfect."

**What it is NOT:** not `/research!`-a-diff code review, not a feature sprint, and — critically —
**never a big-bang rewrite** (rewrites throw away hard-won edge-case knowledge; you refine in place
via small safe steps). If the request is "add feature X" or "fix this crash," that's a different
tool. This one answers "make what already works *good*."

## Step 0 — parse args
- A token like `theta=2.5` (or `θ=2.5`) sets the **ROI threshold** — the one dial that controls how
  deep the loop goes (lower = does more, smaller-value work; higher = only the big wins). Default
  **`theta=2.0`** on a 1–5 Impact/Confidence/Effort scale. Strip it out.
- A **focus lens** (e.g. `usability`, `architecture`, `production`, `security`) narrows discovery to
  one pillar; absence means all four. Strip it out.
- Everything remaining is the **target** — the repo/app/path to refine. Default to the current repo.

## Step 1 — REFINE-GOAL.md = the target + scope, plainly
Target repo = the arg, a named path, or cwd. Docs go in `<repo>/docs/` (create if missing).
- **No `docs/REFINE-GOAL.md` yet:** write it from `templates/REFINE-GOAL.md` — the target, the θ, the
  focus (or "all four lenses"), and today's date (`date +%F`). Keep it a faithful statement of *what
  we are refining and how deep*, nothing more — no invented mission.
- **It already exists:** don't overwrite. Append a dated `## <date> — continuation` with the new
  directive (e.g. a lowered θ, a new focus). Append-only history.
- If there's no target and no GOAL, ask for one line: which app, and how deep. That's the only
  acceptable upfront question — without a target there's nothing to refine.

## Step 2 — recover state (resume the loop, never restart it)
Read, if present: `docs/REFINE-GOAL.md` (what + how deep), the **bottom** of
`docs/REFINE-WORKLOG.md` (what shipped last), `docs/REFINE-BACKLOG.md` (the scored candidates + the
**plateau/circuit-breaker counters** — the live stop state), and `docs/adr/` (decisions already
made, so you don't re-litigate them). These survive context loss; resume from them. Create the
worklog + backlog from their templates on first run.

## Step 2.5 — engage persistent mode (so it's hands-off)
The loop must keep rolling without pings.
- **If OMC is available:** invoke the **`ralph`** skill with the goal from `REFINE-GOAL.md` — it is
  the persistence engine ("the boulder never stops") and re-fires each turn. Use `/implement!` for
  the execution sub-step and a fresh reviewer agent for verification when you have them.
- **Plain Claude Code (no OMC):** the loop is self-contained — its continuation state lives in
  `docs/REFINE-BACKLOG.md` (round number + counters), so any resume reads that file and carries on.
- **The stop counters MUST live in the backlog file, never in your narrative.** This is not
  optional: agents have no built-in concept of "done" and will happily invent work forever, and a
  loop that judges its own termination will talk itself past any limit. The round budget and the
  plateau/circuit-breaker counts are checked against the FILE each round — external state the model
  reads, not a feeling the model has.
- **Clean exit is mandatory** (see Step 5): when the loop stops, run `cancel` (OMC) / tear down any
  persistence flag, then report. A dangling persistence state re-fires forever.

## Step 3 — the four lenses (what "refined" means)
Every discovery round samples across **all four** unless a focus lens was set. The full, concrete
checklist — what to look for and why it matters — lives in **`references/dimensions.md`**; load it
when you need the detail. In one line each:

1. **Architecture & code health** — modularity / coupling / cohesion, the dependency rule
   (domain doesn't import infra), 12-factor, config & secrets isolation, dependency & supply-chain
   hygiene, API contracts & semver, data model & migrations, single-source-of-truth state (state
   machines over boolean soup), a real error taxonomy + resilience (timeout / retry+backoff /
   circuit breaker), idempotency, concurrency correctness, testability, and **churn×complexity
   hotspots** as the priority map.
2. **Usability & thoughtfulness** — first-run & sane defaults, empty / loading / success states,
   human error messages, Nielsen's 10 heuristics, **accessibility (WCAG 2.2 AA)**, i18n readiness,
   responsive layout + design tokens, *perceived* performance (feedback <100ms, progress past 10s),
   information architecture & discoverability, destructive-action safety (undo > confirm), microcopy
   & in-product help, offline/degraded behavior — and the same rigor for **CLIs / APIs / libraries**
   (`--help` with examples first, clean stdout vs stderr, meaningful exit codes, specific error
   types), not just GUIs.
3. **Production-readiness & operations** — security (OWASP Top 10, least privilege, fail-closed
   defaults), observability (structured logs / metrics / traces, the four golden signals, actionable
   alerts), health checks (liveness ≠ readiness), graceful shutdown, rate limiting & input-size
   caps, data safety (expand-contract migrations, *tested* restores, PII redaction), deploy
   (rollback, canary, flags), SLOs, cost, runbooks, SBOM + license hygiene.
4. **Refactoring discipline** — this lens governs *how* every change in the other three is made:
   behavior-preserving atomic steps, characterization tests + seams **before** touching untested
   code, strangler-fig / branch-by-abstraction / expand-contract for anything cross-cutting, ADRs
   for every structural decision, and fitness functions as the architecture regression oracle.

## Step 4 — RUN THE LOOP (one round at a time)
Each round is DISCOVER → SCORE → RANK → EXECUTE → RECORD. Keep the app **shippable and green at
every commit**.

1. **DISCOVER (breadth first).** Find candidate improvements across every lens (fan out one agent
   per lens when you can). Two hard rules borrowed from tournament loops: **cover every lens each
   round** (skipping one needs a logged reason — otherwise "found nothing" is untrustworthy), and
   **don't propose the same finding-family 3 rounds running without a win** (anti-thrash). Add each
   candidate to `docs/REFINE-BACKLOG.md`.
2. **SCORE.** Two tiers, deliberately not one uniform gate:
   - **Tier 0 — must-fix** (correctness bugs, security holes, data-loss / crash / single-point-of-
     failure risk): **bypass the ROI gate entirely.** Always fix if the effort is sane — this class
     of quality has a hard floor, no diminishing-returns curve. (These are "blockers" in
     production-readiness-review terms.)
   - **Tier 1 — improvement** (architecture elegance, usability polish, robustness beyond baseline,
     performance, maintainability, docs): score `ROI = Impact × Confidence ÷ Effort` on a 1–5 scale.
     It qualifies only if `ROI ≥ θ`. Two anti-gold-plating gates reject candidates before scoring:
     **YAGNI** (no structure for a hypothetical future feature — only real, present friction) and
     the **Rule of Three** (don't abstract fewer than ~3 real occurrences).
3. **RANK.** Tier 0 first, then Tier 1 by descending ROI. Weight Impact by **blast radius** — how
   much of the codebase or how many user flows the change touches (churn×complexity hotspots and
   hot user paths beat dead-code tidying).
4. **EXECUTE — one candidate at a time, behavior-preserving, externally verified.**
   - If the target code is untested, the **next step is a characterization test** that pins current
     behavior — not the change itself.
   - Make the smallest atomic step. For anything cross-cutting, use strangler-fig /
     branch-by-abstraction / expand-contract so the app never breaks mid-flight.
   - **Verify externally, never by self-report:** tests + build + lint green, and for a real
     structural change a *fresh reviewer* (a separate agent / `/review-pr!`), because the agent that
     wrote the change is the worst judge of it. On any regression, **roll back** and try the next
     candidate.
   - Record an **ADR** (`docs/adr/NNNN-*.md`) for every architectural decision — so the loop
     remembers *why* and never reverses its own call without new information.
5. **RECORD & update counters.** Append a `REFINE-WORKLOG.md` entry (what shipped + evidence:
   test/build/review). Then update the two counters in `REFINE-BACKLOG.md`:
   - if **≥1 candidate was accepted** this round → `plateau_count = 0`; else → `plateau_count += 1`.
   - if the round's discovery/execution itself **errored** (not "found nothing" — actually broke) →
     `fail_count += 1`; else → `fail_count = 0`.
6. **Commit** (conventional commits; keep CI green; PR workflow if the repo requires it), then loop
   back to step 1.

## Step 5 — when to stop (the diminishing-returns predicate)
Check the counters — which live in `REFINE-BACKLOG.md`, not in your head — at the end of every round.
**Stop when ANY of these holds:**

- **`plateau_count ≥ K`** (default **K = 2–3**) → **SUCCESS.** This is the real "diminishing returns"
  exit: several consecutive full-breadth rounds found nothing worth doing above θ. Report:
  *"Diminishing returns at θ=<θ>. Lower θ (e.g. to <θ−0.5>) to go deeper."* The user's dial is the
  only thing that reopens the search — that is exactly the "100% sure new paths yield diminishing
  returns **at this bar**" contract.
- **`round ≥ max_rounds`** (default **25**) → **BUDGET.** A hard, model-independent ceiling so a
  broken value signal can never run away. Report what's still on the backlog.
- **`fail_count ≥ N`** (default **N = 3**) → **ERROR**, reported **distinctly** from SUCCESS: the
  search itself is broken (tests won't run, environment is wrong), not converged. Surface it, don't
  dress it as done.
- **User cancel.**

**Tier-0 findings always execute regardless of `plateau_count`** — a security hole discovered on a
"quiet" round is not diminishing returns, it's a blocker. Plateau only governs Tier-1 polish.

On stop: run `cancel` / clear any persistence flag, then report — what shipped (with evidence), what
sits **below θ and parked** (so the user sees the road not taken), and the exact θ to lower to
continue. Never claim "done/perfect"; claim "no candidate clears θ=<θ>."

## Rules
- **Behavior-preserving or it's not a refinement.** A change that alters observable behavior is a
  feature/fix — out of scope; different safety machinery.
- **Verify externally, never self-grade.** Green tests / a fresh reviewer decide acceptance.
- **One atomic step per iteration; roll back on regression.** Never batch unrelated changes.
- **Never a big-bang rewrite.** Strangler-fig / branch-by-abstraction / expand-contract in place.
- **θ is the human's dial.** The loop stops itself at θ; only the user lowers it to go deeper.
- **Don't gold-plate.** YAGNI + Rule of Three reject speculative and premature abstraction.
- **ADR every structural decision.** The loop's own durable memory.
- **Report honestly.** A worklog "done" needs real evidence; a stop is SUCCESS, BUDGET, or ERROR —
  say which.

## Templates & references
- `templates/REFINE-GOAL.md` — the target + θ + focus, verbatim intent.
- `templates/REFINE-BACKLOG.md` — the scored candidate ledger + the plateau/circuit-breaker counters
  (the loop's external stop-state).
- `templates/REFINE-WORKLOG.md` — append-only, newest at the bottom, evidence on every "done."
- `templates/adr.md` — one architectural decision (Context / Decision / Consequences).
- `references/dimensions.md` — the full four-lens audit checklist (what to look for + why).
