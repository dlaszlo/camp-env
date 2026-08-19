# The checks that need an install, and the ones that need a terminal

**The namespace group is no longer waiting.** On 2026-08-16 camp was
installed at `/usr/local/bin/camp` with its profile, and the whole suite
was run through it — `camp run -- go test ./internal/... -count=1`, which
opens the namespace with the installed binary and lets the test binary
create its own inside. Every package passed and **nothing skipped**.

**The privileged lifecycle is no longer waiting either.** On 2026-08-19,
after the branch that answered the twelve-finding review changed how
every bind is made, `camp up` and `camp down` were run once by hand on a
scratch composition and both completed —
`log/2026-08-19-the-privileged-mode-on-the-new-mount-path.md` has the
mount table, the record and the teardown. What is still waiting is the
part of the terminal group that is about failure rather than about
working: **the kill matrix and the rename race**. The small experiment
that stood in front of them has run, and it decided the question -- C34
to C36 in `constraints.md`, and the descriptor-safe teardown is written
on C36.

**Both now have an instrument.** `drivers/killmatrix` and
`drivers/renamerace` are written and are waiting for somebody to run them
at a terminal; `drivers/README.md` says how. Neither can run unattended,
for the terminal reason below, and neither has been run. Those two, and
the person-gated measurements at the end, are what is left.

That first real run earned its keep immediately: two source guards had
been walking the module's directory tree, which inside a composition also
carries the notes repository at `.notes`, whose design documents quote the
very strings the guards forbid. They passed on the host and failed in the
composition. They now ask git what this repository tracks.

Everything else in this file is written, runs, and is waiting. It cannot
run from a checkout on this machine, for two reasons that have nothing to
do with the code.

**The namespace gate.** `kernel.apparmor_restrict_unprivileged_userns=1`
lets any binary create a user namespace and then confines the process
inside it to the `unprivileged_userns` profile, which denies every mount.
The permission is granted by profile to **one installed binary path**, so
a binary built in the checkout is not that path. Every test that needs a
real composition skips with the install commands in its message.

**The terminal gate.** sudo cannot prompt without a terminal, so nothing
that elevates can run unattended. That is the specification's own
position — the privileged mode is always driven from a real terminal by
the user — and not a limitation to work around.

## Unlock the first one

From a terminal outside any session:

```
cd ~/dev/camp-env/camp
go build -o camp ./cmd/camp
sudo install -m 755 camp /usr/local/bin/camp
sudo install -m 644 packaging/apparmor/camp /etc/apparmor.d/camp
sudo apparmor_parser -r /etc/apparmor.d/camp
camp doctor          # the namespace line should read "permitted, and a mount inside one succeeds"
```

Then, to run the namespace group, let the installed binary open the
namespace and run the tests inside it:

```
cd ~/dev/camp-env && camp run -- go test ./internal/... -count=1
```

`go test` run directly from the checkout still skips these: the profile
grants the permission to one binary path, and the test binary is not it.
A skip is not a pass.

## Install-gated: the namespace

**Run and passed through the installed binary on 2026-08-16**, in the
composition camp's own environment builds. They skip with an explanatory
message when run from a checkout.

| check | where |
|---|---|
| route A: real user inside, ambient capability survives execve, mounts succeed, capabilities gone after the drop, overlay still works, supplementary groups still grant host access | `internal/nsx` — `TestRouteAKeepsTheUserAndGivesTheCapabilityBack` |
| the whole composition passes every check; guarded paths `EROFS`; new files land in the code repository; teardown clean; no residue | `internal/compose` — `TestTheCompositionPassesEveryCheckAndBehavesAsDesigned` |
| a composition on `/tmp` (nosuid tmpfs) passes with the locked flags replicated, and the omitted-flags remount reproduces the `EPERM` | `internal/compose` — `TestACompositionOnTmpfsWorksWithTheLockedFlagsReplicated`, `TestACompositionOnTmpfsPassesEndToEnd` |
| a deliberately shadowed mount is caught by identity | `internal/compose` — `TestAShadowedMountIsCaught` |
| the exclude is visible only through the composed tree | `internal/compose` — `TestTheGeneratedExcludeIsVisibleOnlyThroughTheComposedTree` |
| an island is read-only and the water around it is not | `internal/compose` — `TestAnIslandIsReadOnlyAndTheWaterAroundItIsNot` |
| up / down / up with islands: the second run accepts its own scaffolding | `internal/compose` — `TestARepeatedSessionAcceptsItsOwnScaffolding` |
| a teardown held by a process fails loudly, names it, and leaves the mount | `internal/compose` — `TestATeardownHeldByAProcessFailsLoudlyAndNamesIt` |
| a foreground command's exit status propagates | `internal/session` — `TestAForegroundCommandsExitStatusIsPropagated` |
| a signalled workload reports 128 + the signal | `internal/session` — `TestASignalledWorkloadReportsTheShellsConvention` |
| the tree exists inside and the live directory is empty outside | `internal/session` — `TestTheCompositionIsBuiltInsideAndInvisibleOutside` |
| a daemonised workload returns at once while the init holds the locks | `internal/session` — `TestADaemonisedWorkloadReturnsWhileTheInitHoldsTheLocks` |
| the locks survive a daemonised tmux server (spec §22 stage 2) | covered by the row above, with `setsid` standing in for tmux; run the tmux form by hand once, below |
| a declared variable reaches a direct workload, a shell and a daemonised descendant; `/proc/1/environ` proves it absent from camp-as-init; `/proc/self/status` proves the workload holds no capability | `internal/session` — `TestADeclaredValueReachesTheWorkloadAndNotTheInit` |
| a workspace executable reachable only through the declared `PATH` is what a bare command name selects | `internal/session` — `TestABareCommandIsFoundThroughTheDeclaredPathInASession` |
| nothing declared survives the session: the calling process's variables are unchanged and the live directory is empty outside | `internal/session` — `TestNothingDeclaredLeaksOutOfTheSession` |

The three rows above are the session environment's own group (spec §6,
§14). Everything else about that feature runs unattended: the grammar and
resolution tables in `internal/envx`, the section's parsing and refusals in
`internal/config`, the rendering and redaction rules in `internal/report`,
the workload construction and the `CAMP_SESSION` source guard in
`internal/session`, and the OpenSSH arrangement — launchers, `-F` on every
entry point, a git fetch through the declared command — in
`internal/session/openssh_test.go` against fake programs.

## Terminal-gated: the privileged mode

sudo needs a terminal, so these are for the owner to run by hand.

**`camp up` and `camp down` were each run once, by hand, on 2026-08-16,
and both completed.** That first run was worth the whole exercise: the
mode had never been executed before, and it carried six defects — four
that the reviews had found and two that only running it could show, the
shared parent that `MS_MOVE` refuses and the shared destination that marks
the moved mounts shared. All six are repaired, and a seventh, a failed
`up` leaving root-owned residue nothing could clear, with them.

Run, in an environment whose `camp plan` is clean:

```
camp up
```

and check, from a **second terminal**:

1. **Staging invisibility.** While `camp up` is running, `ls <live>` in
   the second terminal shows nothing until the move completes.
2. **The machine-wide workspace freeze.** Once it is up,
   `touch <workspace>/x` from the second terminal fails with `EROFS`.
   `camp up` prints this as its first line, so it should not be a
   surprise.
3. **sudo exactly once.** The password is asked for once per command, and
   `ps` shows sudo wrapping `camp helper-mount` and nothing else.
4. **`sudo camp up` is refused** with the message to run it unprivileged.
5. **`trusted.overlay.*` forensics** (spec §23): after a copy-up, look at
   `getfattr -d -m - <upper>/<the file>` and record which attributes the
   privileged mount leaves. The `user.overlay.*` side is already
   measured; this is its counterpart.
6. **The kill-point matrix** (spec §12, §22 stage 4). With the
   composition up, delete the configuration file, then:
   `camp status` must describe the composition from the record alone, and
   `camp down` must take it apart from the record alone. Repeat with a
   `kill -9` of `camp up` injected at each boundary — after the record is
   written, during the helper's mounts, after the move — and check the
   same two commands converge each time. The record's phase says which
   boundary was reached.
7. **The rename race.** Between validation and the mount, replace a
   component of a mount source with a symlink; the helper's
   descriptor-relative resolution must refuse it with "is not the object
   camp checked". Constructing this by hand needs a slow filesystem or a
   debugger breakpoint; the code path is `privileged.checkIdentity`.

### Run again on the descriptor mount path, 2026-08-19

And once more, because every bind changed again: the privileged mode now
takes a detached copy of the source with `open_tree`, attaches it to the
checked target descriptor with `move_mount`, and addresses the mount it
made rather than the name it landed on. `camp up` and `camp down` both
completed on a scratch composition of five mounts, built from
`fix/review-8ddf464` at `f2ca05a`.

What the kernel's own table said, and what each part of it settles, is in
`log/2026-08-19-the-privileged-mode-on-the-new-mount-path.md`: the
composed tree records the real paths and not `/proc/self/fd/N`; the
read-only remount and the propagation change made through the clone's
descriptor both take; the clone is the one mount `MS_BIND` would have
made and is not recursive; and the composed tree's parent is the live
self-bind. The teardown was run with the configuration moved aside while
the composition was up, and worked from the record alone: no mount left,
the work directory removed, the record discarded, both repositories at
their original `HEAD` with nothing modified.

Of the numbered checks above, this run covers 1, 2 and 3, and confirms
that a teardown needs no configuration. **6 and 7 are not covered**: the
successful path is what was measured, and those two are about the
interrupted and the adversarial one.

### Run again on the new mount path, 2026-08-18

The results further down were measured against syscalls camp no longer
makes: the composed tree is now made with the kernel's mount API, the
layers as descriptors, and the move is `move_mount` on two pinned
descriptors rather than `MS_MOVE` on two names. So the whole lifecycle
was run once more, with the binary built from `3148185`.

**`camp up` and `camp down` both completed, and the new path is in the
kernel's own words.** The composed tree's line reads

```
overlay none rw,lowerdir+=<workspace>,upperdir=<code>,workdir=<work>,nouserxattr
```

-- `lowerdir+` and the source `none` are the mount API's signature, and
the paths recorded are the real ones rather than the `/proc/self/fd/N`
strings the old API would have kept for the life of the mount. The
post-move verification passed on all eleven mounts, which is also the
first time its reading of `lowerdir+` was exercised under root. Every
mount is private. `move_mount` carried the ten submounts with the tree.

**The teardown was complete**: twelve of twelve removed (the eleven
recorded, plus the self-bind the move needed), the composed tree's
directory empty afterwards, the workspace writable again, the record
forgotten, and the kernel's root-owned `work/` gone -- which is the
helper's fsx path running as root, resolving component by component
beneath the job's base and following no symlink. Nothing under `.camp` is
root-owned.

**Item 2 re-measured**: `touch` and `mkdir` in the workspace from a
process outside the composition both fail `EROFS` while it is up, and
succeed again after `down`.

**Item 1 re-measured, and it holds on the new move.** A sampler outside
the composition, listing the live path about 230 times a second and
printing only when the listing changed, saw `entries=0` and then one jump
to the whole tree 358 ms later. No intermediate value: `move_mount` makes
the tree appear at the live path in one step, exactly as `MS_MOVE` did.

**Item 6, first half: the record alone is enough.** With the composition
up, the configuration file was moved aside. `camp status` described the
composition from the record, said the configuration cannot be read, and
said that changes nothing about the teardown. `camp down` then removed
twelve of twelve from the record alone and reported that the drift and
leak scans were skipped because they need the configuration -- which is
the honest form of an omission that would otherwise read as "no drift
found".

**What that run found.** The work directory survived the whole lifecycle:
`work/<hash>/` still held the generated exclude, the islands expansion
and the staging point after a clean `down`, although §12 names this
mode's `down` as what removes it. The function existed with no caller at
all. Repaired in camp `8ddf464`, under the live lock; it wants one more
up-and-down to be measured.

**Item 6, second half: the kill-point matrix, all three boundaries.**
`camp up` was started and killed with `SIGKILL` at each of them; after
each, `camp status` had to describe what was there and `camp down` had to
converge -- nothing mounted, no record, the work directory gone, the live
directory empty, the workspace writable.

1. *After the record, before any mount* (killed the instant the record
   appeared in phase `mounting`): nothing mounted. status named the phase
   and both ways out; down converged.
2. *During the helper's work*: the helper is root's process and not the
   test's to kill, so it finished the whole job -- twelve mounts standing
   with the front end gone and the record still saying `mounting`. status
   said exactly that: every recorded mount present, eleven of them with
   no recorded identity because the record was written before the mounts
   were made, and *the run stopped between the helper's work and the
   check that follows it*. down converged.
3. *At the move*: shell polling could not hit the two milliseconds
   between the move and the front end's next step, so a watcher reading
   mountinfo in a tight loop sent the signal the moment the overlay
   appeared at the live path. Same shape as 2, and down converged.

Worth keeping: in two of the three the record carries **no identities**,
because the front end merges the helper's reply into the record only
after reading it. A teardown that insisted on identities would have
walled the user in behind twelve mounts, which is why the record's
`JobTarget` says a missing identity means "unmount what you were given".
That decision is now measured rather than reasoned.

**Item 7, the rename race -- run, and it takes two swaps to see both
halves.** A watcher replaced the `.notes` mount source in the window
between the front end's identity capture and the helper's mount:

- *Replaced with a symlink*: refused by the resolution itself, before
  there was anything to compare -- *a component of the path is a symbolic
  link: camp-notes in /home/dlaszlo/dev/camp-env*. Nothing mounted, and
  the workspace writable again.
- *Replaced with a different real directory*: refused by the identity --
  *camp-notes is not the object camp checked: it was 64513:6308409 and it
  is now 64513:6295314. Something replaced it between the check and the
  mount. Nothing has been mounted.* The helper removed everything it had
  made before it stopped.

**And the window that is not covered.** Swapping the same source *before*
the job is built -- triggered by the record appearing, which happens
first -- is accepted: the front end takes each operand's identity when it
builds the job, so a directory replaced between validation and that
moment is the one that gets mounted, with the validation having looked at
something else. The composition came up with the impostor at `.notes`.
Whether that window is worth closing (identities taken at validation and
carried forward, or the validation re-run against the job) is a design
question, not a defect in what the helper promises: everything from the
job onwards is exactly what was checked.

**A rolled-back `up` leaves camp's work directory**, because the record
is deliberately forgotten when the rollback is complete and it is `down`
that removes the directory. It is not a leak: the next `camp run` in the
environment sweeps it, which is what happened here, and `doctor` lists it
in the meantime. Worth a decision only if the privileged mode is used
without any namespace session.

Items 3, 4 and 5 are unaffected by the change and were not repeated.

**The sudoers rule is back**, in the same shape and for the same reason
as on 2026-08-16 -- `/etc/sudoers.d/camp-testing`, the two helper
subcommands of the installed binary, nothing else. It is what let the
runs above happen unattended, and it is to be removed when this group is
finished. It is also why a binary built anywhere else cannot elevate: an
`up` from a scratch build is refused for the password it cannot ask for,
which is the rule doing its job.

**And it is still installed.** The group that needed it is finished, and
removing it is the one step of this file a session cannot take: `sudo rm
/etc/sudoers.d/camp-testing` needs a password, and sudo cannot prompt
without a terminal (C33). It waits for one command from the owner's
terminal.

### Item 7 finished: a swap for every operand, 2026-08-18

The half CAMP-REVIEW-004's repair asked for -- a swap exercised
independently for each operand -- is now measured. Two triggers were
enough: the helper's process existing, which is after the front end has
taken every identity and about ten milliseconds before the helper's first
check, and a named mount point appearing in the machine's mount table.
Every run converged afterwards: nothing mounted, no record, camp's work
directory gone, the live directory empty, the workspace writable.

- **The overlay's upper layer**, replaced with a different real
  directory: refused -- *camp is not the object camp checked: it was
  64513:6295166 and it is now 64513:6319724*. Replaced with a symlink:
  refused by the resolution -- *opening the overlay's upper layer ...: a
  component of the path is a symbolic link*.
- **The overlay's work directory**, replaced with a different real
  directory: refused, by identity, naming
  `.camp/work/56a06176bd04/work`.
- **The overlay's lower layer**: refused, but by the check that reaches
  that path first. The workspace is also the frame's first mount -- bound
  onto itself to hold it read-only -- so its identity is compared as that
  mount's source before the overlay's operands are opened at all. And
  once that mount stands, the directory cannot be swapped at all: the
  rename fails *device or resource busy*, measured, with the composition
  coming up normally.
- **The live directory**, replaced with a different real directory
  between the staging tree's last mount and the move: **accepted**. `camp
  up` finished, eleven mounts verified, and the composition stands on the
  directory the swap put there while the one camp validated sits beside
  it, empty. No identity is carried for the live path, and the descriptor
  the move goes to is opened after the swap. What the descriptor covers is
  the interval between that open and `move_mount` -- microseconds. The
  interval since validation is milliseconds, and it is not covered. A
  design question for the owner, beside the other open window.
- **The live directory, replaced with a symlink** pointing outside the
  environment root: refused by the open, rollback complete. The step
  before the open, the self-bind that keeps the move from propagating,
  still resolves the live path **by name** and therefore follows the
  symlink; a sampler reading the mount table 12,000 times a second never
  caught the bind it makes at the symlink's target, and the rollback
  removes it. Recorded, not repaired.

**What this found.** A refusal from the helper's precheck -- before the
first syscall that changes anything -- was reported by `camp up` as
*camp up failed, and what it built is still on the machine*, with an
empty list of what was still mounted and a record left in phase
`partial`, while `camp status` a second later called all eleven mounts
gone. Repaired in camp `5ce1fec`; the end-to-end re-measurement waits for
an install, because the binary that elevates is the installed one.

**And the baseline it began with**: a plain `camp up` and `camp down` on
camp `8ddf464`, which measured the work-directory repair that session
left untested -- `down` removed twelve of twelve and then *removed:
.camp/work/56a06176bd04, camp's own work directory*, and the directory is
empty afterwards.

### What has run, and what it found

On 2026-08-16, in the session after the first one. Running them found
two defects, both recorded in
`log/2026-08-16-recovery-from-the-record.md`: `camp up` failed its own
post-move check on every composition, and a failed `up` reported a clean
machine while the composition stood.

**1, staging invisibility — passed.** A sampler in a second terminal,
printing only when the listing changed, saw `entries=0` and then one jump
to the whole tree at the moment of the move. No intermediate value and no
error: the half-built staging tree is never visible at the live path.

**2, the machine-wide freeze — passed.** From a process outside the
composition, `touch` and `mkdir` in the workspace both fail `EROFS` while
it is up, and both succeed again after `camp down`.

**3, sudo exactly once — passed, from the log rather than from the
prompt.** One `camp up` produces exactly one sudo invocation,
`COMMAND=/usr/local/bin/camp helper-mount`, and nothing else.
`journalctl _COMM=sudo` is where that is read. The prompt itself is not
observable while the testing sudoers rule described in that log file is
in place, and counting invocations is the stronger measurement anyway.

**6, the kill-point matrix — ran, and passed at every boundary.** It
began by failing: with the composition up and the configuration moved
aside, `camp status` and `camp down` both refused for want of a
configuration, while the record held the whole plan. After that repair,
a `kill -9` of `camp up` was fired at each boundary by watching the
machine for it rather than by waiting a length of time -- an `up` takes
50 ms, so nothing else would hit. All five states converged: `status`
described each one from the record alone, and `down` took each apart with
no configuration present. The matrix found two defects in the repairs
themselves; both are written up in
`log/2026-08-16-recovery-from-the-record.md`, along with what the
boundaries turn out to be, which is not quite what §12's sentence says.

**4, `sudo camp up` is refused — passed.** It stops before anything: no
mount, no record, the workspace still writable and carrying no self-bind,
and the message says to run it without sudo.

**The two manual ones passed as well.** `tmux attach` from an outside
terminal onto a session started with `camp run -- tmux new-session -d`
lands in the composed tree: `pwd` is the live path, `ls` shows both
repositories merged, and nine mounts are visible from inside. And Ctrl-C
in a `camp shell` stops the workload and leaves the shell standing --
`sleep 300` interrupted, `$?` is 130, the session ends only at `exit`.

**5, the `trusted.overlay.*` forensics — ran.** It needed a throwaway
environment, because camp's own compositions cannot produce a copy-up at
all: every workspace root entry carries a read-only guard, and the one
hole is a directory named in `allow_overlap` (CAMP-REVIEW-005). What the
privileged mount leaves, and what survives the teardown, is written up in
`log/2026-08-16-copy-up-forensics.md`. The short version: the code
repository's own root ends up carrying `trusted.overlay.uuid` and
`trusted.overlay.impure` permanently, invisible to git and unreadable
without root.

**7, the rename race — the mechanism is tested, the race is not.** The
helper's comparison between what it opened and what the front end checked
now has a unit test, including the refusal's wording and the case where
there is nothing to compare. Constructing the race itself still needs a
debugger or a slow filesystem. 4 needs a person, because it is a `sudo camp
up` and the testing rule deliberately does not cover it. 5 has no
copy-up candidate in this environment: every workspace root entry is
either bind-mounted or shadowed by the code repository, so measuring it
means putting a file in the workspace repository for the purpose, or
doing it in a throwaway machine.

## The one primitive the teardown is still waiting on

`umount2("/proc/self/fd/<a directory descriptor>/<one name>", 0)`.

What it would settle: camp's two self-binds cannot be moved -- the kernel
refuses to move a mount whose parent is shared, and their parent is the
mount `/` is, which is shared on a systemd machine -- so they are the only
mounts the descriptor-safe teardown cannot take to the graveyard, and they
still come down by a name the kernel resolves again. This form would close
that: the parent is named by a descriptor and cannot be redirected by any
rename above it, and C34 says the one component below it cannot be renamed
while something is mounted on it.

Why it is not written: nobody has run it. The magic link is resolved to
the object the descriptor holds and the rest of the path is walked from
there -- that is the documented behaviour and it is not a measurement. C35
is the reason to be careful about assuming: the obvious form of the same
idea fails, in two opposite ways, and only running it showed that.

It needs a private mount namespace and root, like the probe that produced
C34 to C36: `~/campcheck/umountprobe/main.go` is the shape to copy.

## Person-gated: the session environment against the real world

These need the installed binary for the namespace and a person for
credentials and host-key decisions. They need no sudo. Both are named in
spec §23 as open measurements of this feature.

- **The in-composition OpenSSH run.** In an environment whose `session:`
  section declares `GIT_SSH_COMMAND` and the launcher directory, and whose
  workspace carries the three launchers (docs/install.md, "ssh inside a
  session"), start `camp run -- <shell>` and run against a real peer:

  ```
  ssh <a host from your ~/.ssh/config> hostname
  scp <a small file> <host>:/tmp/
  sftp -b /dev/null <host>
  git ls-remote <an ssh remote>
  ```

  What is being measured: no "Bad owner or permissions" refusal, the
  user's own configuration still applying (host aliases, keys, ports), and
  each of the three entry points reaching the real program through its
  launcher. Then, in the **same terminal after the session ends**, run
  `which ssh` and one `ssh` — command resolution and behaviour outside
  must be exactly what they were before. Nothing camp did may have touched
  the host.

  **Half of this is now measured** (2026-08-18). The launchers are in the
  workspace repository and the inventory is accepted, so the arrangement
  exists: inside a session `ssh` and `scp` resolve to
  `<live>/.workspace/bin/`, raw ssh reached through `$OUTER_PATH` fails
  with *Bad owner or permissions on
  /etc/ssh/ssh_config.d/20-systemd-ssh-proxy.conf*, and the launcher's ssh
  answers `ssh -G <host>` with status 0 — which is the configuration
  parsing that used to refuse. Outside, `command -v ssh` is `/usr/bin/ssh`
  and `ssh -G` is clean. What is left needs a real peer: one connection
  through each of the three entry points, and `git ls-remote` over ssh.

- **The keyring over the namespace boundary.** Whether `libsecret` /
  `gnome-keyring` is reachable from inside a session — the socket lives in
  the user's own runtime directory, so it probably is, but nobody has run
  it. A `git push` to an https remote whose credentials come from the
  keyring is the shortest test. This one matters because a push that
  cannot reach the keyring is the same user-visible failure as the ssh
  refusal that started this work, and no configuration key fixes it.

## Manual, once

- **`tmux attach` from an outside terminal.** `camp run -- tmux
  new-session -d -s work`, then `tmux attach -t work` from another
  terminal, and check that the tree is the composed one. `tmux ls` and
  `send-keys` from outside are already measured; `attach` rests on the
  same client-as-pipe mechanism and has not been run.
- **Ctrl-C reaches the workload and not the init.** `camp shell`, start
  something long-running, press Ctrl-C: the command stops and the shell
  stays. This needs a controlling terminal, which a test process does not
  have. The mechanism is in `session.startWorkload` (the workload gets
  its own process group and becomes the foreground group) and
  `session.supervise` (the init ignores `SIGINT`, `SIGQUIT`, `SIGTTIN`,
  `SIGTTOU`).
- **The single-instance GUI editor handoff** (spec §23). Start the editor
  from inside a session and see whether it opens the composed path or
  hands it to an instance outside, which would open the raw directory.
- **The identity spike at one nesting level**, after the install — the
  same test as above, without the vehicle.
