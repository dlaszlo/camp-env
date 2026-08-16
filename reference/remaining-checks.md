# The checks that need an install, and the ones that need a terminal

Everything in this file is written, runs, and is waiting. None of it can
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

```
cd ~/overlayfs/ply
go build -o camp ./cmd/camp
sudo install -m 755 camp /usr/local/bin/camp
sudo install -m 644 packaging/apparmor/camp /etc/apparmor.d/camp
sudo apparmor_parser -r /etc/apparmor.d/camp
camp doctor          # the namespace line should read "permitted, and a mount inside one succeeds"
```

Then `go test ./...` runs everything below except the sudo group.

## Install-gated: the namespace

These skip with an explanatory message today and pass once the binary and
its profile are installed. Every one of them has already been **run and
passed** through the installed `/usr/local/bin/ply` used purely as a
vehicle for opening a namespace, which the specification's testing note
allows; what the install changes is the number of nesting levels, not the
mechanism.

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

## Terminal-gated: the privileged mode

sudo needs a terminal, so these are for the owner to run by hand. The
code is written and its pieces are unit-tested where they can be — the
helper refuses a job it does not understand, the teardown instruction is
built from the record alone with the configuration deleted, running the
front end as root is refused — but the end-to-end behaviour has not been
exercised on this machine.

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
