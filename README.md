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
  it. Walks the whole tree, deepest first, so child
  commits bubble into parent gitlinks in one run. Commit message is
  generated automatically (never opens `$EDITOR`).
- **`sm`** — register / rebind / clean up nested repos as proper
  submodules. **Single level by default**; pass `-R` to walk top-down
  through every registered submodule.
- **`flatten`** — fold nested git repos into the outer repo's history as
  plain tracked content. **Three modes**, one per nested repo, differing only
  in where its history goes — **all three keep it**: **`--merge`** (the whole
  object database is transplanted *into the outer repo*, refs pinned under
  `refs/my-git/merged/<path>/`), **`--sidecar`** (relocated into a
  self-contained `.git.real` capsule — portable, but no longer live/IDE-usable),
  or **`--zip`** (archived into a compact `.git.zip`/`.git.real.zip`).
  A bare `go` with no mode is refused.
  See [`flatten`](#flatten--merge-nested-repos-into-the-outer-repo-as-content) below.
- **`unflatten`** — the reverse: rebuild a live `.git` at a nested path.
  All three modes are exact restores — `--merge` hands back the full
  pre-merge history, its branches, tags **and its remote**, so the restored
  repo can push again. It falls back to a best-effort `git subtree split`
  only for snapshots written before merge preserved history. Auto-detects
  the mode per path. See [`unflatten`](#unflatten--rebuild-a-live-git) below.
- **`purge`** — remove a path's content **and its entire history** from a
  repo; the only command that rewrites history, behind a typed `I am sure`.
- **`shadow` / `unshadow`** — content-mirror: keep a nested repo's **live
  `.git`** (so the IDE works) *and* have the parent track its files too;
  `unshadow` reverses it.
- **`unsm`** — the reverse of `sm`: de-register a submodule back to a plain
  `raw-nested-git` (de-absorbing its gitdir; the repo stays live, history
  intact). `sm <path>` and `unsm <path>` both act on the path's **immediate
  enclosing repo**.
- **`ignore` / `unignore`** — toggle a generic "leave this nested repo alone"
  marker (`.git/my-git.ignore`) honoured by every subcommand.

`st`, `pull`, `push` and `mc` all act on the same thing: **every live
repo at or below the scope root**. Acting on one repo is what plain `git`
already does — my-git exists for the multitude, so the multitude is the
default. Narrow it with `--sm` (the submodule tree only, git's own idea of
nesting) or `--sh` (only the shadowed repos).

`sm` is the exception: it stays single-level with `-R`, because
registration is a statement about one repo's `.gitmodules`.

## The layout this was built for

The tree that drove my-git's design, and the reasoning behind it:

```
/syst/global            root's repo, own server         ← submodule: src
└── src                 'me' repo, own server (ada)     ← self-contained
    ├── icfp            local-only  →  MERGED into src   (history lives in src)
    ├── sh/trash        local-only  →  MERGED into src
    ├── py/my-plex      GitHub      →  SHADOWED into src (own .git stays live)
    ├── c/squashfuse    GitHub      →  SHADOWED into src
    └── …
```

**Two rules decide what happens to every nested repo:**

| the repo… | becomes | why |
|---|---|---|
| has **no remote** (`⌂` local-only) | **merged** into src | nowhere else to live; src becomes its home and keeps its history |
| **has a remote** (`↑` publishable) | **shadowed** into src | src carries the files *and* the repo keeps its own `.git`, so work still pushes upstream |

**Why shadow rather than submodules for the published ones.** The goal is that
cloning `src` gives you *everything, immediately* — no `git submodule update`,
no network round-trip to GitHub/GitLab, no empty directories. A submodule
records only a gitlink, so a fresh clone is a skeleton until every third-party
remote is reachable. Shadowing puts the files in src's own history **and**
keeps each project's live `.git`, so you get both halves:

- all your work is reflected **in src**, and
- each project's work still goes to **its own remote**.

The duplication is the point. It is only waste when both repos live on the
same server — which is exactly why `src` is a plain **submodule** of `global`
(both are on the same box, so a gitlink is enough) while the third-party
projects inside src are **shadowed**.

**Ownership follows the same split:** `global` is root's, `src` is `me`'s.
Keeping them as separate repos means the boundary is enforced by git rather
than by file permissions.

`my-git st` shows which rule each path landed on, and names the other repo
involved with `@<repo>`:

```
└── src                          [raw-nested-git]      ↑
    ├── icfp                     [merged @src]         ⌂  (history preserved in /syst/global/src)
    ├── py/my-plex               [shadowed @src]       ↑  CLEAN, ahead 1
```

## Nested repos: archived / private / published

Every nested `.git` in a tree my-git manages falls into one of these
cases; the right answer for a **published** repo depends on how often you
still develop it here:

| # | Case | Meaning | Handling |
|---|------|---------|----------|
| 1 | **Archived** | Kept only for its history; never developed or pushed to again. | `my-git flatten go <path> --zip` — archive its `.git` (or a `.git.real` sidecar, `+.status`) into a single `.zip` the parent tracks as content (`st` shows it `[zipped]`); `unflatten --zip` restores it, `-k`/`--keep` archives alongside instead of replacing. The `gz`/`gu` shell aliases (`etc/zshrc.mine`) do the same thing interactively without my-git. |
| 2 | **Private** | Developed here, but never published outside this machine (no real origin). | `my-git flatten go <path> --merge` merges it into the parent as plain content and **deletes** its `.git`. No separate history is kept — a repo that's never published shouldn't have had its own identity to begin with. |
| 3a | **Published, seldom changed here** | Pushed to its own origin; you rarely edit it locally. | Toggle `flatten --sidecar` (relocate its `.git` into a self-contained `.git.real` alongside the parent-tracked files) ↔ `unflatten --sidecar` (back to a live `.git` to resume developing/pushing). The parent captures its files either way, so a fresh clone needs no `submodule update`. **Caveat:** while sidecared, its git dir is `.git.real`, which **no IDE/editor recognizes** — fine when you're not actively editing it. |
| 3b | **Published, actively developed here** | Pushed to its origin AND edited here often, so you need live IDE git integration. | `my-git shadow go <path>` (a content-mirror) — keep the **live `.git`** (so the IDE works) *and* have the parent track its files too. It bootstraps once (momentarily sets `.git` aside, `git add -f` its files into the parent, restores the live `.git`) and writes a `.git.shadow.status` marker so `st` tags the dir `[shadowed]`. That first add is the only tricky part (git would otherwise make a gitlink); once a blob is tracked under the dir the live-`.git` boundary is transparent, so ordinary `git add` / `mc` carry new files + mods + deletions straight through. `last-shadowed` is bumped at **commit** time by a small managed `pre-commit` hook `shadow` installs in the parent (only for repos a commit actually touches; never clobbers a foreign hook). `unshadow` reverses it (parent stops tracking, marker removed, hook retired; `.git` stays live). Sidecaring is **unacceptable** here — `.git.real` breaks IDE git. |

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

## Version and builds

`--version` identifies the exact bytes you are running, not just a number:

```
my-git 1.4 (commit 1a2b3c4, build 9f86d081884c)
my-git 1.4 (build 9f86d081884c, unstamped)
```

The **build id** is the first 12 hex of the file's own SHA-256, so two copies
that differ by one byte report different versions — comparing installs is
`--version` on each and a diff. The **commit** is a literal in the file,
written by `my-git stamp-version go`, which rewrites that line and amends the
release commit; a deployed copy therefore needs neither git nor a repository
to say where it came from. Both are printed so a stale stamp shows up as a
disagreement.

`stamp-version` refuses when any other file is dirty (the amend would fold
that work in), is a no-op once the stamp on disk is the one in HEAD, and
renames the rewritten file into place rather than editing it — it is the file
the running process is reading. The stamp lags HEAD by exactly one amend,
because a commit cannot contain its own sha.

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
| `PULL_STRATEGY`                       | `merge`                                  | How `pull go` reconciles a repo that is behind or diverged: `merge` (fast-forwards when it can) or `rebase` |
| `AUTOHEAL_PULL_CONFIG`                | `1`                                      | `st`/`pull` record `PULL_STRATEGY` in the repo owner's **global** gitconfig — only when it states nothing |
| `MC_NEW_NESTED_REPO`                  | `warn`                                   | What `mc go` does with a nested repo whose relationship to its parent is unrecorded: `warn` / `shadow` / `ignore` |

## Quick start

```sh
my-git st                      # recursive OVERVIEW of current tree (CWD-aware)
my-git st /LINKS/global        # recursive overview of a specific super-repo tree
my-git mc                      # analyze: one line per repo, shows what 'mc go' WOULD do
my-git mc go                   # apply: commit+push every live repo in the tree (deepest first)
my-git mc go --sm              # ...only the submodule tree
my-git push                    # analyze: what is ahead, and where it would go
my-git push go --sh            # publish the shadowed projects
my-git pull                    # analyze: fetch + report who's behind (no merge)
my-git pull go                 # fetch + merge everywhere (ff when it can)
my-git fetch                   # refs-only update across the tree (never merges)
my-git sm                      # analyze nested unregistered repos (THIS level)
my-git sm go                   # register them (THIS level only)
my-git sm -R go                # register everything, walking the whole tree top-down
my-git sm --clean-stale        # ONLY remove stale registrations, anywhere in the tree
my-git sync go -R              # one-shot: sm → pull → mc across the tree
my-git remote --check -R       # audit remotes (⚠ on suspicious URLs)
my-git flatten                 # analyze: nested repos this would merge as content (THIS repo)
my-git flatten go --merge       # apply one of the 3 modes (--merge/--sidecar/--zip)
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
| `masscommits`   | `mc`, `c`      | whole tree  | Analyze (default) / `go` = add+commit+push, deepest first; `--sm`/`--sh` narrow |
| `pull`          | `pl`, `fetch`  | whole tree  | Fetch origin + reconcile what is behind or diverged (`PULL_STRATEGY`); `fetch` = `pull --fetch-only` |
| `push`          | `ps`           | whole tree  | Analyze (default) / `go` = publish what is ahead. Never invents an upstream, never pushes a diverged branch, skips local-only |
| `submodules`    | `sm`, `sub`    | opt-in `-R` | Discover & register nested git repos as submodules; `-R` = top-down walk. A PATH registers into the repo that **directly encloses** it (its immediate parent), not the toplevel you run from |
| `unsm`          | `unsub`        | whole-tree | Reverse of `sm`: de-register a submodule back to a `raw-nested-git` — drops `.gitmodules`+gitlink, **de-absorbs** its gitdir (`.git/modules/<name>` → `<path>/.git`); the repo stays live, history intact |
| `ignore` / `unignore` | —        | whole-tree (`unignore`); PATH (`ignore`) | Toggle the **generic** `.git/my-git.ignore` marker — honoured by every subcommand (`st` won't descend, `sm` won't register, `mc` won't commit, `flatten` won't touch) |
| `sync`          | `syn`          | opt-in `-R` | Composite: `sm` → `pull` → `mc`. Settles the whole tree in one command.         |
| `remote`        | `rem`          | opt-in `-R` | List / audit git remotes; `--check` flags suspicious urls (file://, http, ./…)  |
| `flatten`       | `fl`           | single-repo (or `-i`/PATH-scoped) | Nested repo → parent content; **3 equal modes** for its git dir, all history-preserving: `--merge` (history moves into the parent, `refs/my-git/merged/<path>/`), `--sidecar` (`.git.real`), `--zip` (`.git.zip`/`.git.real.zip`); `--rm-git` is the one destructive option and demands typing `I am sure`; `-i`/PATH for per-item or forced-mode control |
| `unflatten`     | `unfl`         | single-repo (or PATH-scoped) | Reverse of `flatten`: rebuild a live `.git` — `--sidecar`/`--zip`/`--merge` all exact (merge restores the preserved history, branches and tags); falls back to best-effort `git subtree split` only for snapshots written before preservation existed; auto-detects the mode per path |
| `shadow`        | `sh`           | single-repo (or `-i`/PATH-scoped) | Content-mirror: keep the nested **live `.git`** AND make the parent track its files too (move-aside bootstrap); analyze / `go` / `-i` |
| `unshadow`      | `unsh`         | single-repo (or `-i`/PATH-scoped) | Reverse of `shadow`: parent stops tracking the files (`git rm --cached` + commit); the live `.git` is left untouched |
| `purge`         | —              | single PATH | Remove a path's content **and its entire history** from a repo — the only command that rewrites history. Resolves the repo by searching enclosing repos' **history** (not their index), `--from` to disambiguate; refuses unless the path is a repo in its own right; backs up `.git`, then demands you type `I am sure` |
| `audit`         |                | always    | Read-only health checks: `--sidecar` (sidecar setup) |
| `rehydrate`     |                | whole tree | Give a shadowed project its `.git` back on a machine that has only the container's files — clones from the remote the marker records, `--no-checkout` so the files on disk win, then puts HEAD on the commit those files actually are. Also repairs a project sitting on the wrong branch, remote or commit |
| `stamp-version` |                | single-repo | Record the current commit in `SCRIPT_COMMIT` and amend it, so `--version` identifies the exact build (see [Version and builds](#version-and-builds)) |
| `repair`        |                | always    | Fix sidecar setup problems (`--sidecar`); analyze by default, `go` to apply       |
| `help`          |                | —         | Show top-level help                                                               |
| *(none)*        |                | always    | `status`, paged through `less` when stdout is a TTY                               |

Run `my-git --help <subcommand>` (or `my-git <sub> --help`) for details.

### Which repo does each command touch?

my-git routinely acts on a repo **other than the one you are standing in** — it
walks up to a toplevel, or down to the repo that directly encloses a PATH. That
used to be implicit; it no longer is. **Every mutating command prints a
`TARGET REPO` banner before doing anything**, and every `-i` prompt names the
repo that will receive the result:

```
 >>> flatten: TARGET REPO  /syst/global
    > this repo's .git receives the commit (the nested files become ITS content)
    > scoped to: src/sh/my-power (only this nested repo is touched)
    > each nested history is transplanted INTO this repo (refs/my-git/merged/<path>/)
```

| Command | Acts on | Commits to | Also touches |
|---|---|---|---|
| `st` | toplevel + every nested repo | **nothing — read-only** | — |
| `mc` | **every repo in scope** | each repo's **own** `.git`, pushes to its **own** remote | — |
| `pull` / `fetch` | every repo in scope | each repo's own `.git` | — |
| `sm` | the repo **directly enclosing** the PATH — *not* the toplevel you ran from, *not* the root repo | that same enclosing repo | nested gitdir is **absorbed** into `<enclosing>/.git/modules/<name>/` |
| `unsm` | same enclosing repo `sm` used | that same enclosing repo | gitdir **de-absorbed** back to `<path>/.git` |
| `flatten` | the toplevel (auto-jumps there from a subdir) | the toplevel's `.git` | each nested `.git`: relocated (`--sidecar`), moved into the toplevel (`--merge`), archived (`--zip`), or **deleted** (`--rm-git`) |
| `unflatten` | the toplevel | the toplevel's `.git` (teardown) | each restored path gets its **own** live `.git` back |
| `shadow` | the toplevel | the toplevel's `.git` (mirrors the files) | nothing — the nested `.git` stays live and authoritative |
| `unshadow` | the toplevel | the toplevel's `.git` (`git rm --cached`) | nothing — no history is affected |
| `ignore` / `unignore` | toplevel used **only** to resolve the path | **nothing is committed** | marker written inside the nested repo's **own** gitdir |
| `purge` | the repo whose **history** contains the path (`--from` to disambiguate) | **rewrites every commit** in that repo | nothing — the path's own repo and files are left alone |
| `gz` / `gu` | the named path | **nothing** — parent-neutral | that path's `.git` ⇄ `.git.zip` |

The rule behind the table: **`sm`/`unsm` resolve *downward* to the immediate
enclosing repo; everything else resolves *upward* to the toplevel.** When those
two differ, the banner tells you which one won.

### Nothing is destroyed except by `--rm-git`

Every mode keeps history somewhere:

| | files → parent | history |
|---|---|---|
| `flatten --sidecar` | yes | → `.git.real` beside the worktree |
| `flatten --merge` | yes | → **into the parent's object DB**, refs pinned under `refs/my-git/merged/<path>/` |
| `flatten --zip` | no | → `.git.zip` in place |
| `flatten --rm-git` | yes | **DESTROYED** |

`--merge` transplants the nested repo's *entire* object database into the parent
and verifies the tip is readable there **before** anything is deleted; if that
fails, nothing is deleted at all. `unflatten --merge` hands it back verbatim —
original branches and tags — leaving post-merge worktree changes as ordinary
uncommitted changes.

Because preserved history lives outside `refs/heads/*`, a default clone/fetch
and `git push --all` would skip it, so my-git adds the fetch **and** push
refspecs to every remote and pushes `refs/my-git/merged/*` explicitly. To pick
it up in a clone made by hand:

```sh
git config --add remote.origin.fetch '+refs/my-git/merged/*:refs/my-git/merged/*'
git fetch origin
```

### `purge` — reclaiming space after content moved out

`unshadow`/`unflatten` answer *"who tracks this path now?"*. `purge` answers
*"who has ever tracked it?"* — and is the only command that rewrites history.

The case it exists for: a nested repo was merged into its parent, then later
given its own `.git` again. The parent still carries that repo's entire
history as dead weight. In this tree, `src` accounted for **93 MB of global's
96 MB** of reachable objects.

```sh
my-git purge src                        # analyze: commits rewritten, MB reclaimed
my-git purge go src --from /syst/global # apply, behind the "I am sure" gate
```

It resolves the target repo by searching enclosing repos' **history**, not their
index — after an `unshadow` no repo *tracks* the path any more, yet its objects
still sit in the repo above, which is exactly when purge is wanted. If two repos
qualify it refuses and makes you pick with `--from`.

Safety: it refuses unless the path is a repo in its own right (so it relocates
nothing and destroys nothing; `--force` overrides), copies `.git` to
`.git.purge-backup-<timestamp>` first, and requires you to type `I am sure`.

The rewrite runs on a **bare copy** of the gitdir, never the live repo — history
is a property of the gitdir, so the worktree is never involved. That avoids
filter-branch's clean-worktree requirement *and* its final checkout, which would
otherwise delete the very files being purged. On failure nothing is changed at
all. Afterwards the index is rebuilt with `reset --mixed`, leaving the path
untracked-but-present.

Like `--rm-git`, `purge` has **no inverse**. Every reversible operation in my-git
comes as a pair (`shadow`/`unshadow`, `flatten`/`unflatten`, `sm`/`unsm`); the
destructive ones stand alone. That asymmetry is the signal.

`--rm-git` is the single destructive operation on a *nested repo's* history. It lists exactly what it will
delete and requires you to type **`I am sure`** — nothing else proceeds. There
is deliberately no "merge but discard the history" flag; typing one points you
here.

### `rehydrate` — a clone of the container is not yet a set of repos

Cloning the container gives you every project's **files**, because the
container tracks them. It does not give you their `.git` directories: those
never travel, so a fresh machine has the content and no history, and no way
to push a project back to its own remote.

`rehydrate` closes that gap, using the `.git.shadow.status` marker each
shadowed project carries:

```sh
git clone <container-url> src && cd src
my-git rehydrate          # what would be restored
my-git rehydrate go       # do it
```

For each project it clones from the remote the marker records, with
`--no-checkout` — **the files already on disk win**; only the history is
attached underneath them. Then it puts HEAD on the commit those files
actually are, so the restored repo reads CLEAN under plain `git` instead of
showing an entire upstream's worth of phantom changes.

It repairs, not just restores. A project that already has a `.git` but sits
on the wrong branch or points at the wrong remote is put back on what the
marker records; one at the wrong commit is reset onto the commit its files
are. No working file is written in either case. A project whose files match
no commit is named and left alone — that is local work, or a state no remote
has.

A project whose marker records no remote was local-only and cannot be
cloned; if it was merged rather than shadowed, its history is inside the
container (see [`unflatten --merge`](#unflatten--rebuild-a-live-git)).

Day to day you rarely need it: `pull go` realigns a lagging HEAD on its own,
and so does `st`. `rehydrate` is for the case only it can fix — a project
with no `.git` at all.

### Why recursion is opt-in for `mc` and `sm`

Git's own model is: each super-repo is responsible only for its own index
and its own submodule gitlinks (one commit SHA per registered submodule).
What lives inside a submodule — further nested repos, sub-submodules,
dirty files — is that submodule's concern, not the super's.

`sm` follows that model: single level, with `-R` for a top-down walk so
parent rebinds settle before children see them.

`mc`, `pull` and `push` do not, because the question they answer is not
"what does this index record" but "what is the state of everything I work
on". `mc` runs deepest-first, so a parent's gitlink lands on a child
commit made in the same run.

`st` is always recursive because overview IS its job — a multi-level
tree seen at a glance, no walking required.

### Auto-commit of my-git-caused changes (both `mc` and `sm`)

my-git **never asks** the user to confirm a change it made itself. Both
`sm go` and `mc go` split dirt into two buckets and commit them
separately:

| Bucket | What counts | Handling |
|--------|-------------|----------|
| my-git-caused | submodule gitlink bumps (super recording a new child SHA); edits to `.gitignore`/`.gitmodules` made by my-git in this run, and paths force-added for a child (see `.git.force-add-patterns`) | **auto-committed** via `git commit --only -- <paths>` with subject `my-git <mc\|sm>: auto-commit (bumps=N recorded=M)` — no prompt, even under `-i`. Pushed if the repo is pushable. |
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

### Ignore marker (generic — honoured by every subcommand)

A nested repo that carries `.git/my-git.ignore` is treated as **leave this
repo alone** across all of my-git — it's **not** an `sm`-only thing:

- `st` does **not descend** into its subtree,
- `sm` does **not register** it as a submodule,
- `mc` silently **skips** it (counted as `ignored=N` in the summary),
- `flatten` does **not touch** it.

Manage it with the top-level verbs — no hand-editing `.git/`:

```sh
my-git ignore   <path>       # analyze; add 'go' to write the marker
my-git unignore              # whole-tree: remove markers everywhere (analyze/go)
my-git sm --list-ignored     # audit which repos carry it
```

Pass `--force` to `sm`/`mc`/`flatten` to override the markers for a single run.
(The on-disk file was historically `my-git-submodules.ignore`; it was renamed to
`my-git.ignore` — a clean break, no dual-name lookup.)

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

### `pull` — how it reconciles

`pull go` fetches, then reconciles every repo that is **behind** or
**diverged** (ahead *and* behind), using `PULL_STRATEGY`:

- **`merge`** (default) — `git merge origin/<branch>`. Fast-forwards
  whenever your branch has no commits of its own; otherwise a real merge
  commit.
- **`rebase`** — replays your local commits on top of origin's.

There is no fast-forward-only mode. Refusing to reconcile a diverged branch
is not a strategy — it is a tree that silently never pulls.

A dirty worktree is never a reason to skip: git itself refuses to overwrite
uncommitted work, per path, and it is the one that knows which paths the
incoming diff touches.

When the merge cannot finish, `pull` stops on **that** repo, names the paths
that need you, and carries on with the rest of the tree:

| | what happened | what you do |
|---|---|---|
| `CONFLICT` | the merge started and is **left in the worktree** — never aborted | resolve + `git commit` (or `git rebase --continue`), then re-run |
| `BLOCKED` | the merge never started: uncommitted or untracked files would have been overwritten. Nothing was touched | commit them (`my-git mc go`) or move them, then re-run |

Both are counted separately from `errors=`, list the first three paths
(`-V` for all), and are repeated per repo in a `needs you` block at the end
— on a tree of forty repos the per-repo line is long gone by then.

```text
WARN:  ,github/my-booking-tool: behind=3 — merge BLOCKED, 75 path(s) in the way
    > modified:  .gitignore, README.md, app/atomic_io.py … (+57 more)
    > untracked: app/backup.py, app/macros.py … (+13 more)
    > commit them (my-git mc go) or move them, then: my-git pull go

 >>> pull: needs you (2)
  >> ,github/my-booking-tool     BLOCKED   75 path(s): commit or move, then re-run
  >> py/my-boox                  CONFLICT   2 path(s): resolve + git commit
```

With `AUTOHEAL_PULL_CONFIG=1` (the default), `st` and `pull` also record
`PULL_STRATEGY` in the **repo owner's global** gitconfig, once per owner per
run, so a `git pull` you run by hand reconciles the same way instead of
dying with *"Need to specify how to reconcile divergent branches"*. Written
only when that config states nothing; one that **conflicts** is warned about
and left exactly as it is.

### New nested repos in `mc go`

A nested git repo whose relationship to its parent is **not recorded** (not a
registered submodule, not shadowed, not sidecar/merged/zipped, no ignore
marker) is never swept into a commit — `git add -A` would record it as a
**gitlink** pinning a commit nobody can fetch, and the damage is silent: the
path becomes tracked, so `git status` reads clean while a clone gets an empty
directory.

`MC_NEW_NESTED_REPO` says what happens instead:

| value | what `mc go` does |
|---|---|
| `warn` (default) | warn, leave it untracked; classify it yourself, or with `mc -i` |
| `shadow` | mirror its files into the parent, keep its `.git` live |
| `ignore` | run `my-git ignore` on it, so it is never carried |

`warn` is what a my-git that had never heard of the knob did, so upgrading
changes nothing until you opt in. Under `-i`, `warn` **asks** per repo
(shadow / submodule / ignore / leave) instead of only warning, and an auto
value confirms each repo first.

`flatten` and `sm` are deliberately not values: `flatten` deletes the child's
`.git` (far too destructive as a side effect of `mc`), and `sm` needs a URL,
which a repo with no origin has nobody to ask for in a non-interactive run —
it *is* in the `-i` menu, where a human exists.

Analyze mode classifies nothing, ever; it reports what `go` would do.

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
| carries `my-git.ignore` marker                                    | silent skip; `--force` treats as fresh            | silent skip; `--force` treats as fresh               |
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

- bypasses per-repo `my-git.ignore` markers (treat as fresh).
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

- **`--merge`** — the nested history is **TRANSPLANTED into the parent's
  object database** under `refs/my-git/merged/<path>/`, and only then is the
  nested `.git` removed. A `.git.merged` *text* snapshot is written alongside
  it (branch, commit, remote, full config, and any `.git*` convention files
  like `.gitignore`/`.gitattributes`/`.gitmodules`, captured then removed) so
  `unflatten --merge` knows what to restore. **The deletion is gated on the
  transplant**: `preserve_merged_history` verifies the tip is readable from
  the parent afterwards, and every caller leaves the nested `.git` alone if it
  is not — so merge never destroys anything. One caveat worth knowing: those
  refs live outside `refs/heads` and `refs/tags`, so a **default clone does
  not fetch them**; `--merge` adds the refspecs to the parent's remote so the
  parent carries them.

  The one deliberately destructive operation is **`flatten --rm-git`**, which
  deletes nested `.git` dirs outright behind a typed `I am sure`.
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

Each reverses with its `unflatten` counterpart. A bare `go` with **no mode is
refused** — flatten always disposes of each `.git`, so you must state how
(`--sidecar` / `--merge` / `--zip`, or `-i` to choose per repo). `--zip` also
stays out of the `-i` "sidecar vs merge" default; it's opt-in only.

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

`--local-only` (alias `--house`) is a **generic filter over everything
flatten does** — counts, the by-mode overview, the plan, and every mode's
apply: keep only repos with **no remote** (the ones `st` marks with the
house icon ⌂). The typical use is `my-git flatten go --zip --local-only` to
archive only the unpublished sidecars, but it applies to
`--sidecar`/`--merge` and bare `flatten` just the same.

A nested repo carrying `.git/my-git.ignore` (the same marker
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

## Completeness

The whole case model is now built: `flatten`/`unflatten` with
`--sidecar`/`--merge`/`--zip`, the `.status`/`.git.merged` snapshots, the
sidecar-aware `ga`/`gz`/`gu` aliases, and case 3b's `shadow`/`unshadow`
content-mirror (keeping an actively-developed *published* repo's `.git`
**live** so the IDE works while the parent *also* tracks its bytes —
bootstrapped by `shadow go`, then maintained by `mc`'s `git add -A` plus
the `ga` alias for new files). Historical design notes are parked in
[README.OLD.md](README.OLD.md).

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
/LINKS/global                          ↑  CLEAN :))
├── src [registered-sm]                ↑  DIRTY (18)  [M:1 ??:17]
│   ├── py/my-plex [registered-sm]     ↑  CLEAN :))
│   └── sh/my-git [raw-nested-git]     ⌂  DIRTY (2)  [M:2]
└── etc [registered-sm]                ↑  CLEAN, ahead 1
```

The column between the `[tag]` and the state is the **flag**: `⌂` = local-only
(no remote configured — nothing to publish to), `↑` = has a remote (publishable),
blank = n/a (a `[zipped]`/stale entry). The end-of-run **Summary** tallies them,
e.g. `⌂ local-only(no remote)=15  ↑ has-remote(publishable)=44  ignored(...)=9`.

States: `CLEAN :))`, `CLEAN, ahead N`, `CLEAN, behind N`,
`DIVERGED (ahead X, behind Y)`, `CLEAN (no remote for B)`,
`DIRTY (N) [M:.. A:.. D:.. R:.. ??:..]`, `[SKIPPED — cross-user policy]`,
`[ERROR …]`.

The tree includes **both** submodule-registered and raw (unregistered)
nested git repos found on disk. The first word of the mark says what the
nested git **is** relative to its parent — `registered-sm` (a registered
submodule) vs `raw-nested-git` (a bare nested `.git`, not a submodule) —
whereas `sidecar`/`merged`/`zipped`/`shadowed` are *transformed* states of
such a repo, so the label is not a simple registered/unregistered binary:

- `[registered-sm]` — nested git repo registered as a submodule in
  `.gitmodules`; `sm` has nothing to do for this entry.
- `[raw-nested-git]` — nested git repo not registered as a submodule (not
  yet in `.gitmodules`). Run `my-git sm go` to register it as one.
- `[registered-sm, ignored]` — registered submodule **also** carrying the
  `my-git.ignore` marker. Conflict state; inspect with
  `my-git sm --list-ignored`.
- `[raw-nested-git, ignored]` — raw nested repo carrying the ignore
  marker; `sm go` skips it and `st` does not descend into its subtree.
- `[STALE REGISTRATION]` — entry in `.gitmodules` whose on-disk path does
  not exist. Run `my-git sm --clean-stale` to remove **only** this —
  `sm go` also adds new raw nested repos and rebinds moved ones,
  which you may not want at the same time. Note the entry may live in
  a *nested* repo's own `.gitmodules` (e.g. shown under a
  `[raw-nested-git]` parent in the tree) — `sm --clean-stale` walks the
  whole tree regardless of depth, so it still finds and fixes it from
  anywhere.
- `[sidecar]` — directory containing a `GIT_SIDECAR_DIRNAME` (a former
  submodule/embedded repo already converted via `flatten --sidecar`).
  Plain tracked content as far as `sm`/`mc` are concerned — purely
  informational, nothing to register or flatten. See the relocated-repo
  note under `flatten` above.
- `[merged @R]` — directory carrying a `.git.merged` file (a former
  submodule/embedded repo absorbed via `flatten --merge`). Its history is
  **not** gone: the whole object database was transplanted into repo `R`,
  with every ref pinned under `refs/my-git/merged/<path>/`. `my-git
  unflatten <path> --merge` hands it back in full — branches, tags and its
  remote. A snapshot written *before* merge preserved history says so
  explicitly (`NO preserved history`), and can only be reconstructed
  best-effort via `git subtree split` — see
  [`unflatten`](#unflatten--rebuild-a-live-git).
- `[zipped]` — directory whose git dir was archived via `flatten --zip`
  (a `.git.zip` / `.git.real.zip` beside the worktree, the live dir
  removed). Purely informational; restore it with `my-git unflatten
  <path> --zip`. (A live repo that merely keeps a `-k` backup zip is *not*
  marked `[zipped]`.)
- `[shadowed @R]` — content-mirror (case 3b): the nested repo keeps its
  **live** `.git` (IDE git works) *and* repo `R` also tracks its files.
  Carries a `.git.shadow.status` marker recording `shadowed-into-repo: R`
  plus `first-shadowed` / `last-shadowed` timestamps and HEAD/remote. The
  line still shows the repo's own live CLEAN/DIRTY state. Set up with
  `shadow`, undo with `unshadow`. Running `my-git` from *inside* a
  shadowed repo says which repo mirrors it, and how to see the full tree.
- `[shadowed STALE: @R no longer tracks it]` — the marker is there but no
  repo mirrors the files any more (e.g. `R` stopped tracking that path).
  `unshadow` cleans such a marker rather than failing on it.

The top-level scope path itself (root of each tree) carries no tag — it is
the super-repo, not a nested entry. DIRTY/CLEAN is orthogonal to
registration: DIRTY just means uncommitted file changes, handled by
`mc`, not `sm`.

`my-git -V st` falls back to the per-node porcelain listing (legacy form).

## Git concepts & caveats my-git works around

These are the git behaviours that shape my-git's design — worth knowing before
you touch the internals:

- **Embedded repos become gitlinks.** `git add` on a path that contains a nested
  `.git` records a **gitlink** (mode `160000`) — a single commit SHA, *not* the
  files — and prints its `adding embedded git repository … hint:` advice. There is
  no `git add` flag to stage a nested repo's *files* as plain content. my-git's
  content-adding steps pass `-c advice.addEmbeddedRepo=false` to silence the hint
  (the nesting is deliberate), and `flatten` is the operation that actually
  disposes of the nested `.git` so its files *can* be tracked.

- **Submodules belong to the repo that DIRECTLY encloses them.** A gitlink lives
  in the index of the nearest enclosing repo, and its history is normally
  *absorbed* into that parent's `.git/modules/<name>/`, with the worktree `.git`
  reduced to a `gitdir: …` **pointer file**. `sm`/`unsm` therefore both operate on
  a path's immediate enclosing repo (not a distant ancestor). `unsm` **de-absorbs**
  — moves `.git/modules/<name>` back to `<path>/.git` and clears the stale
  `core.worktree` — turning it back into a standalone repo.

- **A real `.git` DIR at the worktree wins.** If `<path>/.git` is a directory (not
  a pointer file), git uses it — `git rev-parse --absolute-git-dir` resolves there,
  regardless of any leftover `.git/modules/<name>` copy. So `unsm` needn't fear an
  "ambiguous" both-exist state (unlike `flatten`, which *relocates* a gitdir).

- **`git commit -- <path>` uses `--only` semantics** — it commits the *worktree*
  state of that path. For a path that is now a **live nested repo**, that would
  **re-add it as a gitlink**, undoing a `git rm --cached`. So teardown steps
  (`unshadow`, `unsm`) commit the **index as-is** (no pathspec), after guarding
  that no unrelated changes were pre-staged.

- **`flatten --merge` deletes history.** The `.git.merged` file is a *text*
  snapshot, not an object database. Only `--sidecar`/`--zip` are lossless.

- **Editing a broken repo's config:** use `git config --file <gitdir>/config …`,
  **never** `git -C <path> config …` — the latter tries to `chdir` into a stale
  `core.worktree` first and fails (`cannot chdir to '../../…'`).

- **Ownership / dubious-ownership.** git run by user A on a repo owned by B refuses
  (`detected dubious ownership`) or can't read tracked files owned by C
  (`Permission denied`). my-git runs each repo's git **as that repo's `.git`
  owner** (`run_as_owner_capture`); if a *tracked* file has a different owner the
  read silently returned empty → it **under-reported** (real `[registered-sm]`
  repos shown as `[raw-nested-git]`). That now emits a loud
  `WARN: mixed ownership …`. **Keep each repo internally single-owner.**

- **`safe.directory`.** Reading a differently-owned repo one-off:
  `git -c safe.directory='*' -C <repo> …`.

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

# NOTE: a bare 'flatten go' is refused — you must pick a mode below, because
# flatten always disposes of each nested .git.

# Choose sidecar / merge / zip per nested repo, one at a time:
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

# De-register a submodule back to a raw-nested-git (keeps it LIVE, history intact):
my-git unsm go src/py/my-nimbie/,archive/nimbiestatemachine

# Leave a nested repo alone — st won't descend, sm won't register, mc/flatten skip it:
my-git ignore go some/vendored/repo

# Un-ignore every marked repo in the tree:
my-git unignore go
```

## Scope resolution (st / mc / audit)

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

## Carrying local-only files between machines

A project often has files it must NOT publish but DOES want on every machine
you work from: editor and tooling config, personal notes, granted
permissions. They are two different questions, and they get two different
answers.

**The project decides what it publishes.** Two generic patterns in its
tracked `.gitignore`:

```gitignore
*.local        # a local-only DIRECTORY (memory.local/) or file (notes.local)
*.local.*      # settings.local.json, notes.local.md, anything.local.yaml
```

Nothing names a tool, so nothing about your setup is visible in the
published repo — not even the ignore rule. And because a directory whose
contents are *all* ignored is itself invisible to git, a directory holding
only `*.local.*` files never appears in `git status` and needs no rule of
its own.

Deliberately two patterns, not one `*.local*`: the single-glob form also
swallows `config.locale` and `app.localhost.conf`.

**The container decides what it carries.** A container repo (the outer repo
that mirrors your project repos — `/LINKS/src` here) HONOURS every child's
`.gitignore`, because a file the child ignores is regenerable by definition:
`.venv`, `node_modules`, `build/`. Mirroring those once put two complete
virtualenvs — 14,327 files — into the container in a single command.

The exception is named explicitly, in `.git.force-add-patterns` **at the
container root**:

```
# What this container CARRIES despite a child repo ignoring it.
*.local
*.local.*
```

`mc` and `shadow` force-add exactly the ignored paths matching those globs,
one file at a time. Never `git add -f -- <directory>`, which would override
every `.gitignore` in the tree and sweep in the junk sitting beside them.

The file lives at the container root **only**. A copy inside a child would
be tracked by that child and pushed, publishing the very list that says what
stays unpublished.

Its name is a constant, not a config setting — like `.git.shadow.status`,
`.git.merged` and `.git.zip`. It is a name two machines have to agree on, and
as a setting it would be one more thing that must land before the tree works:
a machine that pulled the repos but not the config would carry nothing.

The pass runs in `mc` as well as in `shadow`. Shadow-time alone is not
enough: a file written afterwards is ignored by the container, so a plain
`git add -A` never sees it, and it would sit there untravelled until someone
happened to re-run `shadow go`.

Under `mc` without `go` it reports (`WOULD carry N file(s)`) and stages
nothing — analyze mode leaves the index exactly as it found it.

**The failure modes, and why this one was chosen.** Force-adding everything
a child ignores gives you "junk or secrets arrived", which needs a history
rewrite — and it would sweep in `.env` and `*.pem`, ignored precisely
because they are secrets. Honouring every child `.gitignore` with no
exception gives you "a file I wanted did not travel", which is visible and
fixed by adding one pattern. The second is the better failure.

An src-side *exclusion* list was considered and rejected: it would have to
enumerate the junk of every ecosystem, and missing one absorbs it silently —
whereas each child's `.gitignore` already lists exactly that, maintained by
whoever knows the project.

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
