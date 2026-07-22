# README.OLD.md — leftover design notes, not yet folded into README.md

This file is a parking lot, not a spec. Everything that was already
decided and concise has been moved into [README.md](README.md) (see its
"Nested repos: three cases" and "Planned / not yet implemented"
sections). What's left here is either:

- still-open exploration that hasn't been distilled into a short answer
  yet (the bulk of this file — Case 3's content-mirror mechanics), or
- unrelated planned features this pass didn't touch (`-V` completeness,
  zsh autocomplete self-install).

Move a section out of here into README.md once it has a decided, concise
answer; delete it from here once it's fully captured there.

## Why `.git.real` is demoted (but NOT dropped)

`realize` relocates a nested repo's git directory to `<path>/.git.real`
so the parent can track the working files as plain content while the
**full history** stays on disk as trackable content too. It fails the
one case it was most wanted for — **a live repo you actively develop
in** — because a `.git.real` is not a `.git`, so **no IDE / editor git
integration recognizes it** (no branch indicator, no inline diff, no
staging UI). For that case v2 routes to plain, standard git instead
(submodule and/or content-mirror, see Case 3).

But realize is **not** dropped, because it is the *only* mechanism that
puts a nested repo's **complete history into the parent as live, tracked,
non-opaque content** — browsable and diffable in place, self-contained
(no origin, no unzip step), just not IDE-live. Nothing else covers that
niche:

| Capture mechanism | Full history in parent? | Form | Live `.git` / IDE? |
|---|---|---|---|
| `zip` (case 1) | yes | **opaque** blob (unzip to use) | no |
| delete (case 2) | **no** — history gone | plain files | no |
| content-mirror (case 3) | no — worktree bytes only | plain files | yes |
| submodule (case 3) | at origin, not in parent | pointer + URL | yes |
| **realize** | **yes** | **plain, tracked, browsable** (`.git.real`) | no |

So realize survives as the "relocate this repo's whole history into a
self-contained `.git.real` alongside the parent-tracked content, for when
I'm not actively developing it here" choice. The `r` option **stays** in
`flatten`.

**Migration is now opt-in, not forced.** Existing `.git.real` dirs are
no longer auto-converted; `unflatten` converts one back to a live `.git`
**when you choose to** (e.g. you want to resume IDE work on it). Leaving
a `.git.real` in place is a valid end state.

## Case 3 — two answers: submodule (identity) and/or content-mirror (bytes)

The question: can the parent track everything inside a **live** nested
repo that the user develops in daily? There are two distinct wants
hiding here, and they have different answers:

- **"Reconstitute the published repo on another machine"** → a
  **submodule** (its identity: URL + exact commit).
- **"Physically hold the kid's file bytes in the parent, as a complete
  backup"** → **content-mirroring** by explicit path (its bytes). This
  is achievable — the earlier "impossible" verdict was too broad and is
  corrected below.

### The embedded-repo boundary — what it actually blocks (verified, git 2.50)

The boundary does **not** mean "the parent can never see inside a live
nested repo." Tested directly; the real rules:

| Operation on live `kid/` | Result |
|---|---|
| `git add -A` / `git add kid/` (directory walk) | **Blocked** — stages `kid` as a gitlink, never its files. The boundary only stops *directory auto-discovery*. |
| `git add kid/.git/anything` | **Refused** — nothing under a literal `.git` is ever trackable. (This part *is* absolute.) |
| `git add -f kid/file`, when nothing under `kid/` is tracked yet (pristine) | **Silent no-op** (rc 0, stages nothing) — you cannot *bootstrap* through a live `.git`. |
| `git add -f kid/file`, once `kid/` already has tracked content | **Works** — modifications *and* brand-new files, staged as ordinary content blobs (no gitlink). |

So two things remain genuinely impossible: `git add -A` will never
descend a live boundary, and `.git/` internals (config/HEAD/refs) are
never trackable — meaning the user's literal "track the status files
via `git add -f`" cannot work *as stated*. **But the underlying goal —
the parent holding the kid's worktree bytes — is reachable**, because
explicit per-file pathspecs *do* cross the boundary once the index is
seeded.

### Mechanism: content-mirror a live kid (verified end-to-end)

Kid's `.git` stays live and IDE-recognized the entire time; the parent
ends up holding real per-file blobs (diffable, delta-compressed — not
opaque archives); no gitlink is ever created. The work splits into three
steps, and — verified below — two of the three are things my-git already
does:

| Step | What it does | Who does it |
|------|--------------|-------------|
| **Bootstrap** (once per kid) | seed the parent index with the kid's current files, `.git` momentarily absent | **`flatten`'s existing move-aside dance** — reuse it, don't rebuild |
| **Ongoing mods + deletions** | keep already-mirrored files current | **free — `mc`'s existing `git add -A`** stages both (tested) |
| **Ongoing new files only** | discover files added since bootstrap | the flag-file step (explicit-path enumeration) |

- **Bootstrap = flatten.** Briefly move `kid/.git` fully *outside* the
  parent tree, `git add -f` the files, move `.git` back. This is the
  *only* step that needs `.git` absent, and it is **exactly what
  `flatten`'s embedded `f` choice already does** (aside → add → restore,
  leaving `.git` live). So the content-mirror's one-time setup is not new
  machinery — it *is* `flatten f`, which seeds every current file under
  `kid/` into the parent index. (A separate `unflatten` is **not** part
  of this; `flatten f` restores `.git` itself. `unflatten` keeps its own
  job — the `.git.real`→`.git` migration. What's shared is the
  underlying move-aside primitive, which the bootstrap should call rather
  than reimplement.)

- **Mods and deletions are already handled — no special step.** Verified
  (git 2.50): once a path under `kid/` is in the parent index, a plain
  `git add -A` — the very command `mc` already runs — stages
  **modifications** *and* **deletions** of those mirrored files, live
  `.git` notwithstanding. The embedded boundary only suppresses
  *discovery of new untracked files*; it does not hide changes to paths
  already tracked. So for a content-mirrored kid, ordinary `mc go` keeps
  the mirror current for everything except newly-created files.

- **New files are the only gap the flag-file must close.** Enumerate and
  force-add by explicit path:

  ```sh
  git -C kid ls-files -z --others --exclude-standard |
    while IFS= read -r -d '' f; do git add -f "kid/$f"; done
  ```

  (`--others --exclude-standard` = new files the kid doesn't itself
  ignore; drop `--exclude-standard` to grab ignored build junk too — a
  knob. Enumerating *all* of `ls-files` instead is harmless, just
  redundant with what `git add -A` already staged.)

**This is where the flag-file finally earns its place — done right.** A
repo carrying `.git.do-track-nested-gits` (or a config setting) tells
`mc` / `ga`, at commit time, to run the new-file enumeration for every
nested kid (after the one-time `flatten f` bootstrap). The user's
instinct was correct on all counts — flatten/unflatten *do* retain their
merit precisely as the bootstrap primitive; only the ongoing mechanism
needed correcting: explicit-path enumeration for new files, **not**
`git add -f` on the directory (which silently fails), while mods and
deletions ride along on `mc`'s normal `git add -A`.

Judged against the two proposals raised:

- **Move `.git` aside for the add** — works (it *is* the bootstrap, and
  `flatten` already does it), but is only strictly needed **once** per
  kid. Doing it on *every* `ga` is heavier than necessary and adds a
  crash window (a mid-op failure leaves a kid with no `.git`). Prefer:
  bootstrap once, then the light explicit-path step.
- **Zip each kid dir on `ga`** — works but is the worst option for a
  *live* kid: the parent stores opaque zip blobs, so no per-file diffs,
  and git cannot delta-compress zips, so every kid change writes a
  near-full new archive and the parent balloons. Zip belongs only to
  genuinely **dead** archives — that is case 1, already covered.

### Two honest caveats on content-mirroring

1. **Worktree content only — never `.git`.** The mirror captures the
   kid's *files*, not its history, branch state, or origin URL (those
   live under `.git`, which is untrackable). So it is a **file backup**,
   not a way to reconstitute the repo. If you also need to rebuild the
   published repo elsewhere, pair it with a submodule (URL via
   `.gitmodules`) or an archived `.git` (case 1). For "the parent is my
   complete file backup of everything," content-mirroring alone is
   exactly right.
2. **A kid is either content-mirrored *or* a registered submodule, not
   both at the same path.** A submodule records `kid` as a gitlink
   (mode 160000); content-mirroring records `kid/*` as a tree of blobs.
   The two representations conflict — pick one per kid based on which
   want (bytes vs. identity) dominates. (They *can* coexist across
   different kids in the same tree.)

### Submodule specifics (the identity answer, still valid)

- **IDE-recognized:** a submodule's `.git` *file* (`gitdir: …`) is
  standard git; every editor and git client resolves it. Unlike
  `.git.real`, nothing needs `--git-dir`.
- **URL + exact state captured:** `.gitmodules` records the origin URL
  (committed, plain text); the parent's gitlink pins the precise SHA.
- **No `reclone` command needed:** `git clone --recurse-submodules`, or
  `git submodule update --init`, reconstructs everything from origin on
  a fresh machine. This is why v2 **drops the previously-planned
  `reclone`.**
- **Uncommitted work covered by `mc -R`:** bottom-up, it commits and
  pushes the submodule *first*, then records the fresh SHA in the
  parent — so `sync go -R` fully settles a case-3 subtree with nothing
  stranded.

## Where the force-adding `ga` wrapper still matters

Retiring `.git.real` removes case 3 as a reason for a force-adding
`git add`. But case 2 leaves a real one: when a nested repo is merged in
by deleting its `.git`, its **`.gitignore` file survives as plain
content** and now applies to the *parent* — silently excluding exactly
the files the parent was supposed to capture. Two ways to handle it, not
mutually exclusive:

- **Wrapper (already built):** `ga` force-adds (`-f`) when the resolved
  top-level carries no `.gitignore` of its own — nothing intentional can
  be defeated. Keep this; it is non-destructive. (Its `.git.real`-aware
  `git()` companion becomes dead once no `.git.real` remain, but is
  harmless.)
- **At merge time:** `flatten`'s `d`/delete could additionally strip or
  neutralize the merged-in nested `.gitignore`, since its rules were for
  a history that no longer exists here. Cleaner, but destructive of a
  tracked file — leave as an opt-in, not default.

## `-V` / `--verbose` — a faithful git-operation trace (unrelated to the nested-repo work above)

Goal: `-V` prints **every external command my-git runs, verbatim with
all arguments as-is**, so the user can audit what happened *to their
repos* (git management), distinct from my-git's own internal debugging
(`-D`, the grey `~~~` lines). The plumbing already exists: `run()` echoes
`$ <cmd>` in grey to stderr at `-V`. The v2 work is (a) guarantee
**100 % of state-changing invocations** (git, zip, unzip, rm, mv, ln,
sudo, su) route through `run()` so the trace is complete and can be
trusted, and (b) keep it verbatim and on stderr (so `$()` captures stay
clean). No new flag — this is `-V` finished properly.

## Self-deploying zsh autocomplete (unrelated to the nested-repo work above)

Mirror `my-appleRAID`'s `install_zsh_completions()` (see
[my-appleRAID](../my-appleRAID/my-appleRAID)):

- Called best-effort at the end of `main`; never fails the run. Runs
  every invocation but is **idempotent** — writes only when content
  changed — so effectively "installs on first run, no-op thereafter."
- Emits `~/.zsh/completions/_my-git` (a `#compdef my-git` zsh function),
  `chmod 644` (compinit refuses mode 600), and busts `~/.zcompdump*`
  when the file changes so the next shell reloads it.
- Patches `~/.zshrc` **once** to add the `fpath` entry (before the
  oh-my-zsh source line if present, else appends `fpath=… ; compinit`).
- **Completions carry inline explanations** — each subcommand and flag
  gets a `:description` / `[description]` string (e.g.
  `'st:recursive status overview of the tree'`,
  `'-R[recurse: bottom-up for mc, top-down for sm]'`,
  `'go:apply — actually perform the analyzed actions'`), so `<TAB>`
  teaches what each option does, not just its name.
- The emitted file is zsh; that is fine even though my-git itself must
  stay POSIX `/bin/sh` (bash 3.2) — my-git only *writes* the file, never
  sources it.
