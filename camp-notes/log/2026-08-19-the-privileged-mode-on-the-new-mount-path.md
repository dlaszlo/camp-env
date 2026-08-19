# The privileged mode runs on the new mount path, and tears down from the record alone

Date: 2026-08-19. camp from `fix/review-8ddf464`, head `f2ca05a`, built
and installed at `/usr/local/bin/camp`. The branch answers the twelve
findings of `log/2026-08-19-read-only-code-review-8ddf464.md` and changes
how every bind is made, so the whole lifecycle was run once by hand — the
same step taken on 2026-08-18 when the composed tree moved to the mount
API, and for the same reason.

Everything below was measured. Where a sentence says what something
proves, the evidence is quoted above it.

## The composition it was run on

A scratch environment, `~/campcheck`: two empty git repositories with one
commit each, a `notes/` directory on the workspace side, a `README.md` on
the code side, the shipped `git_exclude` step, and the `.git` bind the
template ships. Five mounts.

**Not under `/tmp`.** `/tmp` is tmpfs on this machine, and the privileged
mode asks the overlay for `nouserxattr` — the `trusted.overlay.*`
namespace — which is not a thing to discover mid-measurement. The
environment lives on ext4, which is also why `internal/testenv` keeps its
own root out of `/tmp`.

The first attempt refused, correctly and for the documented reason: no
accepted inventory snapshot. `camp accept`, then `camp plan --privileged`
said nothing stops it.

## `camp up`

It completed. In order: the locks on the code repository and the live
path, the gate, the generation, the record, `sudo camp helper-mount`,
five mounts in the staging tree, the move onto the live path, and the
verification there. One password prompt.

## What the kernel says was made

```
11651 33    252:1 …/campcheck/workspace  …/campcheck/workspace  ro,relatime - ext4 …
11690 11694 0:205 /                      …/campcheck/live       rw,relatime - overlay none rw,lowerdir+=…/workspace,upperdir=…/code,workdir=…/work/a1478934bd08/work,uuid=on,nouserxattr
11691 11690 252:1 …/workspace/notes      …/live/notes           ro,relatime - ext4 …
11692 11690 252:1 …/code/.git            …/live/.git            rw,relatime - ext4 …
11693 11692 252:1 …/work/…/exclude       …/live/.git/info/exclude ro,relatime - ext4 …
11694 33    252:1 …/campcheck/live       …/campcheck/live       rw,relatime - ext4 …
```

Six lines, and each one settles something that until today was written
against the kernel's documented contract rather than observed:

**The composed tree records the real paths.** `lowerdir+=…/workspace`,
not `/proc/self/fd/6`. This is the whole reason the overlay is built with
`fsopen`/`fsconfig` rather than an option string, and it now holds for
the privileged mode too.

**The read-only remount through the clone's own descriptor takes.** Three
mounts read `ro` — the workspace freeze, the `notes` root guard and the
exclude. After `move_mount` the clone's `/proc/self/fd/N` names the
attached mount and not the object it was stacked on, which is the single
fact the whole repair rests on.

**So does the propagation change.** No line carries an optional field
between its per-mount options and the `-`. Every one of the six is
private, and the same descriptor is what carried that call.

**`OPEN_TREE_CLONE` copies the one mount `MS_BIND` would have made.**
Each bind is a single line whose root field is the source subtree, and
nothing extra appeared beneath any of them: the clone is not recursive,
which is what `AT_RECURSIVE` would have made it.

**The private parent works as designed.** The composed tree's parent is
11694, the live self-bind, and not `/`. The staging self-bind is absent —
removed after the move, as the last step of the mount job.

## What the record holds

`version: 2`, `phase: up`, and:

- `detached` carries **both** self-binds, in the order they are made:
  the staging point first, the live point second.
- every mount inside the tree carries its staging location as well as its
  target; the workspace self-bind carries none, which is right — it is
  the one mount that is not in the tree and is never moved.
- every mount carries a non-zero identity. The helper aborts and rolls
  back rather than recording a zero, and nothing reached that path.
- the composed tree carries its operands: `lowerdir+`, `upperdir`,
  `workdir`, `nouserxattr`.

`camp status` reported all five as `same`, printed what the tree was made
with, and found **no disagreement** between the recorded operands and the
mounted filesystem. That is the first time the recorded overlay fields
have been checked against a real mount by production code rather than by
a test.

## From outside the composition

`touch …/workspace/x` → `Read-only file system`. The live path shows the
merged tree to any process — `README.md` from the code repository,
`notes` from the workspace. The generated exclude is readable through the
tree, and the repository's own `.git/info/exclude` underneath it keeps
its own content.

## Teardown with the configuration gone

The configuration was moved aside **while the composition was up**, which
is the case the record exists for.

`camp status` described the composition from the record alone. `camp down`
took it apart from the record alone, said so, and named what it skipped:

```
the configuration this was built from cannot be read now (…). That
changes nothing about the teardown, which comes from the record.
…
6 mount(s) removed, and 5 of the places the record names had nothing at
them: a mount is recorded both where it is built and where it is moved
to, and it stands at one of the two.
…
the drift and leak scans need the configuration and were skipped …
The teardown needed none of it.
```

Afterwards: no mount anywhere under the environment, the work directory
removed, the record discarded, the live directory empty, and both
repositories at their original `HEAD` with nothing modified.

## What this retires from the unmeasured list

Of finding 1 — the descriptor mount path in the privileged mode, in every
part quoted above, and that pinning the helper's base broke nothing.

Of finding 3 — that the record names both places, that a teardown works
from it with no configuration, and that the record is discarded once
nothing stands.

Finding 12 entirely: the recorded operands have a production consumer and
it ran against a real mount.

## What is still unmeasured

Three things, none of them about whether camp works and all of them about
what it does when something goes wrong.

**The kill matrix.** Today measured the successful path. Interrupting
`camp up` at each of the eight boundaries and recovering from the record
alone is what would measure the interrupted one, and it is the case the
staging half of the record was written for.

**The rename-race barriers.** A base renamed and replaced with a link at
each barrier. The assertion is not that camp errors: no mount ID, mount
attribute or inode mode outside the original base inode may change.

**The descriptor-safe unmount.** `umount2` still takes a path, in the
teardown and in `Detach`'s cleanup. Whether `umount2` on a descriptor's
own `/proc/self/fd` name acts on the object that descriptor holds is a
small experiment nobody has run, and it decides whether that repair can
be written at all.

## Two defects found on the way

**A test that was a fork bomb inside a composition** (`525093b`). A new
test drove the real `camp doctor`, which starts its capability probe as
`os.Executable()` — from a test binary, the test binary. The child did not
recognise the argument, ran the whole package again inside the user
namespace it had just been given, and each copy started another. Invisible
on the host, where the clone fails; unending inside a composition, where
it does not. `TestMain` answers the probe now, the way `cmd/camp` does.

**`camp plan --privileged` printed calls camp no longer makes**
(`f2ca05a`). Every bind rendered as `mount(source, target, "", MS_BIND,
"")`. The twelfth finding moved the overlay to one description the mount
is performed from; the bind half of the same renderer was still writing
the calls out by hand, and there is nothing in a bind's plan entry to
derive them from — what decides is the mode, which the plan has carried
all along. Found while planning the composition this file is about, on
the one command somebody reads to decide whether to run `camp up`.
