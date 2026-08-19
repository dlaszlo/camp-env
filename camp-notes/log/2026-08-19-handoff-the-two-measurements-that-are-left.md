# Handoff: the two measurements that are left, and what is already built for them

Date: 2026-08-19. Written for whoever picks this up next, including a
later session of the same work. camp is on `fix/review-8ddf464`, head
`f481283`, pushed. `main` is fifteen commits behind it and carries both
P0s; merging is a decision the owner has deferred until these
measurements are made.

## Where things stand

The twelve findings of `log/2026-08-19-read-only-code-review-8ddf464.md`
are repaired. An independent review of the repairs
(`log/2026-08-19-fable-review-of-the-repairs.md`) found three risks the
branch had introduced; all three are fixed. The namespace mode passes the
whole suite from inside a composition, and the privileged mode ran end to
end by hand — `log/2026-08-19-the-privileged-mode-on-the-new-mount-path.md`
has the mount table, the record and the teardown.

What is left is not about camp working. It is about what camp does when
something goes wrong, and it is exactly two things plus one small
experiment:

1. **The kill matrix.** Interrupt `camp up` at each boundary and recover
   from the record alone.
2. **The rename race.** Swap the environment's name under the helper at
   each resolution, and require that nothing outside the original base
   inode changes.
3. ~~The `umount2` experiment~~ — **done, and it says yes, by another
   route.** See below; it turned into a third piece of work.

## What is already built

### The barrier

`internal/privileged/barrier.go` is an empty function and is what every
ordinary build gets. `internal/privileged/barrier_camptest.go` carries
the protocol and is compiled **only** under `-tags camptest`. The reason
for the split is in both files: a pause inside the root helper that the
invoking user can trigger is the attack, not a seam. Two guards in
`barrier_guard_test.go` read the files as data and fail the build if the
constraint is lost — checked against the edit that would really do it.

**Driving it.** Build the binary with the tag and run *that* binary; the
helper is the same executable, so no install is needed:

```
go build -tags camptest -o ~/campcheck/camp-barriers ./cmd/camp
```

Arm a barrier by writing camp's work directory before `camp up`:

```
<work>/camp-barrier      one line: "<name> kill"  or  "<name> wait"
<work>/camp-barrier.go   created by the driver to release a wait
```

`<work>` is `$ENV/.camp/work/<hash>`, and the hash is what `camp plan`
prints. The barrier announces arrival on **stderr** — `camp barrier:
reached <name> (<mode>)` — and never writes, because a write outside
`fsx` is what the source guard refuses.

- `kill` sends the helper `SIGKILL` where it stands. Not an exit: nothing
  deferred runs, no reply is written, and the machine is left holding
  exactly what that boundary left it holding.
- `wait` blocks for up to a minute, or until the driver creates
  `camp-barrier.go`. Past the minute it carries on and says so, so a
  driver that died cannot leave root asleep inside a composition.

**The names**, in the order they happen:

| name | where |
|---|---|
| `base-owned` | the helper has opened and checked its base |
| `prechecked` | every operand compared, nothing mounted yet |
| `staging-bound` | the staging self-bind exists |
| `mount-made` | one nested mount exists and is identified |
| `staging-verified` | the tree passed the check in staging |
| `live-bound` | the live self-bind exists |
| `before-move-open` | before the move's two descriptors are opened |
| `moved` | `move_mount` succeeded; the tree is at the live path |
| `staging-unbound` | the staging self-bind has been removed |
| `reply-encoded` | the reply is built and not yet written |
| `reply-received` | the front end has the reply, before it saves |
| `stands-there` | teardown: identity checked, before the unmount |

`mount-made` fires at every nested mount, so a driver that wants the
third one releases the first two.

The one barrier the review asked for that has no subject any more is
"after a bind is attached and before the reopen". There is no reopen: the
mount is held by the descriptor that made it.

### The `umount2` probe — run, and what it found

`~/campcheck/umountprobe/` — source and a built binary, run as

```
sudo unshare -m --propagation private ~/campcheck/umountprobe/umountprobe
```

Four questions, and the answers are in `reference/constraints.md` as
C34–C36. In short:

- **A mount point cannot be renamed** (`EBUSY`), so the rename race can
  only be played at an ancestor. C34.
- **`umount2` on a descriptor's own `/proc/self/fd` name cannot be the
  answer**, for two opposite reasons: a descriptor on the mount pins it
  and the call gets `EBUSY`; a descriptor on the mount point is *resolved
  through* into the mount, so the path decided and not the descriptor.
  C35.
- **But a mount can be moved by descriptor and unmounted where it
  lands**: `open_tree` then `move_mount` into a directory named by
  another descriptor moves the same mount id, leaves the original path
  clear, and `umount2` there removes it once nothing pins it. C36.

So the residual of finding 1 **is** repairable, by the route the review
itself suggested — identify by descriptor, move to a place only root can
name, unmount it there. That is a third piece of work and it is the one
with the most value in it: it closes the last open criterion of a P0.

### The scratch composition

`~/campcheck` — two git repositories, five mounts, inventory accepted,
`camp plan --privileged` clean. **On ext4 on purpose**: `/tmp` is tmpfs
here and the privileged mode asks for `nouserxattr`, which is the
`trusted.overlay.*` namespace.

## What the next session builds

Three things. The first is a code change and the other two are drivers.

### The descriptor-safe teardown

C36 is measured, so the teardown's `umount2`-by-path and `Detach`'s
cleanup unmount can both stop handing the kernel a name it resolves
again. The shape: resolve the target beneath the pinned root, check its
identity on that descriptor as the teardown already does, then
`open_tree` it, `move_mount` it into a root-owned directory camp makes
for the purpose, and `umount2` it there.

What has to be decided along the way, and is not decided here: where the
graveyard lives (it must be a directory the invoking user cannot rename,
which rules out anywhere under the environment root), what happens when
the move succeeds and the unmount does not, and whether a mount that
cannot be moved should fall back to the unmount by name or be reported
stranded. The comments at `helper.go`'s teardown and at `mountx.Detach`
say what is open today; they are what this replaces.

Nothing here can be run by an assistant: it mounts. It lands the same way
the descriptor binds did — written against a measured primitive, and
proved by the next real `camp up`.

### Two drivers Neither can be run by an assistant — both need sudo on a
real terminal — so they are written to be handed over and run by the
owner, and they must print a verdict rather than a log to read.

**The kill-matrix driver.** For each of the eight boundaries: build the
scratch composition, arm the barrier with `kill`, run `camp up`, then
delete the configuration and recover using the record alone. The
assertions the review states:

- every camp-created mount ID disappears;
- no unrelated mount ID disappears;
- the repositories and the storage hash identically before and after;
- the record survives until every place is clean;
- anything that cannot be removed is named exactly.

The record's phase says which boundary was reached, and `camp status`
must describe the composition from the record with no configuration.

**The rename-race driver.** For each of `base-owned`, `prechecked`,
`before-move-open` and `stands-there`: arm with `wait`, and when the
barrier announces itself, rename the environment root away and leave a
symlink to a root-owned trap tree at its name, then release. The
assertion is **not** that camp errors:

- no mount ID outside the original base inode changes;
- no mount attribute outside it changes;
- no inode mode outside it changes.

Record the trap tree byte for byte before and after.

## Constraints that shaped all of this

The machine is shared, so an assistant runs nothing privileged and
nothing that mounts. The suite runs only as
`CAMP_TEST_ROOT=<a writable directory> go test ./...`, and now also with
`-tags camptest`, which has to stay green too. Every measurement in these
logs was either run by the owner at a terminal or is marked as unrun.
