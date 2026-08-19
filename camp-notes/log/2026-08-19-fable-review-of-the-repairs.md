# Read-only review of the repair branch `fix/review-8ddf464`

Date: 2026-08-19. Reviewed: `camp` on `fix/review-8ddf464`, head
`525093b`, based on `main` at `e648a7f`. Eleven commits,
`git log e648a7f..HEAD`. The branch answers the twelve findings of
`log/2026-08-19-read-only-code-review-8ddf464.md`; that review's
"Acceptance criteria" are the contract measured against. Normative
source `reference/spec.md`; measured facts `reference/constraints.md`;
the deferred-check ledger `reference/remaining-checks.md`.

Strictly read-only on the code: no file under `camp/` was changed, no
commit, no push, and no `up`/`down`/`run`, sudo, mount or unmount was
invoked. What was run: `go build ./...`, `go vet ./...`, the full suite
under `CAMP_TEST_ROOT=/tmp/.../scratchpad/tr-fable`, and — in a throwaway
`git` clone under the scratchpad, never in the reviewed tree —
twenty-odd single-line reversions to test whether the new tests have
teeth. Line and file references are against `525093b`.

## Result and recommended order

The eleven commits do close the twelve findings, at the level this
environment can see. The two P0s and five P1s are repaired in code with
the right shape; the five P2s are repaired and, unlike the originals,
each has an executable test. The build is clean, `vet` is clean, and the
whole suite passes. I checked twenty of the new tests by restoring the
old behaviour in a scratch copy and confirming each one goes red; all
twenty had teeth (list at the end).

The branch is **not yet provably safe to merge**, for one reason the
author already knows and states plainly in the commits: the entire
privileged mount and teardown path is **unrun**. Nothing in this
repository may mount, so the descriptor-mount sequence, the crash-kill
matrix and the rename-race barriers are written against the kernel's
documented contract and proven by argument, not by execution. That is
honest and it is unavoidable here — but it means the P0/P1 acceptance
criteria that require an adversarial mount test are **not measurable in
this environment**, and the merge decision rests on running them once on
the install, exactly as `remaining-checks.md` lays out.

Below the twelve, the repairs introduce three smaller risks of their
own, none a blocker but the first worth deciding before merge:

1. A record can become **permanently undiscardable** when an unrelated
   mount stands beneath the live/staging/work tree, and the message does
   not name the way out. New behaviour, from commit `656fd67`.
2. The record grew fields **without a version bump**, so an older build
   rejects a newer record with a parser error instead of the intended
   "use that build" sentence.
3. `reports.Show` holds a directory flock **across terminal output**, so
   a camp whose stderr is wedged blocks the next camp in that
   environment at startup.

Recommended order: run the install-gated privileged matrix
(`remaining-checks.md`) — that is what converts "argued" into "measured"
for findings 1, 2 and 3. Then decide risk 1. Risks 2 and 3 are cheap and
can ride along.

## Finding by finding

Each entry: what the defect was, what the branch did, and whether each
stated acceptance criterion is **met**, **not met**, or **not
measurable** here. Where a deferral is claimed I say whether the source
records it honestly or drops it quietly.

### 1 — P0, the helper's confinement root could be replaced after validation

Commits `12891bb`, `b304db0`, `78b3ba7`. `ownedBase`
(`internal/privileged/confine.go:255`) now opens the base **once** with
`OpenRootExactly` (`internal/pathx/root.go:82`, `O_NOFOLLOW|O_DIRECTORY`)
and asks ownership of that descriptor via `Fstat`
(`confine.go:yours`, `:289`), not of a name. The descriptor is a
`pathx.Root` held for the whole invocation
(`helper.go:87` `job.confine()`), and every operand, reopen, work-dir and
teardown target is resolved from it by component with
`RESOLVE_NO_SYMLINKS|RESOLVE_BENEATH` (`root.go:Open`, `:170`). Binds
became `open_tree(OPEN_TREE_CLONE)` + `move_mount` with the RO remount
and `MS_PRIVATE` addressed through the created mount's own
`/proc/self/fd/N` (`mountx.go:MountByDescriptor`, `:514`). The two dead
wire fields are now enforced: `checkable` (`confine.go:143`) requires
`TargetAbsent` for a missing staging identity and requires `SourceType`
∈ {directory,file} for every source bind.

- "No helper syscall after `confine` resolves the environment root
  again" — **partly met, honestly deferred.** The mounts, prechecks,
  identity reads, work-dir and the staging/live self-binds are all
  descriptor-relative. The residuals are the teardown's `umount2` and the
  Detach-cleanup unmount, which still take a path (`helper.go:759`
  comment, `mountx.go:Detach` cleanup comment). Both say so in as many
  words, and both name the reason: the descriptor-safe unmount depends on
  kernel behaviour nobody here has measured. Deferral recorded, not
  dropped.
- "A base rename/symlink swap at every barrier changes no outside
  object" — **not measurable** (needs the mount barriers). The code shape
  is correct: the base cannot be re-resolved, so a swap of its name after
  `confine` reaches nothing.
- "A hand-written job cannot omit any identity or type assertion" —
  **met and tested.** `checkable` refuses a missing target identity that
  is not both inside staging and flagged absent, and a source without a
  directory/file type. Reverting the `TargetAbsent`/`SourceType`
  enforcement reddens `TestTheHelperRefusesAnOperandItCannotCheck`.
- Teardown-cannot-unmount-a-replacement and target-ancestor-rename —
  **not measurable**; `standsThere` (`helper.go:701`) compares recorded
  identity against `identityUnder` before every unmount and treats a
  zero recorded identity as "unmount what you were given", which the kill
  matrix in `remaining-checks.md` measured on `8ddf464` and reasoned
  about here.

### 2 — P0, root cleanup followed a swapped symlink while changing mode

Commit `1132f3f`. `removeTreeOnce` (`fsx.go:664`) opens the object
`O_PATH|O_NOFOLLOW`, reads its type off that descriptor, and changes the
mode with `chmodFd` (`fsx.go:923`) through the descriptor's own
`/proc/self/fd/N` rather than `Fchmodat(dir,name,...)`. `makeAt`
(`fsx.go:890`) does the same for the created directory.

- "No `Fchmodat` call in fsx names an object checked in an earlier
  syscall" — **met, and guarded.** `TestNoModeChangeInFsxNamesItsObject`
  (`chmod_test.go:71`) fails the build if `unix.Fchmodat(` reappears;
  reverting `chmodFd` to a name-based `Fchmodat` reddens it.
- Swap tests leave the replacement untouched — **met at unit level.**
  `TestChmodFdActsOnTheDescriptorAndNotTheName` (`chmod_test.go:22`)
  arranges the rename-and-symlink race deterministically and asserts the
  link target's 0600 is unchanged. This is the one P0 whose acceptance
  test can run unprivileged, and it does.
- Recursive delete/chown still cover a hostile tree — **met**; see
  `removetree_test.go` and `typerace_internal_test.go`.

### 3 — P1, the write-ahead record could not recover mounts still in staging

Commit `656fd67`. `FromPlan` (`state.go:324`) now records every mount's
`Staging` location and both self-binds in `Detached`
(`state.go:352`); `Teardown` (`state.go:870`) names live targets, then
staging targets, then both self-binds, in reverse build order; a mount
whose identity cannot be read back **fails and rolls back** rather than
recording a zero (`helper.go:238`); stranded locations are written into
`Record.Stranded` as targets (`frontend.go:196`, `state.go:394`); and a
record is never discarded while `Held` (`state.go:924`) finds anything
standing.

- Every kill-point recovers from the pre-helper record alone; no test
  needs the reply — **not measurable** (the matrix needs a disposable
  machine). What *is* measurable is decidable from data and is tested:
  reverting the staging half of `Teardown` reddens
  `TestTheTeardownNamesBothPlacesInTheOrderTheyComeApart` and
  `TestTheTeardownJobIsBuiltFromTheRecordAlone`.
- A failed cleanup keeps a record with the exact surviving location —
  **met** at the data level; `Strand`/`Release` write held mounts in.
- Honesty: the commit says outright "Unmeasured, and all of it: the kill
  matrix the review asks for needs a disposable machine." Deferral
  recorded.

### 4 — P1, fsx bases and Scratch permitted repository writes

Commit `12891bb`. An `Area` now holds a `pathx.Root` (`fsx.go:57`), a
descriptor resolved once and never re-named. `Scratch` (`fsx.go:197`)
uses `/tmp` and never `os.TempDir()`, so ambient `TMPDIR` cannot aim the
probe at a repository; its cleanup returns failure. The state-in-repo
guard asks `state.Location()` (`state.go:295`) — where the records really
land through the pinned base — rather than comparing the spelled path
(`frontend.go:recordsOutsideRepositories`, `:66`).

- Base-symlink / base-rename / state-alias / `TMPDIR=repository` leave
  the repository identical — **met by construction, partly tested.**
  `fsx/root_test.go` and `fsx/scratch_test.go` cover the base-swap and
  the `/tmp` base; the rename-after-validation barrier is not
  measurable here.

### 5 — P1, the strict mountinfo reader rejected legal host state

Commit `c21f5a7`. Split is on `0x20` only (`mountinfo.go:parse`,
`:170`), two spaces are an empty field and a refusal, the post-`-`
cardinality is **exactly three** in both directions (`:207`), and the
1 MiB scanner cap is gone in favour of `bufio.Reader.ReadString('\n')`
(`mountinfo.go:Read`, `:107`), which grows to whatever the kernel wrote
and does not strip a trailing `\r`. Doctor reads the table before the
configuration, says `mount-table-unreadable`, omits the whole
environment section and exits non-zero (`cli.go:cmdDoctor`, `:345`;
`unreadableTable`, `:410`; `incomplete`, `:437`).

- Every legal-byte fixture parses, every malformed one fails — **met and
  tested.** Reverting the split to `strings.Fields` reddens the raw-`\r`
  and the literal-space fixtures; loosening the `!= 3` guard to `> 3`
  reddens the two-fields-after-`-` case.
- No supported line rejected for size — **met** (no cap remains). The
  max-layer measurement is still open and the commit says so; the "no
  number in the code" claim is true, so the uncertainty is real only in
  the sense that no kernel line was proven longer than what parses. Not a
  defect.
- All commands report a table failure consistently — **met**; doctor's
  non-zero exit is checked by `TestDoctorSaysWhenTheMountTableCannotBeRead`
  (reverting `incomplete` to always-nil reddens it).
- The commit is explicit that two measurements are still missing (a real
  mount carrying raw `\r`/`\v`/`\f`, and a maximum-layer line) and
  neither is faked. Honest.

The escape-set argument for "exactly three" holds against
`constraints.md`: `mangle()`/`seq_show_option` escape space in fstype,
source and the super-option value alike, and the super-option field is
never empty (the kernel writes at least `ro`/`rw`), so a legal host is
not refused. I concur with the grammar.

### 6 — P1, ambient Git variables bypassed the tracked-content guard

Commit `e51e139`. `gitwire.environment` (`gitwire.go:236`) is now an
allowlist — `PATH`, `HOME`, `XDG_CONFIG_HOME`, plus camp's own
`LC_ALL=C`/`LANGUAGE=C` — built, not inherited. `Open` looks once more
before believing git's "not a repository": a `.git` entry in the frame
means `Unreadable` (`gitwire.go:gitSaidNo`, `:150`). Both `TracksUnder`
error sites now propagate as `git-unreadable`
(`plan/sequence.go:289`, `gen/prepare.go:353`).

- Ambient Git variables cannot change which repository is inspected —
  **met and tested.** Reverting `environment()` to `os.Environ()`
  reddens `TestAmbientGitVariablesCannotChangeWhatGitIsAskedAbout` and
  `TestAnAmbientGitVariableCannotUnlockAMountOverTrackedCode`.
- A damaged repository stops both `plan` and `up` with `git-unreadable`
  — **met** at unit level.

On the design question the author flagged: keeping `HOME` and
`XDG_CONFIG_HOME` is defensible. The injection vectors —
`GIT_DIR`/`GIT_WORK_TREE`/`GIT_INDEX_FILE`/`GIT_CONFIG_*` and the loader
family — are all dropped. What `HOME` exposes (`include.path`,
`core.fsmonitor`) is the user's own global git config; an attacker who
can write it already runs code as that user, and camp's read runs
unprivileged before any elevation. The commit's reasoning ("camp's answer
should be the answer the repository's owner's own git gives") is sound.
No finding.

The spec-vs-code disagreement (§19 says core carries no git; `plan`,
`drift`, `health` import `gitwire`) is now **named** in the source
(`gen/exclude.go` package doc, `gitwire.go` package doc) rather than
contradicted. That is the honest half of the repair the review asked for;
which way to settle it is left to the owner, correctly.

### 7 — P1, an allow-listed root mount target was omitted from the exclude

Commit `f34045f`. The mount-target loop in `ExcludeLines`
(`gen/exclude.go`, ~`:150`) no longer skips a one-component target whose
name is allow-listed; every non-root live target goes in.

- Every live mount target appears regardless of `allow_overlap`;
  ordering/escaping deterministic — **met and tested against git.**
  Restoring the allow-list exception reddens
  `TestAMountTargetIsExcludedEvenWhenItsNameIsAllowListed`, whose fixture
  puts the payload into a scratch repo's `info/exclude` and asks
  `check-ignore`/`add -n`, not just the renderer.

### 8 — P2, fsx folded ENOTDIR into absence

Commit `bf333fb`. `isAbsent` (`fsx.go:987`) is now `ENOENT` only; every
type transition is answered where it arises with `ErrChangedType`;
`removeTreeAt` retries to a bound, `chownTreeAt` refuses (`fsx.go:778`),
`Remove` splits the two unlink flags (`fsx.go:606`).

- No destructive/ownership op returns success because an object changed
  type — **met and tested.** Restoring `ENOTDIR` to `isAbsent` reddens
  four tests in `typerace_internal_test.go`
  (`...ADirectoryGoneWhenAFileTookItsPlace`,
  `...RefusesANameThatKeepsChangingType`,
  `...ChownRefusesADirectoryReplacedByAFile`,
  `...AnAreaWhoseNameIsAFile...`).

### 9 — P2, the atomic writer shared one temporary filename

Commit `bf333fb`. `createTemporary` (`fsx.go:559`) makes a per-call,
unpredictable, `O_EXCL` temporary; cleanup unlinks only what this call
made (`fsx.go:502`). Above the bytes, a record transition is serialized
on camp's own record directory via `state.hold()` (`state.go:439`).

- The two-writer schedule cannot mutate a successful publication and
  never produces hybrid bytes — **met and tested from two processes.**
  Reverting to the fixed `.<name>.camp` + `O_TRUNC` temporary reddens
  `TestTwoProcessesReplacingOneName`.

### 10 — P2, report publication and show-once delivery were not atomic

Commit `bf333fb`. `Write` publishes name-and-body together with
`RENAME_NOREPLACE` (`reports.go:Write`, `:74`; `fsx.WriteNew`); `Show`
holds an exclusive flock on the reports directory across list/read/print/
mark (`reports.go:231`); `MarkSeen` uses `RenameNew` and reports
`ErrAlreadySeen` to the loser; `fsx.Rename` no longer folds a missing
source into success (`fsx.go:405`), and log rotation states its own
expected absence.

- Concurrent commands deliver one report once; failed publication
  exposes no final file; the loser can tell it lost — **met and tested.**
  Removing the `hold()` from `Show` reddens
  `TestTwoProcessesDeliverOneReportOnce`. (See new risk 3 for the cost of
  that lock.)

### 11 — P2, always-on logging and two-process rotation had gaps

Commits `3da51cb`, `bf333fb`. The log attaches from the found config path
before the file is parsed (`cli.go:keepUnder`, `:90`; `resolve`, `:253`),
with `keepUnder` restricted to camp's own layout; both flock directions
are checked and rotation goes off with one warning if locking is absent
(`logs.go:hold`/`drop`, `:257`/`:281`); the namespace init routes its own
`logs.Open` failure through the same one-time complaint
(`session.go:keepLog`, `:229`); directory and file come from one pinned
Area.

- Malformed-config / init-open-failure / unsupported-lock each emit one
  warning or retained refusal — **met and tested.** Removing `keepUnder`
  from `resolve` reddens `TestARefusedConfigurationIsKeptInTheEnvironmentsLog`;
  reverting the flock handling reddens
  `TestWithoutInterprocessLocksNothingRotatesAndNothingIsLost`,
  `TestAnUnlockThatFailsAlsoStopsTheRotation` and
  `TestTheSentenceAboutTheLogReachesTheLogItself`.

### 12 — P2, recorded overlay options were a dead, independently rendered assertion

Commit `de2ce6f`. `mountx.DescribeOverlay` (`mountx.go:196`) is the one
derivation; the mount is performed from it (`fill`/`overlayMount`), the
record persists its steps as `Operands` (`state.go:Overlay`, `:776`), the
option line renders from it (`OverlayConfig.Options`), verification
compares against it (`verify.go:overlayOptions`, `:255`), `camp plan`
prints it, and `camp status` compares the recorded operands against the
mounted overlay and **exits non-zero on drift** (`recover.go:664`,
`:703`). A source guard refuses an `fsconfig` call written outside the two
seams.

- An intentional one-sided change to any operand or flag fails a test,
  and every persisted option field has a production consumer — **met and
  tested.** Reversing the lower order in `DescribeOverlay` alone reddens
  three `mountx` tests plus one in `verify` (`camp status`'s consumer is
  the new drift check).

## What the repairs broke, or newly risk

I looked hardest for another instance of the `525093b` shape — a test
that drives real code which self-execs `os.Executable()` and recurses
inside a namespace. The three self-exec sites are
`preflight.probeUserNamespace` (`preflight.go:445`),
`frontend.run` (the sudo helper, `frontend.go:554`) and the session
re-exec (`session.go:134`). Each test driver that could reach one is
guarded: `internal/cli` `TestMain` answers `ProbeArg`
(`cli_test.go:38`), `internal/session` `TestMain` answers `InitArg`
(`session_test.go:33`), and the two `down` tests deliberately stop before
the helper (no record → "no record for", `recover_test.go:111`,
`cli_test.go:117`), so `frontend.run` is never reached. `internal/preflight`
has no test files, so nothing else drives the probe. **No second
instance of the shape was found.** The fix is complete for the class.

The repairs do introduce three risks of their own:

### R1 — a record can become permanently undiscardable, and the message does not name the way out

`Held` (`state.go:924`) reports a record's recorded targets by exact
mount point (`At`) but the work, staging and **live** trees by prefix
(`Under`). `Release` (`state.go:969`) keeps the record and re-strands
whatever `Held` finds. Both `camp down` (`cli/privileged.go:312`) and
`camp forget` (`cli/inspect.go:253`) go through `Release`. So an
unrelated mount that lands *beneath the live directory* after camp's tree
came down — a user bind-mounting something of their own inside what used
to be the composed tree, machine-wide in this mode — makes `down` and
`forget` keep returning `ExitBusy`/`ExitPrecondition` forever. The
distinction from a recorded target (`At`, so an unrelated mount inside
the workspace does *not* trap it) is deliberate and documented at
`state.go:917`; the live tree using `Under` is defensible (it is camp's
own directory). The gap is the exit: `Release`'s message
(`state.go:984`) says "'camp status' says what is there and 'camp down'
removes what camp made; the record goes when the last of it does" — but
`down` cannot remove a mount that is not camp's, and the message never
says the user may unmount their own mount or `rm` the record file. The
record is an ordinary user-owned file, so the way out exists (invariant 5
is not violated); it is simply not signposted, and every `down` in the
meantime is non-zero. **Severity: low-to-medium.** Settle: either exclude
non-camp mounts under `live` from the discard gate, or add the two exits
to the message. This is new on the branch (before `656fd67`, `forget`
just removed the file).

### R2 — record fields grew without a version bump; an older build gives a parser error, not the version sentence

The branch adds `Mount.Staging`, `Mount.Operands` and `Record.Stranded`
while `Version` stays `1` (`state.go:69`). Forward (old record → new
build) decodes cleanly — I confirmed it: an `8ddf464`-shaped record
(`fstype:"overlay"`, no `operands`, no per-mount `staging`) decodes on
`HEAD` and produces no `OverlayDrift`, because `OverlayDrift` early-exits
only when `FSType==""` *and* `Operands` is empty (`state.go:801`), and an
old record's empty `Steps` yields no `Mismatches`. So the commit's claim
"records written by the previous build decode unchanged and produce no
disagreement" is **true** — but the author is right that no committed
fixture asserts it (`state_test.go` builds records with the current
struct). Reverse (new record → old build) I also ran, in a clone at
`e648a7f`: it is **rejected** with `the record does not parse: unknown
field "staging"`, not the intended "written by a different build; use
that build to take the composition down" (`state.go:543`). Because
`Version` did not move, the version guard never fires. **Severity: low**
— it only bites on a downgrade between `up` and `down`, which the
project's own convention already discourages. Settle: bump `Version` to
`2` (the new-build reader accepts a v1 record for read; the old-build
reader then says the right sentence), or add a v1 fixture to lock the
forward direction. Either closes the author's stated gap.

### R3 — `reports.Show` holds a directory flock across terminal output

`Show` (`reports.go:231`) takes `LOCK_EX` on the reports directory and
holds it across the whole delivery, including the `out()` callback, which
in `resolve` writes to the sink and thus to the terminal and the log
file. A camp whose stderr is blocked (piped to a reader that stopped
consuming) holds the reports lock while blocked, and `Show` runs at the
start of every command that resolves a composition — so the next camp in
that environment blocks at startup with no deadline. It is bounded by
process life (flock drops on death) and stderr-to-a-terminal does not
normally wedge, so this is a **low-severity availability nit**, not a
deadlock. I checked for an actual lock cycle and found none: the only
nesting is reports⊃logs (the `out()` write takes the logs flock), always
in that order; `state.hold` locks a third directory and is never nested
with either while writing. Settle, if at all: read and mark under the
lock, print outside it.

## Do the tests have teeth

Yes, on every finding I could exercise without a mount. Restored the old
behaviour in a scratch clone and confirmed red for:

- mountinfo split → `strings.Fields` (raw-`\r`, literal-space fixtures);
  `!= 3` → `> 3` (two-fields-after-`-`).
- exclude allow-list exception restored (`...NameIsAllowListed`).
- fsx fixed shared temporary (`TestTwoProcessesReplacingOneName`).
- gitwire `os.Environ()` inheritance (two tests, `gitwire` and `gen`).
- state `Teardown` staging-half removed (two `privileged` tests).
- `DescribeOverlay` lower order reversed (three `mountx` + one `verify`).
- fsx `chmodFd` → name-based `Fchmodat` (`...NamesItsObject`,
  build-guard).
- fsx `isAbsent` re-folding `ENOTDIR` (four `typerace` tests).
- `init` base → `$ENV/.camp` (init test).
- doctor `incomplete` → always-nil (`...MountTableCannotBeRead`).
- reports `Show` lock removed (`...DeliverOneReportOnce`).
- `keepUnder` removed from `resolve` (`...KeptInTheEnvironmentsLog`).
- state `Release` → unconditional forget
  (`...KeptWhileAnythingItAnswersForIsMounted`).
- logs flock both directions ignored (three `logs` tests).
- `OpenRootExactly` `O_NOFOLLOW` dropped (two `privileged` tests).
- `checkable` `TargetAbsent`/`SourceType` enforcement loosened
  (`...OperandItCannotCheck`).

Every reversion reddened the test the repair added. None of these tests
is decoration. The one class I could not exercise is the mount path:
`TestTheHelpersDescriptorMountCompletes` (`compose_test.go:758`) and the
`SkipIfItCouldMount` unit tests only carry weight inside the installed
namespace, which is `remaining-checks.md`'s job.

## Are the comments true

I found no comment claiming a measurement nobody made. The load-bearing
ones are careful in exactly the places they have to be: `mountx.go`'s
package doc says the descriptor mount path "has never been run" and names
the one narrow thing that *is* measured (the `open_tree` EPERM shape);
`c21f5a7` states which two mountinfo measurements are still missing and
that "neither is faked"; `656fd67` says the kill matrix is "Unmeasured,
and all of it"; `b304db0` marks the two residual by-name syscalls where
they happen. The `de2ce6f` claim that a previous-build record decodes
unchanged is true (I ran it), though — per R2 — unfixtured. The seven
test seams (`doctorTable`, `afterTypeCheck`, `afterTemporaryOpen`,
`failStep`, `afterRead`, `flock`, `fsconfigFd`/`fsconfigFlag`) each
default to the real production value and are only reassigned in tests, so
none can change production behaviour; each says why it exists. That is a
reasonable price.

## The places the author asked to be read hardest

- **Descriptor mount path.** The sequence is right against the documented
  contract: `open_tree(OPEN_TREE_CLONE|AT_EMPTY_PATH)` copies the one
  mount `MS_BIND` makes (not `AT_RECURSIVE`), `move_mount` attaches it to
  the target descriptor, and the RO remount and `MS_PRIVATE` go through
  the clone's `/proc/self/fd/N`, which after the attach names the mount
  and not the object underneath. The not-`mount_setattr` argument is
  sound: it would set RO on the detached clone (strictly better, no
  writable window) but is Linux 5.12 against the 5.2 floor `open_tree`/
  `move_mount`/`fsopen` share, and camp already hard-requires `fsopen`;
  raising the floor needs a measurement there is none of. The safety net
  is real — verification runs after the mounts, at the path, in staging
  and again at live, so a remount or propagation change that did not take
  comes back as a refusal and a rollback. Nothing else in camp assumes
  the old by-name shape except the two acknowledged teardown-by-name
  residuals. The one thing I cannot give you is proof it runs: it is
  unrun, by design, and the first real `up` proves the whole of it at
  once.
- **Seven test seams.** Reasonable, and none can change production
  behaviour (see above).
- **Two new flocks / deadlock.** No cycle: reports⊃logs is the only
  nesting and is consistently ordered; `state.hold` locks a third
  directory and its transition contains no helper and no prompt
  (`state.go:429`), so it is a local read+write. A wedged process blocks
  another camp only in the R3 sense (reports lock across output), bounded
  by process life; the state lock is not held across anything that can
  wedge.
- **`state.Release` undiscardable record.** Confirmed as R1: an unrelated
  mount beneath `live` traps the record, `forget` shares the gate, and
  the way out is real but unsignposted.
- **mountinfo grammar.** Safe for every legal host (see finding 5).
- **git subprocess allowlist.** Defensible (see finding 6).
- **`camp status` non-zero on drift.** The right change, and an old
  record is safe from false positives — confirmed by reading and by
  decoding an `8ddf464`-shaped record on `HEAD` (finding 12, R2).
- **Record grew fields without a version bump.** R2: forward decodes and
  is correct; reverse is rejected with a parser error instead of the
  version sentence; no committed fixture for the old-record case.

## Verification performed

- `go build ./...` — clean.
- `go vet ./...` — clean.
- `CAMP_TEST_ROOT=/tmp/.../scratchpad/tr-fable go test ./...` — every
  package passes, nothing skipped in the report.
- Twenty single-line reversions in a scratch clone
  (`/tmp/.../scratchpad/teeth`, at `525093b`), each confirming the
  matching new test goes red; listed above.
- Cross-build record decode: an old-shaped record decodes on `HEAD`
  (ad-hoc `Decode` test, passed); a new-shaped record is rejected on a
  clone at `e648a7f` with `unknown field "staging"`.

No privileged, mount, sudo or namespace operation was run. The
install-gated and terminal-gated adversarial measurements in
`remaining-checks.md` — the descriptor mount, the kill matrix, the rename
race — were deliberately not attempted here; they are what the merge
decision waits on.

## Verdict

The branch answers all twelve findings with the right shape, clean build
and vet, a passing suite, and a real test behind every P2 and every
unprivileged-testable P0/P1 criterion. It is a careful piece of work and
its comments are honest about what is unproven.

It is **not yet safe to merge on the strength of this review alone**,
because the criteria that matter most — the P0/P1 adversarial mount,
rollback and rename-race tests — cannot run in this environment and are
proven here by argument, not execution. What would have to be true first:

1. The install-gated privileged matrix in `remaining-checks.md` runs and
   converges — the descriptor mount completes and comes back RO and
   private, the kill matrix recovers from the record alone at every
   boundary, and the rename-race barriers refuse. That is the evidence
   this review structurally cannot provide.
2. R1 is decided — exclude non-camp mounts under `live` from the discard
   gate, or name the two exits in the message — so a record cannot be
   walled shut with no signposted way out.
3. R2 is closed cheaply — bump the record `Version`, or add a v1 fixture
   — so a downgrade fails with the right sentence and the forward-decode
   claim is locked by a test.

R3 is optional. With (1) measured and (2)–(3) done, I would call it safe.
