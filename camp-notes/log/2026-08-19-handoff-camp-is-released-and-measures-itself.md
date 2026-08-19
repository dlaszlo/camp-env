# Handoff: camp is released, and it measures itself

Date: 2026-08-19, end of the day. Written for whoever picks this up next,
including a later session of the same work. It replaces
`log/2026-08-19-handoff-the-two-measurements-that-are-left.md`, whose two
measurements are both made.

## Where things stand

`camp` is on `main`. The branch `fix/review-8ddf464` was merged into it by
fast-forward, so `main` carries the twelve findings of the read-only
review, the three an independent review of those repairs found, the
descriptor-safe teardown, and everything below. Both repositories are
pushed and the environment's submodule pointer is on the current camp.

**Three releases went out today** — `v0.1.0`, `v0.1.1`, `v0.1.2` — each a
`.deb` attached to a GitHub release, built by a workflow from a tagged
commit after the suite passed. `camp --version` and `dpkg -l camp` cannot
disagree: the build puts one version string in both.

**Nothing needs a person any more except one measurement.** Two ways to
run everything:

- `camp/measure/vm/run` boots a machine that exists for the length of one
  run and runs every stage in `measure/vm/guest` on it. Plain user-space
  qemu with KVM, no libvirt and no root on the host.
- `.github/workflows/ci.yml` runs the same stage files on a hosted runner
  at every push. A runner *is* a throwaway machine with passwordless
  sudo, which is what those measurements always needed.

Adding a measurement is adding a file to `measure/vm/guest`. They run in
name order and each is a program that exits non-zero when what it
measures does not hold. Both the virtual machine and the runner discover
them the same way; neither knows what the other runs.

## What was measured, and what it found

Everything the review named is measured rather than argued:

- the **kill matrix** holds at all twelve boundaries (the review's eight,
  with `mount-made` expanded to one case per nested mount);
- the **rename race** holds at all four resolutions;
- the **suite** is green in both builds, on ext4 and on tmpfs, from a
  checkout and from inside a composition;
- **ssh from inside a composition** reached a real peer through all three
  entry points, and nothing it arranged survived the session —
  `log/2026-08-19-ssh-from-inside-a-composition.md`.

Six defects were found by running these, and every one is repaired. They
are worth reading as a group, because none of them was visible from the
code:

1. The teardown held the descriptor it had decided on across the unmount,
   so every mount came back busy — C35, quoted in a comment three lines
   above the code that violated it.
2. Root unmounted a mount in the trap tree, because a self-bind was
   removed by the name the record carried. Fixed with `UnmountIn`: the
   parent directory's own `/proc/self/fd` name plus one component.
3. A teardown whose environment had been renamed reported every remaining
   target absent, exited 0, and let the record go.
4. camp refused to run for a user who had never had `~/.local`.
5. The identity spike wrote its own read-only remount without the locked
   flags, so on a `nosuid` filesystem the island stayed writable and the
   test blamed camp.
6. The capability drop and the fork happened on whichever thread Go
   chose, so the workload could inherit the mount capability. `Inside`
   locks its thread now and reads the drop back.

Items 4, 5 and 6 could only be found on a machine nobody had used, on a
filesystem the suite had never run on, and by running the same code twice.
That is what the throwaway machine bought.

## What is left

**One measurement that needs a person**: the keyring across the namespace
boundary. A `git push` to an https remote whose credentials come from
libsecret is the shortest test. It matters for the same reason ssh did —
a push that cannot reach the keyring is the same user-visible failure —
and no configuration key repairs it.

**A few manual conveniences**, all at the end of
`reference/remaining-checks.md`: `tmux attach` from an outside terminal,
Ctrl-C reaching the workload, the single-instance GUI editor handoff.
None is an invariant.

**And the job this repository has always named as its most important**:
`reference/spec.md` still reads as a plan. Build-order stages that are
finished, decisions recorded with the date they were taken, and a
migration of one specific pair of repositories that has nothing to do
with camp. The parts that had gone *wrong* are corrected — what a
teardown does, and §23's register of measurements — but the restructuring
is not done. It should become a description of what camp *is*, in the
present tense, with every "why" kept, because that is the half a reader
cannot recover from the code. `constraints.md` needs none of this: it is
measurement, and it has grown to C37.

## Facts about this environment a new session would otherwise rediscover

- **camp is installed from its own package**, at `/usr/bin/camp`. Do not
  put a hand-built binary in `/usr/local/bin`: it wins the PATH and the
  AppArmor profile names one path.
- **Building in the repository root leaves a `camp` binary there**, which
  the accepted snapshot then reports as a new root entry. Build to
  `/tmp`, or use `packaging/deb/build`. Since v0.1.2 camp says which new
  entries git ignores, so this is at least self-explaining.
- **The workspace repository's main checkout is not edited or run in.**
  That is the rule `~/dev/camp_migration/PLAN.md` establishes for
  diet-coach and the one this environment follows: the workspace is the
  composition's reading side.
- **The prompt marker** comes from `session.environment` in this
  environment's own configuration, not from any file on the machine. camp
  changes nothing outside a session, and that includes the mark that says
  a session is running.
- **The scratch composition** at `~/campcheck` is what the drivers use on
  this machine. In the virtual machine and on a runner one is made from
  scratch instead, so nothing is carried in with a history nobody looked
  at.

## The one rule this day would want kept

Every number in these notes came from running something. Where a claim
was made without running it — an image two releases back refusing a mount
— it was withdrawn the moment that was noticed, in the same file that
made it. "Measured" is worth nothing if it means "argued".
