# grove

```
 ██████╗ ██████╗  ██████╗ ██╗   ██╗███████╗
██╔════╝ ██╔══██╗██╔═══██╗██║   ██║██╔════╝
██║  ███╗██████╔╝██║   ██║██║   ██║█████╗
██║   ██║██╔══██╗██║   ██║╚██╗ ██╔╝██╔══╝
╚██████╔╝██║  ██║╚██████╔╝ ╚████╔╝ ███████╗
 ╚═════╝ ╚═╝  ╚═╝ ╚═════╝   ╚═══╝  ╚══════╝
```

**Centralized git worktrees with tmux.**

Every branch you work on gets its own directory and its own tmux window, with a
coding agent and an editor already open in the right place. Worktrees all live
under one central root, so you switch branches without stashing and without
losing context.

Nothing is added to your repositories. No config file, no restructuring, no
`.bare` layout. Run it from inside any ordinary clone.

## Requirements

| Tool | Version | Purpose |
|------|---------|---------|
| bash | 3.2+ | Runtime |
| git | 2.9+ | Worktrees, remote-branch DWIM |
| tmux | 3.0+ | Named pane support |

One of these coding agents is auto-discovered (or set `$CODING_AGENT`):
`opencode`, `claude`, `codex`, `amp`, `aider`, `goose`, `gemini`

One of these editors is auto-discovered (or set `$GROVE_EDITOR`):
`idea` (IntelliJ IDEA). If none is found, that pane is left as a plain shell.

## Installation

```bash
curl -o ~/.local/bin/grove https://raw.githubusercontent.com/AngeloParrinello/grove/main/grove
chmod +x ~/.local/bin/grove
grove --version
```

## Quick start

```bash
cd ~/projects/zio-blocks     # any ordinary clone
grove                        # list worktrees
grove feature-auth           # create worktree + branch + tmux window
grove main                   # opens ~/projects/zio-blocks itself
grove rm feature-auth        # tear it all down
```

## How it works

### Directory structure

Your repositories stay exactly where they are and are never modified. Worktrees
are created under a single central root:

```
~/projects/zio-blocks/                    ← ordinary clone, untouched
~/projects/grove/                         ← ordinary clone, untouched

~/.grove/worktrees/
  zio-blocks/
    feature-auth/                         ← branch feature-auth
    fix-schema/                           ← branch fix/schema
  grove/
    refactor-list/
```

The repo name comes from git itself (`git rev-parse --git-common-dir`), so grove
works from the primary clone or from inside any worktree. Slashes in a branch
name become hyphens in its directory name; the branch keeps its real name.

### Branch resolution

There is one rule:

> If the branch is already checked out in some worktree, use that worktree.
> Otherwise create one under the central root.

Git allows a branch in exactly one worktree at a time, so this is also what
makes the tool safe. Useful consequences:

- `grove main` opens your primary clone, because that is where `main` lives.
- Re-running `grove feature-auth` switches to the existing window instead of
  failing or duplicating anything.

New branches are forked from `origin/HEAD` (falling back to the repo's own
`HEAD`). A branch that exists only on the remote is checked out with tracking
already set up.

### Tmux layout

One session holds every window, so all your repos and branches are one
`prefix + w` away from each other.

```
session: grove
  1  zio-blocks/main            → ~/projects/zio-blocks
  2  zio-blocks/feature-auth    → ~/.grove/worktrees/zio-blocks/feature-auth
  3  zio-blocks/fix/schema      → ~/.grove/worktrees/zio-blocks/fix-schema
  4  grove/main                 → ~/projects/grove
```

Windows are named `<repo>/<branch>` and are looked up by that name across every
session, so a window is never created twice. When you run grove from inside
tmux, the window is added to your *current* session and your other windows stay
put — no `switch-client`, nothing stolen.

Each window has two panes, both opened in the worktree directory:

```
┌─────────────────┬──────────────────┐
│                 │                  │
│  coding agent   │     editor       │
│  (pane 1)       │     (pane 2)     │
│                 │                  │
└─────────────────┴──────────────────┘
```

### Environment

| Env var | Purpose | Default |
|---------|---------|---------|
| `GROVE_HOME` | Worktree root | `$HOME/.grove` |
| `GROVE_SESSION` | tmux session name | `grove` |
| `GROVE_EDITOR` | Editor command | `idea` |
| `CODING_AGENT` | Agent command | first one found |

## Command reference

```
grove                          List this repo's worktrees
grove <branch>                 Create (if needed) and open a worktree
grove list                     List this repo's worktrees
grove rm <branch> [--force]    Remove worktree, branch, and window
grove path <branch>            Print a worktree's path
grove --version                Print version
grove --test                   Run the built-in test suite
```

### `grove <branch>`

The primary command. Creates the worktree and branch if needed, then opens or
focuses its tmux window.

### `grove rm <branch>`

Removes the worktree, deletes the branch, kills the tmux window, and cleans up
the per-repo directory once its last worktree is gone. It refuses to run if the
worktree has uncommitted changes or its window is still open; `--force`
overrides both. It will never remove your primary clone, even when you ask for
the branch that is checked out there.

### `grove path <branch>`

Prints a path and nothing else, for shell use:

```bash
cd "$(grove path fix/schema)"
```

For a branch that has no worktree yet, it prints where one would be created.

## What it deliberately does not do

Each of these would require the per-repo configuration this tool exists to
avoid:

- Copying ignored files (`.env`, `node_modules`) into new worktrees
- Initializing submodules
- Running setup/teardown hooks
- Syncing or rebasing branches for you

## Self-test

```bash
grove --test
```

## License

MIT — see [LICENSE](LICENSE).
