# The descriptor-safe teardown, and the instruments for the last two measurements

Date: 2026-08-19, following
`log/2026-08-19-handoff-the-two-measurements-that-are-left.md`. camp is
on `fix/review-8ddf464`; this session added one commit to it. The notes
repository gained the drivers and this entry.

Three things were asked for and three were built. The teardown and the
kill matrix have since **been run by the owner at a terminal**, and the
last two sections say what that found; the rename race has not been run
yet. Everything before those sections is what was decided and why, which
is what a reader needs whether or not the measurement agreed.

The teardown's first real run found a defect in it immediately, and the
kill matrix found one in `camp status` and two in its own driver. All of
that is below.

## 1. The descriptor-safe teardown

The last open criterion of finding 1 was: *teardown cannot unmount a
replacement object inserted after its identity check.* It could, because
the check was made on a descriptor and the act was `umount2` on a name the
kernel resolved again, with the invoking user owning every directory above
it.

Built on C36, which is the only route the measurements left open. The
mount is named by `open_tree` on the descriptor the identity check was
made on, `move_mount` takes it into a directory only root can name, and
`umount2` removes it there. The choice is made on a descriptor and cannot
be redirected; the name the kernel finally walks belongs to root.

### What was decided, and why

**Where the graveyard lives.** `/run/camp/graveyard/<the helper's pid>`,
root-owned, 0700. Not under the environment root, because that is the tree
whose names can be swapped — that is the whole point. /run is root's and
world-unwritable on every Linux with an init, it is tmpfs, and a mount
does not survive a reboot either, so the residue and the thing it
describes have the same life. The pid makes two teardowns running at once
two graveyards.

**It is a mount, not just a directory.** A mount attached under a *shared*
parent is copied into that parent's peers, and the copies are not camp's
to remove; /run is `shared:5` here. So the pid directory is bound onto
itself and that bind made private, by descriptor — the same primitive and
the same kernel rule that `Detach` already exists for. It is the only
mount camp makes outside an environment.

**The move worked and the unmount did not.** The mount goes back where it
came from, into the directory descriptor and the single name that came out
of the same walk, and the caller is told it is busy — which is exactly
what it was told before, so the record, `holders`, `camp status` and the
next `camp down` all keep meaning what they meant. Leaving it in the
graveyard was considered and refused: `confine` will not accept a teardown
target outside the base, so a graveyard path in the record is a path the
next `camp down` could never act on. A mount stranded in /run would be a
mount camp could name once and never remove.

**A mount that cannot be moved.** By name, as camp did everywhere until
now — and that is not a choice. The kernel refuses to move a mount whose
parent is shared, and camp's two self-binds are exactly that on any
machine where `/` is shared. The mount table is asked *before* a grave is
made, so an ordinary `camp up` — whose only removal is the staging
self-bind — builds no graveyard at all. What is bounded here: those two
mounts carry no recorded identity for a swap to steer away from, and they
stand at names inside camp's own work directory. What would close it is in
`reference/remaining-checks.md`, and it is not measured, so it is not what
camp is written on.

**A graveyard that cannot be made.** The teardown refuses, with nothing
touched — it asks before it acts, so refusing costs a message. A rollback
falls back to the name and says so in the reply, because a rollback that
refused to act would leave a half-built composition standing, which is the
one thing camp's failure handling may never do.

### What else moved with it

Every unmount the root helper performs now goes one way: the teardown, the
rollback and the staging self-bind removal all open the mount from the
components it was made through and hand the descriptor over. `Detach`
stopped removing its own half-made self-bind — that is the rollback's job,
and one route is the point. The rollback's list carries components rather
than paths, because a mount recorded only as a path has to be resolved
again to be taken down.

Two smaller consequences, both deliberate:

- A path with nothing mounted at it is now answered "absent" from the
  mount table without a syscall, and a path the table names but the helper
  cannot reach beneath its pinned root is a **mismatch** rather than an
  unmount. That second one is the attack's own shape: an ancestor renamed
  away, so the name still reaches a mount and no longer reaches camp's.
- A helper killed at a barrier leaves its graveyard mounted with nobody to
  remove it — no record names it and no teardown would ever target it — so
  every invocation sweeps away the graveyards whose pid is gone.

## 2 and 3. The two drivers

`camp/measure/killmatrix` and `camp/measure/renamerace`, with `camp/measure/README.md`
saying how to build the barrier camp and how to run them. They are in this
repository and not in camp because they measure it from outside: they read
the kernel's mount table and the trees on disk themselves and share no
line with camp. A driver using camp's own parsing would agree with camp by
construction, and agreement is what is being tested.

Both print a verdict rather than a log, and every failure names the object
— a mount as its whole `mountinfo` line, a tree as the two hashes that
differ, a mode as the path and the two values.

**killmatrix.** One ordinary run first, which is where it learns the
composition's hash — the barrier is armed by writing camp's work directory
and that directory's name is the hash — and how many nested mounts there
are. Then the review's eight boundaries, with `mount-made` expanded to one
case per nested mount: arm, run `camp up` into the kill, rename the
configuration away, and require `camp down` to recover from the record
alone.

Stopping at the *n*th nested mount needed no change to the barrier
protocol. The barrier re-reads its arming file at every call, so the
driver waits at the *n-1* before it and rewrites the file to `kill` before
the last release. A run that does not stop where it was aimed reports
itself as **not measured**, which is neither a pass nor a failure.

**renamerace.** A root-owned trap tree at `/var/tmp/camp-trap`, laid out
like the environment and carrying two mounts nothing else on the machine
has, at the two places camp mounts. At each of `base-owned`, `prechecked`,
`before-move-open` and `stands-there` the environment is renamed away and
a link to the trap takes its name.

One directory of the trap belongs to the driver rather than to root: the
work directory. The barrier resolves that path by name like everything
else, so after the swap the release file has to be written on the trap's
side of the link, or the helper waits out its minute.

The assertion is the trap tree and the rest of the machine, and not camp's
exit code: camp may refuse and camp may carry on, and a root process that
acts wherever a name points is a confused deputy whatever it prints.

## What the kill matrix found on its first run

Two things, and only the first is camp's.

**A finding: `camp status` reads camp's own self-bind as the composed
tree.** A run killed at `staging-bound`, at `mount-made` before the
overlay is made, or at `live-bound` leaves camp's self-bind standing at
the very path the record names as the overlay's staging or live location.
`Mount.PresenceAt` asks only whether *something* is mounted there, and
with no identity in the record — which is every killed run, because the
reply never came back — it answers `unverified`. The operand comparison
then runs against the self-bind and says:

```
not as recorded: it answers as "ext4" and the record says "overlay"
not as recorded: its lowerdir is "" and the record says ".../workspace"
...
5 recorded thing(s) about the composed tree are not what is mounted.
```

Every line of that is true about the object it looked at and wrong about
the machine: the composed tree was never mounted at all. The verdict line
counts the self-bind as the overlay as well — *"partly up: 1 of 5
recorded mounts are present"* — and the sentence a person reads is the
one that says camp cannot account for the tree, which is the sentence
that means somebody else's mount is standing there.

Nothing unsafe follows from it. The teardown does not consult this: a
record with no identity is unmounted from the table, both places are
named, and every case here came down clean. It is a message that misleads
at exactly the moment somebody is reading it to find out what happened.

The obvious repair, not made here: the record carries the filesystem each
mount should answer as, and `PresenceAt` has the table. A path answering
as a different filesystem from the one recorded is this mount being
**gone**, not this mount being present and wrong.

**The rest was the driver's fault, twice.** It asked whether the output
contained "no record", which matched camp's own *"no recorded identity to
check it against"*; and then it required `camp status` to exit zero, when
a partly-up composition is exactly when status must exit non-zero. Both
are fixed. What they cost is worth writing down: a driver that asserts on
a phrase rather than on a fact reports the tool as broken and itself as
right.

## What held

With those two corrections, every requirement the review lists held at
all twelve boundaries — the staging self-bind, each of the five nested
mounts, the staging verification, the live self-bind, the move, the
staging self-bind removal, the reply encoding, and the front end holding
the reply. Every mount camp made disappeared, no unrelated mount id
disappeared, the repositories and the storage hashed identically before
and after, the record survived exactly as long as something was standing,
and the teardown named what it could not remove.

The five-step staircase worked: killing at the *n*th nested mount by
waiting at the *n-1* before it reached all five.

## The teardown's first real run

It failed, on the first `camp up`, and the failure was in the new code:
the descriptor the identity check had been made on was still open across
the unmount. A descriptor on a mount is a reference to it and `umount2`
answers `EBUSY` while one is held — which is C35, measured, as the reason
the obvious repair cannot work, and it is the same reference whoever
holds it. `Remove` closed its own handles and left the caller's alone, so
every mount camp tried to remove came back busy, including the
composition it had just built.

`Remove` now takes the mount by pointer, closes that descriptor as soon
as it has asked it everything it is for — which mount this is, and a
handle carrying that decision — and leaves `-1` behind. The caller cannot
be left to close it afterwards: by then the number may belong to
something else the process opened.

Worth keeping for the shape of it: the constraint that broke it was
written down, measured, and quoted in the comment three lines above the
code that violated it.

## What the rename race found on its first run

It stopped at the first swap, `base-owned`, and what it caught is the
reason the whole measurement exists: **root unmounted a mount that was
nothing to do with camp.**

```
a mount outside /home/dlaszlo/campcheck disappeared while the name was swapped:
  11651 33 252:1 /var/tmp/camp-trap/marker-staging
        /var/tmp/camp-trap/.camp/work/a1478934bd08/staging ro,relatime shared:1 - ext4 ...
```

The sequence: the barrier stopped the helper with its base open and
checked, the driver renamed the environment away and left a link to the
trap tree at its name, the run went on and failed, and the rollback
removed camp's staging self-bind. A self-bind cannot be moved — its
parent is the mount `/` is, which is shared — so it had no graveyard
route and came down by the name the record had written down. The kernel
resolved that name through the link, and the trap's own mount at the same
relative path is what went.

This session had written that residual down twice and called it bounded,
on the grounds that those names sit inside camp's own work directory. The
measurement corrected the argument in one run: **the environment root
itself is the thing being swapped**, so a name inside it reaches wherever
the link points. A bound that assumes the attacker leaves the top of the
path alone is not a bound.

The repair is the primitive `remaining-checks.md` had listed as waiting.
`umount2` takes a path and nothing else, so the way to give it a mount by
descriptor is the parent directory's own `/proc/self/fd` name with the
one component appended: the kernel resolves the magic link to the
directory the descriptor holds, then walks one component from there. C34
says that component cannot be renamed while something is mounted on it.
The descriptor comes from the same walk the identity check was made on.

The addressing half is now **C37**, measured unprivileged and with no
mount: a directory renamed away with a link to another tree left at its
name does not redirect its own descriptor's `/proc` name. The half that
is only end-to-end — that the call acts on the mount standing there and
on nothing else — is what the rename race measures whenever it runs.

## The second run: what held, and the second finding

With the descriptor-relative unmount in place, the whole group was run
again. **The kill matrix held at all twelve boundaries**, and the rename
race held at `base-owned`, `prechecked` and `before-move-open`. At
`stands-there` its three assertions held too: no mount id outside the
environment changed, no mount attribute outside it changed, no inode mode
in the trap tree changed, and the trap hashed the same before and after.
Root did not act where the link pointed, at any of the four resolutions.

What did not hold is what happened *inside* the environment, and it is a
finding of its own.

**`camp down` reported success with five of its own mounts still
standing, and then discarded the record.** The sequence, from the mount
table afterwards: the barrier stopped the helper at the first teardown
target, the name was swapped, the helper removed that first target
correctly — through the descriptor, which the swap could not redirect —
and then reported every remaining target as **absent** and exited 0. The
front end took that for a finished teardown and released the record.

The cause is one line's worth and it is not in the new code. Each target
is looked up in the mount table by path:

```go
if len(mountinfo.At(table, target.Path)) == 0 { /* absent */ }
```

A mount point cannot be renamed (C34), but an *ancestor* can, and the
kernel then reports the mount under its new path. So after the swap the
table lists camp's mounts at `.../campcheck.real/...`, the recorded paths
match nothing, and "nothing is mounted there" is the answer to a question
about a name rather than about a mount. `state.Release` asks the same
question the same way, which is why the record went too.

Nothing outside the environment is affected and nothing is a confused
deputy here: the person who renamed their own environment is the person
whose workspace stays read-only until they unmount it by hand. But it
breaks the rule the review stated for finding 3 — never forget a record
while a mount exists at any of its paths — and it breaks it silently.

**The repair this suggests, not made here.** The helper already holds the
base as a descriptor. `readlink("/proc/self/fd/<that descriptor>")` is the
path that base has *now*: when it is not the path the job was built for,
the environment has been renamed underneath the helper and every
path-based lookup in the table is unreliable. A teardown in that state
should refuse with nothing unmounted, which keeps the record and leaves
the person able to put the name back and run it again. It costs one
readlink, needs no primitive nobody has measured, and asks the descriptor
rather than the name — which is the rule this whole repair has been
following.

The thorough version is to stop identifying camp's mounts in the table by
path at all: `statx(fd, STATX_MNT_ID)` names the mount a descriptor holds,
and the table can then be read by id. That is Linux 5.8 and nobody here
has measured it.

## Both findings repaired, and then the machine

The two decisions above were taken: the teardown refuses when the base's
current path is not the one the job names — one `readlink` on the
descriptor it already holds — and a different filesystem at a recorded
path is now the end of the operand comparison rather than the first line
of it. With those in, the rename race holds at all four resolutions and
the kill matrix at all twelve boundaries.

Then the question that mattered more than either: **how does any of this
run without a person?** Three things needed one — sudo's password, which
cannot be typed without a terminal; the apparmor gate, which grants the
namespace to one installed binary path and so needs an install; and the
privileged mode being machine-wide, so it cannot run beside anybody's
work. A machine that exists for the length of one run answers all three
at once, and this machine can boot one: `camp/measure/vm` is plain user-space
qemu with KVM, a cloud image and a seed, no libvirt and no root on the
host.

It earned its keep within the hour. Four defects, none of which could
have been found here:

- **camp refused to run for a user who had never had `~/.local`.** A
  fresh account has no state directory, and camp's records are the first
  thing that would live in it. The specification says whoever needs the
  directory makes it; camp now does.
- **The identity spike wrote its own read-only remount**, without the
  source mount's locked flags — the call this repository keeps under the
  name `RemountReadOnlyWithoutLockedFlags` because it is the deliberately
  wrong one. `/tmp` is `nosuid,nodev` on most machines, so the island
  stayed writable and the test reported camp as letting a write reach the
  workspace. It had never shown because the test only runs where a binary
  may create a namespace, and the one arrangement where that was true put
  the test tree on ext4.
- **overlay has to be loaded before anything is measured** — see below.
  This was first written up as "the image has to be the current release",
  because the older image failed the overlay probe. That machine also had
  no overlay module loaded, the two were never separated, and the claim
  about the release is withdrawn: it was not measured. What was measured
  is the module.
- **overlay is a module a cloud image has never used**, so it is not
  loaded, `/proc/filesystems` lists it nowhere and camp refuses — which
  is right, because a user namespace cannot load a module. The package
  now asks for it at boot.

That last one is part of the other piece of work: camp builds its own
Debian package, so putting it on a machine is one command that also
places the apparmor profile with the right path in it. The machine
installs camp that way rather than by copying a binary, which is the only
way the profile's path, the module and the maintainer scripts get
measured at all.

## What is left

The person-gated group: ssh and the keyring, which need real credentials
and a real peer. Everything else runs on its own — `camp/measure/vm/guest` is
the list, and adding a measurement is adding a file to it.

Then two things follow from whatever they say. If the kill matrix holds,
finding 3's acceptance criteria are met by measurement rather than by
argument. If the rename race holds, finding 1 is closed except for the one
primitive named in `remaining-checks.md` — and measuring that would close
it entirely.

Merging `fix/review-8ddf464` into `main` is still the owner's decision and
still deferred until these are made.
