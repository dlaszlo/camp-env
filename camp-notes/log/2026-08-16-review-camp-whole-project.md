# Whole-project review: camp

Date: 2026-08-16

Reviewed snapshots:

- camp: f7010f185ca397ff03b368f0b97cc9448a86e5e1
  (Scan what this repository tracks, not what is mounted over it)
- camp-notes: 60d9b59bd0ae624bfb84d3df1c19366ee03edfa3
  (Record that the namespace group has actually run)

This is a review of the repository as a whole, not of a diff. The reviewed Go
tree is 20,500 lines: 14,017 production lines and 6,483 test lines across 30
packages.

## Judgment

I would not release or use the privileged mode in this snapshot. Its first
verification pass rejects every honest plan before the move, and its sudoed
helper is a root confused deputy: anyone allowed to invoke the documented
helper command can supply arbitrary unmount paths and an arbitrary tree to
recursively chown.

The namespace mode has substantially better executable evidence, including
the owner's installed-binary run with no skips, but it is not yet entitled to
the repository-safety guarantees in the README. The fsx boundary is lexical
rather than filesystem-confined, allow-listed directories have neither
read-only guards nor complete excludes, a mutable configuration is rebuilt
under locks for a different plan, and the specification explicitly accepts a
mid-session new-root-entry window that contradicts its own non-negotiable
guarantees.

The findings below are defects, not preferences. P0 blocks use or release; P1
can violate a core invariant, corrupt recovery, or execute a materially
different plan; P2 is a correctness or failure-honesty defect with a narrower
trigger. A final P3 section contains maintenance defects under the project's
explicit no-abstraction-without-a-caller rule.

## P0 — release blockers

### CAMP-REVIEW-001 — privileged up cannot pass staging verification

Location: camp/internal/privileged/helper.go:146-172;
camp/internal/verify/verify.go:259-263,287-327,337-354.

verifyStaging constructs plan.Plan with only Live and Mounts. It drops Config,
Storage, and the validated Exclude payload. It also marks every reconstructed
mount InLive=false. Completeness therefore looks for the workspace self-bind
at built.Config.LowerPath(), which is the empty string, while the planned set
still contains the real freeze-lower target. Every honest privileged plan has
that mount, so the pass reports it missing and rolls the whole staging tree
back before mountx.Move. The exclude and storage checks are silently disabled
at the same time.

This is not merely incomplete verification: it makes the primary privileged
success path unreachable. The lack of a real helper lifecycle test allowed it
through.

Normative impact: spec §14 at 1466-1475 and §15 at 1530-1566 require the same
complete pass in staging and at live. It also violates the rule that a
protection which looks installed may not do nothing.

Exact repair; implementation's move: put the complete concrete verification
input in the mount job (resolved Config/lower identity, Storage, expected
exclude bytes or a digest plus pinned payload), reconstruct it without losing
fields, and retain the original InLive semantics so verify.at remaps only the
tree-local targets. Add a privileged staging test that includes the
freeze-lower mount, generated exclude, storage ownership, move, and second
pass. The test must fail on this implementation before the fix.

### CAMP-REVIEW-002 — the sudoed helper accepts arbitrary root operations

Location: camp/cmd/camp/main.go:33-42;
camp/internal/privileged/helper.go:27-73,262-291;
camp/internal/privileged/job.go:55-116;
camp/internal/fsx/fsx.go:255-320.

The hidden helper commands are directly dispatchable. Helper performs ordinary
json.Unmarshal and validates only version and action. For unmount, Targets are
arbitrary absolute paths. Base, WorkParts, UID, and GID are also arbitrary.
After the unmount loop, a job such as Base="/", WorkParts=["etc"] makes the
root process remove /etc/work if present and recursively lchown /etc to the
supplied ids. WorkParts=["."] expands the reach further. Targets can ask root
to unmount any mount the caller names.

The helper therefore does not execute only a validated camp plan and does not
trust nothing it was handed. It is a general root mutation primitive behind
the exact sudo entry point described as narrow. Cached sudo credentials or a
NOPASSWD rule make configured or otherwise user-level code able to invoke it
without another human decision.

Normative impact: invariant 6 in spec §3; spec §14 at 1414-1449; §22 stage 4.
C12 is the measured reason operands need containment, but this finding is
broader than a symlink race.

Exact repair; implementation and packaging's move: use a strict decoder
(unknown fields and trailing data refused), validate an action-specific
schema, bind UID/GID to the real sudo invoker, and descriptor-confine Base,
Targets, staging, live, and work to one authenticated composition. The helper
must derive teardown operands from an authenticated, complete record or from
descriptor-relative components whose expected identities are supplied and
checked; it must never accept raw absolute unmount targets. Stable refusals
such as helper-job-invalid and helper-target-outside need tests. Add hostile
jobs proving /, /etc, another mount, path aliases, dot components, missing
identities, and foreign uid/gid are rejected before any syscall.

## P1 — invariant, race, and recovery defects

### CAMP-REVIEW-003 — fsx does not confine writes to camp-owned areas

Location: camp/internal/fsx/fsx.go:46-62,69-79,96-145,165-229,243-320;
camp/internal/compose/compose.go:65-109;
camp/internal/fsx/writesites_test.go:14-25,42-98.

Area constructors accept any raw string. Area.Path rejects lexical dot and
slash components but ordinary MkdirAll, OpenFile, CreateTemp, Rename, Chmod,
WalkDir, and recursive removal follow the root and intermediate filesystem
components. A symlink at .camp/work, .camp/storage, or a descendant can direct
runtime writes and removals into a repository. XDG_STATE_HOME may also simply
name a repository; state then writes there by construction. The assertion
that no Area can be constructed from a repository path is false in both the
comments and the source guard. That test only proves where write calls are
spelled, not where their targets resolve.

Impact: camp can create, replace, chmod, chown, or remove repository content
while the source-level guard remains green. This is precisely the project's
highest-level invariant.

Normative impact: invariant 1 in spec §3 at 137-142; the no-symlink path rule
in §6 at 572-581; the source-property acceptance in §22 at 1958-1961; C12.

Exact repair; implementation's move: replace raw Area constructors with typed
constructors that receive already validated non-repository roots and pin them
by descriptor. Perform all traversal and writes beneath those descriptors
with no-follow/beneath resolution; WriteAtomic must take a pinned directory
and one name, not an arbitrary path. Reject roots or intermediate components
that are symlinks, aliases of a repository, or nested in one, under a stable
write-root-unsafe rule. Add executable tests for a pre-existing symlink, a
swap between validation and write, and XDG_STATE_HOME inside the code and
workspace repositories.

### CAMP-REVIEW-004 — privileged mounting still uses unpinned overlay and move operands

Location: camp/internal/privileged/frontend.go:223-279;
camp/internal/privileged/helper.go:127-141,175-215;
camp/internal/privileged/job.go:91-115;
camp/internal/mountx/mountx.go:90-122,224-237,266-276.

Bind endpoints are opened, but overlay Lower, Upper, and Work are sent and
used as raw option strings with no identities or descriptors. The staging and
live paths used by MS_MOVE are raw strings too. A target identity failure in
MountJob is silently converted into an empty expected identity, and
checkIdentity explicitly accepts an empty value. SourceType is serialized but
never enforced.

A user-owned component can therefore be replaced after front-end validation.
For overlay operands, the helper mounts the replacement. For the final move,
the live directory can be swapped before root executes MS_MOVE. Verification
can validate a stable attacker-chosen replacement and still say the checked
object is the mounted object.

Normative impact: spec §6 at 572-581 and §14 at 1436-1449; §22's required
rename/symlink race; C12.

Exact repair; implementation's move: require identities for every endpoint;
open and pin lower, upper, work, staging, live, every bind endpoint, and every
intermediate base before the first mount. Render overlay options from pinned
/proc/self/fd references or use the new mount API, and move onto a pinned live
descriptor. Any missing identity or changed object must refuse with
helper-operand-changed. Exercise swaps independently for each operand,
including live between first verification and move.

### CAMP-REVIEW-005 — allow-listed directories are writable and stageable holes

Location: camp/internal/plan/gate.go:42-60,69-107;
camp/internal/plan/build.go:117-149;
camp/internal/gen/exclude.go:63-107;
camp/internal/plan/plan_test.go:181-199.

The gate recursively permits an allow-listed directory when the two sides
have no same-named descendants. Build then skips the entire root directory
from read-only guards, and ExcludeLines skips the entire root directory from
the exclude. For example, lower shared/env.md plus upper shared/code.md with
allow_overlap: [shared] passes. Editing shared/env.md copies it into the code
repository, and git add . can stage it because no /shared/env.md pattern was
generated.

The existing gate test checks only a collision and explicitly treats a
one-sided child as irrelevant; it never tests writability or staging. The
implementation also omits the one location where the specification mandates
file enumeration.

Normative impact: invariants 1-2; spec §7 at 797-814; §9 at 888-891; §10 at
978-984; C1, C4, and C25.

Exact repair; implementation's move: the safest current repair is to refuse an
allow-listed directory with a stable allow-overlap-directory rule. If the
feature must remain, recursively derive read-only binds for lower-only
subtrees, continue through directory/directory merges, and emit byte-sorted
exclude lines for every lower-contributed leaf/subtree exactly as §10
requires. Test lower-only files and directories, new descendants, writes,
deletes, git add ., and same-name collisions.

### CAMP-REVIEW-006 — a failed multi-syscall mount is omitted from rollback

Location: camp/internal/mountx/mountx.go:60-87,99-122;
camp/internal/compose/compose.go:179-198;
camp/internal/privileged/helper.go:93-122.

A bind or overlay can mount successfully and then fail during the read-only
remount or MS_PRIVATE operation. Mount returns only an error. Both callers add
the target to their rollback list only after Mount or MountByDescriptor
returns nil, so the newly created mount is absent from rollback. compose.Build
then says “Nothing is left mounted,” which is false; in the privileged mode
the residue is machine-wide. A read-only bind can specifically remain as a
writable bind after the remount failure.

Normative impact: spec §7 at 745-746, §15 at 1539-1549 and 1563-1566; C11 and
C14; the rule that a failure must say what it leaves behind.

Exact repair; implementation's move: make each mount operation transactional
and synchronously unmount itself if a post-mount step fails, or return a
structured result saying that the initial mount exists so the caller records
it before proceeding. If cleanup fails, return a partial result and never
print the clean-rollback sentence. Add injected failures after the first
mount, after remount, and before/after MS_PRIVATE in both execution paths.

### CAMP-REVIEW-007 — the persisted record cannot drive status, explain, or config-free down

Location: camp/internal/state/state.go:66-119,138-170;
camp/internal/cli/inspect.go:23-44,98-153,222-287;
camp/internal/cli/privileged.go:90-107.

Record defines Options and Created, but FromPlan never populates either and
does not retain enough Plan structure to reconstruct verification. status
loads the current configuration, always prepares a Namespace plan, ignores
state.Load's error, omits the expected Exclude payload, and verifies a
privileged nouserxattr mount against a userxattr plan. explain also describes
the current config and discards generator-preview errors. down says it uses
the record, but it must first find and parse enough of the current config and
derive the same live hash. With the config deleted, resolveForTeardown fails
before a record is selected.

The test named TestDownConvergesFromTheRecordWithTheConfigurationDeleted only
calls state.Load and Record.Teardown; it never invokes the CLI or helper.

Normative impact: spec §12 at 1148-1174; §15 at 1568-1571; §22's kill-point
matrix.

Exact repair; implementation's move: persist every field listed by §12,
including overlay options, expected types, generated artefact bytes/digests,
created paths, mode, and mount identities; provide Plan reconstruction from
that schema. Select a record independently of configuration, using an
explicit record/hash or live argument plus deterministic current-directory
matching with ambiguity refusal. status, explain, and down must then consume
only the record, reporting current config drift separately. Add CLI-level
tests with the config deleted and changed, for up/mounting/partial records.

### CAMP-REVIEW-008 — records are accepted without schema invariants and teardown ignores mount identity

Location: camp/internal/state/state.go:197-224,236-263,272-295;
camp/internal/privileged/frontend.go:188-213;
camp/internal/privileged/helper.go:262-291.

Decode accepts unknown fields, trailing/missing fields, arbitrary phases,
empty hashes, non-canonical paths, and incomplete mounts. State.All turns an
unreadable state directory into an empty list. Although records contain
device/inode fields, UnmountJob sends only raw target paths and the helper
unmounts those paths without checking the recorded identity or filesystem
type. If a camp mount disappeared and an unrelated mount later occupied the
same name, camp down would unmount the unrelated mount. A syntactically valid
but empty plan can also be accepted as a successful teardown.

Normative impact: invariant 6; spec §12 at 1148-1170 requires a complete plan
and path-plus-identity recovery.

Exact repair; implementation's move: decode exactly one strict JSON object,
reject unknown fields, and validate all required fields, phase transitions,
canonical containment, filename/hash agreement, mount kinds/options/types,
unique targets, and identities by action. Send expected identity and fstype
to the helper and compare the current target/mountinfo entry before every
unmount. A mismatch must remain recorded and fail under stable record-invalid
or record-mount-mismatch rules; it must never be treated as absent.

### CAMP-REVIEW-009 — islands without a generation step skip required preparation and still call git

Location: camp/internal/gen/prepare.go:62-109,176-217;
camp/internal/gen/run.go:63-95;
camp/internal/gen/gen_test.go:338-352.

Prepare immediately returns withoutAStep. That branch computes an expansion
but never runs expandedChecks, never creates scaffold attachment points, and
never writes other required preparation output. A first-run file island has
no placeholder, so the later bind fails C10. The same branch says it uses a
raw listing but calls contributed, which opens git and uses tracked content
when the source is a repository. Thus a configuration with no generation step
still has git knowledge and does not implement its documented fallback.

The test asserts only an empty exclude and a note. It never checks the island
set, an untracked source entry, placeholders, mounts, or a second run.

Normative impact: spec §6 at 523-525; §11 at 1002-1009; §19 at 1832-1836; C10.

Exact repair; implementation's move: implement no-generation as an explicit
raw ReadDirBeneath path, validate the expanded plan, create and persist the
same scaffold, and only then return the output. It must not call git at all.
Add first- and second-run tests proving an existing untracked entry becomes an
island, file and directory targets exist, and the real composition mounts.

### CAMP-REVIEW-010 — a custom generator may omit tracked islands or add existing untracked ones

Location: camp/internal/gen/prepare.go:145-173;
camp/internal/gen/validate.go:25-113;
camp/internal/gen/gen_test.go:193-259.

Adopt computes the independently expected output, but Validate compares only
the exclude bytes. For islands it validates the syntax, existence, and type
of whatever the generator chose to emit; it never compares that set with what
the source repository actually contributes. Omitting a tracked entry silently
turns that path into writable water. Adding an existing untracked runtime file
passes because it exists, even though the repository does not contribute it,
and can steer a later root mount.

The hostile-output test covers an invented absent name but not an existing
untracked name, an omitted tracked name, or an omitted islands target.

Normative impact: spec §11 at 1002-1009 and 1041-1045; §19 at 1878-1895; C26
when an omission changes what code sees.

Exact repair; implementation's move: compare the decoded, typed, byte-sorted
set for every target exactly with the independently derived contribution set.
Use stable generate-islands-missing and generate-islands-extra refusals that
name the target and both sets. Test omission, existing-untracked addition,
missing target, extra target, and type change.

### CAMP-REVIEW-011 — git operational failures are repeatedly treated as “not a repository” or “nothing tracked”

Location: camp/internal/gitwire/gitwire.go:70-97;
camp/internal/plan/validate.go:68-84;
camp/internal/plan/sequence.go:182-209;
camp/internal/gen/prepare.go:203-217;
camp/internal/gen/run.go:74-95;
camp/internal/drift/drift.go:70-85;
camp/internal/health/health.go:198-207.

gitwire.Open maps every rev-parse error to isGit=false. Callers cannot
distinguish a plain directory from missing git, permission failure, a corrupt
repository, killed git, or an I/O error. checkTracked also converts a
TracksUnder error into “tracks nothing”; expandedChecks does the same in its
callback. Generation can fall back to a raw listing, while drift and doctor
silently omit their git checks.

This is a fail-open path around the rule that a mount may not cover tracked
code. It can turn C26's commit-a deletion into an accepted plan and can
suppress the end-of-session warning that should reveal it.

Normative impact: invariant 6; spec §15, §16, §17, and §19; C26.

Exact repair; implementation's move: make Open return repository / definitely
not a working tree / operational error, with only git's specific
not-a-work-tree result mapped to the middle state. Carry errors through every
tracked-content callback. Planning and generation must refuse with a stable
git-unreadable rule; reporting paths must append the failure to the report,
not report an empty result.

### CAMP-REVIEW-012 — namespace init rebuilds a mutable plan while holding locks for the old one

Location: camp/internal/cli/session.go:57-72;
camp/internal/session/session.go:92-103,118-138,235-246,343-376;
camp/internal/compose/compose.go:187-198;
camp/internal/mountx/mountx.go:60-87.

The launcher validates Plan and Exclude and passes them in session.Options,
but Launch never reads those fields. It sends only the mutable config path to
the init. The init adopts the old upper/live lock descriptors, reloads the
file, and builds a new plan without checking that the new upper and live are
the inodes those descriptors lock. Changing the config between these moments
can therefore mount upper B and live B while camp holds locks for upper A and
live A, defeating C8.

Even without a config edit, the namespace execution path validates
descriptor-relative operands but later mounts raw path strings. A same-user
process or a generator-descended process can swap a component between check
and mount.

Normative impact: spec §7 at 751-777; §13 at 1197-1233; namespace mode in §14;
C8 and C12.

Exact repair; implementation's move: send an immutable, validated plan and
exclude to init over a sealed memfd or private pipe, or send a digest and
refuse any byte change before use. Compare the plan's upper/live identities
with fstat of the inherited lock fds. Carry pinned descriptors through the
actual mounts, not only validation. Remove the currently dead Options fields
only if this design is deliberately abandoned. Add config-swap and
path-component-swap races that assert no mount occurs under the wrong locks.

### CAMP-REVIEW-013 — privileged down can report success and forget state after cleanup failed

Location: camp/internal/privileged/helper.go:278-290;
camp/internal/privileged/frontend.go:173-185,307-338;
camp/internal/cli/privileged.go:107-162.

The helper puts work removal/chown failures in Reply.Error and exits nonzero.
run intentionally decodes a structured reply even for that exit. Down returns
the reply without turning Reply.Error into a refusal, and the CLI looks only
at Stranded. With no stranded mounts it ignores record.Save, Residue, and
state.Forget errors, prints success, marks the record down, and forgets it
even though privileged cleanup failed or the durable state transition did not
happen. Up rollback similarly discards several Save/Forget errors.

Normative impact: invariant 4; spec §12's durable phases and §16's truthful
partial teardown.

Exact repair; implementation's move: make Reply.Error and every durable
state/cleanup error part of the command outcome. Retain a Partial record until
mount absence, work cleanup, residue inspection, and the down-phase save are
all known. Forget only after those succeed, and report a failed forget rather
than discarding it. Use a stable down-cleanup-failed refusal and inject each
failure boundary in tests.

### CAMP-REVIEW-014 — the accepted mid-session root-entry window contradicts the core guarantee

Location: camp/internal/plan/build.go:117-149;
camp/internal/gen/exclude.go:63-107;
camp-notes/reference/spec.md:797-814,946-995;
camp/README.md:63-83.

Root guards and exclude lines are a finite startup snapshot. The namespace
owner outside the session can create a new lower root name while the session
runs; the documentation explicitly encourages outside edits as a live view.
That name appears through the overlay with no read-only bind and no exclude.
Writing or deleting it copies up/whiteouts into the code repository, and git
add . can stage it. The end report can detect the damage but cannot make the
protection true.

The code conforms to spec §10 at 990-995, which explicitly accepts this
residual window. That local rule conflicts with invariant 1-2 and §7's claim
that every workspace-provided path fails EROFS, as well as the README. This is
therefore a specification/product-contract defect as well as an
implementation limitation; it must not be silently assigned to one side.

Normative impact: spec §3, §7 at 797-814, and the conflicting §10 at 990-995;
C1, C4, and C25.

Exact repair; owner's design move, then implementation's: either build a
synthetic frozen lower root from the accepted root entries (with live
read-only binds for those entries) and use that as overlay lower, so new root
names cannot appear mid-session, or narrow every invariant and user-facing
guarantee to startup-known names and explicitly accept copy-up/staging as
detection-only risk. A watcher is not an exact repair because it leaves a
race. The current combination of absolute promise and accepted hole cannot
remain.

### CAMP-REVIEW-015 — descriptor-based read-only remounts inherit /proc flags, not source flags

Location: camp/internal/mountx/mountx.go:99-137,156-204.

MountByDescriptor remounts /proc/self/fd/N. LockedFlagsAt cannot find that
string as a mount point in mountinfo, so its documented fallback performs a
lexical Containing lookup and selects /proc. Privileged read-only binds
therefore inherit /proc's usual nosuid,nodev,noexec flags rather than the
source mount's locked flags. This can silently make executable workspace
content noexec and fails to prove the C34 behavior it claims. Verification
checks only read-only polarity, not exact inherited flags.

Normative impact: spec §7 at 745-746, §14 at 1395-1404, and §15; C34 as stated
in spec. The separate constraints file does not currently contain C34; see
CAMP-REVIEW-017.

Exact repair; implementation's move: determine locked flags from the pinned
source mount before the bind (or by mount identity rather than lexical fd
path), pass them into the remount, and verify the resulting flag set against
that source. Add a privileged descriptor-path test with source flags
different from /proc's.

### CAMP-REVIEW-016 — privileged steady-state C8 protection compares an upper path string

Location: camp/internal/locks/scan.go:11-44;
camp/internal/mountinfo/mountinfo.go:284-303.

After privileged up exits, the flocks are released and ScanUpper is the only
steady-state guard. It finds overlays only when the decoded upperdir string is
exactly equal to the new plan's string. A bind-mount alias can name the same
upper inode with a different path, so a second privileged composition passes
the scan and gives one upper to two overlays. The transition flocks do not
help because the first transition is over.

Normative impact: spec §13 at 1176-1237; C8. The specification emphasizes
inode identity for precisely this alias class.

Exact repair; implementation's move: stat every visible overlay upperdir with
no symlink following and compare device/inode with the intended upper. An
unresolvable candidate must fail closed rather than disappear from the scan.
Add a bind-alias test covering privileged-up followed by a second plan.

### CAMP-REVIEW-017 — the two normative sources contradict each other about namespace identity

Location: camp-notes/reference/constraints.md:106-116;
camp-notes/reference/spec.md:270-305,1270-1404,2126-2129;
camp/internal/nsx/nsx.go:13-35,65-143.

constraints C18 says the one mapping has to map container uid 0, and C19 says
supplementary groups are lost. The specification says C18 is amended, C19 is
corrected, and implements own-uid mapping with ambient capability plus
retained groups. The code follows the specification, not constraints.md. The
spec also introduces C34 and C35 and later says constraints.md contains
C1-C34, while that file ends at C33.

The installed-binary namespace run is useful evidence for the current code,
but it does not resolve two simultaneously normative files. Under the
review's rules, silently choosing the newer prose is not acceptable.

Normative impact: the normative-source contract itself; C18-C19 and the
missing C34-C35.

Exact repair; notes/spec owner's move: update constraints.md with the measured
supersession history rather than erasing it—mark old C18/C19 conclusions
superseded, record the new measurements, and add C34/C35 with the same
numbering the spec uses. Then make every count/reference agree. If the older
constraints are intended to remain controlling, the spec and nsx code must
instead be reverted; one side must be chosen explicitly.

## P2 — narrower correctness and failure-honesty defects

### CAMP-REVIEW-018 — overlap descent and allow-list type checks fail open

Location: camp/internal/plan/gate.go:76-107;
camp/internal/plan/validate.go:386-425.

Any error reading either side of an allow-listed directory is returned as no
refusal, so an unknown merge is accepted. Root type validation skips
allow-listed names before checking newlines or file type. allow_overlap does
not require an actual overlap, so a lower-only symlink, FIFO, socket, or
newline name can bypass both the root guard and exclude.

Normative impact: invariant 6; spec §6 at 572-581 and §9; C12-C13.

Exact repair; implementation's move: validate every raw lower-root name and
type before coverage/allow-list exemptions. Convert descent errors into a
stable overlap-unreadable refusal naming both sides and the user's repair.
Test unreadable descendants and lower-only invalid allow-listed entries.

### CAMP-REVIEW-019 — scaffold retirement loses provenance on ordinary errors and crashes

Location: camp/internal/islands/islands.go:125-149,290-343.

retire treats every untouched error as “gone already,” ignores Manifest.Remove
errors, and drops the record. For an unchanged scaffold it removes the
manifest entry before the object and ignores both errors. A permission or I/O
error can therefore make camp disclaim an object that still exists; a crash
between the two operations leaves an unrecorded camp scaffold that the next
run refuses as user content. Add and Remove also mutate the in-memory map
before save succeeds.

Normative impact: spec §11 at 1063-1088, especially write-ahead provenance and
the stated lifecycle.

Exact repair; implementation's move: distinguish ENOENT from all other
errors, propagate every persistence/removal error, remove an unchanged object
before striking its record, and update in-memory state only after the durable
save. Crash-boundary tests must prove every intermediate state is either
attributable and recreatable or ordinary user content that camp leaves alone.

### CAMP-REVIEW-020 — end-of-session scans silently reuse stale data or report nothing on failure

Location: camp/internal/drift/drift.go:55-85,208-231;
camp/internal/inventory/inventory.go:298-318;
camp/internal/health/health.go:198-207.

Refresh ignores root-listing errors and reuses the startup snapshot.
inventory.Report maps a missing or damaged accepted snapshot to an empty
report. health maps git worktree failures to no notes. The Report type already
has a Failures field specifically so failed scans do not mean “nothing
found,” but these paths do not use it.

Impact: the exact mid-session changes and worktree risk §16 promises to name
can be suppressed by the failure that prevented looking.

Normative impact: invariant 4's reporting honesty; spec §16 at 1598-1612.

Exact repair; implementation's move: make every scan return data plus an
error/failure item; never retain a stale root as if it were current. A damaged
or missing inventory at teardown must be named. Test EACCES, malformed
inventory, repository failure, and one-side-only root read failure.

### CAMP-REVIEW-021 — doctor does not perform the capability probes the specification promises

Location: camp/internal/preflight/preflight.go:60-74,115-142,178-220.

overlayfs checks only /proc/filesystems. The namespace probe only makes /
private, not an overlay, and the privileged check only finds sudo on PATH.
There is no per-mode throwaway overlay, whiteout/opaque exercise, xattr
namespace check, storage-orphan scan in preflight, or real privileged
capability probe. doctor can therefore announce a mode as available when its
actual overlay mount or xattr behavior will fail.

Normative impact: spec §15 at 1573-1581 and the README's user-facing
availability claim.

Exact repair; implementation's move: perform a disposable overlay probe in
the namespace route and an explicitly terminal-gated privileged probe,
including copy-up, whiteout, opaque behavior, xattr namespace, locked flags,
and cleanup. Where sudo cannot be exercised, print “not tested; requires a
terminal,” not available.

### CAMP-REVIEW-022 — namespace reports overwrite within one second and marking is not once-only

Location: camp/internal/reports/reports.go:37-51,85-101,119-134;
camp/internal/reports/reports_test.go:72-95.

Report names are hash plus Unix seconds and Write atomically replaces an
existing name. Two sessions ending in the same second lose the first report.
The test named “kept apart” explicitly accepts one file. MarkSeen copies,
rewrites, and removes instead of renaming; Show ignores read and mark errors,
so a report can be repeated, silently lost from delivery, or overwritten by a
same-name .seen file.

Normative impact: spec §14's persisted session result and §16 at 1692-1694.

Exact repair; implementation's move: allocate collision-free names with
nanoseconds plus random/O_EXCL, and mark by same-directory atomic rename to a
non-colliding .seen name. Return and report delivery errors. The test must
require exactly two reports after two immediate writes and exactly-once
delivery across injected rename/read failures.

### CAMP-REVIEW-023 — interrupting an external generator can leave its process group running

Location: camp/internal/gen/run.go:106-191.

The generator is put in its own process group. Timeout kills that group, but
there is no SIGINT/SIGTERM forwarding. Terminal Ctrl-C reaches camp's
foreground group, not the separate generator group; camp can exit while the
generator and its descendants continue writing scratch/output. The comment's
claim that an interactive user can interrupt it is therefore not implemented.

Normative impact: spec §19 at 1851-1865 and the general truthful-failure rule.

Exact repair; implementation's move: install scoped signal handling after
start, forward terminal termination signals to the generator process group,
wait for it, and restore handlers. Test a parent and grandchild on SIGINT and
SIGTERM, including cleanup and returned refusal.

### CAMP-REVIEW-024 — filesystem errors are collapsed into absence in ownership and sweeping

Location: camp/internal/verify/verify.go:330-354;
camp/internal/cli/session.go:86-104.

Storage verification stats only the storage root although §12 says everything
under storage belongs to the invoking user, and every Stat error is treated as
“it need not exist,” including EACCES and I/O errors. The namespace sweeper
similarly treats any os.Stat(live) error as proof that live is gone and may
delete that session's work. Only ENOENT supports that conclusion; otherwise
the lock test or an explicit failure must decide.

Normative impact: spec §12 at 1102-1135 and §15 at 1553; invariant 3.

Exact repair; implementation's move: distinguish nonexistence from unknown
errors everywhere. Verify ownership recursively without following symlinks,
or narrow the specification explicitly if root-only ownership is intended.
Sweep only on ENOENT or a successfully acquired live lock; retain and report
everything else.

### CAMP-REVIEW-025 — malformed mountinfo records are silently omitted

Location: camp/internal/mountinfo/mountinfo.go:86-159.

Read drops every line parse rejects and returns the remaining table as
authoritative. A malformed or newly unsupported line can therefore hide the
one camp mount that completeness, residue, C8 scanning, or teardown needed to
see. This is the same dangerous shape as a command-output parser that returns
a partial answer.

Normative impact: invariant 6; spec §15's mount-table cross-check and §13's
steady-state guard.

Exact repair; implementation's move: make parse return a descriptive error
with line number and have Read reject the whole snapshot. Tests should include
short records, missing separators, invalid ids/devices, overlong lines, and a
bad line between valid camp mounts.

### CAMP-REVIEW-026 — virtual target resolution mistakes ENOTDIR for an absent upper path

Location: camp/internal/pathx/resolve.go:120-145;
camp/internal/plan/virtual.go:64-84.

StatBeneath classifies ENOTDIR as absence. When an upper file shadows a lower
directory at an ancestor, virtual.overlay asks for a child, sees the upper
ENOTDIR as absent, and falls through to the lower child. The paper walk can
therefore accept a mount target the real overlay cannot reach. Mounting then
fails late, after preparation and possibly earlier mounts, instead of during
static validation.

Normative impact: spec §7's exact virtual walk; C3, C10, and C13.

Exact repair; implementation's move: distinguish missing final entry from a
non-directory/shadowed ancestor and propagate that state through virtual
resolution. Add upper-file/lower-directory and inverse ancestor tests at
several depths.

### CAMP-REVIEW-027 — unusual worktree paths produce an unparsable record and an unsafe repair command

Location: camp/internal/gitwire/gitwire.go:269-308;
camp/internal/drift/drift.go:103-117,235-280.

Worktrees parses newline-delimited porcelain without -z or Git path
unquoting. Paths containing tabs, newlines, quotes, or backslash are legal and
can be quoted or split by Git. The displayed path and generated
git -C ... worktree repair ... command are then ambiguous; even an ordinary
space makes the “exact” command wrong because neither argument is shell
quoted. Other drift paths are printed raw despite the project's reversible
encoding rule.

Normative impact: spec §16's exact repair promise, §18's hostile-name
encoding, C24, and C32's byte-oriented handling.

Exact repair; implementation's move: request NUL-terminated porcelain,
implement its NUL record grammar, retain raw bytes, encode display with the
shared reversible encoder, and render a correctly shell-quoted argv (or print
one argument per labelled line instead of claiming a pasteable command).

## P3 — maintenance defects, not taste

### CAMP-REVIEW-028 — abstractions with no production caller already drift from their comments

Location: camp/internal/compose/compose.go:221-243,315-322;
camp/internal/preflight/preflight.go:165-176;
camp/internal/session/session.go:92-103.

compose.Check is called only by tests, although its comment says status uses
it; status calls verify.Run directly and already supplies different inputs.
compose.CleanWork and preflight.tool have no caller. session.Options.Plan and
Exclude are populated but unread, and their apparent presence conceals the
P1 plan-rebuild race above.

This is not a file-layout preference: the project explicitly says no
abstraction without a caller that exists today, and the first abstraction has
already become a false statement about the executable path.

Exact repair; implementation's move: either route the real commands through
these functions/fields and test that path, or remove them. In particular,
status should consume the recorded-plan verification API, not keep a second
assembly beside an unused “same pass” wrapper.

## Test review

### Commands run against f7010f1

- GOCACHE under /tmp, go build ./...: passed.
- GOCACHE under /tmp, go vet ./...: passed.
- gofmt -l .: no output.
- CAMP_TEST_ROOT under /tmp, go test -count=1 -v ./...: exit 0, but 16
  real namespace tests skipped. That exit is not counted as evidence for
  those tests.

The direct-checkout skips were eight internal/compose tests, one internal/nsx
test, and seven internal/session tests. They cover the full composition,
tmpfs locked flags, shadow detection, exclude scoping, islands, repeat-run
scaffolding, held teardown, route A, workload status/signals, namespace
visibility, environment isolation, PATH lookup, and daemonized lock lifetime.

After the user installed this version, camp-notes/reference/remaining-checks.md
records an owner-run:

    camp run -- go test ./internal/... -count=1

It says every package passed and nothing skipped. I did not independently run
that command because a camp session writes work/report state and the review
was read-only. I treat it as useful owner-provided namespace evidence, not as
my direct observation. The same notes file explicitly says the terminal-gated
privileged lifecycle has not run. No sudo or privileged E2E was run in this
review.

### Tests whose names overstate what they measure

- internal/state/state_test.go:63-89 deletes the config and proves only that a
  Record can be loaded and reversed. It does not call camp down, record
  selection, status, explain, or the helper.
- internal/reports/reports_test.go:72-95 says reports are kept apart, then
  accepts one file after two writes and labels the overwrite a known limit.
- internal/gen/gen_test.go:338-352 says no-generation fallback uses a raw
  listing but checks only empty exclude plus a note. It misses the git call,
  absent scaffold, expanded validation, and real mount.
- internal/gen/gen_test.go:193-259 checks invented absent output, not an
  existing untracked addition, omitted tracked entry, or omitted target.
- internal/fsx/writesites_test.go:14-98 checks the lexical location of write
  calls, not Area root provenance, symlink traversal, or final identity.
- internal/plan/plan_test.go:181-199 checks only a same-name collision inside
  an allow-listed directory. It never exercises a one-sided lower child,
  read-only behavior, or git staging.

### Material missing tests

- No privileged staging/move/down E2E, which would have caught
  CAMP-REVIEW-001 immediately.
- No helper containment/authentication test, arbitrary-job test, or separate
  race for overlay, work, staging, and live operands.
- No failure injection between successful mount(2), read-only remount, and
  MS_PRIVATE.
- No namespace config-swap-under-old-lock test.
- No strict/corrupt record schema or mount-identity replacement test.
- No package-local test at all in cmd/camp, capsx, holders, islands,
  preflight, refusal, testenv, or verify. Some are exercised indirectly, but
  verify in particular owns the safety decision and deserves direct tables
  for complete/partial inputs.

## README and docs review

These are documentation defects tied to the code findings above, not prose
preferences.

1. README.md:10-11 says nothing is left behind when a session ends, while
   README.md:55-56 and docs/how-it-works.md:16-21 correctly document
   disposable work, persistent storage, and reports. Say instead that the
   namespace and mounts vanish; storage persists, reports may be written, and
   work residue is swept on the next run.

2. README.md:71-77 and docs/how-it-works.md:470-473 claim fsx addressing
   cannot be constructed from a repository path. CAMP-REVIEW-003 is a direct
   counterexample, and the test does not establish that property. Remove the
   claim until the descriptor-confined constructors and provenance tests
   exist.

3. README.md:63-83 promises every workspace-provided write fails EROFS.
   Neither the allow-listed-directory hole nor spec §10's accepted new-root
   window is disclosed. The implementation must close them or README's
   guarantee must be narrowed with the exact consequences.

4. README.md:132-134 and docs/how-it-works.md:406-413 say the helper executes
   an already validated plan, re-resolves every operand by descriptor, and
   moves only after successful staging verification. CAMP-REVIEW-001,
   CAMP-REVIEW-002, and CAMP-REVIEW-004 make all three claims false.

5. docs/how-it-works.md:335 says status is the same verification pass, and
   lines 418-422 say status/down read the record and never configuration.
   The executable path at cli/inspect.go:98-153 and 222-287 does the opposite.
   This must be fixed with CAMP-REVIEW-007, not papered over.

6. docs/how-it-works.md:281-288 says hostile generator output is checked as
   contributed content, but current validation checks only existence/type and
   permits omissions. Document exact-set validation after
   CAMP-REVIEW-010 lands.

7. README.md:147 says doctor tells whether both modes are available, and
   docs/install.md:231-246 describes a mount probe. The namespace probe makes
   mount propagation private but never mounts OverlayFS; privileged mode is
   not probed. Until CAMP-REVIEW-021 lands, the output and docs must
   distinguish preliminary checks from proven availability.

8. internal/report/explain.go:99-117 always tells the user to run camp down
   for worktree repair before later saying namespace mode has no down. Render
   mode-specific repair timing: privileged down output versus the namespace
   end report/next command. Also, gen/run.go:178-182 says a failed generator
   leaves the machine “exactly as it was,” but cli/compose.go:85-93 has already
   created work/storage directories and marker files. Say
   “nothing mounted” and name what scratch was created.

examples/config.yml and packaging/apparmor/camp are internally consistent with
the intended design. Their helper/identity claims inherit the implementation
and normative conflicts above; I found no separate defect in their syntax or
instructions.

## Matters of taste

No style-only concern is promoted to a finding. The code has unusually long
comments and many small internal packages, but the comments generally carry
real design rationale and the package boundaries are mostly coherent.
Renaming packages, shortening prose, or consolidating files would be
preference without a correctness payoff. CAMP-REVIEW-028 is listed because it
violates an explicit project rule and has already caused behavioral drift,
not because I prefer fewer functions.

## Coverage

Read closely, production code:

- cmd/camp;
- internal/capsx, cli, compose, config, drift, enc, envx, fsx, gen, gitwire,
  health, holders, inventory, islands, locks, mountinfo, mountx, nsx, pathx,
  plan, preflight, privileged, refusal, report, reports, session, state,
  testenv, and verify.

Read closely, tests:

- cli, compose, config, fsx, gen, gitwire, inventory, locks, mountinfo,
  mountx, nsx, pathx, plan, privileged, reports, session, and state.

Skimmed rather than line-by-line, after reading their production packages and
enumerating every test:

- drift, enc, envx, health, and report tests.

Documentation and repository support files read closely:

- README.md;
- docs/getting-started.md, docs/install.md, docs/how-it-works.md;
- examples/config.yml, go.mod, and packaging/apparmor/camp;
- all of camp-notes/reference/spec.md and
  camp-notes/reference/constraints.md;
- the current remaining-checks handoff for installed-test evidence.

Not reached:

- No production package or requested documentation file was left unread.
- I did not perform a legal review of LICENSE.
- I did not execute the privileged terminal/sudo lifecycle, kill-point
  matrix, real rename-race injection, person-gated OpenSSH/keyring checks, or
  manual tmux/GUI checks. These are execution-coverage gaps, not packages
  silently counted as reviewed.
