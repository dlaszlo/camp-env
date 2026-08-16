# Handoff: the terminal group is closed, the output work is half built

Date: 2026-08-16, the session after the one that first ran the privileged
mode. camp from `18f243f` to `35c8dd1`, both repositories clean, the gate
green on the host and through the installed binary with nothing skipped.

## What is now measured

**The terminal-gated group is done**, except the rename race itself.
`reference/remaining-checks.md` says which check found what; the two log
files beside this one carry the detail:

- `2026-08-16-recovery-from-the-record.md` — the kill-point matrix, five
  boundaries, and the five readings of the specification that want a
  review's verdict.
- `2026-08-16-copy-up-forensics.md` — what a copy-up leaves on disk, and
  the fact that the code repository's root keeps `trusted.overlay.uuid`
  and `trusted.overlay.impure` for good.

Running them found seven defects, all repaired: two in the privileged
mode itself, three in the repairs this session made, one in a refusal's
advice, and one in the test suite's own hygiene — a test that reached the
machine's real record and got as far as sudo.

**The testing sudoers rule is gone.** It was
`/etc/sudoers.d/camp-testing`, scoped to the installed binary's two
helper subcommands, and the owner removed it when the group finished. Any
further privileged measurement needs a person at a terminal again, which
is the specification's own position.

**What is still unrun**: the rename race as a race (its decision point is
a unit test now); the in-composition OpenSSH run and the keyring, both
waiting on the three launchers in the workspace repository's
`.workspace/bin/` and a `camp accept`; the single-instance GUI editor
handoff; and the identity spike at one nesting level.

## The output work: what is built

Against the list the previous handoff left:

- **`[WARN]` exists and has its caller.** `plan.Warnings` was computed at
  every up and printed by nobody — only `plan` and `doctor` showed it,
  and neither is what somebody runs before starting work.
- **The ownership fact is a `[NOTE]` of its own**, off the identity line.
  That was the collision the previous handoff named, resolved as it
  proposed.
- **Streams by concept.** stdout is the product a reader pipes — the
  plan, the description, the listings. Everything about the run is on
  stderr, including `plan`'s refusals, `doctor`'s rendered error and
  `up`'s price paragraph, which were all mixed into stdout.
- **`down` narrates per operation**, and the work-directory sweep says
  what it swept and what it left alone, in the same columns as everything
  else.

## The output work: what is not

Three pieces, in the order I would do them.

**Refusal grouping** is the largest and the one the previous handoff
called out: nine mounts failing the same check must be one refusal naming
nine paths, not nine refusals. Nothing of it is built. `refusal.List` and
`report.Refusals` are where it would live.

**The log.** Every line that goes to stderr, to a file under
`.camp/logs/`, rotated by size, a few files kept, RFC 3339 local
timestamps, always on. Not built. No logging framework — the previous
handoff's reasoning stands and is worth re-reading before anybody reaches
for one.

**Colour**, only where it is honestly available: a terminal, `NO_COLOR`
unset, `TERM` not `dumb`; the marker only; `[ERROR]` bold red, `[OK]`
green, `[NOTE]` blue, `[WARN]` and `[HINT]` yellow; never into the log
file, because the colouring is per sink and not baked into the string.
Not built. Terminal width is still never queried, which is already true:
`wrap` folds to a fixed 68 columns.

## The rest of the review findings

Repaired this session: 007 and 008 (the record drives status, down and
explain; identity-checked teardown; strict record reading), and 013 (a
teardown whose last step failed no longer reports success). 005 is now
measured rather than argued — the allow-listed directory is a real
writable hole, and a copy-up through it is what the forensics used.

Still open, in the verification's order: 003, 004, 005, 012, 011, 009,
010, then 015–027 and the deletion 028. Two are documents and the
owner's: 014 and 017 — and 017 has grown, because this session measured
two facts that want numbers in `constraints.md`.

## Open decisions, still the owner's

The `.camp` directory rearrangement (the container's name is still not
chosen). Whether the inventory should count a name the repository itself
gitignores — a build artefact at the code root triggered a drift warning
this session. And the five readings listed in
`2026-08-16-recovery-from-the-record.md`.

## Added after the fold

**The environment is one repository now.** `camp-workspace` and
`camp-notes` were folded into `camp-env` with their history, by
`git subtree`; `camp` is the only submodule, because it is the product
and is published on its own. A clone now brings the whole environment
down. Two things follow, and both are in that repository's README: the
workspace stopped being a repository of its own, so its root lost `.git`
and the inventory was re-accepted; and `git` run in `.notes` from inside
a session walks up to the composed root and finds the **code**
repository, so the design record is committed from outside the
composition.

**camp creates the composed tree's directory.** git records no empty
directory and a placeholder in this one would make camp refuse, so a
clone could never bring it and every fresh checkout met a refusal for the
one directory camp can safely make itself. A session creates it now;
`camp plan` still creates nothing and says it is not there yet. A parent
that does not exist is still refused -- that is a typo in `merged:` --
and so is a `merged:` pointing inside a repository, checked at the
creation site because it runs before the validation.

The specification says nothing either way about who creates that
directory. It may want a sentence, and that is the owner's file.
