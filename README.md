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
- **`mc`** — mass add + commit + push. **Two-phase like `sm`**: `mc`
  alone analyzes (one concise line per repo, no writes); `mc go` applies
  it. Single level by default; pass `-R` for a bottom-up walk so child
  commits bubble into parent gitlinks in one run. Commit message is
  generated automatically (never opens `$EDITOR`).
- **`sm`** — register / rebind / clean up nested repos as proper
  submodules. **Single level by default**; pass `-R` to walk top-down
  through every registered submodule.
- **`flatten`** — fold nested git repos into the outer repo's history as
  plain tracked content. **Three equal modes**, one per nested repo,
  differing only in what happens to its git dir: **`--merge`** (stays live
  in place, or is deleted), **`--sidecar`** (relocated into a
  self-contained `.git.real` capsule — portable, full history preserved,
  but no longer live/IDE-usable), or **`--zip`** (archived into a compact
  `.git.zip`/`.git.real.zip` and removed). Any mode can be forced in bulk
  or on a single path; see [`flatten`](#flatten--merge-nested-repos-into-the-outer-repo-as-content) below.
- **`unflatten`** — the reverse: rebuild a live `.git` at a nested path.
  `--sidecar` and `--zip` are exact, lossless restores; `--merge` is a
  best-effort reconstruction from this repo's own commit history (via `git
  subtree split`) for a path whose original `.git` was deleted. Auto-detects
  the mode per path. See [`unflatten`](#unflatten--rebuild-a-live-git) below.

The split between "always recursive" (`st`) and "opt-in recursive"
(`mc` / `sm`) mirrors git itself: a super-repo's index only records its
own submodules' commit SHAs. What lives *inside* a submodule is that
submodule's responsibility, not the super's. `-R` is the escape hatch
for "I want ONE command to settle a whole nested tree."

## Nested repos: three cases

Every nested `.git` in a tree my-git manages is exactly one of three
cases, and each has exactly one right answer:

| # | Case | Meaning | Handling |
|---|------|---------|----------|
| 1 | **Archived** | Kept only for its history; never developed or pushed to again. | `my-git flatten go <path> --zip` — archive its `.git` (or a `.git.real` sidecar, `+.status`) into a single `.zip` the parent tracks as content (`st` shows it `[zipped]`); `unflatten --zip` restores it, `-k`/`--keep` archives alongside instead of replacing. The `gz`/`gu` shell aliases (`etc/zshrc.mine`) do the same thing interactively without my-git. |
| 2 | **Private** | Developed here, but never published outside this machine (no real origin). | `my-git flatten go` merges it into the parent as plain content and drops its `.git`. No separate history is kept — a repo that's never published shouldn't have had its own identity to begin with. |
| 3 | **Published** | Developed here AND pushed to its own origin (e.g. GitHub). | Toggle between `flatten --sidecar` (relocate its `.git` into a self-contained `.git.real` alongside the parent-tracked files) and `unflatten --sidecar` (bring it back to its normal live `.git` to resume developing/pushing) as needed — the parent still fully captures its files either way, not just its origin URL, so a fresh clone of the parent needs no extra `submodule update` step. |

Concrete examples on this machine:

- **`/learn`** — every nested repo is case 1 (e.g. `it/scala`): cloned
  once for reference, nothing here is developed further. `gz`/`gu` alone
  covers it; `my-git` has nothing to do.
- **`/LINKS/global/src`** — has all three side by side:
  - Case 1 (third-party, reference only): `,github/dotfiles`,
    `,github/abootool`, `,gitlab/pulseaudio`, `c/squashfuse`,
    `rust/,i3status-rust`, `,github/TuneIn-Radio-VLC`.
  - Case 2 (private, no origin): `icfp/icfp`, `icfp/mine`,
    `py/imap-check-mail`, `sh/lc`, `sh/debian-dependencies`,
    `sh/build-livecd`, `sh/trash`, `c/keyboard-logger`.
  - Case 3 (own repos, published): this repo (`sh/my-git`),
    `py/my-plex`, `py/my-nimbie`, `sh/my-handbrake`, `sh/my-appleRAID`,
    `,github/create_ap`, `,github/mkinitcpio-ykfde`,
    `,github/rsync-tmbackup___rsync-time-backup`.

## Install

```sh
ln -s /path/to/my-git/my-git  ~/bin/my-git
my-git --create-config          # writes default config to first writable location
```

The script is POSIX `/bin/sh` (works under `dash`, `zsh`, and — verified
directly by compiling and testing against it — Apple's frozen bash 3.2,
which is both `/bin/bash` and `/bin/sh` on every macOS system; see
[Extending](#extending) for the specific parser gotcha that requires
this).

## Configuration

Search order (first existing wins):

1. `$MY_GIT_CONFIG` env var (escape hatch)
2. `--config <FILE>` on the CLI
3. `/LINKS/default/my-git.conf`
4. `~/.my-git.conf`
5. `/etc/my-git.conf`
6. `/usr/local/etc/my-git.conf`

If none exist, `my-git` prints the search list and exits `2` with guidance
to run `--create-config`. Help and `--create-config` always work without a
config file.

### Variables (all optional; defaults shown)

| Variable                              | Default                                  | Meaning                                                                     |
|---------------------------------------|------------------------------------------|-------------------------------------------------------------------------------|
| `GIT_REPOS`                           | see default config                       | Fallback super-repo list (used only when no repo is scoped via CWD or args) |
| `CHECK_CLAUDE_SETUP`                  | `1`                                      | Audit `.claude/` during `mc`; `0` disables                                  |
| `CLAUDE_PROJECT_DIR_CENTRAL_REPO`     | `/LINKS/src/,claude-configs`             | Central tree where project `.claude/` dirs live                             |
| `COMMIT_MSG_LAST_N`                   | `3`                                      | Number of recent log entries in auto-commit body                            |
| `COMMIT_MSG_DIFF_MAX_FILES`           | `10`                                     | Above this count, commit body shows file list instead of diffs              |
| `PRIV_POLICY_USER_TO_ROOT`            | `warn`                                   | Non-root caller vs. a root-owned repo — see [Privilege](#privilege-sudo--su--policy) |
| `PRIV_POLICY_USER_TO_USER`            | `fail`                                   | Non-root caller vs. another user's repo — see [Privilege](#privilege-sudo--su--policy) |
| `DEFAULT_VERBOSE`                     | `0`                                      | Set `1` to behave as `-V` by default                                        |
| `DEFAULT_DEBUG`                       | `0`                                      | Set `1` to behave as `-D` by default                                        |
| `SM_AUTO_SAFE_DIRECTORY`              | `1`                                      | In `sm`, auto-add `safe.directory` on errors                                |
| `GIT_SIDECAR_DIRNAME`                 | `.git.real`                              | `flatten`/`unflatten`: dirname used for a sidecared submodule/embedded repo |
| `AUTO_SIDECAR_SUBMODULES`             | `0`                                      | `flatten`: `1` = bare `go` auto-sidecars every registered submodule, no prompt |
| `AUTO_SIDECAR_EMBEDDED`               | `0`                                      | `flatten`: `1` = bare `go` auto-sidecars every embedded nested repo, no prompt |

## Quick start

```sh
my-git st                      # recursive OVERVIEW of current tree (CWD-aware)
my-git st /LINKS/global        # recursive overview of a specific super-repo tree
my-git mc                      # analyze: one line per repo, shows what 'mc go' WOULD do
my-git mc go                   # apply: commit+push the CURRENT repo only (single level)
my-git mc go -R                # apply: commit+push THIS repo + every submodule (bottom-up)
my-git pull                    # analyze: fetch + report who's behind (no merge)
my-git pull go -R              # fetch + fast-forward everywhere
my-git fetch -R                # refs-only update across the tree (never merges)
my-git sm                      # analyze nested unregistered repos (THIS level)
my-git sm go                   # register them (THIS level only)
my-git sm -R go                # register everything, walking the whole tree top-down
my-git sm --clean-stale        # ONLY remove stale registrations, anywhere in the tree
my-git sync go -R              # one-shot: sm → pull → mc across the tree
my-git remote --check -R       # audit remotes (⚠ on suspicious URLs)
my-git claude-check            # audit .claude/ symlink convention
my-git flatten                 # analyze: nested repos this would merge as content (THIS repo)
my-git flatten go               # apply: merge them in, embedded repos keep a working .git in place
my-git flatten go -i              # per-item: choose skip/sidecar/merge/delete for each nested repo
my-git flatten go subkid --sidecar  # sidecar just ONE nested repo (path-scoped)
my-git flatten go --sidecar         # bulk: sidecar EVERY nested repo, incl. registered submodules
my-git unflatten                # analyze: every .git.real sidecar found (bulk discovery)
my-git unflatten go subkid --sidecar  # restore just ONE sidecar back to a live .git, exactly
my-git unflatten go embkid --merge    # reconstruct a merged (git-deleted) path from this repo's own history
```

## Subcommands

| Subcommand      | Aliases        | Recursion | What it does                                                                      |
|-----------------|----------------|-----------|-----------------------------------------------------------------------------------|
| `status`        | `st`, `s`      | always    | Compact tree summary; `-V` = per-node porcelain listing; end-of-run total counts  |
| `masscommits`   | `mc`, `c`      | opt-in `-R` | Analyze (default) / `go` = add+commit+push; `-R` = bottom-up walk               |
| `pull`          | `pl`, `fetch`  | opt-in `-R` | Fetch origin + fast-forward clean+behind repos; `fetch` = `pull --fetch-only`   |
| `submodules`    | `sm`, `sub`    | opt-in `-R` | Discover & register nested git repos; `-R` = top-down walk                      |
| `sync`          | `syn`          | opt-in `-R` | Composite: `sm` → `pull` → `mc`. Settles the whole tree in one command.         |
| `remote`        | `rem`          | opt-in `-R` | List / audit git remotes; `--check` flags suspicious urls (file://, http, ./…)  |
| `flatten`       | `fl`           | single-repo (or `-i`/PATH-scoped) | Nested repo → parent content; **3 equal modes** for its git dir: `--merge` (kept live/deleted), `--sidecar` (`.git.real`, history preserved), `--zip` (archived to `.git.zip`/`.git.real.zip`); `-i`/PATH for per-item or forced-mode control |
| `unflatten`     | `unfl`         | single-repo (or PATH-scoped) | Reverse of `flatten`: rebuild a live `.git` — `--sidecar`/`--zip` exact, `--merge` best-effort (`git subtree split`); auto-detects the mode per path |
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

### Auto-commit of my-git-caused changes (both `mc` and `sm`)

my-git **never asks** the user to confirm a change it made itself. Both
`sm go` and `mc go` split dirt into two buckets and commit them
separately:

| Bucket | What counts | Handling |
|--------|-------------|----------|
| my-git-caused | submodule gitlink bumps (super recording a new child SHA); edits to `.gitignore`/`.gitmodules`/`.claude` made by my-git in this run | **auto-committed** via `git commit --only -- <paths>` with subject `my-git <mc\|sm>: auto-commit (bumps=N recorded=M)` — no prompt, even under `-i`. Pushed if the repo is pushable. |
| user | everything else (your own edits, new files, etc.) | normal flow — prompted under `-i`, committed straight through without `-i`. |

`git commit --only` guarantees the two buckets become two separate
commits; user state is never dragged into the my-git-caused commit, and
my-git-caused state is never dragged into the user commit.

### Interactive mode for `mc go`

`-i` on `mc go` makes the **user-bucket phase** pause at each repo,
showing what's about to change and prompting `[Y]es / [S]kip / [A]bort`.
The my-git-caused phase is always silent — it runs before the prompt.

- **Before add+commit+push** (user-dirty repo): prints the porcelain list
  and the first line of the auto-generated commit subject.
- **Before push-only** (clean-ahead repo): prints the `git log` of
  commits about to go upstream.
- `[S]kip` moves on to the next repo; `[A]bort` stops the walk with
  exit code 77 (propagates across `-R`) but the my-git-caused commits
  already made are kept.

Without `-i`, `mc go` runs straight through without any prompts —
commit message is the auto-generated template, and every actionable
repo is processed.

### Ignore marker (respected by both `sm` and `mc`)

A nested repo that carries `.git/my-git-submodules.ignore` is treated
as **leave alone** by both subcommands. `mc` silently skips such repos
(counted as `ignored=N` in the summary). Pass `--force` to override the
markers for a single run — identical semantics to `sm --force`.

### End-of-run summary

Instead of printing a full status tree, `mc` ends with a one-line-plus
summary of what was done / what's left:

```text
 >>> mc: summary
  >> total=23: clean=14 pushed=1 auto-committed=3 user-committed=2 skipped=2 ignored=1
  >> left to do: 2 repo(s) skipped under -i; 1 ignored (--force to override markers)
  >> for details:  my-git st
```

Analyze mode prints an equivalent "would-do" summary. The detailed tree
is intentionally left to `my-git st`.

## Verbosity & debug

Two independent axes. Debug implies verbose.

| Prefix  | Color        | Level  | Shown at              |
|---------|--------------|--------|-----------------------|
| ` >>> ` | bold cyan    | MAJOR  | always (milestones)   |
| `  >> ` | blue         | medium | `-V` `-VV` `-D` `-DD` |
| `    > `| default      | minor  | `-VV` `-DD`           |
| ` ~~~ ` | dim (grey)   | debug  | `-D` `-DD` (stderr)   |
| `  ~~ ` | dim (grey)   | debug  | `-D` `-DD` (stderr)   |
| `   ~ ` | dim (grey)   | deep   | `-DD` only (stderr)   |
| `$ cmd` | dim (grey)   | trace  | `-V` `-VV` `-D` `-DD` (stderr) |

Inline status markers inside headlines: `DONE :))` / `CLEAN :))` in
green, `DIRTY (...)` / `[ABORT ...]` in yellow, `ERROR`/`FAIL` in red.

**Grey command trace (`-V`)**: every subprocess my-git runs (git, sudo,
mv, ln, …) is echoed as `$ <cmd>` in dim grey to stderr **before** it
executes. Mirrors [my-appleRAID](../my-appleRAID/my-appleRAID)'s
`run()` helper. Stderr placement keeps `$()` captures clean — pipe
stdout anywhere, the trace still shows on your terminal.

Colors are **TTY-gated**: on non-TTY stdout (pipes, redirects, capture
buffers) all escape codes collapse to empty strings — logs stay plain
ASCII.

`-DD` also enables `set -x` for deep tracing of my-git itself. Never
design output that only makes sense for a specific `-V + -D` combo —
pick one axis per block.

## `sm go` decision table

Every situation `sm go` can encounter, and what it does in default vs
`-i` mode. Anything not listed here is a bug — either in the code or in
this table. The table reflects **current** behavior, not aspirational
design.

| Situation                                                                    | `sm go` (default)                                 | `sm go -i`                                           |
|------------------------------------------------------------------------------|---------------------------------------------------|--------------------------------------------------------|
| already registered (path + url match)                                        | no-op                                             | no-op                                                |
| nested inside an existing repo boundary (registered **or not**)              | silent skip (counted in summary)                  | silent skip (counted in summary)                     |
| stale registration (`.gitmodules` entry, no dir on disk)                     | auto-remove                                       | prompt `[Y]es / [S]kip / [A]bort`                    |
| carries `my-git-submodules.ignore` marker                                    | silent skip; `--force` treats as fresh            | silent skip; `--force` treats as fresh               |
| marker + `.gitmodules` path mismatch (ignore here, registered elsewhere)     | `CONFLICT` note, skip                             | prompt `[U]pdate / [R]emove / [S]kip / [A]bort`      |
| moved (url matches a different registered path), nested repo **clean**       | rebind in place (no re-clone; all local state kept) | same (no extra prompt)                             |
| moved, nested repo **risky** (uncommitted+untracked, unpushed branches, stashes) | **skip** with "push/commit first" hint         | prompt `[P]roceed / [S]kip / [A]bort`                |
| no origin url on nested repo                                                 | silent skip (counted `missing origin URL`)        | prompt for URL                                       |
| unregistered with url, plain new                                             | `git submodule add`                               | same                                                 |
| unregistered with url, path already tracked in index (embedded)              | `UNTRACK` (`git ls-files -z` piped into `git update-index -z --force-remove --stdin`) + `git submodule add` | same |
| url looks suspicious (unusual scheme)                                        | add anyway (warn only, counted)                   | extra `[U]se / [E]dit / [S]kip / [I]gnore perm / [A]bort` prompt first |
| path matches a `.gitignore` pattern (pre-checked with `git check-ignore`)    | silent skip — `.gitignore` IS the marker (no `add` run) | prompt `[N]egate / [F]orce / [S]kip / [A]bort` — `[N]` appends `!<rel>/` to the matching `.gitignore`, then adds |
| stale `.git/modules/<name>/` from an interrupted prior run                   | **ERROR** + skip                                  | same; `--force` reuses the stale gitdir and adds     |
| `git submodule add` fails for an unclassified reason (unreachable host, repo not found, auth failure, timeout) | skip **this item only**, continue with the rest of the plan | `[R]etry new URL / [S]kip / [I]gnore perm / [A]bort` |

The last row matters more than it looks: a single bad/unreachable remote
**anywhere** in a large tree used to silently abort the entire remaining
`sm go` run, non-interactively, with a message claiming "Aborted by
user" — even though no user was ever involved. Every other reason in
this table already skipped-and-continued; this was the one inconsistent
exception, now fixed to match.

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

### `sm --clean-stale` — stale-registration cleanup only

`sm go` bundles stale-registration removal together with adding new
submodules, rebinding moved ones, and ignore-marker handling — running
it can trigger changes you didn't want yet just to clean up one dead
entry. `sm --clean-stale` does **only** the removal, nothing else:

```sh
my-git sm --clean-stale        # walk the WHOLE tree, clean every stale entry found
my-git sm --clean-stale -i     # prompt [Y]es/[S]kip/[A]bort per entry instead
```

Walks the whole tree **regardless of depth**, same as `--list-ignored`
— a stale entry can live inside an unregistered nested repo's *own*
`.gitmodules` just as easily as the current scope's (e.g. an
unregistered `src` that's itself a full repo with its own submodules).
Each affected repo gets committed separately, in its own history —
`sm --clean-stale` run from the outer repo will correctly reach in and
commit the fix inside `src` too, not just at the top level.

## `flatten` — merge nested repos into the outer repo as content

`git`'s embedded-repo detection stages any nested `.git` as a gitlink
(mode `160000`) no matter what `.gitignore` says — there's no `git add`
flag that treats an embedded repo as plain content. `flatten` is the
dedicated operation that actually merges nested repos in, single-repo
scope. Callable from anywhere inside the repo (auto-scopes to the
toplevel, same as `st`) — it always acts on the whole repo, not just
the subdir you're standing in.

```sh
my-git flatten                       # analyze: print the plan, change nothing
my-git flatten go                    # apply
my-git flatten go -i                 # per-item: choose what happens to each nested repo
my-git flatten go --sidecar          # bulk: force sidecar mode on EVERY nested repo in scope
my-git flatten go --merge            # bulk: force merge mode on EVERY nested repo in scope
my-git flatten go --zip              # bulk: archive EVERY existing sidecar to a .zip
my-git flatten go <path> --sidecar   # scope to just ONE repo/sidecar (also --merge / --zip)
```

`flatten` has **three equal modes**. They share one first step — the
nested repo's files become tracked content in the parent — and differ
only in what happens to its git dir:

- **`--merge`** — the `.git` is DELETED (a `.git.merged` snapshot is
  written first — branch, commit, remote, full config, and any `.git*`
  convention files like `.gitignore`/`.gitattributes`/`.gitmodules`,
  captured then removed, since they'd otherwise silently govern a git
  identity that no longer exists here) — or, under bare `go` for an
  embedded repo, left live in place. my-git does **not** preserve the
  pre-merge history (only what the parent's own commits captured survives).
- **`--sidecar`** — the `.git` is RELOCATED (not deleted) into a
  self-contained `.git.real` capsule that keeps its FULL history, plus a
  `.git.real.status` snapshot so `unflatten --sidecar` restores the exact
  prior state.
- **`--zip`** — the git dir is ARCHIVED into a compact `.git.zip` (plain
  repo) or `.git.real.zip` (an existing sidecar) and removed; the parent
  tracks the `.zip` as content and `st` shows the dir `[zipped]`. Lossless
  (integrity-checked with `unzip -t` before anything is removed; tracked
  worktree dotfiles left in place); `-k`/`--keep` keeps the original git
  dir too. Same operation as the `gz` shell helper, ported into my-git so
  it needs no shell function.

Each reverses with its `unflatten` counterpart. Bare `go` (no forced mode)
decides `--sidecar` vs `--merge` per item for the nested live repos it
finds — never `--zip`, which stays opt-in.

A path that's **already a sidecar** can be converted directly —
`flatten <path> --merge` merges it, `flatten <path> --zip` archives it —
no `unflatten --sidecar` round trip needed first.

**Safety net for nested submodules:** if the path being merged has its
own registered (not-yet-sidecared) sub-submodule, its real git directory
normally lives *inside* the very `.git` about to be deleted — merging
used to silently destroy that sub-submodule's entire history. `flatten`
now detects this automatically and sidecars the sub-submodule first,
recording the transfer in `.git.merged`, before anything is deleted.

`--sidecar` / `--merge` / `--zip` force ONE mode across every item flatten
would otherwise decide per-item:

- `go --sidecar|--merge|--zip` — **bulk**-apply that one mode, no prompting.
  `--sidecar`/`--merge` cover every nested LIVE repo (including registered
  submodules, which bare `go` alone leaves untouched); `--zip` archives
  every existing SIDECAR under the toplevel (a live nested repo is zipped
  only when named explicitly by path).
- `go -i --sidecar|--merge|--zip` — instead of bulk, ask **[Y]es / [s]kip /
  [a]bort per repo, one by one** (works with every mode, including `--zip`).
- `<path> --sidecar|--merge|--zip` — restricts to just that one
  repo/sidecar (embedded, registered submodule, orphan gitlink, or an
  existing sidecar for `--zip`) instead of the whole tree.

`--local-only` (alias `--house`) restricts any mode to repos with **no
remote** — the ones `st` marks with the house icon (⌂). The typical use is
`my-git flatten go --zip --local-only` to archive only the unpublished
sidecars. Same flag works for `--sidecar`/`--merge` too. (`unflatten`'s
`-i` behaves identically, one repo at a time.)

A nested repo carrying `.git/my-git-submodules.ignore` (the same marker
`sm`/`mc` already honor) is left entirely untouched by default — not
listed, not offered under `-i`. `--force` includes it anyway.

If a nested repo is itself a full git repository with its **own**
internal, properly-registered submodules (e.g. an unregistered `src`
that has its own `.gitmodules`), `flatten` correctly recognizes those
children as `src`'s own concern and leaves them alone — they show up
nested under `src` in `st`'s tree, not as if they belonged to this
scope directly. Holds regardless of whether `src` itself is registered
here or not.

**Embedded nested repos** (a plain `.git` dir sitting in a subdirectory,
not registered as a submodule) default to being merged with their `.git`
kept fully intact: briefly moved out of the worktree, `git add`ed +
committed in the outer repo, then moved straight back — `cd nested &&
git log`/`pull`/`push` keep working normally, forever, no renaming.
`--rm-git` instead deletes each nested `.git` permanently (types `yes`
to confirm).

Under `flatten go -i`, per embedded repo you can instead choose
**`sidecar`** (self-contained `GIT_SIDECAR_DIRNAME`, no live `.git` left
— see table below) or `delete`/`skip`. Set `AUTO_SIDECAR_EMBEDDED=1` to
sidecar every embedded repo automatically, no prompt.

**Registered submodules** are left untouched by bare `go`. Under
`flatten go -i`, per submodule:

| Choice | What happens |
|---|---|
| `skip` (default) | leave the submodule registration alone |
| **`sidecar`** | merge files into the parent as content; relocate the git directory into `<path>/GIT_SIDECAR_DIRNAME` (config, default `.git.real`) — self-contained: moves/copies with `<path>` as a unit, addressable via `git --git-dir=<path>/.git.real` (no `--work-tree` needed). Full history preserved. Also writes `<path>/GIT_SIDECAR_DIRNAME.status` — see below. |
| `flatten` (merge mode) | merge files; keep history only as an inert backup under the parent's own `.git/modules/<name>/` — **not** self-contained, lost if `<path>` is ever copied/moved without the parent's `.git` |
| `delete` (merge mode) | merge files; **permanently delete** `.git/modules/<name>/` too — a `.git.merged` snapshot is written first (see below) |

Set `AUTO_SIDECAR_SUBMODULES=1` to make bare `flatten go` auto-sidecar
every registered submodule with no prompt.

A submodule registered but never `init`'ed/cloned offers `skip`
(default) / `deregister` instead — nothing to flatten until it's
actually cloned. `--rm-git` never touches anything already sidecared —
it only affects items still left in the "to merge" pool afterward.

**Old-style submodules** (a real `.git` *directory* sitting directly at
the submodule's own path, rather than the modern lightweight pointer
file to the parent's `.git/modules/<name>/`) are detected and handled
correctly — sidecaring just relocates that already-complete directory in
place, without needing anything from `.git/modules/`. If **both** a
real `.git` at the path *and* a populated `.git/modules/<name>/` exist
simultaneously (genuinely ambiguous — they may have diverged), sidecaring
refuses outright and touches nothing rather than guessing which is
current; resolve by hand (compare their history, remove whichever is
stale) and re-run.

**The `.status` file — a readable, parent-tracked sidecar snapshot.**
Every sidecar also writes `<path>/GIT_SIDECAR_DIRNAME.status` (default
`.git.real.status`) alongside it — an ordinary text file, swept up by
the same commit as the merged content. It records branch, commit,
commit subject, remote URL, whether the path was a registered submodule
(and its name/URL), and a full dump of `git config --local --list` at
sidecar time — enough to glance at the sidecar's state without
`--git-dir` tricks. Two timestamps: `sidecar-since` (when it was
sidecared) and `last-published-to-parent` (bumped by the `ga` shell alias
— see below — every time it force-adds new content under this sidecar;
compare the two to gauge how far the `.git.real` object database's last
commit lags what the parent has actually captured since).

**`GIT_SIDECAR_DIRNAME` is a self-contained relocated repo, not a dead
archive — but the outer repo is the active tracker now.** The `.git.real`
is a fully real, usable git directory (`git --git-dir=<path>/.git.real`
works). What changes is the *workflow*: once sidecared, the *outer* repo
tracks that content as plain files, so ongoing edits show up in the outer
repo's status and are committed there. Nothing commits to the sidecar's
own object database on your behalf, so checking `git status` through it
later just keeps comparing against the commit it was at when sidecared and
drifts from the worktree the outer repo now tracks — not useful for
day-to-day work. Bring it back to the normal live position (its own
`.git`) any time with `my-git unflatten --sidecar <path>` — see
[`unflatten`](#unflatten--rebuild-a-live-git) below.

**Not every nested repo is a dead archive — only sidecared ones are.**
The parent tree and most nested repos (registered submodules not yet
flattened, and embedded repos merged the default way, which keep their
own live `.git` in place) are all still used normally and frequently:
`cd`'ing into them and running plain `git status`/`add`/`commit`/`push`
works exactly as it would in any standalone repo, and their own
`.gitignore` should be respected as-is. **Only** a path carrying
`GIT_SIDECAR_DIRNAME` (i.e. it was sidecared) or `.git.merged` (i.e. it
was merged) is a dead archive — the parent is the sole active tracker
for that content going forward. Any tooling that decides whether to
override a nested `.gitignore` (e.g. the `ga` alias below) should key
off the presence of `GIT_SIDECAR_DIRNAME` specifically, not off "nested
repo" in general — treating every nested repo as archived would
silently defeat intentional `.gitignore` rules in the ones still live.

**`ga` — the shell alias that closes the "old `.gitignore` blocks new
files forever" gap.** Sidecaring relocates a submodule's git dir but
never touches its old `.gitignore` — that file survives as ordinary
tracked content in the parent, and `mc go`'s ongoing commit path
(`commit_one()`) runs plain `git add -A`, not `-f`, so any *new* file
under a sidecared path that its old `.gitignore` would have excluded is
silently skipped forever. `flatten`'s own merge step already force-adds
at merge time, which is why the *existing* files are captured correctly
— it's only files added *after* that point that need help. The `ga()`
function in `etc/zshrc.mine` (next to `gz`/`gu`) closes this: on every
call, it force-adds (bypassing the top-level's own missing `.gitignore`,
same as before) and additionally walks every `GIT_SIDECAR_DIRNAME` found
in the tree, force-adding any file that sidecar's own worktree considers
untracked (deliberately ignoring *that* sidecar's `.gitignore` too — it's
stale, the parent is the authoritative tracker now — while still
respecting its `info/exclude` self-exclusion of `.git.real`/
`.git.real.status` themselves, via explicit `-x` patterns, so the
sidecar's own internals never get swept in as plain content). Each pass
also bumps that sidecar's `.status` file's `last-published-to-parent`.

A path can occasionally be both an embedded repo *and* carry a stale
orphan gitlink at once (leftover from an old half-converted submodule).
Choosing `sidecar` clears that stale gitlink automatically regardless
of how the separate orphan prompt is answered — otherwise the sidecared
content would end up masked/untracked despite `git status` reporting
clean.

`**/GIT_SIDECAR_DIRNAME` is added once to the parent's own **tracked**
`.gitignore` (not the local, untracked `.git/info/exclude`) the first
time anything is sidecared, in its own dedicated commit — a fresh clone
inherits the exclusion automatically. This matters because restoring a
sidecar's full history onto a fresh clone (the supported way to carry
it to a new machine — a plain `git clone` only brings the tracked files
+ `.status` snapshot, not the object database itself) is a
filesystem-level copy of `GIT_SIDECAR_DIRNAME` back into place; without
a tracked exclude, that clone would show it as untracked clutter until
someone re-added the rule by hand.

## `unflatten` — rebuild a live `.git`

The reverse of `flatten`. Same **three equal modes**, same shapes
(analyze/`go`/`-i`/PATH), but the mechanism differs between them:

```sh
my-git unflatten                      # analyze: every sidecar / merged / zipped path found (bulk)
my-git unflatten go                   # apply: restore every discovered one
my-git unflatten go -i                # confirm [Y]es/[S]kip/[A]bort per item
my-git unflatten go <path> --sidecar  # restore just ONE sidecar
my-git unflatten go <path> --merge    # reconstruct just ONE merged (git-deleted) path
my-git unflatten go <path> --zip      # un-archive just ONE zipped path (or bare: auto-detected)
```

- **`--sidecar`** — EXACT, lossless. `<path>/GIT_SIDECAR_DIRNAME` moves
  back to `<path>/.git`; `core.worktree`/`core.bare` (sidecaring's own
  overrides, only ever correct at the relocated position) are unset — a
  plain `<path>/.git` needs no override at all, same as any repo `git`
  creates itself. Nothing was ever deleted by sidecar mode, so this
  loses nothing. The `.status` file is removed (a fresh `flatten
  --sidecar` would just recreate it later; nothing useful survives
  keeping a stale one around). If the path was a registered submodule
  before, that's noted on success — re-register it yourself with `my-git
  sm go` if wanted; `unflatten` doesn't do this automatically.
- **`--merge`** — BEST-EFFORT RECONSTRUCTION, not exact. For a path that
  was merged in with its original `.git` deleted (`flatten --merge`
  choosing `delete`, or `--rm-git`) — there is no git artifact left
  there at all, so the only recoverable history is whatever *this
  repo's own commits* did to that path, extracted via `git subtree
  split --prefix=<path>`. This starts from the day it was first merged
  in, forward — anything before that was permanently discarded when the
  original `.git` was deleted, and is **not** recoverable by this or any
  other means. Bulk-discoverable via its `.git.merged` marker (written
  by `flatten --merge` just before deletion, removed again once
  `unflatten` successfully reconstructs a live `.git`).
- **`--zip`** — EXACT, lossless. Un-archives a git dir that `flatten
  --zip` packed away: `.git.zip` → `.git`, or `GIT_SIDECAR_DIRNAME.zip` →
  the sidecar dir. The `.zip` is removed on success (`-k`/`--keep` keeps
  it). Refuses to overwrite an existing live dir. Bulk-discoverable via
  its `.git*.zip` marker (`st` marks the dir `[zipped]`). Same operation
  as the `gu` shell helper, ported into my-git.
- **Auto-detection**: with a PATH and no mode flag, the mode is inferred
  from what's on disk — a `.git*.zip` means `--zip`; a
  `GIT_SIDECAR_DIRNAME` present means `--sidecar`; otherwise `--merge` is
  attempted.

Bulk (no PATH) discovers **all three** kinds via their own on-disk
marker: sidecars via `GIT_SIDECAR_DIRNAME` (`st` marks `[sidecar]`),
merged paths via `.git.merged` (`[merged]`), zipped paths via `.git*.zip`
(`[zipped]`) — unless restricted to one with a mode flag.

## Not yet implemented

Everything the three-case model called for is built: `flatten`/
`unflatten` with `--sidecar`/`--merge`, the `.status`/`.git.merged`
snapshots, the sidecar-aware `ga`/`gz`/`gu` aliases, and self-deploying
zsh completions (see below). What's left is unrelated to the case
model — older exploration parked in [README.OLD.md](README.OLD.md): a
live content-mirror for case 3 that keeps a *published,
still-developed* repo's `.git` live while the parent *also* tracks its
bytes directly. None of it has a decided, concise answer yet — that's
why it's parked rather than folded in here.

**Zsh completions** deploy themselves on first real use — every
invocation that reaches an actual subcommand (not `--run-tests`,
`--help`, or `--create-config`, none of which need it) writes
`~/.zsh/completions/_my-git` if missing/stale, adds the `fpath` entry to
`~/.zshrc` (inserted before `oh-my-zsh.sh`'s `source` line when present,
otherwise appended with `compinit`), and invalidates `~/.zcompdump*` so
the next shell picks it up. Idempotent — a no-op once already installed.
Mirrors [my-appleRAID](../my-appleRAID/my-appleRAID)'s
`install_zsh_completions()`.

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
  not exist. Run `my-git sm --clean-stale` to remove **only** this —
  `sm go` also adds new unregistered repos and rebinds moved ones,
  which you may not want at the same time. Note the entry may live in
  a *nested* repo's own `.gitmodules` (e.g. shown under an
  `[unregistered]` parent in the tree) — `sm --clean-stale` walks the
  whole tree regardless of depth, so it still finds and fixes it from
  anywhere.
- `[sidecar]` — directory containing a `GIT_SIDECAR_DIRNAME` (a former
  submodule/embedded repo already converted via `flatten --sidecar`).
  Plain tracked content as far as `sm`/`mc` are concerned — purely
  informational, nothing to register or flatten. See the relocated-repo
  note under `flatten` above.
- `[merged]` — directory carrying a `.git.merged` file (a former
  submodule/embedded repo merged via `flatten --merge`, its `.git`
  permanently deleted). Also purely informational; `my-git unflatten
  <path> --merge` can attempt a best-effort reconstruction from this
  repo's own commit history — see [`unflatten`](#unflatten--rebuild-a-live-git).
- `[zipped]` — directory whose git dir was archived via `flatten --zip`
  (a `.git.zip` / `.git.real.zip` beside the worktree, the live dir
  removed). Purely informational; restore it with `my-git unflatten
  <path> --zip`. (A live repo that merely keeps a `-k` backup zip is *not*
  marked `[zipped]`.)

The top-level scope path itself (root of each tree) carries no tag — it is
the super-repo, not a nested entry. DIRTY/CLEAN is orthogonal to
registration: DIRTY just means uncommitted file changes, handled by
`mc`, not `sm`.

`my-git -V st` falls back to the per-node porcelain listing (legacy form).

## Examples

```sh
# Status of a three-level nested tree (global → src → py/my-plex):
my-git st /LINKS/global

# Preview what mc would do for the CURRENT repo only (default — no writes):
my-git mc

# Apply for the CURRENT repo only:
my-git mc go

# Preview what mc -R would do for the whole tree (no writes, bottom-up):
my-git mc -R /LINKS/global

# Apply bottom-up across the whole tree:
my-git mc go -R /LINKS/global

# Register everything (THIS repo only) + show what it did verbosely:
my-git -V sm go

# Register everything across the whole tree, top-down, verbose:
my-git -V sm -R go /LINKS/global

# See every subprocess my-git runs (grey '$ cmd' trace to stderr):
my-git -V mc go

# Deep debug on a single repo, no pager:
my-git -DD st /LINKS/global/src

# Preview what flatten would merge in (no writes):
my-git flatten

# Merge in embedded nested repos, keeping each one's .git usable in place:
my-git flatten go

# Convert every registered submodule to a self-contained .git.real, one at a time:
my-git flatten go -i

# Sidecar every nested repo in bulk, submodules included, no prompting:
my-git flatten go --sidecar

# Sidecar just one nested repo:
my-git flatten go py/my-plex --sidecar

# See every .git.real sidecar currently in the tree (no writes):
my-git unflatten

# Bring one sidecar back to a live, IDE-usable .git, exactly:
my-git unflatten go py/my-plex --sidecar

# Reconstruct a merged (git-deleted) path's history from this repo's own commits:
my-git unflatten go some/old/kid --merge
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
|-----------|---------------|----------------------------------------|
| user      | same user     | direct exec — no escalation           |
| root      | user B        | `su B -c ...` — drops rights, no password |
| root      | root          | direct exec                           |
| user A    | root          | `PRIV_POLICY_USER_TO_ROOT` (default `warn`) |
| user A    | user B        | `PRIV_POLICY_USER_TO_USER` (default `fail`) |

Each policy variable takes one of:

| Value    | Behavior                                                                   |
|----------|------------------------------------------------------------------------------|
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

**Git's own "dubious ownership" guard** (when a process's user doesn't
match a repo's on-disk owner) is handled separately from the policy
table above: `my-git` detects it and auto-registers the `safe.directory`
exception (`git config --global --add ...`), then retries — governed by
`SM_AUTO_SAFE_DIRECTORY` (default on). This applies wherever `my-git`
resolves a repo's toplevel itself (e.g. `flatten`), not just inside `sm`.

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

**Self-test suite (`--run-tests`)**: builds real git repos in a temp
dir and exercises the script end to end — touches nothing outside that
temp dir, safe to run anywhere. Every empirical scenario verified while
developing a change should get turned into a permanent test here, not
just checked once by hand.

```sh
my-git --run-tests              # run everything
my-git --run-tests flatten      # substring filter: only test names containing 'flatten'
my-git --run-tests --name sidecar_status  # same, explicit --name form
```

No separate group-definition table — test names follow
`test_<area>_<specific-behavior>`, so a bare area name (`flatten`,
`unflatten`, `sidecar`, `merge`, `sm`, `st`, …) doubles as a
functional-area filter for free.

**Bash 3.2 gotcha — test any new code against it, not just `dash`/modern
bash.** macOS ships Apple's bash 3.2 (frozen since ~2007, GPLv2, never
updated) as **both** `/bin/bash` and `/bin/sh` — so this script runs
under it on every Mac target regardless of shebang. Bash 3.2 has a real
parser bug: **a `case...esac` statement written inline inside a `while`
or `for` loop fails to parse whenever that whole loop sits inside a
`$(...)` command substitution** — independent of pipes vs heredocs vs
plain loops, and independent of the case pattern's content. It parses
fine under `dash` and under bash ≥4, which is exactly why this can slip
through testing and only surface on an actual Mac. Confirmed by
compiling real bash 3.2 from source and testing directly — `dash -n`
and modern `bash -n` are **not sufficient** to catch this class of bug.

Two safe patterns going forward:
- For a simple prefix/suffix test, use parameter expansion instead of
  `case`: `[ "${var#"$prefix"/}" != "$var" ]` (quoting the pattern
  keeps it literal, not a glob) rather than `case "$var" in "$prefix"/*)`.
- For genuine multi-branch dispatch, extract the `case` into its own
  function and call that function from inside the loop, rather than
  writing the `case` inline inside the loop body.

If you add a loop containing a `case` inside any `$(...)`, test with
real bash 3.2 before shipping — `bash -n` on a modern system will not
catch it. Building it locally is one option:
```sh
wget http://ftp.gnu.org/gnu/bash/bash-3.2.tar.gz
tar xzf bash-3.2.tar.gz && cd bash-3.2
./configure --without-bash-malloc && make
./bash -n /path/to/my-git   # true bash-3.2 syntax check
```
