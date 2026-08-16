# The checks that need an install, and the ones that need a terminal

**The namespace group is no longer waiting.** On 2026-08-16 camp was
installed at `/usr/local/bin/camp` with its profile, and the whole suite
was run through it — `camp run -- go test ./internal/... -count=1`, which
opens the namespace with the installed binary and lets the test binary
create its own inside. Every package passed and **nothing skipped**. What
is still waiting is the terminal group and the two person-gated
measurements at the end.

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

Everything below is still unrun. It is the next session's first job.

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
