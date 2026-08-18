# The output work is finished, and the review's code findings are closed

Date: 2026-08-18. camp from `e619e9b` to `ca6fabd`, twenty-three commits.
Both gates green with nothing skipped: `go build ./... && go vet ./... &&
gofmt -l . && go test ./...` on the host, and
`camp run -- go test ./internal/... -count=1` through the installed
binary, checked with `-v` for `--- SKIP` (none).

## The output work: the three pieces that were missing

**Refusals group.** A rule that can fire more than once now says itself
in three parts -- an opening for one subject, an opening for several, and
the explanation both share -- and the subjects gather onto one refusal.
The explanation carries nothing about any particular subject, which is
what makes the gathering possible at all. Converted: the overlap gate,
the per-mount sequence checks, the per-repository and root-entry checks,
the inventory's blocking differences, and the per-mount verification.
Measured on a composition with four overlapping root names: one refusal,
four paths, one explanation, and the count a command prints is problems
rather than subjects.

**The log.** `internal/logs`: every line camp writes to stderr goes to
`$ENV/.camp/logs/camp.log` as well, with an RFC 3339 local timestamp,
rotated at a megabyte with three files kept. Always on. Two processes
write it -- the launcher and a session's init -- so writes append under a
lock on the directory and a writer that finds the file rotated away
reopens it; measured with two writers over a rotation. No framework, for
the reasons the earlier handoff gives.

**Colour**, on the marker alone, when the stream is really a terminal,
`NO_COLOR` is unset and `TERM` is not `dumb`. The colouring is put on per
sink as the line goes out, so the log keeps the plain words -- measured
through a pseudo-terminal, escapes in the terminal's copy and none in the
file's. The width is still never asked for.

The three share one thing worth knowing: `report.Sink`, the line-oriented
writer every command's stderr now goes through. It is where the two ends
of a line differ, and the only place that knows about either.

## The review: every code finding is repaired

In the verification's order. 015 was already done (the locked flags are
read with `fstatfs` on the descriptor camp holds).

- **003** -- fsx claimed no area could be built from a repository path,
  and that was true of the constructors and of nothing else: the areas
  joined strings and called MkdirAll, OpenFile, Rename and a recursive
  remove, all of which follow symlinks. An area is now a base camp trusts
  plus the components below it, and every operation resolves them with
  `openat2` under `RESOLVE_BENEATH|RESOLVE_NO_SYMLINKS`, in the call that
  acts. Tests: a planted link, a link inside an area, a swap after the
  look, a state directory linked into a repository. `camp up` also
  refuses to keep its records inside a repository, which `XDG_STATE_HOME`
  can name and no filesystem check would notice.
- **004** -- the overlay's operands crossed into root as bare paths. The
  composed tree is mounted through the kernel's mount API now, with each
  layer a descriptor, and the final move is `move_mount` on pinned
  descriptors. The helper opens every operand beneath its base and
  compares it against the identity the front end recorded; a job carrying
  nothing to compare against is refused before its first syscall.
- **005** -- the exclude now names each workspace path inside an
  allow-listed directory, which is the one place the specification asks
  for file-level enumeration and the implementation had none. The gate's
  descent and the exclude's lines come out of one walk. An allow-listed
  directory also warns: it is the one place nothing is read-only.
- **012** -- the session's init compares the inherited lock descriptors
  with what its own plan says, by identity. Measured with a configuration
  swapped between the launcher and the init.
- **011** -- `gitwire.Open` answers three states. A git that cannot
  answer is no longer "this is not a repository", so the rule about
  covering tracked content cannot pass without running.
- **009** -- a composition without a generation step is prepared as fully
  as one with it, and asks git nothing at all.
- **010** -- a generator's islands are compared with what the source
  contributes, exactly, in both directions.
- **016** -- the steady-state guard against a second composition on one
  upper compares inodes, not upperdir strings.
- **018** -- every raw workspace root name is judged before any
  exemption, and a descent that cannot read a side refuses.
- **019** -- an attachment point camp cannot look at stays camp's; the
  object goes before the record; the in-memory copy follows the write.
- **020** -- a scan that could not run says so, and a stale root listing
  is never reused as if it were current.
- **021** -- the namespace probe builds a real overlay, writes through it
  and removes through it. The privileged mode says "not tested; it needs
  a terminal", because answering it would mean running sudo to find out
  whether sudo works.
- **022** -- reports are named to the nanosecond and claimed with
  `O_EXCL`; marking is one rename; a delivery that fails is said.
- **023** -- SIGINT, SIGTERM and SIGHUP reach the generator's process
  group, and camp waits for it. Measured with a real signal and a
  grandchild.
- **024** -- only ENOENT means absent, in storage ownership and in the
  sweeper. Ownership checks each store and island attachment point;
  everything else under storage is deliberately not walked.
- **025** -- a mountinfo line that does not parse fails the whole read.
- **026** -- ENOTDIR is no longer absence, so a file shadowing a
  directory is refused on paper rather than at mount time.
- **027** -- worktrees are read in git's NUL form and the repair command
  is shell-quoted; measured by handing each quoted argument to `/bin/sh`.
- **028** -- `status` goes through `compose.Check`; `compose.CleanWork`,
  `preflight.tool` and the two dead `session.Options` fields are gone.

## Two measured facts for constraints.md

Both from a real overlay mounted inside a namespace, and both decided the
shape of 004. They want numbers in `constraints.md`, which is the owner's
file (finding 017).

**The old mount API accepts `/proc/self/fd/N` as an overlay operand and
records that string.** The mount is made against the right object, and
`/proc/self/mountinfo` then says `lowerdir=/proc/self/fd/6` for the life
of the mount -- so nothing afterwards, camp's own verification included,
can see what was mounted.

**The kernel's mount API takes the layers as descriptors and records the
real paths.** `fsopen("overlay")`, `fsconfig(FSCONFIG_SET_FD,
"lowerdir+"/"upperdir"/"workdir")`, `fsmount`, `move_mount` all succeed
unprivileged inside a user namespace, and mountinfo shows
`lowerdir+=<the real directory>`. The key is `lowerdir+`, not `lowerdir`,
which the verification now reads. `move_mount` with
`MOVE_MOUNT_F_EMPTY_PATH|MOVE_MOUNT_T_EMPTY_PATH` moves a whole tree with
its submounts, which is what replaced `MS_MOVE`.

Related: `fsopen("overlay")` answers `EPERM` rather than `ENOSYS` for an
unprivileged process outside a namespace, which is what `doctor`'s new
mount-API check distinguishes.

## The ssh launchers, and what they measured

`.workspace/bin/{ssh,scp,sftp}` are in the workspace repository, in the
shape `docs/install.md` specifies, and the inventory is re-accepted for
the new root entry.

Measured inside a session: `ssh` and `scp` resolve to the launchers;
raw ssh reached through `$OUTER_PATH` fails with *Bad owner or
permissions on /etc/ssh/ssh_config.d/20-systemd-ssh-proxy.conf*, and the
launcher's ssh answers `ssh -G nas` with status 0. Outside the session,
`command -v ssh` is `/usr/bin/ssh` and `ssh -G` is clean -- camp touched
nothing on the host.

What is still the owner's: the run against a real peer (`ssh <host>
hostname`, `scp`, `sftp -b /dev/null`, `git ls-remote` over ssh). It
needs credentials and host-key decisions.

## What is still open

- The **installed binary is behind**: `camp run` composes with the camp
  from `/usr/local/bin`, so nothing measured through it exercised today's
  code except the tests themselves. Reinstalling is the owner's move, and
  it is what the privileged-mode measurements need.
- **The privileged mode has not been run since 004 changed how it
  mounts.** The overlay goes through the mount API and the move through
  `move_mount` now; both are exercised by the namespace suite, and the
  helper's own path is not. That run needs a person and a terminal.
- The **rename race** as a race, the single-instance GUI editor handoff,
  the identity spike, and the keyring measurement: unchanged from the
  last handoff.
- **014** and **017** are documents and the owner's, unchanged.
- The `.camp` rearrangement, the inventory-and-gitignored-name question,
  and the five readings in the recovery log: unchanged.

## One thing to know about the history

Two commits were made from the wrong directory during this session. The
result was caught and repaired: `camp`'s history was split so that the
session-lock repair and the git-state repair are one commit each, and the
environment repository's commit message was corrected to describe what it
actually contains. The tree after the split is byte-identical to the tree
before it.
