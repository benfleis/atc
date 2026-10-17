# Task notes format

`bin/tasks` reads a tree of plain Markdown TODO files and treats them as a work queue. This file
describes that format. People and agents edit the files by hand; the tool only reads them, and
writes only the claims log.

## Layout

```
$TASKS_DIR/                 default /opt/workspace/state/tasks
  <topic>/TODO.md           one per long-lived topic
  atc/claims.log            who holds which entry
```

A topic is the name of its directory: `[A-Za-z0-9_.-]+`. Only `*/TODO.md` one level down is read.

## TODO.md

A TODO file has two parts, split by a line that is exactly `---`:

1. **Priorities.** A dated list of what to do next, newest thinking only. Rewrite it when the
   priorities change.
2. **Log.** A running list of entries, oldest first. New entries go at the bottom. Entries stay
   in the file when they close.

### Priorities

- A line that starts with `**P<n>**` (n = 0..9) opens priority level n. P0 is the highest.
- Any other line that starts with `**` closes the level.
- Under a level, each item is `- <date> <words>`, optionally followed by ` -- <note>`.
- An item names a log entry loosely: the entry with the same date and the most words in common
  with the item's text (`tasks check` reports an item that matches no entry). Use the entry's
  title, or a short form of it.

Everything else above `---` (headings, prose) is ignored.

### Log entries

```
## <date>: <title>

Status: <status>

<free text: source, context, findings, links>
```

- `<date>` is `YYYY-MM-DD`, `YYYY-MM`, or `undated`. Use the date of the finding.
- The first non-empty line after the heading must be the `Status:` line.
- `## ` lines are entry headings only after `---`; use `###` or lower inside an entry.

### Status

| value | meaning |
| --- | --- |
| `open` | work to do |
| `blocked-by <ref>[; <ref>]` | waits for the named entries; ready again when all of them close |
| `done <commit, PR or why>` | closed |
| `dropped <why>` | closed without doing it |

Change the status when the work changes it. Do not delete closed entries.

## Refs

A ref names one log entry across all topics:

```
<topic>/<date>: <title>
```

A case-insensitive prefix of the title is enough if it matches exactly one entry of that topic
and date; an exact title wins over prefixes. Refs appear in `blocked-by` and in the claims log.

## Claims

Claims are not in the TODO files. `atc/claims.log` is an append-only log, one event per line:

```
<utc-timestamp> <agent> claim|release <ref>
```

The last event for a ref wins. The tool writes the full title, takes a file lock while it
appends, and refuses:

- a claim on an entry another agent holds, or on a closed entry;
- a release by an agent that does not hold the entry.

Lines that do not have this shape (e.g. a `#` header) are ignored.

## Commands

The agent name comes from `--as NAME` or `$TASKS_AGENT`; it must not contain spaces.

| command | does |
| --- | --- |
| `tasks ready [-p N] [--all]` | lists priority items at level ≤ N (default 1) that are not claimed, blocked or closed, by level then topic. `--all` also shows the others, with their state. |
| `tasks claim REF --as NAME` | claims an entry. Do this before you start on it. |
| `tasks release REF --as NAME` | gives a claim back. Or close the entry with its `Status:`. |
| `tasks check` | reports entries without a `Status:` line, `blocked-by` refs that do not resolve, priority items that match no entry, and claims on refs that no longer resolve. Exits 1 if it finds any. Run it after each edit to a TODO file. |

## Example

`$TASKS_DIR/web/TODO.md`:

```
# web TODO

## Priorities (2026-10-17)

**P0**
- 2026-10-12 Login fails behind the proxy -- users are locked out.

**P1**
- 2026-10-15 Move CI to the new runner

---

## Log

## 2026-10-12: Login fails behind the proxy

Status: open

The proxy strips the `Authorization` header. Repro in `web/login-repro.md`.

## 2026-10-15: Move CI to the new runner

Status: blocked-by ops/2026-10-14: New runner pool

## 2026-10-16: Pin the node version

Status: done 3f2a9c1
```

```
$ tasks ready --all
P0  web        ready    2026-10-12: Login fails behind the proxy
P1  web        blocked  2026-10-15: Move CI to the new runner  <- ops/2026-10-14: New runner pool
$ tasks claim 'web/2026-10-12: Login fails' --as LOGIN-FIX
claim web/2026-10-12: Login fails behind the proxy
```
