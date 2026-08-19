# camp — design and measurements

This repository is **not** part of the tool. It is where camp's design
lives, and the record of what was measured while building it — the
working material that would otherwise have to sit in the product's
history, or nowhere.

It is mounted read-write into the composed tree at `.notes`, so it can be
read and edited from inside a session. The workspace repository is
read-only in there on purpose; this one is not, because the design is
written *while* the work happens.

## The three kinds of thing in here

```
reference/     what is true now — look it up, no dates
├── spec.md               the design the implementation was built from
├── constraints.md        what the kernel, git and the instruments do (C1–C36)
└── remaining-checks.md   what still needs an install, sudo, or a person

log/           what happened, in order — dated, never edited afterwards
├── 2026-08-16-build-measurements.md
├── 2026-08-16-handoff-ssh-inside-a-session.md
├── 2026-08-16-session-handoff-state-of-the-work.md
├── 2026-08-16-design-session-environment.md
└── 2026-08-16-review-and-final-plan-session-environment.md

drivers/       the instruments — programs, run by a person at a terminal
├── killmatrix/           kill the helper at each boundary, recover from the record
└── renamerace/           swap the environment's name under root at each resolution
```

The split between the first two is the ordering rule, and it is worth
keeping. A reference document answers "what is true"; a date in its name
would say *stale* the moment it is not. A log entry answers "what did we
know when", and the date belongs in the name. What neither can carry is
which document followed from which — a filename encodes one dimension and
that is a graph, so it lives here in this index instead.

The third is code rather than prose, and it is here rather than in the
tool because it measures the tool from outside: the drivers read the
kernel's mount table and the trees on disk themselves and share no line
with camp, because a driver using camp's own parsing would agree with
camp by construction. `drivers/README.md` says how to build the camp they
need and how to run them.

## What each is

**`reference/spec.md`** — the single source the implementation was built
from. Where it and the code disagree, **the code is the truth**: the code
was measured, the document was written. It superseded two earlier design
documents and answered two reviews; all four were working papers, their
findings are written into it, and they have been deleted rather than left
for somebody to read the wrong one.

**`reference/constraints.md`** — what was *found* rather than decided:
what OverlayFS does and refuses to do, what a bind mount requires, what a
user namespace costs, what git cannot represent, and which instruments
answer a different question than the one asked. An argument that
contradicts something in here is wrong rather than merely different.

**`reference/remaining-checks.md`** — the checks that cannot run
unattended: the ones needing sudo on a real terminal, and the handful
needing a person at a keyboard.

**`log/2026-08-16-build-measurements.md`** — what the build itself
measured, including the two places where it corrected the design.

**`log/2026-08-16-handoff-ssh-inside-a-session.md`** — a design brief,
written to be handed to a session that was not there. ssh does not work
inside a composition, for a reason no mapping can fix; the file carries
the measurements, every candidate solution with its cost, the class of
problems this one belongs to, and what is still undecided.

**`log/2026-08-16-session-handoff-state-of-the-work.md`** — where the work
stands: what is done, what is open, and the facts about this environment a
new session would otherwise rediscover.

**`log/2026-08-16-design-session-environment.md`** — the design that
answers the ssh handoff: one top-level `environment:` key, applied to a
namespace session's workload, with the candidates weighed and the
alternatives closed.

**`log/2026-08-16-review-and-final-plan-session-environment.md`** — that
design reviewed against the code and adopted with amendments, then
reshaped in a second round with the owner: the keys live under a
`session:` section, the privileged mode announces the section instead of
refusing it, and the commands narrate their steps. The normative result
is in `spec.md` (§4 C35, §6, §14, §16, §17, §19, §23); this file is the
review record and the build order.

## Where this stands

The tool is built and working. Ten commits, a fresh history with nothing
internal in it, tests green, and camp composes its own development
environment — which is what this repository is part of.

**The design document has not been brought up to date with what was
built.** It still reads as a plan: build-order stages that are finished,
deferred items that are done, decisions recorded with the date they were
taken, and a section about migrating one specific pair of repositories
that has nothing to do with camp. None of that is wrong exactly, but all
of it would mislead somebody — a person or a model — reading it as a
description of the tool that exists.

That is the next job, and it is the one that matters most for this
repository: **design and proven measurements, nothing else.** Specifically:

- rewrite `spec.md` as a description of what camp *is*, in the present
  tense, with the reasoning kept and the process removed — no build
  order, no dated decisions, no migration of anything;
- fold the corrections the build measured into it, so it stops disagreeing
  with the code (the capability set an overlay mount needs, the locked
  flags, the shape of the namespace restriction);
- keep every "why", because that is the half a reader cannot recover from
  the code;
- check `constraints.md` the same way — it is mostly measurement and
  should survive nearly intact.
