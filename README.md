# oclaude

Shell scripts for saving and resuming [Claude Code](https://claude.ai/code) sessions by directory.

## Overview

Claude Code sessions are ephemeral by default. These scripts let you save a session ID against a directory, resume it later, and manage saved sessions from the command line.

| Script | Purpose |
|--------|---------|
| `oclaude` | Interactive picker — browse and launch any saved session |
| `sclaude` | Save a session for the current directory |
| `rclaude` | Resume the saved session for the current directory |
| `lclaude` | List all saved sessions |
| `eclaude` | Edit session ID, or move/merge a directory (updates all Claude metadata) |
| `dclaude` | Delete a saved session |

Sessions are stored in `claude-sessions.db` (one `path|session-id` entry per line) alongside the scripts.

## Quick start

```bash
oclaude   # pick any saved session and jump straight in
```

## Installation

Clone the repo and symlink the scripts into a directory on your `$PATH`:

```bash
git clone git@github.com:allsharer/qclaude.git ~/Dropbox/Bin/qclaude
cd ~/Dropbox/Bin/qclaude

ln -s "$PWD/oclaude" ~/bin/oclaude
ln -s "$PWD/sclaude" ~/bin/sclaude
ln -s "$PWD/rclaude" ~/bin/rclaude
ln -s "$PWD/lclaude" ~/bin/lclaude
ln -s "$PWD/eclaude" ~/bin/eclaude
ln -s "$PWD/dclaude" ~/bin/dclaude
```

Adjust the target directory (`~/bin`) to any directory already on your `$PATH`.

## Usage

### `oclaude` — Session picker

```bash
oclaude
```

Opens an interactive picker listing all saved sessions. Use `↑`/`↓` to navigate, `PgUp`/`PgDn` to page, `Home`/`End` to jump to the first/last entry, `Enter` to launch, `q` to quit. Long lists scroll within a viewport that follows the selection, with `↑ N more above` / `↓ N more below` markers.

- Active sessions are shown with an `[active]` label and cannot be selected — exit the running session first.
- On selection, Claude Code resumes in `--permission-mode acceptEdits` with that session's directory as the working directory.
- Your shell's working directory is unchanged after you exit claude.

---

### `sclaude` — Save a session

```bash
sclaude [session-id]
```

Run this inside a project directory after starting a Claude Code session.

- **With a session ID** — saves it explicitly for the current directory.
- **Without a session ID** — auto-detects the most recent session Claude Code has on record for the current directory.

```bash
cd ~/myproject
sclaude                            # auto-detect latest session
sclaude 248f0e73-e6ea-4311-...    # save a specific session ID
```

---

### `rclaude` — Resume a session

```bash
rclaude [-r|-c] [extra claude args...]
```

Run this from a project directory to pick up where you left off.

- **No flag** — if a session is saved for the current directory, runs `claude --continue --permission-mode acceptEdits`. If no session is saved, starts a fresh `claude --permission-mode acceptEdits` session.
- **`-r`** — forces `claude --resume <session-id> --permission-mode acceptEdits` using the saved session ID. If no session is saved for the current directory, runs plain `claude --resume --permission-mode acceptEdits` so Claude's own picker is shown.
- **`-c`** — forces `claude --continue --permission-mode acceptEdits`, regardless of what is or isn't saved in `claude-sessions.db`.
- Any extra arguments are passed through to `claude`.

```bash
cd ~/myproject
rclaude
rclaude -r
rclaude -c
rclaude --model claude-opus-4-5
```

---

### `lclaude` — List sessions

```bash
lclaude
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
| `missing` | Session file not found (deleted outside of `dclaude`) |

---

### `eclaude` — Edit a session

```bash
eclaude
```

- **If the current directory has a saved session** — edits that entry directly.
- **If the current directory is not in the database** — opens an interactive picker to choose which entry to edit.

After selecting an entry, choose what to update:

- **1) Edit session ID** — prompts for a new session ID. If a newer session exists in `~/.claude/projects/` for that directory it is offered as a suggestion; press `Enter` to accept or type a different ID.
- **2) Move directory** — prompts for a new directory path, then moves everything there in one step (see `eclaude --move` below for exactly what moves).

Active sessions cannot be edited. A session ID already saved for another entry will be rejected.

---

### `eclaude --move` — Move a directory non-interactively

```bash
eclaude --move <source-dir> <destination-dir>
```

Moves a directory and everything Claude Code knows about it, so `claude --resume` / `claude --continue` work correctly from the new location afterward:

- the files themselves
- the `~/.claude/projects/` session data (all `.jsonl` transcripts, not just the one `eclaude`/`sclaude` happen to have saved)
- the `cwd` field recorded inside each moved session file
- matching entries in `~/.claude/history.jsonl`
- the project's entry in the `~/.claude.json` registry
- its row in `claude-sessions.db`, if any

Works whether or not `<source-dir>` is currently saved in the database — it only needs to exist on disk and/or have Claude session data under `~/.claude/projects/`.

**If `<destination-dir>` already exists**, its contents are merged into rather than overwritten: files are moved in one by one. If any name would actually collide (same filename in both directories, or the destination is already a separately-tracked Claude project with its own session data), nothing is changed and an error explains what conflicts — safe to re-run after resolving it. When the destination is already a separately-tracked project with no filename conflicts, both sides' session history end up living together under the destination (multiple `.jsonl` transcripts, all resumable), and the destination's existing `claude-sessions.db`/`~/.claude.json` entries are kept while the source's are discarded (since a path can only have one entry).

```bash
eclaude --move ~/work/old-project-name ~/work/new-project-name
```

---

### `dclaude` — Delete a session

```bash
dclaude
```

- **If the current directory has a saved session** — shows the details and asks for confirmation before deleting.
- **If the current directory is not in the database** — opens an interactive picker listing all saved sessions. Use `↑`/`↓` to navigate, `PgUp`/`PgDn` to page, `Home`/`End` to jump, `Enter` to select, `q` to quit. Long lists scroll within a viewport that follows the selection.

Active sessions (currently running) cannot be deleted. Exit the Claude Code session first.

On confirmation, `dclaude` removes the database entry **and** deletes the entire `~/.claude/projects/<dir>/` directory (session history, memory, everything).

```bash
dclaude --clean
```

Removes every database entry whose status is `missing` (see `lclaude`) in one pass. Lists the sessions to be removed, asks for a single confirmation, then rewrites the database — active and `ok` sessions are left untouched, and since there's no project directory to match a missing session, only the database entry is removed.

## How it works

Claude Code stores session files under `~/.claude/projects/<encoded-path>/` where the directory path has `/` and `.` replaced with `-`. The `sclaude` auto-detect reads the most recent `.jsonl` file from that directory to find the latest session ID.

Active session detection works by scanning `~/.claude/sessions/*.json` for a matching `sessionId` and confirming the recorded `pid` is still a running process.

## Requirements

- [Claude Code](https://claude.ai/code) CLI (`claude`)
- bash 4+
- Standard Unix tools: `awk`, `find`, `grep`, `tput`
