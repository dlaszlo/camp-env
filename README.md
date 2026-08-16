# camp — the design record

This repository is **not** part of the tool. It is the record of how the
tool was designed and what was measured while building it: the working
material that would otherwise have to live in the product's history, or
nowhere.

It is mounted read-write into the composed tree at `.notes`, so it can be
read and edited from inside a session. The workspace repository is
read-only in there on purpose; this one is not, because the design record
is written *while* the work happens.

```
design/
├── spec.md                    the single source the implementation was built from
├── constraints.md             what the kernel, git and the instruments actually do (C1–C34)
├── measurements-2026-08-16.md what the build itself measured, with how
├── install-gated-checks.md    the checks that need an install, sudo, or a person
├── review-2026-08-15.md       the second-pass review, with its mount measurements
├── review-of-spec.md          the review of the specification (17 findings)
├── model.md                   an earlier state of the design — historical
├── redesign-2026-08-15.md     the redesign that superseded half of it — historical
└── realpair_test.go.txt       an acceptance check against a real repository pair
```

`spec.md` wins where it disagrees with `model.md` or
`redesign-2026-08-15.md`; those describe earlier states and are kept as
records rather than rewritten. `constraints.md` records what was *found*
and cannot be argued with; the rest records what was *decided* and can.

Keep it private. It names real paths, real repositories and real
decisions, and none of that is anybody else's business.
