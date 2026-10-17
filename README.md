# atc

Air traffic control for a box where several Claude Code agents work in parallel: a work queue
over plain Markdown TODO files, and the conventions for one session that hands out the work.

- **`bin/tasks`**: lists ready work, records claims, and checks TODO files for refs and
  `Status:` lines that do not resolve. A single Python 3 file, standard library only.
- **[SPEC.md](SPEC.md)**: the TODO file format that `tasks` reads.
- **[ATC.md](ATC.md)**: the orchestration role: one host session delegates to agents in herdr
  panes and keeps the high-level talk.

## Layout

```
bin/tasks     the queue tool
SPEC.md       TODO.md format, Status values, refs, claims, commands
ATC.md        how the ATC session spawns, tracks, reaps and respawns agents
```

## Install

The tool reads `$TASKS_DIR` (default `/opt/workspace/state/tasks`). Put `bin/` on your `PATH`:

```sh
export PATH="/opt/workspace/config/atc/bin:$PATH"
```

or link the tool into a directory that is on it:

```sh
ln -s /opt/workspace/config/atc/bin/tasks ~/.local/bin/tasks
```

Keep the checkout where agents in a sandbox can read it but not write it: host sessions run
this code.

## Use

```sh
tasks ready                       # P0 and P1 work that nobody holds
tasks claim 'web/2026-10-12: Login fails' --as LOGIN-FIX
tasks check                       # after each edit to a TODO file
```

See [SPEC.md](SPEC.md) for the rest.
