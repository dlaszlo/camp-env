# Drivers — the two measurements that need a person

These are not part of camp. They are the instruments for the two
measurements the code review left open, and they are here rather than in
the tool's repository because they measure it from outside: they read the
kernel's mount table and the trees on disk with their own eyes, and they
share no line of code with camp. A driver that used camp's own parsing
would agree with camp by construction, and agreement is what is being
tested.

Neither can be run unattended. Both need sudo on a real terminal — sudo
asks for a password there and cannot ask anywhere else, which is C33 —
and both make and remove real mounts on the machine they run on.

## One command for all of it

```
./measure
```

Builds both binaries, does one ordinary `camp up` and `camp down` to show
the mount paths work at all, then runs both drivers — stopping at the
first stage that fails, because a stage measuring what the one before it
left measures nothing. Everything it printed is also in `measure.log`, so
somebody who was not at the terminal can read it afterwards.

A single stage on its own: `./measure build`, `./measure run`,
`./measure killmatrix`, `./measure renamerace`. It takes the environment
from `CAMP_ENV` and the camp repository from `CAMP_REPO` if the defaults
(`~/campcheck` and the sibling checkout) are not where they are.

The rest of this file is what the two drivers do and how to run them by
hand.

## Building the camp they drive

Both need a camp with the barrier protocol compiled in. That build is not
one anybody ships: a pause inside the root helper that the invoking user
can trigger is the attack the measurements exist to prove camp is safe
from, so it exists only under a build tag.

```
cd <the camp repository>
go build -tags camptest -o ~/campcheck/camp-barriers ./cmd/camp
```

The helper is the same executable, so no install is needed: run *that*
binary and its `sudo camp helper-mount` is itself.

## `killmatrix` — recovery from the record alone

Interrupts the privileged helper at each of the eight boundaries the
review lists, deletes the configuration, and requires `camp down` to take
the composition apart from its record alone. `mount-made` fires once per
nested mount and each one is measured separately.

```
go run ./killmatrix -env ~/campcheck -camp ~/campcheck/camp-barriers
```

What it requires at every boundary: every mount camp made is gone, no
unrelated mount is gone, the repositories and the storage hash the same
before and after, the record survives exactly as long as something is
still standing, and anything that could not be removed is named.

## `renamerace` — the environment's name swapped underneath root

Renames the environment away at four of the helper's resolutions and
leaves a symbolic link to a root-owned trap tree at its name, then lets
the helper carry on.

```
go run ./renamerace -env ~/campcheck -camp ~/campcheck/camp-barriers
```

The assertion is **not** that camp errors. camp may refuse and camp may
carry on; what it may never do is act at the link's target. So what is
measured is the trap tree and the rest of the machine: no mount id
outside the environment changes, no mount attribute outside it changes,
no inode mode outside it changes, and the trap tree hashes identically
before and after.

## Both of them

- Run them as yourself, not with sudo. camp's front end refuses to run as
  root on purpose, and these drive the front end; camp elevates for the
  one step that needs it.
- Point them at a scratch composition. They take it up and down many
  times, rename its root, and kill things in the middle of it.
- They print a verdict, not a log: one line per case while they run, and
  at the end what failed, with what was seen and what was required.
- Exit 0 means every case held, 1 means something failed, 2 means
  something could not be measured — which is not a pass.

## Reading a failure

Every failure names the object rather than describing it: a mount is
quoted as its whole `/proc/self/mountinfo` line, a tree as the two hashes
that differ, a mode as the path and the two values. That is deliberate.
These are run when somebody wants to know whether camp survives a thing,
and a verdict that only said "failed" would send them back to do the
measurement again by hand.
