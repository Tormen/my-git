# CLAUDE.md — working guide for `my-git`

Continuation notes so any Claude session can pick up seamlessly. Read this +
`HANDOVER.md` (open points/status) before making changes.

## What this is

`my-git` is a **single POSIX-sh script** (`src/sh/my-git/my-git`, several thousand
lines) that manages a large tree of nested git repos from one command. It is
deployed by being on `PATH` — **the script IS the source**, so edits are live
immediately; no build/deploy step. Target shell is `/bin/sh` (dash/bash-3.2
compatible — no bashisms, no `local` unless already used, no `mapfile`, macOS BSD
userland: BSD `sed`/`grep`/`stat`, `stat -f`, etc.).

## Golden rules

1. **After ANY edit:** `sh -n my-git` then `./my-git --run-tests` (currently
   **105 tests**). Add/adjust a test for every behavior change — tests live inside
   the script itself and are registered in the big `_rt_all_tests=` string.
2. **my-git commits must contain ONLY my-git's own changes.** Every mutating
   command self-commits, scoped to the paths it touched, and refuses/guards against
   a pre-staged dirty index. Preserve this.
3. **No backward-compat shims** (user preference). Rename cleanly; provide a
   one-time migration script for on-disk state.
4. **Consistency is the prime directive.** Verb pairs are symmetric; all mutating
   commands use the same idiom (below); messages are mode-accurate.
5. **Loud over silent.** Never return an empty result where an error/permission
   problem occurred — warn.
6. **NO AI co-author trailer, ever.** Commits, PRs and tags carry the human
   author only — no `Co-Authored-By: Claude …`, no `🤖 Generated with …`
   footer. The user authors; the assistant is a tool, not a contributor.
   Committing may need `git -c safe.directory='*'` when the repo owner ≠ you.

## The command idiom (shared by the mutating commands)

`flatten`, `unflatten`, `shadow`, `unshadow`, `ignore`, `unignore`, `unsm`, `sm`:
- **analyze by default**, apply with **`go`**, confirm per-item with **`-i`**.
- A bare verb acts **tree-wide** (except `sm` which is single-level + `-R`, and
  `ignore` which needs a path — no bulk "ignore everything").
- `pull`, `push` and `mc` walk **the tree `st` shows** — every live repo at or
  below the scope root, not the submodule tree. Narrow with `--sm`
  (submodules only) or `--sh` (shadowed only). There is no `-R`: operating on
  one repo is what plain `git` already does.
- A single **PATH** scopes to one repo. For `sm`/`unsm` the path is resolved to its
  **immediate enclosing repo** (the repo that directly contains it), not the
  toplevel you run from — so `sm <path> go` and `unsm <path>` act on the same repo.

## Vocabulary: st tags + flags

Tags (first bracket): `[registered-sm]` (git submodule in `.gitmodules`),
`[raw-nested-git]` (bare nested `.git`, not a submodule), `[sidecar]`
(`.git`→`.git.real`, history kept), `[merged]` (`.git` DELETED, `.git.merged`
snapshot only — **no object DB**), `[zipped]` (`.git`→`.git.zip`), `[shadowed]`
(live `.git` KEPT + files also mirrored into parent; marker `.git.shadow.status`),
`[..., ignored]` (carries `.git/my-git.ignore`), `[STALE REGISTRATION]`.

Flag column (after the tag): **`⌂`** = local-only (no remote), **`↑`** = has a
remote (publishable), blank = n/a (zipped/stale). The Summary shows counts:
`⌂ local-only(no remote)=N  ↑ has-remote(publishable)=M  ignored(...)=K`.

## Command map (dispatch near end of file, `case "$_sub" in`)

- `status|st|s` → `cmd_status` (recursive tree overview + Summary).
- `masscommits|mc|c` → `cmd_masscommits` (whole tree, deepest first; `--sm`/`--sh` narrow).
- `push|ps` → `cmd_push` (whole tree; analyze/`go`; never invents an upstream,
  never pushes a diverged branch, skips local-only).
- `submodules|sub|sm` → `cmd_submodules`/`_sm_run_here` (register nested→submodule).
- `unsm|unsub` → `cmd_unsm` (reverse of sm: de-register → raw-nested-git,
  de-absorbing the gitdir; whole-tree default).
- `ignore` / `unignore` → `_ignore_apply` (toggle the generic `.git/my-git.ignore`
  marker; the marker lives in the repo's gitdir → tracked by nothing, no commit).
- `flatten|fl` → `cmd_flatten` (nested→parent content). Modes: `--sidecar`
  (relocate `.git`→`.git.real`, history kept), `--merge` (**DELETE** `.git`,
  history NOT kept, `.git.merged` snapshot for partial subtree-split reconstruct),
  `--zip` (archive `.git`→`.git.zip`, commit ONLY the zip). A bare `go` (no mode)
  is refused.
- `unflatten|unfl` → `cmd_unflatten` (reverse; auto-detects; self-commits teardown).
- `shadow|sh` / `unshadow|unsh` → content-mirror keep-.git-live / reverse.
- `remote|rem`, `pull`, `fetch`, `sync`, `audit`, `repair`, `claude-check`.
- `gz`/`gu` — zip/unzip a live `.git` in place, **parent-neutral** (no commit).

## Key mechanisms / gotchas

- **Ownership:** `run_as_owner_capture <repo> <cmd>` runs git as the owner of
  `<repo>/.git` (via `su`/`sudo -u`). If a tracked file is owned by a different
  user → `Permission denied` → it now **WARNs** (`_mixed_ownership_warn`, deduped
  per repo) instead of silently under-reporting. Keep each repo single-owner.
- **st de-dup:** `walk_tree_lines` unions reg/unreg/stale/markers/shadowed then
  **collapses to top-most** (`is_under` awk) so a nested repo shows once (via
  recursion), never also flat. Don't remove the collapse.
- **Embedded-repo hint:** content-adding `git add -f` (shadow, flatten) use
  `-c advice.addEmbeddedRepo=false` to silence git's stock nested-repo advice.
- **De-absorb (unsm):** `mv <parent>/.git/modules/<name>  <path>/.git`, then
  `git config --file <path>/.git/config --unset core.worktree` (NEVER `git -C`,
  which chdir's into the stale worktree path and fails). A real `.git` dir at the
  worktree is authoritative (`rev-parse --absolute-git-dir` resolves there).
- **Sidecar awareness:** many helpers fall back to `GIT_SIDECAR_DIRNAME`
  (`.git.real`) when there's no live `.git`.
- **Verbosity:** `-V` human, `-VV` command trace (via a dedicated fd, survives `2>`
  redirects), `-D`/`-DD` debug. `run` wraps executed commands for tracing.

## Testing helpers (inside the script)

`_rt_repo`, `_rt_repo_file`, `_rt_conf`, `_rt_my_git`, `_rt_ok`, `_rt_fail`,
`_rt_contains`/`_rt_not_contains` (args: desc, **FILE**, needle — greps a file),
`_rt_exists`/`_rt_absent`/`_rt_isdir`, `_rt_clean`. Tests run under
`MY_GIT_SELFTEST=1` (suppresses the zsh-completion self-install, etc.).

## Environment / paths

- Live tree: `/syst/global` (= `/Users/MINE/system/global` = `/LINKS/global`).
  Source: `/syst/global/src/sh/my-git/my-git`.
- **Sandbox:** the user rsyncs `/syst/global/` → `/Users/us/global/` and
  `chown -R us:` it — a safe, us-owned, writable copy for testing/validation.
  (It may be a *pre-merge* snapshot — e.g. it still holds the full `src/.git`.)
- Ownership policy: `/syst/global` = root, `/syst/global/src` = us.
- Claude memory: `/Users/us/.claude/projects/-Users-MINE-system-global/memory/`
  (`MEMORY.md` index + `feedback_no_backward_compat.md`,
  `feedback_success_fail_pairing.md`). Update it when you learn a durable
  preference/fact.

## Right now (see HANDOVER.md §3 for detail)

- **Top open item:** add the "ARE YOU SURE / this DELETES history" gate to
  `flatten --merge go` (offered, not built).
- Live `src` was `flatten --merge`'d (its `.git` deleted); recover from the sandbox
  copy (`/Users/us/global/src/.git`, 873 commits) or origin.
- `sh/my-git` is **ahead 59, unpushed**.
