# qclaude

Shell scripts for saving and resuming [Claude Code](https://claude.ai/code) sessions by directory.

## Overview

Claude Code sessions are ephemeral by default. These scripts let you save a session ID against a directory, resume it later, and manage saved sessions from the command line.

| Script | Purpose |
|--------|---------|
| `qclaude` | Interactive picker — browse and launch any saved session |
| `sclaude` | Save a session for the current directory |
| `rclaude` | Resume the saved session for the current directory |
| `lclaude` | List all saved sessions |
| `dclaude` | Delete a saved session |

Sessions are stored in `claude-sessions.db` (one `path|session-id` entry per line) alongside the scripts.

## Quick start

```bash
qclaude   # pick any saved session and jump straight in
```

## Installation

Clone the repo and symlink the scripts into a directory on your `$PATH`:

```bash
git clone git@github.com:allsharer/qclaude.git ~/Dropbox/Bin/qclaude
cd ~/Dropbox/Bin/qclaude

ln -s "$PWD/sclaude" ~/bin/sclaude
ln -s "$PWD/rclaude" ~/bin/rclaude
ln -s "$PWD/lclaude" ~/bin/lclaude
ln -s "$PWD/dclaude" ~/bin/dclaude
```

Adjust the target directory (`~/bin`) to any directory already on your `$PATH`.

## Usage

### `qclaude` — Session picker

```bash
qclaude
```

Opens an interactive picker listing all saved sessions. Use `↑`/`↓` to navigate and `Enter` to launch. Active sessions (already running) are shown with an `[active]` label and cannot be selected — exit the running session first.

On selection, Claude Code resumes in the project's directory. Your shell's working directory is unchanged after you exit.

```bash
ln -s "$PWD/qclaude" ~/bin/qclaude
```

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
rclaude [extra claude args...]
```

Run this from a project directory to pick up where you left off.

- If a session is saved for the current directory, runs `claude --resume <id>`.
- If no session is saved, starts a fresh `claude` session.
- Any extra arguments are passed through to `claude`.

```bash
cd ~/myproject
rclaude
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

### `dclaude` — Delete a session

```bash
dclaude
```

- **If the current directory has a saved session** — shows the details and asks for confirmation before deleting.
- **If the current directory is not in the database** — opens an interactive picker listing all saved sessions. Use `↑`/`↓` to navigate, `Enter` to select, `q` to quit.

Active sessions (currently running) cannot be deleted. Exit the Claude Code session first.

On confirmation, `dclaude` removes the database entry **and** deletes the entire `~/.claude/projects/<dir>/` directory (session history, memory, everything).

## How it works

Claude Code stores session files under `~/.claude/projects/<encoded-path>/` where the directory path has `/` and `.` replaced with `-`. The `sclaude` auto-detect reads the most recent `.jsonl` file from that directory to find the latest session ID.

Active session detection works by scanning `~/.claude/sessions/*.json` for a matching `sessionId` and confirming the recorded `pid` is still a running process.

## Requirements

- [Claude Code](https://claude.ai/code) CLI (`claude`)
- bash 4+
- Standard Unix tools: `awk`, `find`, `grep`, `tput`
