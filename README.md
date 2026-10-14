# qclaude

A single command for saving and resuming [Claude Code](https://claude.ai/code) sessions by directory.

## Overview

Claude Code sessions are ephemeral by default. `qclaude` lets you save a session ID against a directory, resume it later, and manage saved sessions from the command line — all through one tool, dispatched by command.

Running `qclaude` with no arguments opens an interactive table of every saved session, and most day-to-day work happens right there: `Enter` resumes the highlighted one, `e` edits it, `d` deletes it — no need to `cd` into the directory first or remember a separate command for each. The individual commands below still exist for direct, scriptable, single-purpose use.

| Command | Flag | Purpose |
|---------|------|---------|
| — | (bare `qclaude`) | Open the interactive table — resume/edit/delete from one view |
| `qclaude resume` | `-r` | Resume the saved session for the current directory directly, no table |
| `qclaude save` | `-s` | Save a session for the current directory |
| `qclaude list` | `-l` | List all saved sessions (non-interactive; the table's Active column already shows this) |
| `qclaude open` | `-o` | Same interactive table as bare `qclaude` |
| `qclaude edit` | `-e` | Edit session ID, or move/merge a directory (updates all Claude metadata) |
| `qclaude delete` | `-d` | Delete a saved session, or bulk-remove stale ones |
| — | `-h`, `--help` | Show usage |

Each command accepts either form (`qclaude -d --clean` and `qclaude delete --clean` are identical). Anything after the command is that command's own arguments, passed through unchanged.

Sessions are stored in `claude-sessions.db` (one `path|session-id` entry per line) alongside the script.

## Quick start

```bash
qclaude          # open the table: Enter resumes, e edits, d deletes
```

## Installation

Clone the repo and symlink `qclaude` into a directory on your `$PATH`:

```bash
git clone git@github.com:allsharer/qclaude.git ~/Dropbox/Bin/qclaude
cd ~/Dropbox/Bin/qclaude

ln -s "$PWD/qclaude" ~/bin/qclaude
```

Adjust the target directory (`~/bin`) to any directory already on your `$PATH`.

## Usage

### `qclaude -r` / `qclaude resume` — resume a session directly

```bash
qclaude -r [--resume|--continue] [extra claude args...]
```

Run this from a project directory to pick up where you left off, without going through the interactive table — useful for scripts, aliases, or muscle memory. (Bare `qclaude`, with no command at all, opens the table instead — see below.)

- **No flag** — if a session is saved for the current directory, runs `claude --continue --permission-mode bypassPermissions`. If no session is saved, starts a fresh `claude --permission-mode bypassPermissions` session.
- **`--resume`** — forces `claude --resume <session-id> --permission-mode bypassPermissions` using the saved session ID. If no session is saved for the current directory, runs plain `claude --resume --permission-mode bypassPermissions` so Claude's own picker is shown.
- **`--continue`** — forces `claude --continue --permission-mode bypassPermissions`, regardless of what is or isn't saved in `claude-sessions.db`.
- Any extra arguments are passed through to `claude`.

These are `resume`'s own flags (distinct from the `-r` that selects the `resume` command itself), named after the underlying `claude` flag each forces.

```bash
cd ~/myproject
qclaude -r
qclaude -r --resume
qclaude -r --continue
qclaude -r --model claude-opus-4-5
```

---

### `qclaude -s` / `qclaude save` — save a session

```bash
qclaude -s [session-id]
```

Run this inside a project directory after starting a Claude Code session.

- **With a session ID** — saves it explicitly for the current directory.
- **Without a session ID** — auto-detects the most recent session Claude Code has on record for the current directory.

```bash
cd ~/myproject
qclaude -s                            # auto-detect latest session
qclaude -s 248f0e73-e6ea-4311-...    # save a specific session ID
```

---

### `qclaude -l` / `qclaude list` — list sessions

```bash
qclaude -l
```

Prints all saved sessions with their status:

```
DIRECTORY                                                SESSION ID                            STATUS
---------                                                ----------                            ------
/home/trevor/.local/bin/qpass                            8fdab79f-9a63-4157-8cee-a334109a1b8b  active *
/home/trevor/myproject                                   9d9ce2bd-80ed-4483-bab1-5afb25b90d0e  ok
/home/trevor/oldproject                                  da97dddd-8bd7-4b86-865b-d76034abb57e  missing

  * = current directory
```

| Status | Meaning |
|--------|---------|
| `active` | A Claude Code process for this session is currently running |
| `ok` | Session file exists on disk, not currently active |
| `missing` | Session file not found (deleted outside of `qclaude -d`) |

---

### `qclaude` / `qclaude -o` / `qclaude open` — the interactive table

```bash
qclaude
qclaude -o     # identical — just the explicit form
```

Opens a ruled table of every saved session — **Directory**, **Session ID**, **Active** — and lets you act on the highlighted row directly:

| Key | Action |
|-----|--------|
| `↑`/`↓` | Move the selection |
| `PgUp`/`PgDn` | Page up/down |
| `Home`/`End` | Jump to the first/last entry |
| `Enter` | Resume the highlighted session |
| `e` | Edit the highlighted session (same menu as `qclaude -e`) |
| `d` | Delete the highlighted session (same confirmation as `qclaude -d`) |
| `q` | Quit |

Long lists scroll within a viewport that follows the selection, with "↑ N more above" / "↓ N more below" markers.

- A session already active is shown with `ACTIVE` in its row and can't be resumed from here — exit the running session first. (`e`/`d` on an active session still work and will refuse with the same error `qclaude -e`/`qclaude -d` give directly.)
- `e` and `d` drop back to a normal screen to run their confirmation/prompts (so answers go through your shell exactly as they would from `qclaude -e`/`qclaude -d`), then wait for `Enter` before returning you to a freshly-reloaded table — so a delete or a rename is reflected immediately.
- `Enter` resumes Claude Code in `--permission-mode acceptEdits`, cd'd into that session's directory. Your shell's own working directory is unchanged after you exit claude.

---

### `qclaude -e` / `qclaude edit` — edit a session

```bash
qclaude -e
```

- **If the current directory has a saved session** — edits that entry directly.
- **If the current directory is not in the database** — opens an interactive picker to choose which entry to edit.

After selecting an entry, choose what to update:

- **1) Edit session ID** — prompts for a new session ID. If a newer session exists in `~/.claude/projects/` for that directory it is offered as a suggestion; press `Enter` to accept or type a different ID.
- **2) Move directory** — prompts for a new directory path, then moves everything there in one step (see `qclaude -e --move` below for exactly what moves).

Active sessions cannot be edited. A session ID already saved for another entry will be rejected.

---

### `qclaude -e --move` — move a directory non-interactively

```bash
qclaude -e --move <source-dir> <destination-dir>
```

Moves a directory and everything Claude Code knows about it, so `claude --resume` / `claude --continue` work correctly from the new location afterward:

- the files themselves
- the `~/.claude/projects/` session data (all `.jsonl` transcripts, not just the one `qclaude` happens to have saved)
- the `cwd` field recorded inside each moved session file
- matching entries in `~/.claude/history.jsonl`
- the project's entry in the `~/.claude.json` registry
- its row in `claude-sessions.db`, if any

Works whether or not `<source-dir>` is currently saved in the database — it only needs to exist on disk and/or have Claude session data under `~/.claude/projects/`.

**If `<destination-dir>` already exists**, its contents are merged into rather than overwritten: files are moved in one by one. If any name would actually collide (same filename in both directories, or the destination is already a separately-tracked Claude project with its own session data), nothing is changed and an error explains what conflicts — safe to re-run after resolving it. When the destination is already a separately-tracked project with no filename conflicts, both sides' session history end up living together under the destination (multiple `.jsonl` transcripts, all resumable), and the destination's existing `claude-sessions.db`/`~/.claude.json` entries are kept while the source's are discarded (since a path can only have one entry).

```bash
qclaude -e --move ~/work/old-project-name ~/work/new-project-name
```

---

### `qclaude -d` / `qclaude delete` — delete a session

```bash
qclaude -d
```

- **If the current directory has a saved session** — shows the details and asks for confirmation before deleting.
- **If the current directory is not in the database** — opens an interactive picker listing all saved sessions. Use `↑`/`↓` to navigate, `PgUp`/`PgDn` to page, `Home`/`End` to jump, `Enter` to select, `q` to quit. Long lists scroll within a viewport that follows the selection.

Active sessions (currently running) cannot be deleted. Exit the Claude Code session first.

On confirmation, `qclaude -d` removes the database entry **and** deletes the entire `~/.claude/projects/<dir>/` directory (session history, memory, everything).

```bash
qclaude -d --clean
```

Removes every database entry whose status is `missing` (see `qclaude -l`) in one pass. Lists the sessions to be removed, asks for a single confirmation, then rewrites the database — active and `ok` sessions are left untouched, and since there's no project directory to match a missing session, only the database entry is removed.

---

### `qclaude -h` / `qclaude --help` — usage

```bash
qclaude -h
```

Prints the full command summary (the text at the top of this README's Overview table, plus each flag/word form) and exits.

## How it works

Claude Code stores session files under `~/.claude/projects/<encoded-path>/` where the directory path has `/` and `.` replaced with `-`. The `save` command's auto-detect reads the most recent `.jsonl` file from that directory to find the latest session ID.

Active session detection works by scanning `~/.claude/sessions/*.json` for a matching `sessionId` and confirming the recorded `pid` is still a running process.

## Legacy standalone scripts

`oclaude`, `sclaude`, `rclaude`, `lclaude`, `eclaude`, and `dclaude` — the original six standalone scripts `qclaude` replaces — remain in this repo unchanged and still work exactly as before. `qclaude` is the maintained, recommended entry point going forward; the six scripts have no planned removal date but are no longer being developed.

## Requirements

- [Claude Code](https://claude.ai/code) CLI (`claude`)
- Python 3.8+ (stdlib only — no third-party packages)

The legacy standalone scripts additionally require bash 4+ and `awk`, `find`, `grep`, `python3`, `sed`, `tput`.
