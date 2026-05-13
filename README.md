# unattended-loop

A [Claude Code](https://claude.com/claude-code) skill for running Claude unattended on a planned work track — overnight, day-trip, multi-hour meeting block.

Wraps Claude Code's native `/loop` primitive with a 4-file harness and a baked-in engineering playbook (TDD, `/review` on meaningful PRs, PR-first workflow, 3-strike failure budget, no-stop policy) so you don't retype your standards every session. You drop the harness, brief the goal, walk away. Claude executes — and if you change your mind mid-flight, you edit a single file (`loop-brief.md`) and the next tick picks it up.

## TL;DR — when to invoke this skill

This skill is the **executor** at the tail end of a deliberate engineering pipeline. It assumes the upstream thinking is already done. If you invoke it on half-baked input, you'll get half-baked output at scale.

**Recommended pipeline** (each step is a separate, finished activity before the next begins):

| # | Step | Skill / activity | What you leave behind |
|---|---|---|---|
| 1 | **Design** | [`brainstorming`](https://github.com/obra/superpowers/tree/main/skills/brainstorming) — explore intent, requirements, shape of the solution across multiple rounds | A comprehensive design document, committed |
| 2 | **Architecture review** | An architecture-critic agent (if your project ships one) + the `/review` skill on the design itself | Settled architectural decisions; a "do not re-litigate" set recorded in the design doc |
| 3 | **Plan** | [`writing-plans`](https://github.com/obra/superpowers/tree/main/skills/writing-plans) — convert the design into an executable plan; iterate with review | Phased plan with explicit dependencies, scope carve-outs, and definition-of-done per phase |
| 4 | **Track** | File the umbrella issue + per-phase / per-task child issues in your tracker (GitHub Issues, Linear, JIRA, etc.) | A clean issue graph the loop can advance against, with `Ref:` / `Fixes:` IDs |
| 5 | **Commit upstream artifacts** | Push the design doc, plan, and any reference material to git | All planning artefacts in version control, referenced from issues |

**Then — and only then:**

```
/unattended-loop
```

The skill scans recent conversation context (design doc, plan, open issues, recent submodule activity), drafts a `loop-brief.md`, runs a pre-flight checklist, and hands you the trigger to fire. You walk away. Claude executes the plan against the tracked issues: TDD where applicable, PRs for every change, `/review` on meaningful ones, follow-up issues filed on findings, stops at the Definition of Done.

### Don't invoke this skill if…

| Symptom | What to do instead |
|---|---|
| The design isn't settled — open questions remain | Run [`brainstorming`](https://github.com/obra/superpowers/tree/main/skills/brainstorming) first |
| The plan doesn't exist yet | Run [`writing-plans`](https://github.com/obra/superpowers/tree/main/skills/writing-plans) first |
| The work isn't tracked anywhere | File the umbrella + child issues first |
| You want Claude to plan AND execute in one pass | Consider [`subagent-driven-development`](https://github.com/obra/superpowers/tree/main/skills/subagent-driven-development) — it's a different model where each task gets a fresh subagent. This skill prefers continuity (state across tasks, in-flight PRs to watch, mid-flight redirect via the brief). |
| The task fits in one session anyway | You don't need this skill — just work directly. The overhead only pays off for multi-hour autonomous runs. |

## What it does

When invoked, the skill drops 4 files into `<your-repo>/.claude/`:

| File | Purpose |
|---|---|
| `loop.md` | **Directive** — rules + per-tick recipe. Static across sessions. |
| `loop-brief.md` | **Session goal + scope.** The only file you edit between fires. |
| `loop-status.sh` | **State observer** — git branches, PRs, log tail. Static. |
| `loop-progress.md` | **Append-only checkpoint log.** Claude writes; you read. |

Then it outputs the trigger:

```
/loop <<loop.md-dynamic>>
```

The `<<loop.md-dynamic>>` sentinel is required — bare `/loop` prints usage. The skill walks you through this at handoff.

## Install

```bash
# Clone the repo
git clone git@github.com:endario/unattended-loop.git ~/unattended-loop

# Symlink into your Claude Code skills directory
mkdir -p ~/.claude/skills
ln -s ~/unattended-loop ~/.claude/skills/unattended-loop
```

Then in any Claude Code session:

```
/unattended-loop help
```

This prints a quick overview — what the skill does, available toggles, install / stop conditions — without writing any files.

## Usage

```
/unattended-loop                                  # infer goal from current conversation
/unattended-loop wrap up the auth-refactor PRs    # explicit goal
/unattended-loop --no-review on the docs cleanup  # disable /review playbook rule
/unattended-loop --no-tdd --no-strike-budget X    # multiple overrides
```

The skill auto-triggers on natural-language phrases too: *"set up an unattended loop on the present context"*, *"loop overnight on this"*, *"go autonomous while I'm out"*.

## Playbook (defaults — toggle off with `--no-<rule>`)

| Rule | Effect when on (default) |
|---|---|
| `tdd` | Logic changes follow failing-test → minimal impl → green → commit |
| `self-test` | Run project tests locally before opening a PR |
| `local-ops-greenlight` | Autonomous control of local services + Docker |
| `review` | Invoke `/review` skill on PRs with logic / new APIs / schema changes |
| `strike-budget` | 3 attempts at a failure, then file issue + move on |
| `pr-first` | Branch → PR → squash-merge (vs direct-push to main) |

Always-on (not toggleable): no `--no-verify`, no signing bypass, hook discipline, don't re-litigate locked decisions, keep docs in sync.

See [`SKILL.md`](SKILL.md) for the full setup procedure and [`templates/help.md`](templates/help.md) for the printable quick reference.

## Mid-flight redirect

Edit `<your-repo>/.claude/loop-brief.md` at any time. Change the goal, the scope, off-limits items, or the playbook overrides. The next tick reads the brief fresh — no need to interrupt the loop.

## Stopping the loop

- Edit `loop-brief.md` to empty — next tick writes a final-state and exits.
- Interrupt during an active tick (Ctrl-C / ESC).
- Let Claude self-detect Definition-of-Done and write `## Final state` to the progress log.

## Updating

```bash
git -C ~/unattended-loop pull
```

The symlink updates automatically — no reinstall step.

## What this skill is and is not

**Is:** A thin wrapper around Claude Code's native `/loop` primitive, plus a session-brief file you edit live, plus a status observer, plus an append-only progress log.

**Is not:** A custom loop runtime. Native `/loop` handles scheduling, the [60s, 3600s] delay clamp, and compaction survival. This skill does not duplicate any of that.

**Is not:** A planner. The plan / spec must already exist. This skill executes a plan — pair it with [`writing-plans`](https://github.com/obra/superpowers/tree/main/skills/writing-plans) or your favourite planning workflow.

## Compatibility

- **Claude Code** with the native `/loop` primitive (standard in current builds).
- **macOS, Linux, WSL** — anywhere bash + git + gh CLI run.
- **Any git-tracked project.** The harness lives in `<your-repo>/.claude/` — works with monorepos, submodule trees, single-repo projects alike.

## License

[MIT](LICENSE) — use, modify, redistribute freely.

## Acknowledgments

Patterns borrowed from [obra/superpowers](https://github.com/obra/superpowers) (the `writing-plans` / `subagent-driven-development` skills) and [ethosengine/elohim](https://github.com/ethosengine/elohim)'s `agentic-developer` skill.

## Contributing

Issues and PRs welcome. The skill's playbook reflects one engineer's preferences — if there's an opinionated default that breaks your workflow, file an issue and let's talk through it. The `--no-<rule>` toggle mechanism is meant to be the escape hatch for most disagreements without forking the skill.
