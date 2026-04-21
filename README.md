# my-git

Multi-repo git dispatcher with recursive submodule support.

One command (`my-git`) walks a set of super-repos — and every nested
submodule — to show status, mass-commit changes, or register new nested
repos as submodules. Designed around the reality that a single tree can
span multiple git repos owned by multiple users and nested several levels
deep.

## Install

```sh
ln -s /path/to/my-git/my-git  ~/bin/my-git
my-git --create-config          # writes default config to first writable location
```

The script is POSIX `/bin/sh` (works under `dash`, `bash`, `zsh`).

## Configuration

Search order (first existing wins):

1. `$MY_GIT_CONFIG` env var (escape hatch)
2. `--config <FILE>` on the CLI
3. `/LINKS/default/my-git.conf`
4. `~/.mygit.conf`
5. `/etc/my-git.conf`
6. `/usr/local/etc/my-git.conf`

If none exist, `my-git` prints the search list and exits `2` with guidance
to run `--create-config`. Help and `--create-config` always work without a
config file.

### Variables (all optional; defaults shown)

| Variable                              | Default                                  | Meaning                                                                     |
|---------------------------------------|------------------------------------------|-----------------------------------------------------------------------------|
| `GIT_REPOS`                           | see default config                       | Fallback super-repo list (used only when no repo is scoped via CWD or args) |
| `CHECK_CLAUDE_SETUP`                  | `1`                                      | Audit `.claude/` during `mc`; `0` disables                                  |
| `CLAUDE_PROJECT_DIR_CENTRAL_REPO`     | `/LINKS/src/,claude-configs`             | Central tree where project `.claude/` dirs live                             |
| `COMMIT_MSG_LAST_N`                   | `3`                                      | Number of recent log entries in auto-commit body                            |
| `COMMIT_MSG_DIFF_MAX_FILES`           | `10`                                     | Above this count, commit body shows file list instead of diffs              |
| `ALLOW_SUDO_SU`                       | `1`                                      | Permit `sudo su <owner>` for cross-owner repos                              |
| `DEFAULT_VERBOSE`                     | `0`                                      | Set `1` to behave as `-V` by default                                        |
| `DEFAULT_DEBUG`                       | `0`                                      | Set `1` to behave as `-D` by default                                        |
| `SM_AUTO_SAFE_DIRECTORY`              | `1`                                      | In `sm`, auto-add `safe.directory` on errors                                |

## Quick start

```sh
my-git st                      # status of current repo + submodules (CWD-aware)
my-git st /LINKS/global        # status of a specific super-repo tree
my-git mc                      # mass add+commit+push of current repo + submodules
my-git sm                      # analyze nested unregistered repos
my-git sm go                   # register them as submodules
my-git claude-check            # audit .claude/ symlink convention
```

## Subcommands

| Subcommand      | Aliases        | What it does                                                                      |
|-----------------|----------------|-----------------------------------------------------------------------------------|
| `status`        | `st`, `s`      | Compact tree summary (one line per repo); `-V` = per-node porcelain listing       |
| `masscommits`   | `mc`, `c`, `go`| Bottom-up add + commit + push through submodule tree; runs `.claude` audit first  |
| `submodules`    | `sm`, `sub`    | Discover & register nested git repos as proper submodules                         |
| `claude-check`  |                | Audit `.claude/` symlink convention (read-only by default)                        |
| `help`          |                | Show top-level help                                                               |
| *(none)*        |                | `status`, paged through `less` when stdout is a TTY                               |

Run `my-git --help <subcommand>` (or `my-git <sub> --help`) for details.

## Verbosity & debug

Two independent axes. Debug implies verbose.

| Prefix  | Level    | Shown at            |
|---------|----------|---------------------|
| ` >>> ` | MAJOR    | always (milestones) |
| `  >> ` | medium   | `-V` `-VV` `-D` `-DD` |
| `    > `| minor    | `-VV` `-DD`         |
| ` ~~~ ` | debug    | `-D` `-DD` (stderr) |
| `  ~~ ` | debug    | `-D` `-DD` (stderr) |
| `   ~ ` | deep     | `-DD` only (stderr) |

`-DD` also enables `set -x` for deep tracing. Never design output that only
makes sense for a specific `-V + -D` combo — pick one axis per block.

## Status output

Default is a one-line-per-repo ascii-art tree. Each line renders the repo's
branch position in the tree followed by its state summary:

```text
/LINKS/global                    CLEAN :))
├── src                          DIRTY (18)  [M:1 ??:17]
│   ├── py/my-plex [unregistered]  CLEAN :))
│   └── sh/my-git [unregistered]   DIRTY (2)  [M:2]
└── etc                          CLEAN, ahead 1
```

States: `CLEAN :))`, `CLEAN, ahead N`, `CLEAN, behind N`,
`DIVERGED (ahead X, behind Y)`, `CLEAN (no remote for B)`,
`DIRTY (N) [M:.. A:.. D:.. R:.. ??:..]`, `[SKIPPED — cross-user policy]`,
`[ERROR …]`.

The tree includes **both** registered submodules and unregistered nested
git repos found on disk; unregistered ones are tagged `[unregistered]`.
Run `my-git sm go` to register them. Nested repos with
`.git/my-git-submodules.ignore` are suppressed from the tree.

`my-git -V st` falls back to the per-node porcelain listing (legacy form).

## Examples

```sh
# Status of a three-level nested tree (global → src → py/my-plex):
my-git st /LINKS/global

# Preview what mc would do (no writes):
my-git mc --dry-run

# Register everything + show what it did verbosely:
my-git -V sm go

# Deep debug on a single repo, no pager:
my-git -DD st /LINKS/global/src
```

## Scope resolution (st / mc / claude-check)

1. **Explicit `paths...`** — operate on each.
2. **No paths + CWD is inside a git repo** — operate on that repo + its submodules.
3. **No paths + not in a repo** — fall back to `$GIT_REPOS` from config.

## Privilege (sudo / su / policy)

When a repo's `.git` is owned by a different user than the caller,
`my-git` switches user **only for that repo**. Running as root always
works (plain `su <owner> -c ...`, no password). For non-root callers,
behavior is governed by two policy variables in the config:

| Caller    | Repo owner    | Mechanism                             |
|-----------|---------------|---------------------------------------|
| user      | same user     | direct exec — no escalation           |
| root      | user B        | `su B -c ...` — drops rights, no password |
| root      | root          | direct exec                           |
| user A    | root          | `PRIV_POLICY_USER_TO_ROOT` (default `warn`) |
| user A    | user B        | `PRIV_POLICY_USER_TO_USER` (default `fail`) |

Each policy variable takes one of:

| Value    | Behavior                                                                   |
|----------|----------------------------------------------------------------------------|
| `sudo`   | escalate via `sudo su <owner>` — requires sudo rights (macOS: `admin` / `wheel` / `sudo` group). If caller has no sudo rights, degrades to `fail`. |
| `fail`   | ERROR and abort the whole run.                                             |
| `ignore` | silently skip this repo; continue with the rest.                           |
| `warn`   | print a `WARN:` line to stderr and skip this repo; continue with the rest. |

So running `my-git` as root on a tree where most repos are user-owned
does **not** leave every git invocation running as root — each repo's
commands run as its actual owner. A non-root user running `my-git` over
a tree that mixes their own repos with a root-owned super-repo will by
default get warnings for the root-owned nodes and full operation on the
rest.

**Scope-level vs deep-level fail:** root-level policy violations (the
top of `GIT_REPOS` / an explicit argument) abort the whole run cleanly
via the parent shell. Policy violations discovered deeper in the
submodule walk can only be reported per-node (shell subshell limits);
those print `[ERROR]` and skip that subtree without aborting the run.

## `.claude` setup enforcement

`mc` (and the standalone `claude-check`) enforces the convention that each
project's `.claude/` is a **symlink** into `$CLAUDE_PROJECT_DIR_CENTRAL_REPO`,
not a real directory inside the project repo. On each repo:

1. `.gitignore` — must contain `.claude/` (auto-appended if missing).
2. `.claude` — classified:
   - Symlink into the central repo → OK (silent).
   - Symlink pointing elsewhere → MAJOR warning; no auto-fix.
   - Real dir with only `settings.local.json` (± other non-`settings.json`
     files) → auto-migrate: move into central repo, `rmdir`, create
     symlink.
   - Real dir containing `settings.json` → **hard error**; `settings.json`
     rules must be split manually (narrow → `~/.claude/settings.json`;
     blanket → central `settings.local.json`; one-offs → drop).

Disable with `CHECK_CLAUDE_SETUP=0` in the config.

## Extending

Subcommands are dispatched in `main()` via a single `case` on `cmd_*`
function names. To add one: write a `cmd_foo()` function plus a
`show_help_foo()` page, then register both in the dispatcher. The logging
helpers (`log_major`, `log_medium`, `log_minor`, `log_dbg_*`) and the
scope/privilege/walk primitives (`resolve_scope`, `run_as_owner`,
`walk_submodules`) are ready to reuse.
