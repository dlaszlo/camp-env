# The operand swaps, the last three test gaps, and what they found

Date: 2026-08-18, the session after the one that handed over
`log/2026-08-18-handoff-what-is-left-to-test.md`. It started at camp
`8ddf464` and the environment at `af4814a`, with the testing sudoers rule
installed, and it did what that file listed: the operand swaps first,
while the rule was there, then the three Go tests that need no privilege.

Both halves found something. The swaps found a failed `camp up` telling
the user the composition was standing when nothing was mounted; writing
the test the review asked for found a tracked-content check whose failure
still read as "nothing tracked". Both are repaired, each with the test
that catches it.

## The operand swaps: CAMP-REVIEW-004's own test request

The review's repair pinned every operand the helper mounts by device and
inode, and asked for a swap exercised **independently for each operand**.
What was measured before today was a bind source, in both its forms. What
is measured now is the rest: the overlay's `lower`, `upper` and `work`,
and the live directory.

The instrument is two small Go programs, about forty lines together, in
this session's scratchpad. One waits for a trigger and then renames a
directory aside and puts something else in its place, timing both. The
other samples `/proc/self/mountinfo` in a tight loop and prints only when
what it is watching for appears or disappears. Two triggers were enough:
*the helper's process exists* -- which is strictly after the front end has
taken every identity, and about ten milliseconds before the helper's first
check -- and *a named mount point appears in the machine's mount table*.

Each run ended the same way: `camp status`, then `camp down`, then a
check that nothing is mounted under the environment root, no record is
left, camp's work directory is gone, the live directory is empty and the
workspace is writable again. All of them converged.

**The baseline first.** A plain `camp up` and `camp down`, which also
measures the repair camp `8ddf464` made and nobody had run: `up` took
86 ms and mounted eleven, `down` removed twelve of twelve and then said
*removed: .camp/work/56a06176bd04, camp's own work directory*. The work
directory is empty afterwards. That closes the one thing the previous
session left half-measured.

**The overlay's upper layer, replaced with a different real directory.**
Refused: *camp is not the object camp checked: it was 64513:6295166 and it
is now 64513:6319724.* Nothing was mounted.

**The overlay's upper layer, replaced with a symlink** pointing at the
real repository. Refused by the resolution before there is anything to
compare: *opening the overlay's upper layer /home/dlaszlo/dev/camp-env/camp:
a component of the path is a symbolic link: camp in
/home/dlaszlo/dev/camp-env.*

**The overlay's work directory, replaced with a different real
directory.** Refused: *.camp/work/56a06176bd04/work is not the object camp
checked: it was 64513:6319705 and it is now 64513:6319724.*

**The overlay's lower layer.** Refused -- *camp-workspace is not the
object camp checked: it was 64513:6308495 and it is now 64513:6319724* --
but by the check that reaches that path first, which in this composition
is not the overlay's. The workspace is also the frame's very first mount,
bound onto itself to hold it read-only, so its identity is compared as
that mount's source before the overlay's operands are opened at all. In
this environment the overlay's own lower check cannot be reached by a
swap: there is no window between the two that anything outside can hit.

**And after that first mount there is no window at all.** With the
freeze bind standing, renaming the workspace aside fails outright:
*rename camp-workspace camp-workspace.aside: device or resource busy*. The
composition came up normally, eleven mounts, exit 0. A mounted directory
is not renameable, so from the helper's first mount onwards the operand
is held by the kernel rather than by a check.

**The live directory, replaced with a different real directory** in the
window between the staging tree's last mount and the move. **Accepted**:
`camp up` finished, eleven mounts verified at the live path, and the
composition stands on the directory the swap put there -- device 84,
inode 6295166 -- while the directory camp validated sits beside it, empty.
`camp status` says *up: every recorded mount is present, and each is the
object camp mounted*, which is true of every mount it records; the live
directory is not one of them.

This is not the helper breaking a promise. No identity is carried for the
live path -- the job has `LiveParts` and no `LiveIdent` -- and the
descriptor the move goes to is opened after the swap, so it opens the
impostor. What the descriptor buys is the interval between the open and
`move_mount`, which is microseconds; the interval the front end's
validation covers is milliseconds, and it is not covered. Worth the
owner's decision, next to the other window the previous session recorded.

**The live directory, replaced with a symlink** pointing outside the
environment root. Refused, and the machine left clean: *opening the
composed tree's directory /home/dlaszlo/dev/camp-env/camp-live: a
component of the path is a symbolic link: camp-live in
/home/dlaszlo/dev/camp-env*, followed by *Nothing is mounted: the helper
removed everything it had made before it stopped.*

One detail under that refusal is worth keeping. The step before the open
is `mountx.Detach`, which binds the live directory onto itself so the move
cannot propagate -- and it does that **by name**, so it follows the
symlink and makes root's bind on whatever the symlink points at. A
sampler reading the mount table about 12,000 times a second (146,053
samples in 12 s) never saw it: the rollback removes it within the same
handful of microseconds. It is a mount root makes outside the composition
at a name the user controls, and it lasts less than a hundredth of a
millisecond. Recorded, not repaired.

## What the swaps found: a failed up that says the composition is standing

The upper-layer swap was refused by the helper's precheck -- the pass that
compares every operand before the first syscall that changes anything --
and `camp up` then said:

```
[ERROR] camp up failed, and what it built is still on the machine:
        /home/dlaszlo/dev/camp-env/camp-workspace stays read-only until it
        comes down.
[ERROR] ... Nothing has been mounted
        The rollback could not finish. Still mounted: .
```

Nothing was mounted, the workspace was writable, and `camp status` run a
second later reported all eleven recorded mounts as *gone*. The record was
left in phase `partial` for a composition that was never built.

The cause is one line: the precheck's refusal returned the reply directly
instead of through `unwind`, and `unwind` is what sets the rolled-back
flag the front end reads to choose between "nothing is mounted" and "what
it built is still there". Specification §12 is explicit -- *a failure with
clean rollback removes the record* -- and a rollback with nothing in it is
the cleanest there is. Repaired in camp `5ce1fec`, with a test that fails
against the old line.

The repair is not yet measured end to end, and that needs the owner: the
installed binary is what elevates, `sudo install` needs a terminal, and
this session had none. The front end's honest branch *is* measured today
-- the live-symlink run above took it and reported the machine clean --
so what the repair does is move the precheck's refusal onto that branch.

## The three test gaps

**CAMP-REVIEW-019, the scaffold's crash boundaries** (camp `7c2231d`).
Both intermediate states are now built by hand and run through
`gen.Prepare`. A manifest that records an attachment point with nothing on
disk -- what a crash between the write-ahead record and the object leaves
-- is created again, with the record intact. A record left by a retirement
that had removed the object and not yet struck it is struck, silently,
because there is nothing to report. The third state, an object with no
record, is the one no crash can leave under this order: it is refused as
content camp cannot account for, and the test now asserts that the file's
bytes survive the refusal.

**CAMP-REVIEW-011, a repository git cannot read** (camp `5ee09c4`). The
three states had only been measured with no git on PATH. The states a real
machine produces are measured now, and they are not one answer:

- a repository directory camp cannot enter is `Unreadable` -- git never
  reaches the question;
- a working tree whose `.git/index` cannot be read answers the frame
  question and fails the one that matters, and that failure now stops the
  composition instead of arriving as "tracks nothing";
- a repository whose `.git` camp cannot read is reported by **git itself**
  as *not a git repository (or any parent up to mount point /)* -- git's
  own no, character for character. camp reads git's answer and has nothing
  behind it to read. Changing that means asking a different question, not
  classifying the answer differently, and the test says so where somebody
  will meet it.

**CAMP-REVIEW-022, a failed mark** (camp `9ec92e1`). The rename is now
made to fail -- the reports directory is made unwritable -- and the
delivery holds: the report is printed once, the failed mark is said with
what follows from it, nothing is marked, and once the directory is
writable again the report is delivered and marked exactly once more.

## What the second test gap found: a check that failed open

Writing the 011 test found the other defect. `plan`'s own tracked-content
check read a `TracksUnder` failure exactly like "tracks nothing":

```go
tracked, err := c.code.TracksUnder(mount.Rel.String())
if err != nil || len(tracked) == 0 {
	return
}
```

That is the shape CAMP-REVIEW-011 was about, and its repair -- *carry
errors through every tracked-content callback* -- reached the generation
pass and not this one. Measured: with the code repository's index
unreadable, `plan.Prepare` refused nothing at all, and only generation
stopped the composition. So nothing ever mounted with the rule unchecked,
but `camp plan` printed a clean plan for a composition camp would refuse,
and the rule that keeps `git commit -a` from recording deletions nobody
made was not the one doing the stopping. Repaired in camp `e648a7f`, with
the test.

## Two small things, neither repaired

**A temp record from 2026-08-16 is still in the state directory**:
`~/.local/state/camp/.56a06176bd04.json.753417962`, a complete record
written as the temporary half of an atomic save and never renamed --
almost certainly a kill from that session's kill-point matrix, landing
between the write and the rename. `camp list` says *no records* and
`camp doctor` says nothing about it. It is inert, it is camp's own, and
nothing sweeps it or names it.

**A rolled-back `up` still leaves camp's work directory**, as recorded
before. It was swept by the next `camp run` here, which is the documented
route.

## The state this leaves

camp is at `e648a7f`, five commits on from `8ddf464`: three that measure,
two that repair. Both gates are green with nothing skipped, the second one
through the installed binary:

```
cd ~/dev/camp-env/camp && go build ./... && go vet ./... && gofmt -l . && go test ./...
cd ~/dev/camp-env      && camp run -- go test ./internal/... -count=1
```

**The installed binary is no longer this commit.** It is `8ddf464`, so
`camp up` and `camp down` still exercise the old precheck reply. Anything
privileged measured from here needs the install first:

```
cd ~/dev/camp-env/camp && go build -o camp ./cmd/camp
sudo install -m 755 camp /usr/local/bin/camp
```

**The testing sudoers rule is still installed**, and this session could
not remove it: `sudo rm /etc/sudoers.d/camp-testing` needs a password and
sudo cannot prompt without a terminal (C33). It is one command from the
owner's terminal, and it is the last step of this group.

Re-measuring the precheck repair needs no sudoers rule at all, only a
terminal: install the binary, run `camp up` with the upper layer swapped
the moment the helper's process appears, and read the closing lines. It
should now say the machine is clean and leave no record.
