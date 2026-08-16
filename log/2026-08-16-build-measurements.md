# Measurements taken while building, 2026-08-16

What the rebuild found on the owner's machine that the specification
could not have known. Facts, with how they were obtained; the
specification stays the design, this file is evidence.

## The identity route (spec §14 route A, §23's first named measurement)

**Question.** Map the caller's own uid to itself instead of to 0, carry
`CAP_SYS_ADMIN` across `execve` in the ambient set, mount, verify, drop —
does that work, does the user look like themselves inside, and do the
mounts keep working after the drop?

**Answer: yes, with one correction.** `CAP_SYS_ADMIN` alone is not
enough. The overlay mount is refused with `EACCES` unless
`CAP_DAC_OVERRIDE` is carried with it.

Bisected over the ambient set, with the lower, upper and work
directories created moments earlier by the mounting process itself,
owned by it, mode 0755:

| ambient set | overlay mount |
|---|---|
| `SYS_ADMIN` | `EACCES` |
| `SYS_ADMIN`, `DAC_OVERRIDE` | succeeds |
| `SYS_ADMIN`, `DAC_OVERRIDE`, `DAC_READ_SEARCH` | succeeds |
| the six further capabilities tried after it | no change |

A bind mount and a tmpfs mount need only `SYS_ADMIN`; the requirement is
specific to the overlay. It does not widen what a session can do: both
capabilities are dropped before the workload starts, and while camp holds
them it is the user acting on the user's own directories.

The rest of the chain measured as designed. Inside the namespace `id`
shows the real user (uid 1000, gid 1000, not root); the ambient set
survives `execve` (`CapEff=0000000000200002`); the overlay mounts; a
write to a read-only bind standing in the composed tree fails `EROFS`; a
write through the overlay lands in the code repository; after
`capsx.Drop()` every set is zero; a further mount attempt is refused;
and a write through the overlay still lands in the code repository —
the kernel recorded the mounter's credentials at mount time, as
reasoned.

**C19 confirmed again, under the own-uid mapping this time.** Inside the
namespace `setgroups` is denied and the supplementary groups display as
`nogroup`, but the kernel credential retains them: a host file readable
only through a supplementary group (`/var/log/syslog`, `root:adm 0640`)
opened and read. This is what makes the code repository's own pre-push
gate run in the primary mode.

**How it was run.** A freshly built binary cannot do any of this here —
see the install gate below — so the measurement was taken inside a
namespace opened by the installed `/usr/local/bin/ply`, which the
specification's testing note allows as a vehicle. Inside that vehicle
the caller is already uid 0, which would not exercise the non-zero-euid
condition the whole route turns on, so the spike maps the nested
namespace's uid 1000 to the vehicle's uid 0. That reproduces the
condition exactly: a non-zero euid inside, capabilities available only
through the ambient set.

    go test -c -o /tmp/nsx.test ./internal/nsx
    CAMP_SPIKE_INSIDE_UID=1000 CAMP_SPIKE_INSIDE_GID=1000 \
      /usr/local/bin/ply run -f <lab config> -- /tmp/nsx.test -test.run TestRouteA -test.v

**Still to confirm after the install:** the same run at one level of
nesting, on the host's own mapping. Nothing in the mechanism depends on
the nesting, and every step of the chain was exercised, but the number
of levels is the one thing the vehicle changes.

## The install gate is not a refusal — it is a confinement

`kernel.apparmor_restrict_unprivileged_userns=1` on this machine does
**not** refuse the namespace. It lets the namespace be created and
confines the process inside it to the `unprivileged_userns` profile,
which then denies every mount. A check that only asked whether the clone
succeeded reported "user namespaces: permitted" for a namespace nothing
can be built in.

So `doctor`'s probe now attempts a mount inside the namespace it creates,
with the same identity mapping and the same carried capabilities a real
session uses, and reads `/proc/self/attr/apparmor/current` when it
fails. It reports:

> the namespace can be created, but this machine confines it to the
> unprivileged_userns profile, which refuses every mount

and names the install as the repair.

One detail worth keeping: the probe execs its own binary **by real path,
not through `/proc/self/exe`**. The process doing the `execve` is
already inside the new namespace and therefore already confined, and
that profile refuses the magic symlink — probing through it turns
"confined" into "refused to create one", which is a different diagnosis
with a different repair.

## The locked flags, and /tmp (spec C34)

**The old failure on /tmp was a missing OR, not a property of tmpfs.**
Measured: `/tmp` here is a tmpfs whose mount carries `nosuid,nodev`, and
`strictatime` by absence of an atime flag. A read-only remount inside a
user namespace that does not carry those flags forward is refused with
`EPERM`; one that ORs them in succeeds.

So a whole composition on `/tmp` now passes, end to end: every mount
made, every check passed, a write to a workspace-provided path `EROFS`,
a new file landing in the code repository, teardown clean, nothing left
in the live directory. The test asserts the fix. The old bug is still
reachable, but only through a function that exists for that purpose --
`mountx.RemountReadOnlyWithoutLockedFlags` -- so a machine where the
flags happened to be empty could not make the test pass for the wrong
reason.

## The composition, measured against the kernel

Run inside a real user namespace, on ext4 and on tmpfs:

- every planned mount present, reachable by path, and the right way
  round;
- a write to a workspace-provided path in the composed tree: `EROFS`;
- a write to the workspace through its own absolute path, from inside:
  `EROFS`;
- a new file created in the composed tree: lands in the code
  repository;
- a write under the record repository's mount: lands in that
  repository, not in the code one;
- the generated exclude readable at `live/.git/info/exclude`, carrying
  camp's marker;
- teardown removes everything, and the live directory is empty
  afterwards.

**A shadowed mount is caught.** Binding something over one of the
composition's own mounts, after it is up, makes verification fail with
`verify-identity`. This is the case a table-based check cannot see: the
covered mount is still listed in `/proc/self/mountinfo`, and only asking
the path what it resolves to notices.

## The kernel's leftover work directory

Observed directly, confirming C6: after an overlay is unmounted the
kernel leaves `<workdir>/work` behind as `d---------` — mode 000, owned
by the invoking user. Anything that sweeps a work directory has to chmod
it before it can remove it, which is why the specification says so in
two places.

## Generation, islands and the repeated session

Measured in a real namespace, with the shipped `git_exclude` step:

- **The exclude's scoping.** `live/.git/info/exclude` carries camp's
  generated block; `code/.git/info/exclude` does not, at the same
  moment. The bind is on the composed tree's copy, so the repository and
  every checkout registered outside keep reading their own file and
  nothing camp does survives the session there.
- **Islands and water.** Editing an entry the workspace contributes,
  through the composed tree, is `EROFS`. Writing a machine-local name
  the same directory has never held succeeds and lands in camp's
  storage.
- **The repeated session.** up, down, up again, with islands: the second
  run meets its own attachment points already standing in the water and
  accepts them, because the scaffold manifest records whose they are.
  Without the manifest the collision rule — which exists to stop camp
  hiding your machine-local files — would refuse camp's own objects on
  every second run. What the first session wrote into the water was
  still there in the second.

## The supervisor

Measured through a real session, with the workload's stdio wired to
files:

- a foreground command's exit status propagates (`exit 7` → 7);
- a command that dies on a signal reports 128 + the signal;
- the composed tree really exists inside (the merged listing) and the
  live directory is empty outside at the same moment;
- **a daemonised workload returns at once** while the init stays
  resident and the locks stay held: a `setsid`'d child that closes its
  descriptors keeps the session alive, the launching command's caller
  gets its shell back immediately, and a second composition on the same
  upper is still refused.

That last one found a real bug in the first implementation. The launcher
was waiting for the handshake pipe to close rather than for the
workload's exit message, so `camp run -- tmux new-session -d` would have
waited for the entire session instead of returning. The init now reports
the workload's status the moment the workload is reaped and goes on
reaping until the namespace empties.

## The teardown that is held

A process whose working directory is inside the tree makes the unmount
fail. Measured: the mount stays mounted and is still reachable through
the tree, the report names the holding process from `/proc` with what
ties it there, and the command fails. There is no lazy detach anywhere in
camp — a source scan fails the build if `MNT_DETACH`, `umount -l` or a
detach flag appears.
