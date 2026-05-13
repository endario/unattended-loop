# unattended-loop — overview

Sets up an unattended autonomous `/loop` run on Claude Code's native `/loop` primitive. Bakes a standard engineering playbook (TDD, `/review` on meaningful PRs, PR-first, no-stop policy) into a directive file so you don't retype it each session.

## When to use

You should be at the **execution** end of a deliberate pipeline — the upstream thinking is done:

- ✓ Design document (multi-round design, e.g. via `brainstorming`) — committed
- ✓ Architecture review settled (e.g. architecture-critic + `/review`) — "do not re-litigate" recorded
- ✓ Plan written (e.g. via `writing-plans`) — phased, with dependencies + scope carve-outs
- ✓ Issues filed (umbrella + children) in your tracker — GitHub / Linear / etc.
- ✓ Docs committed; issues link to them

Then this skill executes the plan unattended:

- You're leaving Claude Code (overnight, day-trip, multi-hour block)
- The work involves real engineering output (commits, PRs, issue closures), not chat
- You want continuity across tasks (state, in-flight PRs, mid-flight redirect via the brief)

**Don't use** if the design / plan / tracking isn't in place — invoke this skill on half-baked input and you'll get half-baked output at scale.

## Invocation

    /unattended-loop                                  # infer goal from conversation
    /unattended-loop <goal text>                      # explicit goal
    /unattended-loop help                             # show this message
    /unattended-loop --no-<rule> <goal>               # disable a playbook rule

## What it produces

A 4-file harness in `<repo-root>/.claude/`:

- `loop.md` — directive (rules + per-tick recipe; static across sessions)
- `loop-brief.md` — session goal + scope (the only file you edit between fires)
- `loop-status.sh` — state observer (run on every tick)
- `loop-progress.md` — append-only checkpoint log

Then outputs the trigger: `/loop <<loop.md-dynamic>>`. The sentinel is required — bare `/loop` prints usage.

## Default playbook (all on; toggle off with --no-<rule>)

- `tdd` — failing test → minimal impl → green → commit (logic changes)
- `self-test` — run tests locally before opening PR
- `local-ops-greenlight` — autonomous control of local services + Docker
- `review` — invoke `/review` skill on PRs with logic / new APIs / schema changes
- `strike-budget` — 3 attempts at a failure, then file issue and move on
- `pr-first` — branch → PR → squash-merge (vs direct-push to main)

Always-on (not toggleable): no `--no-verify`, no signing bypass, hook discipline, don't re-litigate locked decisions, keep docs in sync with code.

## Examples

    /unattended-loop                                  # all defaults, infer goal
    /unattended-loop wrap up the auth-refactor PRs    # explicit goal
    /unattended-loop --no-review on the docs cleanup  # disable /review
    /unattended-loop --no-tdd --no-strike-budget X    # multiple overrides

## Mid-flight redirect

While the loop runs, edit `<repo-root>/.claude/loop-brief.md`. The next tick reads it fresh. You can change the goal, scope, off-limits list, or toggle rules on/off via the `## Playbook overrides` section.

## Stop the loop

- Edit `loop-brief.md` to empty (next tick writes a final-state and exits), OR
- Interrupt during an active tick, OR
- Let it self-detect DoD and write `## Final state` to `loop-progress.md`.

## Full detail

`~/.claude/skills/unattended-loop/SKILL.md`
