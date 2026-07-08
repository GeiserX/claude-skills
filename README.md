# claude-skills

Open-source [Claude Code](https://claude.com/claude-code) skills — reusable, self-contained
autonomous loops you drop into `~/.claude/skills/` (personal) or a project's `.claude/skills/` and
invoke as slash commands. The directory name becomes the command.

## Skills

### `/refine-loop` — refine a finished app until further work stops paying off
The loop you run *after* a thing works. It takes a functionally-done app and makes it genuinely
**usable, thoughtful, robust, and well-architected** — auditing four lenses (architecture & code
health · usability & thoughtfulness · production-readiness & operations · refactoring discipline),
fixing the highest-ROI improvement one **behavior-preserving** step at a time, and **stopping on
diminishing returns** — not on a clock and not on "looks perfect."

- Correctness bugs and security holes are **always** fixed (they bypass the value gate).
- Everything else is ranked by `ROI = Impact × Confidence ÷ Effort` against one tunable threshold
  **θ** — lower it to go deeper.
- Every change is verified **externally** (green tests / a fresh reviewer), never self-graded; it
  never rewrites from scratch; it records an ADR for every structural decision.
- It halts when several full-breadth rounds surface nothing worth doing, reporting *"diminishing
  returns at θ; lower θ to go deeper."*

```
/refine-loop <app>              # audit + refine until nothing clears θ=2.0
/refine-loop theta=1.5 <app>    # turn the dial down for smaller-value work
/refine-loop usability <app>    # focus a single lens
```

The full four-lens audit checklist (grounded in ISO 25010, OWASP, Google SRE, Nielsen/WCAG, and
Fowler/Feathers refactoring practice) lives in `skills/refine-loop/references/dimensions.md`.

### `/sergio-loop` — write the goal, then run the loop
A single-entry autonomous loop for a repo. It saves your directive **verbatim** to `docs/GOAL.md`,
engages persistent mode (`ralph` / `autopilot`), then drives a `research → implement → review` cycle
until the goal holds — logging to a worklog and parking genuine human-only calls in
`docs/DEFERRED-QUESTIONS.md`. Add `coordinate` to run two sessions on one repo off a shared board.

```
/sergio-loop <your directive>
/sergio-loop coordinate <directive>    # multi-session mode
```

> Both loops lean on the **oh-my-claudecode (OMC)** skill layer for persistence and the `/research!`
> · `/implement!` · `/review-pr!` sub-steps; `refine-loop` also degrades to a self-contained loop
> without OMC.

## Install

Each skill is a directory containing a `SKILL.md`. Symlink (or copy) the ones you want:

```bash
# personal — available in all projects:
ln -s "$PWD/skills/refine-loop" ~/.claude/skills/refine-loop

# or per-project:
ln -s "$PWD/skills/refine-loop" /path/to/project/.claude/skills/refine-loop
```

See the [Claude Code skills docs](https://code.claude.com/docs/en/skills) for how skills are
discovered and invoked.

## License

[GPL-3.0](LICENSE).
