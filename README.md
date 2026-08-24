```
 ██████╗ ██████╗  ██████╗ ██╗   ██╗███████╗
██╔════╝ ██╔══██╗██╔═══██╗██║   ██║██╔════╝
██║  ███╗██████╔╝██║   ██║██║   ██║█████╗
██║   ██║██╔══██╗██║   ██║╚██╗ ██╔╝██╔══╝
╚██████╔╝██║  ██║╚██████╔╝ ╚████╔╝ ███████╗
 ╚═════╝ ╚═╝  ╚═╝ ╚═════╝   ╚═══╝  ╚══════╝
```

**Centralized git worktrees with tmux.**

Every branch gets its own directory under `~/.grove/worktrees/<repo>/<branch>` and
its own tmux window with a coding agent and an editor already open there. Your
repositories are never modified — no config file, no `.bare` layout.

Requires bash 3.2+, git 2.9+, tmux 3.0+.

## Install

```bash
curl -o ~/.local/bin/grove https://raw.githubusercontent.com/AngeloParrinello/grove/main/grove
chmod +x ~/.local/bin/grove
```

## Use

```bash
cd ~/projects/zio-blocks     # any ordinary clone
grove                        # list worktrees
grove feature-auth           # create worktree + branch + tmux window
grove main                   # opens the primary clone itself
grove rm feature-auth        # tear it all down
cd "$(grove path fix/schema)"
```

If a branch is already checked out in some worktree, grove opens that one;
otherwise it creates one, forking from `origin/HEAD` (re-running is idempotent).

## Commands

```
grove                          List this repo's worktrees
grove <branch>                 Create (if needed) and open a worktree
grove list                     Same as bare grove
grove rm <branch> [--force]    Remove worktree, branch, and window
grove path <branch>            Print a worktree's path
grove --version                Print version
grove --test                   Run the built-in test suite
```

`rm` refuses to run on a dirty worktree or an open window unless `--force`, and
never removes your primary clone.

## Environment

| Env var | Purpose | Default |
|---------|---------|---------|
| `GROVE_HOME` | Worktree root | `$HOME/.grove` |
| `GROVE_SESSION` | tmux session name | `grove` |
| `GROVE_EDITOR` | Editor command | `idea` |
| `CODING_AGENT` | Agent command | first found of `opencode`, `claude`, `codex`, `amp`, `aider`, `goose`, `gemini` |

## License

MIT — see [LICENSE](LICENSE).
