# The composition model

What a composed tree contains, who may write it, and why each rule is
the way it is.

**Status: agreed, partly built.** The mechanisms and their behaviour are
settled and measured. The configuration shape described here — `plies:`
and `tree:` — is the agreed target and is *not* what the current binary
reads; it still takes the earlier `code:` / `workspace:` / `live:` form.
The gap is listed at the end.

---

## 1. Three mechanisms, and nothing else

Everything in a composed tree arrives by one of three routes. This is the
whole vocabulary.

| mechanism | covering | what happens on write |
|---|---|---|
| **overlay** | merges | content in the upper: **written directly**; content from a lower layer: **copied up** into the upper first, then written |
| **bind, read-only** | covers completely | the write **fails** with `EROFS` |
| **bind, read-write** | covers completely | **written directly into the source** — no copy, no divergence |

Covering is not a separate choice. A bind covers by nature; merging is
what an overlay does. Choosing the mechanism chooses both.

The overlay row is the only place where the outcome is not visible from
the outside: two files side by side in the same directory can behave
differently, depending on which layer each currently lives in. That is
what `ply resolve` exists for.

## 2. Layers, read top-down

- A layer either **provides** a path or it does not.
- For a **file**, the topmost providing layer wins.
- For a **directory**, every providing layer is **merged**.
- The base ply is the topmost layer, and the only writable one.

`merge` is therefore not a mode to switch on: it is what directories do.
`override` — a lower layer replacing the base's copy — is not offered at
all, and the reason is not squeamishness: a read-only bind of the lower
over a path the base *tracks* makes git report the base's tracked files
as deleted, and a `git commit -a` would record that deletion.

## 3. Read-only is the absence of a declaration

Only the base is written from the composed tree. Lower plies are never
written — not because writing them is impossible, but because the code
is what you edit in the tree, and a workspace is a template shared across
projects.

So a path resolving to a lower ply is bound read-only. Not to forbid
something, but so that the failure is **loud and immediate** rather than
silent and delayed: copy-up would put your edit in the base repository
while leaving the workspace untouched, and the next composition would
show you your own edit from the base — so the change looks applied and is
not. You would find out in another project, weeks later.

Where you genuinely need to write, declare a writable bind. It writes
into the source, so the change lands where you would look for it.

## 4. What a layer contributes

**What git tracks in it** — not what its directory happens to contain.

A repository has already said which of its files are content and which
are generated: that is what `.gitignore` is. Reading the directory would
sweep up `.mypy_cache`, `.pytest_cache` and an editor's state directory,
and each would then have to be named again in a list that drifts out of
step with the `.gitignore` beside it.

It matters more than tidiness. Contributed paths are mounted read-only,
so a contributed cache directory is one the tool writing it can no longer
write — failing far from the cause.

Never contributed, whatever git says:

- **git metadata** — `.gitignore`, `.gitattributes`, `.gitmodules`,
  `.mailmap`. Every repository has them; the ones governing the composed
  tree are the base's.
- **anything in that ply's `exclude`** — for judgements no rule can make,
  such as a workspace's own `README.md`.

A ply that is not a git repository falls back to reading the directory
and says so, because without git there is no way to tell generated files
from content.

## 5. The filtered view

The contributed names are bound read-only into a private, empty directory,
and *that* is given to the overlay as its lower layer — not the ply
itself.

Without it, filtering decides only what is *protected*, not what is
*visible*: the overlay's lower would be the whole ply directory, so an
untracked cache would still appear in the tree, unprotected, listed by
`git status`, and copied up by anything writing to it.

With it, `exclude` means exactly what it says: the name is absent, not
covered by an empty placeholder.

Cost: one bind per contributed name, so a real composition runs to a few
dozen mounts rather than eight. The alternative — leave the raw ply as
the lower and cover excluded names with empty directories — is cheaper by
about a dozen mounts and leaves empty entries visible in the tree.

### Directories that contain a writable hole are assembled

A bind needs its target to exist. A machine-local file such as
`.claude/settings.local.json` can never exist in the ply — it is
gitignored by nature — so there is nothing to bind onto.

So a contributed directory with a hole declared inside it is not bound
whole into the view. It is assembled: a real directory holding a bind of
each contributed entry, **plus an empty placeholder for each declared
hole**. The placeholder gives the read-write bind something to attach to.

The alternative — making the directory its own small overlay with the
tool's storage as upper — would allow any file to be created there, and
would also let an edit to a *tracked* file copy up silently into scratch
storage. Precision was preferred: what is writable is declared, and
everything else fails loudly.

Note what this means: a `writable:` line does not merely add a mount, it
changes how the view is built. That kind of consequence is invisible in
the configuration, which is the second reason `ply resolve` is not
optional.

## 6. Writable holes: four reasons

| # | why | example | source |
|---|---|---|---|
| 1 | isolation target — a checkout must not land in the overlay's upper | `.claude/worktrees` | tool-provided, starts empty |
| 2 | the record — written by its own tool, into its own repository | `.registry` | a ply with `role: bind` |
| 3 | machine-local runtime state, deliberately untracked | `settings.local.json`, `*.lock` | tool-provided, starts empty |
| 4 | scratch space | `scratch/` | either |

The workspace's own `.gitignore` names most of them: every path in it
that lies *inside* a contributed tree is, by definition, something that
exists at runtime and is not content. `ply doctor` suggests these; the
configuration declares them. Suggested, never inferred — `__pycache__` is
in that same list and is merely junk.

Storage provided for a hole is **created and never removed**. It starts
empty and then holds real work: a `down` that cleaned it would take your
unfinished worktrees with it.

## 7. Disjointness, and why it holds itself up

The rule: **the base and the lower layers share no top-level name.**

It is self-maintaining. No overlap means every lower-provided name is
bound read-only; every name bound read-only cannot be copied up; nothing
copied up means the base never acquires a copy of a lower path — so the
overlap never appears.

The exception is a directory provided by both, which merges. There
copy-up is not a leak in the design, it *is* the merge — the two are the
same thing, and you cannot have one without the other. So a merge stays
possible but is a stated decision, warned about on every run until it is
acknowledged.

### Copy-up leaves a trace

On copy-up the kernel typically sets `trusted.overlay.origin` on the
upper file, and `trusted.overlay.impure` on the containing directory.
They are only visible with `CAP_SYS_ADMIN`, and git never stores xattrs,
so a commit carries no trace of them.

That makes them a **forensic marker**: a file in the base tree carrying
an origin xattr did not get there by being written, it arrived by
copy-up. Where copy-up cannot be prevented, it can at least be found
afterwards — which is the difference between discovering a divergence at
the next `doctor` and discovering it in another project weeks later.

*Unmeasured:* exactly when `origin` is written depends on the kernel's
`index` / `xino` / `nfs_export` configuration. Worth measuring on a live
composition rather than assumed.

## 8. Git wiring

Two mechanisms, and they are not equally strong:

- `.git/info/exclude` keeps names out of `git status` and `git add .`. It
  is convenience: `git add -f` stages them anyway.
- the **pre-commit hook** refuses a commit containing them. That is the
  boundary in git terms.

Neither is the guarantee. The read-only mounts are.

Both are generated from one list, so they cannot drift apart:

> everything the tree adds that is **not the base's** — the layers'
> contributions minus the overlapping names, **plus every bind target**.

Two details that are easy to get wrong:

- **Overlapping names must not be excluded.** For a name both sides
  provide, the tree shows the *base's* file, which is tracked. Excluding
  it would hide the base's own changes from `git status`.
- **Bind targets must be excluded.** A record repository bound at
  `.registry` appears in the tree and the base knows nothing about it, so
  `git status` would report it as untracked and `git add .` would stage
  it. The proof of concept never saw this: there the record sat inside an
  already-excluded directory.

## 9. What is deliberately impossible

- **Hiding content that belongs to the base.** The base is the top layer;
  covering it with an empty directory would make git report its tracked
  files as deleted. If you do not want something in the tree, take it out
  of the repository — that is git's job, not the composition's.
- **Preventing copy-up through an overlay mount option.** No such option
  exists: on this kernel the overlay module offers `redirect_dir`,
  `xino`, `index`, `nfs_export` and `metacopy`, and `metacopy` only
  defers the *data* copy. The only native way to have no copy-up is to
  have no upper layer, which makes everything read-only. Hence the binds:
  a read-only bind removes the path from the overlay, so there is nothing
  left to copy up.
- **Overlayfs's own way of hiding a lower entry** — a whiteout, which is
  a `0:0` character device written into the *upper* layer, i.e. into the
  base repository. Excluding a name from the filtered view achieves the
  same result and writes nothing anywhere.

### And a filesystem of our own

Writing the composition as a FUSE filesystem in Go was considered and
**rejected**. It is the architecturally tidy answer — the task is
literally "serve a computed view of several directories", which is what a
filesystem does, and it would dissolve the mount count, the root
requirement, the placeholder files and the whole filtered view.

It was rejected on blast radius. A mistake in a mount composition
produces a wrong *view*: visible, annoying, and reversible, with every
underlying directory intact. A mistake in a filesystem produces wrong
*data on disk*, and git is the least forgiving possible client — it mmaps
packfiles, replaces its index by atomic rename, depends on fsync
ordering, and hardlinks objects. A subtle error in rename or fsync
semantics corrupts a repository in a way that surfaces days later.

The characteristic failure is also worse than a failure. When a FUSE
daemon deadlocks, the mount point hangs: every process touching it blocks
uninterruptibly, and in a development tree that is the editor, the
language server, the test runner and the agent at once. A kernel mount
cannot do that.

And it is an obligation that does not belong to this tool. Being a
filesystem means owning inode allocation, cache coherency, locking and
every kernel's changes to the FUSE protocol, indefinitely — for a program
whose actual value is a configuration model. The gains it offered are
ergonomic: mount count is a readability problem, not a correctness one,
and needing root once per session is an installation nuisance rather than
a design flaw. Trading an install step for maintaining a filesystem
underneath git is a bad trade.

The model above stays free of mount vocabulary anyway. Not to keep the
option open, but because rules that are stated in terms of a mechanism
cannot be checked against it.

## 10. Configuration shape

```yaml
plies:
  code:      { path: ~/dev/diet-coach,           role: base }
  workspace: { path: ~/dev/diet-coach-workspace, role: layer }
  general:   { path: ~/ply/workspace-general,    role: layer }
  registry:  { path: ~/dev/diet-coach-registry,  role: bind }

live: ./live

tree:
  - from: workspace          # contributes what git tracks in it
  - from: general
  - path: .registry
    from: registry
    writable: true
  - path: .claude/settings.local.json
    writable: true           # no source: storage is provided, starts empty
  - path: .claude/worktrees
    writable: true
```

Roles: **base** is the upper and the only writable layer, exactly one;
**layer** is a read-only lower, and the order in `tree` is its
precedence; **bind** is not a layer at all — its own mount at a fixed
path, which may be written.

A path inside a ply is written `<ply>:<path>`, for the escape hatch:

```yaml
  - mount: bind_ro
    source: workspace:vendor/reference-docs
    target: docs/vendor
```

## 11. The derived plan is the output

The configuration is declarative and the plan is derived, which is only
safe if the derivation can be read back. Two things make it so:

`ply plan --literal` prints the plan in mount vocabulary, labelled rather
than positional, because the left-to-right precedence of `lowerdir` is
exactly what people read backwards:

```text
live:            ~/dev/diet-coach-live
bind_ro:         workspace:ops  ->  view-workspace/ops
overlay:         lower=view-workspace:view-general  upper=~/dev/diet-coach
bind_ro:         merged/ops     ->  live/ops
bind_rw:         ~/dev/diet-coach-registry  ->  live/.registry
bind_rw_empty:   live/.claude/worktrees
```

The work directory and the generated stores do not appear: that output
is for reviewing decisions, not plumbing. Saved to a lockfile and
committed, a change in a workspace repository shows up as a diff rather
than as a silent change in behaviour.

`ply resolve <path>` answers the inverse question, which is the one
people actually have:

```text
$ ply resolve live/docs/note.md
  overlay, from a lower layer (workspace), inside a MERGED directory
  no bind: the base provides docs/ too
  on write: COPY-UP into the code repository
```

## 12. Not yet built

The model above is agreed. The current binary implements the earlier
configuration shape and the earlier plan. Outstanding:

1. **`plies:` / `tree:` configuration** replacing `code:` / `workspace:`.
2. **The filtered view**, including assembling directories that contain
   a declared hole.
3. **Type-correct writable holes** — a file needs an empty file, not an
   empty directory; today a hole on a file path fails with `ENOTDIR`.
4. **Bind targets in the git exclude and the commit hook.**
5. **`ply plan --literal`** and the lockfile.
6. **`ply resolve`.**
7. **Recognition-based refusal** — a contributed name that is a symlink
   (a bind follows it and would pull in its target from anywhere on the
   machine), a name that is a file on one side and a directory on the
   other, a ply nested inside another ply, and contradictory declarations
   such as a writable path inside an excluded tree. Anything not
   recognised is refused with a reason rather than mounted on a guess.
8. **Copy-up forensics** in `doctor`, from the origin xattr.
