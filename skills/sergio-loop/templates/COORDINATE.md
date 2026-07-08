# COORDINATE.md — {{REPO}} shared board

> Append-only multi-agent board. This file **is** the shared memory between independent sessions —
> it survives context loss/compaction, so re-read it at the **top of every loop iteration** and after
> any release. Every agent appends **dated, signed** entries; newest at the **BOTTOM** of each section.
> Keep ACKs explicit and fast.

## Lanes (who owns what)

- **{{LANE}} 🪨** — {{LANE_OWNERSHIP}}
- _(other lanes register themselves here when they join)_

## The work loop (shared)

{{LOOP_LINE}} → commit → repeat.
Re-read this board at the top of each iteration; **ACK** anything new before starting your own work.

## Hard rules

- **Don't edit another lane's files** — propose an ask; the owning lane implements.
- **Single-driver resources** (a shared box, a deploy target): post `CLAIMING <resource>` and wait for an
  `ACK` before using it; post `RELEASING <resource>` when done.
- **Contracts are the unit of exchange** — if another lane must depend on your surface, write it here
  verbatim (or in `CONTRACT.md`) and announce changes **before** merge.
- **Status vocabulary:** `OPEN` / `ACK` / `IN-PROGRESS` / `DONE` / `BLOCKED`.
- Edit a status **in place**; add a dated note under it rather than deleting.

## What I'm doing NOW (so we don't collide)

- **{{LANE}}:** {{CURRENT_FOCUS}}

## Blocked on you / Sergio (NOT touching until answered)

- _(none yet)_

## Log

### [{{DATE}}] {{LANE}} — loop kickoff

Registering this lane. Read the board, will ACK any open asks before starting. Driving the loop on: {{CURRENT_FOCUS}}.
