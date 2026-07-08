# refine-loop — the four-lens audit checklist

The concrete "what to look for and why" behind Step 3. Each item is a *candidate source* for the
DISCOVER step — not a mandate to change everything. Score every candidate by ROI (Impact ×
Confidence ÷ Effort ≥ θ); the Tier-0 items (marked **[T0]**) bypass the ROI gate and are always
fixed. Grounded in ISO/IEC 25010, the Twelve-Factor App, OWASP, Google SRE, Nielsen/NN-g, WCAG 2.2,
clig.dev, and Fowler/Feathers refactoring practice.

---

## Lens 1 — Architecture & code health

**Modularity, coupling, cohesion**
- Single responsibility: a module with >1 reason to change (I/O + logic + presentation in one file) is a split candidate.
- Coupling quality: flag content/common coupling (reaching into another module's internals, shared mutable globals) and stamp coupling (passing a whole config blob when a field would do). Prefer data coupling.
- Cohesion: `utils`/`helpers`/`misc` files >200 lines are dumping grounds — regroup by real sub-domain.
- Connascence: convert Connascence of Position (`fn(a,b,c,d,e)`) → named params/objects; Connascence of Meaning (magic numbers/booleans, `status===2`) → enums/named constants.
- Law of Demeter: 3+-deep accessor chains (`a.getB().getC().doThing()`) couple a caller to a whole object graph.
- Named smells: God Object, Feature Envy (a method fixated on another class's data), Shotgun Surgery (one change → many files), Divergent Change (one file changes for many reasons), Primitive Obsession, Data Clumps, Long Parameter List.

**Layering & dependency direction**
- The Dependency Rule: source dependencies point inward — domain/core never imports infra/db/http/ui (invert via a port the core owns). The single highest-leverage long-term property.
- **[T0-ish]** No dependency cycles between modules (`madge --circular`, `dependency-cruiser`, ArchUnit) — a cycle means the "modules" are really one unit.
- Bounded contexts: two unrelated domains silently sharing one model/table is how "modular" rots into a big ball of mud — add a translation layer at the seam.
- Directory structure reads as capabilities (`billing/`, `auth/`) not only technical layers (`controllers/`, `models/`) past a certain size.

**12-factor / operability shape**
- Config in the environment, not code; one codebase many deploys; explicit pinned dependencies; backing services as attached, swappable resources; strict build/release/run separation; stateless share-nothing processes; fast startup + graceful shutdown; logs to stdout as an event stream.

**Config & secrets**
- **[T0]** No hardcoded secrets in source or git history (`gitleaks`/`trufflehog` over `--all`); `.env` gitignored + never committed; a `.env.example` exists.
- **[T0]** No server secret shipped to a client bundle via a public-var convention (`NEXT_PUBLIC_*`, `VITE_*`, `REACT_APP_*`).
- Per-environment credentials (never a prod cred used locally); secrets from a manager/vault, not plaintext; secrets never logged/surfaced in errors (redaction hook).
- Config validated at startup (schema: zod/pydantic) — fail fast, don't discover a missing var deep in a 2am request.

**Dependency & supply-chain hygiene**
- Lockfile committed; CI uses `--frozen-lockfile`/`npm ci`.
- **[T0]** Known-vuln scanning in CI (`npm/pip/cargo audit`, `govulncheck`, Dependency-Track); triage critical/high.
- Unused deps removed (`depcheck`/`knip`/`cargo-udeps`); deliberate update cadence (Renovate/Dependabot actually merged); license compatibility checked; SBOM generated (CycloneDX/SPDX) per release.

**API / interface / contracts**
- Semver honored — diff the public surface; a "patch" that removes/renames/retypes an exported symbol or response field is a lie.
- Per-protocol breaking-change rules (REST: adding optional = safe, removing/retyping = breaking; protobuf: never reuse a field number; GraphQL: removing/narrowing = breaking) checked mechanically (`openapi-diff`, `buf breaking`, `graphql-inspector`).
- Schema-first spec (OpenAPI/proto/SDL) as source of truth, validated in CI; a real deprecation window with a machine-readable signal before removal.

**Data model & state**
- Normalized by default; any denormalization has one tested write path (not multiple that drift).
- **Expand-contract** for every backward-incompatible schema change (add → dual-write/backfill → drop only after nothing reads the old shape); never schema+dependent-code in one deploy.
- Single source of truth per fact; caches/derived data have a guaranteed invalidation path.
- Explicit state machines (enum + transitions) for lifecycle entities, not independent boolean flags that allow impossible states.
- Stateless compute (state only in a backing store); **[T0]** idempotent transitions for anything triggered by a retryable event (webhook/queue/double-click) or it double-charges/double-sends.
- UTC storage; money as integer minor units or decimal, never float; units named in field names (`timeoutMs`).

**Error handling & resilience**
- One error taxonomy (validation / business-rule / transient / fatal) with consistent representation, not a single `Error` everywhere.
- **[T0]** No silent failure: empty `catch {}`, swallowed promise rejections, ignored error returns, `except: pass`.
- Error context preserved across boundaries (wrap with `cause`, don't replace); fail fast at the earliest safe point.
- **[T0]** Every outbound network call has a bounded timeout; retries use exponential backoff + jitter and are capped; circuit breakers + bulkheads on critical dependencies; graceful degradation over all-or-nothing.

**Concurrency**
- **[T0]** Check-then-act on shared state (TOCTOU) wrapped in a transaction/lock/atomic upsert (`INSERT ... ON CONFLICT`).
- Documented thread-safety on shared mutable objects; prefer immutability for value objects; no un-awaited fire-and-forget promises swallowing errors; no CPU-bound work blocking an event loop; distributed locks (not in-process mutexes) where coordination spans instances.

**Testability & test architecture**
- A deliberate test shape (pyramid vs trophy) that actually exercises the seams between modules, not all-mocked units that pass while integration is broken.
- **Characterization tests before refactoring untested/legacy code** (the load-bearing safety net).
- Coverage prioritized by criticality × churn, not a blanket %; interaction-heavy mocks that assert *which methods were called* make refactoring impossible — flag as a blocker to fix first.
- Mutation testing (Stryker/PIT/`mutmut`/`cargo-mutants`) on critical modules to prove tests would catch a bug; flaky tests quarantined, not "just re-run"; dependencies (clock, network, randomness) injected for testability.

**Complexity & prioritization signal**
- Cyclomatic complexity >~10 and cognitive complexity >~15 per function are refactor candidates.
- **Churn × complexity hotspots are the priority map** — `git log` frequency × complexity; a small % of files carries most defects/effort. Complex-but-stable and simple-but-hot are both lower priority than complex-and-hot.

**Tech-debt framing**
- Real debt has a nameable *principal* (cost to fix) and *interest* (extra cost per change while unfixed) — if you can't name the interest, it may be a preference, not debt.
- Fowler's quadrant: Deliberate+Prudent needs a tracked repayment trigger; Inadvertent+Reckless is exactly what this loop fixes; Inadvertent+Prudent (learning) is often fine to accept.
- Debt beyond code: architecture, test, docs, infra/ops, knowledge (bus-factor-1 via `git blame` concentration).

---

## Lens 2 — Usability & thoughtfulness

*(Applies to GUIs, CLIs, APIs, and libraries — see the CLI/API block at the end.)*

**First-run & defaults** — usable within seconds, no mandatory account/config gate; sane defaults auto-detected from the OS (locale/theme/timezone), cursor pre-focused; wizards skippable/resumable with step X-of-Y; permissions requested just-in-time; reset symmetric to setup.

**States** — designed empty states (distinguish first-use / filtered-to-zero / error-caused, each with different copy + a way forward); skeleton screens for full-page loads, spinners for isolated refreshes, nothing under ~1s; every mutation gets a visible acknowledgment (never silent success); quantified progress past ~10s.

**Errors & validation** — **plain-language messages, no stack traces/error codes to end users** (Nielsen #9), state what went wrong + the concrete next step; validate inline on blur, re-validate live once a field has errored; never wipe entered data on a failed submit; prevent errors (disable invalid submits) over reporting them; honest generic fallback + retry/report for unexpected errors, never a raw trace/blank screen.

**Nielsen's 10 heuristics** — visibility of status; match real-world language (no internal jargon/DB names in UI); user control (Cancel/Back/Esc/Undo everywhere); consistency & standards; error prevention; recognition over recall; flexibility (shortcuts, command palette, batch); aesthetic/minimalist (progressive disclosure); recognize-diagnose-recover from errors; help & documentation at the point of need.

**Accessibility (WCAG 2.2 AA)** — keyboard operable + no traps, visible focus (never `outline:none` bare), focus order matches visual; contrast 4.5:1 text / 3:1 UI; reflow usable at 400% zoom / 320px; target size ≥24×24 (aim 44–48); name/role/value on custom controls; status messages via `aria-live`; semantic HTML first ("no ARIA beats bad ARIA"); never color alone; honor `prefers-reduced-motion`. New in 2.2: focus not obscured, drag alternatives, redundant-entry, accessible auth (allow paste/password managers). Audit = automated scan (a floor, ~30–57% of issues) + keyboard-only pass + real screen-reader pass.

**Internationalization** — zero hardcoded user-facing strings (verify via pseudo-localization); ICU pluralization not `count + " item(s)"`; dates/numbers/currency via `Intl`; full RTL mirroring via logical CSS properties; 30–200% text-expansion headroom; configured fallback locale (never a raw key leaking to UI).

**Layout & consistency** — real-device breakpoints, no horizontal scroll; one design-token source of truth (color/spacing/type) so theming is a swap not a re-skin; component reuse; dark mode re-checks contrast rather than filtering the light theme.

**Perceived performance** — Nielsen bands: <0.1s instant (no indicator), <1s unbroken flow, ~10s attention ceiling (quantified progress past it); Doherty ~400ms is the "feels responsive" target; optimistic UI for rare-failure mutations (NOT payments) with visible rollback+undo on failure; prefetch on intent, stale-while-revalidate, lazy-load, code-split. A user can't tell "slow" from "broken" — feedback is what prevents slow-but-working reading as failure.

**IA, navigation, discoverability** — progressive disclosure (advanced options one level deeper); task-oriented structure (user's mental model, not the DB schema); search/filter wherever a list exceeds ~15–20 items; breadcrumbs past 2 levels; a new capability findable via ≥2 of {nav, search, contextual, announcement}; forgiving typo-tolerant search.

**Destructive-action safety** — tiered: reversible → no interruption or an Undo toast; medium → confirm / `--force`; **irreversible → type the exact name** (defeats habituation). **Undo is the primary defense, confirmation secondary** (over-used dialogs get click-through-without-reading). Keep dangerous options spatially separated from benign ones. Discoverable keyboard shortcuts (shown in menus, OS-convention, a `?` overlay), command palette (Cmd/Ctrl+K), batch ops, saved views.

**Microcopy, help, notifications** — one defined voice; active voice, second person, specific button labels ("Delete project" not "OK"/"Submit"); never blame the user; suspend personality in error/destructive/billing contexts; in-product help at the point of confusion; task-oriented docs ("how do I export") alongside reference; notifications answer what-happened/what-it-concerns/what-to-do, frequency-capped, user-controllable; changelog non-intrusive (badge, not a forced modal).

**Offline / degraded / large data** — visible network-status ("offline / reconnecting"), never silent failure; deliberate graceful-degradation vs progressive-enhancement per feature; cache-first reads survive a drop, offline writes queue + sync with a pending-state UI; paginate/virtualize long lists (never 10k+ DOM rows); stream large import/export with progress; timeouts + bounded retries on every request; resumable/idempotent long operations.

**CLIs / APIs / libraries (same thoughtfulness, text-only surface)** — `--help` on root + every subcommand, **examples first** before the flag wall; a command needing args prints concise help, not silence; **primary output to stdout, logs/errors/progress to stderr** (piping never breaks); a `--json` machine-readable mode + TTY detection; **exit codes: 0 = success, distinct nonzero = distinct failures, documented**; errors rewritten for humans with a fix suggestion, full debug detail to a file not the terminal; something visible within ~100ms; validate + bail **before** mutating state; detect non-TTY stdin (don't hang on a prompt), support `--no-input`/`--yes`; **never a secret via a bare flag** (leaks to shell history) — use a file/stdin; for libraries, specific documented error types (`RateLimitedError`) not one generic `Error` callers must string-match.

---

## Lens 3 — Production-readiness & operations

**Security** — **[T0]** walk the OWASP Top 10 (2021): A01 Broken Access Control (authz on *every* request, server-side, object-level/IDOR), A02 Crypto Failures, A03 Injection (parameterized queries, no string-built SQL/shell), A04 Insecure Design, A05 Misconfiguration (no stack traces to users), A06 Vulnerable Components (SCA), A07 Auth Failures (session rotation on login, `Secure`/`HttpOnly`/`SameSite`), A08 Integrity, A09 Logging Failures, A10 SSRF (validate user-supplied fetch URLs against an allowlist). Least privilege per identity; **fail-closed defaults** (a check that throws/times out must deny, not admit); TLS everywhere with automated renewal; non-root containers, minimal base images.

**Observability** — structured logs (JSON, not printf) + metrics + traces with a correlation/trace ID propagated through every hop and log line; the **four golden signals** (latency [success vs fail separately], traffic, errors, saturation) dashboarded per user-facing service; **actionable alerts only** — every page maps to a specific action + a linked runbook; delete alerts that say "check the logs."

**Lifecycle & resilience** — **liveness ≠ readiness** (liveness must NOT call downstream deps or a dependency blip restarts the instance in a loop; readiness does); startup probe for slow boots; **graceful shutdown** (fail readiness → drain in-flight → close resources, all before SIGKILL); timeouts / retries+backoff+jitter / circuit breakers / bulkheads on every dependency; **[T0]** idempotency keys on non-idempotent mutating endpoints; deduplication on at-least-once consumers.

**Abuse & limits** — centralized (shared-store) rate limiting with `429` + `Retry-After`, tiered by traffic class; **[T0]** explicit request-body / header / URL / array-length caps enforced before parsing (unbounded input = trivial DoS); abuse rejections logged with identity + dashboarded.

**Data safety** — expand-contract migrations (backward-compatible with still-running old code); **backups with a *tested restore drill*** (an untested backup is an unverified hope) stored in a separate failure domain, with defined RTO/RPO; per-category data-retention TTLs (logs & backups are the commonly-missed ones); **[T0]** PII redacted at ingestion into logs/traces/analytics (the most common leak vector, not the primary DB).

**Deployment** — reproducible CI-only builds (never "build locally, push the binary") with pinned toolchain + signed provenance (SLSA/Cosign); **automated rollback** wired to error/latency/health thresholds, tested regularly; canary or blue-green with metric-gated progression; feature flags decouple deploy from release (fastest kill-switch) — with a flag-retirement process; zero-downtime = readiness + graceful-shutdown + compatible-migration + safe-rollout working together (prove it with a rolling deploy under load, asserting client-side zero errors).

**Reliability & cost** — SLIs/SLOs tied to real user experience + an error budget with an actual policy when exhausted; graceful degradation (core vs enhancing features classified, enhancing failures wrapped in fallbacks); chaos/failure injection to *prove* the resilience patterns work; explicit resource requests/limits (no BestEffort for anything that matters); cost-allocation tags + idle-resource detection; load-test at 5–10× real peak and confirm the load generator wasn't itself the bottleneck.

**DR, runbooks, ownership** — runbooks readable in <5 min, *specific* ("run this exact query" not "check the logs"), linked from the alert; a documented on-call rotation + explicit incident roles; every dependency has a defined "what happens when it's down" (timeout + retry + breaker + fallback); a *drilled* full-region failover, not just a data restore.

**Compliance & licensing** — a working data-export + data-deletion path (test it end-to-end); a `LICENSE` file matching intent; dependency licenses scanned for incompatibility (copyleft in a proprietary tree); SBOM per release with component hash + license.

---

## Lens 4 — Refactoring discipline (governs HOW every change above is made)

- **Behavior-preserving only.** A change that alters observable behavior is a feature/fix, not a refinement — it needs new tests, not just green regression tests. Classify before you touch.
- **Two hats.** Never mix refactoring and behavior change in one commit — tag each commit "refactor" or "change" so any regression is instantly attributable.
- **Safety net first.** Characterization tests + seams (an interface to insert a test double) before refactoring untested code; golden-master (snapshot outputs over many inputs) when the code is too tangled to unit-test.
- **Small atomic steps.** Each step "too small to be worth doing" alone; the program is never broken for more than minutes; auto-rollback on a red test is trivial.
- **The move library.** Fowler's catalog (Extract Method/Class, Move Method, Replace Conditional with Polymorphism, Introduce Parameter Object, …) — each has known-safe mechanics, the right granularity for an autonomous step.
- **Big changes without a big bang.** Strangler-fig (grow the replacement beside the original, redirect callers piece by piece, delete the old); branch-by-abstraction (abstraction layer → new impl behind it → cut over → delete); expand-contract (parallel change) for interface/schema. Never a from-scratch rewrite — it discards hard-won edge-case knowledge and stalls delivery for months.
- **ADRs.** Record every significant structural decision (Context / Decision / Consequences, 1–2 pages, numbered, superseded-not-deleted) — the loop's durable memory and the auditor's trail.
- **Fitness functions.** Automated architecture guardrails in CI (no new dependency cycle, no forbidden import, complexity/coupling thresholds, no latency regression) — the architecture regression oracle, parallel to the test suite's behavior oracle. Architecture-as-implemented drifts from architecture-as-designed without them.

---

## The stop discipline (why the loop ends)

- **Design-stamina reality:** a functionally-done app is already past the "design payoff line" (~weeks), so continued refinement is presumptively still worth it — the stop signal is *not* elapsed time and *not* "is it perfect."
- **No hard quality metric exists** (Fowler says so) — so rank by the computable proxy `Impact × Confidence ÷ Effort` and stop by *saturation*, not by measuring absolute quality.
- **Satisficing / good-enough:** stop searching once nothing clears the bar θ — you are not seeking the global optimum, only improvements above θ. James Bach's fourth test — *"further improvement would be more harmful than helpful"* — is real: churn has regression-risk and complexity cost, not just shrinking benefit.
- **Kano:** must-be quality (bugs/security = Tier 0) has a hard floor and no diminishing-returns curve → always fix; attractive/performance quality (Tier 1) is where the θ-and-plateau machinery belongs.
- **Convergence pattern:** "K consecutive rounds below threshold" is the same stall/patience rule independently arrived at by ML early-stopping, genetic-algorithm stall generations, simulated-annealing cooldown, and OMC's own `self-improve` loop — about as validated as a stop rule gets.
