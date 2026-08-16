# What a copy-up leaves behind, in the privileged mode

Date: 2026-08-16. Spec §23 asks for this as an open measurement: the
`user.overlay.*` side was already measured, and this is its counterpart —
what the mount made as root leaves on disk. Measured on this machine, in
a throwaway environment built for it, on ext4.

## Getting a copy-up to happen at all

The first finding is that camp's own compositions cannot produce one, and
that is by design rather than by accident. Every workspace root entry
carries a derived read-only bind, so a write to any of them fails
`EROFS` before the overlay is involved — which is exactly what those
guards are for. In this environment there is no copy-up candidate at all:
each of the six workspace root entries is either bind-mounted or shadowed
by the code repository.

One hole remains, and it is CAMP-REVIEW-005: a directory named in
`allow_overlap`. Inside it the workspace's files are visible through the
overlay and nothing guards them, so a write copies the file up into the
code repository. Constructed and confirmed: appending one line through
the composed tree to a file that existed only in the workspace left the
workspace's copy untouched and produced a full copy in the code
repository.

## The attributes

Read as root, because `trusted.*` is invisible to everybody else — an
ordinary user's `listxattr` returns an empty list on all of these paths.

| path | attributes |
|---|---|
| the copied-up file, in the code repository | `trusted.overlay.origin` (30 bytes) |
| the directory holding it | `trusted.overlay.origin`, `trusted.overlay.impure` = `y` |
| **the code repository's own root** | `trusted.overlay.uuid` (16 bytes), `trusted.overlay.impure` = `y` |

`origin` is the file handle of the lower object the copy was made from.
`impure` marks a directory whose entries are not all purely the upper
layer's, so that reading it consults the lower layers per entry.
`uuid` is the overlay's own identifier, stamped on the upper root at
mount time, which ties those stored handles to one overlay instance.

**They survive the teardown.** Re-read after `camp down`, with nothing
mounted: all three are still there. They live in the inode, and camp
removes nothing.

So one privileged `camp up` leaves the code repository's root carrying
`trusted.overlay.uuid` and `trusted.overlay.impure` permanently. Git does
not see them, `git status` is silent about them, and a user without root
cannot even list them. Whether that matters is a decision this file does
not make; what it does is stop it being a surprise later.

This belongs in `constraints.md` as a measured fact. It is not written
there because that file's numbering is the owner's (CAMP-REVIEW-017).

## Two things measured on the way past

**The gate catches the residue at the next up.** A copied-up file now
exists on both sides inside the allow-listed directory, and the next
`camp plan` refuses until somebody decides which copy is the real one.
That is the check §20 promises, working: the trace of a copy-up is a name
on both sides of a merged directory.

**And its advice was wrong**, which is repaired. It named the file and
then told the reader to add the *directory* to `allow_overlap` — which
was allow-listed already, and would have changed nothing, because
`allow_overlap` names root entries and the check deliberately moves one
level down rather than switching off. The refusal now says what the
repair actually is.
