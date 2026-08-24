# grove — Agent Instructions

## What this repository is

`grove` is a single-file bash script (`grove`) that manages git worktrees in a
central root (`$GROVE_HOME/worktrees/<repo>/<branch>`) and opens each one in a
tmux window with a 2-pane layout (coding agent left, editor right).

It never modifies the repositories it operates on: no config file, no
restructuring, no `.bare` layout. Everything it needs is derived from
`git rev-parse`.

The repository also contains:
- `grove_test` — the test suite, sourced by `grove` to provide `--test`
- `README.md` — user-facing documentation

## Commands

```bash
grove                          # list this repo's worktrees
grove <branch>                 # create (if needed) and open a worktree
grove list                     # list this repo's worktrees
grove rm <branch> [--force]    # remove worktree, branch, and window
grove path <branch>            # print a worktree's path
grove --test                   # run the full test suite (must pass before committing)
grove --version                # print the current version
grove --help                   # usage
```

## Working with the codebase

The entire implementation lives in the single file `grove`. It is a bash
script — read and edit it directly. Do not create additional source files;
`grove_test` is the one exception and holds only tests.

**Before committing:**
- Bump `VERSION=` — patch (x.y.Z) for fixes, minor (x.Y.0) for features
- Run `grove --test` and confirm: `All N tests passed (0 skipped)`

**Do not:**
- Create additional source files
- Commit with failing or skipped tests
- Add per-repo configuration, scaffold hooks, or anything that writes into a
  user's repository — avoiding that is the point of the design

## Architecture

`grove` is organized top to bottom:

1. **Constants and helpers** — `VERSION`, `GROVE_HOME`/`WT_ROOT`,
   `DEFAULT_SESSION`, colors, `die`/`warn`/`info`/`dim`, `check_*`
2. **Repo identity** — `_git_common_dir`, `_repo_name`, `_slug`, `_worktrees`,
   `_wt_of_branch`, `_default_branch`, `_git_is_dirty`
3. **Tool resolution** — `_resolve_ide`, `_resolve_ai`
4. **Worktree resolution** — `resolve_worktree` (sets `WT_PATH`, `WT_WINDOW`)
5. **Tmux** — `_find_window`, `_setup_panes`, `launch_window`
6. **Subcommands** — `cmd_open`, `cmd_list`, `cmd_rm`, `cmd_path`
7. **Usage and dispatch** — `usage_main`, `main`
8. **Self-test** — `cmd_test()`, sourced from `grove_test`

### Invariants worth preserving

- **One porcelain parser.** `_worktrees` is the only place that parses
  `git worktree list --porcelain`. Everything else consumes its
  `<path>\t<branch>` output.
- **One resolution rule.** A branch already checked out anywhere resolves to
  that worktree; otherwise a new one is created. This is what makes
  `grove main` open the primary clone and re-runs idempotent. Do not add a
  second path that bypasses it.
- **Prune before lookup.** `git worktree prune` must run before
  `_wt_of_branch` in `resolve_worktree`, or a worktree whose directory was
  deleted by hand resolves to a dead path.
- **Windows are targeted by id, never by `session:name`.** tmux refuses
  ambiguous name targets and parses `.`/`:` in target strings. `_find_window`
  returns a `@id`; pass that to `-t`.
- **`_find_window` searches all sessions** (`list-windows -a`), so a window is
  found wherever it lives and never duplicated.
- **`GROVE_HOME` must not be `readonly`** — `GROVE_HOME=... grove foo` is a
  command-prefix assignment, which fails against a readonly variable.
- **`rm` never removes the primary clone**, even when asked for the branch
  checked out there.
- **A missing editor is not fatal.** `_resolve_ide` warns and returns empty;
  the agent pane is still worth having.

## Code conventions

- Subcommand functions: `cmd_<subcommand>`
- Helper functions: `_<name>` (underscore prefix)
- Fatal errors: `die`; non-fatal: `warn`; success: `info`
- `die` inside a command substitution only kills the subshell. Validate in the
  top-level shell (`check_repo`) or rely on `set -e` propagating a failed
  assignment.
- Deliberate simplifications are marked with a `ponytail:` comment
- Tests: `_t_check <desc> <cmd>` (pass if exit 0), `_t_grep <desc> <pattern>
  <cmd>` (pass if output matches), `_t_eq <desc> <expected> <actual>`
- Test section headers: `_section "XX1 — short description"` where `XX` is a
  2-letter prefix unique to that section

## Tests

Every new feature or behaviour change must be accompanied by tests in
`cmd_test()` — no exceptions. Add tests in a new or existing `_section` block
using the appropriate prefix, choosing the next available number
(e.g. if `RM3` exists, add `RM4`).

Current prefixes: `ID` (identity/slugs), `WT` (worktree creation), `LS` (list),
`RM` (removal), `TX` (tmux), `PA` (path), `ER` (errors), `AG` (agent/editor),
`MT` (meta).

**Test isolation rules — the suite runs against real git and real tmux:**

- Invoke child `grove` through the `_g` / `_grc` helpers only. They set
  `GROVE_HOME` and `GROVE_SESSION` to temp values, stub `CODING_AGENT=cat` and
  `GROVE_EDITOR=true` so no real IDE or agent launches, and pass `env -u TMUX`.
- `env -u TMUX` is not optional: the test runner is often *itself* inside tmux,
  and grove deliberately adds windows to the current session when `$TMUX` is
  set. Without it, tests spray windows into the user's live session.
- The temp dir is resolved with `pwd -P`. `git worktree list` reports physical
  paths (`/private/var/...` on macOS) while `mktemp -d` returns the `/var`
  symlink.
- Use `$self`, not `$0`, to re-invoke the script. `$0` may be relative and the
  helpers `cd` first.
- Do not assert on `#{pane_title}` — the pane's shell overwrites it. Assert on
  `#{pane_start_path}`, which comes from the `-c` grove passes.
