# Review and final plan — the session section (`session:`)

**Status: the design is settled, in two rounds.** The proposal in
`2026-08-16-design-session-environment.md` was reviewed against the
code and the brief and adopted with five amendments (§3 below); a
second round with the owner then reshaped the mode boundary — the
keys moved under a `session:` section, the privileged-mode refusal
became an announced non-application, and the commands narrate their
steps (§4 below). The result is written into the specification in
the present tense — `reference/spec.md` §4 (C35), §6 (the `session:`
grammar), §14 (both modes), §16 (plan/explain and the narration),
§17 (the refusals), §19 (one sentence), §23 (the open register).
**The specification is the normative source; this file is the review
record and the implementation session's handoff**: what was
verified, what changed in review and why, and the build order with
acceptance checks.

Evidence labels follow the design document's convention: *(measured:
source inspection)* means this session read the named current Go
source; *(measured: handoff)* means the 2026-08-16 session ran it and
`2026-08-16-handoff-ssh-inside-a-session.md` records it; *(reasoned)*
is a judgment. No product code, test, or composition was run or
changed in this session; only the three notes files named at the end
were written.

## 1. What the review verified in the code

The project's rule is that the code is the truth, so every
load-bearing claim in the design was checked against the source. All
of them held. *(measured: source inspection, 2026-08-16)*

- The strict configuration reader ends at `identity:` and refuses
  unknown top-level keys (`internal/config/config.go:296-345`,
  `KnownFields(true)` at line 336). `environment:` is a new field,
  not a loosening.
- The launcher appends `CAMP_SESSION=<live-hash>` to the init's
  `execve` environment (`internal/session/session.go:136`); the init
  inherits it and passes it on through `os.Environ()`. Nothing in the
  repository reads it back. Removal is internally safe.
- The workload environment today is the init's environment plus
  `CAMP_LIVE` and `PWD` (`session.go:373`).
- The workload binary is resolved with `exec.LookPath` **before** the
  child's environment is constructed (`session.go:365` versus `373`),
  and the fallback shell comes from the init's own `SHELL`
  (`session.go:396-401`). Both would ignore a declared `PATH` or
  `SHELL`. The design's effective-lookup correction is load-bearing,
  not pedantry: without it, `plan` would print a declared `PATH`
  while the host's command silently ran.
- The capability drop precedes the workload start
  (`session.go:268-272`), so "apply after the drop" fits the existing
  structure without reordering anything.
- `camp explain` renders whenever a usable plan comes back, checking
  only `built.Live == ""` and dropping refusals that arrive beside a
  plan (`internal/cli/inspect.go:31-34`). The design's demand that
  `explain --privileged` honour every refusal fixes a real,
  pre-existing gap, and the fix should honour *all* refusals, not
  only the new one. `camp plan` already prints the plan and then the
  refusals with a non-zero exit (`internal/cli/cli.go:163-179`),
  which is exactly the behaviour the design asks of
  `plan --privileged` — no control-flow change needed there.
- The privileged record and the helper job serialise mounts, paths
  and identities only (`internal/state/state.go:84-119`,
  `internal/privileged/job.go:56-105`); no environment value has a
  path into either format today, and a regression test keeps it so.
- Generation receives `os.Environ()` plus its own `CAMP_GEN_IN`,
  `CAMP_GEN_OUT`, `CAMP_ENV`, `CAMP_LIVE` contract
  (`internal/gen/run.go:148-152`); the map stays out of it.
- The host-side repair the owner rejected is currently recommended in
  four places and all four must be replaced:
  `internal/report/explain.go:119-137`, `README.md:171-185`,
  `docs/install.md:116-161` (the "ssh inside a session" section), and
  `docs/how-it-works.md` around line 376.

## 2. The verdict

**Adopted.** The design answers the brief's six constraints as well
as any candidate can, and its core decisions survive attack:

- one general key, `environment:`, with no program name anywhere in
  the core; the class is served, not the instance *(reasoned)*;
- candidate 5 as the mechanism, candidates 1 and 2 accompanying it,
  candidate 4 reduced to the one mode refusal, candidate 3 and the
  sudo-preserving run mode deferred with explicit reopening triggers
  (now in spec §23) *(reasoned)*;
- session-scoped, with `down` unconditional from its record
  *(reasoned; preserves invariant 4)*. The design's privileged-mode
  *refusal* was superseded in the second round by the announced
  non-application — §4 below has the reasoning;
- interpolation as a single simultaneous non-shell substitution;
  undefined references refuse; no sibling references, so no cycles
  and no order sensitivity *(reasoned)*;
- application strictly after the capability drop, with the effective
  `PATH`/`SHELL` used for command resolution — the two corrections
  the review rates as the design's most valuable, because each closes
  a "looks applied, is actually something else" hole *(measured:
  source inspection for both current gaps)*;
- `CAMP_SESSION` removed, `CAMP_` reserved, `CAMP_LIVE` and `PWD`
  the whole camp-owned contract. The design could search only the
  camp repository for consumers; this review searched the owner's
  other repositories too — `camp-workspace`, `camp-notes`,
  `diet-coach`, `diet-coach-workspace` — and found none, so the
  design's one unmeasurable assumption is now measured as far as
  this machine allows *(measured: grep across the four
  repositories, 2026-08-16)*;
- inherited insertions rendered as `<inherited NAME>` in plan and
  explain, never as resolved bytes. The handoff's sketch said "print
  the value after interpolation"; the design's deviation is right,
  and this environment is itself the proof — plan output here lands
  in agent transcripts. Literal configuration text stays visible, no
  name-based secret guessing exists *(reasoned)*;
- resolved values in no record, job, report, or file *(reasoned)*;
- `camp init` teaches the grammar in comments and ships no active
  ssh policy *(reasoned)*.

## 3. The five amendments

**A1 — no unconditional per-session warning.** The design had every
namespace session print an ownership-view warning to stderr before
the workload. Dropped. The namespace mode is the primary mode, used
many times a day; a warning printed unconditionally at every start is
noise by the second day, and a warning nobody reads any more is the
"protection that looks installed and does nothing" the project calls
worse than none. The privileged mode's price line is not a precedent:
it announces an effect camp is causing that moment, in the
exceptional mode; this would have announced a property that matters
only if the user later does one particular kind of work, in the
default mode. The class stays loud where somebody deciding or
debugging will actually read it: the "Ownership view" section of
`camp explain`, one line in `camp plan`'s namespace output, and the
documentation (spec §16). *(reasoned — this is the review's largest
deviation; the design's own §12.2 concedes the habituation cost.)*
**Revised in the second round:** with the commands now narrating
their steps (§4), the session's identity line carries the ownership
fact as one clause — "only your uid is mapped; files owned by anyone
else appear as nobody" — so the forensic breadcrumb Codex wanted in
captured logs exists after all, as a fact of the run inside a
compact block rather than a standalone warning banner.

**A2 — one name grammar for both interpolation forms.** `${NAME}`
accepts the same `[A-Za-z_][A-Za-z0-9_]*` grammar as `$NAME`; the
braced form exists to delimit a name from adjacent text
(`${CAMP_LIVE}bin`), not to admit exotic names. The design allowed
any `}`-free name in braces; no known case needs one, and the
narrower rule is a refusal with a stated limit rather than a corner
nobody tests. *(reasoned — the project's no-abstraction-without-a-
caller rule)*

**A3 — `$PWD` as an interpolation input refuses.** The design
resolved `$PWD` to the live path. But a reader writing `$PWD` most
plausibly means the invoking directory, which does not exist inside
the session — resolving it silently to something else is a guess.
The refusal says the reference is ambiguous and names `$CAMP_LIVE`
as what to write. *(reasoned — refuse rather than guess)*

**A4 — the skeleton points at the recipe.** `camp init`'s commented
`environment:` example stays generic (the design's choice, kept),
and the comment names the documentation section that carries the
complete OpenSSH application, so the path from first contact to the
working configuration is one hop, not a search. *(reasoned)*

**A5 — `env:` versus `environment:` is said once, everywhere the key
is introduced.** The configuration already has a top-level `env:`
meaning the environment *root directory*; the new key is the
*process environment*. The collision is survivable — the shapes
differ — but every introduction (spec §6, the skeleton comment, the
docs) carries the one distinguishing sentence so no reader has to
work it out. Renaming either key was considered and rejected: `env:`
is shipped, and the alternatives to `environment:` (`variables:`,
`workload_environment:`) are vaguer or clumsier than the standard
name. *(reasoned)*

## 4. The second round — the mode boundary

The owner attacked the reviewed design at its mode seam: the
configuration was common to both modes, and a flat `environment:`
key does not say in the file what it applies to. The attack found a
real gap and forced a reversal.

**The gap.** `environment:` is not the only session-scoped key.
`identity:` selects the namespace's uid route, is equally meaningless
in the privileged mode — and the current code silently ignores it
there: `config.Identity` is read exactly once, in the session
launcher (`internal/session/session.go:116`); no privileged-path
reader, no refusal *(measured: source inspection)*. Neither the
design nor the first review round noticed. So the "mode-scoped key"
class had two members all along, the second already violating the
design's own "never silently ignored" principle, and the
no-abstraction-without-a-second-caller argument against structural
scoping was standing on a miscount.

**Three shapes were weighed.**

1. *Flat keys, privileged mode refuses* (the first round's position,
   extended to `identity:` for consistency). Correct but illegible:
   the scope lives in the spec and in a refusal met later, not in
   the file; and the refusal forces editing the configuration to
   move between modes while telling the user nothing a printed line
   could not.
2. *Per-key mode flags* (`sandbox_only:`-style). Rejected: on the
   keys where such a flag would be legal it would be mandatory —
   it records no decision — and everywhere else it would refuse.
   Grammar simulating a choice that does not exist. A real per-entry
   mode qualifier returns only with mode-only *mounts* (the
   handoff's candidate 4), which stays deferred.
3. *A scoped section.* Adopted, as **`session:`**, holding
   `identity:` and `environment:`. The name is the tool's own word
   for exactly the scoped object — the supervised run `camp run`
   and `camp shell` start, which the spec has always called a
   session and has never called the privileged mode. Not a mode
   name (`namespace:`): the configuration describes the composition
   and the command chooses the mode. Not `sandbox:`: camp documents
   precisely that it is not one (invariant 8). No `host:`
   counterpart: no host-only key exists, and a one-sided section is
   honest about that.

**Announcement replaced refusal.** The first round rejected a
skippable section as "scoping by silence". The owner's further
requirement — the commands print, in order, what they do — dissolves
that objection: `camp up` with a `session:` section present prints
one line saying the section configures the session this mode does
not start, naming `camp run`/`camp shell` as where it applies. An
explicit statement of non-application cannot "look applied", which
was the whole ground for refusing; and the refusal's one remaining
effect — forcing a config edit between modes — was pure friction.
The empty-map-still-refuses rule from the design dissolves with it:
an empty map simply declares nothing.

**Narration.** `camp run`, `camp shell` and `camp up` print a short
line per frame step as it completes (spec §16): locks, gate,
generation, mounts and verification; the session's identity route
and declared environment names as applied; up's record, helper,
move, machine-wide effects, and the session announcement when the
section is present. The identity line carries the ownership-view
clause (see A1's revision above). This makes the two modes legible
at the moment of use, which config structure alone never could —
the section says *where* a key applies, the narration says *what is
happening now*.

## 5. The build order

For the implementing session. The normative behaviour is in the
specification sections named above — build from those; this list is
the order and the acceptance. Each stage ends with its checks green
and `go build ./... && go vet ./... && gofmt -l . && go test ./...`
clean. "Unattended" needs no installed binary, sudo, network peer or
person; install-gated checks run only through the installed binary
and its AppArmor profile, and **a skip is not a pass**. The working
style: commit subjects one line, imperative; bodies two to six lines,
only the why and what was measured.

**Stage 1 — grammar and pure resolution.** A small `internal/envx`
package (or a tightly scoped file in `config` if dependencies stay
one-way): tokenisation, validation, simultaneous resolution against
an explicit base map, reserved names, the effective-environment
merge, safe display reconstruction, and lookup of a bare command
against an explicit `PATH` value. No filesystem access, no process
mutation, no shell anywhere. Configuration parsing gains the
`session:` section — `identity:` moves under it, `environment:` is
new inside it — strict as everything else, and unknown keys inside
the section refuse. The old top-level `identity:` gets a refusal
that names the move (`session.identity:`), not the generic
unknown-key message: a shipped key that relocates owes the reader
its forwarding address.

*Accept, unattended:* table-driven tests cover absent versus `{}`;
literal values; override versus inherit; inherited-empty versus
absent; `$NAME`, `${NAME}`, adjacency, `$$`; no recursive expansion
of inserted bytes; no sibling references; declaring `PWD` and
`CAMP_*` refused; referencing `$PWD` refused with `$CAMP_LIVE` in
the message; referencing an absent name refused; non-string YAML
values, empty and `=`-bearing and NUL-bearing names, NUL in values,
malformed and unclosed `$` forms each refused with their §17 rule;
control bytes rendered reversibly; a map with four defects reports
all four in one pass.

**Stage 2 — planning, the mode boundary, and reporting.** Resolution
wired into namespace planning with authoritative `CAMP_LIVE`; the
privileged path (`up`, `plan --privileged`, `explain --privileged`)
prints the one-line `session:` announcement when the section is
present — applied nowhere, refused nowhere, silent nowhere; while
in there, fix `internal/cli/inspect.go:31-34` so `explain` honours
*every* refusal that arrives beside a plan (a pre-existing gap);
the session block in `plan` and `explain` with `<inherited NAME>`
rendering; the explain "Ownership view" section per spec §16,
replacing the host-side recommendation in
`internal/report/explain.go`; `camp up`'s in-order narration lines
(spec §16), the announcement among them.

*Accept, unattended:* plan output is byte-stable under differently
ordered YAML maps; a sentinel value inherited from the test
environment appears in no plan text, explain text, state JSON or
helper-job JSON, while camp-owned paths do appear; privileged
planning with a populated and with an empty `session:` both produce
the announcement line, exactly once, and no session block; a
recorded privileged teardown still builds and runs from its record
after the configuration gains `session:`; `explain --privileged`
prints no tree description when any refusal stands.

**Stage 3 — application in the session.** Remove
`CAMP_SESSION` (`session.go:136` and the inheritance behind it);
build the duplicate-free effective environment in the init without
`os.Setenv`; attach it to the workload only after `capsx.Drop()`
succeeds; choose the shell from the effective `SHELL`; resolve a
bare argv[0] through the effective `PATH` via the stage-1 lookup,
never `exec.LookPath`; absolute and slash-containing argv[0] does
not search. `camp run` and `camp shell` gain their in-order
narration lines on stderr (spec §16); the identity line carries the
ownership-view clause.

*Accept, unattended:* command-construction tests with an explicit
base environment and a scratch executable prove the declared `PATH`
selects it and an absolute argv does not search; a missing command
names that the configured `PATH` was searched without printing its
bytes; the effective `SHELL` is selected; the init's own process
environment is unchanged after construction; the child list has no
duplicate names; `CAMP_LIVE` and `PWD` win over any inherited entry;
`CAMP_SESSION` appears nowhere in the tree (`grep` guard); a
rendering test fixes the narration lines — order, stderr only, the
ownership clause on the identity line, no inherited environment
bytes in any of them.

*Accept, install-gated:* in a real one-level namespace, a declared
sentinel is visible to a direct workload, an interactive shell, and
a daemonised descendant; `/proc/1/environ` read from inside proves
the sentinel absent from camp-as-init; `/proc/self/status` proves
the workload holds no capability; a workspace executable reachable
only through the declared `PATH` is selected by `camp run -- <bare
name>`; after exit the parent shell's variables are unchanged and
the live directory is empty outside.

**Stage 4 — the OpenSSH application and the documents.** Replace the
rejected host-side repair in `README.md`, `docs/install.md`,
`docs/how-it-works.md` with the composition-owned pattern: the
`GIT_SSH_COMMAND` declaration for git, and a workspace launcher
directory on the declared `PATH` for `ssh`, `scp` and `sftp`, each
launcher finding the original program through an inherited path
saved under a composition-chosen name — no `/usr/bin/ssh`, no
`CAMP_*` switch, and camp neither generates nor blesses the
launchers. `camp init`'s skeleton gains the commented `session:`
block with the generic example, the sentence distinguishing
`session.environment` from the top-level `env:` root, the scope
sentence (what `camp up` does with the section), and the pointer to
the docs recipe; `examples/config.yml` gains the same commented
stanza.

*Accept, unattended:* with fake `ssh`/`scp`/`sftp` executables and
logging launchers in a scratch workspace, a git operation against a
local fake remote receives the declared command; each entry point
typed directly receives `-F` through its launcher; the launcher
finds the original through the saved path; no absolute distribution
binary path appears in implementation or examples; a missing
launcher produces the program's own loud failure or a loud lookup
failure, never a success camp reported.

**Stage 5 — close.** The full gate, then the namespace test group
through the installed binary so nothing hides behind a skip. Add to
`reference/remaining-checks.md` the two person-gated rows: the
in-composition OpenSSH run (`ssh`, `scp`, `sftp`, `git ls-remote`
against a real peer through the launchers, outside behaviour
unchanged in the same terminal) and the keyring/libsecret
measurement. Then read spec §4 (C35), §6, §14, §16, §17, §19, §23
once more against what was built — where they disagree, stop and
say so; the code is the truth, but a silent divergence is how the
last spec grew stale.

## 6. Out of scope for the implementing session

- **camp-env's own adoption**: the `environment:` stanza in
  `.camp/config.yml`, the launcher files in the workspace, and the
  `camp accept` a new workspace root entry requires. These are owner
  actions, from a terminal outside any session — the workspace is
  read-only inside one, by design.
- **The diet-coach migration**: the launcher directory belongs under
  the workspace's `.workspace/` container per spec §5; nothing to do
  until the migration itself.
- **The full present-tense rewrite of `spec.md`**: still its own
  task (`.notes/README.md`); the sections this design touched are
  already written in the target style.

## 7. Guardrails, restated

No file outside the composition is written for this feature — not
`~/.local/bin`, not a shell startup file, not global git
configuration. No resolved value in any record, job, report, or
file. No `MNT_DETACH`, no `--force`, no new mount target outside the
composed tree. Declared values touch no process holding a
capability. Whatever refuses, refuses with the thing named, what is
true, and the way out.
