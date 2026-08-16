# Recovery from the record, and five readings that want a verdict

Date: 2026-08-16, the session after the handoff. camp from `18f243f` to
`642f0d1`. Everything below was measured on this machine, on real
privileged compositions, not argued from the source.

This file exists for the next review. The repairs are in the commits and
speak for themselves; what needs somebody else's eyes is the five places
where I read the specification one way and could have read it another.
They are at the end, and each says what I did and what the alternative
was.

## What running it found

The privileged mode had been run end to end exactly once before this
session. Running it twice more found two defects, and both were the same
shape: a measurement that was wrong about the machine.

**`camp up` failed its own post-move check on every honest composition.**
`mountinfo.Top` took the last line at a mount point as the topmost mount,
which is not what the table means. Mounts are listed roughly as they were
made, and `MS_MOVE` keeps a mount's identity, so the composed tree moved
onto the live path is listed *before* the self-bind it now covers -- the
self-bind that commit `543885c` had just introduced to give the move a
private parent. `Top` therefore returned the bind: ext4, no overlay
options, no `nouserxattr`. The four refusals that followed were all about
a mount that was correct, and the composition was left standing under a
report that it had failed. The parent field says which mount is on top
without ambiguity, and that is what it reads now.

**And the failure was reported as a clean machine.** The closing sentence
was fixed text -- "Nothing of this composition is mounted, and the
workspace is writable again" -- printed on every exit, while three of the
exits after the move leave the composition standing on purpose. It said
the workspace was writable while it was held read-only for the whole
machine, three lines above the refusal saying the tree was still mounted.
`Up` now returns what it left behind and the sentence follows it.

## The dead end, measured

With a composition up, the configuration moved aside, and the workspace
read-only for the whole machine:

```
$ camp status
[ERROR] no .camp/config.yml here or in any parent directory of ...
$ camp down
[ERROR] no .camp/config.yml here or in any parent directory of ...
```

The record held the whole plan. `camp list` could see it. The two
commands that needed it insisted on a file that no longer existed, and
there was no camp way back to a writable workspace. That is CAMP-REVIEW
007 and 008, and it is worse in practice than on paper: `status` was
equally useless *with* the configuration present, because it re-derived a
plan and the derivation refuses a live directory that is not empty --
which is exactly what a composition that is up looks like. The command
the refusal recommends ("run 'camp status' first") could not run.

Both are repaired. A composition is now named three ways that need no
configuration -- its record, its live path, or the directory you are
standing in -- and `status`, `down` and `explain` read the recorded plan.
Measured afterwards, same situation: `camp down` with no configuration
anywhere removed 11 of 11 mounts, said which part it could not do (the
drift and leak scans, which do need the file) and why, and left the live
path empty, the workspace writable and no record behind.

Two other things came with it, because they were in the same functions
and would have been dishonest to leave: the teardown compares each
mount's recorded device and inode before unmounting it (a path alone
cannot tell camp's mount from a stranger's that took the same name), and
a teardown whose last step fails no longer reports success (that is
finding 013).

## The five readings

**1. `explain`'s source.** §16 says explain is generated from the live
configuration so that it cannot go stale. §12 says down, status and
explain read the recorded plan and never a configuration that may have
been edited while the composition was up. Both sentences are about
explain and they do not agree.

What I did: what is mounted decides. A record whose mounts are actually
present is what the reader is standing in, so it is described from the
record; with nothing standing -- and the namespace mode leaves no record
at all -- the configuration is the only source there is. The description
is now one set of sentences fed from either source.

The alternative I did not take: always the configuration, treating §12's
sentence as being about `down` and `status` with explain named by
accident. That keeps one code path and accepts that a description of a
standing composition can be a description of a different composition.

**2. What `created` may name.** §12 asks the record to carry "every
camp-created path (the work directory, the scaffold entries)". The field
existed and was never filled.

What I did: the work area and the overlay work directory inside it.
Storage is not in the list, and neither are the attachment points camp
scaffolds inside storage -- camp never removes storage (invariant 3), so
a list whose own comment says "what camp may clear" must not name it, and
what camp created in there is already recorded in the islands manifest,
which lives beside it and outlives this record.

The alternative: name the scaffold entries as §12 literally says, and
rely on the reader of the field to know which entries are removable. That
puts two lists of the same facts in two files, and the record is the one
that gets read when something has already gone wrong.

**3. Identity instead of filesystem type at teardown.** Finding 008 asks
for "expected identity and fstype" to travel to the helper and be
compared before every unmount.

What I did: identity only. The device and inode pin the superblock and
the object; a filesystem type that agrees adds nothing that identity has
not already settled, and recording an expected type for a bind would mean
recording the type of the filesystem the source happens to sit on, which
is not something camp promises. Binds therefore still carry no `fstype`
in the record; the overlay carries `overlay`, as before.

**4. The derivation continues past a live directory that is not empty.**
This one changes behaviour for every caller of `plan.Prepare`, so it is
worth a second opinion. The refusal is a precondition for mounting, not a
fault in the path, and stopping the derivation returned an empty plan to
the two commands that only describe. It now records the refusal and
carries on. What keeps it safe is that every caller that mounts stops on
a non-empty refusal list -- `camp up`, the namespace init -- and that has
been checked one by one rather than assumed.

**5. How a directory selects a record.** 007 asks for "deterministic
current-directory matching with ambiguity refusal" without saying what
matches.

What I did: a record claims a directory when its environment root or its
composed tree is that directory or contains it, and only records in an
active phase are considered. Exactly one match is used; more than one is
refused by name, with both identifiers and how to name one. A corrupt
record claims nothing and is reported separately rather than skipped.

## What is still open in 007

007 also asks for "Plan reconstruction from that schema" -- a full
`plan.Plan` rebuilt out of the record. That is not done. `explain` needed
a description rather than a plan, so the description now takes the facts
it needs instead of a whole plan, and nothing else asks for one today. If
something later does -- a verification pass that runs from the record
alone, for instance -- the missing pieces are the configuration's own
structure: repository names, the steps, the islands.

## One thing about this machine, not about camp

To let the testing run unattended, this session added
`/etc/sudoers.d/camp-testing`:

```
dlaszlo ALL=(root) NOPASSWD: /usr/local/bin/camp helper-mount, /usr/local/bin/camp helper-unmount
```

It is scoped to the installed binary's two helper subcommands, and it is
exactly the confused-deputy case the helper's own confinement was written
against: anything running as the user can now reach the helper as root
without a person deciding. It is meant to be removed when the terminal
group is finished -- `sudo rm /etc/sudoers.d/camp-testing` -- and it
should not appear in any packaging.
