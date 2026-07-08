# REFINE BACKLOG — {{TARGET}}

> The loop's memory AND its stop signal. The counters below are checked against **this file** at the
> end of every round — never against the model's own judgement (agents have no built-in concept of
> "done"). Candidates are scored `ROI = Impact × Confidence ÷ Effort` on a 1–5 scale; **Tier-0**
> (correctness / security / data-loss / SPOF) bypasses the θ gate and is always fixed.

## Loop state — the stop counters (update EVERY round)
- **round:** 0 / {{MAX_ROUNDS}}
- **θ (threshold):** {{THETA}}
- **plateau_count:** 0 / {{K}}   — consecutive rounds with **no** accepted candidate above θ → **SUCCESS** stop
- **fail_count:** 0 / {{N}}   — consecutive rounds where discovery/execution itself **errored** → **ERROR** stop

## Candidates
| ID | Lens | Tier | Candidate | Impact | Conf | Effort | ROI | Status |
|----|------|------|-----------|:------:|:----:|:------:|:---:|--------|
| {{FIRST_ID}} | {{LENS}} | {{TIER}} | {{CANDIDATE}} | – | – | – | – | proposed |

<!-- Status vocab: proposed → accepted (shipped + externally verified) | rejected (below θ, parked) |
     rolled-back (regressed, reverted) | blocked. Lens ∈ {architecture, usability, production, refactoring}.
     Tier ∈ {T0 (must-fix, ignores θ), T1 (ROI-gated)}. Weight Impact by blast radius (churn×complexity
     hotspots + hot user paths beat dead-code tidying). Below-θ candidates STAY in the table (parked). -->

## Parked below θ={{THETA}} (surfaced on stop — lower θ to reach these)
- _(none yet)_
