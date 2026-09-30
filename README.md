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
cd ~/Projects/<project> && ralph   # run the loop for that project
ralph tail                         # follow the running session: thinking, text, tool calls
ralph stop                         # stop the loop and everything it started (Ctrl-C works too)
```

The project is the git repository you run it in. A worktree counts as its main
checkout, so `~/Projects/clubs-services` plans into `clubs`. Code is worked on
the current branch.

## Layout

```
~/Projects/planning/<project>/
  GOAL.md                          you write it: what to build, what is fixed, in what order
  NOTES.md                         ralph writes it: every choice he made instead of asking
  phases/NN-<name>/goal.md         written by /planning
  phases/NN-<name>/NN-<task>.md    one task each, with frontmatter `status: todo|in_progress|done|blocked`
```

Ralph refuses to start without a `GOAL.md`. Put everything that belongs to the
project in it (the stack, what is fixed, the order to work in); the script only
holds the rules that apply to every project.

## Each step

1. Find the first task that isn't `done`, in phase order.
2. If there is one, run `/work` on exactly that task. A task blocked on a
   decision gets decided, noted, rewritten and done.
3. If there is none, run `/planning` for the next phase of the goal.
4. Stop when:
   - the planner answers `RALPH: NOTHING TO PLAN`: done;
   - a blocked task is still blocked after a step that tried to decide it:
     something only a person can do, like a secret; its Notes say what;
   - 3 steps in a row made no commit: stuck, see the last log;
   - `RALPH_MAX` steps have run.

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

| Variable         | Default                     |                                            |
| ---------------- | --------------------------- | ------------------------------------------ |
| `RALPH_MAX`      | `60`                        | steps before it stops                      |
| `RALPH_MODEL`    | Claude Code's default       | model for every step                       |
| `RALPH_PROJECTS` | `~/Projects`                | where project repositories live            |
| `RALPH_PLANNING` | `$RALPH_PROJECTS/planning`  | where the planning directories live        |

Logs are in `~/.local/state/ralph/<project>/`, one per step. `current.log`
points at the running one.

Steps run with `--permission-mode auto` and no MCP servers.
