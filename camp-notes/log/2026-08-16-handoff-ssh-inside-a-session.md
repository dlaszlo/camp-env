# Design and plan session handoff — ssh inside a camp session

**Status: a problem statement with measurements and candidate solutions.
Nothing here is decided except what the "The direction the owner chose"
section says, and even that needs a design before it is code.**

**Who this is for.** It goes to a design session first and a planning
session after it. The design session's job is to settle the open decisions
at the end and produce a specification change; the planning session's job
is to turn that into an implementation order with acceptance checks. A
third session writes the code from the plan. None of them was present when
this was found, so this file carries everything: the symptom, the
mechanism, how far it reaches, every candidate weighed with its cost, and
what still has to be decided.

Found on 2026-08-16 by pushing camp's own repository from inside camp's
own composition.

## The brief

What follows is as far as the owner and one session got. It is **not a
decision to implement**; it is the state of the thinking, with everything
that was measured and every candidate that was weighed. The design session
is asked to look for something better, and it is asked to hold to four
things while looking:

  1. **The least pain for the user.** Ideally it works out of the box.
     Failing that, the smallest possible thing a person has to do once,
     visible, and in a place they already own.
  2. **No meddling with the host system.** camp must not install itself
     into a user's `~/.local/bin`, their shell startup files, their global
     git configuration, or anything else outside a session. A composition
     is something you are *inside*; wiring the outside creates
     incompatibilities elsewhere, and other people will not want a tool
     that does that.
  3. **A complete answer inside itself.** Whatever camp provides should
     solve the problem within the composition — not hand the user a
     workaround to carry by hand, and not solve one program's case while
     leaving the class open.
  4. **No principle bent to get there.** The invariants, the write door,
     the refusal-rather-than-guess rule and the target rules are not
     negotiable for the sake of convenience. If a candidate needs one of
     them relaxed, that is the headline of the proposal, not a footnote.
  5. **Either invisible or understandable — nothing in between.** This is
     the one the owner stressed hardest. Users should have nothing to do
     with this at all; and where they do have something to do, they must
     be able to understand what it is and why, in the words of the
     problem rather than the words of the mechanism. A solution that
     works but that a user cannot explain to themselves is not a
     solution — they will not adopt it, and when it breaks they will have
     no idea where to look. "Three wrapper scripts" failed exactly this
     test, and so does anything that asks somebody to know what an
     overflow uid is before they can push.

  6. **General, not tied to this machine.** Everything below was measured
     on one Ubuntu box, but the problem is not Ubuntu's: every
     distribution ships a root-owned `/etc/ssh/ssh_config`, and every
     unprivileged user namespace shows it as `nobody`. The AppArmor
     restriction camp already documents *is* Ubuntu-specific; this is
     not, and a solution that hard-codes `/etc/ssh`, assumes an
     `ssh_config.d` exists, or assumes a system-wide file exists at all
     would be wrong somewhere else. Nor should the answer assume Linux
     distribution layout beyond what camp already assumes.

If the design session finds an answer that beats candidate 5 below on
those six, take it. Candidate 5 is where this session stopped, not where
the search has to end.

Everything marked *(measured)* was run on the owner's machine, inside a
real `camp run` session, on 2026-08-16. Everything marked *(reasoned)* is
a consequence that was not itself run. Where the two disagree, the
measurement wins — that is this repository's rule and it applies to this
file too.

## The symptom

Inside a session, every ssh fails before it opens a connection *(measured)*:

    $ ssh nas
    Bad owner or permissions on /etc/ssh/ssh_config.d/20-systemd-ssh-proxy.conf
    $ echo $?
    255

The same for `ssh librechat`, for `scp`, and for `git push` over an ssh
remote. It is not the connection, not the key, not the host: ssh stops
while reading its configuration and never gets that far.

Outside the session, on the same machine, in the same second, all of them
work.

## Why it happens

ssh reads two configurations: the user's (`~/.ssh/config`) and the
system-wide one (`/etc/ssh/ssh_config`, plus everything `Include`d from
`/etc/ssh/ssh_config.d/`). Before reading the system-wide one it checks
who owns the file, and accepts only root or the invoking user. This is
ssh's own long-standing rule: do not take configuration from a file
somebody else could have written.

A camp session maps exactly one user id *(measured)*:

    $ cat /proc/self/uid_map
          1000       1000          1

Every other id on the host is unmapped, and the kernel shows an unmapped
owner as the overflow id, 65534 *(measured: `/proc/sys/kernel/overflowuid`
is 65534)*. So inside the session *(measured)*:

    $ stat -c '%U:%G %a %n' /etc/ssh/ssh_config
    nobody:nogroup 644 /etc/ssh/ssh_config

The file is unchanged. What changed is who ssh thinks owns it — neither
root nor the user — so ssh refuses it and exits.

The user's own half is untouched and correct *(measured)*: `~/.ssh` is
`dlaszlo:dlaszlo 700`, `~/.ssh/config` is theirs, the private key is
`0600`. camp does not touch the home directory at all. **Mounting `~/.ssh`
somewhere would change nothing**; it is already right, and it is not the
file ssh refuses.

## Why no mapping can fix it

An unprivileged process may map only the ids it owns: its own, and any
range assigned to it in `/etc/subuid`. The host's uid 0 is not one of
them, and never will be — that is the property the whole rootless mode
rests on *(reasoned, from the kernel's user-namespace rules)*.

This holds for the specification's route B as well (`newuidmap` /
`newgidmap` with subuid ranges): those map *your* range, not root's.

Idmapped mounts (`mount_setattr` with `MOUNT_ATTR_IDMAP`) do not help
either: the mapping they apply must consist of ids already mapped in the
caller's user namespace *(reasoned; not run)*. Host root is not among
them.

**Conclusion: inside any unprivileged user namespace, host root-owned
files are `nobody`. camp cannot change that, and a design that promises to
is wrong.** Rootless podman has the same property; it is less visible
there only because a container brings its own `/etc`.

The class is wider than ssh: *any* program that validates the ownership of
a system-wide configuration file will refuse it inside a session. ssh is
simply the one a developer meets on the first day.

## How far it reaches — measured

| path | inside a session |
|---|---|
| `ssh librechat`, `ssh nas` typed by the user | fails, exit 255 |
| `git push` / `git ls-remote` over an ssh remote | fails |
| `scp`, `sftp` | fails |
| `ssh -F ~/.ssh/config librechat` | **works** — connected and ran `hostname` |
| everything over https | unaffected |
| `camp up` (no namespace, no mapping) | unaffected |

`-F` is what makes the difference, and for a reason worth writing down:
per ssh(1), giving a configuration file on the command line **also makes
ssh ignore the system-wide file**. The refused file is not read at all, so
there is nothing left to refuse. The user's aliases, keys, ports and
options all keep working, because they are in the file `-F` names.

Two further measurements decide which repairs can work at all:

**git resolves `ssh` through `PATH`** *(measured)*. A wrapper named `ssh`
placed first on `PATH` was used by `git ls-remote`, with no git
configuration of any kind:

    wrapper used: -o SendEnv=GIT_PROTOCOL git@github.com git-upload-pack 'dlaszlo/camp.git'

**`scp` does not** *(measured)*. The same wrapper was bypassed entirely and
`scp` failed with the original error; `scp` starts ssh from a compiled-in
absolute path. Given its own `-F` wrapper, `scp` succeeded. `sftp` is the
same shape.

**A program started by `camp run` reads no shell startup file**
*(measured)*. Inside `camp run -- claude`, the shell the agent runs
commands in reports flags `hmtBc` — not interactive, not a login shell —
`BASH_ENV` is empty, and the aliases defined in the user's `~/.bashrc`
(`ll`, `la`) are not defined. So **an alias cannot reach an agent's
commands**, and any repair that lives in a shell startup file covers the
user's own terminals and nothing else.

**The ssh client has no environment variable for its own options**
*(measured, from ssh(1))*. Its ENVIRONMENT section lists only variables
ssh *sets* — `DISPLAY`, `HOME`, `LOGNAME`, `MAIL`, `PATH`, `SSH_ASKPASS`
and so on. There is no equivalent of git's `GIT_SSH_COMMAND`. **An
environment variable can therefore reach git, but cannot change how a
plainly typed `ssh` reads its configuration.**

## What was tried, and why each was rejected

Each of these works; each was rejected for a reason that is part of the
design rather than a matter of taste. They are recorded so they are not
rewalked.

**`ssh -F ~/.ssh/config` typed each time.** Correct and immediate, but it
is not a solution — it is the user carrying the tool's limitation by hand,
every time, forever.

**`git config --global core.sshCommand 'ssh -F ~/.ssh/config'`.** Covers
git everywhere including inside a session, and needs no shell. Rejected
because it writes into the *host's* configuration to fix something that is
only true inside a session, and because it silently overrides a setting
the user may already be using for their own purpose. It also covers git
alone: `ssh nas` and `scp` stay broken.

**An alias in `~/.bashrc`.** Rejected on measurement: it never reaches a
program `camp run` starts, which is precisely the case that matters —
an agent's `git push`.

**Wrapper scripts named `ssh`, `scp`, `sftp` in `~/.local/bin`,
conditional on `$CAMP_SESSION`.** Measured to cover all three paths. The
owner rejected it, and the reasons are the design's own:

  - it wires camp into the host environment, which is the opposite of what
    camp is for — a user's `~/.local/bin` is theirs, and a tool that
    installs itself there creates incompatibilities somewhere else;
  - three files is not something users will do, so it is not a solution
    "out of the box";
  - it turns `CAMP_SESSION` from a convention into a contract. Somebody
    else may use that name for something else, and then the wrapper
    changes behaviour for reasons nobody can see.

The last objection reaches further than the wrapper: **camp exports
`CAMP_SESSION` and `CAMP_LIVE` into the workload today**
(`internal/session/session.go`), and anything built on those names inherits
the same fragility. Worth settling deliberately rather than by habit.

**A mount that presents `/etc/ssh` inside the session as a user-owned
copy.** This would work — ownership follows the copy, and the copy is made
by the invoking user — and in the namespace mode it is invisible outside,
so it fits camp's model rather than wiring the host. It was deferred, not
dismissed, for three reasons:

  - it requires relaxing the rule that **every mount target lies inside
    the composed tree**. That rule is what makes the whole plan walkable
    on paper before anything is mounted; relaxing it turns camp into a
    general-purpose mount tool and changes the shape of the safety
    argument. It would be the largest change to the language since
    `steps:`;
  - in the **privileged mode a mount over `/etc/ssh` would be
    machine-wide**, changing ssh's configuration for every process on the
    machine. Invariant 7 allows exactly two machine-wide effects and this
    cannot be a third — so the kind would have to be namespace-only, and
    refused with a reason in `camp up`;
  - a copy can diverge from the real file, which is the "looks applied,
    is actually something else" shape the whole design exists to prevent.
    Bounded by the session, but present. And `/etc/ssh` as a whole cannot
    simply be copied: the host keys are `0600` and root's, so a copying
    step would have to refuse or silently skip them, and both are wrong.
    The client reads only `ssh_config` and `ssh_config.d/`, so the entry
    would have to name narrow paths. One of them is a **symlink** into
    `/usr/lib/systemd` on this machine, and camp follows no symlink
    anywhere (rule 6) — so such a step would have to carry content rather
    than links, or refuse.

## The class this belongs to — think ahead, do not fix one instance

The owner's instruction: ssh is the first one we walked into, so work out
what else has the same shape and **design for the class, not for ssh.** A
solution that fixes ssh and leaves the rest is a solution that will be
rewritten.

A session changes exactly three things a program can disagree with the
outside about. Everything below follows from one of them.

  - **Who owns a file.** One id is mapped; every other owner on the host
    reads back as the overflow id.
  - **Which pid is which.** The session is a pid namespace with its own
    `/proc`.
  - **What is mounted.** The composed tree exists inside and nowhere else.

### Owner-of-a-file, the class ssh belongs to

Everything here is *(reasoned, unmeasured)* unless marked otherwise. The
design session should measure the ones its candidate would not cover.

  - **ssh, scp, sftp** — the measured instance. Refuse a system-wide
    configuration file they cannot attribute to root or to the user.
  - **setuid binaries in general.** In a user namespace the setuid bit on
    a binary owned by an unmapped id confers nothing. `sudo` cannot
    elevate inside a session at all; so cannot `pkexec`, `newuidmap`, or
    anything else that relies on it. This is not a bug to fix — it is the
    mode being rootless — but it is the same root cause and users will
    meet it, so it belongs in the same explanation.
  - **git's own ownership check.** `safe.directory` refuses a repository
    whose directory belongs to another user ("detected dubious
    ownership"). The composition's repositories are the user's, so this
    does not bite today — but a *system-wide* checkout, or any repository
    owned by root, would become "dubious" only inside the session.
  - **Anything that records ownership into an artefact.** `tar`, `rsync
    -a`, a container build context, a package build: a host file that is
    root's outside is written into the artefact as `65534` from inside.
    This is the dangerous member of the class, because **nothing fails** —
    the artefact is simply wrong, and the failure surfaces somewhere else,
    much later. It is the "looks applied, is actually something else"
    shape the whole design exists to prevent, arriving through a door
    camp did not build.
  - **`chown` to any other user** fails, because no other id exists to
    chown to. Build systems that install files as root inside a prefix
    will fail or silently produce user-owned files.
  - **Tools that check the owner of a socket or a runtime directory** —
    as opposed to relying on its permission bits. Access itself is
    unaffected: measured earlier in this project's build log, a
    `root:docker 0660` socket is usable from inside because the kernel
    keeps the supplementary groups. Only a *name-based* check would be
    fooled.
  - **Credential helpers and keyrings over D-Bus** (`libsecret`,
    `gnome-keyring`) — the socket lives in the user's own runtime
    directory, so this probably works, but nobody has run it. Worth
    measuring, because a `git push` that cannot reach the keyring is the
    same user-visible failure as the one that started all this.

### Which-pid-is-which

  - Anything that hands a pid across the boundary: `systemctl`, a service
    manager, a debugger attaching from outside, a profiler, a supervisor.
    Inside sees namespace pids; outside sees different numbers for the
    same processes.
  - This is already partly known: the project's own lock-holder reporting
    had to stop trusting `/proc/locks`' recorded pid for exactly this
    family of reasons (commit `c199abf`).

### What-is-mounted

  - Anything started outside that must see the composed tree — the case
    `camp up` exists for, already designed and documented.
  - A single-instance GUI editor launched from inside that hands the path
    to its outside instance — already in the specification's deferred
    register as an unmeasured trap.

### What this implies for the design

Two things, and they are the reason this section exists.

**Prefer a mechanism that is not about ssh, or about any named program.**
Candidate 5 qualifies: it sets whatever the configuration says, so the
next member of the class is handled without changing camp. Candidate 3
would also qualify if its unit is "a host path presented differently
inside", rather than "the ssh configuration".

**Prefer a mechanism that makes the silent members loud.** The artefact
case above is the one that matters most and the one no configuration key
fixes: nothing refuses, nothing warns, the tar file is just wrong. If a
candidate cannot fix it — and none of the five can — then camp should at
least be able to *say* it, in `explain`, in one sentence, where somebody
building an artefact inside a session might read it.

## The five candidates the owner put on the table

Late in the session the owner listed five things and said "or all three,
that is, all five". They are not alternatives to each other — one, two and
five can coexist — so each is written out with what it costs and what it
leaves unsolved. The design session should decide which of them ship, in
what order, and which are dropped.

### Candidate 1 — camp documents the problem thoroughly

Everything a user could need: the mechanism, why no mapping fixes it, what
breaks and what does not, and what to do about it. Partly done already
(see "What is in the tool today").

**Cost:** none beyond writing. **What it leaves unsolved:** everything.
The owner's own verdict: *"with 1 and 2 you do not actually solve the
problem, you live with it."* Necessary, not sufficient.

### Candidate 2 — document the `camp up` route as the way out

`camp up` builds no namespace, so no id is remapped and ssh behaves as it
does anywhere else. It is the only answer that is genuinely out of the box
today and asks nothing of the host.

**Cost:** the mode's own price — the workspace becomes read-only for every
process on the machine until `camp down`. A large hammer for an ssh
connection. **What it leaves unsolved:** the namespace mode, which is the
primary one.

### Candidate 3 — a general mount kind that presents a host path as a user-owned copy

Weighed in full above under "What was tried". The important correction the
design session must not lose: **read-only is not the property that
matters.** A read-only bind does not change ownership — the file belongs
to whoever owns the inode, so it would still be `nobody` and ssh would
still refuse it. The mechanism has to be a *copy* made by the invoking
user (in camp's storage, or a tmpfs inside the namespace), bound over the
host path inside the session.

**Cost:** it relaxes the rule that every mount target lies inside the
composed tree, which is what makes the plan walkable on paper before
anything is mounted. Namespace-only by necessity (see candidate 4). Copies
can diverge. `/etc/ssh` cannot be copied wholesale (root-only host keys),
and one of the files that matters is a symlink into `/usr/lib/systemd`,
which camp follows nowhere.

**Verdict reached in the session:** deferred, not dismissed. Candidate 5
covers the real cases at a fraction of the cost. Revisit only if a case
appears that 5 cannot reach.

### Candidate 4 — mounts that exist in one mode only

Raised by the owner as a question: does every mount belong in `camp up`?
**No, and the case is sharper than it looks.** A mount over `/etc/ssh` in
the privileged mode would be **machine-wide** — it would change ssh's
configuration for every process on the machine. Invariant 7 permits
exactly two machine-wide effects and this cannot become a third.

So any mechanism that reaches outside the composed tree must be
namespace-only, and `camp up` must **refuse** it with the reason rather
than accept it and quietly do something different.

Two ways to express it, and the design session should pick one:

  - **the kind itself is namespace-only**, and the privileged mode refuses
    a configuration that uses it. One rule, one message, no new grammar;
  - **steps gain a general qualifier** (`when: namespace`). More
    expressive, more machinery, and no second use case exists today.

The project's own rule — no abstraction without a second caller that
exists today — points at the first.

This same question applies to candidate 5: a declared environment cannot
apply in `camp up` either, because there is no workload there to apply it
to. The answer is the same: refuse, with the reason.

### Candidate 5 — the session's environment comes from the configuration

The owner's own correction, and the one that reshaped the answer: camp
should not set a *fixed* variable, it should set **what the configuration
says**. Not `CAMP_SESSION` turned into a contract, not `GIT_SSH_COMMAND`
silently overridden by camp — a map in the configuration file, applied to
the session's processes.

The owner also raised a shipped step that configures git. It is written up
under "The direction" below, with the reason it is not the first thing to
build.

## The direction the owner chose

**One new key in the configuration: an `environment:` map, applied to the
session's own processes and to nothing else.**

```yaml
environment:
  GIT_SSH_COMMAND: "ssh -F ~/.ssh/config"
  PATH: "$CAMP_LIVE/.workspace/bin:$PATH"
```

Why this shape:

  - **It is configuration valid inside the sandbox only.** Nothing is
    written to the host, nothing survives the session: the variables come
    into being with the namespace and vanish with it.
  - **camp knows nothing about ssh or git.** It sets what the
    configuration says. The next program that breaks the same way is
    fixed by the same key, with no change to camp.
  - **It is the user's decision, recorded and diffable** — the same
    principle as `allow_overlap` being the escape hatch instead of
    `--force`.
  - **It covers both paths, despite ssh having no options variable**,
    because `PATH` is an environment variable. The wrapper does not go on
    the host: it goes into the **workspace repository**, which is what
    carries the development environment already, is versioned, and is
    part of the composition. Prepending `PATH` reaches it.

What it does not do, stated plainly: `camp up` has no workload — camp
mounts, and the user then enters the tree from a shell that never passed
through camp. Declared environment therefore cannot apply there, and camp
must **refuse the key in the privileged mode with the reason** rather than
accept it and do nothing. A setting that looks applied and is not is the
failure shape this project refuses everywhere else.

A separate shipped step that configures git was considered and left for
later: one line in `environment:` is smaller than a new step kind, and the
rule is no abstraction without a second caller that exists today. If a
second tool turns out to need the same treatment, a step earns its place —
and it belongs among the steps, beside `git_exclude`, never in the core.
Note also that a git-configuring step has nowhere clean to write: not the
repository (invariant 1), not the user's home (the host), which leaves the
environment again — so the step would be a convenience wrapper over
candidate 5, not an alternative to it.

**The honest gap in candidate 5, which the design session must weigh.**
The wrapper it relies on for interactive `ssh` has to exist somewhere.
Putting it in the workspace repository keeps it out of the host and under
version control, which is right — but it is still a file somebody has to
write, and camp cannot ship it without carrying ssh knowledge. So
candidate 5 is out of the box for **git** and one-line-of-configuration
for **interactive ssh**. Whether that is good enough is exactly the
question the brief above asks.

### What building candidate 5 would involve

A sketch, not a specification — the design session owns the final shape.

**Grammar.** A top-level `environment:` mapping from name to value. Names
are what the kernel allows in an environment variable and must not contain
`=` or NUL. Values are strings. Everything else about the path language
(§6) is unaffected because these are not paths.

**Interpolation.** `$NAME` in a value means the value `NAME` had in the
environment camp itself was started with, and nothing else — no shell, no
command substitution, no defaulting, no nesting. This exists because
prepending to `PATH` is the whole point and cannot be written without it.
`$$` for a literal dollar. An undefined `$NAME` is an open decision below.

**Where it is applied.** To the workload the session's init starts, and
therefore to everything descended from it. Not to the generation step —
that already has its own contract (`CAMP_GEN_*`) and runs before anything
exists. Not to camp's own processes.

**Order against camp's own variables.** camp exports `CAMP_SESSION` and
`CAMP_LIVE` today. Decide explicitly whether a declared variable may
override them (probably not — a configuration that redefines
`CAMP_LIVE` is describing a tree that is not there) and refuse rather than
silently pick a winner.

**Refusals.** `environment:` present while running `camp up` — refuse,
naming the mode and why there is no process to apply it to. A name that is
not a legal variable name — refuse. `$NAME` that does not close or is
malformed — refuse. Consistent with every other refusal: name the thing,
say what is true, give the way out.

**`camp plan` must print it.** The plan is the description of what will
happen; a variable that changes how every program in the session behaves
belongs in it, with its value after interpolation.

**`camp explain` must print it**, because it describes the tree to
whoever is standing in it, and "why does ssh work in here" is exactly the
kind of question it answers.

**Tests.** Listed under "What must be measured" below.

## Open decisions

**Value interpolation.** Prepending to `PATH` requires referring to the
value inherited from outside. The smallest rule that serves it: `$NAME`
means the inherited value of `NAME`, and nothing else — no shell, no
command substitution, no conditionals. Is more ever needed? What should
`$NAME` mean when `NAME` is not set outside — empty, or a refusal?

**Whether `camp init`'s skeleton carries the ssh lines**, commented, with
the reason. If it does, a reader deletes one character and it works, and
nothing happens behind anyone's back. The core still knows nothing about
ssh; a skeleton is documentation. If it does not, every user meets the
problem once and must find this document.

**Whether `CAMP_SESSION` and `CAMP_LIVE` stay as they are.** They are
camp's own names and exporting them is reasonable, but the objection above
applies: anything a user builds on them is built on a convention. Decide
whether they are a promise or an accident.

## What must be measured before any of it is trusted

  - that a declared variable actually reaches the workload, and reaches a
    program `camp run` starts directly as well as an interactive shell;
  - that `PATH` prepending reaches a wrapper living in the workspace
    repository, through the composed tree's own path;
  - that `camp up` refuses the key, with the message, rather than
    accepting it;
  - that nothing declared leaks outside the session — the same variable
    unset in the parent shell after the session ends.

## What is in the tool today

`camp explain` already carries a section for the namespace mode saying
that host files owned by anyone else appear as `nobody`, why no mapping
fixes it, and that ssh is where it is met. The README, `docs/install.md`
and `docs/how-it-works.md` carry the same fact.

**They currently recommend the host-side repair** — `core.sshCommand` plus
an alias — which the owner has since rejected. Those passages have to be
replaced by whatever this design settles on. Commit `566782f`.
