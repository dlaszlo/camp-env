# Constraints

What the kernel, git and the environment impose. None of these is a
choice; every design has to be built around them.

Kept apart from `model.md` on purpose. That document records what we
*decided*; this one records what we *found*. A design argument that
contradicts something here is wrong, and a design argument that
contradicts something in `model.md` is merely a different opinion.

Each item says how it is known: **measured** on this machine, **read**
from documentation, or **assumed** and still unverified.

---

## OverlayFS

**C1. Copy-up cannot be disabled.** *(measured, kernel 7.0.0-29)* The
overlay module offers `redirect_dir`, `redirect_always_follow`,
`xino_auto`, `index`, `nfs_export`, `metacopy`, `redirect_max`, and
`check_copy_up` — which is documented as "Obsolete; does nothing". None
prevents copy-up. `metacopy` defers only the *data* copy; metadata is
still copied up. Writing to a lower-layer file through an overlay with an
upper layer will copy it into the upper.

**C2. An overlay without an upper layer is read-only.** *(read)* This is
the only overlayfs-native way to have no copy-up, and it makes the entire
mount read-only.

**C3. Directories merge, files do not.** *(read, and relied upon)* A path
present in several layers: a file takes the topmost layer's version; a
directory shows the union of every layer's entries.

**C4. Deleting lower content writes into the upper.** *(read)* Removing
an entry that comes from a lower layer creates a whiteout — a character
device with major/minor `0:0` — at that path in the upper layer. Hiding a
lower directory's contents where the upper has the same directory sets
`trusted.overlay.opaque=y` on the upper directory.

**C5. Copy-up leaves an xattr trace.** *(assumed — worth measuring)*
`trusted.overlay.origin` is typically set on the copied-up file in the
upper, and `trusted.overlay.impure` on the containing directory. Exactly
when depends on the kernel's `index` / `xino` / `nfs_export`
configuration. `trusted.*` is only visible with `CAP_SYS_ADMIN`.

**C6. The work directory is constrained.** *(read)* It must be empty, on
the same filesystem as the upper directory, and not inside it. The kernel
leaves a root-owned `work/` inside it after use.

**C7. `lowerdir` is leftmost-wins, and its syntax is fragile.** *(read)*
`:` separates layers, `,` separates mount options, `\` escapes both. A
path containing either character must be escaped.

**C8. One upper directory cannot serve two overlay mounts.** *(read)*
Sharing an upperdir between mounts is unsupported and can corrupt data.
One composition per base repository at a time.

**C9. The xattr namespace depends on privilege.** *(read)* A privileged
mount uses `trusted.*` and wants `nouserxattr`; a mount made inside an
unprivileged user namespace cannot write `trusted.*` at all and requires
`userxattr`.

## Bind mounts

**C10. The target must already exist.** *(measured, by failure)* A bind
cannot create its own mount point. This is why a writable path needs a
directory or file to attach to, and why a machine-local file that no
repository will ever track has nowhere to attach without a placeholder
being provided.

**C11. A bind is writable until remounted read-only.** *(read, and
tested)* Read-only is not inherited from anywhere; it takes a second
`mount -o remount,bind,ro`. A one-step version is silently writable.

**C12. A bind follows symlinks.** *(read)* Binding a symlink binds what
it points at. A symlink in a contributed tree could therefore pull in
content from anywhere on the machine, silently.

**C13. Type must match.** *(measured, by failure)* Binding a directory
onto a file fails with `ENOTDIR`, and the reverse fails too.

**C14. Mounts propagate by default.** *(measured — cost an afternoon)* On
a systemd machine `/` has shared propagation, so a bind joins a peer
group with its source, and every later mount *inside* the bind propagates
back to the source. Twelve mounts appeared where eight were planned, four
of them on the workspace repository's own path. Each mount must be made
private as it is created.

**C15. Mounting requires `CAP_SYS_ADMIN`.** *(measured)* Root, or a user
namespace the process created itself.

**C16. A mount cannot be removed while something holds it.** *(measured,
twice)* A process's working directory, an open descriptor, its executable
or its root inside the tree all count. A session rooted in the composed
tree cannot unmount the tree it is standing in.

## User namespaces

**C17. Unprivileged user namespaces may be forbidden.** *(measured)*
Ubuntu 23.10 and later set
`kernel.apparmor_restrict_unprivileged_userns=1`. An AppArmor profile can
grant the permission, but it attaches to the **binary's path** — so a
copy of the same executable run from elsewhere is not covered, and an
interpreted program would need the profile on the interpreter.

**C18. Only one id can be mapped, and it has to be 0.** *(read, and
reasoned)* An unprivileged user namespace may map a single uid without
`newuidmap`. It must map container 0 to the caller, because capabilities
are dropped on `execve` for a non-zero euid without file or ambient
capabilities — so a process mapped to its own uid could not mount after
exec. Consequence: inside the namespace you are uid 0, and that uid *is*
your identity outside; files you create are yours.

**C19. Supplementary groups are lost.** *(read)* `setgroups` is denied,
so group membership does not survive. Access that depends on being in the
`docker` group — the socket is `root:docker 660` — does not work inside.

**C20. Mounts are invisible outside the namespace.** *(read)* A daemon
running outside — `dockerd`, an editor started from the desktop — cannot
see the composed tree, and a bind-mount request naming a path inside it
means nothing to that daemon.

**C21. The namespace and its mounts vanish with the last process.**
*(read)* Teardown cannot fail, and nothing is left to clean up. This is
the mode's main advantage and the reason it needs no `down`.

## Git

**C22. Git does not store extended attributes.** *(read)* An overlay
origin marker or an opaque flag never reaches a commit.

**C23. Git cannot track an empty directory.** *(read)* A mount point that
must exist in a repository needs a file in it, or it cannot be committed.

**C24. A worktree's git directory is an absolute path, compared as a
string.** *(measured)* A worktree registered from the composed tree can
only be administered from there, and shows as `prunable` once the
composition is down — while its checkout stays intact.

**C25. `.git/info/exclude` is not a boundary.** *(measured)* It keeps
paths out of `git status` and `git add .`; `git add -f` stages them
anyway. Only a hook can refuse the commit.

**C26. Covering a tracked path breaks the working tree's status.**
*(reasoned, decisive)* If a bind covers a path the base repository
tracks, git reports those files as deleted — or, for a file with
different content, as modified — and `git commit -a` would record it.

**C27. Linked worktrees share the common git directory.** *(measured)*
`info/exclude` and hooks installed there apply to every linked worktree
at once. A hook that matches on absolute paths therefore fires inside
every worktree, including on ordinary source files.

**C28. Git cannot represent a character device.** *(read)* A whiteout
left in the working tree is an untracked oddity that `git add` refuses,
and it has to be removed by hand.

## Environment and instruments

**C29. Command output is translated.** *(measured)* `LANG=hu_HU.UTF-8`
here, so `umount`'s "not mounted" never appears. Anything parsed from a
command's output must run under `LC_ALL=C`, and state is better asked of
`/proc` than read from a message.

**C30. `fuser -m` answers a different question.** *(measured)* On a bind
mount it reports every process using the underlying *device*: a hundred
and twenty processes where two held the mount. `/proc/*/cwd`, `/proc/*/fd`
and friends name the process and are device-agnostic.

**C31. `findmnt -R <dir>` fails for a directory that is not itself a
mount point.** *(measured)* A worktree never is — the mount is its parent.
`findmnt -T` asks which mount the directory sits on, which was always the
question.

**C32. A locale-aware sort and a byte-wise comparison disagree.**
*(measured)* `comm` compares bytes; `sort` uses the locale. Together they
silently fall out of step: a namespace collision check reported two of
four real collisions and called the rest a warning on stderr. Any
comparison of file names must sort by bytes.

**C33. `sudo` cannot prompt without a terminal.** *(measured)* A
subprocess with no stdin gets no password prompt; the command simply
fails.

**C34. A mount point cannot be renamed.** *(measured, kernel 7.0.0-29)*
`rename(2)` answers `EBUSY` for a directory that has something mounted on
it. So a name camp mounted on is pinned for as long as the mount stands,
and the swap the review's rename race is about can only be made at an
*ancestor* of a mount point, never at the mount point itself. That
narrows what has to be defended and it is the reason the barrier for that
race is placed at the resolutions rather than at the mounts.

**C35. `umount2` on a descriptor's own `/proc/self/fd` name cannot be a
descriptor-safe unmount, for two opposite reasons.** *(measured, kernel
7.0.0-29)* A descriptor opened on the mount *after* it exists holds a
reference to that mount, and the reference is what makes it busy:
`umount2` answers `EBUSY`, and still does after an ancestor has been
renamed. A descriptor opened on the mount point *before* the mount does
unmount it — but only because resolving the `/proc/self/fd` name crosses
into the mount, so the answer came from path traversal and not from the
descriptor. Holding a descriptor therefore buys nothing here: the first
case cannot act and the second does not obey.

**C36. A mount can be moved by descriptor and unmounted where it lands.**
*(measured, kernel 7.0.0-29)* `open_tree(path, 0)` gives a handle on the
mount at a path; `move_mount` with `MOVE_MOUNT_F_EMPTY_PATH` moves *that*
mount — the same mount id arrives — into a directory named by another
descriptor, leaving the original path clear; and once no descriptor pins
it, `umount2` at the new path removes it. This is what makes a
descriptor-safe teardown possible after C35 rules out the obvious route:
the *choice* of what to remove is made on a descriptor, so a name swapped
after the identity check cannot redirect it, and the unmount itself
happens at a path only root can name.

---

## Where these came from

Most of the measured items were found by running the tool against real
directories rather than by reading its code — four of them in a single
afternoon, none visible from the source and none reproducible in the
proof of concept's synthetic layout. That is worth remembering when the
next constraint is suspected but inconvenient to verify: **the synthetic
case measures what you designed into it; the real one measures what you
did not.**
