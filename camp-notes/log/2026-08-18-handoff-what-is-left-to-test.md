# Handoff: what is left to test, and how to run it

Date: 2026-08-18, the session that finished the output work and closed
the review's code findings. camp at `8ddf464`, the environment at
`c8c675e`, all three repositories clean, nothing mounted, no record.
**The installed binary is this commit** (`/usr/local/bin/camp`,
reinstalled after the work-directory repair), so `camp run` and `camp up`
both exercise today's code.

Both gates are green with nothing skipped:

```
cd ~/dev/camp-env/camp && go build ./... && go vet ./... && gofmt -l . && go test ./...
cd ~/dev/camp-env      && camp run -- go test ./internal/... -count=1
```

`log/2026-08-18-output-finished-and-the-code-review-closed.md` says what
was built and repaired. `reference/remaining-checks.md` is the testing
record and has today's results in it. This file says only what is left.

## The sudoers rule is still installed

```
dlaszlo ALL=(root) NOPASSWD: /usr/local/bin/camp helper-mount, /usr/local/bin/camp helper-unmount
```

It is what lets a session run `camp up` and `camp down` unattended, and
it is the only reason the privileged measurements below can be automated.
It is scoped to the two helper subcommands of the installed path and to
nothing else, and **it is to be removed when this work is done**:
`sudo rm /etc/sudoers.d/camp-testing`. A binary built anywhere else
cannot elevate through it, which is why a scratch build's `up` fails on
the password it cannot ask for.

## First, while the rule is there: the operand swaps

This is the half of CAMP-REVIEW-004 its own repair asked for and today's
session only started: *exercise a swap independently for each operand,
including the live directory between the first verification and the
move.* What is measured so far is a **bind source** swapped in the window
between the front end's identity capture and the helper's mount, in both
forms -- a symlink (refused by the resolution) and a different real
directory (refused by the identity). What is unmeasured: the overlay's
`lower`, `upper` and `work`, and the **live directory**.

The live one is the interesting one. The move now goes to a descriptor
opened beneath the job's base rather than to a name, so a live directory
replaced after the staging verification must either be refused or be
irrelevant -- the move lands on the object that was opened, not on
whatever now answers to the name. Nobody has watched that happen.

The tooling from today is in the session scratchpad and is three small Go
programs, each about twenty lines: one that watches `/proc/self/mountinfo`
in a tight loop and signals a pid the moment a mount appears, and two
that rename a path aside and put something else in its place on the same
trigger. Rewriting them takes minutes; the shape that works is:

- start `camp up` in the background and hold its pid;
- poll for the trigger -- the record appearing in phase `mounting`, or
  the staging self-bind appearing in the mount table, or the overlay
  appearing at the live path;
- do the swap, or send the signal, the instant the trigger fires;
- then check `camp status` describes what is there and `camp down`
  converges: nothing mounted, no record, work directory gone, live empty,
  workspace writable.

Shell polling is fast enough for the record and the staging trigger. It
is **not** fast enough for the move: that window is about two
milliseconds, and it takes a compiled loop.

One window is known to be open and is a design question rather than a
defect: a source swapped *before* the job is built is accepted, because
the front end takes each operand's identity when it builds the job. The
validation looked at the directory that was there earlier. Whether to
take identities at validation and carry them forward is the owner's.

## Then, and no privilege needed: three test gaps

These are Go tests, and each names a repair that was made without the
test the review asked for.

- **CAMP-REVIEW-019, crash boundaries.** The repair made retirement
  order-safe -- object first, record second, and only ENOENT means gone.
  The review asked for tests proving every intermediate state is either
  attributable and recreatable or ordinary user content camp leaves
  alone. Today's test covers the error path (an attachment point camp
  cannot look at stays camp's); the crash boundaries themselves are
  untested. They can be built by hand: write the manifest and not the
  object, the object and not the manifest, and run `gen.Prepare` over
  each.
- **CAMP-REVIEW-011, a repository git cannot read.** The three states are
  measured with no git on PATH. A repository that exists and cannot be
  read -- mode 000, or a damaged `.git` -- takes the same path and is not
  measured.
- **CAMP-REVIEW-022, a failed mark.** The read failure and the `.seen`
  collision are measured; an injected rename failure is not, so
  "delivered exactly once" rests on inspection at that one point.

## Then the rule goes

`sudo rm /etc/sudoers.d/camp-testing`, and `reference/remaining-checks.md`
gets the sentence saying it is gone -- the same shape as on 2026-08-16.

## What stays person-gated, and is nobody's to automate

- **OpenSSH against a real peer.** Half of it is measured: the launchers
  are what `ssh` and `scp` resolve to inside a session, raw ssh through
  `$OUTER_PATH` fails with *Bad owner or permissions*, the launcher's ssh
  answers `ssh -G <host>` cleanly, and outside the session nothing
  changed. What is left needs credentials and host-key decisions: one
  connection through each of `ssh`, `scp` and `sftp`, and `git ls-remote`
  over ssh.
- **The keyring over the namespace boundary**: a `git push` to an https
  remote whose credentials come from the keyring.
- **`tmux attach` from an outside terminal.**
- **Ctrl-C reaches the workload and not the init** -- it needs a
  controlling terminal, which no test process has.
- **The single-instance GUI editor handoff**, and **the identity spike at
  one nesting level**.

## Open, and the owner's

Unchanged by today: specification finding 014, the constraints bookkeeping
017 -- which has grown by the two overlay facts this session measured --
the `.camp` rearrangement, whether the inventory should count a name the
repository gitignores, and the five readings in
`log/2026-08-16-recovery-from-the-record.md`.

Added today, all small and all documents: the specification says nothing
about the log, the colour or `.camp/logs`; §12's sentence that everything
under storage belongs to the invoking user is broader than what camp
checks (the storage root, each store and each island attachment point,
deliberately not the worktrees); and §12 gives the work directory's
removal to `down`, which is now true -- a rolled-back `up` still leaves it
for the next namespace sweep, which is worth a decision only if this
environment ever runs the privileged mode without any session.
