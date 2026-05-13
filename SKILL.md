---
name: unattended-loop
description: Use when the user is leaving Claude Code unattended (overnight, day-trip, multi-hour meeting block) and wants Claude to execute a planned work track autonomously until done or blocked. Trigger phrases include "set up an unattended loop on the present context", "loop overnight on this", "go autonomous while I'm out", or the slash command `/unattended-loop` (with or without a goal description). Bakes in TDD, /review, local-ops greenlight, PR-first, and a no-stop policy; drops a 4-file harness (`loop.md` directive, `loop-brief.md` session goal, `loop-status.sh` state observer, `loop-progress.md` log) and outputs the native `/loop` trigger.
---

# Unattended /loop Session

## Overview

Sets up an unattended `/loop` run on Claude Code's native file-mode primitive (`.claude/loop.md`). The skill bakes the user's standard playbook — TDD, self-test before PR, local-ops greenlight, /review after every major PR, PR-first workflow, no-stop policy — directly into the directive file so the user does not retype it per session.

**The user supplies at most one thing**: the session goal (or just invokes with no args and lets Claude infer it from current conversation context). DoD has a canonical default; the skill handles everything else.

**Announce at start:** "I'm using the unattended-loop skill to set up the autonomous run."

## Invocation

Either:

1. **Slash command**: `/unattended-loop` (optionally with a goal: `/unattended-loop on the auth-rewrite phase 3 track`).
2. **Natural language**: "set up an unattended loop on the present context", "loop overnight on this", "go autonomous while I'm out". The skill auto-triggers via description match.

With no goal specified, infer from the most recent conversation context (recent plan / design doc / issue mentioned; recent submodule touched; recent PR opened). With a goal specified, treat it as the `## Goal` paragraph in the brief.

## Arguments

The skill accepts arguments after the slash command:

- **Bare** — `/unattended-loop` — infer goal from conversation; all default rules active.
- **Goal text** — `/unattended-loop wrap up the auth refactor` — explicit goal; all default rules active.
- **Help** — `/unattended-loop help` (also `--help`, `-h`) — print the contents of `templates/help.md` verbatim, then stop. **No setup, no file writes.** This is the "show me what this skill does" path; recognise it early in step 1 and short-circuit before any other work.
- **Playbook toggle flags** — `--no-<rule>` — disable specific opinionated playbook rules. The disabled rule is rendered as "OFF" in the brief's `## Scope of autonomy → Playbook overrides` section; the directive honors it on every tick.

### Recognised toggle flags

| Flag | Default | Effect when disabled |
|---|---|---|
| `--no-tdd` | TDD on | Logic changes don't require a failing test first. |
| `--no-self-test` | on | OK to open a PR with locally-untested changes (rely on CI). |
| `--no-local-ops-greenlight` | on | `PushNotification` before any local service start/stop/restart instead of acting autonomously. |
| `--no-review` | on | Don't invoke the `/review` skill on meaningful PRs. |
| `--no-strike-budget` | on (3 attempts) | No retry cap; keep attempting until success or user intervention. |
| `--no-pr-first` | on | Direct-push to `main` is acceptable (project does not require PR-first workflow). |

Flags can be combined. Goal text may appear before, after, or interleaved with flags — strip the flags out and treat the rest as the goal.

**Always-on (not toggleable, no corresponding flag):** no `--no-verify`, no signing bypass, hook discipline, don't re-litigate locked decisions, keep docs in sync with code. These are baked into `loop.md` and the brief override section explicitly cannot disable them.

Natural-language invocation ("loop overnight on this") implies all defaults active. To override defaults, use the slash-command form with flags, or edit `## Playbook overrides` in the brief after setup.

## Default Definition of Done

**Use this verbatim** when the user did not specify a DoD on invocation:

> The entire track has been concluded — all checkboxes ticked in the plan / umbrella issue, all phase issues closed, smoke tests green where applicable. OR a genuine unresolvable blocker prevents further progress per `loop.md` §When to ping (after one `PushNotification` attempt). Normal back-pressure (CI flake, a bug you can debug, a test you can write) is not a stop condition — push through.

Only deviate from the default if the user explicitly stated a different DoD on invocation. When in doubt, use the default and note any specifics under `## Notes for future-you` in the brief.

## What this skill is and is not

- **Is**: a thin wrapper around Claude Code's `/loop` + `.claude/loop.md` primitive, plus a session-brief file the user edits to redirect mid-flight, plus a concrete bash status observer, plus an append-only progress log.
- **Is not**: a custom loop runtime. Native `/loop` already handles scheduling, the [60s, 3600s] delay clamp, and compaction survival (`resetAutonomousLoopDelivered` re-fires the preamble post-compact). This skill does not duplicate any of that.
- **Is not**: the planner. The plan / spec must already exist (run `writing-plans` first). This skill executes a plan.

## How the four-file harness works

```
<repo>/.claude/
  loop.md           ← directive: rules + per-tick recipe.  STATIC across sessions.
  loop-brief.md     ← session-specific: goal, DoD, scope.  THE ONLY FILE THE USER EDITS.
  loop-status.sh    ← state observer: branches, PRs, log tail.  STATIC.
  loop-progress.md  ← append-only checkpoint log.  Claude writes; user reads.
```

The native `/loop` (no args) reads `.claude/loop.md` on every tick. `loop.md` instructs Claude to read the brief + run the status script + advance one task + append to the progress log + schedule next tick. The brief is editable live — the next tick picks up changes.

Compaction is handled by the runtime; the harness needs no memory entry to survive it.

## Procedure (at setup)

### 1. Establish the session

**Parse arguments first.** Before any inference or file work:

1. If the first non-empty arg is `help`, `--help`, or `-h` (case-insensitive): read `~/.claude/skills/unattended-loop/templates/help.md` and print it verbatim to the user. Then **stop** — do not proceed with setup, do not write any files, do not run the status script. The user wanted to understand the skill, not run it.
2. Otherwise, scan the args for `--no-<rule>` flags (see "Recognised toggle flags" in `## Arguments` above). Record which rules are OFF for this session — these become the `{{PLAYBOOK_OVERRIDES}}` substitution in step 5. Unrecognised flags: surface them to the user as a warning ("ignored unknown flag: `--no-foo`") and proceed; do not abort setup over a typo.
3. The remaining text (with flags stripped) is the goal. If empty, fall through to goal inference below.

Default invocation: `/unattended-loop` or "set up an unattended loop on the present context" (no arguments). Infer from conversation:

- **What is being worked on** — most recent plan / design doc / umbrella issue referenced. If the user passed a goal arg, that overrides inference.
- **Affected repos** — most recent submodule(s) touched, or noted in the plan.
- **Definition of done** — use the canonical default from the "Default Definition of Done" section above unless the user explicitly stated otherwise on invocation.

If the **goal** is genuinely ambiguous AND the user is still present, ask once. Otherwise infer, document the inferences under `## Notes for future-you` in the brief, and proceed. The DoD never blocks setup — the default always applies if not stated.

Skip the conventional "are you sure?" — the user invoked the skill knowing what it does.

### 2. Self-summarize the context (the user will not raise these themselves)

This is the load-bearing step. The user is leaving — they will not volunteer watch-outs, scope carve-outs, or context-specific gotchas. **You must derive them now**, while context is fresh, and bake them into the brief. Future-you, six hours and one compaction event in, will not be able to reconstruct any of this.

Run this scan checklist. For each source, look for the listed patterns and route findings into the brief:

| Source | What to look for | Brief section |
|---|---|---|
| The plan / spike doc | Sections titled "open questions", "risks", "deferred", "phase N out of scope", "blockers", "assumptions"; numbered phases with dependencies; long-running operations (Docker pulls, model downloads, CI builds >10min) | `## Notes for future-you` |
| The design doc | Locked architectural decisions, ground rules marked "do not re-litigate" or equivalent; phases parked as "Phase N" / future scope | `## Off-limits without user input` (parked work) + `## Notes` (locked rules — restate succinctly) |
| Linked issues | Open vs closed status; cross-cutting issues filed as parked (e.g. "gates v2", "blocked on X"); umbrella issue checklists | `## Off-limits` (parked) + track section (open issues to advance) |
| Recent commits + open PRs | What just shipped (don't redo); what's mid-flight by another author (don't step on); branches in unusual states | `## Notes` |
| `MEMORY.md` + persistent project memory | Memory entries touching the affected modules — these codify gotchas you've already taught Claude once. Especially: known race conditions, test-flake patterns, deployment quirks, naming-convention traps | `## Notes` (cite memory entries by filename — future-you can read them in full if needed) |
| `CLAUDE.md` (root + module-level) | Module-specific sections for affected modules; sections that codify hard-won lessons (CI-watching patterns, pointer-bump conventions, local-services notes) | `## Notes` (cite the section, don't restate) |
| External-resource dependencies | Phases or tasks that require a console action the loop cannot self-serve (Auth0 console, AWS root, billing portal, vendor support ticket) | `## Notes` — flag explicitly: "this is a PushNotification trigger, do not retry" |

**Calibration**: if you find fewer than ~3 items, you didn't look hard enough — go back to the plan / design doc. If you find more than ~10, you're padding — cut to the items that would genuinely surprise a fresh Claude six hours in.

Format findings as terse bullets. Each bullet should be **actionable** ("Phase 3 Auth0 PEM flow may require console action — PushNotification trigger") not descriptive ("Phase 3 is about Auth0").

This output feeds the `{{NOTES}}`, `{{OFF_LIMITS}}`, and `{{PRE_APPROVED}}` substitutions in step 5.

### 3. Pick the repo

The default and strongly-preferred location is `<repo-root>/.claude/`. Use `~/.claude/` (home-level `~/loop.md`) only when the loop genuinely spans repos with no primary one.

`<repo-root>` = `$(git rev-parse --show-toplevel)` from the cwd. If multiple repos are in play (e.g. monorepo + submodule), put the harness at the parent repo level — the status script handles submodules.

**CRITICAL — the harness must live where `/loop` will fire, which is the user's normal cwd (typically the project root they opened CC from).** Native `/loop` reads `.claude/loop.md` from the cwd, not from arbitrary paths. Do NOT put the harness in a sibling git worktree expecting `/loop` to find it there — when the user types `/loop` from their normal project directory, it will print usage text and exit because no `.claude/loop.md` is present at that path. Isolation from the user's WIP belongs at the **work layer** (per-module on-demand worktrees, see step 3a), not at the harness layer. Harness lives where the user types `/loop`; the loop itself does its work in sibling worktrees.

### 3a. Worktree strategy for multi-submodule tracks

If the track spans multiple submodules AND the user has WIP on any of those submodules in their normal checkout, prefer **per-module on-demand sibling worktrees** over a parent-worktree-with-submodules-populated approach. The latter has two known failure modes:

1. **`git submodule update --init --recursive` inside a `git worktree add`'d parent is unreliable.** It can hang for minutes, fail silently leaving `modules/<name>/` empty, or get tangled by worktree-specific submodule git-dir paths. Even when it eventually succeeds, every submodule ends up at the parent-pinned SHA in detached HEAD — every one needs an explicit `git switch main` before work can begin. Avoid this whole class of problems by not populating submodules in a parent worktree at all.

2. **Submodules need `git submodule deinit --all -f` cleanup if the populate half-completes.** Dirty submodule entries from a failed init leave `git status` noisy on every loop tick.

**Per-module on-demand pattern instead:**

- Parent worktree (if needed for parent pointer-bump commits): `git worktree add --detach <path> origin/main`. Run `git submodule deinit --all -f` to silence the submodule-dirty status. This worktree has empty `modules/*` directories — that's fine, pointer bumps go through `git update-index --cacheinfo 160000,<sha>,modules/<name>` per the repo's `CLAUDE.md` §Pointer-bump pattern.
- Per-submodule worktrees, created on-demand when each phase needs that submodule:
  ```bash
  git -C <repo-root>/modules/<submodule> fetch origin main
  git -C <repo-root>/modules/<submodule> worktree add \
      -b feat/<track>-<phase-slug> \
      <worktrees-root>/<project>-<submodule>/<track>-<phase-slug> \
      origin/main
  ```
- Per-submodule worktrees are siblings of (not children of) the parent worktree. They use the submodule's own git directory, so they're independent of the user's main checkout's submodule WIP — both worktrees can exist on different branches simultaneously.

**When the simple approach (no worktrees) is fine:**

- Single-repo project (no submodules)
- All affected submodules have clean main checkouts with no user WIP
- The user is OK with the loop committing into their primary working tree

For these cases skip the worktree ceremony entirely. The worktree pattern is only worth the indirection when isolation is load-bearing.

**Documenting the pattern in the brief:**

When the per-module-worktree pattern is in use, the brief should contain a `## Worktree topology` section with the convention paths and the creation/cleanup recipes (the loop will need to reference these on every phase). The status script (`loop-status.sh`) should enumerate per-module worktrees under the convention path — NOT use `git submodule foreach`, which assumes submodules are populated in the parent.

### 4. Archive any prior session

Filenames are hardcoded by design — the native `/loop` binary reads `.claude/loop.md` by literal path, and `loop.md` in turn references `loop-brief.md` / `loop-progress.md` / `loop-status.sh` by literal path. One active loop per repo at a time. **The content is per-session; the filenames are constants.**

But that means we'd lose history if we overwrote a prior session's brief + progress log. So: archive first.

```bash
# Run from repo root. Skip silently if no prior session.
if [ -f .claude/loop.md ] || [ -f .claude/loop-brief.md ] || [ -f .claude/loop-progress.md ]; then
  slug="$(grep -oE '^# Session brief — .*' .claude/loop-brief.md 2>/dev/null | head -1 | sed -E 's/^# Session brief — //;s/[^a-zA-Z0-9-]/-/g;s/-+/-/g;s/^-|-$//g' | tr '[:upper:]' '[:lower:]')"
  : "${slug:=untitled}"
  ts="$(date '+%Y%m%d-%H%M%S')"
  dest=".claude/loop-archive/${ts}-${slug}"
  mkdir -p "$dest"
  for f in loop.md loop-brief.md loop-progress.md loop-status.sh; do
    [ -f ".claude/$f" ] && mv ".claude/$f" "$dest/"
  done
  echo "archived prior session to $dest"
fi
```

Tell the user where the archive went. If they invoked the skill expecting to *resume* an existing session rather than start a new one, stop and confirm — archival is for fresh starts.

### 5. Write the four files

For each template under `templates/` in this skill's directory:

| Template | Target | Substitutions | Notes |
|---|---|---|---|
| `loop.md.tmpl` | `<repo>/.claude/loop.md` | none | Copy verbatim. Rules + recipe baked in. |
| `loop-brief.md.tmpl` | `<repo>/.claude/loop-brief.md` | `{{TOPIC}}`, `{{GOAL_PARAGRAPH}}`, `{{PLAN_PATH}}`, `{{DESIGN_PATH}}`, `{{UMBRELLA_ISSUE}}`, `{{AFFECTED_REPOS}}`, `{{DEFINITION_OF_DONE}}`, `{{PLAYBOOK_OVERRIDES}}`, `{{OFF_LIMITS}}`, `{{PRE_APPROVED}}`, `{{NOTES}}` | `{{DEFINITION_OF_DONE}}` defaults to the canonical text in `## Default Definition of Done` above. `{{PLAYBOOK_OVERRIDES}}` rendering rules: if step 1 recorded no `--no-<rule>` flags, write the literal string `(none — all default \`loop.md\` rules active)`. If one or more flags were recorded, write one bullet per disabled rule in the form `- **<rule>: OFF** — <effect>` using the "Effect when disabled" column from `## Arguments` as the effect text. `{{OFF_LIMITS}}` / `{{PRE_APPROVED}}` default to `(none — defaults from loop.md apply)`. `{{NOTES}}` is where the step-2 self-summary bullets go. Use literal `(none)` for any other empty fields. The first-line `# Session brief — {{TOPIC}}` is what step 4's archival uses to slug the directory — keep it. |
| `loop-status.sh.tmpl` | `<repo>/.claude/loop-status.sh` | none | Copy verbatim, then `chmod +x`. |
| `loop-progress.md.tmpl` | `<repo>/.claude/loop-progress.md` | `{{TOPIC}}`, `{{START_TIMESTAMP}}` | Start timestamp: `date '+%Y-%m-%d %H:%M:%S %Z'`. |

Step 4 has already cleared any prior session into the archive, so these writes will not clobber history.

### 6. Verify the harness works

Before telling the user it's ready, run two checks:

**(a) Confirm `/loop` will find the harness from the user's normal cwd:**

```bash
# From the user's expected /loop firing location (their project root, NOT a worktree path):
ls <repo-root>/.claude/loop.md <repo-root>/.claude/loop-brief.md
```

Both files must exist. If you placed the harness in a sibling worktree, `/loop` will print usage text and exit when fired from the project root — fix this before handing off (move the harness, or document that the user must `cd` into the worktree before firing).

**(b) Run the status script:**

```bash
bash <repo>/.claude/loop-status.sh
```

Confirm the output shows current branch, recent commits, and "(loop-progress.md not found — this looks like the first tick)" or similar reasonable output. If the script errors, fix the cause (often: gh not authenticated, not in a git repo, missing `.claude/loop-brief.md`) before proceeding.

### 7. Pre-flight handoff (review brief + checklist)

Two parts: first show the user the brief, then walk the pre-flight checks.

**Show the brief**:

Print `<repo>/.claude/loop-brief.md`. The user should sanity-check the inferred goal, DoD, scope carve-outs, and (especially) the step-2 self-summary notes — they have the domain context to spot a misread. Remind them:

- The brief is the **only file they need to edit** if anything's off.
- They can edit it AT ANY TIME during the loop — the next tick reads it fresh.
- The four-file harness is at `<repo>/.claude/loop-*` for inspection.

**Pre-flight checklist**:

Run the checks below. Don't auto-fix machine state without permission — these are user decisions. For each item, report `[✓]` or `[✗] <what's wrong>`. The user fixes red items or explicitly accepts the risk.

- [ ] **Power**: laptop plugged in. On battery, the loop dies when it sleeps. Suggest `caffeinate -dimsu &` (terminal-bound) or System Settings → Lock Screen → "Prevent automatic sleeping on power adapter when display is off".
- [ ] **gh auth**: `gh auth status` returns a non-expired token. Status script + PR ops fail silently without it.
- [ ] **Disk**: if Docker / image-build / deploy-ops work is in scope, `df -h` shows >20 GB free on `/` and on the Docker volume. Docker pulls + image builds eat disk fast — overnight loops have run laptops out of space.
- [ ] **No active debugger**: if any locally-running services are about to be touched, no IDE-attached debugger with active breakpoints on those services. The loop will not pause on a breakpoint — it will hang or skip. (If the project's `CLAUDE.md` has a local-services section, consult it for the detection recipe.)
- [ ] **No mid-flight branches**: parent repo + affected submodules are on `main` with a clean working tree (or the user has explicitly told you to continue on a feature branch). Session-start drift is a known foot-gun for pointer-bump operations.
- [ ] **Tunnel / VPN if needed**: if the loop's tasks reach a remote environment via a Cloudflare tunnel, VPN, or SSH tunnel, confirm it's up before handing off. It will not auto-recover overnight.
- [ ] **Harness at the user's `/loop` firing location**: `.claude/loop.md` exists at `$(git rev-parse --show-toplevel)` of the user's normal cwd, NOT only in a sibling worktree. If unsure, ask the user where they will type `/loop` and confirm the harness is there.
- [ ] **Prior loop archived if reusing the same `.claude/`**: if you installed the new harness at the user's project root and a prior loop's files were there, confirm they went into `.claude/loop-archive/<timestamp>-<slug>/` (step 4's job). A stale `loop-brief.md` with `## Final state` already written will cause the new `/loop` to short-circuit and end immediately.

### 8. Output the trigger

```
/loop <<loop.md-dynamic>>
```

The prompt argument is **required** — bare `/loop` prints the usage text and does not enter dynamic mode. The sentinel `<<loop.md-dynamic>>` is what the runtime resolves to "fire `.claude/loop.md` on each tick, model-paced." Without it, `ScheduleWakeup` calls have no runtime to fire into — they are queued but orphaned, and the loop silently never ticks.

**This is the single most painful gotcha of this skill.** A loop that "looks set up" but never actually fires is identical from the user's vantage to a working loop — until they notice 30 minutes have passed with no progress. State this trigger exactly as written above. Do not paraphrase to "just `/loop`" or "press `/loop` and we're off" — both are wrong on the current binary.

How to recognise the failure mode if you ever encounter it again:
- Bare `/loop` returned the help/usage block (`Usage: /loop [interval] <prompt>`) instead of starting work.
- You called `ScheduleWakeup` and got back `Next wakeup scheduled for ...` but no tick fired at that time.
- The user comes back to ask "did it start?" — they didn't see ticks.

In that case: the loop was never running. The harness is fine; the user just needs to fire the correct trigger above. Tell them, don't waste cycles trying to chain via `ScheduleWakeup` from outside a live loop.

### 9. Closing message

One short paragraph:

- Harness written; brief location.
- Pre-flight checklist results.
- Trigger: `/loop <<loop.md-dynamic>>` (the literal sentinel — bare `/loop` only prints usage).
- Mid-flight redirect: edit `loop-brief.md`.
- Stop the loop: edit `loop-brief.md` to empty, OR interrupt during a fire, OR wait for Claude to detect DoD and write "## Final state".
- What they'll see when they return: `cat <repo>/.claude/loop-progress.md` for the at-a-glance log; PRs / commits / issue updates for the work itself.

## During loop execution (Claude on each /loop tick)

The full per-tick recipe lives in `loop.md` — read that file, not this section. This skill's SKILL.md is invoked only at setup; during ticks the runtime fires `loop.md` directly.

If you're seeing this section because someone invoked the skill again mid-loop (e.g. to re-setup or to update the brief), the workflow is:

- Re-setup (different goal): write a new `loop-brief.md` — do NOT touch `loop.md`, `loop-status.sh`, `loop-progress.md` (preserve continuity). Run the status script to verify.
- Update rules (rare — they're meant to be canonical): edit `templates/loop.md.tmpl` in this skill directory, then copy to `<repo>/.claude/loop.md`. The next tick picks up the new rules.

## Anti-patterns

- **Don't write rules inline into a one-shot `/loop <long prompt>`.** It works for the first fire but ages badly. Use `loop.md`.
- **Don't pick a fixed interval** (`/loop 5m ...`) for heterogeneous work. Different waits have different optimal delays — `<<loop.md-dynamic>>` self-paces.
- **Don't author a parallel loop runtime.** Native `/loop` already solves scheduling + compaction survival.
- **Don't ping the user for trivial decisions** during an unattended run. That defeats the point. See `loop.md` §When to ping for the three legitimate triggers.
- **Don't put rules in the brief.** Rules belong in `loop.md` (canonical, static); the brief is for *what changes* (goal, DoD, scope carve-outs).
- **Don't auto-stage a memory entry just for compaction survival.** The runtime handles it. Memory is for durable cross-conversation knowledge, not session state.
- **Don't put the harness in a sibling worktree expecting `/loop` to fire there.** Native `/loop` reads `.claude/loop.md` from the user's cwd (typically their project root, not arbitrary paths). If you set up the harness in `<worktrees-root>/<project>/<slug>/.claude/`, `/loop` from the user's normal directory will print usage text and exit because no harness is present at that path. **Harness lives at `$(git rev-parse --show-toplevel)` of the user's normal cwd.** Use sibling worktrees for isolating the *work*, not the harness.
- **Don't populate submodules inside a parent worktree.** `git submodule update --init --recursive` after `git worktree add` is unreliable — it can hang for minutes, fail silently with empty `modules/*` directories, or leave half-populated state that pollutes `git status` on every loop tick. Use the per-module on-demand worktree pattern (step 3a) instead, with `git submodule deinit --all -f` on the parent worktree to silence dirty-submodule noise.
- **Don't tell the user "fire `/loop`" without the sentinel.** Bare `/loop` prints usage; only `/loop <<loop.md-dynamic>>` enters dynamic mode. `ScheduleWakeup` only chains ticks within an already-running dynamic-mode `/loop` — if the loop was never actually launched, your `ScheduleWakeup` calls are orphaned and the user will see no progress despite "Next wakeup scheduled" responses. See step 8 for the recognition cues.

## Related skills

- `writing-plans` — produces the plan this loop executes.
- `subagent-driven-development` — alternative execution model where a fresh subagent runs each task. Use when each task is large enough to merit a clean context; use this skill when continuity (state across tasks, in-flight PRs to watch, mid-flight redirect) matters more than fresh context per task.
- `review` — the directive invokes this on meaningful PRs. `live-ops`, `wrap-up` — referenced by the directive's cheatsheet if your project has analogous deploy-ops / PR-wrap-up skills; otherwise harmless (the directive's wrap is "if your project has…").

## References

- Native primitives: `<<loop.md>>` / `<<loop.md-dynamic>>` / `<<autonomous-loop>>` / `<<autonomous-loop-dynamic>>` (binary symbols in `~/.local/bin/claude`).
- Compaction-survival primitive: `resetAutonomousLoopDelivered` (binary).
- Skill-authoring conventions: `obra/superpowers/skills/writing-skills/SKILL.md`.
- Repo workflow this skill layers in: any sections of the project's `<repo>/CLAUDE.md` covering PR / direct-push policy, test discipline, CI-watching patterns, local-services notes, and UI verification — referenced by the directive when present.
