# ATC: the orchestration role

One host session is named **ATC** (air traffic control). It does not do the work. It sends work
out to agents in [herdr](https://herdr.dev) panes, keeps the high-level talk with the user,
tracks the agents, relays decisions, and commits their output to `state/tasks`.

The goal is one stable place for decisions, apart from the doing.

## Rules

- **Delegate.** In ATC, prompt or spawn an agent rather than run a task in-session. Small
  checks are fine: status, reading an agent's output, committing.
- **Pick the kind of agent.** Default to iso-claude (the sandbox). Use a native host session
  only when the task needs host access or the web.
- **Notify with pointers, not copies.** When an agent's output is for the user to read, say
  where it is ("read tab X", "see `reviews/<date>/<name>/report.md`"). Do not repeat it in ATC.
- **Agents write files; the host commits.** Agents put their output in Markdown under
  `state/tasks`, e.g. `reviews/YYYY-MM-DD/<name>/`. That tree is visible from host and
  sandbox; screen reads of a pane are lossy. Agents do not commit there. ATC commits only the
  paths it means to: other files in `state/tasks` can be dirty for unrelated reasons.
- **Work items go through `tasks`.** An agent claims its TODO entry before it starts and sets
  its `Status:` when it stops (see [SPEC.md](SPEC.md)).

## Naming

Give each agent a short name for its task, upper case with dashes: `LOGIN-FIX`,
`CI-REVIEW`, `SEARCH-v2` for a version of a long-lived thread. Use one name for the
herdr agent, the Claude session (so `--resume NAME` works), and `tasks --as`.

## Spawn

Herdr finds an agent by the foreground process name of a pane. The `iso-claude` wrapper runs
Claude as `claude` (`exec -a claude`), so herdr sees it. `herdr agent start` only runs a plain
`claude`, so for iso-claude do it by hand:

1. Make a pane: `herdr tab create` or `herdr pane split`.
2. Start the agent: `herdr pane run <pane> "iso-claude [--resume NAME]"`.
3. Name it: `herdr agent rename <pane> NAME`.
4. Give it the task: `herdr agent prompt NAME "..."`. Say where to write output and where the
   stop point is (e.g. "do not push; reply with a summary").

The sandbox gets no herdr socket on purpose: with the socket, an agent could `pane run`
anything on the host.

## Replay into a tab

Text that ATC sends with `herdr agent prompt` shows in the target tab as plain user input: no
Markdown, tables do not render. To show content to the user in an agent's tab, ask the agent to
repeat it as its own reply.

## Reap

When an agent is done and its output is in files:

1. Ask it to exit (`/exit`), or send the keys.
2. Close the pane or tab only if ATC created it. Leave the user's own tabs alone.

A reaped agent can still be resumed by name.

## Respawn after a stop

Before a full stop (host reboot, container rebuild), write
`state/tasks/atc/respawn-YYYY-MM-DD.md`: one row per running agent with its name, kind (iso or
native), resume command and cwd. Also list agents reaped earlier that may come back.

To bring each one back, in a fresh pane (a new shell loads the current env):

- iso: `iso-claude --resume NAME`. iso sessions always start in `/opt/workspace`.
- native: `cd` to its own cwd first, then `claude --resume NAME-or-id`. Claude looks up
  sessions by the real directory, so a resume from another cwd does not find it.
- Then `herdr agent rename <pane> NAME`.

## Gotchas

- `ctrl+z` in an iso-claude pane suspends Claude inside the container, and the host shell's `fg`
  cannot reach it. Find the pid with `docker exec iso-claude ps -eo pid,args`, then
  `docker exec --user "$(id -u):$(id -g)" iso-claude kill -CONT <pid>`.
- A stray key can interrupt an agent mid tool call. If a call shows as rejected, check what
  actually ran before you assume nothing did.
