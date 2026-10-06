# ralph

An unattended loop that works a project toward a goal. Each step is a fresh
Claude Code session, running either `/planning` (plan one phase) or `/work`
(implement one task). Ralph alternates the two until the planner says the goal
is fully covered.

Ralph makes the choices a person would normally be asked about. Each one goes to
`NOTES.md`: the options he saw, the one he picked, why, and what changes if you
reverse it. You review the notes afterwards instead of answering questions while
the loop runs.

## Run it

```sh
cd ~/Projects/<project> && ralph   # run the loop for the project's one open goal
ralph <goal>                       # run it for that goal, when there are several
ralph tail                         # follow the running session: thinking, text, tool calls
ralph stop                         # stop the loop and everything it started (Ctrl-C works too)
```

The project is the git repository you run it in. A worktree counts as its main
checkout, so `~/Projects/clubs-services` plans into `clubs`. Code is worked on
the current branch.

## Layout

```
~/Projects/planning/<project>/
  <goal>/GOAL.md                   you write it (or /planning goal): frontmatter, then what to build, what is fixed, in what order
  <goal>/NOTES.md                  ralph writes it: every choice he made for that goal instead of asking
  phases/NN-<name>/goal.md         written by /planning, with a `**Goal:** <goal>` line naming the goal it serves
  phases/NN-<name>/NN-<task>.md    one task each, with frontmatter `status: todo|in_progress|done|blocked`
  SUMMARY.md                       written by /work: what the project is
```

A project has as many goals as you like, one directory each. Ralph works one at
a time. A goal keeps its directory when it ends, so you can write the next one
while ralph is still working the current one. Put everything that belongs to
the goal in its `GOAL.md` (the stack, what is fixed, the order to work in); the
script only holds the rules that apply to every project.

Phases stay in the project's one `phases/`, numbered across goals. Ralph works
the first task that isn't `done` in any phase, so an open task left behind by an
earlier goal gets finished first.

## Which goal

`ralph` with no name runs the project's one open goal: one that hasn't
`succeeded` or `failed`. If there are several open goals, it lists them and you
name one. `ralph <goal>` also resumes a `failed` goal once you've fixed what
failed it. It refuses one that `succeeded`.

## GOAL.md frontmatter

```markdown
---
status: todo             # ralph sets it, see below
max: 400                 # steps before it stops
model: claude-opus-5-5   # model for every step; this is the default
---

# Goal
...
```

All keys are optional. When ralph starts a goal, he writes in the default for
any of `max` and `model` it leaves out, so the goal shows what it runs with (an
env override is for that run only and is never written). Unknown keys are
ignored with a warning. Steps are told
the frontmatter is ralph's, not part of the goal.

A task can name its own model for its work step, in its frontmatter
(`model: claude-sonnet-5`). `/planning` does this for tasks that repeat a
pattern the repo already has, to stretch the usage limit. That model gets one
try. If the task is not done after it, the goal's model works it from there.

Ralph sets `status` and commits only `GOAL.md` each time:

| `status`    | when                                                                        |
| ----------- | --------------------------------------------------------------------------- |
| `todo`      | not started yet; the same as no status                                      |
| `running`   | ralph is working it                                                         |
| `stopped`   | `ralph stop` or Ctrl-C; run it again to carry on                            |
| `succeeded` | the planner found nothing left to plan; adds `finished:`                    |
| `failed`    | blocked, stuck or out of steps; adds `finished:` and `reason:` saying which |

## Each step

1. Find the first task that isn't `done`, in phase order.
2. If there is one, run `/work` on exactly that task. A task blocked on a
   decision gets decided, noted, rewritten and done.
3. If there is none, run `/planning` for the next phase of the goal. A phase
   belongs to the goal its `**Goal:**` line names. A phase without one dates from
   before goal directories, and the planner counts it if it serves the goal.
4. Stop when:
   - the planner answers `RALPH: NOTHING TO PLAN`: `succeeded`;
   - a blocked task is still blocked after a step that tried to decide it:
     something only a person can do, like a secret; its Notes say what. `failed`;
   - 3 steps in a row made no commit: stuck, see the last log. `failed`;
   - `max` steps have run. `failed`.
5. A step that runs out of Claude usage (the five-hour or the weekly limit) is not
   a step: ralph waits until the limit resets, then runs it again. It counts toward
   neither `max` nor the 3 steps without a commit. `ralph stop` ends the wait too.

## The rules every step gets

- No one is at the keyboard: never ask, decide and write it to `NOTES.md`.
- Everything is local: never push, deploy, open a PR or touch production.
- Never skip a problem. Fix it in the current task, or add a task right after
  it.
- `status: blocked` is only for what no local session can ever do.

## Install

Needs [Claude Code](https://claude.com/claude-code), `jq`, `flock` (util-linux),
and the `planning` and `work` skills in `~/.claude/skills/`.

```sh
git clone git@github.com:jevido/ralph.git ~/Projects/ralph
ln -sf ~/Projects/ralph/ralph ~/.local/bin/ralph
```

## Environment

`RALPH_MAX` and `RALPH_MODEL` override `max` and `model` from the frontmatter
for one run. The other two say where to find goals, so they can only be set
here.

| Variable         | Default                         |                                     |
| ---------------- | ------------------------------- | ----------------------------------- |
| `RALPH_MAX`      | `max`, else `400`               | steps before it stops               |
| `RALPH_MODEL`    | `model`, else `claude-opus-5-5` | model for every step, task models included |
| `RALPH_PROJECTS` | `~/Projects`                    | where project repositories live     |
| `RALPH_PLANNING` | `$RALPH_PROJECTS/planning`      | where the planning directories live |
| `RALPH_LIMIT_WAIT` | `600`                         | seconds to wait when out of usage with no reset time |

Logs are in `~/.local/state/ralph/<project>/`, one per step. `current.log`
points at the running one.

Steps run with `--permission-mode auto` and no MCP servers.
