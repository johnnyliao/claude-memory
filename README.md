# Claude Code — Auto Memory (Johnny)

Persistent context that Claude Code reads/writes across sessions: user profile,
project state, technical decisions, references to external systems. The intent
is that future Claude sessions on this account pick up the same context without
having to re-explain it.

## What's in here

| File | Purpose |
|---|---|
| `MEMORY.md` | Index — loaded into every session's context. Keep one line per entry. |
| `user_profile.md` | Johnny's role, dev setup, device, workflow preferences (cross-project). |
| `scoreboard_project.md` | Current state of the soccer scoreboard / YouTube RTMP streaming app. |
| `scoreboard_tech.md` | Technical decisions, failures, and the H.264 profileLevel root cause for the long-running 1080p bug. |

New project memories should be added as new `<topic>.md` files and registered with one line in `MEMORY.md`.

## How Claude finds this directory

Claude Code stores per-cwd memory under
`<HOME>/.claude/projects/<encoded-cwd>/memory/`, where `encoded-cwd` is the
absolute working-directory path at session start with separators replaced by
`-` (Windows `\` and Unix `/`). For example:

- Windows session started in `C:\Users\Johnny` → `C--Users-Johnny`
- macOS session started in `/Users/Johnny` → `-Users-Johnny`

This means the memory directory path **changes per machine and per starting cwd**.

## Cloning to a new machine

To make this memory available on another machine (or under another cwd), clone
this repo INTO the matching `.claude/projects/<encoded-cwd>/memory/` path:

```bash
# 1. Figure out the encoded cwd for the directory you start Claude Code in.
#    Example: starting from C:\Users\Johnny on Windows
ENCODED="C--Users-Johnny"

# 2. Make sure the parent path exists
mkdir -p "$HOME/.claude/projects/$ENCODED"

# 3. Clone into that path's memory/ subfolder
git clone https://github.com/johnnyliao/claude-memory.git \
  "$HOME/.claude/projects/$ENCODED/memory"
```

After cloning, the next Claude Code session started from that cwd will read
these memories.

## Keeping in sync

This repo is not auto-synced — Claude writes locally and we push on demand. To
sync changes from another machine before a session, just `git pull` inside the
memory dir. To save changes back, `git add . && git commit && git push`.

(Future improvement: a hook that auto-commits memory writes, but that hasn't
been set up.)
