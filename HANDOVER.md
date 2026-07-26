# my-git — Session Handover

_Last updated: 2026-07-26. Covers the work in this session on `my-git`
(the single POSIX-sh script at `src/sh/my-git/my-git`) plus the state of the
live tree at `/syst/global`._

---

## 1. TL;DR status

- **Tests:** `./my-git --run-tests` → **105 passed, 0 failed**. Always run this
  after any edit; also `sh -n my-git`.
- **`sh/my-git` repo:** on `main`, **ahead of `origin/main` by 59 commits**
  (all committed, none pushed). Author shows as `Tormen`.
- **Deployment:** none needed — `my-git` on PATH runs this source file directly.
- **Ownership (as of this session):** `/syst/global` = **root**, `/syst/global/src`
  = **us** (you ran `chown -R us: src`). This matters — see §4.

---

## 2. What changed this session (newest first)

| Commit | What |
|---|---|
| `7121c05` | **sm** now registers into the **immediate enclosing repo** of a path (matches `unsm`) |
| `a266a14` | **st** collapses the whole tree set to top-most → fixes repos listed **twice** |
| `c3c362b` | **flatten** plan message is **mode-aware**; silence git's embedded-repo hint |
| `a0808e4` | **unsm** no longer refuses a live `.git` that also has an orphaned absorbed gitdir |
| `b87fcd0` | `run_as_owner_capture` **warns loudly** on mixed-ownership under-reporting |
| `250c683` | **unsm** defaults to **whole-tree** (matches `st`'s recursive view) |
| `cf00b5d` | **Added `unsm`** — the reverse of `sm` (submodule → `raw-nested-git`) |
| `269dfbb` | **ignore/unignore** adopt the analyze/`go`/`-i`/bulk idiom |
| `f300c3f` | Renamed the ignore marker `my-git-submodules.ignore` → **`my-git.ignore`** |
| `046de1d` | **Added top-level `ignore` / `unignore`**; decoupled the marker from `sm` |
| `d0e0e50` | **`↑` has-remote flag** + flag counts folded into the Summary |
| `a49f8ce` | `unflatten --zip` **self-commits** its teardown; flags legend shown by default |
| `4e2914e` | `unflatten` **self-commits** the `.git.real` teardown (leaves parent clean) |
| `67b5e59` | Renamed tags `[registered]`/`[unregistered]` → **`[registered-sm]`/`[raw-nested-git]`** |

Non-code cleanups done earlier in the session: `/learn` bookmark restore + orphan
`_files` cleanup; venv untracked in `src` (`99ada81`); `.git.real` teardown
committed in the global root (`e0f7f4c0`).

---

## 3. OPEN POINTS (do these next)

### 3a. ⚠️ Add an "ARE YOU SURE" gate to `flatten --merge go` — NOT DONE (offered)
`flatten --merge` is the **only irreversible, history-deleting** mode: it deletes
the live `.git` (keeping only a text `.git.merged` snapshot — **no object DB**),
and `unflatten --merge` can only *partly* rebuild history via `git subtree split`
(forward from the merge; and for `src` it **failed**, `restored=0`). This bit you.
**Plan:** `flatten --merge go` should **refuse by default** and require an
interactive `type "yes"` confirmation (or a `--yes-delete-history` flag for
scripts), and should point out when the repo has a remote (history also on origin).
The current text warnings are not enough. → **just say the word and I'll build it.**

### 3b. Recover `src`'s history on the live tree — commands given, NOT applied
You ran `flatten --merge go src` on the **real** tree, deleting `src/.git`.
**History is safe** in two places:
- Sandbox `/Users/us/global/src/.git` — full **873 commits**, HEAD `99ada81`
  (exactly the merge point).
- Remote `my-gits:/LINKS/gits/src.git` (from the `.git.merged` snapshot).

Recover (src is `us`-owned now, so easy):
```sh
cp -R /Users/us/global/src/.git /syst/global/src/.git
rm -f /syst/global/src/.git.merged
git -C /syst/global/src log --oneline | head   # 873 commits back
```

### 3c. Clean up the mis-placed `nimbiestatemachine` submodule registration
An earlier `sm go` (before commit `7121c05`) registered it into **global** instead
of **my-nimbie**. `unsm` can't reach it (it targets the enclosing repo, my-nimbie).
Remove it from global by hand, then re-register into my-nimbie (now works from
anywhere thanks to `7121c05`):
```sh
cd /syst/global
name=$(git config -f .gitmodules --get-regexp '\.path$' \
       | awk '$2=="src/py/my-nimbie/,archive/nimbiestatemachine"{print $1}' \
       | sed 's/^submodule\.//;s/\.path$//')
git rm --cached "src/py/my-nimbie/,archive/nimbiestatemachine"
git config -f .gitmodules --remove-section "submodule.$name" 2>/dev/null
git config          --remove-section "submodule.$name" 2>/dev/null
[ -s .gitmodules ] || git rm -f .gitmodules
git commit -m "drop mis-placed nimbiestatemachine submodule from global"
# then:
my-git sm src/py/my-nimbie/,archive/nimbiestatemachine go   # → registers into my-nimbie
```

### 3d. The live tree is mid-experiment
`/syst/global` currently has `src` **[merged]**, ~20 repos **[shadowed]**,
`nimbiestatemachine` zipped/re-registered. If you want a clean state, unwind with
`unflatten --merge` (see 3b — prefer restoring the real `.git`), `unshadow`, and
`unsm`. None of this is lost work; it's reversible, but it's a tangle right now.

### 3e. Push when ready
`sh/my-git` is **ahead 59, unpushed**. Push when you're happy: `git -C
/syst/global/src/sh/my-git push` (as the owner).

---

## 4. Key lessons from this session (the "why", so it doesn't bite again)

1. **Mixed ownership silently breaks my-git.** `run_as_owner_capture` runs git as
   the owner of `<repo>/.git`; if a *tracked* file (e.g. `.gitmodules`) is owned by
   a **different** user, that user gets `Permission denied` and my-git returned
   **empty** → it under-reported (showed real `[registered-sm]` repos as
   `[raw-nested-git]`). Fixed to **warn loudly** now (`b87fcd0`). **Keep each repo
   internally single-owner.** Your policy: global=root, src=us.

2. **`flatten --merge` destroys history.** Only `--sidecar` (`.git`→`.git.real`)
   and `--zip` keep it. `--merge` deletes it. See 3a.

3. **Operations act on ONE level = the git toplevel where you run them** (or, for
   `sm`/`unsm` with a path, the path's **immediate enclosing repo** — now
   consistent). `shadow`/`sm` don't cross a nested-repo boundary. To act on a deep
   repo, run from its parent (or pass the path — `sm`/`unsm` retarget for you).

4. **A submodule belongs to its immediate parent**, not a distant ancestor.
   Registering deep gitlinks in a far ancestor creates cross-boundary messes.

5. **`sm`/`unsm` are now symmetric** — both target the immediate enclosing repo, so
   `unsm <path>` always undoes exactly what `sm <path> go` did.

6. **The `ignore` marker is generic**, not sm-specific — every subcommand
   (`st` no-descend, `sm` no-register, `mc` no-commit, `flatten` no-touch) honours
   `<repo>/.git/my-git.ignore`. Manage it with top-level `ignore`/`unignore`.

7. **git's embedded-repo advice** (`adding embedded git repository … hint:`) is
   noise when my-git deliberately manages nesting — silenced via
   `-c advice.addEmbeddedRepo=false` on the content-adding `git add -f`s.

8. **st renders a tree; a nested repo must appear ONCE** (via recursion), never
   also flat at the top. Recursive discovery (`list_shadowed_dirs`, deep
   `list_unregistered_nested` through `[merged]` content) had to be collapsed to
   top-most over the whole set (`a266a14`).

9. **`unsm` + a live `.git` that also has an orphaned `.git/modules/<name>`** is
   NOT ambiguous for unsm (it keeps the live `.git`, only de-registers) — unlike
   `flatten` which relocates and must refuse. Fixed (`a0808e4`).

10. **Everything self-commits cleanly.** flatten/unflatten/gz/shadow and now
    unflatten's teardown all commit **only their own changes** (guard against a
    dirty index). Keep that invariant: *my-git commits must contain ONLY my-git's
    changes.*

---

## 5. Your standing preferences (also in Claude memory)

- **No backward-compatibility shims** — clean breaks + a one-time migration script
  (that's how the `my-git.ignore` rename was done).
- **Consistency everywhere** — same analyze/`go`/`-i` idiom, symmetric verb pairs
  (`sm`/`unsm`, `shadow`/`unshadow`, `flatten`/`unflatten`, `ignore`/`unignore`),
  `SUCCESS:`/`FAIL:` prefixes paired.
- **Loud over silent** — degrade with a visible WARN, never a silent empty result.
- **Big warnings before destructive ops** (the open `flatten --merge` gate).
- **Clean, self-contained commits** with clear messages of what happened.
- **Uppercase = deliberate** for destructive interactive choices (`S`/`M`/`Z`).

---

## 6. How to work on it

```sh
cd /syst/global/src/sh/my-git
sh -n my-git                 # syntax check
./my-git --run-tests         # 105 tests, ~1–2 min
# commit (repo may need safe.directory if owner != you):
git -c safe.directory='*' add my-git && git -c safe.directory='*' commit -F - <<'MSG'
<subject>

<body>

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
MSG
```

See `CLAUDE.md` (next to this file) for the technical continuation guide.
