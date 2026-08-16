# Verifying the whole-project review, and what will be repaired

Date: 2026-08-16. Verified against camp `f7010f1` and this repository at
`60d9b59` — the same snapshots the review names.

`log/2026-08-16-review-camp-whole-project.md` is a whole-project review by
a reviewer who could not modify the tree and did not run the privileged
mode. Twenty-eight numbered findings, two of them called release
blockers. A review is not evidence, so nothing in it was acted on before
it was checked: this file records what was checked, by whom, what
survived, what was corrected, and what is being repaired in what order.

Two verifications were run against it, independently and adversarially.
The implementing session read the two P0 findings in the source and
traced their mechanisms. A second reviewer went through all
twenty-eight, built real mounts inside a namespace through the installed
binary, ran hostile helper jobs directly, and isolated one failure by
changing a single variable. Both left the working tree clean.

**Every one of the twenty-eight findings survived.** None was wrong. Four
were misstated in a way worth correcting before anybody acts on them, and
the verification found **one further release blocker the review missed**
— the one that fails first of all.

## The privileged mode does not work, for three independent reasons

This is the headline, and it is worse than the review said. The
privileged mode cannot mount anything, cannot verify what it mounted, and
misreports the failure. Each of the three would be sufficient on its own.

**The first mount fails.** *(found by the verification, not in the review;
measured — a real bind inside a namespace, then isolated by changing one
variable)* `mountx.MountByDescriptor` opens the target once, with
`O_PATH`, before the bind. An `O_PATH` descriptor captures the
`(vfsmount, dentry)` pair as it was at open time, so after the bind is
stacked on that dentry, `/proc/self/fd/N` still resolves to the object
*underneath*. The read-only remount and the `MS_PRIVATE` call are then
made against the wrong mount and fail `EINVAL`. The counter-test: the
identical `MS_PRIVATE` call succeeds through a descriptor reopened after
the bind and fails through the stale one, with nothing else changed. The
namespace mode is unaffected — it uses the plain-path `mountx.Mount`,
which is exercised by every composition test. `MountByDescriptor` is
privileged-only and no mounting test has ever run it.

Spec §14 asks for the new mount API (`open_tree`/`move_mount`) where the
kernel has it and `/proc/self/fd/N` otherwise. Only the fallback is
implemented, and it is implemented against a descriptor that no longer
names what the code believes it names.

**Verification then rejects every honest plan** (CAMP-REVIEW-001;
*confirmed twice, once by reading and once by running the helper's own
`verifyStaging` against a real freeze-lower mount*). `verifyStaging`
rebuilds a `plan.Plan` carrying only `Live` and `Mounts`, leaving `Config`
zero. Completeness registers the workspace self-bind as present only
through `mountinfo.At(table, in.Plan.Config.LowerPath())`, and
`LowerPath()` on a zero `Config` is the empty string, which matches
nothing. The mount is planned at its real target and never found, so
`verify-missing-mounts` fires and the staging tree is rolled back before
the move. The workspace self-bind is the frame's first, unconditional
mount: every honest plan has it. Restoring `Config` in the same input
returns zero refusals, which pins the cause to that one dropped field.
Dropping `Storage` and `Exclude` in the same reconstruction silently
disables the ownership and artefact checks as well.

*Correction to the review.* It asks for the original `InLive` semantics
to be restored. That part is not a defect: `MountJob` already rewrites
the target components of tree-local mounts to the staging tree, so
`verify.at` must **not** remap them a second time, and `InLive = false` is
what makes it leave them alone. The repair is smaller than the review
suggests — give the verification input the lower path, the storage path
and the payload explicitly instead of reaching for them through a
`Config` the helper does not have.

**And the failure is reported as a clean machine** (CAMP-REVIEW-006;
*measured*). A mount is added to the rollback list only after the whole
multi-syscall operation returns. When the bind succeeds and the remount
fails — which is exactly what happens above — the target is never
recorded, rollback skips it, and the reply says `RolledBack: true` while
one mount is still standing on the workspace path. The same shape exists
in `compose.Build`.

## The helper is a general root primitive

CAMP-REVIEW-002, *confirmed twice: by reading, and by running hostile
jobs through the real code paths.* `Helper` plain-unmarshals its job and
checks only the version and the action. On the unmount path, `Targets`
are handed to `unix.Unmount` verbatim, and then — guarded only by there
being work parts and no stranded mounts — `Base` joined with `WorkParts`
is passed to `RemoveTree`, which deletes recursively, and to `Chown`,
which walks the tree calling `Lchown` with the uid and gid **taken from
the job**. A job naming `base: "/"`, `work_parts: ["etc"]` therefore makes
root chown the whole of `/etc` to whatever id the caller asked for. Both
were run and observed doing exactly that in a scratch tree, failing only
on the verifier's own lack of privilege.

This contradicts the helper's own documentation in two places at once —
that it trusts nothing it was handed, and that everything camp creates is
chowned to `SUDO_UID`/`SUDO_GID` rather than to a number out of the job.

**The threat model, stated exactly**, because the review leaves it
implicit and the difference decides how alarmed to be. Somebody who can
already run arbitrary `sudo` gains nothing here; they are root already.
What this creates is a confused deputy, and it bites in three situations:
a `NOPASSWD` rule scoped to `camp` — which is the obvious way to stop
`camp up` reprompting, and which anybody packaging this would reach for;
a cached sudo timestamp, which by default lasts about fifteen minutes
after a `camp up`; and any code running as the user during that window.
The last one is named in the specification: §19 requires that whoever can
edit the configuration must never gain root through it, and a configured
generator runs as the user moments before `camp up` elevates. camp ships
no sudoers rule today, so a default install does not hand this to a
non-sudoer — but the entry point is documented, and a rule that makes the
tool pleasant to use turns it into a root hole. That is a release
blocker regardless of today's packaging.

## The three other corrections

Acting on a misstated finding costs as much as missing one, so these are
recorded before the repairs begin.

**CAMP-REVIEW-004** lumps the bind endpoints in with the overlay operands
as unpinned. The bind source and target genuinely are descriptor-pinned
and identity-checked; only the overlay's `lower`/`upper`/`work` option
strings and the final `MS_MOVE` operands are raw. Whoever repairs it must
not tear out the pinning that works.

**CAMP-REVIEW-021** says `doctor` performs no capability probe. The
namespace probe does attempt a real mount and does catch AppArmor
confinement. The genuine gap is that nothing probes overlayfs, whiteouts
or xattrs, and that the privileged side only looks for `sudo` on the
path.

**CAMP-REVIEW-017** says the two normative sources contradict each other
silently. They contradict each other, but not silently: the specification
records the supersession of C18 and C19 explicitly, with its
measurements. What is actually wrong is bookkeeping — `constraints.md`
stops at C33 and was never updated, while the specification's own
reference claims it carries C1–C34. The code follows the specification
and is right; the documents need the repair, not `nsx`.

## The order of repair

The first four are what stands between this snapshot and a privileged
mode that can be used at all. They are being repaired now, in this order,
because each is a precondition for observing the next.

1. **Reopen the target after binding.** The remount and the propagation
   change must be made against a descriptor that names the new mount, not
   the object it was stacked on. Until this lands, nothing else about the
   privileged mode can be measured.
2. **Give the staging verification a complete input.** The lower path,
   the storage path and the validated payload travel in the job; the
   reconstruction stops dropping them. Keep `InLive = false`, which is
   correct.
3. **Stop the helper being a general primitive.** Strict decoding, uid
   and gid from `SUDO_UID`/`SUDO_GID` rather than from the job, every
   operand confined beneath one authenticated composition, and no raw
   absolute unmount target accepted.
4. **Record a mount for rollback the moment it exists**, not when the
   whole operation finishes — and never report a clean machine while
   something is still mounted.

Each of the four needs the test that fails on today's code before the
repair lands. The absence of any privileged lifecycle test is what let
all four through, and the terminal-gated group in
`reference/remaining-checks.md` has never run.

After those, in the order the verification put them: the lexical rather
than descriptor-confined `fsx` boundary (003); the unpinned overlay and
move operands (004); the record that cannot drive verification, status or
a configuration-free teardown (007, 008); `down` reporting success after
cleanup failed (013); allow-listed directories being writable holes with
no exclude (005); the namespace init rebuilding a plan under locks taken
for another (012); git's operational failures being read as "nothing
tracked" (011); and the islands paths (009, 010). The narrower
correctness and honesty defects (015, 016, 018–027) follow. The
maintenance finding (028) is last and is a deletion, not a design
question.

Two of the findings are not code at all and belong to whoever owns the
documents: the specification accepting a mid-session new-root-entry
window that its own invariants forbid (014), and the constraint
bookkeeping above (017).
