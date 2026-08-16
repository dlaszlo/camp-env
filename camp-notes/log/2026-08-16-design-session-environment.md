# Design and plan — ownership-sensitive programs inside a session

**Status:** proposed design for review. No product code or configuration was
changed in this session. Candidate 5 wins after being weighed; candidates 1
and 2 accompany it, candidate 4 is used only as a narrow mode refusal, and
candidate 3 stays deferred. *(reasoned — decision)*

**Short name:** `session-environment`. *(reasoned — naming decision)*

## 1. Decision in one page

Add one top-level configuration key, `environment:`, whose value is a map of
environment-variable names to string expressions. Resolve it once for a
namespace session and apply it to the workload process and its descendants,
after camp has dropped every capability. Do not apply it to camp, generation,
the session init, or a privileged helper. *(reasoned — design decision)*

```yaml
environment:
  GIT_SSH_COMMAND: "ssh -F ${HOME}/.ssh/config"
  PATH: "$CAMP_LIVE/.workspace/bin:$PATH"
```

The map is not an ssh feature. `GIT_SSH_COMMAND` is one use; `PATH` can expose
composition-owned launchers for programs with no option variable; another
program can use another variable without a camp change. The core knows no
program name and no distribution path. *(reasoned — design decision)*

`environment:` is namespace-session-only. `camp up`, `camp plan
--privileged`, and `camp explain --privileged` refuse it because the
privileged mode starts no workload to receive it. Accepting it there would
make a declared effect look applied while doing nothing. `down` must still
tear down from its record even if the current configuration has since gained
this key. *(reasoned — design decision, preserving invariant 4)*

This is not a transparent repair for the uid view. It gives a composition a
general, versionable place to adapt programs that already have an environment
or command-path control. Plain `ssh`, `scp`, and `sftp` still need launchers in
a workspace-owned tool directory if they are to work in namespace mode;
programs that silently record uid 65534 cannot be corrected by an environment
map. Ownership-faithful work uses `camp up`, which creates no user namespace,
and pays that mode's documented machine-wide workspace freeze. *(reasoned —
known limit and trade-off)*

Every namespace session prints one short warning before its workload starts:

> camp: namespace ownership view: host files that are not yours look owned by
> “nobody”; a tool that records ownership can silently record the wrong owner.
> `camp explain` gives the limits and the `camp up` alternative.

The warning goes to camp's stderr once per session, including non-interactive
sessions, so the dangerous member of the class leaves a line in an automated
log rather than remaining completely silent. It changes no exit status and no
workload stdout. *(reasoned — design decision)*

Keep and formalise `CAMP_LIVE` as camp-owned: it is the canonical composed-tree
path and is available while expressions are resolved and in the workload.
Reserve `PWD` to the actual workload directory. Remove the current,
undocumented `CAMP_SESSION` export rather than turning a stable live-path hash
into a public “session” contract. Reserve the whole `CAMP_` prefix against
configuration declarations. *(reasoned — design decision; the compatibility
assumption is named in §3)*

No mount kind, mount target, target rule, repository rule, generation
contract, state schema, or privileged-helper job changes. In particular, no
path outside the composed tree becomes a mount target. *(reasoned — design
boundary)*

## 2. Evidence and method

This document uses only the two labels requested by the brief:

- *(measured: handoff)* means the 2026-08-16 session ran the command recorded
  in `2026-08-16-handoff-ssh-inside-a-session.md`; this session did not repeat
  it. *(measured: the handoff records command, output, machine, and date)*
- *(measured: source inspection)* means this session read the named current Go
  source or test with `sed`/`rg`; it is a measurement of the implementation,
  not a runtime claim. *(measured: read-only inspection of `camp/` on
  2026-08-16)*
- *(measured: project record)* means the current specification, constraints,
  build-measurement log, or remaining-checks register records the result and
  how it was obtained. *(measured: read-only inspection of the named record)*
- *(reasoned)* means a proposed rule or a consequence not run on this machine.
  Planned checks that would turn it into a measurement are in §11.
  *(reasoned — evidence convention)*

No `camp run`, `camp up`, composition build, compilation, or test command was
run in this session. The merged directory was left alone. Namespace checks
need the installed binary path on this machine, and privileged checks need a
terminal for sudo; `reference/remaining-checks.md` records both gates.
*(measured: this session's tool log; measured: project record for the gates)*

### Current implementation truth

The current strict YAML reader has top-level fields through `identity:` and no
environment field. Unknown top-level keys are refused by `KnownFields(true)`;
steps are parsed separately as an ordered sequence. *(measured: source
inspection of `internal/config/config.go` and `config_test.go`)*

The namespace launcher currently appends `CAMP_SESSION=<live-hash>` to the
re-executed init's environment. After mounting and capability drop, the init
starts the workload with its inherited environment plus `CAMP_LIVE=<live>` and
`PWD=<live>`. It resolves the workload binary with the init's inherited
`PATH`, before constructing the child's final environment. *(measured: source
inspection of `internal/session/session.go`, especially `Launch`, `Inside`,
and `startWorkload`)*

That ordering is security-relevant: putting configured values on the init's
`execve` environment would let values such as `LD_PRELOAD`, runtime tuning
variables, or a replacement `PATH` affect camp while the init still carries
`CAP_SYS_ADMIN` and `CAP_DAC_OVERRIDE`. The declarations therefore have to
remain inert data until after `capsx.Drop()`. *(reasoned from the measured
process order and the general semantics of process environments)*

The launcher validates and generates as the user, then the init reloads the
configuration, rebuilds the plan, and revalidates generated output before it
mounts. A design that resolves the workload environment in both halves can
preserve this second-reader property without sending declarations through the
capability-bearing init's own environment. *(measured: source inspection of
`internal/cli/compose.go` and `internal/session/session.go`; reasoned for the
extension)*

`camp plan` already renders the derived mount plan; `camp explain` already has
a namespace ownership section, but it currently recommends a host-global git
setting and a shell alias. The same host-side recommendation appears in the
README, `docs/install.md`, and `docs/how-it-works.md`. *(measured: source
inspection of `internal/report/report.go`, `internal/report/explain.go`, and
`rg` over the three documents)*

`camp explain --privileged` currently continues when `plan.Prepare` returns a
usable plan plus refusals; it checks only whether `built.Live` is empty. The
new mode refusal would therefore be printed by `plan --privileged` and enforced
by `up`, but could be hidden by `explain --privileged` unless that command is
corrected to honour every refusal. *(measured: source inspection of
`internal/cli/inspect.go`; reasoned for the consequence)*

The privileged state record serialises mount operations explicitly rather
than serialising the Go `plan.Plan` wholesale, and the helper job likewise
contains only mount instructions. Resolved environment values therefore need
not enter either format, and a regression test should keep it that way.
*(measured: source inspection of `internal/state/state.go` and
`internal/privileged/job.go`; reasoned for the test)*

### Runtime facts inherited from the brief

Inside the measured namespace, the uid map contains only host uid 1000 mapped
to inside uid 1000; host-root-owned ssh configuration reads as uid 65534,
`nobody`, and OpenSSH refuses it before a connection. Outside, the same ssh
commands work. *(measured: handoff — `cat /proc/self/uid_map`,
`stat -c`, `ssh`, and exit status were recorded)*

`ssh -F ~/.ssh/config` worked and skipped the system-wide file. Git found an
`ssh` launcher through `PATH`; `scp` bypassed that launcher for its internal
ssh and succeeded only when `scp` itself received `-F`; `sftp` has the same
shape. *(measured: handoff — wrapper logs and live ssh/scp/sftp runs were
recorded)*

A command launched directly by `camp run` read no shell startup file, and ssh
has no environment variable for its own options. An alias therefore cannot
cover direct workloads, and an environment entry can cover plain ssh only by
changing command resolution, not by supplying a nonexistent ssh option
variable. *(measured: handoff — shell flags, empty `BASH_ENV`, missing aliases,
and ssh(1)'s environment list were recorded; reasoned for the consequence)*

No unprivileged uid mapping, subordinate-id route, or idmapped mount can make
host uid 0 appear as root or as the caller in this namespace. A transparent
rootless answer has to avoid reading the ownership-sensitive host object,
adapt the program, or stop using a user namespace. *(measured: project record
for the map and installed route; reasoned from the kernel mapping rule recorded
in the handoff)*

## 3. Assumptions and open measurements

1. The configuration author and the namespace workload run with the same host
   user authority. An environment map may cause arbitrary user-level code to
   run when the workload consults `PATH`, `LD_PRELOAD`, a language startup
   variable, or a program-specific command variable; that is within the same
   authority as the existing custom generator, not a grant of root. *(reasoned
   — trust assumption)*
2. `environment:` is configuration intent and may be versioned. Literal
   secrets should not be placed there; inherited-secret references are allowed
   and their resolved bytes are not printed. Same-uid processes may still read
   a workload's environment through `/proc`, because camp is not a security
   boundary. *(reasoned — secrecy assumption and invariant-8 consequence)*
3. No external consumer depends on the undocumented `CAMP_SESSION` variable.
   Repository-wide source and documentation search found no consumer, but that
   cannot measure scripts outside the repository. If the reviewer knows of a
   consumer or applies an “every observable value is API” policy, retain it for
   one release with a deprecation notice, then remove it; do not use it for the
   new mechanism. *(measured: source inspection for known consumers; reasoned
   — compatibility fallback)*
4. The existing Linux-only product boundary remains. The design is
   distribution-independent within Linux; it is not a portability proposal for
   non-Linux process or mount semantics. *(measured: source inspection of
   `internal/preflight/preflight.go`; reasoned — scope assumption)*

The following remain measurements, not design choices:

- whether a declared variable reaches a direct workload, an interactive shell,
  a daemonised descendant, and a program resolved only through the declared
  `PATH` in a one-level namespace created by the installed binary;
  *(reasoned — install-gated check specified in §11)*
- whether the real workspace launcher arrangement covers `ssh`, `scp`, and
  `sftp` on this machine without an absolute binary path; *(reasoned —
  install- and keyboard-gated check specified in §11)*
- whether the desktop keyring/libsecret path works across the namespace as the
  handoff expects. *(reasoned — person-at-keyboard check specified in §11)*

## 4. The six-constraint weighing

Legend: **pass** satisfies the constraint as a design; **partial** buys some of
it with a named cost; **fail** contradicts it. Every assessment below is a
trade-off judgment, not a new runtime measurement. *(reasoned — scoring rule)*

| candidate | 1. least pain | 2. no host meddling | 3. complete inside itself | 4. no principle bent | 5. invisible or understandable | 6. distribution-general | disposition | evidence |
|---|---|---|---|---|---|---|---|---|
| 1. Documentation only | fail: every user carries the limit | pass | fail: changes no outcome | pass | partial: understandable only if read | pass | accompany the solution, never stand alone | *(reasoned)* |
| 2. Use `camp up` | partial: sudo and a machine-wide workspace freeze for one connection | partial: no persistent host edit, but two documented machine-wide mount effects | pass for ownership and pid fidelity because there is no user namespace | pass: it stays within invariant 7's named mode | pass: the cost is printed | pass: no ssh path assumption | retain as the faithful escape, not the primary repair | *(reasoned from measured mode shape)* |
| 3. User-owned copy over a host path | partial: invisible after per-path setup | pass only in namespace mode; privileged must refuse | partial: can cover named files, not discovery or silent artefacts | fail: relaxes the all-targets-under-live rule and widens the mount safety argument | fail: a stale copy looks authoritative | partial: mechanism is general, each path/layout is not | defer until an unadaptable second case exists | *(reasoned)* |
| 4. Mode-only mounts or `when: namespace` | fail: it solves nothing by itself | pass when it refuses privileged use | fail: policy without a mechanism | pass for a kind-level refusal; partial for a general qualifier with no second caller | pass if the refusal names why | pass | use only the narrow namespace-only rule; reject `when:` | *(reasoned)* |
| 5. Declarative workload environment | pass for environment-aware programs; partial for plain ssh launchers | pass: child environment only | pass as a general in-composition adaptation surface, with `camp up` for fidelity-only cases; not a transparent uid repair | pass: no invariant or target rule changes | pass with plan/explain, strict refusals, and the runtime ownership warning | pass: no program or host path in core | **winner** | *(reasoned)* |
| 6. New sudo-backed, identity-preserving `camp run` | fail: a privilege prompt/session and no unattended-agent route | partial: no persistent edit, but a much larger privileged path | pass for the owner-id class, including silent artefacts | partial: invariants can hold, but the “narrow helper” trust model becomes a long-lived launcher | pass if explicitly selected | partial: Linux-general, privilege frontend policy varies | reject now; revisit only on the trigger in §12.9 | *(reasoned)* |

Candidate 5 wins because it is the only new mechanism that stays entirely in
the composition, gives the user one recorded decision, preserves the mount and
write arguments unchanged, and remains useful for the next adaptable program.
Candidate 1 supplies the explanation and candidate 2 supplies fidelity when a
program cannot be adapted. Candidate 4 contributes only the refusal that stops
candidate 5 from pretending in `camp up`. *(reasoned — conclusion)*

Candidate 5 does not win by pretending to be transparent. For git over ssh it
is one line. For plainly typed OpenSSH programs it is configuration plus
workspace-owned launchers. For silent ownership-bearing artefacts it is a
warning and a deliberate choice of `camp up`, not a rewrite of metadata after
the fact. That is the central cost the reviewer must accept. *(reasoned —
trade-off)*

## 5. Configuration language

### 5.1 Shape

`environment:` is an optional top-level YAML mapping. Its mapping order has no
meaning; entries are rendered in byte order for deterministic output. An absent
key means no declared variables. An explicitly empty map is valid in namespace
mode but still counts as present and is refused by privileged-mode planning,
because mode-scoped syntax is not silently ignored. *(reasoned — proposed
grammar)*

```yaml
environment:
  NAME: "value"
  PATH: "$CAMP_LIVE/.workspace/bin:$PATH"
```

Every value must be a YAML string scalar. Integers, booleans, nulls, sequences,
and mappings are refused rather than coerced; quoting `123` is the exact repair
for a numeric-looking string. Unsetting is not in this iteration: empty string
sets an empty value, and absence inherits. *(reasoned — proposed grammar;
YAGNI because no required unset caller is known)*

A name must be nonempty and contain neither `=` nor NUL, matching the structure
that `execve` receives. A value may contain any byte representable by the YAML
reader except NUL. Output encodes control characters reversibly before writing
them to a terminal. The core does not maintain a registry of program-specific
names, so `GIT_SSH_COMAND` is structurally legal even though git will ignore the
misspelling; that semantic limit is recorded in §13. *(reasoned — proposed
validation and known limit)*

Every name beginning `CAMP_` is reserved. `CAMP_LIVE` is supplied by camp;
future camp-owned names must not collide with configuration written today.
`PWD` is also reserved because camp sets both the actual working directory and
the matching value. *(reasoned — proposed ownership rule)*

### 5.2 Interpolation

Interpolation is a single, non-recursive data substitution, never a shell.
*(reasoned — proposed semantics)*

- `$NAME` references a portable identifier
  `[A-Za-z_][A-Za-z0-9_]*`. *(reasoned — proposed grammar)*
- `${NAME}` is the delimited form and permits any legal environment name that
  does not contain `}`. *(reasoned — proposed grammar)*
- `$$` produces one literal dollar. A lone dollar, an invalid unbraced name, an
  empty braced name, or a missing `}` is refused with the byte offset and the
  `$$` repair. *(reasoned — proposed refusal)*
- A reference reads the environment inherited by the camp command, except
  that `$CAMP_LIVE` and `$PWD` resolve to camp's authoritative live path.
  Inherited `CAMP_*` values are not interpolation inputs. *(reasoned — proposed
  precedence)*
- All declarations resolve simultaneously against that base, not against one
  another. `PATH: "prefix:$PATH"` reads the inherited `PATH`; `A: "$B"`
  does not read a sibling declaration `B`. This makes YAML mapping order
  irrelevant and makes cycles impossible. *(reasoned — proposed semantics)*
- An inherited name that exists with an empty value expands to empty. A name
  that is absent is refused; it is never silently replaced with empty text.
  *(reasoned — proposed typo policy)*
- Inserted bytes are not scanned again. An inherited value containing `$HOME`
  contributes those literal bytes. There is no `~` expansion, command
  substitution, defaulting, splitting, globbing, quote removal, or backslash
  processing. *(reasoned — proposed non-shell boundary)*

Examples:

```text
base PATH=/usr/bin, live=/work/shop-live
"$CAMP_LIVE/bin:$PATH"  ->  "/work/shop-live/bin:/usr/bin"
"$$HOME"                ->  "$HOME"
"$MISSING"              ->  refusal, not ""
```

*(reasoned — examples of the proposed grammar)*

### 5.3 Effective environment and precedence

Build the child environment as one duplicate-free list: preserve the relative
order and bytes of untouched inherited entries; remove entries that a
declaration or camp-owned name replaces; append new declarations in byte-sorted
name order; then append authoritative `CAMP_LIVE` and `PWD`. Reporting sorts
names independently. This keeps mapping order semantically irrelevant without
needlessly reordering the caller's inherited environment. *(reasoned —
proposed semantics)*

`CAMP_LIVE` is the canonical live path. `PWD` has the same value because the
workload's actual cwd is the live path. `CAMP_SESSION` is absent. All other
inherited variables remain byte-for-byte unless the map overrides them.
*(reasoned — proposed contract)*

If `camp shell` has no explicit argv, choose the shell from the effective
`SHELL` value when nonempty, otherwise `/bin/sh`. For an argv whose first item
has no slash, resolve it against the effective `PATH`, not camp init's path.
Never mutate the init with `os.Setenv`; use an explicit lookup routine and the
child's `Cmd.Env`. A missing command after that lookup fails loudly and names
that the configured `PATH` was used, without printing inherited path bytes.
*(reasoned — proposed execution semantics)*

This lookup rule is load-bearing for the advertised composition-owned launcher
case. Setting a child's `Cmd.Env` after calling ordinary `exec.LookPath` would
print a correct-looking plan while still selecting the host command, exactly
the “looks applied” failure the design rejects. *(measured: source inspection
of the current pre-`Cmd.Env` lookup; reasoned for the required correction)*

### 5.4 When resolution happens

The unprivileged launcher parses, validates, and resolves while preparing the
namespace plan, before generation or any mount. The re-executed init reloads
the configuration and repeats resolution from its inherited host snapshot as
part of rebuilding the plan. If the configuration changed between readers,
the second result is the one the workload receives. *(reasoned — proposed
extension of the measured two-reader model)*

Resolution may occur while the init still has capabilities because it only
builds inert strings. No declared string is installed in the init environment,
used for its executable lookup, or added by this feature to a child environment
until `capsx.Drop()` has succeeded. A capability-drop failure starts no
workload. *(reasoned — proposed security boundary)*

Generation continues to receive the invoking environment plus only its existing
`CAMP_GEN_*`, `CAMP_ENV`, and `CAMP_LIVE` contract. A configured generator does
not receive `environment:` declarations as environment entries; as the same
host user, it can still read the configuration file, just as it can today.
End-of-session git/report processes also use camp's original environment, not
the workload map. *(reasoned — proposed scope and non-security boundary)*

The map applies to the first workload process. Ordinary operating-system
inheritance carries it to descendants unless a program deliberately changes or
scrubs its environment. A daemon that keeps the namespace alive keeps its
environment; the parent shell outside camp is never modified. *(reasoned from
process inheritance and the measured supervisor shape)*

## 6. Privileged mode and refusals

### 6.1 Mode refusal

Use stable rule identifier `environment-namespace-only`. The human message is:

> `environment:` is present in `<config>`, but this is the privileged
> (`camp up`) mode. `camp up` mounts the composed tree and exits; it starts no
> workload, so there is no process to receive these variables. Accepting the
> key would make it look applied when nothing used it. Use `camp run --
> <command>` or `camp shell`, or remove `environment:` before using `camp up`.
> Nothing has been mounted.

The command name in the first sentence may be adapted for `plan --privileged`
or `explain --privileged`, but the reason and the two ways out remain. The
refusal fires for an empty-but-present map as well as a nonempty one. *(reasoned
— proposed refusal text)*

`camp explain --privileged` must stop on the refusal instead of rendering a
plan returned alongside it. `camp plan --privileged` may print the otherwise
derived mount plan, then the refusal and its normal “nothing was mounted”
ending. `camp up` refuses before generation, record creation, or sudo. *(reasoned
— proposed command behavior)*

`camp down` is different: it reads the recorded mount list and attempts
teardown regardless of the current key. It may report configuration drift after
teardown, but environment mode incompatibility can never wall the user in.
`status`, `list`, `forget`, `accept`, and `doctor` neither apply nor pretend to
apply workload variables. *(reasoned — proposed invariant-4 behavior)*

### 6.2 Parse and resolution refusals

Use distinct stable rules because the repairs differ. *(reasoned — proposed
message design)*

| rule | condition | message must say | evidence |
|---|---|---|---|
| `environment-shape` | key is not a mapping, or a value is not a string | name the key/value type; show the quoted-string repair | *(reasoned)* |
| `environment-name` | empty name, `=`, or NUL | encode the name safely; say why it cannot form one `name=value` entry | *(reasoned)* |
| `environment-reserved` | `CAMP_*` or `PWD` declared | name camp's authoritative value/prefix; say to reference `$CAMP_LIVE` rather than assign it | *(reasoned)* |
| `environment-value` | NUL in a value | name the variable and byte offset; NUL cannot cross `execve` | *(reasoned)* |
| `environment-expansion` | malformed `$` expression | name variable and byte offset; list `$NAME`, `${NAME}`, and `$$` | *(reasoned)* |
| `environment-undefined` | referenced base name is absent | name declaration and missing name; say unset is different from set-empty; correct, set, or make literal | *(reasoned)* |
| `environment-namespace-only` | present in privileged plan | use §6.1 verbatim in substance | *(reasoned)* |

Canonical message templates, with names and values passed through the safe
renderer, are: *(reasoned — proposed product wording)*

> `environment:` at line `<n>` is a `<sequence|null|scalar>`. It must be a
> mapping from variable names to string values, for example `NAME: "value"`.
> camp does not coerce another YAML shape into process settings.

> `environment.<NAME>` at line `<n>` is a `<number|boolean|null|mapping|sequence>`,
> not a string. Environment bytes have no YAML number or boolean type. Quote
> the intended value: `<NAME>: "<value>"`.

> The environment name `<encoded-name>` is empty / contains `=` / contains
> `\x00`. A child receives each variable as one `name=value` entry, so this
> cannot name one variable. Rename the key.

> `environment.CAMP_LIVE` cannot be set. Names beginning `CAMP_` belong to camp;
> this workload's `CAMP_LIVE` is `<live>`. Reference `$CAMP_LIVE` in another
> value, or remove the declaration. (`PWD` gets the parallel message: its value
> must agree with the actual workload directory.)

> `environment.<NAME>` contains `\x00` at byte `<n>`. A NUL cannot be passed
> through `execve`; remove it or encode it in the consuming program's own data
> format.

> `environment.<NAME>` has an invalid `$` expression at byte `<n>`: `<encoded
> fragment>`. Write `$NAME` or `${NAME}` to inherit a value, and `$$` for a
> literal dollar. camp runs no shell here.

> `environment.<NAME>` refers to `<BASE>`, but `<BASE>` is not set in the
> environment that started camp. An absent value is not the same as a value set
> to empty, and replacing it silently would make a typo look applied. Correct
> the name, set `<BASE>` before running camp, or use `$$` if the dollar was
> literal.

*(reasoned — proposed message templates)*

All syntactic problems in the map are collected in the configuration parser's
normal one-pass refusal list. Undefined references need the command's inherited
environment and are added during planning. No partial map is applied if any
entry fails. *(reasoned — proposed validation order)*

## 7. Plan, explain, and secret handling

`camp plan` prints the camp-owned values, every declared name, whether it
inherits or overrides, and a safe reconstruction of the expression. It does
not print the resolved bytes of inherited references. *(reasoned — proposed
reporting rule)*

```text
workload environment (namespace sessions only):
  CAMP_LIVE = "/home/you/work/shop-live"       camp-owned
  PWD       = "/home/you/work/shop-live"       camp-owned
  GIT_SSH_COMMAND = "ssh -F " + <inherited HOME> + "/.ssh/config"
  PATH = "/home/you/work/shop-live/.workspace/bin:" + <inherited PATH>
  all other variables: inherited unchanged
  applied after camp drops its mount capabilities
```

The renderer expands camp-owned values because they are already paths in the
plan. It replaces each inherited insertion with `<inherited NAME>` and encodes
literal control characters with camp's existing reversible C-style scheme.
This deliberately departs from the handoff sketch's “print the value after
interpolation”: an inherited token or credential must not be copied into a
terminal transcript merely because somebody asked what would mount. The exact
bytes still reach the child and can be inspected there by their owner.
*(reasoned — security/usability trade-off)*

Literal text written directly in the configuration is shown; it is already in
the file being explained. No heuristic redaction by names such as `TOKEN` or
`PASSWORD` is allowed, because guessing which values are secrets would miss
some and hide others without a rule. *(reasoned — refusal-rather-than-guess
principle)*

`camp explain` has a “Workload environment” section with the same safe
rendering, then an “Ownership view” section covering the whole class rather
than only ssh. It names the runtime warning, the composition-owned launcher
pattern, and `camp up` as the ownership-faithful route. *(reasoned — proposed
report)*

No resolved value is written to the privileged record, helper job, work
directory, namespace report, or any repository. The only persistent copies are
literal expressions the user put in the configuration. *(reasoned — proposed
data-lifetime rule)*

## 8. What the design does for every member of the class

| member | namespace-mode answer | what remains | evidence |
|---|---|---|---|
| `git push`/`ls-remote` over ssh | declare `GIT_SSH_COMMAND`; git receives it even with no shell | the config author chooses the user config path | *(measured: handoff for git's direct launch; reasoned for declared delivery)* |
| plainly typed `ssh` | prepend a workspace-owned launcher directory through `PATH`; workload lookup uses that effective path | camp does not author the launcher or assume `/usr/bin/ssh` | *(measured: handoff for PATH lookup; reasoned for design)* |
| `scp`, `sftp` | provide launchers for these entry points too, each supplying `-F`; keep the original host path separately so launchers find the real program without a distribution path | wrapping `ssh` alone is insufficient because scp used an absolute internal ssh path | *(measured: handoff; reasoned for workspace arrangement)* |
| setuid programs (`sudo`, `pkexec`, `newuidmap`) | no environment repair; explain that rootless mode intentionally supplies no elevation | use `camp up` if host-like setuid behavior is truly required | *(reasoned from recorded user-namespace ownership semantics)* |
| git `safe.directory` on a foreign-owned checkout | the composition may declare git's command-scope configuration environment if it deliberately accepts that checkout | camp neither auto-whitelists paths nor edits global git config | *(reasoned)* |
| `tar`, `rsync -a`, container/package builds that record ownership | always print the namespace ownership warning; explain shows the exact risk | no rewriting or detection of the resulting artefact; use `camp up` when owner metadata is part of correctness | *(reasoned — deliberate limit)* |
| `chown` or install-as-another-user | no repair; the requested uid is not mapped | use a mode/environment designed for that ownership model, normally `camp up` or a container, not a fake success | *(reasoned)* |
| owner-checking sockets or runtime directories | preserve all inherited connection variables; allow a program-specific declaration if one exists | no generic stat-result override; `camp up` is the faithful route | *(reasoned)* |
| libsecret/keyring over D-Bus | inherit the existing address/runtime variables unchanged | compatibility is unmeasured and stays an open manual check | *(reasoned; handoff marks it unmeasured)* |
| host pid consumers (`systemctl`, debuggers, profilers) | nothing; environment does not translate pids | existing pid-namespace explanation remains | *(reasoned)* |
| outside processes that need the mounted tree | nothing; environment cannot make a namespace-local mount visible | existing `camp up` mode remains the answer | *(measured: project record for mount visibility; reasoned)* |
| single-instance GUI handoff | nothing; an outside instance may still open the raw path | remains in the manual/deferred register | *(reasoned; project record marks it unmeasured)* |

The OpenSSH workspace arrangement is an application of the mechanism, not
part of camp's grammar. A distribution-neutral launcher must find the original
program through the inherited path saved under a composition-chosen name; it
must not hard-code `/usr/bin`, and it must not activate by testing
`CAMP_SESSION`. The workspace repository is the place a composition already
owns for such development tooling. *(reasoned — application constraint)*

The environment stanza and launchers are a one-time composition change, then
travel with the environment/workspace rather than with each invocation or each
developer's host setup. That is why this qualifies as an answer inside the
composition rather than the rejected “remember `-F`” workaround. *(reasoned —
constraint-3 rationale)*

`camp init` should teach the generic `environment:` grammar in comments but
should not include active ssh settings or manufacture launchers. Not every
machine has OpenSSH, `~/.ssh/config`, or a system-wide file, and a skeleton
that appears to repair a missing launcher would violate the no-guess rule.
The README and install/how-it-works documents carry the complete OpenSSH
example instead. *(reasoned — skeleton decision)*

## 9. Alternatives closed here

The following are rejected so a later session does not rediscover only their
working half.

1. **Type `ssh -F` each time.** It works, but the person carries the tool's
   limitation forever and direct child programs remain unhandled. *(measured:
   handoff for success; reasoned for rejection)*
2. **Write `core.sshCommand` globally.** It covers git but modifies host-global
   configuration, can overwrite an existing user decision, and leaves plain
   OpenSSH tools open. *(measured: handoff for scope; reasoned for rejection)*
3. **Put an alias in shell startup.** Direct `camp run` workloads read no
   startup file, so the protection looks installed and is absent where agents
   run. *(measured: handoff; reasoned for rejection)*
4. **Install host-side wrappers in `~/.local/bin`, conditional on a camp
   variable.** It reaches several programs but wires camp into the host, has a
   name-collision activation condition, and asks the user to maintain global
   files for a session-local fact. *(measured: handoff for behavior; reasoned
   for rejection)*
5. **Automatically set `GIT_SSH_COMMAND` in camp.** It is painless but gives
   the core git/ssh policy, silently overrides an inherited setting, and solves
   one caller. The configuration may choose that value; camp may not.
   *(reasoned — rejection)*
6. **Turn `CAMP_SESSION` into the wrapper switch.** The current value is an
   undocumented stable composition hash, not a unique session identity; a
   global wrapper can activate under an unrelated producer of the same name.
   Reachability through a configured, composition-owned `PATH` already scopes
   the launcher without a switch. *(measured: source inspection for current
   value; reasoned for rejection)*
7. **Add a shipped git-configuring step.** It has nowhere honest to write: not
   a repository, not the host home. As an environment wrapper it has only one
   caller and adds grammar around one map entry. Revisit after a second real
   tool needs the identical higher-level operation. *(reasoned — YAGNI
   rejection)*
8. **Generate arbitrary command shims from a new `commands:` map.** This removes
   workspace files but introduces another language and interception layer,
   still enumerates programs, and still cannot correct silent metadata.
   `environment:` plus ordinary workspace tooling is smaller. *(reasoned —
   rejection)*
9. **Copy a user-owned `/etc/ssh` view over the host path.** A read-only bind
   keeps the refused owner; a copy can work but is stale, cannot copy root-only
   server keys wholesale, meets symlinks, needs distribution-specific path
   discovery, and relaxes the all-targets-under-live rule. *(measured: handoff
   for the machine's paths/symlink and bind ownership; reasoned for rejection)*
10. **Add `when: namespace` to all steps.** There is no second mode-specific
    step today. The environment feature itself carries one explicit mode rule
    and one refusal; a general conditional is an abstraction without another
    caller. *(reasoned — rejection)*
11. **Use FUSE, `LD_PRELOAD`, fakeroot, ptrace/proot, or syscall interception to
    fake `stat` ownership.** Coverage differs for static/setuid programs and
    direct syscalls, the apparent metadata can diverge from what the kernel
    records, and the design would become a security-sensitive runtime rather
    than a mount composer. FUSE is already rejected as the core backend in
    specification §22. *(measured: project record for FUSE decision; reasoned
    for the other mechanisms)*
12. **Create an identity-preserving mount namespace through sudo.** A root
    helper could unshare only the mount/pid view, mount, drop to the caller, and
    leave host uid ownership visible; this is the only newly considered route
    that also fixes silent owner metadata. It requires a privilege interaction
    for every session, expands the narrow helper into a long-lived launcher of
    arbitrary workloads, needs a new supervisor/security proof, and cannot run
    unattended where sudo has no terminal. Those costs lose against the six
    constraints today. *(reasoned — rejected candidate 6)*
13. **Map host root or use an idmapped root mount.** An unprivileged map cannot
    include host uid 0; a privileged map that does changes the security model,
    and shifting a filesystem view would still write misleading ownership into
    artefacts rather than preserve it. *(reasoned from the recorded mapping
    constraint)*

## 10. Files a later implementation is expected to change

This list is a plan, not an edit made by this session. *(measured: no listed
file was edited in this session; reasoned for expected scope)*

- `camp/internal/config/config.go` and `config_test.go`: retain presence,
  parse the map strictly, and hold raw expressions. *(reasoned)*
- A small `camp/internal/envx/` package, or equivalently a tightly scoped file
  in `config`, for pure tokenisation, simultaneous resolution, safe rendering,
  environment merging, and explicit-PATH lookup. It has at least three real
  callers: planning, session execution, and reporting. *(reasoned — package
  boundary; reviewer may keep it in an existing package if dependencies stay
  one-way)*
- `camp/internal/plan/plan.go`, `validate.go`, and their tests: mode refusal and
  resolved namespace-workload plan data. *(reasoned)*
- `camp/internal/session/session.go` and `session_test.go`: remove
  `CAMP_SESSION`, keep declarations off the init, apply after capability drop,
  use effective `SHELL`/`PATH`, and print the ownership warning. *(reasoned)*
- `camp/internal/report/report.go` and `explain.go`: safe environment rendering,
  generic class explanation, and removal of host-global repairs. *(reasoned)*
- `camp/internal/cli/inspect.go`: honour refusals in `explain --privileged` even
  when a partial plan exists. Other CLI call sites need regression coverage so
  `down` remains unconditional. *(reasoned)*
- `camp/internal/state/state_test.go` and
  `camp/internal/privileged/privileged_test.go`: prove a sentinel resolved
  value is absent from records and helper jobs. No schema bump is expected.
  *(reasoned)*
- `camp/internal/testenv/testenv.go`: optional fixture support; do not put an
  active environment stanza in every fixture, because absence is still the
  common case and mode tests need it. *(reasoned)*
- `camp/README.md`, `camp/docs/install.md`,
  `camp/docs/how-it-works.md`, `camp/docs/getting-started.md` if it introduces
  configuration, and `camp/examples/config.yml`: grammar, OpenSSH application,
  class limits, warning, and privileged refusal. *(reasoned)*
- `camp-notes/reference/spec.md`: its rewrite must incorporate the decisions in
  §§5–8 and the stages below; this session deliberately did not edit it.
  *(reasoned — named follow-up)*
- For camp's own composition, a later owner action may add the environment
  stanza to `/home/dlaszlo/dev/camp-env/.camp/config.yml` and, if plain
  OpenSSH tools are required, reviewed launcher assets under a new tracked
  workspace tool directory. Adding a new workspace root name also requires the
  owner's explicit `camp accept`; camp must not do that migration itself.
  *(reasoned — deployment step, not core implementation)*

## 11. Implementation plan in §22 shape

Each stage ends in an acceptance check. “Unattended” means no installed camp,
namespace permission, sudo, network peer, or person at a keyboard is needed.
The checks are plans and were not run in this design-only session. *(reasoned —
plan convention)*

### Stage 1 — grammar and pure resolver

Implement the top-level map, presence bit, string-only decoding, structural
name/value checks, interpolation tokenizer, simultaneous resolution against an
explicit base map, reserved names, deterministic ordering, and safe display
tokens. Keep this layer free of filesystem and process mutation. *(reasoned —
implementation stage)*

**Accept — unattended:** table-driven tests cover absent versus `{}`, literal
values, override versus inherit, inherited empty versus absent, `$NAME`,
`${NAME}`, `$$`, no recursive expansion, sibling non-reference, control-byte
rendering, NUL, malformed/unclosed forms, non-string YAML values, empty/`=`
names, and every reserved name. A four-error map reports all four in one parse.
No test invokes a shell. *(reasoned — acceptance check)*

### Stage 2 — plan, mode boundary, and reports

Resolve a namespace plan with authoritative `CAMP_LIVE`/`PWD`; add
`environment-namespace-only` to privileged planning; render declarations with
inherited substitutions hidden; correct `explain` to honour refusals; ensure
state and helper conversions ignore workload environment. *(reasoned —
implementation stage)*

**Accept — unattended:** `plan` output is byte-stable under differently ordered
YAML maps; a sentinel inherited secret never appears in plan/explain text,
state JSON, or helper-job JSON; camp-owned paths do appear; `plan
--privileged`, `explain --privileged`, and the `up` preparation path return the
specified rule for both empty and nonempty maps; a fake sudo executable records
that it was never invoked; a recorded teardown still builds and runs from the
record after the current config gains `environment:`. *(reasoned — acceptance
check)*

### Stage 3 — post-capability workload application

Build the duplicate-free effective child environment without mutating the
init; remove `CAMP_SESSION`; select the default shell and resolve a bare argv[0]
through the effective values; attach `Cmd.Env` only after `capsx.Drop()`; print
the ownership warning once before the workload. *(reasoned — implementation
stage)*

**Accept — unattended:** pure command-construction tests use an explicit base
environment and a scratch executable to prove declared `PATH` selects it,
absolute argv does not search, a missing executable names configured `PATH`
without leaking it, effective `SHELL` is selected, the parent process map is
unchanged, output has no duplicate names, `CAMP_LIVE`/`PWD` win, and
`CAMP_SESSION` is absent. A rendering test fixes the exact warning and verifies
it goes to camp stderr only. *(reasoned — acceptance check)*

**Accept — installed binary required:** in a real one-level namespace, a custom
generator asserts that `SESSION_SENTINEL` is absent while the direct workload,
an interactive shell, and a daemonised descendant each see its declared value.
The workload reads `/proc/1/environ` and proves the sentinel is absent from
camp-as-init; `/proc/self/status` proves the workload has no capabilities. A
workspace executable reachable only through declared `PATH` is selected by
`camp run -- <bare-name>`. After exit, the parent shell still has its original
values and the live directory remains empty outside. A skip is not a pass.
*(reasoned — install-gated acceptance check, gate documented in
`remaining-checks.md`)*

### Stage 4 — OpenSSH application and class-level regressions

Replace the host-side repair in reports/docs with a complete, explicitly
composition-owned example: direct git variable, original-path preservation,
and launchers for all three OpenSSH entry points. Document that camp neither
generates nor blesses those program-specific files. Add the generic ownership
class and faithful-mode choice. *(reasoned — implementation/documentation
stage)*

**Accept — unattended:** use fake `ssh`, `scp`, and `sftp` executables plus
logging launchers in a scratch workspace. Git's local fake-remote invocation
receives the declared command; each direct entry point receives `-F`; the
launcher finds the original through a saved inherited path; no absolute
distribution binary path appears in implementation or example; and a missing
launcher either makes the original program's existing failure visible or makes
bare command lookup fail — never a reported success from camp. *(reasoned —
acceptance check)*

**Accept — installed binary plus person/network peer:** inside camp's real
composition, run `ssh`, `scp`, `sftp`, and `git ls-remote` against a known host
and record that the system-wide ownership refusal is absent and the expected
user configuration still applies. Outside in the same terminal, prove normal
command resolution is unchanged. This needs the installed binary for the
namespace and a person for credentials/host-key decisions; it needs no sudo.
*(reasoned — manual acceptance check)*

### Stage 5 — silent-member notice and compatibility measurements

Make the warning visible in both interactive and non-interactive session logs;
expand `explain` to list every class member in §8; measure, rather than assume,
the keyring route. *(reasoned — implementation/measurement stage)*

**Accept — installed binary:** archive a host-root-owned fixture from namespace
mode and show that the archive records the projected owner while the warning is
present in stderr; repeat with the equivalent environment-free configuration
in `camp up` and show ownership fidelity. This acceptance demonstrates the
limit rather than claiming to repair it. The `camp up` half needs sudo and a
real terminal under the existing privileged-mode gate. *(reasoned — acceptance
check)*

**Accept — person at keyboard:** with the desktop session unlocked, exercise
the configured credential helper/libsecret path and record whether its user
socket works. A failure becomes a named known limit or a new real caller for a
future mechanism; it does not silently expand this design. *(reasoned — manual
acceptance check)*

### Stage 6 — whole-build and documentation close

Update the named documents and the specification rewrite, then run the
repository's required quality gate. *(reasoned — implementation stage)*

**Accept — unattended except install-gated skips:**
`go build ./... && go vet ./... && gofmt -l . && go test ./...`; all pure tests
pass, and every namespace skip prints the install command. Run the full test
suite through the installed binary/profile so the namespace group passes. No
new feature check requires sudo except the explicit Stage-5 comparison, and no
test weakens a real namespace check into a mock. Source guards still prove all
writes go through `fsx` and no detach operation exists. *(reasoned — acceptance
check following current project policy)*

## 12. Attack on the design

### 12.1 The eight invariants, one by one

1. **Camp only composes and never modifies a repository.** The map is read from
   configuration and installed in memory on a child. Camp does not create the
   optional workspace launchers; a user commits them as ordinary workspace
   content. No runtime repository write is added. **Touches invariant: no.**
   *(reasoned)*
2. **The lower is never written.** Executing a launcher reads a lower-provided
   file through its existing read-only root guard. Attempted self-modification
   still receives `EROFS`. **Touches invariant: no.** *(reasoned)*
3. **The user deletes; camp checks.** The environment has no persistent runtime
   object to delete. Removing a declaration or launcher is the user's ordinary
   repository/config edit. **Touches invariant: no.** *(reasoned)*
4. **`up` may refuse; `down` may only report.** `up` refuses the mode mismatch;
   `down` ignores it for authority and consumes the old record. A regression
   test is mandatory because current drift reporting calls privileged planning
   after teardown. **Touches invariant: reinforces it.** *(reasoned from source
   inspection)*
5. **No `--force`.** There is no override flag, quiet “apply anyway”, or mode
   fallback. The recorded config decision is the only setting. **Touches
   invariant: no.** *(reasoned)*
6. **Anything unrecognised is refused.** Unknown YAML keys already fail; new
   shapes, malformed expressions, undefined references, reserved names, and
   wrong mode each have rules. Program-specific misspellings remain opaque
   data and are a known semantic limit, not something camp guesses at.
   **Touches invariant: extends it.** *(reasoned)*
7. **Outside visibility.** A child environment cannot alter its parent or an
   outside process. Privileged mode refuses, so no third machine-wide effect is
   introduced. **Touches invariant: no.** *(reasoned)*
8. **Not a security boundary.** Resolved secrets are hidden from reports but
   not from the same-uid `/proc` view or the workload itself. The docs say so;
   the feature is configuration, not secret storage. **Touches invariant: no.**
   *(reasoned)*

### 12.2 A user who never reads documentation

With no declaration, ssh still fails loudly, but the session now prints the
ownership-view warning before the first command. With a correct declaration,
git or a workspace launcher simply works and the user has nothing further to
do. With a silent artefact builder, the tool cannot detect which bytes encode
owners, but the warning is in the terminal or build log and names the faithful
mode. This is less silent, not automatic correction. *(reasoned)*

The warning may become familiar noise. Suppressing it automatically would
reopen the exact silent case; a persistent acknowledgement would create stale
state and let one person's acknowledgement hide the issue from another. This
design accepts one line per session as the cost. *(reasoned — explicit
trade-off)*

### 12.3 Another distribution or filesystem layout

The core names no `/etc/ssh`, include directory, `/usr/bin`, wrapper directory,
or system configuration file. A machine with no system-wide configuration has
no ownership failure and need not declare a repair. A composition chooses its
own tool path through `$CAMP_LIVE`; examples find original commands through the
inherited path. *(reasoned)*

A noexec filesystem can still make a workspace script unexecutable; current
`doctor` already reports inherited filesystem restrictions. The design neither
pretends to bypass `noexec` nor hard-codes an interpreter as a fallback.
*(measured: source/project record for current noexec reporting; reasoned for
launcher effect)*

### 12.4 Privileged mode

It refuses before generation, state, mount, or sudo and says there is no child
to receive the map. An empty map refuses too. `down` remains record-driven and
cannot be blocked by a later edit. There is no “best effort” export file and no
instruction to `eval` camp output in a host shell. *(reasoned)*

### 12.5 Hostile or careless configuration

Whoever edits the map can alter workload execution with `PATH`, `SHELL`,
`LD_PRELOAD`, language startup variables, or a program-specific command. That
power is explicit and user-level. It never reaches generation, mount
selection, the init's pre-drop execution, or the privileged helper. *(reasoned)*

Control characters are encoded in reports; NUL is refused; inherited secret
bytes are not rendered; state/helper formats omit the values. A malicious
value can still be interpreted by the consuming workload — for example, git
defines the meaning of `GIT_SSH_COMMAND` — because the requested behavior is
precisely to configure that workload. *(reasoned)*

### 12.6 Second run, crash, stale record, drifted inventory

The map writes no state. A normal second run resolves again from the parent
environment, so a prepended `PATH` does not accumulate across sessions started
from the same outside shell. A daemonised descendant keeps the first session
and its environment alive, and the existing inode lock refuses a second
composition. *(reasoned from the measured supervisor/lock model)*

A crash before workload exec leaves no declared environment anywhere; a crash
after exec leaves it only in surviving descendants, which also keep the
namespace alive. When they end, both disappear. Namespace mode has no state
record. A stale privileged record contains no environment and remains adequate
for teardown. *(reasoned)*

Inventory drift changes the tree and may block the existing plan, but it cannot
change expansion except through the canonical live path. Adding a new tracked
workspace launcher root is an ordinary inventory change that the user must
accept explicitly. *(reasoned)*

Starting a different camp from inside an already modified environment can
inherit the first session's `PATH` and deliberately compose it again. The same
composition is lock-refused; cross-composition nesting is not a target of this
feature and is recorded as a limit rather than “deduplicated” by guessing path
segments. *(reasoned — known edge)*

### 12.7 A typo or wrong value

Unknown YAML, wrong value type, malformed `$`, missing input name, reserved
name, and privileged use fail loudly before the workload. A configured `PATH`
that cannot find the requested direct command fails at lookup and names the
effective-path rule. *(reasoned)*

Camp cannot know that `GIT_SSH_COMAND` is a misspelling, that a literal path
names the wrong file, or that a prepended directory contains no launcher when
some later program falls through to another command. Accepting only a fixed
registry of known variables would destroy the general mechanism and still miss
values. Plan/explain expose the exact declaration safely, and application-level
acceptance proves important settings. This is a known limit, not silently
claimed validation. *(reasoned)*

### 12.8 Every class member

The owner-of-file members split three ways: adaptable programs use the map;
unadaptable or metadata-faithful programs use `camp up`; operations intrinsic
to rootless semantics (`setuid`, foreign `chown`) remain unsupported and are
said plainly. The pid and mount-visibility classes are unchanged. §8 enumerates
each known member so “general” is not allowed to mean “we stopped at ssh.”
*(reasoned)*

### 12.9 What would make this design be abandoned in six months

Abandon candidate 5 as the primary answer if either of these becomes measured:

1. two common, important programs cannot be adapted through environment or
   composition-owned command resolution and force per-program camp code; or
2. ownership-bearing artefacts are routinely built in namespace mode and the
   warning/`camp up` choice is not operationally acceptable.

Those facts would justify reopening candidate 6: a sudo-created,
identity-preserving, namespace-local run mode. They would also justify its
privileged security review and recurring interaction cost. They would not
justify host wrappers, fake metadata, or outside-tree copies, whose correctness
problems are already visible now. *(reasoned — abandonment criterion)*

## 13. Known limits accepted now

- Namespace mode still cannot report host-root-owned files as root. The design
  does not say otherwise. *(reasoned from measured constraint)*
- Plain OpenSSH tools are not out of the box in namespace mode; a composition
  supplies reviewed launchers once, or chooses `camp up`. *(reasoned)*
- The map does not correct or inspect ownership stored in an artefact. The
  warning makes the risk loud; `camp up` supplies fidelity. *(reasoned)*
- Camp validates expression structure, not whether an arbitrary consumer knows
  a variable or likes its value. *(reasoned)*
- Environment values are not secrets from same-uid processes. *(reasoned)*
- Programs may scrub or rewrite inherited environment, and shell startup files
  may override configured values after the initial exec. Camp promises the
  initial environment only. *(reasoned)*
- An explicitly environment-configured composition cannot use `camp up`
  without removing that key. This inconvenience is preferred to a setting that
  looks active and is not. *(reasoned)*

## 14. Reviewer decisions and remaining open questions

The design decisions are closed unless the reviewer rejects their trade-off:

- candidate 5 wins; 1 and 2 accompany it; no new mount kind or sudo run mode;
  *(reasoned — proposed decision)*
- undefined references refuse; interpolation is one-pass and non-shell;
  *(reasoned — proposed decision)*
- inherited resolved bytes are redacted in plan/explain; *(reasoned — proposed
  decision)*
- `CAMP_LIVE` is public, `PWD` authoritative, `CAMP_SESSION` removed, and
  `CAMP_` reserved; *(reasoned — proposed decision)*
- `environment:` in any form refuses privileged planning; *(reasoned — proposed
  decision)*
- the generic skeleton is commented and carries no active ssh policy;
  *(reasoned — proposed decision)*
- every namespace session emits the ownership warning. *(reasoned — proposed
  decision)*

Two questions remain genuinely open because this design session could not
measure them:

1. Does an external, undocumented consumer require a deprecation interval for
   `CAMP_SESSION`? Repository evidence says no; only the owner can supply
   evidence outside it. *(measured: source inspection; reasoned — open
   compatibility question)*
2. Does the desktop keyring path work inside the installed one-level namespace?
   The answer determines documentation or a future caller, not this map's
   semantics. *(reasoned — open measurement)*

Everything else that the attack broke has been moved into the design (the
post-capability boundary, effective-PATH lookup, mode refusal, safe rendering,
unconditional teardown, and runtime ownership warning) or listed explicitly in
§13. *(reasoned — self-review conclusion)*
