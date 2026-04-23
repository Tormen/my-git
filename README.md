# my-git

Multi-repo git dispatcher with recursive submodule support.

**Purpose:** manage a multitude of git repos — plus nested git trees and
repos that use submodules — easily, from a single command. Designed
around the reality that a single tree can span multiple git repos owned
by multiple users and nested several levels deep.

`my-git` gives you:

- **`st`** — a recursive **overview** of every repo in a tree (status,
  divergence, registration state). Always walks the full submodule tree,
  because the whole point is to see everything in one glance.
- **`mc`** — mass add + commit + push. **Single level by default** (git's
  own scope); pass `-R` to walk the full submodule tree bottom-up so
  child commits bubble into parent gitlinks in one run.
- **`sm`** — register / rebind / clean up nested repos as proper
  submodules. **Single level by default**; pass `-R` to walk top-down
  through every registered submodule.

The split between "always recursive" (`st`) and "opt-in recursive"
(`mc` / `sm`) mirrors git itself: a super-repo's index only records its
own submodules' commit SHAs. What lives *inside* a submodule is that
submodule's responsibility, not the super's. `-R` is the escape hatch
for "I want ONE command to settle a whole nested tree."

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
my-git st                      # recursive OVERVIEW of current tree (CWD-aware)
my-git st /LINKS/global        # recursive overview of a specific super-repo tree
my-git mc                      # commit+push the CURRENT repo only (single level)
my-git mc -R                   # commit+push THIS repo + every submodule (bottom-up)
my-git sm                      # analyze nested unregistered repos (THIS level)
my-git sm go                   # register them (THIS level only)
my-git sm -R go                # register everything, walking the whole tree top-down
my-git claude-check            # audit .claude/ symlink convention
```

## Subcommands

| Subcommand      | Aliases        | Recursion | What it does                                                                      |
|-----------------|----------------|-----------|-----------------------------------------------------------------------------------|
| `status`        | `st`, `s`      | always    | Compact tree summary (one line per repo); `-V` = per-node porcelain listing       |
| `masscommits`   | `mc`, `c`, `go`| opt-in `-R` | Add + commit + push; `-R` = bottom-up walk through every submodule              |
| `submodules`    | `sm`, `sub`    | opt-in `-R` | Discover & register nested git repos; `-R` = top-down walk                      |
| `claude-check`  |                | always    | Audit `.claude/` symlink convention (read-only by default)                        |
| `help`          |                | —         | Show top-level help                                                               |
| *(none)*        |                | always    | `status`, paged through `less` when stdout is a TTY                               |

Run `my-git --help <subcommand>` (or `my-git <sub> --help`) for details.

### Why recursion is opt-in for `mc` and `sm`

Git's own model is: each super-repo is responsible only for its own index
and its own submodule gitlinks (one commit SHA per registered submodule).
What lives inside a submodule — further nested repos, sub-submodules,
dirty files — is that submodule's concern, not the super's.

`mc` and `sm` follow that model by default. Pass `-R` / `--recursive`
when you want one command to settle a whole nested tree:

- `mc -R` — **bottom-up**: deepest children commit+push first so each
  parent's gitlink lands on the freshly-pushed child commit in the
  same run.
- `sm -R` — **top-down**: parent rebinds and registrations settle first
  so children see a stable parent state.

`st` is always recursive because overview IS its job — a multi-level
tree seen at a glance, no walking required.

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

## `sm go` decision table

Every situation `sm go` can encounter, and what it does in default vs
`-i` mode. Anything not listed here is a bug — either in the code or in
this table. The table reflects **current** behavior, not aspirational
design.

| Situation                                                                    | `sm go` (default)                                 | `sm go -i`                                           |
|------------------------------------------------------------------------------|---------------------------------------------------|------------------------------------------------------|
| already registered (path + url match)                                        | no-op                                             | no-op                                                |
| nested inside registered submodule                                           | silent skip (counted in summary)                  | silent skip (counted in summary)                     |
| stale registration (`.gitmodules` entry, no dir on disk)                     | auto-remove                                       | prompt `[Y]es / [S]kip / [A]bort`                    |
| carries `my-git-submodules.ignore` marker                                    | silent skip; `--force` treats as fresh            | silent skip; `--force` treats as fresh               |
| marker + `.gitmodules` path mismatch (ignore here, registered elsewhere)     | `CONFLICT` note, skip                             | prompt `[U]pdate / [R]emove / [S]kip / [A]bort`      |
| moved (url matches a different registered path), nested repo **clean**       | rebind in place (no re-clone; all local state kept) | same (no extra prompt)                             |
| moved, nested repo **risky** (uncommitted+untracked, unpushed branches, stashes) | **skip** with "push/commit first" hint         | prompt `[P]roceed / [S]kip / [A]bort`                |
| no origin url on nested repo                                                 | silent skip (counted `missing origin URL`)        | prompt for URL                                       |
| unregistered with url, plain new                                             | `git submodule add`                               | same                                                 |
| unregistered with url, path already tracked in index (embedded)              | `UNTRACK` (`git rm --cached -r`) + `git submodule add` | same                                            |
| url looks suspicious (unusual scheme)                                        | add anyway (warn only, counted)                   | extra `[U]se / [E]dit / [S]kip / [I]gnore perm / [A]bort` prompt first |
| path matches a `.gitignore` pattern (pre-checked with `git check-ignore`)    | silent skip — `.gitignore` IS the marker (no `add` run) | prompt `[N]egate / [F]orce / [S]kip / [A]bort` — `[N]` appends `!<rel>/` to the matching `.gitignore`, then adds |
| stale `.git/modules/<name>/` from an interrupted prior run                   | **ERROR** + skip                                  | same; `--force` reuses the stale gitdir and adds     |

`[A]bort` in any `-i` prompt exits the whole run with code 77; under
`-R` it propagates up and terminates the entire recursive walk (not
just the current node).

`--force` effects, consolidated:

- bypasses per-repo `my-git-submodules.ignore` markers (treat as fresh).
- reuses a stale `.git/modules/<name>/` rather than erroring out.
- skips the suspicious-URL guard.

### Auto-commit at end of `sm go`

`sm go` commits **only** the paths my-git touched in this run:

- `.gitmodules`
- registered submodule gitlinks (new adds, rebinds, stale-removals)
- any `.gitignore` files where a negation was appended via `[N]egate`

Unrelated changes the user had staged or modified (other submodule
gitlink drift, unrelated `.gitignore` / `.DS_Store` edits, etc.) are
**left alone** — they stay staged/unstaged exactly as they were. The
commit is created with `git commit --only -- <paths>`, so the user's
other index state is preserved.

Commit subject example: `my-git sm: registered=2 stale-removed=1 gitignore-negations=1`.

Under `-i`, my-git prints the planned subject and path list and prompts
`[Y]es / [S]kip / [A]bort` before committing. Skipping leaves my-git's
changes staged for manual commit.

This also closes a prior gap where stale-section removal
(`git config -f .gitmodules --remove-section`) edited `.gitmodules`
without staging it, leaving the removal unstaged until the user
re-staged manually.

## Status output

Default is a one-line-per-repo ascii-art tree. Each line renders the repo's
branch position in the tree followed by its state summary:

```text
/LINKS/global                          CLEAN :))
├── src [registered]                   DIRTY (18)  [M:1 ??:17]
│   ├── py/my-plex [registered]        CLEAN :))
│   └── sh/my-git [unregistered]       DIRTY (2)  [M:2]
└── etc [registered]                   CLEAN, ahead 1
```

States: `CLEAN :))`, `CLEAN, ahead N`, `CLEAN, behind N`,
`DIVERGED (ahead X, behind Y)`, `CLEAN (no remote for B)`,
`DIRTY (N) [M:.. A:.. D:.. R:.. ??:..]`, `[SKIPPED — cross-user policy]`,
`[ERROR …]`.

The tree includes **both** registered submodules and unregistered nested
git repos found on disk. Marks shown after the display name make registration
state explicit:

- `[registered]` — nested git repo registered in `.gitmodules`; `sm` has
  nothing to do for this entry.
- `[unregistered]` — nested git repo not yet in `.gitmodules`. Run
  `my-git sm go` to register.
- `[registered, ignored]` — registered submodule **also** carrying the
  `my-git-submodules.ignore` marker. Conflict state; inspect with
  `my-git sm --list-ignored`.
- `[unregistered, ignored]` — unregistered nested repo carrying the ignore
  marker; `sm go` skips it and `st` does not descend into its subtree.
- `[STALE REGISTRATION]` — entry in `.gitmodules` whose on-disk path does
  not exist. Run `my-git sm go` to clean up (removes the `.gitmodules`
  section, the `.git/config` entry, the index entry, and
  `.git/modules/<name>/`).

The top-level scope path itself (root of each tree) carries no tag — it is
the super-repo, not a nested entry. DIRTY/CLEAN is orthogonal to
registration: DIRTY just means uncommitted file changes, handled by
`mc`, not `sm`.

`my-git -V st` falls back to the per-node porcelain listing (legacy form).

## Examples

```sh
# Status of a three-level nested tree (global → src → py/my-plex):
my-git st /LINKS/global

# Preview what mc would do for the CURRENT repo only (no writes, single level):
my-git mc --dry-run

# Preview what mc -R would do for the whole tree (no writes, bottom-up):
my-git mc -R --dry-run /LINKS/global

# Register everything (THIS repo only) + show what it did verbosely:
my-git -V sm go

# Register everything across the whole tree, top-down, verbose:
my-git -V sm -R go /LINKS/global

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
