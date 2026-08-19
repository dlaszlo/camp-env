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
3. **The `umount2` experiment**, which decides whether the last open
   residual of finding 1 can be repaired at all.

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

### The `umount2` probe

`~/campcheck/umountprobe/` — source and a built binary. It answers
whether `umount2` on a descriptor's own `/proc/self/fd` name acts on the
mount that descriptor holds, including after the path has been renamed
and a trap left at its name, with a control for a descriptor opened
*before* the mount. It refuses to run outside a private mount namespace,
because it mounts things:

```
sudo unshare -m --propagation private ~/campcheck/umountprobe/umountprobe
```

If both answers come back right, the teardown's `umount2`-by-path and
`Detach`'s cleanup unmount can both be closed, and finding 1 has no
residual left. If not, the residual stays and the comment already in the
source is the honest state.

### The scratch composition

`~/campcheck` — two git repositories, five mounts, inventory accepted,
`camp plan --privileged` clean. **On ext4 on purpose**: `/tmp` is tmpfs
here and the privileged mode asks for `nouserxattr`, which is the
`trusted.overlay.*` namespace.

## What the next session builds

Two drivers. Neither can be run by an assistant — both need sudo on a
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
