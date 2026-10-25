# Beaver Backlog — Design

A task tracker that lives inside the project it tracks. One directory of
markdown files, a CLI, and a local web UI, all built for humans and coding
agents to use together. No external service, and no database.

This document records the decisions made before writing code. It is the
reference for what v1 is and is not.

## Goals

- Replace GitHub Issues / Jira / Linear / scattered notes for personal and
  small-team software projects.
- The backlog is plain files in the repo, version-controlled with the code.
- Humans and coding agents share the same tool through the CLI or the web UI.
- One static binary, no runtime dependencies, clean and small.

## Non-goals for v1

- JSON output mode, MCP server, or any programmatic API contract.
- Git integration of any kind (auto-commit, branches, hooks).
- Sprints, milestones, estimates, due dates, manual board ordering.
- Archive folder for finished tasks.
- Project configuration file.
- Compatibility with Backlog.md or other existing tools.
- npm / Homebrew packaging.

## Storage

### Location

- Directory `backlog/` at the project root.
- Every command except `init` locates it by walking up from the current
  directory, like git does. If none is found the command fails with
  `no backlog found, run beaver-backlog init`.
- The directory is the only source of truth. No cache, index, or database.

### Task files

- One file per task: `backlog/<id>-<slug>.md`.
- A file is a task only if its name matches `^[23456789abcdefghjkmnpqrstuvwxyz]{4}-.+\.md$`.
  Other files in the directory (such as `README.md`) are ignored.
- File content: YAML frontmatter followed by a free-form markdown body.
  There are no required sections or structure in the body.

### IDs

- 4 characters from the alphabet `23456789abcdefghjkmnpqrstuvwxyz`
  (lowercase, no `0/o`, `1/l/i`). Random, no prefix, no sequence.
- Parsed from the filename only; not duplicated in frontmatter.
- Generated at creation, regenerated on collision with an existing file.
- Random IDs mean parallel branches and parallel agent worktrees never mint
  the same ID.
- Every command that takes an ID accepts any unambiguous prefix.

### Slug and renames

- Slug derived from the title: lowercase ASCII, words joined by `-`,
  max 60 characters.
- Changing a task's title (CLI or UI) renames the file to match. Git detects
  this as a rename because the content barely changes.
- Agents are told to reference tasks by ID, never by path.

### Frontmatter schema

| Field        | Type            | Required | Notes                                        |
|--------------|-----------------|----------|----------------------------------------------|
| `title`      | string          | yes      |                                              |
| `status`     | enum            | yes      | `todo`, `in-progress`, `done`, `canceled`     |
| `priority`   | enum            | yes      | `low`, `medium`, `high`, `critical`           |
| `labels`     | list of strings | no       |                                              |
| `assignee`   | string          | no       | free text; a human name or an agent name      |
| `parent`     | task ID         | no       | makes this a subtask                          |
| `blocked_by` | list of task IDs| no       |                                              |
| `created`    | RFC 3339 UTC    | yes      | `2026-10-24T09:14:02Z`                        |
| `updated`    | RFC 3339 UTC    | yes      | rewritten on every write                      |

Rules:

- The tool writes fields in the order above.
- Optional fields are omitted when empty.
- Defaults on create: `status: todo`, `priority: medium`.
- Unknown keys are preserved untouched on every write, so people and agents
  can extend tasks with their own fields.
- "Blocked" is not a status. A task is blocked when any ID in `blocked_by`
  refers to a task that is not `done` or `canceled`.

Example:

```markdown
---
title: Fix login timeout on slow connections
status: in-progress
priority: high
labels: [auth, bug]
assignee: claude
blocked_by: [m4qd]
created: 2026-10-24T09:14:02Z
updated: 2026-10-24T11:40:19Z
---

Users on 3G see a spinner forever because the auth token request has a 5s
timeout. Raise it and show an error state.
```

## CLI

- Binary name: `beaver-backlog`. Users who want something shorter add a
  shell alias (`alias bbl=beaver-backlog`). No second binary is shipped.
- Human-readable output only. There is no JSON mode in v1; earlier
  experience showed agents read the markdown output more easily and the
  JSON was noisier. Programmatic consumers can be served later through
  the HTTP API or an added flag.
- Honors `NO_COLOR`. No spinners or interactive prompts.
- Exit code 0 on success, 1 on any failure.

### Commands

```
beaver-backlog init                      create backlog/ and write the AGENTS.md pointer
beaver-backlog add "title" [flags]       create a task, print its id
beaver-backlog ls [term] [filters]       list tasks
beaver-backlog show <id>                 print the task file verbatim
beaver-backlog edit <id> [flags]         change fields; with no flags, open $EDITOR
beaver-backlog start <id>                status -> in-progress
beaver-backlog done <id>                 status -> done
beaver-backlog cancel <id>               status -> canceled
beaver-backlog reopen <id>               status -> todo
beaver-backlog rm <id>                   delete the task file
beaver-backlog doctor                    validate the backlog directory
beaver-backlog serve [--port N] [--open] run the local web UI
```

### `init`

- Creates `backlog/` with a short `README.md` explaining the format. The
  README keeps the directory present in git while empty and is never
  parsed as a task.
- Appends a two-line pointer to `AGENTS.md` at the project root, creating
  the file if missing. Only `AGENTS.md` is written; Claude Code reads it
  when `CLAUDE.md` is absent, so one standard file is enough.
- Idempotent: running it again changes nothing.

### `add`

- `add "title"` creates the task and prints the new ID.
- Flags: `--body <text>` (or `--body -` to read stdin), `--priority`,
  `--label` (repeatable, sets the list), `--assignee`, `--parent`,
  `--blocked-by` (repeatable), `--status`.
- Body is empty unless provided. No template sections are seeded.

### `ls`

- One aligned line per task: `id  status  priority  title`.
  Labels and assignee are shown when filtering by them or with `-l`.
- Blocked tasks carry a marker. Subtasks are indented under their parent.
- Default sort: priority descending, then `created` ascending.
- `done` and `canceled` are hidden unless `--all`.
- Filters: `--status`, `--label`, `--assignee`, `--unblocked`,
  `--parent <id>`, plus an optional free-text term that matches titles
  only. Body search is left to `grep`.

### `show`

- Prints the file exactly as it is on disk: frontmatter and body, no
  reformatting, no colorization.

### `edit`

- Flags: `--title` (also renames the file), `--body` / `--body -`,
  `--status`, `--priority`, `--assignee`, `--parent` (`""` clears),
  `--label <x>` adds, `--unlabel <x>` removes, `--blocked-by <id>` adds,
  `--unblock <id>` removes.
- With no flags, opens the file in `$VISUAL`, then `$EDITOR`, then `vi`.

### Status shortcuts and `rm`

- `start`, `done`, `cancel`, `reopen` set the status and touch `updated`.
- `start` on a blocked task and `done` on a parent with open subtasks go
  through but print a one-line warning.
- `rm` deletes without confirmation. The file is under version control.

### `doctor`

Walks `backlog/`, parses every task file, prints findings grouped as
errors and warnings. Exit 1 if there are errors. Report only; no `--fix`.

Errors:

- Frontmatter does not parse.
- Missing required field.
- Invalid `status` or `priority` value.
- Duplicate ID across two files.
- Filename does not match `<id>-<slug>.md`.
- `parent` or `blocked_by` refers to a nonexistent ID.
- `parent` cycle.

Warnings:

- Slug does not match the current title (only possible via hand edits).
- `blocked_by` refers to a `done` or `canceled` task.
- Non-task files in the directory other than `README.md`.

### Malformed files in normal commands

`ls`, the web UI, and every other reader skip a file that does not parse,
print a one-line warning on stderr pointing at `doctor`, and continue.
Never crash, never silently hide.

## Agent integration

- `init` writes the pointer to `AGENTS.md`:

  ```
  This project tracks work with Beaver Backlog (tasks live in `backlog/`).
  Run `beaver-backlog --help` before reading or changing the backlog.
  ```

- The root `--help` text carries a short workflow section at the top
  (find unblocked work, `start` it, work, `done` it; reference tasks by
  ID; the body is free-form markdown you may edit directly) followed by
  the command reference. The help is the single entry point; nothing
  else is copied into the repo so nothing goes stale.
- Agents may edit task files directly with their file tools. The body has
  no imposed structure; a developer's prompts or skills decide whether to
  keep a progress log, acceptance criteria, etc.

## Web UI

Served by `beaver-backlog serve`.

- Binds `127.0.0.1` on port 4242 by default, falls to the next free port
  if taken so several projects can run side by side. `--port` overrides.
  Prints the URL; `--open` launches the browser.
- Views: board (one column per status), list, task detail.
- Full create, edit, delete.
- Board: dragging a card between columns changes only its status. Within
  a column cards are ordered by the selected sort; there is no manual
  ordering and no `rank` field.
- Filters (status, label, assignee, title search) and sort (priority,
  created, updated) are stored in the URL query string so views are
  bookmarkable. Nothing is stored server-side.
- Task detail: an edit form with current state and a save button.
  Status and priority as selects, labels as a tag input, assignee as
  text, parent as text with ID autocomplete. Body in a markdown WYSIWYG
  editor with a raw-markdown toggle.
- Body round-tripping: a WYSIWYG editor re-serializes markdown on save,
  which can normalize formatting. If the body was not edited, the
  original bytes are written back untouched. If it was, normalized
  formatting is accepted.
- Live updates: the server watches `backlog/` with fsnotify and sends a
  "changed" ping over Server-Sent Events. The UI re-fetches. Cards are
  keyed by ID so a renamed file re-renders in place.
- Conflicts: the UI sends the `updated` value it loaded; the server
  rejects the save if the file changed since and the UI shows
  "changed on disk, reload". No merging.
- Theme follows the system light/dark preference.

### HTTP API

- Small JSON API under `/api/` used by the SPA: list, get, create,
  update, delete, and the SSE event stream.
- Internal and unversioned in v1. Not a documented contract.
- All logic lives in the core package; the API and the CLI are thin
  interfaces over it. A future JSON mode or MCP server plugs in the same
  way.

## Stack

- Go 1.27, Cobra for the CLI, standard library `net/http` for the server.
- Viper is not used: there is no configuration to load.
- YAML through a library whose node API round-trips unknown keys and
  preserves order (`goccy/go-yaml` or `yaml.v3`, chosen at implementation).
- `fsnotify` for watching.
- Frontend: Vite, React 19, TypeScript, Tailwind 4, Base UI
  (`@base-ui/react`) for components, dnd-kit for the board,
  MDXEditor for the body editor. Preact was considered and rejected
  because Base UI does not support it; bundle size is irrelevant for a
  localhost tool.
- Frontend built to `web/dist` and embedded with `go:embed`. End users
  never need Node.

### Repository layout

```
cmd/beaver-backlog/   main package
internal/backlog/     core: discovery, parsing, writing, IDs, slugs, validation
internal/cli/         Cobra commands (thin layer over internal/backlog)
internal/server/      HTTP API, SSE, embedded static files
web/                  Vite + React app; web/dist is embedded
DESIGN.md             this file
justfile              build tasks
.goreleaser.yaml      release config
LICENSE               MIT
```

### Development workflow

- `just web` builds the frontend; `just build` builds the frontend then
  the Go binary. The Go build fails loudly if `web/dist` is missing.
- During UI development the Vite dev server proxies `/api` to a running
  `beaver-backlog serve`.
- Releases via goreleaser: linux, macOS, Windows on amd64 and arm64.
  Linux is the day-to-day tested platform. `go install` also works.

## Testing

- `internal/backlog`: golden-file tests for parsing and writing, table
  tests for ID and slug generation, validation tests mirroring `doctor`.
- `internal/cli`: command tests against a temporary directory.
- Frontend: type checking and lint only; no component tests in v1.

## Sibling project

Beaver Blueprint is a planned sibling tool shipped as its own binary from
its own repository. Beaver Backlog's internals are kept as clean packages
so shared code (markdown/frontmatter handling, directory discovery,
embedded-UI plumbing) can be extracted into a shared module later if
duplication becomes a problem.

## Prior art consulted

- Backlog.md: markdown tasks in `backlog/tasks`, Bun runtime, sequential
  IDs, web kanban, MCP. Borrowed: `init` writing an agent pointer, the
  one-file-per-task layout. Rejected: runtime dependency, sequential IDs,
  the larger model (docs, decisions, acceptance criteria).
- git-bug: issues stored as commit chains in git refs. Rejected: opaque
  to humans and agents.
- agent-browser: flat verb commands, help as the entry point for agents,
  terse output. Borrowed: those three.
