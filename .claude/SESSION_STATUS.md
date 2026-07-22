# my-git: merge/sidecar rework — session status

Working notes for picking this up after context loss/compaction. Written
mid-session, while `flatten`/`unflatten` were being reworked around the
"merge" vs "sidecar" two-mode model. **This is a status dump, not
documentation** — see `README.md` for the user-facing doc (being kept
concise separately) and `README.OLD.md` for older parked design notes.

## What this session did, in order

1. **Terminology rename**: "realize"/"encapsulate"/"merge-to-parent" →
   settled on **`merge`** and **`sidecar`** as the two flatten modes.
   Config vars renamed: `GIT_REAL_DIRNAME`→`GIT_SIDECAR_DIRNAME`,
   `AUTO_CONVERT_SUBMODULES`→`AUTO_SIDECAR_SUBMODULES`,
   `AUTO_CONVERT_EMBEDDED`→`AUTO_SIDECAR_EMBEDDED`. On-disk dirname
   default (`.git.real`) intentionally UNCHANGED (backward compat with
   already-realized dirs on the user's real tree). All user-facing
   "realize"/"realized" text swept to "sidecar" (internal function
   names like `_fl_realize_submodule` deliberately kept — internal
   plumbing, not user-facing, renaming ~220 occurrences wasn't worth
   the risk).
2. **`flatten --sidecar` / `--merge` flags** on top of the existing
   `flatten [go] [-i]`: force one mode in bulk (`go --sidecar`, ignores
   the old skip-default for registered submodules) or on a single PATH
   (`flatten <path> --sidecar` — NEW capability, flatten previously
   only ever acted on the whole repo). `-i` combined with a forced mode
   is a no-op by design (forcing everywhere leaves nothing to prompt).
3. **`.git.real.status`** snapshot file, written by every `--sidecar`
   realize: branch, commit, remote, was-submodule(+name/url), full
   `git config --local --list`, two timestamps (`sidecar-since`,
   `last-published-to-parent` — the latter meant to be bumped by the
   `ga` shell alias, see below). Does NOT capture `.gitignore` etc. —
   see point 8, capturing something that isn't being removed would
   just create a stale duplicate (this was a real bug, fixed).
4. **`unflatten` command** (brand new): mirrors `flatten`'s shape
   (analyze/`go`/`-i`/PATH/`--sidecar`/`--merge`).
   - `--sidecar`: EXACT restore — moves `GIT_SIDECAR_DIRNAME` back to
     `.git`, unsets `core.worktree`/`core.bare` unconditionally (NOT
     restore-from-capture — the pre-sidecar values only ever made sense
     at the OLD `.git/modules/<name>/` location, meaningless at a plain
     `<path>/.git`; this was a real bug, found via testing, fixed).
     Removes the `.status` file (nothing to gain keeping it — a fresh
     `flatten --sidecar` recreates it).
   - `--merge`: BEST-EFFORT reconstruction via `git subtree split
     --prefix=<path>` from the parent's OWN commit history — only
     recovers history from the merge-day commit forward. Materializes
     via `git init` + `git fetch` into `FETCH_HEAD` + `git reset --hard`
     (NOT fetch straight into `refs/heads/main` — git refuses to fetch
     into the currently-checked-out branch; this was a real bug, fixed).
     Restores `.gitignore` from the `.git.merged` snapshot afterward,
     removes the snapshot.
   - Bulk (no PATH) discovers BOTH sidecars (`list_realized_dirs`) and
     merged paths (`list_merged_dirs`, via the `.git.merged` marker).
5. **`--merge` now actually deletes `.git`** (originally it mapped to
   the "keep .git live" `flatten` choice — a real design bug, since
   that made `--merge` barely different from doing nothing; fixed per
   explicit user feedback: "with a --merge the .git would need to be
   removed at the end").
6. **`.git.merged`** snapshot file (parallel to `.git.real.status`),
   written by `_fl_write_merge_snapshot` just BEFORE the `.git` gets
   deleted: `merged-at` timestamp, branch/commit/remote,
   was-submodule(+name/url), full local git config, captured
   `.git*`-file contents (see point 8), and — critically — a record of
   any sub-submodules transferred (see point 9). Gives `my-git st` a
   `[merged]` tag (`list_merged_dirs`, mirrors `list_realized_dirs`/
   `[sidecar]`) and lets bulk `unflatten` discover merged paths at all
   (previously impossible — no on-disk marker existed before this).
7. **Direct sidecar→merge conversion**: `flatten <path> --merge` on a
   path that's ALREADY a sidecar now converts it straight through — no
   `unflatten --sidecar` + `flatten --merge` round trip needed. Simpler
   than the embedded/registered case (the git-dir is already a normal,
   directly-addressable directory). New function
   `_fl_merge_sidecar_path`, special-cased early in `cmd_flatten` right
   after `cd "$_fl_top"`.
8. **Generic `.git*` capture-and-remove**: `_fl_capture_and_remove_gitfiles
   <path>` captures + removes EVERY `.git*`-prefixed regular file found
   (`.gitignore`, `.gitattributes`, `.gitmodules`, whatever else) —
   deliberately generic, not a hand-picked list, per explicit user
   request ("ALL that is being REMOVED"). Only ever called from
   `_fl_write_merge_snapshot` (merge mode — files govern a git identity
   that's about to stop existing here) — NOT from `_fl_write_sidecar_status`
   (sidecar mode never removes these files, so capturing them would
   just create a stale duplicate; this was a real bug caught by the
   user mid-session and fixed). `.gitmodules` specifically: verified
   empirically that nested `.gitmodules` files are inert to git (only
   ever read at a repo's OWN toplevel, unlike `.gitignore` which is
   hierarchical) — so removing it is safe, NOT the "orphaned index
   entry" risk originally (wrongly) suspected.
9. **CRITICAL FIX — sub-submodule history loss**: found via direct
   testing (not theoretical) that merging a path with its OWN
   registered, not-yet-realized sub-submodule destroyed that
   sub-submodule's ENTIRE git history — its real gitdir lived inside
   `<path>/.git/modules/<name>/`, deleted wholesale along with
   `<path>/.git`. Fixed with `_fl_transfer_sub_submodules` /
   `_fl_transfer_one_sub_submodule`, called automatically from inside
   `_fl_write_merge_snapshot` (so it covers ALL three merge paths:
   embedded delete, registered delete, AND direct sidecar→merge — an
   existing sidecar predating this fix could ALSO be harboring an
   unrealized sub-submodule). Relocates each at-risk sub-submodule to
   its own `GIT_SIDECAR_DIRNAME` sidecar BEFORE the parent's gitdir is
   touched; records what was transferred in `.git.merged`. Verified
   against the exact failure scenario — sub-submodule history now
   survives, `.git.merged` records the transfer.
   **This fix has NOT yet gotten its own regression test — see TODO.**
10. **`-V` trace fix**: `run()`'s trace previously went to plain
    stderr, silently swallowed whenever a caller redirected stderr to
    capture a command's own error output (e.g. `git add`/`git commit`
    inside flatten). Fixed with a dedicated fd 3
    (`exec 3>&2` near the very top of the script, before any
    `main()` cmd parsing), immune to later per-command `2>` redirects.
    This retroactively fixed EVERY already-`run()`-wrapped call site
    with a local stderr redirect, not just flatten's — found and fixed
    live while the user was testing on their real tree.
11. **PATH-scope bug**: a PATH-scoped `flatten <path> --merge` was
    still reporting UNRELATED ignore-marked repos elsewhere in the
    tree in its "marked my-git-submodules.ignore" report. Fixed by
    filtering `_fl_ignored_nested`/`_fl_ignored_registered` too, not
    just `_fl_nested`/`_fl_registered`/`_fl_orphan`. Also added a
    specific error ("is marked my-git-submodules.ignore — pass --force")
    instead of the generic "not a nested repo" when the scoped PATH
    itself is the ignored one.
12. **Sidecar exclude switched from `.git/info/exclude` to a TRACKED
    `.gitignore`** — the local exclude never traveled with a clone (a
    real portability gap the user caught), so a fresh clone (or a
    `.git.real` restored onto one via filesystem-level copy — the
    supported way to carry full sidecar history to a new machine, since
    `git clone` alone only brings tracked files + `.status`, not the
    object database) would show every sidecar as untracked clutter.
    `ensure_sidecar_exclude` now writes `**/GIT_SIDECAR_DIRNAME` to a
    tracked `.gitignore` and commits it, called proactively from
    `flatten` (not from read-only `st`/`mc`).
13. **`--run-tests [PATTERN]` / `--name PATTERN` filtering**: run a
    single test or a functional-area group (substring match against
    test names — `sidecar`, `merge`, `unflatten`, `flatten`, `sm`, `st`
    all work as natural groups given the `test_<area>_...` naming
    convention already in use). No separate group-definition table —
    the naming convention IS the grouping mechanism.
14. **47 regression tests**, all passing (`./my-git --run-tests`).
    Every empirical/scratch verification done this session got turned
    into a permanent test, per explicit user instruction (this is now
    a standing practice — see memory files).
16. **Sub-submodule transfer fix (point 9) now has full test coverage
    and two MORE real bugs it uncovered, also fixed**:
    - `test_flatten_merge_transfers_sub_submodule_history` and
      `test_flatten_merge_direct_from_sidecar_transfers_sub_submodule_too`
      added, both pass.
    - **Found live while writing these tests — a second, INDEPENDENT
      critical bug**: the direct sidecar→merge special case
      (`cmd_flatten`, the `[ -d "$_fl_scope_path/$GIT_SIDECAR_DIRNAME" ]`
      check) used a bare `$GIT_SIDECAR_DIRNAME` with no fallback, unlike
      every other call site in the file. `load_config()` deliberately
      does NOT bake in runtime defaults ("all values come from the
      user's config") — so a config file written before this session's
      `GIT_REAL_DIRNAME`→`GIT_SIDECAR_DIRNAME` rename (a real scenario
      for the user's own existing config) leaves the var EMPTY, which
      makes that check silently degrade to `[ -d "$_fl_scope_path/" ]`
      — true for ANY existing directory. Confirmed by direct testing:
      `flatten go kid --merge` on an ORDINARY live registered submodule
      (not a sidecar at all) took the "already a sidecar, merge
      directly" branch and the directory was destroyed. **Fixed** by
      adding `: "${GIT_SIDECAR_DIRNAME:=.git.real}"` to
      `load_config()`'s existing back-compat-defaults block (same
      pattern already used there for `MC_AUTO_COMMIT_UNTRACKED`/
      `ST_EXCLUDE_DIRS`) — this protects every call site in the file at
      once, not just this one. New test:
      `test_flatten_merge_stale_config_missing_sidecar_dirname_does_not_destroy_live_repo`
      (writes a config missing the var on purpose, confirms the repo
      survives and gets properly merged instead).
    - **Found live while fixing the above — a third bug**: a
      sub-submodule transferred to its own new sidecar (either merge
      path) wasn't covered by the tracked `.gitignore` exclude, because
      `ensure_sidecar_exclude` only ran BEFORE the transfer happened
      (either at `cmd_flatten`'s top, gated on `$_fl_realized` being
      non-empty — which sub-submodule transfers never populate — or not
      at all, in the direct-sidecar-merge special case, which returns
      early before reaching that check). Fixed: the general-path
      re-check is now unconditional (relies on `ensure_sidecar_exclude`'s
      own internal no-op guard instead), and the direct-sidecar-merge
      special case now calls it explicitly after
      `_fl_merge_sidecar_path` succeeds, before staging.
    - **Found live while fixing THAT — a fourth bug**, specific to the
      direct-sidecar→merge path: `_fl_transfer_one_sub_submodule` only
      tried the LITERAL gitfile-pointer path, never the
      `GIT_SIDECAR_DIRNAME`-substituted fallback the rest of the script
      already has a dedicated function for
      (`_fl_resolve_gitfile_pointer`, existing, reused elsewhere e.g.
      for `sm`/orphan-pointer diagnosis). A sub-submodule inside an
      ALREADY-sidecared parent has a pointer broken in exactly the way
      that fallback exists to recover (parent's `.git` got renamed to
      `.git.real`) — the literal-only lookup silently treated this as
      "nothing to transfer" and left the pointer dangling, which is the
      exact history-loss shape this whole safety net exists to prevent.
      Fixed by rewriting `_fl_transfer_one_sub_submodule` to call
      `_fl_resolve_gitfile_pointer` instead of duplicating pointer
      resolution inline.
    All four fixes found via `--run-tests`-driven empirical testing
    (not theoretical), each with its own new regression test. Full
    suite: 47/47 passing.
15. **`gz`/`gu` shell aliases** (in `etc/zshrc.mine`, root-owned,
    needs sudo to edit): extended to recognize `.git.real`(+`.status`)
    as well as plain `.git`, generalized to fold in EVERY `.git*` file
    (not just `.gitignore`) on the finalize-and-remove path (matching
    `_fl_capture_and_remove_gitfiles`'s own genericness). New
    `-k`/`--keep` flag to skip removal (for backup-before-merge use).
    Verified in scratch (finalize, restore, and `-k` paths all correct).
    Patch script handed to the user for sudo application — see TODO.
16. **README.md polish pass**: done. Swept "realize"/"realized" →
    "sidecar"/"sidecared" to match actual script output (`[sidecar]`
    tag, `'r' sidecar` prompt choice), fixed the stale
    `.git/info/exclude` claim (now tracked `.gitignore`), fixed
    `unflatten`'s stale "bulk only discovers sidecars"/"--merge with no
    PATH refused" claims (now discovers both `[sidecar]` and `[merged]`
    markers), added `[merged]` tag docs to Status output, added a
    `--run-tests` usage blurb to Extending, updated Case 1's row for
    `.git.real`/`-k` gz support.
17. **Self-deploying zsh autocomplete** (mirroring `my-appleRAID`'s
    `install_zsh_completions()`): implemented as `install_zsh_completions()`
    near the end of the script, called from inside `main()` — critically,
    AFTER the `--run-tests`/`--help`/`--create-config` early exits, NOT
    unconditionally at the top like `my-appleRAID`'s own placement.
    **Found live while testing**: even with that placement,
    `cmd_run_tests` itself spawns real subcommand invocations (`flatten
    go`, `status`, ...) as subprocesses for each test — those go through
    the normal dispatch path and would ALSO trigger the installer against
    the REAL `$HOME`, breaking `--run-tests`' own "touches nothing of
    yours" promise. Confirmed this actually happened against this
    session's own sandbox `$HOME` before the fix (cleaned up
    afterward). Fixed with a `MY_GIT_SELFTEST=1` env var exported once
    at the top of `cmd_run_tests` (inherited by every subprocess it
    spawns), checked as a guard at the top of `install_zsh_completions`.
    New test: `test_zsh_completions_self_install_on_real_subcommand_not_on_run_tests`
    (explicitly `unset`s the guard for its one "should install" assertion
    to prove the real end-user path still works).

18. **Thorough code-review pass (user-requested "ensure error free")**:
    reviewed every function coded this session. Found + fixed FOUR more
    real bugs, each now with its own regression test, all found via
    empirical testing not inspection alone:
    - **`.git.merged` corruption under `-V`/`-D`**: the sub-submodule
      transfer record was built by `$(...)`-capturing a function that
      ALSO emits a `log_medium` line — and `log_medium` writes to
      stdout, so under verbose the log prose was captured into the
      snapshot's "transferred" section. Fixed by routing the
      machine-readable record through a temp file (separate channel from
      the log). Test: `test_flatten_merge_snapshot_uncorrupted_under_verbose`.
    - **`unflatten --merge` `.gitignore` restore swallowed later capture
      blocks**: `sed /marker/,$ p` ran to EOF, so with the generic
      multi-`.git*` capture a path with both `.gitignore` and
      `.gitmodules` got the `.gitmodules` block appended to the restored
      `.gitignore`. Fixed to stop at the next `--- ... ---` header (awk).
      Test: `test_unflatten_merge_restores_only_the_gitignore_block`.
    - **`unflatten` commit swept unrelated user-staged changes**: blanket
      `git commit` with no pathspec. Fixed to commit ONLY the touched
      paths (`git commit -- <paths>`), matching flatten/sm/mc isolation.
      Test: `test_unflatten_commit_does_not_sweep_unrelated_staged_changes`.
    - **old-style registered submodule `--merge` wrote no `.git.merged`
      marker**: the delete path only handled the absorbed
      `.git/modules/<name>/` gitdir; an old-style real-`.git`-dir
      submodule got no marker (so `st` couldn't tag `[merged]`, bulk
      `unflatten --merge` couldn't find it). Added a `$_p/.git` fallback.
      Test: `test_flatten_merge_old_style_registered_submodule_writes_marker`.
    Also: hardened `_rt_contains`/`_rt_not_contains` test helpers to
    `grep -qF -e` (a needle starting with `-`, e.g. a `--- section ---`
    header, was parsed as a grep option — caused a false failure during
    review); added an empty-parent-gitdir guard to the sub-submodule
    scan (an empty gitdir made the containment glob `/*` match every
    absolute path); made the sub-submodule transfer fall back to
    `/dev/null` if `mktemp` fails (the history-preserving MOVE must not
    be skipped just because recording it failed); turned the transfer's
    silent `|| return 0` failure exits into `log_warn`s.
19. **DBG-quality pass (user-requested)**: added `log_dbg_*` diagnostics
    at every decision/skip/failure point across the merge/sidecar/
    unflatten/transfer code so `-D` narrates the whole flow and pinpoints
    WHERE a failure occurs — e.g. each sub-submodule skip now logs its
    reason, the merge/reconstruct steps log their gitdir/branch/commit/
    counts, mode auto-detection logs its choice, and the scoped commit
    shows its pathspec. Verified the `-D` output tells a clear
    step-by-step story on both `flatten --merge` and `unflatten --merge`.

## What's still TODO (in priority order)

1. **`gz`/`gu`: apply the update to `etc/zshrc.mine`** — needs the user
   to run a sudo command (same pattern as the earlier `ga()` patch:
   write a verified Python patch script to scratchpad, give the user
   the exact `sudo python3 <path>` command). Patch script + command
   already handed to the user in-conversation; NOT yet confirmed run.

All other originally-planned work for this session is complete: **52/52**
regression tests passing, README polished, zsh autocomplete implemented
and safety-verified, and a full error-review pass done (4 more bugs
fixed + DBG diagnostics added throughout). The only outstanding action
is the user running the handed-off `gz`/`gu` sudo command themselves.

## Known limitations / deliberately NOT done

- `.gitmodules` nested inside a merged/sidecared path is captured +
  removed like any other `.git*` file — NOT specially merged into the
  parent's OWN top-level `.gitmodules`. Confirmed this doesn't matter
  functionally (git never reads a non-toplevel `.gitmodules` anyway),
  and re-registering as a proper submodule of the parent later is just
  `my-git sm go` (discovers the now-plain nested repo fresh via its own
  intact `.git`/`.git.real` remote URL).
- `unflatten --merge`'s reconstruction is explicitly NOT a full undo —
  only recovers history from the merge-day commit forward. This is
  inherent to the mechanism (subtree-split from the parent's own
  commits), not a bug.
- No content-mirror-for-a-still-live-published-repo mechanism (parked
  in README.OLD.md) — separate, unrelated feature, not touched this
  session.
- `-V`'s fd-3 fix only covers callers that already route through
  `run()`. A handful of `mv`/`rm` calls elsewhere in the script
  (outside flatten/unflatten, e.g. in `sm`) still bypass `run()`
  entirely and were NOT audited this session — only flatten/unflatten's
  own code got that audit.

## Real-world testing context

The user is actively testing this against their REAL production tree
at `/LINKS/global` (specifically `/LINKS/global/src`, which has many
levels of nesting, third-party clones, and previously-realized
sidecars from before this session's work even started). They have NOT
yet run a real `--merge` on production data at time of writing — was
about to, prompted the sub-submodule-loss investigation that led to
fix #9 above. A manual backup command was given for backing up a
sidecar before merging it:
`zip -qr .git.real.backup.zip .git.real .git.real.status` (run
directly, not through `gz`, since `gz` didn't support `.git.real` yet
at that point in the session).

## Later-session additions (code review + production testing + st polish/perf)

Continued work after the review pass, driven by testing against a
writable COPY of the production tree (`/Users/us/global`, with
`/Users/us/global.BKP` as the reset source; restore via
`chmod -R u+w` then `rm -rf` + `cp -R` because some subdirs are mode
0550). `diff -rq global global.BKP` is the go-to "what changed" check.

20. **CRITICAL — direct sidecar→merge on a sidecar that itself contains
    nested repos.** A sidecar (`src`) can contain embedded/uninitialized
    nested repos (`src/,bewerbungen` embedded; `src/py/my-books` with no
    commit). `git add -A -- <path>` recursing into a repo with no commit
    checked out fails FATALLY ("does not have a commit checked out"),
    staging nothing — and the direct path had already `rm -rf`'d
    `.git.real`, then reported false success. Fixed with self-contained
    `_fl_shield_dir_nested`/`_fl_unshield_dir` (mirrors the main
    pipeline's shielding: move nested repos aside before `git add`,
    restore after) + honest failure handling. Verified on the real
    production copy (`diff -rq` showed ONLY the 4 expected changes:
    `.git.merged` added, `.git.real`/`.gitignore`/`.gitmodules` removed;
    all 16 sub-submodules + both embedded repos byte-identical). Tests:
    `test_flatten_merge_sidecar_containing_embedded_repos`,
    `test_flatten_merge_sidecar_with_embedded_repo_only_no_false_success`.
21. **Scope: bare `my-git` inside a root-owned repo wandered to
    `$GIT_REPOS`.** `resolve_scope` relied on `git rev-parse`, which
    fails on an unreadable (root-owned 0700) `.git`, so it fell back to
    the whole `$GIT_REPOS` list and scanned unrelated trees. Added
    `fs_toplevel_of` (filesystem walk-up for an enclosing `.git`, works
    even when unreadable) as a fallback before `$GIT_REPOS`. Now scopes
    to the enclosing repo and lets privilege policy skip it. Verified on
    real `/LINKS/global`. Test:
    `test_scope_walks_up_to_unreadable_enclosing_repo_not_git_repos`.
22. **Trace tiers (`-V`/`-D`/`-DD`).** `run()` traces meaningful/repo-
    affecting commands at `-V`; new `runq()` traces internal "noise"
    (temp-file mktemp/rm) only at `-D`; `-DD` keeps `set -x`. Wrapped the
    previously-untraced repo mutations (`.gitmodules` removals,
    `git config --unset`, exclude `mv`, snapshot read-commands) in
    `run`, and core-path temp ops in `runq`. So `-V` = clean command
    stream, `-D` = + noise, `-DD` = everything. Verified on a real merge.
23. **`st` local-only `⌂` flag column.** `repo_is_local_only` +
    a fixed-width flag COLUMN rendered AFTER the `[tag]` column (aligned
    vertically): `⌂` = no remote configured (case-2 private), blank =
    has a remote (case-3 published). Plus a `-V` LEGEND explaining the
    state words, dirty codes (M/A/D/R/??), tags, and the `⌂` flag. Test:
    `test_st_local_only_flag_column`.
24. **"frozen" terminology purge.** A sidecar is NEVER "frozen" — it's a
    self-contained RELOCATED live repo; the outer repo is just the active
    tracker now. Removed all "freeze/frozen/frozen-snapshot" language for
    sidecars from code + README (kept only the legit "bash 3.2 is frozen
    software" refs). Memorized as a standing correction.
25. **`st` performance (~40% faster, byte-identical output).**
    (a) `status_summary` now makes ONE `git status -b --porcelain` call
    (status + branch + upstream + ahead/behind) instead of four
    (status + rev-parse + branch-a + rev-list). (b) The three discovery
    `find`s (`list_realized_dirs`, `list_merged_dirs`,
    `list_unregistered_nested`) now PRUNE git-internal dir contents
    (`.git`/`.git.real`/`.git.merged` hold ~37% of a real tree's files
    but never a nested repo we search for) — each find ~3x cheaper,
    verified byte-identical discovery. Net: 4.7s → 2.86s on the 41-repo
    production tree. NOT done (bigger/riskier, left as a future option):
    reducing the *number* of finds (156) by combining the per-node
    passes / precomputing the tree map once.

Test count is now **57**, all passing. Order test (merge parent-first
vs kid-first, and kid-while-src-unmerged vs kid-while-src-merged) both
confirmed to converge to identical results. Merging local-only ("house")
repos confirmed working (empty remote captured, `[merged]` tag shown).

Known cosmetic `st` issue (NOT yet fixed): a `[merged]` repo nested
under a `[sidecar]` can appear twice in the tree (once nested, once as a
flat `src/<path>` from bulk discovery).
