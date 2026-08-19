# Read-only whole-project code review at 8ddf464

Date: 2026-08-19. Code reviewed: `camp` on `main` at
`8ddf464955...`, with the twenty-six-commit change set
`3975b37..8ddf464` given the greatest weight. Normative source:
`reference/spec.md`, especially the seven invariants in §3 and §§7, 10,
12, 15 and 19. Measured constraints came from
`reference/constraints.md`. The two required closing/verification logs
from 2026-08-16 and 2026-08-18 were read before the code.

This was a strictly read-only review of `camp`. No file in the code
repository was changed, no commit or push was made, and no `camp up`,
`camp down`, `camp run`, sudo, mount or unmount operation was invoked.
The shared checkout was advanced by the concurrently running tester
during the review, so all final source inspection and permitted Go
verification used an exact temporary Git copy of `8ddf464` under `/tmp`.
All file and line references below are against that commit.

## Result and recommended repair order

The revision should not be closed or released. There are two P0
containment failures, five P1 correctness/safety failures and five P2
protocol or assertion failures.

The repair order should follow the dependencies:

1. Pin the privileged helper's base and replace every post-check path
   operation with an operation on a retained descriptor or mount handle.
2. Remove the root `fchmodat` symlink-following primitive and make every
   fsx permission change act on the inode that was checked.
3. Redesign the privileged write-ahead record so one pre-helper record
   can recover both staging and live locations at every kill boundary.
4. Pin all unprivileged fsx roots, including state, and move Scratch away
   from ambient `TMPDIR`.
5. Repair mountinfo tokenisation, Git subprocess isolation and the
   allow-listed-root exclude omission.
6. Close the smaller ENOTDIR, atomic publication, report delivery and log
   rotation gaps, then make the record's option assertions executable.

The accepted kernel-mount design in spec §22 does not need reopening.
The `fsopen`/`fsconfig`/`fsmount` portion is sound in isolation. The
failures are in how the helper reaches the objects around that sequence,
how it recovers them after interruption, and how several supposedly
atomic filesystem protocols are published.

## 1. P0 — the helper's confinement root can be replaced after validation

Primary locations:

- `internal/privileged/confine.go:18-40`, threat model.
- `internal/privileged/confine.go:81-86`, `confine` and `ownedBase` call.
- `internal/privileged/confine.go:187-232`, pathname-only base check.
- `internal/pathx/resolve.go:75-98`, base reopened for every walk.
- `internal/pathx/resolve.go:186-210`, zero-component and ordinary opens.
- `internal/privileged/helper.go:191-219`, staging detach and post-mount
  reopen.
- `internal/privileged/helper.go:249-280`, live detach and final move
  opens.
- `internal/privileged/helper.go:527-545`, teardown identity check.
- `internal/privileged/helper.go:585-619`, mountinfo/stat/unmount sequence.
- `internal/mountx/mountx.go:253-311`, descriptor mount followed by a
  pathname reopen.
- `internal/mountx/mountx.go:497-510`, pathname unmount.
- `internal/mountx/mountx.go:533-558`, pathname self-bind/private detach.

### Mechanism

`ownedBase` performs `unix.Lstat(base)` and checks that the inode visible
at that instant is a directory owned by `SUDO_UID`. It then returns the
same string. It does not retain a descriptor or identity.

Every subsequent `pathx.OpenBeneath` starts with:

```
unix.Open(base, O_PATH|O_DIRECTORY|O_CLOEXEC, 0)
```

That call follows a symlink at the final base component. The
`RESOLVE_BENEATH|RESOLVE_NO_SYMLINKS` flags apply only to components
opened after that base descriptor. They therefore confine the walk
beneath whichever inode the pathname names at the later call, not
beneath the inode `ownedBase` inspected.

The invoking user owns the environment and normally its parent. After
the `Lstat`, that user can rename the environment and place a symlink at
the old name. A crafted helper job whose components passed the lexical
checks can then make root resolve them beneath `/` or another tree. The
helper's own comments correctly include a warm sudo timestamp or a
camp-scoped `NOPASSWD` rule in the threat model, so direct invocation of
the internal helper with attacker-controlled stdin is in scope.

The two-stage TOCTOU argument does not close this:

- `precheck` reopens the base by name.
- Per-mount `resolve` reopens the base by name again.
- After a bind exists, `MountByDescriptor` deliberately asks the caller
  to reopen the target. That reopen is not tied to the target descriptor
  used to place the bind and does not prove it opened the new mount.
- Verification receives a staging pathname assembled from the same
  replaceable base.
- The final move opens staging and live through the same base pathname.
- Teardown reads mountinfo, stats a pathname and later passes the
  pathname to `umount(2)`, with a rename window between all three.

`Detach` is worse than the ordinary mounts because both its bind and its
propagation change use raw names. Teardown is likewise entirely
pathname-based.

Two wire fields intended to make the helper fail closed are dead:
`JobMount.TargetAbsent` and `JobMount.SourceType` are populated by the
front end but never read. `checkable` permits a missing target identity
solely from the caller-supplied staging prefix and does not require
`TargetAbsent`; it also does not require a source identity for a
non-overlay operation. This independently contradicts the comments at
`internal/privileged/job.go:19-26` and `:139-179` that every endpoint is
checked.

### Consequence

Root can be induced to bind, remount, change propagation, move or unmount
objects outside the checked environment. This is a confused-deputy root
primitive, not merely a composition race. The same defect lets rollback
or teardown act on a replacement mount after it checked the intended
one.

Even when the base itself remains stable, relocating an ancestor of a
newly placed nested mount before the post-bind reopen can make rollback
look at the replacement pathname, report the original mount absent and
leave the relocated mount unrecorded.

### Measurement

Use an isolated mount namespace or VM and test-only barriers at all of
these points:

1. immediately after `ownedBase`;
2. after `precheck` and before the first mount;
3. after a bind is attached and before `reopen`;
4. after `standsThere` and before `Unmount`;
5. before the staging/live descriptors for `Move` are opened.

At each barrier, rename the original environment and replace its name
with a symlink to a root-owned trap tree containing uniquely identified
mounts. The assertion is not merely that the helper errors: no mount ID,
mount attributes or inode mode outside the original base inode may
change.

### Repair

- Open the base exactly once for the whole helper invocation.
- Check ownership with `Fstat` on the opened descriptor, not with an
  earlier `Lstat` on its name.
- Retain the descriptor and pass it to every resolution routine.
- Make `pathx`/helper APIs accept a base fd and component vectors, never
  a base string.
- For binds, create a detached clone with `open_tree`, apply read-only and
  propagation attributes with `mount_setattr`, and attach with
  `move_mount`. Retain a handle to the created mount; do not reopen the
  target name after placement.
- Replace `Detach` with the same descriptor/mount-handle model.
- For teardown, identify the exact mounted object by descriptor and
  recorded identity/mount ID, move it to a root-owned non-user-renamable
  graveyard, and unmount it there. If that precise sequence depends on
  kernel behaviour, measure it before selecting the primitive.
- Make `TargetAbsent` and `SourceType` enforced fields or remove them from
  the wire and replace their promises with an actually checked
  representation.

### Acceptance criteria

- No helper syscall after `confine` resolves the environment root again.
- A base rename/symlink swap at every barrier changes no outside object.
- A target ancestor rename after bind placement either operates on the
  retained mount or fails while recording the real mount's reachable
  location.
- Teardown cannot unmount a replacement object inserted after its
  identity check.
- A hand-written helper job cannot omit any identity or type assertion
  that an honest front end would have supplied.

## 2. P0 — root cleanup follows a swapped symlink while changing mode

Primary locations:

- `internal/fsx/fsx.go:410-454`, recursive removal.
- `internal/fsx/fsx.go:425-430`, name-based chmod after no-follow stat.
- `internal/fsx/fsx.go:582-603`, directory creation and another
  name-based chmod.
- `internal/privileged/helper.go:142-163`, root caller.

### Mechanism

`removeTreeAt` obtains the final name's type with
`Fstatat(..., AT_SYMLINK_NOFOLLOW)`. If it saw a directory lacking owner
write/search permission, it next calls:

```
Fchmodat(dirfd, name, 0700, 0)
```

Flags zero means the final symlink is followed. There is a race between
the stat and chmod. The user can present the expected mode-000 directory,
rename it and replace the name with a symlink before the chmod. The
privileged helper then changes the symlink target's mode as root. The
subsequent `openat2` correctly rejects the symlink, but the damage has
already happened.

`makeAt` has the same shape: it safely opens the newly created/found
directory, then ignores that descriptor and chmods the name again.

The rest of the recursive delete is substantially better: child opens
use `openat2` with no symlink following, unlink acts relative to a pinned
parent, and recursive chown uses `Fchownat(...,
AT_SYMLINK_NOFOLLOW)`. No outside-tree delete or chown was found. The
chmod alone is sufficient for the P0.

### Consequence

A user can make the camp helper change an arbitrary root-owned file or
directory to mode `0700`. Targets such as a system directory turn this
into a machine-wide denial of service. It also falsifies the fsx package
claim that every chmod acts on the inode resolved without a gap.

### Measurement

Insert a deterministic barrier between the no-follow stat and chmod.
Replace the directory with a symlink to a root-owned trap file and trap
directory. Run the helper variant only in a disposable VM or isolated
environment. Both trap modes must remain byte-for-byte unchanged.

Repeat the same swap after `makeAt` opens its directory and before its
chmod. The ordinary-user variant must not change a repository inode's
mode either.

### Repair

Open the object with `O_PATH|O_NOFOLLOW`, verify its type with `Fstat`,
and change mode on that exact descriptor. On kernels that provide it,
`fchmodat2` with `AT_EMPTY_PATH` is the natural primitive for an `O_PATH`
descriptor. Otherwise select and measure another descriptor-safe
approach. For mode-000 cleanup, open `O_PATH` first, chmod the pinned
inode, then reopen it for listing.

### Acceptance criteria

- No `Fchmodat` call in fsx names an object that was checked in an earlier
  syscall.
- Stat/chmod and open/chmod swap tests leave the replacement target
  untouched.
- Recursive delete and chown tests still cover a hostile tree containing
  symlinks, files replacing directories and directories replacing files.

## 3. P1 — the write-ahead record cannot recover mounts still in staging

Primary locations:

- `internal/state/state.go:1-22`, complete-plan and no-unknown-mount
  assertions.
- `internal/state/state.go:124-132`, `Staging` and `Detached` fields.
- `internal/state/state.go:172-214`, record construction with final live
  targets.
- `internal/privileged/frontend.go:134-160`, pre-helper record write.
- `internal/privileged/frontend.go:185-190`, identity merge only after a
  complete helper reply.
- `internal/privileged/frontend.go:246-284`, teardown job derived without
  staging.
- `internal/privileged/helper.go:191-201`, staging self-bind.
- `internal/privileged/helper.go:203-238`, mounts built under staging.
- `internal/privileged/helper.go:241-286`, verification and move.
- `internal/privileged/helper.go:568-578`, rollback.
- `internal/cli/privileged.go:200-269`, down cleanup and record removal.

### Mechanism

The record is correctly written before the helper starts, but it records
the plan's final live targets. It records `Staging` as one string, yet no
recovery, presence, status or teardown path consumes that field. The
only detached point recorded by `FromPlan` is `built.Live`.

Until `Move`, the actual mount tree and all nested mounts are under the
staging directory, and the staging self-bind is the tree's parent. A
SIGKILL after any of those mounts but before the move leaves a valid
`mounting` record whose teardown targets do not name a single mount that
actually exists.

The resulting behaviour depends on the kill point:

- If the overlay is active in staging, teardown processes the absent
  live targets, then `clearWork` normally fails because the kernel's
  overlay work directory is still active. The record is kept, but every
  subsequent `camp down` repeats the same failure because it still cannot
  address staging. The CLI's sentence that every mount came down is
  false.
- If only the staging self-bind exists, privileged work cleanup may
  finish. Unprivileged `RemoveWorkDir` then encounters the mounted staging
  directory and reports it left alone, but `state.Forget` runs
  unconditionally afterward. The global mount remains without recovery
  authority.
- If ordinary helper rollback strands a staging mount, `reply.Stranded`
  appears only in prose. The record is changed to `partial` but the
  stranded paths are not added to it, so `down` still cannot target them.

`mountx.Detach` adds a second rollback hole. If the bind succeeds,
`MS_PRIVATE` fails and the attempted cleanup unmount also fails, the
cleanup error is discarded. The helper appends staging/live to `made`
only after `Detach` returns success, so unwind can report a clean rollback
over an existing self-bind.

Finally, after each successful mount, `identityOf(target)` errors are
silently ignored. A fully successful helper reply can therefore leave a
zero identity in the record. Teardown treats zero as authority to
unmount whatever now stands at that pathname.

### Consequence

This violates spec §12's record-alone recovery requirement and the state
package's claim that there is no moment when a mount exists and nothing
knows where it is. It can leave machine-wide mounts that camp cannot
recover, forget the only record, or later unmount an unverified
replacement.

The active overlay workdir normally blocks the most dangerous recursive
work-tree removal, so this review does not claim deterministic repository
deletion from this path. The confirmed failure is unrecoverable or
unrecorded mount state and a false teardown report.

### Measurement

Implement the complete privileged kill matrix in a disposable mount
namespace or VM. Kill after:

1. staging self-bind;
2. each nested mount;
3. staging verification;
4. live self-bind;
5. successful `move_mount`;
6. staging self-bind removal;
7. reply encoding/writing;
8. receipt of the reply but before the front-end record save.

For each point, remove or corrupt the configuration, run recovery using
only the record, and require:

- all camp-created mount IDs disappear;
- no unrelated mount ID disappears;
- repositories and storage content hash identically before and after;
- the record remains until all possible locations are clean;
- every mount that cannot be removed is named exactly.

Fault-inject every stage of `Detach` and identity acquisition as well.

### Repair

- Before invoking the helper, record every mount's staging target and
  final live target, plus both self-binds.
- Treat both as possible locations of the same planned mount until a
  durable transition proves otherwise.
- Teardown should inspect mountinfo and identity at both locations in
  reverse order, rather than assuming the move happened.
- If the reply is available, merge exact mount identities and transition
  state; if not, the original record must still be sufficient.
- Persist stranded rollback locations as concrete recovery targets, not
  only message text.
- Make `Detach` return whether the first bind exists and propagate failed
  cleanup.
- Obtain mount identity from a retained descriptor. Failure must abort and
  roll back; it must not produce a successful zero-identity result.
- Never forget a record while any mount exists at or beneath its work,
  staging or live paths.

### Acceptance criteria

Every kill-point test recovers from the pre-helper record alone. No test
requires the configuration or a successful helper reply. A failed
cleanup keeps a record containing the exact surviving location and
identity.

## 4. P1 — fsx bases and Scratch permit repository writes

Primary locations:

- `internal/fsx/fsx.go:1-30`, package confinement assertion.
- `internal/fsx/fsx.go:67-75`, arbitrary string-backed `At` constructor.
- `internal/fsx/fsx.go:84-87`, state area.
- `internal/fsx/fsx.go:110-130`, Scratch and ignored cleanup result.
- `internal/fsx/fsx.go:544-579`, base reopened by name.
- `internal/pathx/resolve.go:75-98`, same base behaviour in reads/opens.
- `internal/state/state.go:141-167`, ambient state base.
- `internal/privileged/frontend.go:68-94`, lexical repository guard.
- `internal/preflight/preflight.go:306-353`, Scratch consumer.

### Mechanism

An `Area` stores a base string and components. It strictly resolves the
components, but it does not pin or no-follow the base. That is narrower
than the package comments promise.

There are two direct, non-mounting ways to write a repository:

1. `XDG_STATE_HOME` can be a symlink into a code repository.
   `recordsOutsideRepositories` compares the lexical state path with the
   lexical repository roots, so the symlink alias is not recognised.
   `fsx.State` then follows the base symlink and creates `camp/<hash>.json`
   in the repository. The existing test covers a symlink in the `camp`
   child, not a symlink at the state base itself.
2. `Scratch` uses `os.MkdirTemp("", prefix)`, which honours ambient
   `TMPDIR`. When `TMPDIR` is a repository, the capability probe creates
   and writes its lower/upper/work/merged tree in that repository. Normal
   cleanup removes it, but the invariant is "never writes", not "usually
   removes afterward", and a crash preserves the residue.

The same base-reopen design permits a user to rename the resolved
environment between validation and any later fsx write and replace it
with a symlink to another repository.

### Consequence

Invariant 1 is violated by ordinary unprivileged code. State files,
probe files, directory modes and mtimes can land in a Git repository. A
short-lived write is still visible to concurrent Git and filesystem
watchers, and a crash makes it durable.

### Measurement

- Create a temporary Git repository and a symlink used as
  `XDG_STATE_HOME`. Save a state record and compare the repository tree,
  index/status and directory metadata.
- Set `TMPDIR` to a temporary repository and run the probe function in a
  disposable child. Observe both the live write window and crash residue.
- Add a barrier after plan validation and before `WriteInputs`, rename the
  environment and substitute a repository symlink. No repository inode
  may be created, changed, chmodded or removed.

### Repair

- Make every `Area` own or borrow a pinned base descriptor and identity.
- Resolve the environment once to an inode, not only to a canonical
  string, and retain that capability through the command.
- Resolve the state base without following a replaceable final symlink,
  compare its actual identity/location against repository identities, and
  pin it before writing.
- Do not use ambient `TMPDIR` for the capability probe. Move the probe
  after environment validation and allocate it in an explicitly
  non-repository camp area, or build it on a detached in-memory filesystem
  whose creation does not touch an arbitrary user-selected directory.
- Return and report Scratch cleanup failure.

### Acceptance criteria

Base-symlink, base-rename, state-alias and `TMPDIR=repository` tests all
leave the repository byte- and metadata-identical.

## 5. P1 — the strict mountinfo reader rejects legal host state

Primary locations:

- `internal/mountinfo/mountinfo.go:86-119`, whole-table reader and 1 MiB
  scanner cap.
- `internal/mountinfo/mountinfo.go:122-184`, parser.
- `internal/mountinfo/mountinfo.go:128-130`, `strings.Fields`.
- `internal/mountinfo/mountinfo.go:146-149`, trailing-field off-by-one.
- `internal/cli/cli.go:290-295`, doctor silently ignores the error.

### Mechanism

The mountinfo ABI separates fields with literal ASCII space. Linux
escapes space, tab, newline and backslash in pathname fields. Other legal
filename bytes that Go considers whitespace, including carriage return,
vertical tab and form feed, can remain raw. `strings.Fields` splits on
all Unicode whitespace and coalesces runs. One unrelated mount containing
one of those bytes can therefore change the field count or shift the
separator, causing the new whole-table policy to reject every entry.

The fail-closed choice is correct once a record is genuinely malformed.
The bug is using a grammar stricter and different from the kernel's.

The independent 1 MiB `bufio.Scanner` limit is not justified by a cited
kernel bound. The new mount API permits lower layers to be appended one
at a time, so a legal overlay line can be much larger than the historical
single-page option string. Whether current overlay layer/path limits can
cross 1 MiB should be decided by an actual maximum-layer measurement;
the review did not require that uncertainty for the finding because the
raw-whitespace case is already decisive.

The opposite strictness error is at line 146. A record needs three fields
after `-`; when exactly two remain, `separator+3 > len(fields)` is false
and the malformed record is accepted with an empty super-options field.

Doctor suppresses any read error and continues after printing that the
configuration is sound. Other commands that use the table refuse, so the
same host state is presented inconsistently.

### Consequence

A legitimate composition can become unusable on a machine with an
unrelated legal mount name. More importantly, `camp down` can be blocked
from removing its mounts indefinitely. Doctor fails to disclose the
cause and can issue a misleading positive verdict.

### Measurement

Parser tests should include:

- raw `\r`, `\v` and `\f` in root, mountpoint and source positions;
- encoded space/tab/newline/backslash controls;
- repeated or leading literal spaces, which should be rejected if the
  kernel grammar does not emit them;
- zero, two and exactly three fields after `-`;
- a line larger than 1 MiB;
- invalid octal escapes and unknown optional fields.

Then create an actual mount with one of the legal raw bytes in an
isolated namespace and compare camp's parsed record with the kernel line.
For the size question, construct the maximum supported new-API overlay
and record the longest emitted line.

### Repair

- Tokenise bytes on literal `0x20`, not `strings.Fields`.
- Preserve non-separator bytes exactly until the mountinfo escape decoder
  handles them.
- Enforce the exact mandatory field count around `-`.
- Replace Scanner's arbitrary cap with a dynamically growing reader or a
  measured ABI-derived maximum.
- Make doctor report `mount-table-unreadable`, omit any environment
  health verdict that depends on the table and return non-success or an
  explicitly incomplete status.

### Acceptance criteria

Every legal-byte fixture parses; every malformed delimiter/cardinality
fixture fails; no supported kernel-produced line is rejected solely for
size; all commands report a table failure consistently.

## 6. P1 — ambient Git variables bypass the tracked-content guard

Primary locations:

- `internal/gitwire/gitwire.go:71-113`, tri-state contract and `Open`.
- `internal/gitwire/gitwire.go:127-139`, message classifier.
- `internal/gitwire/gitwire.go:181-192`, inherited environment.
- `internal/plan/validate.go:79-97`, initial Git classification.
- `internal/plan/sequence.go:285-299`, swallowed `TracksUnder` error.
- `internal/gen/prepare.go:331-363`, expanded recheck.
- `internal/gen/exclude.go:1-13`, assertion that Git is outside core.

### Mechanism

`gitwire.run` appends locale variables to `os.Environ` but does not remove
Git's repository-selection variables. `GIT_DIR`, `GIT_WORK_TREE`,
`GIT_INDEX_FILE`, `GIT_COMMON_DIR`, `GIT_CEILING_DIRECTORIES` and related
discovery controls can redirect or disable the query despite `git -C`
naming the intended code repository.

For example, on the valid `8ddf464` checkout this command was measured:

```
GIT_DIR=/tmp/camp-review-no-such-git-dir \
  git -C <checkout> rev-parse --is-inside-work-tree --show-prefix
```

It exits 128 and prints `fatal: not a git repository: ...`. That is the
exact message `notARepository` converts into `NotAWorkTree`, not
`Unreadable`. Plan and generation then omit the tracked-target guard
instead of refusing.

A damaged `.git` file pointing to an absent gitdir can produce the same
classification even though the comments explicitly name a damaged
repository as `Unreadable`.

There is also a narrower operational gap. Once `Open` succeeds,
`checker.checkTracked` silently returns on a `TracksUnder` error.
`gen.Prepare` usually asks again later and can stop an actual `up`, but
`camp plan` uses the static result and can print `nothing stops this
composition` after the check failed.

The architecture assertion has drifted as well: `plan` directly imports
and invokes `gitwire`, while comments in `gen` and the no-generation path
say core carries no Git knowledge.

### Consequence

A mount can cover tracked code while camp believes the upper is not a Git
worktree. Git then reports the covered paths deleted, and `git commit -a`
can record those deletions. This defeats the purpose of the new tri-state
reader.

### Measurement

For a repository with a tracked file beneath a configured target, run
both `plan` and generation under every repository-selecting `GIT_*`
variable. Also test a broken `.git` indirection and an injected failure
between `Open` and `TracksUnder`. Every inability to inspect the intended
repository must produce `git-unreadable`; it must never become
`NotAWorkTree` or an empty tracked set.

### Repair

- Build the Git subprocess environment by filtering repository-selection
  and discovery `GIT_*` variables, then add the fixed locale.
- When Git says `not a repository` but a `.git` control entry exists in
  the intended discovery frame, classify it as `Unreadable`.
- Propagate every `TracksUnder` error into the caller's refusal list.
- Make `plan` and the generation architecture agree with spec §19: either
  move the tracked-content query behind the generation boundary or make
  the normative documentation explicitly acknowledge where the query
  lives. Do not leave opposite `never` claims in source.

### Acceptance criteria

Ambient Git variables cannot change which repository is inspected. A
damaged repository and every operational read failure stop both `plan`
and `up` with `git-unreadable`.

## 7. P1 — an allow-listed root mount target is omitted from the exclude

Primary locations:

- `internal/gen/exclude.go:37-62`, normative formula and explanation.
- `internal/gen/exclude.go:76-108`, lower-root and allow-listed-directory
  handling.
- `internal/gen/exclude.go:110-119`, mount-target loop and exception.

### Mechanism

The implementation correctly performs the intentional 005 deviation:
inside an allow-listed directory it enumerates lower-only children
instead of excluding the whole root. That is consistent with the current
specification.

The next loop then skips any live mount target whose relative path is one
component and that component is allow-listed. This contradicts the
function's own formula:

```
exclude = (workspace root entries) - (allow_overlap) + (every mount target)
```

Overlap policy decides which lower content may merge. It does not make a
mount placed over that directory safe to expose to Git.

A valid example is an allow-listed `shared/` directory present on both
sides, with an untracked or empty code-side portion, followed by a store
or repository mount at target `shared`. The tracked-target rule can
legitimately pass. Because `/shared` is absent from the exclude, files
from the mounted source appear as untracked code and `git add .` can
stage them.

### Consequence

Machine-local storage, workspace data or another repository's data can
enter the code repository's index. This violates spec §10's byte-exact
exclude formula and the core non-leakage purpose of the exclude.

### Measurement

Build a fixture with both roots containing `shared/`, add
`allow_overlap: [shared]`, and place a mount at target `shared`. Assert
that `ExcludeLines` contains exactly one `/shared` line. In a temporary
worktree, use `git check-ignore` and `git add -n` against files supplied
only by the mount source.

### Repair

Remove the allow-list exception from the mount-target loop. Always add
every non-root live mount target. The existing `seen` map provides the
required de-duplication and lets the coarser target exclusion override
the per-child allow-list enumeration naturally.

### Acceptance criteria

Every live mount target appears in the generated exclude regardless of
`allow_overlap`; byte ordering and escaping remain deterministic.

## 8. P2 — fsx still folds ENOTDIR into absence

Primary locations:

- `internal/pathx/resolve.go:100-110`, new `ErrNotDirectory` translation.
- `internal/pathx/resolve.go:158-165`, pathx correctly treats only ENOENT
  as absence.
- `internal/fsx/fsx.go:410-454`, recursive remove.
- `internal/fsx/fsx.go:476-510`, recursive chown.
- `internal/fsx/fsx.go:606-619`, unlink.
- `internal/fsx/fsx.go:658-660`, fsx still treats ENOTDIR as absence.

### Mechanism

The pathx rewrite correctly distinguished an intermediate non-directory
from a missing path. All production `pathx.StatBeneath` callers were
audited; no legitimate composition was found that should treat an
intermediate file as absence. In repository, source, live, virtual
overlay, generator, islands and lock-identity checks, such a file really
makes the declared descendant unreachable.

The same correction was not made in fsx. Its `isAbsent` still returns
true for `ENOTDIR`. With a pinned parent directory, the relevant race is
at the final component:

- `removeTreeAt` stats a directory, it becomes a file, and the subsequent
  `O_DIRECTORY` open returns `ENOTDIR`; removal returns success.
- `chownTreeAt` has the same type transition and reports success without
  descending.
- `Unlinkat(..., AT_REMOVEDIR)` can encounter a file replacement and have
  its type error swallowed.

### Consequence

Rollback, sweep or teardown can report that an object was absent/cleared
while a replacement remains. Later operations often detect the residue,
which keeps this at P2, but the cleanup contract and several load-bearing
comments are false.

### Measurement

Use barriers after the initial stat and before the directory open or
unlink. Replace directory with file and file with directory. Success is
allowed only when the name is truly absent; every type race must return a
specific non-nil error or retry to a stable conclusion.

### Repair

Make fsx absence mean `ENOENT` only. Handle `ENOTDIR`, `EISDIR` and other
type conflicts explicitly as a race/refusal or bounded retry.

### Acceptance criteria

No destructive or ownership operation returns success merely because an
object changed type.

## 9. P2 — the atomic writer shares one temporary filename

Primary locations:

- `internal/fsx/fsx.go:314-328`, `Area.Write` contract.
- `internal/fsx/fsx.go:331-357`, implementation.
- `internal/fsx/fsx.go:334-335`, fixed temporary name and missing
  `O_EXCL`.

### Mechanism

Every writer replacing `name` uses `.<name>.camp` and opens it with
`O_CREAT|O_TRUNC|O_WRONLY`. Two processes therefore open the same inode.
A deterministic failing sequence is:

1. A and B both open the same temporary inode.
2. A writes, syncs and renames it to the final name.
3. A returns success.
4. B writes through its still-open descriptor, which now refers to the
   final file's inode.
5. B's rename reports source `ENOENT` because A already moved the name.

A successful atomic write has changed after return, even if each
individual regular-file write was complete and non-interleaved.

State records are the highest-risk consumer: concurrent down/recovery
commands can publish different phases or identities for the same hash.

### Consequence

The final file can contain the failed writer's bytes, a mixed/truncated
payload, or semantically stale state. For recovery records this can
destroy the authority needed after a crash.

### Measurement

Synchronise two writers after open, give them different-sized patterned
payloads, let A rename before B writes, and assert:

- the final bytes are exactly one complete successful payload;
- the bytes cannot change after a writer returns success;
- a failed writer cannot mutate the published file.

Run the same test from separate processes, not only goroutines.

### Repair

Create a unique unpredictable temporary file per call with `O_EXCL`,
write/chmod/fsync it, rename it over the destination, and fsync the
directory. Where several processes may perform semantic transitions on
the same state record, add a directory/inode lock or compare-and-swap
protocol above the atomic byte replacement.

### Acceptance criteria

The deterministic two-writer schedule cannot mutate a successful
publication and never produces hybrid bytes.

## 10. P2 — report publication and show-once delivery are not atomic

Primary locations:

- `internal/reports/reports.go:38-61`, report publication.
- `internal/reports/reports.go:64-84`, empty final-name claim.
- `internal/reports/reports.go:118-164`, non-atomic mark-seen protocol.
- `internal/reports/reports.go:183-207`, read/print/rename sequence.
- `internal/fsx/fsx.go:287-311`, global Rename ENOENT suppression.

### Mechanism

`reports.Write` claims the final report name by creating an empty file
with `O_EXCL`, then calls `Area.Write` to replace it with the real body.
The final name is visible as an unseen empty report during that window.
A reader can print and mark it before publication; a write failure leaves
the empty report behind permanently.

`Show` lists and reads before it marks. Two simultaneous commands can
both read and print one unseen report before either rename occurs.
`MarkSeen` also finds its destination with `Lstat` before a normal
replacing rename.

The losing rename sees source `ENOENT`, but `fsx.Area.Rename` globally
turns that into success. Both commands therefore believe they performed
the once-only transition.

The same global suppression makes log rotation unable to distinguish an
optional missing old generation from the current file disappearing
unexpectedly.

### Consequence

Reports can be delivered empty, delivered more than once, silently
published after an empty delivery, or left as misleading empty evidence.
These are diagnostic rather than authority files, hence P2, but they
contain the only post-session evidence for detached namespace sessions.

### Measurement

- Barrier two `Show` processes after `Read`; exactly one output callback
  may receive the body.
- Pause `Write` after the empty claim and run `Show`; no empty report may
  be observable.
- Inject every write/sync/rename failure; no empty final report may
  remain and no completed body may be lost.
- Race two mark-seen operations and require the loser to receive an
  explicit state-change result.

### Repair

- Write the body to a unique `O_EXCL` temporary and publish it to a final
  unseen name only after all writes and syncs succeed, preferably with
  `renameat2(RENAME_NOREPLACE)` and retry on collision.
- Hold a stable directory flock across unseen listing, read, terminal
  output and mark-seen. A process crash then releases the lock while the
  original unseen file remains retryable.
- Return source `ENOENT` from `fsx.Rename`. Let log rotation explicitly
  ignore only the old generations for which absence is expected.

### Acceptance criteria

Concurrent commands deliver one report once; failed publication exposes
no final file; the caller can distinguish a lost rename race from
success.

## 11. P2 — always-on logging and two-process rotation have uncovered gaps

Primary locations:

- `internal/cli/cli.go:50-73`, always-on and `keepUnder` contracts.
- `internal/cli/cli.go:161-185`, config load before log attachment.
- `internal/logs/logs.go:65-103`, directory and file acquisition.
- `internal/logs/logs.go:163-175`, ignored flock errors.
- `internal/logs/logs.go:207-228`, multi-rename rotation.
- `internal/session/session.go:268-283`, namespace init silently ignores
  `logs.Open` failure.

### Mechanism

Once `resolve` has found the standard configuration path, it knows which
environment's log should receive the run. It nevertheless calls
`config.Load` before `ctx.keep`. A malformed or unreadable configuration
therefore produces a terminal error that is never logged. `keepUnder`
already encodes the intended mechanism but is not used on this path.

The directory flock is the serialization for a two-process rotation, but
both `LOCK_EX` and `LOCK_UN` errors are ignored. On a filesystem without
working flock semantics, both writers execute the multi-step generation
renames concurrently. Normal rename replacement and the separate
ENOENT-suppression bug can overwrite or skip rotated history. The comment
claiming that only which side of rotation receives a line is at risk is
not true for concurrent rotations.

The namespace init creates a second sink and attempts to open the same
log. If that open fails, it silently continues. This is most of a
session's narration, so a failure that occurs only after handoff can
remove the interesting portion of the log without the promised one-time
complaint.

The directory descriptor and append file are also opened through
separate base-path resolutions. The general fsx base-swap finding can
make them belong to different directory inodes, invalidating the lock;
pinning the Area base fixes that shared cause.

### Consequence

Some of the runs most worth diagnosing—bad configurations, handoff-time
failures and concurrent rotations on an unsupported filesystem—are
missing or lose prior history despite comments saying logging is always
on and failure is reported once.

### Measurement

- Resolve a standard but malformed configuration and require its full
  refusal in the environment log.
- Make the launcher's log open succeed and the init's fail; require one
  warning and continued execution.
- Inject `EOPNOTSUPP`/`ENOSYS` from flock and run two tagged writers over
  the rotation threshold. Account for every line that should remain
  within the retention policy and ensure no generation is replaced by a
  race.
- Swap the environment/log base between directory open and append open;
  both descriptors must still identify the same directory tree.

### Repair

- Once a standard config path is found, attach the log with a validated
  `keepUnder` before parsing it.
- Check flock results. If interprocess locking is unavailable, continue
  appending but disable rotation for that run and issue exactly one
  warning through the sink.
- Route the namespace init's `logs.Open` failure through the same
  one-time complaint path.
- Open the directory and log file relative to one pinned Area descriptor.

### Negative checks

The line-oriented sink itself is correct: it buffers partial writes and
flushes the final partial line on `Close`. Main evaluates `cli.Main` and
its deferred close before calling `os.Exit`. The namespace farewell body
currently always ends in a newline, so its final `os.Exit` does not lose
a present camp line, although it bypasses the defer. No current last-line
loss was demonstrated.

Camp narration stays on stderr; command products and the configured
generator/workload stdout routes remain intentionally untouched. Colour
is added only to the terminal copy and never to the log.

### Acceptance criteria

Malformed config, init-open failure and unsupported-lock tests each emit
one useful warning or retained refusal; concurrent rotation preserves the
defined generations without races; logs remain colour-free and stdout
remains unmodified.

## 12. P2 — recorded overlay options are a dead, independently rendered assertion

Primary locations:

- `internal/state/state.go:72-86`, recorded `Options` and `FSType`.
- `internal/state/state.go:205-210`, claim that the mount and record share
  a renderer.
- `internal/mountx/mountx.go:147-200`, actual descriptor/API mount.
- `internal/mountx/mountx.go:472-485`, string renderer used by state only.
- `internal/state/state.go:292-347`, record validation does not validate
  those fields.
- `internal/state/state.go:415-452`, presence checks ignore them.

### Mechanism

The record says the overlay option string is rendered by the same
function that mounts it. That stopped being true with the kernel mount
API rewrite. Actual mounting performs one `FsconfigSetFd` per lower,
separate `upperdir` and `workdir` fd assignments and an optional
`FsconfigSetFlag`. `mountx.Options` independently rebuilds a legacy-style
string from plan pathnames. Its only production caller is the state
serializer.

Neither `Options` nor `FSType` is consumed during record validation,
presence checking, teardown, status or explain. Tests establish only
that serialization filled a string. An actual fsconfig change can drift
without any test or runtime consumer noticing.

The record's `Staging` field and the helper wire's `TargetAbsent` and
`SourceType` fields are other examples of fields whose comments promise
behaviour but whose consumers do not exist; their functional consequences
are covered in findings 1 and 3.

### Consequence

The §12 `complete concrete plan` property is documentary rather than
executable for overlay options. A future xattr, lower-order or mount-API
change can make the record describe a mount different from the one the
helper created while all current tests remain green.

### Measurement

In a mutation test, change an actual fsconfig key, flag or lower order
without changing `mountx.Options`; the existing state test still passes.
The repaired test should compare one normalized structured mount
description used by the helper, state record and post-mount verifier.

### Repair

Create one structured overlay operand/option representation. The mount
executor, state encoder and verifier must consume that object rather than
re-rendering independently. Recovery/status should either validate the
recorded facts against the mounted object or the fields should be removed
only after the normative requirement is explicitly changed.

### Acceptance criteria

An intentional one-sided change to any overlay operand or flag fails a
test, and every persisted option field has a production consumer.

## Specification and intentional-deviation assessment

### Spec §7 — privileged fixed frame

The fixed-frame order, two verification passes and final descriptor move
are present. The violations are implementation-level: the base and
post-bind mount are not pinned, rollback can omit existing self-binds,
and the path-based verification can be redirected with the base.

### Spec §10 and intentional deviation 005

File-level enumeration of lower-only content inside an allow-listed
directory matches the current normative text and is not reopened here.
The exception for an allow-listed root that is itself a mount target is
not part of deviation 005 and violates the separate `every mount target`
term.

### Spec §12 — write-ahead state and crash recovery

The record is written at the correct time and contains the planned live
sequence, but it is not complete for the helper's staging phase. The kill
matrix must cover every helper transition, not only the final live tree.

### Spec §15 — path authority and mountinfo cross-check

Failing closed on an unreadable whole mount table is the correct policy.
The parser must first accept every table the kernel can legally emit.
Doctor's silent omission is incompatible with that policy.

### Spec §19 — hostile generation and Git

Generator outputs are re-read and validated, stdout remains direct as
specified, and no configured code runs under privilege. The ambient Git
environment can still switch off the tracked-content guard. The code and
comments also disagree about whether Git knowledge lives in core.

### Intentional deviation 024

The narrowed ownership check is implemented consistently. Verification
checks the storage root and camp-created store/island source objects, and
does not recursively claim ownership of arbitrary user content. No
finding is raised against this deviation.

### Logging and colour

The specification is silent, so the feature itself is not a spec
deviation. The findings arise from stronger load-bearing source comments
(`always`, `every line`, and the stated rotation guarantee). The colour
implementation matches its own claims.

## Other negative results

- The `fsopen` → `fsconfig` → `fsmount` → `move_mount` overlay sequence
  closes the filesystem context and detached mount descriptors on normal
  and error returns. Operand descriptors are closed by their caller. No
  independent fd leak was found there.
- No lazy unmount was found. `mountx.Unmount` uses flags zero.
- The session lock handoff compares the inherited lock descriptors with
  the init's independently rebuilt plan by device/inode. Apart from the
  shared base-reopen issue, that identity check is sound.
- The new pathx treatment of intermediate `ENOTDIR` is correct at all
  current production callers; the residual problem is fsx's separate
  error folding.
- The ordinary recursive chown uses `AT_SYMLINK_NOFOLLOW` and its descent
  uses openat2. The identified root escape is the permission-change call,
  not chown traversal.
- Grouped refusals, NUL-delimited worktree parsing, strict generator
  interruption, mount source reporting, upper identity, islands output
  matching, and the narrowed storage ownership change did not yield an
  additional finding.
- Sink colour is terminal-only and the log receives plain text.
- The current stdout routes match §19: camp's narration stays on stderr,
  while a configured generator and workload retain their direct stdout.

## Verification performed

The exact target was archived and placed in a temporary Git checkout
because the shared tester advanced the workspace after review began.
The archive was compared with `8ddf464` and had no source differences.

The following permitted checks passed against that exact checkout, with
all caches and test roots under `/tmp`:

```
go build -buildvcs=false ./...
go vet ./...
CAMP_TEST_ROOT=/tmp/camp-review-test-root-8ddf464 go test ./...
```

An earlier test attempt used the suite's default
`/home/dlaszlo/overlayfs/.camp-tests` and failed before exercising code
because that location was not writable in the review sandbox. Re-running
with `CAMP_TEST_ROOT` under `/tmp` passed. That environmental failure is
not a project finding.

No privileged or mount-dependent adversarial measurement described in
this document was run. Those tests are required for repair acceptance
but were deliberately excluded from this read-only review because a
separate session was measuring machine state on the same host.

## Final verdict

The twenty-six commits close many of the earlier review's visible gaps,
and the descriptor-based overlay creation is the right kernel API shape.
The revision is nevertheless not safe to close. The helper's unpinned
base reintroduces a root confused deputy around the descriptor work, and
root cleanup contains a direct symlink-following chmod. Crash recovery
does not describe the staging state that the new privileged design
creates. The remaining P1 findings can either block recovery on an
otherwise legitimate host or bypass repository-protection gates.

The two P0s and all five P1s should be fixed and measured before another
closure claim. The P2s should be repaired in the same pass because they
sit in the durability and diagnostic mechanisms needed to prove those
larger fixes.
