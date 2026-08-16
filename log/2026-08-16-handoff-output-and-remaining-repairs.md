# Handoff: where the testing stands, and what comes next

Date: 2026-08-16. Written to close a session, so the next one starts from
fact rather than from reconstruction. camp at `18f243f`, both repositories
clean, gate green on the host and through the installed binary with
nothing skipped.

## What happened in this session, in order

The `session:` configuration section was built in five stages and closed.
Then two whole-project reviews were run against it — one by another
reviewer, one adversarial against that review — and their findings were
verified and partly repaired. `log/2026-08-16-verification-of-the-whole-
project-review.md` is the record; this file only says where that left off.

The privileged mode was then run end to end **for the first time**. It had
never been run before, and that is why it carried four defects nothing
caught. Two more were found by running it, and both are repaired.

## The privileged mode: what running it found

`camp up` now reaches the end. Getting there took six repairs, four from
the reviews and two that only a real run could show.

From the reviews: the read-only remount went through an `O_PATH`
descriptor opened before the bind, so it addressed the object underneath
and failed `EINVAL`; the staging verification rebuilt a plan without
`Config`, so the frame's first mount read as missing on every honest
composition; the helper was a general root primitive; and a mount was
recorded for rollback only after its whole multi-syscall operation
finished, so a half-made mount was left standing under a report of a clean
machine.

From running it: `MS_MOVE` refuses a mount whose parent is shared, and on
a systemd machine `/` is — so the staging tree needs a private parent of
its own. And the destination needs one too, for the other half of the same
rule: attaching under a shared parent marks the moved mounts shared.
Measured three ways; making them private after the move is too late,
because the copies in the peers exist from the moment of attachment and
are not camp's to remove.

A seventh, smaller one: a failed `up` left the kernel's root-owned overlay
work directory behind and forgot its record, so neither `camp down` nor
`camp run` could clear it — `camp run` refused outright until it was
removed by hand. The helper clears it now before reporting a clean
machine.

## Where the testing stands

**Namespace mode**: the whole suite runs through the installed binary with
nothing skipped. `cd ~/dev/camp-env && camp run -- go test ./internal/...
-count=1`.

**Privileged mode**: `camp up` and `camp down` have each been run once, by
hand, and both completed. The terminal-gated list in
`reference/remaining-checks.md` is otherwise **still unrun** — staging
invisibility, the machine-wide freeze seen from a second terminal, sudo
exercised exactly once, `sudo camp up` refused, `trusted.overlay.*`
forensics, the kill-point matrix, the rename race. That list is the next
session's first job, and it needs a person at a terminal.

**Person-gated**: the in-composition OpenSSH run and the keyring
measurement are still waiting. The configuration now has the `session:`
section they need; what is missing is `.workspace/bin/` in the workspace
repository with the three launchers, and a `camp accept` after it appears.

## The next task: output and logging

Decided with the owner in this session, not yet built. This is the next
piece of work.

**Every line says what happened and the one fact needed with it, in one
line.** The progress lines used to explain themselves — why a lock is
taken, what a capability drop is for — and that is what made a failed run
unreadable. The long form belongs in `camp explain` and in the documents.
Refusals stay full, because they are repair instructions (§21), but one
problem is stated once: nine mounts failing the same check must be one
refusal naming nine paths, not nine refusals. **That grouping is not
built yet** and is the largest remaining piece of this.

**Two columns, nothing ragged.** The marker in the first, the text in the
second, and every continuation line in the second as well.

**Colour**, only where it is honestly available: the stream is a terminal,
`NO_COLOR` is unset, `TERM` is not `dumb`. Only the marker is coloured, so
the message stays calm and greppable. `[ERROR]` bold red so it cannot be
missed, `[OK]` green, `[NOTE]` blue, `[HINT]` yellow. Never into the log
file — the colouring is per sink, not baked into the string.

**Terminal width is never queried.** It can change during a run, and text
folded to a number camp picked cannot be reflowed by whatever reads it
later. Short lines are the answer; tty-ness may be detected, width may
not.

**Streams, by concept rather than by habit.** stdout is the command's
product — what you would pipe: `plan`'s plan, `explain`'s description,
`status`'s listing, `list`'s records. stderr is everything about the run:
progress, warnings, refusals, errors. stdin is read by exactly one thing,
the privileged helper reading its job. Today this is mixed — `camp up`
puts its price paragraph and closing line on stdout, `camp plan` puts
refusals on stdout, `doctor` puts a rendered error there too — and that
mixing is what made the output feel undifferentiated.

**A log, always on.** Every line that goes to stderr goes to a file as
well, under `.camp/logs/`, rotated by size with a few files kept. Not a
second format and not a second severity system: the same lines, with a
timestamp. Always on rather than configurable, because a log you have to
remember to switch on is missing exactly when the surprising run happens.

**Timestamps** in RFC 3339 with the machine's own offset —
`2026-08-16T20:09:33.412+02:00`. Standard, sortable, local as asked. The
one caveat worth knowing: across a daylight-saving change local times
repeat for an hour, so a plain lexicographic sort is briefly out of order
even though the offset in the string is unambiguous to a parser. The state
record stamps UTC and was left alone.

**No logging framework.** Not zap, logrus or zerolog. What those buy —
runtime level filtering, several appenders, pattern layouts, structured
fields, async buffering — is for a long-running process emitting events
somebody greps later. camp is a short command a person watches, with one
sink and prose composed for a reader. Adopting one would mean writing a
custom handler to reproduce these sentences, and gaining levels nobody
sets. If several sinks and levels are ever genuinely wanted, `log/slog` is
in the standard library and needs no dependency; rotation is about forty
lines either way.

**Every command narrates**, not only `up` and `run`: `down` per operation
as it tears down, and the rest where they do something worth reporting.

## Also decided, also not built

**The `.camp` directory.** `config.yml` is the only file a person edits,
and it sits beside `inventory`, which nobody would ever assemble by hand —
so people meet the directory and back away. Decided: the top level holds
`config.yml` and its README alone, everything camp writes moves into one
clearly named container beneath it, and every machine-written file says so
in its first line. The inventory can carry that line safely: its records
always begin `lower` or `upper`, so a `#` line can never be one. The
container's name is still open — `camp/` was sketched, but `.camp/camp/`
reads badly.

**A `session:` section is now live** in this environment's own
`.camp/config.yml`, declaring `GIT_SSH_COMMAND`, the launcher directory on
`PATH`, and `OUTER_PATH`. It is written but not committed in the camp-env
repository, along with the submodule pointer.

## The rest of the review findings

Twenty-four remain, in the order the verification put them: the lexical
rather than descriptor-confined `fsx` boundary (003), the unpinned overlay
and move operands (004), the record that cannot drive status or a
configuration-free teardown (007, 008), `down` reporting success after
cleanup failed (013), allow-listed directories as writable holes (005),
the namespace init rebuilding a plan under locks taken for another (012),
git's operational failures read as "nothing tracked" (011), the islands
paths (009, 010), then the narrower ones (015–027) and one deletion (028).

Two are documents, not code, and are the owner's: the specification
accepting a mid-session new-root-entry window its own invariants forbid
(014), and `constraints.md` never having been updated for the C18/C19
supersession while the specification's own cross-reference claims C1–C34
(017).
