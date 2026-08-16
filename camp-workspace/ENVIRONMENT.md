# The tree you are working in

Read this before your first write. It is short, and it is about something
you cannot see from inside: **this directory is not one project. It is
several repositories composed into one tree**, and they do not all accept
writes.

Nothing here is a permissions accident. Every read-only path in this tree
is read-only on purpose, and working around one causes exactly the damage
the arrangement exists to prevent.

## Find out, do not guess

```
camp explain
```

**If you take one thing from this file, take that command.** It describes
*this* tree — every path, where the real file behind it is, what is
writable and what is not, and why — and it is generated from the live
configuration rather than written by hand, so it cannot be out of date.

That is the whole reason this file is short. A document listing which
directories are writable would be wrong the first time somebody changes
the composition, and you would have no way of knowing. `camp explain`
answers from the composition that is actually running. This file teaches
you the shape and the habit; that command supplies every fact.

`camp plan` answers the other question: what this composition is made of,
in order, with the reason for each piece.

## The three kinds of place

**Almost everywhere: the code repository.** This is the product, and it is
where ordinary writes land. A file you create anywhere the composition has
not covered is a file in that repository. This is the normal case and you
need to think about it no further.

**Paths the workspace provides: read-only.** These carry the development
environment — instructions, agent definitions, skills, notes about the
work. Writing one through this tree fails with `EROFS`, and that is the
design working. Without it, the write would copy the file up into the
*product's* repository, where the change would look applied while living
in the wrong history.

To change one of them, edit the real file at its own path — `camp explain`
prints that path — from outside the composition. The change appears here
immediately, because what you are looking at is a live view and not a
copy.

**Machine-local storage: writable, and belongs to no repository.** Runtime
files, local settings, worktrees. They survive the session and they are
never committed anywhere, because there is nothing to commit them to.

A composition may also mount another repository writable at its own path —
a record repository, a design record. Writes there land in *that*
repository. `camp explain` names them.

## When a write fails

**Do not work around it.** Not with `chmod`, not with `sudo`, not by
remounting, and above all not by copying the file somewhere else and
editing the copy. That last one is the failure this whole arrangement
exists to prevent: a change that looks applied and exists in no repository
at all.

Say what you needed to write and which path refused you. That is a
complete and useful answer. If the file genuinely has to change, it
changes at its own path, outside this tree, and the person you are working
with can do that in seconds.

## git in here

The `git` you run in this tree is the **code repository's** git. Its
history is the product's history.

Paths the workspace provides are excluded from it deliberately, so
`git status` stays quiet and `git add .` cannot pick them up. That
exclusion is convenience, not a boundary: `git add -f` still reads such a
file through this tree and stages its bytes. **Never force-add a path the
exclude covers.** It is detected when the session ends, not prevented, and
by then it is in a commit.

A git worktree created from in here records this tree's path, and git
compares those paths as strings — so it stops resolving when the
composition comes down. The files are fine; camp prints the exact repair
command at the end of the session.

## Who you are in here

If the session was built without privilege, only your own user id is
mapped into it. Every file on the machine owned by anyone else — root
included — is shown as `nobody`. Reading and writing are unaffected; what
changes is what a program sees when it asks who owns a file.

Some programs refuse a system-wide configuration file they cannot
attribute to root or to you. `ssh` is the one you will meet, and it fails
before it opens a connection. This is not broken permissions and there is
nothing to repair on disk. `camp explain` says what to do about it.

## What this is not

**It is not a sandbox.** The read-only mounts stop accidental writes and
copy-up. You can still walk to the backing directories and read anything
on the machine. Do not confuse "read-only" with "contained".

**Nothing here is hidden from you.** Every path in this tree has a real
file behind it somewhere, and `camp explain` will tell you where. If
something surprises you, that command is the first place to look, not the
last.
