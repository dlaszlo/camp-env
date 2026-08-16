# camp — implementation specification

**Status: the single source for implementation.** This document
consolidates every decision made through 2026-08-15 evening: the
redesign conversation, the review sessions, the measurements, and the
final design conversation. Where it disagrees with `model.md`,
`redesign-2026-08-15.md` or `README.md`, **this document wins** — those
describe earlier states of the design. `constraints.md` remains valid
except where §4 below corrects it. A fresh session should be able to
build the whole system from this file alone, without asking — that is
the standard this document is written to; §23 registers what is
deferred and which load-bearing behaviours are still unmeasured, each
with the check that settles it.

**The 2026-08-16 rounds.** `docs/review-of-spec.md` raised 17
findings against this file. The owner settled five questions in the
design conversation of 2026-08-15 night — the lock model (§13), the
deletion of `--allow-detach` (§14, §16), one ordered `steps:`
sequence in place of the three mount keys (§6, §7), `.git` as an
ordinary declared mount rather than a camp-derived one (§6, §7), and
which promise gives way in the privileged mode (§3, §14) — closing
three findings and part of a fourth. A second pass on 2026-08-16
wrote in the remaining fourteen, following the review's own fix
paragraphs, with three further owner decisions: the privileged mode
is an unprivileged front end plus one narrow privileged helper (§14),
every record camp writes uses reversible C-style escaping over raw
bytes (§18), and a namespace session's end-of-session report persists
as a file the next camp command surfaces (§14, §16). Every "owner
decision, 2026-08-16" tag below belongs to one of these rounds.
Nothing from the review remains unanswered; §23 registers what stays
deferred or unmeasured.

Facts are tagged like `constraints.md`: *(measured)* — ran on the
owner's machine; *(read)* — documentation or a file's own content;
*(reasoned)* — a consequence, not itself run. Anything unmeasured that
the implementation depends on is listed in §23 with the check that
settles it.

---

## 1. What camp is

`camp` composes several git repositories into one working directory
using OverlayFS and bind mounts, without any of them learning about the
others. The code repository stays the product and only the product; the
workspace repository carries the development environment (instructions,
agent definitions, skills, docs-about-work); the filesystem presents
them as one tree. Writes land in the code repository or in machine-local
storage — **never** in the workspace, and never silently in the wrong
place.

The name: `camp` (2026-08-15). The earlier name `ply` collided four
times over: `jeansimeoni/ply` (a Claude-Code-asset package manager,
active since May 2026 — the same audience), `iovisor/ply` (a
well-known Linux kernel tracer — the same *Linux* audience), Python's
PLY (lex-yacc), and a Rust UI engine. The candidate trail, kept so it
is not rewalked: `crossply` was approved, then superseded by the
four-letter name; `xp` rejected (everyone hears Windows); `qw`
carried a niche claim (QualWeb's npm CLI installs a `qw` binary);
`devcamp` rejected (a bootcamp company owns it, and the compound
reads "coding bootcamp", losing the metaphor); `weft`, `jig`,
`saddle`, `tack`, `trestle`, `yoke` all taken — several by
AI-agent-space tools. **No short alias ships**: four letters dissolved
the alias question. The metaphor is exact: you pitch camp for the
work, break camp after, and leave no trace — "no trace" is a measured
guarantee, not marketing. Subcommands stay `up` / `down` (CLI muscle
memory); `pitch`/`strike` are the camp-true idioms and were noted as
flavor, not adopted; *commissioning/decommissioning* was considered
and rejected — wrong register (ships and power plants, not camps).

The environment directory layout:

```
$ENV/                          # e.g. ~/dev
├── .camp/
│   ├── config.yml             # the configuration (intent)
│   ├── work/<live-hash>/      # DISPOSABLE: overlay workdir, generated files
│   └── storage/<live-hash>/   # PERSISTENT: real work, never removed by camp
├── diet-coach/                # code       (upper)
├── diet-coach-workspace/      # workspace  (lower)
├── diet-coach-registry/       # record repository (mounted rw)
└── diet-coach-live/           # merged     (the composed tree)
```

`<live-hash>` is derived from the live path — the first 12 hex
characters of SHA-256 over the live directory's `realpath` (symlinks
resolved, no trailing slash) — never random, so an orphan left by a
crash can be attributed, and stable, because `work/` and `storage/`
naming depend on it. The locks (§13) use neither a hash nor a lock
file: they are taken on the upper's and the live directory's own
inodes. Inside
`storage/` the target's path is **mirrored**, not escaped: a path
component is capped at 255 bytes and `%2F`-escaping overflows it on deep
targets *(measured reasoning, settled)*. The two stores never share a
parent or naming scheme because their lifecycles are opposite: `work/`
may be garbage-collected whenever nothing is mounted; `storage/` holds
unfinished worktrees and machine-local state and may **never** be
removed by the tool.

Runtime state (records of privileged compositions) is JSON under
`$XDG_STATE_HOME/camp/` (fallback `~/.local/state/camp/`), written
and read only by the unprivileged front end (§14), so it always
resolves in the invoking user's environment — the schema is in §12.
The configuration states intent; state files are generated.
Namespace-mode runs create **no** state record *(measured)* — the
namespace is the state, and it vanishes with its last process. What a
namespace session does leave is its end-of-session **report** under
`$ENV/.camp/reports/` (§14, §16): output to be read once, never
authority.

## 2. Vocabulary

- **code repository** — the upper layer, the product, the only place
  ordinary writes land. (Older docs call a participating repository a
  "ply"; that term is legacy — this spec says code repo / workspace
  repo. The Go type `Ply` may be renamed during the rebuild or kept;
  user-facing text must not use "ply".)
- **workspace repository** — the lower layer; the development
  environment. Also called the **sidecar** in older docs.
- **layer** — an OverlayFS `lowerdir`/`upperdir`/`workdir` and nothing
  else. Plumbing vocabulary. A repository attached by bind mount is not
  a layer.
- **composed tree / live** — the merged directory you work in.
- **islands mount** — `camp`'s third mount type: the source's
  contributed entries shown read-only (the *islands*), standing in a
  writable, machine-local floor provided by camp storage (the *water*).
  See §11.
- **store** — the storage-backed writable directory under an islands
  mount or a sourceless `mount_rw`.
- **the gate** — the overlap check at `up` (§9).
- **the inventory** — the accepted snapshot of both repositories' root
  entries (§18).

## 3. Invariants — the non-negotiables

1. **camp only composes. It never modifies a repository.** Reading is
   not modification (`git ls-files`, `git check-ignore`, `ls-tree` are
   fine); the line is at writing. No file, no xattr, no hook, no
   exclude line is ever written into a repository. This is a property
   of the source code: filesystem writes live in one place, and no
   write target may be derived from a repository path (§22).
2. **The lower is never written — by any route.** No copy-up can reach
   it (it is the lower), no `mount_rw` may source from it (validated,
   refused), and while the composition is up it is bind-mounted onto
   itself read-only as the very first mount, so a process inside the
   composition cannot write it even via its absolute path. Consequence,
   measurable: after a session the workspace is byte-identical.
3. **The user deletes; camp checks.** camp never deletes a repository,
   a checkout or a branch; never clones; never commits. It removes only
   what it created (work dir, its own state). If the live directory is
   not empty after unmounting, that is evidence of a problem and is
   reported, never cleaned.
4. **`up` may refuse; `down` may only report.** A refusing `down` would
   wall the user in. `down` attempts teardown; what it cannot unmount
   (something holds it) it reports with the holder named. Drift is
   reported at `down` as well as `up`, because the end of a session is
   when the cause is fresh. **Reporting is not pretending**: camp
   never lazily detaches a mount to make the report look clean
   (`MNT_DETACH` is not in the tool at all — §14), so what could not
   be removed is still mounted, is said to be still mounted, and makes
   `down` exit non-zero.
5. **No `--force`, anywhere.** The escape hatch is configuration
   (`allow_overlap`): the same decision, recorded and diffable. Nothing
   can wall the user in, because the repositories are ordinary
   directories reachable without the tool.
6. **Anything not recognised is refused with a reason** — never mounted
   on a guess. The full refusal list is §17.
7. **Nothing camp does changes what a process outside the composition
   sees — without exception in the namespace mode, with two named
   exceptions in the privileged one.** In the namespace mode the
   kernel guarantees it (C20). In the privileged mode there is a
   single mount table for the whole machine, so there is no "inside"
   and no "outside", and exactly two effects are machine-wide, for as
   long as the composition is up: the composed tree appears at the
   live path, and **the workspace is read-only for every process,
   including the owner's editor**. That is the price of the mode, paid
   knowingly (owner decision, 2026-08-16 — the reasoning is in §14).
   Everything else stays scoped: the exclude mount goes on the live
   path, not the repository's (§10), and nothing is visible at live
   until the whole tree is built and verified (§7).
8. **Not a security boundary.** The read-only mounts prevent accidental
   writes and copy-up. A process inside can still walk to the backing
   stores and *read* anything. camp does not pretend to sandbox.

## 4. Facts the implementation is built on

Condensed from `constraints.md` (C-numbers kept for cross-reference)
plus this round's measurements. An argument that contradicts a measured
item here is wrong.

### OverlayFS

- **C1** Copy-up cannot be disabled; `metacopy` defers only data.
- **C2** An overlay with no upper is read-only — the only native
  no-copy-up.
- **C3** Directories merge; files do not (topmost wins outright).
- **C4** Deleting lower content writes a whiteout — a `0:0` character
  device — into the upper; hiding a lower dir sets an `opaque` xattr on
  the upper dir. This is why `opaque` was rejected as a mechanism: it
  writes into the code repository, survives `down`, and its xattr
  namespace differs by mode.
- **C5** *(now half measured)* Copy-up sets `user.overlay.origin` on
  the file and `user.overlay.impure` (+`user.overlay.uuid`) on the
  upper dir in the namespace mode *(measured)*; the privileged
  `trusted.*` side is unmeasured (§23). Git never stores xattrs (C22),
  so these are forensic markers only.
- **C6** The workdir must be empty, on the same filesystem as the
  upper, not inside it. The kernel leaves a `work/` inside it: in the
  namespace mode it is **user-owned, mode 000** *(measured)* — camp
  chmods and removes its own; under root it is root-owned and camp
  gives it away or removes it while it still has privilege.
- **C7** `lowerdir` syntax: `:` separates layers, `,` options, `\`
  escapes; paths containing them must be escaped, and `plan` prints the
  escaped string it will pass.
- **C8** One upper must serve one overlay mount. **The kernel does not
  enforce this** *(measured: a second overlay on the same upper mounted
  without complaint)* and the current binary does not either *(measured:
  two concurrent runs each composed the same upper)*. Enforcement is
  camp's job: the locks, §13.
- Overlay super-options echo what was passed **plus kernel-added
  defaults** (`redirect_dir=nofollow`, `uuid=on`, and in a user
  namespace `userxattr` even when not requested) *(measured)* —
  verification must compare per option, not by string equality.

### Bind mounts

- **C10** The target must already exist; a bind cannot create its own
  mount point.
- **C11** At the `mount(2)` syscall level, `MS_BIND|MS_RDONLY` in one
  call silently ignores the read-only flag; read-only takes a second
  remount (or the new mount API). mount(8) now does this itself
  *(measured)*, but Go code issuing raw syscalls must two-step — and
  §15's verification never trusts the call, only the result.
- **C12** A bind follows symlinks — a symlink source could pull in
  anything on the machine; symlink sources are refused.
- **C13** Types must match: directory onto directory, file onto file.
  File-on-file binds work *(measured)*.
- **C14** Mounts propagate by default on a systemd machine (`/` is
  `shared:1` *(measured)*); every camp mount is made private as it is
  created, and verified private. (History: propagation once produced
  twelve mounts where eight were planned, four of them on the
  workspace's own path.)
- **A bind is a live view, not a snapshot** *(read — standard bind
  semantics, relied upon)*: files created in the source after mounting
  are visible (and, under a read-only bind, protected) immediately.
  Only new *top-level* names need a new `up`. This fact underpins both
  the per-file-mount rejection (§7, derived root protections) and the
  coarse exclude (§10).
- **`mount --move` carries a mount's submounts** *(measured)* — the
  staging option in §14 relies on this; the parent must be private.
- **A covered (shadowed) mount stays alive in `/proc/self/mountinfo`
  but is unreachable by path** *(measured)*: presence in mountinfo
  proves nothing about what a process sees. Path-based checks
  (`statvfs`, `stat`) are the authority (§15).

### User namespaces

- **C15** Mounting needs `CAP_SYS_ADMIN` regardless of file ownership
  *(measured)* — from root, or from a user namespace the process
  creates.
- **C17** Ubuntu ≥23.10 restricts unprivileged user namespaces
  (`kernel.apparmor_restrict_unprivileged_userns=1`). An AppArmor
  profile grants the permission **to one binary path**
  (`packaging/apparmor/camp`, attaches to `/usr/local/bin/camp`); a
  copy of the binary elsewhere is not covered, and an interpreted
  program would need the profile on the interpreter. The sysctl stays
  on. The profile is installed on the owner's machine *(measured; both
  modes report available in `doctor`)*.
- **C18 (amended by the identity decision, §14)** Without `newuidmap`,
  a single uid can be mapped. Mapping it to 0 was chosen originally
  because caps drop on `execve` for non-zero euid; §14 replaces this
  with an own-uid mapping plus ambient capabilities.
- **C19 — CORRECTED.** `setgroups` is denied inside the namespace, so
  supplementary groups cannot be *changed* and display as `nogroup` —
  but the kernel credential **retains** them and host-filesystem
  permission checks keep honouring them. Measured: inside the
  namespace, `docker ps` succeeds against the `root:docker 0660`
  socket and `/var/log/syslog` (`root:adm 0640`) is readable.
  **Consequence: the pre-push docker gate works inside the namespace
  mode; pushing to main from inside works.**
- **C20** Mounts are invisible outside the namespace. A daemon or
  GUI program started outside cannot see the composed tree.
- **C21** The namespace and all its mounts vanish with the last process
  *(measured: zero mounts, zero state after exit)*. Teardown cannot
  fail; the namespace mode has no `down`.
- **C34 (new, measured).** A read-only remount inside a user namespace
  must preserve the source mount's **locked flags** (`nosuid`, `nodev`,
  `noexec`, atime flags) or it fails `EPERM`. `/tmp` here is tmpfs
  `nosuid,nodev` — the current binary's namespace mode fails on it;
  the same composition on ext4 (`rw,relatime`) works. The remount code
  must OR in the source's locked flags; `doctor` warns when the
  environment sits on such a filesystem.
- `flock` is inode-based and crosses namespaces *(read)*; a lock fd
  survives in any child that inherited it (§13).
- `newuidmap`/`newgidmap` are **not installed** on this machine;
  `/etc/subuid`/`/etc/subgid` **are** configured *(measured)*.

### Git

- **C22** Git does not store xattrs. **C23** Git cannot track an empty
  directory — a committed mount point needs a file in it.
- **C24 + Part D** A worktree's git directory is an absolute path
  compared as a string. A worktree created *through the live tree*
  records the live path **on both sides**, so after `down` both
  pointers are dead: the checkout's files are intact but git cannot see
  them. `git worktree repair <storage-path>` rewrites both pointers to
  paths that outlive the composition (code + storage), after which the
  worktree is composition-independent. `git gc` prunes a
  dead-pointer registration after `gc.worktreePruneExpire` — default
  **three months** — and auto-gc runs from ordinary commands: the one
  failure that happens while nobody is looking. Committed work survives
  on its branch; **uncommitted work is stranded as plain files**.
- **C25** `.git/info/exclude` is not a boundary: `git add -f` stages
  through it. It stops `git status` noise and `git add .` — the
  accidental leak, not the forced one.
- **C26** Covering a tracked path makes git report those files deleted
  (or modified), and `git commit -a` records it. Hence the target rule
  in §17.
- **C27** Linked worktrees share the common git directory:
  `info/exclude` applies to all of them at once, and a per-worktree
  `info/exclude` is **not honoured** *(measured)*.
- **C28** Git cannot represent a character device: a whiteout in a
  working tree is an untracked oddity `git add` refuses. (With §8's
  full coverage, deleting lower names through live is already
  impossible — `EROFS` or `EBUSY` — so the trap is precluded, but the
  constraint stands.)
- `git status` rewrites `.git/index` (stat cache); `git log` and
  `git ls-files` do not *(measured)*. Reporting code that runs git
  against a repository uses `--no-optional-locks` *(read — verify once
  when built)*.
- `.git/info/exclude` (and even `.git/info/`) may not exist — a
  repository initialised from an empty template has neither
  *(measured)*. camp refuses and prints the two commands
  (`mkdir -p .git/info && touch .git/info/exclude`); creating them
  itself would write into the repository.
- Gitignore pattern facts *(measured)*: an **unanchored bare name
  matches at every depth** (a line `scripts` hides new files under the
  code repo's real `frontend/scripts` and `deploy/mcp-server/scripts`);
  any pattern containing a separator anchors itself; a leading `/`
  anchors a bare name. Escape set: `\`, `[`, `]`, `*`, `?`, and a
  trailing space; `#` and `!` need nothing once the line starts with
  `/`. A **newline in a name cannot be expressed** — and the attempt
  silently ignores the intended file while hiding two unrelated names —
  so such a name is refused outright.
- Both real repositories set `core.hooksPath` (code: absolute
  `deploy/git-hooks`; workspace: relative `scripts/hooks`)
  *(measured, re-checked 2026-08-15)*: `.git/hooks/` is bypassed
  entirely, so an installed hook would never run. This is why there is
  **no camp pre-commit hook** — see §20.
- The code repository's real `pre-push` hook tests a `git archive` copy
  in a temp directory, not the working tree *(read from the hook)*: it
  never needs the composed tree to be visible to dockerd, and (per the
  C19 correction) it runs in both modes.

### Environment and instruments

- **C29** Command output is translated (`hu_HU.UTF-8` here): anything
  parsed runs under `LC_ALL=C`, and state is asked of `/proc`, not
  parsed from messages.
- **C30** `fuser -m` answers the wrong question (device, not mount);
  holders are named from `/proc/*/cwd`, `/proc/*/fd`, `/proc/*/exe`,
  `/proc/*/root`.
- **C31** `findmnt -R` fails on a non-mountpoint dir; `-T` asks the
  right question.
- **C32** Byte-wise comparison vs locale sort silently disagree; all
  name comparisons sort by bytes.
- **C33** `sudo` cannot prompt without a terminal: the privileged mode
  is always driven from a real terminal by the user; camp never assumes
  scripted sudo.
- Verification primitives: `statvfs(path).f_flag & ST_RDONLY` reports
  read-onlyness as a process would experience it *(measured)*; `stat`
  `(st_dev, st_ino)` equality across a bind proves source↔target
  identity *(measured)*; mountinfo optional fields empty ⇔ private
  propagation *(measured)*; a write attempt on a read-only mount fails
  `EROFS` without side effects *(measured)*. The overlay's identity was
  read from mountinfo's fstype field *(measured)*; `statfs f_type`
  against the overlay magic is its programmatic equivalent *(read)*.

## 5. The repositories, and what the migration must establish

camp is judged in the steady state *after* the migration of
`~/dev/diet-coach` (code) and `~/dev/diet-coach-workspace` (workspace).
Preconditions the migration must establish (camp verifies them via the
gate and validations, it does not perform them):

1. **Zero overlap between the two root namespaces.** The single
   permitted exception: `.gitignore` exists in both roots (each repo
   needs its own) and sits in `allow_overlap` — a *file* overlap, so
   the tree shows the code's tracked copy, which is correct: the live
   tree is governed by the code's ignore rules, and git applies no
   ignore rules to tracked files anyway, so excluding it would be
   inert. (`README.md` joins the list only if the workspace keeps one
   at its root.)
2. **Tool-bound names stay at the workspace root**: `CLAUDE.md`,
   `AGENTS.md`, `.claude/` — they must appear at the composed root and
   the tools hard-code them. (Whether `.mcp.json` follows them out of
   the code repository is an unresolved migration detail — flag it, do
   not assume.)
3. **Everything else the workspace carries moves under one container
   directory** (working name `.workspace/`), so no root name can ever
   collide with a code name.
4. **Generated files are cleaned out of the workspace working directory
   and named in its `.gitignore`** — caches regenerate through the
   live tree into the upper thereafter, governed by the code's
   `.gitignore`.
5. **`.registry` becomes its own repository** — a *precondition*, not
   hygiene, because no `mount_rw` may source from the lower (§3.2). The
   workspace keeps a **committed, empty `.registry/` mount-point
   directory** (with a placeholder file — C23) so the rw bind has an
   attachment point.
6. **No permanent lane worktrees.** Parallel agent work uses Claude
   Code's built-in worktrees under `.claude/worktrees/` (both
   repositories' `.gitignore` files already route them there).

## 6. Configuration language

Semantics fixed; serialisation is YAML at `$ENV/.camp/config.yml`
(commands search upward from the cwd for `.camp/config.yml`). The
target configuration for this environment:

```yaml
env: /home/dlaszlo/dev
merged: diet-coach-live            # $ENV/$MERGED

repositories:
  - { name: workspace, path: diet-coach-workspace }
  - { name: code,      path: diet-coach }
  - { name: registry,  path: diet-coach-registry }

overlayfs:
  lower: [workspace]               # a list; exactly one entry accepted today
  upper: code

allow_overlap: [.gitignore]

steps:
  - mount_rw:
      - { source: "code/.git",         target: ".git" }
      - { source: "registry",          target: ".registry" }
  - mount_islands:
      - { source: "workspace/.claude", target: ".claude" }
  - git_exclude                      # the shipped git step: generates the
                                     # exclude and binds it over
                                     # live/.git/info/exclude (§10, §19)
```

Rules of the file:

- **`steps:` is one ordered sequence, and its order is the mount
  order** (owner decision, 2026-08-16). The earlier language had three
  separate keys — `mount_ro`, `mount_rw`, `mount_islands` — and a rule
  saying declared mounts run "in file order". That rule was not
  implementable: in YAML a *sequence* carries order, a *mapping's*
  keys do not, so there was no defined interleaving between three
  sibling keys, and the review found the example and §7 already
  disagreeing about it. One sequence makes the order true by
  definition. The shape is the Ansible playbook / GitHub Actions
  `steps:` line — an ordered task list whose items name their own
  kind — and it fits a mount sequence, which really is an ordered
  series of operations building on each other.
- **A step is either a bare kind** (`- git_exclude`) **or a
  single-key mapping** from kind to its arguments
  (`- mount_rw: [...]`). An item with several keys, or a kind camp
  does not know, is refused (§17) — never guessed at (invariant 6).
- **Mount kinds take a list of `{source, target}` entries.** The total
  order is the steps in list order, and within a step its entries in
  list order. Sources are addressed `<repository-name>/<path>` (or a
  bare repository name for its root); targets are relative to the
  merged root.
- **`mount_rw` without `source`** is a plain writable hole: camp
  provides empty storage (mirrored path under `storage/<hash>/`) and
  it starts empty. This form stays in the language even though the
  `.claude` case no longer needs it.
- **The nesting rule, now checkable exactly as written**: an order in
  which an earlier mount's target lies inside a later mount's target
  is refused (the later would silently cover the earlier — the
  measured shadowing). Parent before child is legal and useful; child
  before parent is refused; the same target twice is refused. Because
  the sequence is a real total order, validation can walk it (§15) —
  and the rule then earns its keep on the everyday case: `git_exclude`
  targets `.git/info/exclude`, which lies inside the `.git` mount's
  target, so listing it before the `.git` bind is refused with the
  reason, instead of mounting an exclude that the next mount covers.
- **The generation step** produces the artefacts (§19) and mounts
  them. `git_exclude` is the shipped one; the custom form is
  `- generate: { command: ["prog", "arg", ...] }` — the same contract,
  an external program instead of the built-in git reads. Generation
  runs in the `prepare` phase, before anything is mounted and always
  as the invoking user; the mount — the exclude bind — happens at the
  step's own position. **At most one generation step per
  configuration** (§17): there is one exclude payload, and two steps
  claiming it cannot both be right. A composition that is not
  git-based simply does not list one, and then camp has no git
  knowledge anywhere in the run.
- **`.git` is declared, never derived** (owner decision, 2026-08-16).
  The code repository's `.git` bind is an ordinary `mount_rw` entry;
  camp does not add it on its own, because a core that reaches for the
  name `.git` carries git knowledge, and §19 exists precisely to keep
  git out of the core — a composition of two non-git directories never
  mentions the name, and neither may camp. Leaving the entry out of a
  git-based config is not a hole: `.git` exists in both roots, nothing
  covers it, so the gate (§9) refuses the composition before anything
  is mounted. The requirement is enforced structurally, by a rule that
  knows nothing about git.
- **Everything else is a set, not a sequence** (`allow_overlap`,
  `repositories`). One rule that only sometimes bites is worse than
  two honest ones.
- `lower:` stays a list while only one entry is accepted; a second
  entry is refused with a clear message. Several lowers are a later
  iteration and will be merged **by the tool** into a read-only
  `merged_lower`, not by giving overlayfs several lowerdirs; file
  shadowing between lowers must be reported when that lands (§23).
- A **deep target** (inside a directory) is a legal escape hatch for
  other projects but is not used here; it obeys the target rules of
  §17 like everything else (must exist; must not sit on code content).
- The `read_only:` self-freeze list sketched in the redesign was **not
  carried** into the language; `mount_ro` with a source covers every
  discussed case. Revisit only if a real use appears.

**The path language** (owner round, 2026-08-16 — a privileged
pathname engine needs a normative grammar, not conventions):

- **Bases.** `env:` is absolute (`~/` expanded; any other relative
  form refused) and resolved once with `realpath` at startup. Every
  other path field is relative: `merged` and a repository `path` to
  `$ENV`, a source to its repository's root, a target to the merged
  root, a generated islands entry to its store. An absolute path in
  any of these is refused.
- **Grammar.** A path field splits on `/`; every component must be
  nonempty, not `.`, not `..`, and free of NUL. A repository `name`
  is one such component. `allow_overlap` entries are **root names** —
  single components — because the gate compares root entries and a
  deeper path could never match one. A name containing a newline is
  refused at `up` (§17).
- **Comparisons.** Equal-and-nested target checks are lexical over
  the normalised component lists — no filesystem access, which is
  what lets validation walk the virtual tree (§7). Identity questions
  — two repositories the same, live inside a repository, nesting —
  are answered by `realpath` plus `(st_dev, st_ino)`, never by string
  equality. Two repositories resolving to one directory are refused.
- **Symlinks.** After `$ENV` itself is resolved once, no symlink is
  followed anywhere: sources, targets and every intermediate
  component are opened descriptor-relative with
  `openat2(RESOLVE_NO_SYMLINKS | RESOLVE_BENEATH)`, and a symlink in
  any mount operand refuses the composition — C12 generalised from
  "the source" to every component of every operand.
- **Types.** A mount source or protected root entry is a regular
  file or a directory, nothing else: a symlink, socket, FIFO or
  device at a lower root refuses (the inventory records types — §18 —
  so the refusal can say what changed).

## 7. The composition — the full mount sequence

The run has two halves: a **frame** that always executes, in a fixed
order, and the configuration's own **`steps:`** (§6) in the middle of
it. The split is the point: everything the safety rests on is frame,
so no configuration can move it, weaken it or leave it out, while
`steps:` carries what is genuinely this composition's decision. Every
mount is made private as it is created (C14) and made read-only by
remount where applicable, replicating locked flags (C34).

**Frame — the part before the steps.** Steps 1–5 compute and prepare;
mounts begin at 6.

1. **Take the two locks** (§13): the upper's inode and the live
   directory's inode, both exclusive, both non-blocking. Refuse if
   either is held. In the privileged mode the front end takes and
   holds them through the whole `up` (§14); in the namespace mode the
   launcher takes them and the init inherits the fds (§14).
2. **Validate and gate** (§9, §17) — the complete resolved plan,
   statically, **while nothing is mounted**, which is the one moment
   repairing repositories by hand is safe. Preserve that property.
   Validation plays the sequence through on paper: it walks the steps
   in their own order over a virtual tree, so every target is judged
   in the state its own step will actually meet — a target supplied by
   an earlier mount counts as present, and a later mount that would
   cover an earlier one is refused here, before any of it exists.
   This is the static half; what generation produces is validated in
   step 4, after it exists.
3. **Generate** (`prepare` phase, §19, always as the invoking user):
   the generation step's payloads — the exclude, the islands
   expansions — into `work/<hash>/`. Nothing is mounted yet, so the
   output is still only data at this point.
4. **Validate the generated output as hostile data** (§19): the
   concrete plan now exists, so the checks static validation could
   not run, run here — every islands entry against its grammar, type
   and containment rules, the exclude payload byte for byte against
   §10's assembly, and the equal/nested and tracked-code checks of
   §17 re-run over the expanded mounts. The generator ran as the
   user, and whoever can edit the configuration can edit the step:
   nothing it produced steers a mount unchecked.
5. *(privileged mode only)* Open the staging root
   `work/<hash>/staging`, mode 0700, and build everything below in it.
   Mandatory in this mode, not optional hardening — see §14.
6. **Workspace self-bind, read-only** — the workspace bound onto
   itself, first: while the composition is up, the lower cannot be
   written through its own path either. In the namespace mode this
   bind exists only inside the namespace (C20): processes *inside*
   (the agents) cannot write the workspace even by absolute path,
   while the owner outside edits it freely, and those edits appear
   through the binds immediately (live view). In the privileged mode
   the same bind is machine-wide, and that is the mode's price:
   the workspace is read-only for everyone until `down` (invariant 7,
   §14).
7. **The overlay**: `lowerdir=workspace, upperdir=code,
   workdir=$ENV/.camp/work/<hash>/work` at the merged root. Explicit
   xattr option per mode: `userxattr` in the namespace mode,
   `nouserxattr` under root (C9; the kernel forces `userxattr` in a
   userns anyway *(measured)* — pass it explicitly and verify
   per-option).
8. **Derived root protections**: every lower root entry that is neither
   a mount target nor in `allow_overlap` gets a read-only bind over
   its path in the merged tree — a directory bind for a directory
   (`.workspace`), a file bind for a root file (`CLAUDE.md`,
   `AGENTS.md`). Derived from the raw root listing at `up`; no names
   in the configuration. This is what makes every workspace-provided
   path fail loudly (`EROFS`) instead of copying up — the POC's
   measured architecture (thirteen write attempts, all refused, zero
   residue), restored at three to four mounts in the steady state.
   Per-file mounting was considered and **rejected**: a bind is a live
   view, so directory-level protection already covers files born in
   the source mid-session; per-file coverage would cost thousands of
   mounts (the workspace tracks **3,183 files** against **15 tracked /
   19 raw root entries** today, ~5 root entries post-migration —
   *measured*); and a store under content directories would silently
   absorb stray writes into storage — "looks applied, exists nowhere",
   the design's worst failure shape. Content directories must refuse
   loudly; only the dual-natured directory absorbs.

**The steps** (§6), in their declared order:

9. Each entry of each step, in sequence. For this environment: the
   `.git` rw bind, the `.registry` rw bind, the `.claude` islands
   mount (store first, then its islands — ordering internal to the
   entry, §11), and the `git_exclude` step's bind of the generated
   file over **`live/.git/info/exclude`** — reachable only through the
   live path *(measured: the code path keeps reading the repository's
   own file)*. The generation step's output was already validated as
   hostile data in step 4 (§19); its mount here mounts that verified
   payload and nothing else.

   **On `.git`**: it is an ordinary declared mount here, not a special
   step, and camp never adds it on its own (§6). What it does is
   unchanged and still load-bearing: both repositories have `.git` at
   the root — the one overlap that never migrates away — and
   directories merge (C3), so without this bind the two git
   directories would union their loose refs and objects, and
   `refs/heads/main` could resolve to the workspace's history. The
   bind covers completely; nothing merges. Its *omission* is caught by
   the gate before anything mounts (§9), and its *failure* aborts the
   composition like any other mount's.

**Frame — the part after the steps:**

10. **Verify everything** (§15). Any failure: unmount in reverse,
    name the failed check, exit non-zero.
11. *(privileged mode only)* `mount --move` the verified staging tree
    onto the empty live directory — submounts follow *(measured)*,
    which is why the parent must be private (C14) — and then **run the
    path-based verification again at the live path**. Until this
    moment nothing of the composition is visible at live.
12. **Declare up** (privileged: the record moves to phase `up` — §12;
    namespace: the init forks the workload — §14).

Teardown is the list reversed, and never lazy (§14): what is held
stays mounted and is reported (invariant 4). After unmount, camp
removes its own work dir (chmod the kernel's mode-000 `work/` first —
it is the invoking user's in the namespace mode, camp's-as-root in the
privileged one) and reports anything left in the live directory
(invariant 3: evidence, not garbage).

What copy-up means after this: **nothing lower-provided is writable**,
so copy-up cannot occur at all in the steady state; new files can be
born only at the merged root and in code-owned directories, where they
belong — in the upper. The gate (§9) still re-checks every `up`, and
the whiteout trap (C4/C28) is precluded rather than handled.

## 8. Ownership of every path in the tree

| path in the composed tree | owner | writable | mechanism |
|---|---|---|---|
| everything not listed below | code repo | yes — the product | overlay upper |
| `.git` | code repo | yes — it *is* the code repo | rw bind |
| `.git/info/exclude` | camp (generated) | no | file bind over the live path |
| workspace root names (`.workspace`, `CLAUDE.md`, `AGENTS.md`) | workspace repo | **no** — `EROFS` | derived ro binds |
| `.claude` | mixed | islands no; water yes (machine-local) | `mount_islands` |
| `.registry` | registry repo | yes — writes go to its source | rw bind |
| `.gitignore` | code repo (workspace's copy shadowed) | yes | overlay file rule + `allow_overlap` |

## 9. The overlap gate

At `up`, before any mount: if a name exists in **both** the lower and
the upper root and is not in `allow_overlap`, the composition does not
start.

- Applies to **directories as well as files** — a directory overlap is
  a merge, which is a real decision, not a detail.
- **Names completely covered by a mount target are exempt, and the
  descent stops there** (`.git`, `.claude`): the overlay's merge at a
  covered path is never visible, so it cannot leak. Without this
  exemption the gate would be unsatisfiable on `.git` forever.
- **Inside an allow-listed directory the check descends**: a file
  present on both sides within a merged directory is exactly the trace
  of a copy-up — the thing most worth catching. (Steady state has no
  allow-listed directories, so no descent in practice.)
- **No `--force`** (invariant 5). The steady state is zero overlap
  (`allow_overlap` ≈ `[.gitignore]`); after migration the gate should
  almost never fire, and when it does, something really changed.
- The gate reads the same configuration that declares the mounts, so
  it knows every target before anything is mounted.
- **The refusal message is load-bearing**: it names the path, what is
  on each side, **which side is the one that matters** (for a file:
  the upper's copy wins and the lower's becomes unreachable), and both
  ways out (fix by hand — safe, nothing is mounted — or add to
  `allow_overlap`). "Overlap detected" is not acceptable. §21 applies.

## 10. The exclude

**Why it exists.** The read-only binds stop *writes*; git *reads*
through them. Without an exclude, `git status` in live lists every
workspace name as untracked and `git add .` stages their content into
the code repository. Three levels of defence, stated honestly: the
kernel stops writes (binds); the exclude stops *accidental* staging;
nothing stops `git add -f` — that is detected, not prevented (§20).

**Who provides it.** The `git_exclude` step (§6, §19) — a step in the
configuration's `steps:` list, not a fixed part of the frame, because
the core carries no git knowledge. A composition that does not list it
has no exclude at all; `plan` says so plainly rather than silently
leaving the defence out.

**How it is provided.** Generated at every `up` into
`work/<hash>/exclude` and **bind-mounted over `live/.git/info/exclude`**
(never written into the repository — the old binary's
`gitwire.InstallExclude` does exactly that and must be deleted). The
mounted content = the repository's own existing exclude lines,
unchanged, + the generated block, whose first line is the marker:
`# camp:generated <live-hash> -- do not edit; regenerated at every up`.
**Byte-exactly** (owner round, 2026-08-16): the payload is the
repository's exclude bytes, unchanged and complete; if that file is
nonempty and does not end in a newline, exactly one `\n` is inserted
before the block — direct concatenation would fuse the marker into
the last pattern — and an empty file contributes nothing.
Verification (§15) compares the mounted file against this whole
payload byte for byte; a marker-prefix match alone would accept a
payload whose repository half was dropped (§19). After `down`, no
trace.

**Why the live path and not the repository's**: measured — a bind on
`live/.git/info/exclude` is visible only through live; `code/` and any
checkout registered outside keep reading the repository's own file;
`git status` through live honours the generated list while `git status`
in `code/` does not. Under the privileged mode this scoping is the
difference between a composition detail and a machine-wide change to
what git reports in `code/`. Worktrees under `live/.claude/worktrees`
resolve the common git directory through the live path and therefore
*do* see the generated list (C27) — intended, and harmless with
anchored lines and zero overlap.

**Derivation:**

```
exclude = (lower root entries) − (allow_overlap) + (every mount target, by target path)
```

- **Shape: one line per lower root entry, anchored with a leading
  `/`, in the shortest possible form** — `/CLAUDE.md`, `/AGENTS.md`,
  `/.claude`, `/.workspace`, `/.registry`; about five lines in steady
  state (19 today, against the 3,183 lines a file-level list would be
  — *measured*). Decided explicitly by the owner (the redesign conversation
  had left coarse-vs-file-level unresolved; file-level was the owner's
  original ask and was consciously retired). Why coarse wins: a
  workspace file born mid-session — normal workspace editing, arriving
  instantly through the live-view binds — is automatically covered by
  its directory's line; under file-level enumeration it would match no
  line and `git add .` could stage it, in exactly the window the tool
  cannot re-check. The reviewable-inventory value of file-level lives
  in §18 instead.
- **Why coarse is lossless**: the zero-overlap invariant. No name
  exists on both sides, so a root line can never hide code-side
  content — and the gate re-verifies the invariant at every `up`, so
  the exclude never runs on a tree its shape is not valid for.
- **Why the leading `/` is load-bearing, not cosmetic**: the gate
  compares **root entries only**, so a lower root name and a
  same-named directory at depth in the code repository never meet in
  the comparison — measured concretely: an unanchored `scripts` line
  hides new files under the code repo's real `frontend/scripts` and
  `deploy/mcp-server/scripts`, and no gate ever fires. The anchor is
  the only guard for this class. (File-level enumeration was immune by
  construction — any pattern containing a separator self-anchors —
  which is why the coarse form must carry the `/` to match it.)
- `allow_overlap` entries are never excluded (file: inert; directory:
  would hide the upper's own untracked files).
- Mount targets are excluded **by target path**, whatever their depth
  — a deep escape-hatch bind yields a deep, self-anchoring line.
- File-level enumeration survives in exactly one place: **inside an
  allow-listed directory**, where the two sides' content genuinely
  mixes. Steady state: none.
- Escaping per §4's gitignore facts; a name containing a newline is
  refused.
- Missing `.git/info/exclude` (or `.git/info/`): refuse, print the two
  commands, the user runs them (invariant 1 and 3).

**Residual windows, identical under either shape, handled by
detection**: a new lower *root* entry mid-session (no line, no bind
until next `up`; the inventory blocks at next `up`, the `down` report
names it same-day); a new file inside an allow-listed directory
(snapshot staleness; avoid directory overlaps — the gate makes them
explicit decisions).

## 11. `mount_islands`

The third mount type. `{ source: "workspace/.claude", target:
".claude" }` means:

1. camp mounts a **writable store** over the target from its own
   storage (`storage/<hash>/<target>`, mirrored path, created on first
   use, owned by the invoking user, persistent — never removed);
2. inside it, camp mounts a **read-only island** for every entry the
   source **contributes**: a directory entry as one directory bind, a
   file entry as a bind onto a **placeholder file camp creates in its
   own store** (file-on-file binds work *(measured)*; creating the
   placeholder writes camp's storage, never a repository).

Why this shape is the only legal one for a dual-natured directory (it
was §10.3's open question; it is settled by structure, not taste): a
machine-local file like `settings.local.json` exists in no repository,
so a writable hole has nothing to bind onto (C10), and creating the
attachment point through the overlay would copy the directory up into
the code repository — forbidden. Only a store covering the whole
directory can provide attachment points without touching a repository.

One more alternative was rejected on mechanism, recorded so it is not
relitigated: making the directory **its own small overlay** with camp
storage as the upper. There, an edit to a *tracked* entry copies up
**silently** into scratch storage — the change "looks applied" and
exists in no repository, the design's worst failure shape. The islands
form is precise instead: what is writable is the water, everything
contributed fails loudly.

Semantics and consequences:

- **Runtime writes — known and future-unknown names — land in the
  water**: machine-local, persistent. Approvals survive sessions;
  worktrees survive `down`. Measured runtime names today:
  `settings.local.json`, `worktrees/`, `scheduled_tasks.lock`,
  `commands/` — including one nothing predicted, which is why the
  name-set is treated as open and absorbed rather than enumerated.
- **Islands fail loudly**: editing a workspace-contributed entry
  through live is `EROFS`, not a silent copy anywhere.
- **The workspace's own runtime junk never appears**: the store covers
  the raw lower directory entirely, so the workspace's own
  `settings.local.json` or lock files (from using the workspace repo
  standalone) are invisible in the composed tree.
- **"Contributes" = what the source repository tracks there**, from
  the same enumeration step as the exclude (§19; git by default; a
  non-git source falls back to the raw listing and `doctor` says so).
  Derived from tracked content, *not* the raw listing — the raw
  listing would give islands to the source's own junk.
- **Ordering is internal to the entry** (store, then islands) and
  cannot be misdeclared; where the entry sits among the other mounts
  is the `steps:` order (§6). §15's identity check still verifies
  every island by path, so even an implementation bug surfaces at
  `up`.
- **Expansion is visible**: `camp plan` prints the concrete derived
  mounts (today: one store + `agents`, `output-styles`, `skills`,
  `settings.json`); `up` prints the mounted island list.
- **Staleness**: a new tracked entry in the source gets its island at
  the next `up` — same class as the exclude; reported, accepted.
- **Island-over-water collision**: if, at `up`, the water already
  holds a name that a new island would cover (the user's machine-local
  file, now shadowed by newly tracked source content), camp **refuses**
  with both sides named — silently hiding the user's local content is
  the design's enemy, and the remedy is the user's move (rename or
  remove the water file; the user deletes, camp checks). The one
  exception is camp's own recorded scaffold — next bullet.
- **The scaffold manifest** (owner round, 2026-08-16; closes the
  second-run refusal the review found): the attachment points camp
  creates in the water — the placeholder file under a file island,
  the directory under a directory island — persist in storage, so on
  the next `up` they are already in the water, and the collision rule
  above would refuse camp's own objects. Provenance is therefore
  recorded: `storage/<live-hash>/.camp-scaffold`, beside
  `.camp-target` and outside every exposed store subtree, lists every
  camp-created attachment point as a `<type> TAB <store-relative
  path>` record (§18's encoding), updated by temp-file-and-rename.
  **Write-ahead**: the manifest entry is written and synced *before*
  the attachment point is created, so a crash leaves at worst a
  recorded name with nothing on disk — recreated harmlessly — and
  never an unattributable object.
- **The scaffold lifecycle**, one rule for file and directory islands
  alike: at `up`, a needed attachment point that already exists is
  accepted only if the manifest records it **and** it is unchanged —
  a zero-length file, an empty directory. Recorded but modified
  (bytes in the file, entries in the directory): refuse, both sides
  named — mounting would shadow what is now user content. Present but
  unrecorded: refuse the same way — camp cannot prove it is not the
  user's. When an island disappears from the source, its still-empty
  recorded scaffold is removed and struck from the manifest —
  deletion of camp's own object, which invariant 3 permits exactly —
  while a modified one is left in place, reported, and struck from
  the manifest: it has become ordinary water content, the user's.
- An islands mount's target is a mount target like any other: the gate
  exempts it, the exclude carries one line for it.

Syntax history, recorded so it is not relitigated: an
`islands_from:` key on sourceless `mount_rw` was rejected (read-only
content must not hide under an rw key; the name revealed nothing);
`mount_files_ro` / `mount_tree_ro` were rejected because they name only
the read-only half and fail the "how does it differ from `mount_ro`?"
test, while the type exists *because* of the writable floor — and
"files" misleads (islands are entries; a directory island is one
bind). `mount_storage` was the descriptive runner-up. **`mount_islands`
is final.**

## 12. Storage, work, and crash attribution

- `work/<live-hash>/` — disposable: overlay workdir, generated exclude,
  islands expansion, staging. Garbage-collectable whenever nothing is
  mounted. **Who removes it**: the privileged mode's `down`; in the
  namespace mode — which has no `down`, and where the kernel's
  `work/work` residue measurably outlives the session — the **next
  `up` sweeps stale `work/<hash>/` entries** (chmod the mode-000 dir
  first), and `doctor` lists leftovers it finds in the meantime.
  A work directory carries the same `.camp-target` marker as a storage
  directory (the live path and the config path), because after the
  lock change (§13) there is no lock file to test: the sweeper reads
  the marker, and an entry is stale when its recorded live directory
  no longer exists, or when a non-blocking `flock` on that live
  directory succeeds — nobody is composing there. An entry whose
  marker is missing or unreadable is reported, not removed (invariant
  3). The current run never sweeps its own entry: it holds that live
  lock itself, and its work directory goes at teardown.
- `storage/<live-hash>/` — persistent: island stores, writable holes,
  the worktrees. **Never removed by camp** (invariant 3): it holds
  half-done work.
- Each storage directory carries a marker file written at creation
  (`.camp-target`: the live path and the config path). Renaming the
  live directory changes the hash and orphans the old storage —
  nothing is lost, but nothing points at it either — so `doctor` lists
  storages whose recorded live composition no longer exists.
- Ownership: everything under storage belongs to the invoking user.
  In the namespace mode this is automatic; in the privileged mode it
  is automatic too, because storage and work are created by the
  unprivileged front end (§14) — the helper mounts, and creates
  nothing there. The kernel's root-owned `work/work` residue is the
  helper's to chmod away or hand over while it still has privilege
  (C6). The one path the design guarantees writable must not end up
  root-owned.

**The privileged record** — the state §1 names, now with its schema
(the review found "JSON under the state directory" unimplementable).
One file per composition at `$XDG_STATE_HOME/camp/<live-hash>.json`,
written and read **only by the unprivileged front end** (§14) — sudo
wraps the helper alone, so `$XDG_STATE_HOME` and `~` always resolve
in the invoking user's environment, and the root-home-versus-user-home
ambiguity cannot arise. Owner: the invoking user; file mode 0600,
directory 0700. Every write is temp-file-and-rename in the same
directory, file and directory fsynced — the record is what crash
recovery stands on.

Fields (`version: 1`; a reader refuses a version it does not know):
the invoking uid and gid; the canonical (`realpath`) config, env,
live, upper and workspace paths; SHA-256 digests of the config bytes
and the inventory bytes; **the complete concrete mount plan, in
order** — for each mount its kind, resolved source and target,
options and expected filesystem type, and, once mounted, its
`(st_dev, st_ino)` identity; every camp-created path (the work
directory, the scaffold entries); the phase — `mounting`, `up`,
`partial`, `down`; the tool version; created and updated timestamps.

**Write-ahead, and what each phase means** (closes the crash gap):
the record is written with phase `mounting` and the full plan
**before the helper mounts anything**; it moves to `up` only after
the post-move verification passes (§15). A failure with clean
rollback removes the record; a failed rollback, or a `down` that
could not finish, leaves `partial` with the plan intact. Recovery
therefore never needs the configuration: `down`, `status` and
`explain` read the **recorded** plan — never a config that may have
been edited while the composition was up (the rule the pre-rewrite
source already had, restored here) — and check each recorded mount
against mountinfo by path and identity. `down` unmounts whatever of
the recorded plan is still present, in reverse recorded order; drift
between the record and the current config is reported separately.
The acceptance for all of this is a kill-point matrix (§22): a kill
injected at every mount, record-write, move and unmount boundary,
after which `status` and `down` must converge from the record alone,
with the config file deleted.

## 13. The locks — one composition per upper, one per live

The kernel permits a second overlay on the same upper and the previous
binary permitted a second concurrent composition *(both measured)*;
sharing an upper corrupts data (C8). In the namespace mode the other
composition's mounts are invisible (C20), so **no mountinfo scan can
enforce this** — and a state *file* cannot either: a record can go
stale after `kill -9` and would then need exactly the `--force` the
design refuses. The guard must be something *held*, not something
*written*.

**What is locked is not the composition but the directories taking
part in it** (owner decision, 2026-08-16). A composition is simply
whoever holds that set of locks:

| directory | lock | why |
|---|---|---|
| upper (code) | exclusive | the measured C8 corruption: one upper, one composition |
| live | exclusive | the merged root, and `work/` is keyed from it; two compositions on one live path is nonsense |
| lower, registry, other mount sources | **none** | ordinary git-level parallelism; shared locks were weighed and dropped as a cost without a matching measured danger |

**The lock is the directory's own inode**: `flock(LOCK_EX|LOCK_NB)` on
a descriptor for the directory itself. No lock file exists anywhere,
and the earlier `$ENV/.camp/lock.<upper-hash>` is gone together with
the hole the review found in it — a lock *file* under `$ENV` meant two
environment directories naming the same upper locked two different
inodes and neither saw the other. An inode cannot be missed that way:
every path to the same directory is the same lock. Measured on this
machine: a second `flock` on the same directory is refused; a `flock`
through a symlink to it is refused as well (same inode); two different
directories lock independently and one process holds both at once; the
lock is released when the process dies; and locking writes nothing
into the directory — no entry appears, mtime and ctime are unchanged
*(measured)*, which is why locking the code repository does not touch
invariant 1.

- **Namespace mode**: the locks are held by **camp itself, resident as
  the namespace's init (pid 1)** — it mounts, verifies, holds both
  lock fds, execs the workload as a child, reaps, and exits when the last
  process is gone. This deliberately does *not* rely on fd inheritance
  through the workload: daemonising programs (tmux among them)
  routinely close inherited fds, and whether any given one keeps the
  lock open must never matter. A daemonised tmux server reparents to
  camp-as-init inside the namespace, so the init — and the locks —
  live exactly as long as the composition, and release themselves on
  any crash. No staleness is possible.
- **Privileged mode**: no camp process outlives `up`, so the flocks are
  held only across the `up`/`down` transitions; the steady-state guard
  is a machine-wide mountinfo scan for an overlay whose `upperdir`
  equals ours. This is also why lazy detach had to go (§14): a
  detached mount leaves the mount table while it is still alive, and
  in this mode the table is the only guard there is. Within a
  transition the front end (§14) takes the flocks before validation
  and releases them only after the final verification — generation,
  the helper and both verification passes all run under them.
- **Both locks are taken non-blocking, upper first and live second**,
  so two camps racing can only refuse each other — a deadlock is not
  reachable.
- The four pairings close *(reasoned)*: ns↔ns and ns-then-priv via the
  flocks (the session still holds them); priv↔priv via the scan;
  priv-then-ns via the scan too, because a new namespace inherits a
  copy of the host mount table.
- The refusal message names which of the two directories is locked and
  what holds it, and says the way in is *entering* the running
  composition (tmux attach — §14), not building a second one. The
  holder is found without parsing any program's output (C30):
  `/proc/locks` lists each `FLOCK` entry with its owning pid and its
  `major:minor:inode` — matching the directory's `stat` identifies the
  row, and `/proc/<pid>/cmdline` names the process *(measured)*.
- **`live` must exist before `up`**, as a real directory and not a
  symlink — a lock needs an inode to sit on. Refused otherwise (§17).
- **What the dropped shared locks cost, recorded so it is not
  rediscovered as a surprise**: the cross-role case — the same
  directory used as a lower in one composition and as an upper in
  another — no longer collides in the kernel by itself. Shared locks
  on the lower would have caught it for free; the owner weighed that
  against the simpler two-lock model and chose the simpler one,
  because this is an exotic configuration mistake, not the measured
  corruption class. It stays a *reasoned* gap, not a measured one.
- Several *different* compositions (different uppers, different lives)
  remain fine.

## 14. Modes

**The namespace ("rootless") mode is primary** (owner decision,
2026-08-15). The privileged mode stays documented as the fallback for
sessions where a program started *outside* must see the tree — on this
machine that is the GUI editor (Sublime Text was running, VS Code
installed *(measured)*), and nothing else found: the docker gate works
inside (C19 correction), and dockerd never needs to see live (§4).
Known trap, marked *(reasoned, unmeasured)*: a single-instance GUI
editor launched from inside hands the path to its outside instance,
which opens the raw directory — "start it from inside" does not work
for those.

### Namespace mode

- **The session's shape — the supervisor contract** (owner round,
  2026-08-16; the review found the promised behaviour — locks held by
  "camp as pid 1", a daemonised tmux reparenting to it — unbuildable
  from the text; here is the process tree and its state machine).
  `camp run -- <cmd>` is two processes:

  1. **The launcher** — the command the user typed — performs §7
     steps 1–4 (locks, static validation, generation, output
     validation), all as the user, nothing privileged existing yet;
     then clones **the init** with
     `CLONE_NEWUSER | CLONE_NEWNS | CLONE_NEWPID` and a pipe between
     them, and closes its own copies of the lock fds once the init
     confirms — a flock lives on the open file description, so the
     inherited fds carry it (§4).
  2. **The init** — camp resident as pid 1 of the PID namespace —
     writes the uid/gid maps (route A: own uid to itself, `setgroups`
     denied, own gid to itself), mounts a namespace-local procfs over
     `/proc` (pids inside are namespace pids; holder-naming and `ps`
     need the local view), performs the mount sequence of §7, runs
     §15's verification, **drops the ambient capability set**, and
     only then forks the workload.
  3. **The handshake**: the init reports over the pipe exactly once —
     "up, workload started", or a §15/§17 refusal, which the launcher
     prints. The launcher then waits not for the init but for the
     **workload's exit status**, which the init sends over the same
     pipe when the workload child exits; the launcher exits with that
     status (128+signal if signalled). This is what makes
     `camp run -- tmux new-session -d` return at once — the tmux
     client exits immediately, its server reparents to the init —
     while the init stays resident, holding the locks.
  4. **The init's remaining life**: it reaps everything reparented to
     it, forwards `SIGTERM` and `SIGHUP` to the workload's process
     group, and takes no terminal input — the workload's group is the
     foreground group on the tty, and the init ignores `SIGINT`,
     `SIGQUIT`, `SIGTTIN`, `SIGTTOU`, so a Ctrl-C reaches the
     workload and never kills the supervisor holding the locks
     mid-session. When the last other process is gone it runs the
     end-of-session report (below) and exits. Exit = teardown by the
     kernel (C21); there is no `down`, no state record, nothing to
     clean *(measured)*. A `kill -9` of the init loses the report and
     nothing else — the kernel still tears down to zero.

- **The end-of-session report** (owner decision, 2026-08-16; closes
  the review's finding that §10's and §20's promised detections had
  no delivery path in the primary mode). Before exiting, the init —
  the one process positioned to look while the composition still
  exists — runs the same read-only pre-down pass as the privileged
  `down` (§16): worktrees to repair, the gate re-run, the inventory
  comparison, the untracked and index scans. The result is written to
  `$ENV/.camp/reports/<live-hash>-<unix-time>` (temp-file-and-rename)
  and printed to stderr when one is still attached — a detached tmux
  session's terminal is long gone, which is exactly why the file
  exists. **The next camp command run in that environment prints any
  unseen report once and renames it `.seen`**; `doctor` lists both.
  A report is output, not authority — nothing reads it back as state,
  so "namespace mode leaves no state record" stands: what it leaves
  is the message the mode otherwise had no way to deliver.
- **Multi-terminal entry: the tmux pattern**. `camp run -- tmux
  new-session -d` returns immediately while the tmux server stays
  inside and keeps the namespace alive; from any outside terminal the
  tmux client reaches it — the client is only a pipe; windows are
  children of the server, hence inside. Measured: `tmux ls` from
  outside saw the session; a `send-keys` command from outside ran
  inside and saw the composed tree (workspace content, read-only)
  while `ls` outside showed the live directory empty at the same
  moment; `tmux kill-server` tore everything down to zero mounts.
  `attach` itself was not run (no tty in the lab) and rests on the
  same client-as-pipe mechanism *(reasoned; §23)*. camp does not
  depend on tmux; the pattern is documented (and may later get a
  convenience wrapper). The lock is held by camp-as-init per §13,
  independent of tmux's fd habits. A `setns`-based `camp shell --join`
  (podman's pause-process model) is deferred — tmux covers the
  workflow.
- **User identity is restored inside** (owner requirement: tools check
  for root — Claude Code refuses its permission-skip flag as apparent
  root, npm changes behaviour — and the workload must see the real
  user). Route A (primary, needs a spike — §23): map the caller's own
  uid to itself (not to 0), keep `CAP_SYS_ADMIN` through **ambient
  capabilities** (Go: `SysProcAttr.AmbientCaps`) exactly until the
  mounts and verification are done, then drop the ambient set and exec
  the workload — inside, `id` shows the real user, files created are
  the user's, and no capability remains. Overlayfs keeps working after
  the drop because it records the mounter's credentials at mount time
  *(reasoned — part of the spike)*. The gid side is a named spike
  question: writing an own-gid `gid_map` without `newgidmap` requires
  denying `setgroups` first (kernel rule) — acceptable, because the
  C19-corrected behaviour means the *retained* supplementary groups
  keep granting access to host files regardless; the spike verifies
  exactly that under the own-uid/own-gid mapping. Route B (fallback,
  and the road
  rootless-podman-inside would need anyway): install `uidmap`
  (`newuidmap`/`newgidmap`; subuid/subgid already configured), map
  root→subuid and user→user, podman-style. The C19-corrected group
  behaviour is unaffected by either route (real credentials
  unchanged), and verification runs before the drop. **Selection**
  (owner round, 2026-08-16): route A is the only automatic route —
  route B never engages by silent fallback, because the two routes
  present different uid worlds to the workload; it is chosen
  explicitly in the configuration (`identity: uidmap`). Route B's
  maps are podman's `keep-id` shape: the caller's uid and gid map to
  themselves, 0 and the rest of the range come from subuid/subgid via
  `newuidmap`/`newgidmap`, and `setgroups` stays permitted on this
  route.
- **Locked flags** (C34): the remount step ORs the source mount's
  locked flags (`nosuid`, `nodev`, `noexec`, atime flags) into every
  read-only remount. With that in place a nosuid filesystem is
  **supported, not refused** — the measured `/tmp` failure was the
  missing-flags bug, not a property of tmpfs — and the acceptance
  test asserts the fix, not the bug (§22): a composition on `/tmp`
  must pass, and only a deliberately flag-omitting remount (a
  test-only path) may reproduce the `EPERM`. `doctor` still *reports*
  inherited restrictions — a `noexec` environment cannot run scripts
  from the tree — as information, never as refusal.
- AppArmor: the shipped profile attaches to `/usr/local/bin/camp`,
  grants `userns,` and nothing else (the binary stays unconfined — a
  dev tool that runs your editor and shell cannot be meaningfully
  confined, and a pretend-confinement is worse than none). Install/
  uninstall instructions live in the profile's comment. The sysctl
  protection stays on machine-wide.

### Privileged mode

- **The shape: an unprivileged front end and one narrow privileged
  helper** (owner decision, 2026-08-16; the review showed the literal
  `sudo camp up` model contradicting §19 — a process that is root
  from its first instruction has no "before sudo" in which `prepare`
  could run as the user). `camp up` runs as the invoking user from
  start to finish: it locks, validates, gates, generates, validates
  the generated output, writes the `mounting` record (§12) — and then
  invokes the helper as `sudo camp helper-mount`, an internal
  subcommand that does exactly one thing: execute the validated
  concrete plan handed to it on stdin (never argv — `/proc` exposes
  argv machine-wide). The helper parses the plan, resolves and
  mounts, reports per-mount results on stdout, and exits; it reads no
  configuration, runs no generator, and consults no state. `camp
  down` is the same shape: the front end reads the record and drives
  `sudo camp helper-unmount` over the recorded plan. sudo is
  exercised exactly once per command, prompts on the real terminal
  (C33), and wraps code a reviewer can hold in one hand. Running
  `sudo camp up` directly is **refused**, with the message to run
  `camp up` unprivileged — under root the generators would run as
  root, which §19 forbids, and everything camp creates would be
  root-owned. The tree is visible machine-wide; multi-terminal is
  trivial; GUI editors work.
- **The helper trusts nothing it was handed** (the validation-to-use
  race the review named: the user owns every parent directory of the
  operands, and a component can become a symlink between the front
  end's check and the helper's `mount(2)`). The helper re-resolves
  every operand itself, descriptor-relative, with
  `openat2(RESOLVE_NO_SYMLINKS | RESOLVE_BENEATH)` from the recorded
  bases, verifies each endpoint's `(st_dev, st_ino)` against the
  plan, and mounts **by descriptor** — the new mount API
  (`open_tree`/`move_mount`) where the kernel has it,
  `/proc/self/fd/N` paths otherwise — so the object checked is the
  object mounted. Any mismatch fails closed: nothing mounted, or
  rollback of what is. The upper and live flocks are held by the
  front end across the whole sequence, so the plan cannot go stale
  against a concurrent camp (§13).
- Everything camp creates is chowned to `SUDO_UID`/`SUDO_GID` (§12);
  nothing is ever run as root *inside* the tree by camp.
- **The workspace is read-only for the whole machine while the
  composition is up** (owner decision, 2026-08-16). One mount table
  means there is no inside and no outside: either the workspace
  self-bind exists and the owner's editor also gets `EROFS`, or it
  does not exist and a process in the tree can write the workspace by
  absolute path. Both promises cannot hold at once, and the protection
  wins. Why: a write landing in the wrong repository is the design's
  central enemy, `EROFS` is a loud failure while a stray write is a
  silent one, and this is the exceptional mode anyway — normal work
  runs in the namespace mode, where both promises do hold. Invariant 7
  states the narrowed promise; `up` prints it as a line of its own, so
  nobody meets it as a surprise: *"privileged mode:
  `~/dev/diet-coach-workspace` is read-only for the whole machine
  until `camp down`."*
- **Staging is mandatory here** *(measured feasible)*: the whole
  composition is built at `work/<hash>/staging` (mode 0700), verified
  there, `mount --move`d onto the empty live directory, and verified
  again at the live path (§7). It used to be labelled optional
  hardening, which made a correctness race optional: this mode exists
  *for* programs already running outside, and those are precisely the
  ones that would meet a half-built tree — writing into a
  lower-provided path before its protection is up, or running git
  against a briefly merged `.git`. Worthless in the namespace mode
  (C20 already hides intermediate states).
- `down` attempts teardown in reverse; what is held (C16) it reports
  with holders named from `/proc` (C30) — a session standing in the
  tree cannot unmount the tree under itself, and the message says to
  leave the directory. **There is no lazy unmount**: `--allow-detach`
  is deleted and `MNT_DETACH` appears nowhere in camp (owner decision,
  2026-08-16). It was a `--force` wearing another name. Measured: after
  a lazy detach the target is gone from mountinfo, a shell whose cwd is
  inside keeps writing through it, and a second overlay mounts on the
  same upper and writes — the exact C8 state the lock exists to
  prevent, re-entered by the tool's own switch, and in this mode the
  mountinfo scan is the only steady-state guard. So a mount that
  cannot be removed is an **error**: named, reported, non-zero exit
  (§16), never a pretend success. `status` distinguishes *up*,
  *partly up*, *down*.

## 15. Verification — what `up` checks, every time

Two measured facts shape it: mountinfo presence does not prove
reachability (a shadowed mount stays listed), and path-based syscalls
see exactly what a process would. So the **path is the authority;
mountinfo is the cross-check**.

Before mounting (static, §17 lists the refusals): the `steps:`
sequence is first resolved into one concrete plan and walked on a
virtual tree in its own order, so every check below sees each target
in the state its own step will meet — the path language of §6 over
every field; order validation (nested **and equal** targets); targets
inside the live tree; targets exist with matching types; target not
on code content; sources exist and are not symlinks; live exists as a
real, non-symlink, **empty** directory (an overlay over user content
would hide it for the whole session, and only `down` would name it);
workdir on the upper's filesystem; locked-flags feasibility; the two
locks; the gate; the inventory; no active record for this live path
(§12); **and a residue scan** — refuse if mounts already exist under
the live prefix or on the workspace path (crash or partial-teardown
leftovers; the message points at `status` and `down`, and in the
privileged mode at the record and the mountinfo inventory that
identify whose they are). Between generation and mounting, the
concrete plan is validated once more (§7 step 4, §19) — the static
pass cannot check output that does not yet exist.

After mounting, before declaring up, per planned mount:

1. **Identity & reachability, by path**: binds — `stat(target)` equals
   `stat(source)` in `(st_dev, st_ino)`; the overlay — `statfs` magic
   at live, and the mountinfo entry at live carries
   `lowerdir`/`upperdir`/`workdir` equal to the plan, compared **per
   option** (the kernel appends its own). This single check also
   catches every ordering or shadowing mistake, because a covered
   mount fails it *(measured)*.
2. **Writability polarity, by path**: `statvfs` `ST_RDONLY` matches
   the plan for every target — this is what catches a one-step bind at
   the syscall level (C11): inspect the result, never trust the call.
3. **Artefact outputs**, one check per artefact the configuration
   actually generates: for the exclude, read `live/.git/info/exclude`
   and require **byte equality with the validated payload** (§10's
   assembly, marker included) — a marker-only match would accept a
   payload whose repository half was dropped. A composition without a
   generation step is not checked for something it never promised.
4. **Propagation**: every camp mount private (empty optional fields in
   its mountinfo entry).
5. **Completeness**: the set of mounts under the live prefix **plus
   the workspace self-bind at the workspace path** equals the plan
   exactly — fewer is a failed mount, more is residue or interference.
6. **Storage ownership**: storage paths stat as the invoking user.

In the privileged mode this pass runs **twice**: once on the staging
tree, and again at the live path after `mount --move` (§7). The
second pass is the one that decides — the move is the moment the tree
becomes machine-visible, and only a path-based check at the final
location can prove what an outside process now sees. Checks 1 and 5
are expressed against the staging prefix in the first pass and the
live prefix in the second.

On any failure: unmount in reverse, name the failed check, exit
non-zero (invariant 4 — `up` refuses; and here is where). If reverse
teardown itself fails, report *partly up* with holders named — and,
per §14, never resolve it with a lazy detach.

`status` **is this same verification pass, run read-only**, reporting
instead of refusing — one code path, two exits. In the namespace mode
a composition is invisible from outside (C20) and leaves no record, so
`status` answers from inside (or via the tmux window).

Placement of everything else:

- **`doctor`**: per-mode capability (userns permitted / sudo present —
  exists today and reads correctly); locale; environment on a nosuid
  filesystem (C34); a throwaway-overlay probe per mode exercising
  whiteout and opaque; copy-up forensics in **both** xattr namespaces
  (`user.overlay.*` measured, `trusted.*` at first privileged up);
  storage orphans via the marker file; prunable worktrees with repair
  commands.
- **Nowhere**: the thirteen-write attack battery (a kernel property,
  measured once in the POC and re-measured in the lab — per-run, check
  2 asserts the same thing in one syscall per mount); byte-manifest
  and strace harnesses (the write-nothing invariant is a source
  property, §22 — "read the code, once, and keep a guard on it").

## 16. Reporting: `down`, `doctor`, `plan`, `explain`

`down` (always attempts, never deletes, and never walls the user in —
but it may end in an error, see 3):

1. Worktrees registered under the live path — for each, the exact
   `git -C <code> worktree repair <storage-path>` and the three-month
   `gc.worktreePruneExpire` window, because that is the failure that
   happens while nobody is looking. A repaired worktree stops dying
   at every `down`; `explain` steers users toward repairing.
2. Drift and suspected leaks, while fresh — four scans, all
   read-only: the gate's comparison re-run; **the inventory
   comparison re-run** (a lower root entry born mid-session is named
   the same day, not at the next `up`); the code repository's
   untracked paths whose first component matches a lower root name or
   a mount target, reported as suspected copy-up residue; **and the
   index scan** — `git ls-files --stage` — reporting any indexed path
   under a lower root name or mount target that was not a code path
   at `up`. The index scan is the one that sees `git add -f`: a
   force-add through the live tree leaves an *indexed* path with
   **no file in the raw working tree** *(measured — constructed with
   `update-index --cacheinfo`; `ls-files --others` shows nothing,
   `ls-files --stage` shows the path)*, so an untracked-file scan is
   structurally blind to exactly the leak §20 promises to detect.
   All git reads with `--no-optional-locks`.
3. Partial teardown — an **error**, not a footnote. camp never detaches
   a busy mount to make the ending look clean (§14), so a mount it
   could not remove is still there and `down` exits non-zero. The
   message carries: the mount path; the holder — pid, command, and
   what ties it there (cwd, an open file, its root), read from `/proc`
   (C30); the one move that helps, in the user's hands (*leave that
   directory / close that file, then run `camp down` again*); and the
   state camp is leaving behind — *partly up*, with everything still
   mounted listed by path. `status` prints that same list at any time,
   so the situation is never guessed at.
4. Own residue: the kernel `work/` dir removed (chmod first); only
   what cannot be removed is reported.

`doctor`: §15's list. `plan`: the full derived plan — every mount with
the reason it exists, the islands expansion, the exact commands or
syscalls, and the problems that would stop `up` — nothing executed.
`plan` output uses labels, not `lowerdir` positional syntax, because
left-to-right precedence is exactly what people read backwards.
`explain`: describes the tree to whoever stands in it (what is
read-only and why, where the backing store is, what can never enter a
commit, how worktrees behave), generated from the live configuration
so it cannot go stale.

`list` reads every record in the state directory (§12) and prints one
line per composition — live path, phase, age — newest first; a record
it cannot parse is printed as *corrupt*, with its path, and never
silently skipped or deleted. `forget` drops exactly one record and
deletes nothing else — and **refuses while any mount of the recorded
plan is still present** (checked against mountinfo first): forgetting
an `up` or `partial` composition would discard the only authoritative
teardown list, which is `down`'s to consume, not `forget`'s to lose.
In the namespace mode both commands have nothing to read (no record
exists); any unseen end-of-session reports (§14) are printed — once,
then marked `.seen` — by whichever camp command runs next in that
environment. `camp init` survives as the bootstrap: it writes a
commented `$ENV/.camp/config.yml` skeleton in §6's shape and refuses
to overwrite an existing one.

## 17. The refusal list

Refused before mounting, each with a reason and, where the user must
act, the exact commands (§21):

- an overlap not in `allow_overlap` (the gate, §9);
- a `steps:` item that is neither a bare kind nor a single-key
  mapping, or that names a kind camp does not know (§6);
- a declared order in which an earlier mount's target lies inside a
  later mount's target;
- **two mounts declaring the same target**;
- **mounts already present** under the live prefix or on the workspace
  path before `up` (crash residue — §15's residue scan);
- a mount target outside the live tree;
- **a mount target with code-tracked content at or under it** — no
  mount, island or directory may cover anything the code repository
  tracks (C26). Checked as: no code-tracked file resolves at-or-under
  the target. This rule needs no exception list: `.git` and
  `.git/info/exclude` pass automatically because git tracks nothing
  under `.git`;
- a missing target (C10) — including the exclude's, with the two
  commands printed; the general remedy named: commit an empty
  mount-point with a placeholder file in the workspace (C23);
- a `mount_rw` whose source lies under a lower (invariant 2);
- source/target type mismatch (C13);
- a source or root entry that is a symlink (C12);
- a name containing a newline (inexpressible in an exclude, ambiguous
  in a report);
- a repository nested inside another, or live inside a repository;
- a live path that does not exist, is not a directory, is a symlink
  (§13 — the lock needs its inode), or **is not empty** — an overlay
  over user content would hide it for the whole session and report it
  only at `down`;
- a second composition on the same upper **or on the same live
  directory** — the two locks and the scan (§13);
- inventory violations: a new lower root entry (block), a type change
  including became-a-symlink (block) — §18;
- no inventory yet: refuse and name `camp accept` (an `up` that
  generated it would swallow the signal);
- a workdir on a different filesystem than the upper (C6), or locked
  flags the mode cannot replicate (C34);
- a configured generation step that would run with privileges (§19);
- `overlayfs.lower` with more than one entry (deferred feature, clear
  message);
- a path field violating §6's path language: an absolute source or
  target, an empty, `.` or `..` component, a repository name
  containing `/`, or two repositories resolving to the same
  directory;
- a `generate:` step alongside `git_exclude`, or more than one
  generation step (§19 — there is one exclude payload);
- generator output failing the hostile-data checks (§19): an islands
  entry that is not exactly one component, a type disagreeing with
  `lstat`, an entry the source does not contribute, a duplicate, an
  unsupported type, or an exclude payload not byte-identical to §10's
  assembly;
- a needed island attachment point that the scaffold manifest does
  not record, or records but finds modified (§11);
- an existing record for this live path in phase `mounting`, `up` or
  `partial` (§12) — `status` and `down` first.

After mounting: any §15 failure refuses the whole composition.

## 18. The inventory — `camp accept`

A generated snapshot of the root entries of the lower and the upper,
**with type** — file, directory, or symlink and its target. Stored at
**`$ENV/.camp/inventory`**, beside the configuration (intent vs
snapshot); one record per line, **tab-separated fields** —
`<side> TAB <type> TAB <name>`, a fourth field carrying a symlink's
target — byte-sorted over the decoded name bytes (C32), so its diff
is reviewable. Generated **only by the explicit command**
`camp accept`, never by `up` — silent refresh would swallow the very
signal the file exists to raise. At `up`: a new lower root entry
**blocks**; a type change **blocks**; a disappeared entry or an
upper-side change **warns**. At `down`, and at a namespace session's
end, the comparison re-runs and **reports**, never blocks (§16). It
shares one enumeration with the exclude, so the two cannot drift
apart.

**Encoding — one scheme for every record camp writes** (owner
decision, 2026-08-16; the review showed the earlier space-and-arrow
line was not injective — names may legally contain spaces, tabs,
` -> ` and non-UTF-8 bytes, so two different filesystem states could
serialise identically or fail to parse). Fields are raw bytes under
reversible C-style escaping: `\\` for a backslash, `\t` for TAB,
`\n` for LF, `\r` for CR, `\xHH` for every other byte below 0x20 and
for 0x7F; all remaining bytes — non-UTF-8 included — pass through
verbatim. A decoded field can hold anything a Linux name can; an
encoded field can never hold a raw TAB or newline, so the framing is
unambiguous. A line that does not decode (a `\` opening no valid
escape) refuses the whole file. Sorting compares decoded bytes. The
same encoding governs the generator lists (§19) and the scaffold
manifest (§11); golden tests cover names with spaces, tabs, arrows,
backslashes, control bytes and invalid UTF-8. A *name* containing a
newline is still refused at `up` (§17 — inexpressible in an
exclude); the encoding is what lets the snapshot and report layer
speak truthfully about everything else — a symlink *target*
containing a newline, for instance, is legal and now representable.

## 19. Pluggable generation — git leaves the core

The core needs git knowledge in exactly two artefacts: the exclude
payload and the islands expansion. Both are produced by **one
generation step** with a shipped default (`git_exclude`), so
configurable never means obligatory, and the core itself contains no
git.

- **Where the step sits.** The generation step is an item of `steps:`
  (§6): its **generation** runs in the `prepare` phase, before
  anything is mounted, always as the invoking user; its **mount** —
  the exclude bind — happens at the step's own position in the
  sequence. So the configuration decides *where in the tree order*
  the artefact lands, while the privilege rule stays fixed. Islands
  expansions are generated in the same pass, one list per
  `mount_islands` entry, and consumed by those entries wherever they
  sit. A configuration has **at most one** generation step — the
  shipped `git_exclude` or the custom `- generate: { command: [...] }`
  form; both at once is refused (§17). A configuration with neither
  has no exclude — `plan` says so plainly (§10) — and islands fall
  back to the raw listing (§11).
- **Phases — narrowed** (owner round, 2026-08-16). `prepare` is the
  **only** phase at which configured code runs. The other lifecycle
  points — after mounting, before teardown, after teardown — belong
  to camp's own frame (§15's verification, §16's reports); they are
  not extension points, and a general hook mechanism is explicitly
  deferred (§23). This closes the review's elevation questions for
  later phases by making them unreachable: no configured step ever
  exists to run with, or drop, privilege.
- **Privilege**: configured code runs as the invoking user, always —
  and in the privileged mode that is now true by construction, not by
  a drop protocol: `prepare` runs in the unprivileged front end
  (§14), and no privileged camp process ever executes a configured
  command. Whoever can edit the config must never gain root through
  it.
- **Process contract** (owner round, 2026-08-16). `command` is an
  argv vector — `execve` directly, never a shell: no word-splitting,
  no expansion. cwd: `work/<hash>/gen/`, so a naive generator's
  relative writes land in camp's scratch, never in a repository.
  Environment: the invoking user's, plus `CAMP_GEN_IN`,
  `CAMP_GEN_OUT`, `CAMP_ENV`, `CAMP_LIVE` (absolute paths). stdin is
  `/dev/null`; stdout and stderr pass through to the terminal. No
  default timeout — camp is interactively driven (C33) — but an
  optional `timeout: <seconds>` field kills the step's process group
  on expiry. A nonzero exit, a timeout or a fatal signal fails the
  step, and a failed `prepare` aborts `up` before anything is
  mounted.
- **Contract on disk**: camp materialises inputs under
  `work/<hash>/gen/in/` — `lower-root.list` (`<type> TAB <name>`
  records, §18's encoding), `mount-targets.list` and
  `allow-overlap.list` (one encoded field per line),
  `upper-exclude.current` (the file's raw bytes), and
  `islands/<target>.source` (the source path) per islands mount. The
  step writes `work/<hash>/gen/out/exclude` (the complete payload to
  mount, marker line included, assembled per §10's byte rule) and
  `work/<hash>/gen/out/islands/<target>.list` (`<type> TAB <entry>`
  records). All lists byte-sorted over decoded bytes (C32). The
  default implementation reads git via `ls-files`/`ls-tree` — reads
  only (invariant 1).
- **The output is hostile data** (the review's finding: a user-level
  generator could otherwise steer a privileged mount). After
  generation, before anything is mounted, camp validates the output
  against the *final* plan (§7 step 4) and refuses on any of: a line
  that does not decode; an islands entry that is not **exactly one
  path component** (nonempty, no separator, not `.` or `..`); a
  declared type that disagrees with `lstat` of the source entry —
  resolved descriptor-relative, no symlink traversal, contained in
  the declared island source; an entry the source does not actually
  contribute; an entry type other than regular file or directory; a
  duplicate entry; an island-over-water collision (§11, checked
  before any placeholder is created); an exclude payload that is not
  byte-identical to §10's assembly — the repository's own bytes,
  unchanged and complete, plus the marker block. The equal/nested
  target and tracked-code checks of §17 re-run over the concrete
  expanded mounts. Static validation before generation checks
  everything configured; this pass checks everything generated;
  nothing is mounted on the strength of either alone.

## 20. The git leak — what is accepted, and why

There is no camp pre-commit hook: both real repositories set
`core.hooksPath`, so a hook installed in `.git/hooks/` would never run
— a protection that looks installed and does nothing is worse than
none — and installing one is git manipulation, outside the tool's
remit (owner ruling; the hooks-bind workaround is dead: under the
privileged mode it would change which hooks run machine-wide).

Stated plainly: **nothing prevents `git add -f`** — it *reads* the
workspace file through the merged tree and stages its bytes; the
read-only binds stop writes, not reads. What exists is detection: the
gate at the next `up`, and — at `down`, and at a namespace session's
end (§14) — the drift report **with the index scan**, `git ls-files
--stage`, because a force-add leaves an indexed path with no raw
working-tree file, which no untracked scan can see (§16). The framing that
makes detection valuable: **the point of no return for a shared
history is `push`, not `commit`** — a leak caught at `down` is
usually still free (`git reset` in `code/`, composition down, user's
hand). The last automated gate before that point — the repository's
own pre-push hook — measurably runs in both modes. Anything that
reaches a pushed history cannot be fixed retroactively, only
rewritten, which is outside the tool.

## 21. Message quality

Every refusal and report speaks to someone who has not read this
document: name the path; say what is true on each side; say which side
matters; give the exact repair commands; say whose move it is (the
user acts, camp checks). Command output that names processes uses
`/proc`, never parsed program output (C29/C30); comparisons sort by
bytes (C32).

## 22. Implementation notes

**The backend is settled: kernel mounts.** Writing the composition as
a FUSE filesystem in Go was considered and **rejected** — do not
reopen it. Grounds: blast radius (a bad mount composition produces a
wrong *view*, visible and reversible; a bad filesystem produces wrong
*data on disk*, and git is the least forgiving client — mmap'd
packfiles, atomic-rename index, fsync ordering); the characteristic
failure (a deadlocked FUSE daemon hangs the mountpoint and blocks the
editor, the language server and every agent uninterruptibly, which a
kernel mount cannot do); and the obligation (being a filesystem is
forever, for a program whose value is the configuration model).

**Starting point.** The repository (module `github.com/dlaszlo/camp`,
binary `camp`, renamed throughout; tests green; **the rename is
uncommitted in the working tree** — the first `git status` will show
it, and committing it is the owner's call) implements the **old**
design: `code:`/`workspace:`/`live:` config, contributed-name
binds with covers, and — critically — `gitwire.InstallExclude`, which
`MkdirAll`s and `WriteFile`s into `code/.git/info/exclude`. **That
function violates invariant 1 and is deleted**, replaced by the
generate-and-mount path. `internal/mountx`, `preflight`, `state`,
`holders`, `testenv` are reusable; `config`, `plan`, `composition`,
`gitwire` are rebuilt to this spec. The apparmor profile is in
`packaging/apparmor/camp`. Pending owner-side steps: `sudo install`
of the renamed binary and profile; the GitHub repository when
publishing; optionally renaming the working directory.

**The write-sites guard** (the invariant is a source property): all
filesystem writes live in one package; no write target may be derived
from a repository path; a reviewer checks the write sites (18 call
sites across 7 files at the time of the rename), not a harness.

**Build order**, each stage with its acceptance check:

1. **Config + gate** — new language (§6), validations (§17), gate
   (§9). Accept: unit tests on scratch repos; the real pair's
   pre-migration state blocks with the documented message; **a
   `steps:` order that nests a later target inside an earlier one is
   refused while the reverse order passes** (a total order is only
   worth having if it is checked); **a config omitting the `.git`
   mount is stopped by the gate, not by a git-specific rule.**
   Path-language refusals each fire with their §17 message (absolute
   target, `..` component, duplicate repositories); a config
   declaring two generation steps is refused.
2. **The locks** (§13). Accept: the measured double-run scenario now
   refuses; **a second config in a different environment directory,
   naming the same upper by a different path, also refuses** — the
   identity is the inode, not a lock file; a second composition on the
   same live path refuses; the locks are released by killing the
   session; **and they survive a daemonised tmux server** (start
   `tmux new -d` inside, let the launching command exit, verify a
   second `up` still refuses — this is the camp-as-init property).
3. **Identity spike** (§14 route A). Accept: inside — `id` shows the
   real user; mounts succeeded; after the drop — no capability
   remains, a write to an island is `EROFS`, a write through the
   overlay lands in the upper; **and the retained supplementary
   groups still grant host-file access under the own-uid/own-gid
   mapping** (the docker-socket or syslog read repeats).
4. **Composition + verification** (§7, §15) with locked-flags fix
   (C34). Accept: the POC-shaped scratch composition passes all
   checks; a deliberately shadowed island is caught; **a composition
   on `/tmp` (nosuid tmpfs) passes with the locked flags replicated,
   and only a deliberately flag-omitting remount reproduces the
   `EPERM`** — the test asserts the fix, not the old bug; ext4 passes
   too; a nonempty live directory refuses; **in the privileged mode
   an outside `ls` of the live directory sees nothing until the move
   completes, and an outside write to the workspace gets `EROFS`
   while the composition is up** (§14's two named machine-wide
   effects, asserted rather than assumed); sudo is exercised exactly
   once (wrapping the helper), and a direct `sudo camp up` is
   refused; **a source component swapped for a symlink between
   validation and mount is refused by the helper's descriptor
   resolution** (§14); **the kill-point matrix**: a kill injected at
   every mount, record-write, move and unmount boundary, after which
   `status` and `down` converge from the record alone with the config
   file deleted (§12).
5. **Generation** (§10, §11, §19): exclude + islands via the default
   git step. Accept: the measured scoping test (live sees generated,
   code sees own); coarse lines anchored; islands expand per plan;
   hostile-output tests — a generator emitting `..`, a separator, a
   wrong type or an exclude payload missing the repository's own
   lines is refused (§19); **the repeated session** — `up`/`down`/
   `up` twice with islands, the scaffold manifest accepting its own
   attachment points, no collision refusal (§11); golden encoding
   tests over hostile names (§18).
6. **down/doctor/status/plan/explain** (§16). Accept: worktree repair
   lines correct against a live-created worktree; orphan storage
   listed; **a `down` run from a shell standing inside live fails
   loudly, names that shell, leaves the composition mounted and exits
   non-zero** — and `grep MNT_DETACH` over the source finds nothing;
   the index scan catches a constructed `git add -f` leak (§16); a
   namespace session's end writes the report file and the next camp
   command prints it once, then marks it `.seen` (§14); `forget`
   refuses on a `partial` record and `list` shows phases (§12, §16);
   the supervisor traces — a foreground command propagating its exit
   status, `tmux new-session -d` returning while the init holds the
   locks, Ctrl-C reaching the workload and not the init (§14).
7. **Docs**: README and `explain` text to the new design; tmux pattern
   documented. (The historical docs — `model.md`,
   `redesign-2026-08-15.md`, the reviews — are records and are not
   rewritten; this spec supersedes them.)

Testing note: until the stage-4 locked-flags fix lands, scratch
compositions run under `~/overlayfs` or any ext4 path — the installed
old binary still fails on `/tmp` (C34); from stage 4 on, the `/tmp`
case must pass and is itself part of the acceptance. Mind the AppArmor attachment (C17):
the userns permission is granted **to the installed binary's path**.
Today that is `/usr/local/bin/ply` under the *ply* profile; a freshly
built `./camp` in the checkout **cannot create a namespace** until the
owner installs the camp binary and profile (`sudo install … camp`,
`apparmor_parser -r /etc/apparmor.d/camp`). Until then, lab tests can
use the installed old binary purely as a namespace entry vehicle (as
this session's measurements did), or wait for the install.

## 23. Deferred, and unmeasured things the build must settle

- **Identity route A spike** (§14): ambient-caps mounting with an
  own-uid mapping, and that overlayfs operations survive the drop.
  Only camp can create a namespace on this machine, so this is a
  build-time measurement.
- **`trusted.overlay.*` forensics** at the first privileged `up`.
- **`--no-optional-locks`** actually preventing index writes: verify
  once when the `down` report is built.
- **The single-instance editor handoff** (one manual check on the
  owner's GUI).
- **`tmux attach` from an outside terminal** — one manual check;
  `ls`/`send-keys` are measured, attach is reasoned (§14).
- **Whether the agent stack runs under the restored identity** — with
  route A this becomes moot (no apparent root), which is part of why
  route A is primary.
- **Several lowers**: tool-merged `merged_lower`, shadowing reported
  by name (file shadowing between read-only layers is silent whoever
  merges them). Not before a real second workspace exists.
- **`setns` join** (`camp shell --join`): only if the tmux pattern
  proves insufficient.
- **Hooks at the `mounted`/`pre-down`/`post-down` lifecycle points**
  (§19): configured code runs only at `prepare` today; a general
  hook mechanism waits for a real use.
- **`resolve`** (per-path "what happens on write here" from the old
  model) — nice-to-have; `explain` + this spec cover the need for now.
- **`.mcp.json` placement** in the migration (§5.2).
- The **"ply" unit noun** in older docs and the Go type name: owner's
  call, cosmetic, any time.

**The review's findings are closed.** The 2026-08-15 night decisions
closed WRONG-1, WRONG-2, UNDERSPECIFIED-1 and part of GAP-3; the
2026-08-16 second pass wrote in the remaining fourteen, following the
review's own fix paragraphs, with three further owner decisions — the
front-end-plus-helper elevation model (§14), C-style escaped record
formats (§18), and the persisted namespace session report (§14, §16).
Nothing from `docs/review-of-spec.md` remains unanswered in this
file.

**Named build-blocking measurements** — the register the review asked
§23 to be. Each carries its expected outcome; an item leaves this
list only when its mechanism *and* acceptance test are both written,
and each already appears in §22's stages:

- **The full privileged lifecycle, end to end** (not only `trusted.*`
  xattrs): staging invisibility until the move, machine-wide
  workspace `EROFS` for an outside process, private propagation,
  invoking-user ownership of storage, both verification passes —
  §22 stage 4.
- **Kill-point recovery**: `status`/`down` converge from the record
  alone at every injected crash boundary — §22 stage 4, §12.
- **The repeated session**: `up`/`down`/`up` with islands; the
  scaffold manifest accepts its own attachment points — §22 stage 5,
  §11.
- **The rename/symlink race**: a component swapped after validation
  is refused by the helper's descriptor-relative resolution — §22
  stage 4, §14.
- **The supervisor traces**: a foreground command propagating its
  exit status; `tmux new-session -d` returning while the init holds
  the locks; Ctrl-C reaching the workload, not the init; the report
  file written when the namespace empties — §22 stages 2 and 6, §14.
- **Locked flags**: `/tmp` passes with flags replicated; the
  omitted-flags `EPERM` reproduces only in the deliberate test — §22
  stage 4, C34.

## 24. References (background, not required reading)

- `docs/constraints.md` — the original constraint set (C1–C33).
- `docs/review-2026-08-15.md` — the second-pass review: the
  measurements behind §4's corrections and §15's design.
- `docs/review-of-spec.md` — the review of *this* file (17 findings);
  all seventeen are closed by the two 2026-08-16 rounds (§23).
- `docs/redesign-2026-08-15.md`, `docs/model.md` — historical;
  superseded where they differ.
- `~/overlayfs/prompts/00-decisions-2026-08-15.md` — the decision log
  (Part A/B settled record; the mode entry B13 is closed by §14).
- `~/overlayfs/diet-coach-poc/project/runtime/evidence/` — the POC
  measurements (`01-mounts.txt` is the lineage of §7's derived root
  protections;
  `04-copyup-negative.txt` the thirteen refusals;
  `21-live-down-worktrees.md` the teardown-with-worktrees run).
