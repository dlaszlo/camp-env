# Review of `docs/spec.md`

**Verdict:** No. *(reasoned)* A fresh session cannot yet build the right
system from `docs/spec.md` alone. The central post-migration shape is coherent,
but the file currently specifies two ways to violate the one-upper invariant,
an impossible pair of privileged-mode visibility guarantees, a machine-visible
partly assembled tree, a `down` check that does not detect the leak it promises
to detect, and an islands lifecycle that rejects its own second run. *(read,
measured, reasoned as identified below)* Several implementation contracts are
also absent: the ordered configuration model, generator invocation and wire
formats, namespace-supervisor protocol, durable privileged state, and exact
path model. *(read)* Those are not matters of pleasant organisation; different
reasonable implementations produce observably different or unsafe systems.
*(reasoned)*

The tags below have the meanings defined by the spec. The real repositories
were inspected read-only with `find`, `git ls-files`, `git check-ignore`, and
file reads; `git status` was not run in either. *(measured)* The installed
`ply` was used only as the AppArmor-authorised namespace entry vehicle for one
scratch ext4 experiment; all mounts measured in that experiment were direct
kernel mounts created by the test script, not behavior inferred from `ply`.
*(measured)* In every item, the provenance-tagged issue/state paragraph is the
“how known”; the two challenged measured claims also include their observed
output. *(read)*

## WRONG — an implementation following the text builds the wrong system

1. **The one-upper guard has two bypasses — §3.5, §4/C8, §13, §14
   (`--allow-detach`).**

   **Issue.** The namespace lock lives at
   `$ENV/.camp/lock.<upper-hash>`, so two otherwise-valid configurations in
   different environment directories that name the same upper lock different
   inodes; §6 and §17 contain no rule that forbids that configuration. *(read,
   reasoned)* Namespace-to-namespace detection then has neither a shared flock
   nor visible mountinfo, so both overlays may start. *(reasoned)* Separately,
   the privileged steady-state guard is only a mountinfo scan, while
   `--allow-detach` deliberately removes a busy mount from that table before
   the mount is dead. *(read)*

   **How known.** `umount2(2)` says `MNT_DETACH` immediately disconnects the
   filesystem from the mount table and performs the real unmount only when it
   ceases to be busy. *(read)* In a scratch composition on ext4, after a shell
   changed cwd into an overlay, the direct-kernel experiment produced:
   *(measured)*

   ```text
   detached target absent from mountinfo
   detached cwd remains writable
   second overlay on the same upper mounted and wrote successfully
   ```

   This is the exact C8 state the lock exists to prevent. *(measured,
   reasoned)* This finding explicitly reopens the existing
   `--allow-detach` choice because measured evidence contradicts the singleton
   guarantee; it does not reopen the kernel-mount backend or the no-`--force`
   decision. *(reasoned)*

   **Fix.** Put the flock in one machine-wide lock namespace (or another
   location proved common to every identity authorised to use the upper), not
   under `$ENV`, and key it by a stable identity of the upper such as
   `(st_dev, st_ino)` plus a recorded canonical path; namespace and sudo paths
   must open the same inode. *(reasoned)* Specify safe creation and permissions
   for that shared lock. *(reasoned)* Remove lazy detach for a writable overlay,
   or keep a guardian process holding that same flock until every detached
   reference has actually died; a state file or mountinfo tombstone alone
   cannot prove that. *(reasoned)*

2. **The privileged mount sequence cannot satisfy its own visibility
   invariants — §3.2, §3.7, §7.4–§7.8, §14.**

   **Issue.** In privileged mode, binding the workspace read-only onto its own
   path is machine-wide, so an editor or shell outside `live` sees the
   workspace become read-only. *(reasoned)* That directly contradicts §3.7's
   promise that nothing outside the composition changes. *(read, reasoned)* No
   implementation can simultaneously make one path read-only for every
   “inside” process and leave that same path unchanged for every “outside”
   process when all processes share the privileged mount namespace.
   *(reasoned)*

   The same sequence mounts the writable overlay at `live` before the code
   `.git` bind and before the derived read-only protections. *(read)* During
   that window, an already-running outside process can write a lower-provided
   path and cause copy-up, or invoke Git against the temporarily merged
   `.git`; privileged mode exists precisely for already-running outside
   processes. *(reasoned)* Calling staging “optional hardening” therefore
   makes a correctness race optional. *(reasoned)*

   **Fix.** For privileged mode, use a camp-owned read-only bind alias as the
   overlay lower and as the source of derived protections; do not self-bind
   the raw workspace path. *(reasoned)* This preserves §3.7 and all protection
   through `live`, but it necessarily weakens §3.2's absolute-path promise;
   that settled wording must be narrowed explicitly because the two promises
   cannot both hold globally. *(reasoned)* Assemble and verify the complete
   mount tree at a mode-0700 staging path, move it to an empty `live`
   atomically, then repeat path-based verification at `live`; make this
   mandatory in privileged mode. *(reasoned)*

3. **A user-level generator can steer root-level mounts — §7.2–§7.4,
   §15, §17, §19.**

   **Issue.** Static validation occurs before generation, but concrete island
   mounts do not exist until generation. *(read)* §19 validates generated
   island output only as line-shaped and newline-free. *(read)* It does not
   reject `/`, `..`, `.`, duplicate entries, special files, symlink
   components, a declared type that disagrees with `lstat`, a source outside
   the declared island source, or a target outside the islands store.
   *(read)* A configured step running as the user can therefore emit a path
   that the privileged half later resolves and mounts as root; merely running
   the step itself without privilege does not close the privilege boundary.
   *(reasoned)* The same weak validation allows an output `exclude` to omit or
   alter the repository's existing exclude bytes despite §10 requiring them
   unchanged. *(read, reasoned)*

   **How known.** The only output checks named in §19 are marker presence,
   line shape, and no newline in a name, while §7 orders full validation before
   the output exists. *(read)* Path joining a permitted base with an entry
   containing enough `..` components escapes that base unless a later
   containment check rejects it; no such post-generation check is specified.
   *(reasoned)*

   **Fix.** Split validation into pre-generation validation of configured
   inputs and post-generation validation of the final concrete plan.
   *(reasoned)* Treat generator output as hostile data: require exactly one
   immediate entry component, reject `.`/`..`/separators and unsupported file
   types, use `lstat`/descriptor-relative containment with no symlink
   traversal, re-run equal/nested target and tracked-code checks, and check
   every water collision before creating a mountpoint. *(reasoned)* Verify the
   complete exclude bytes against the §10 derivation and preserved original
   bytes, not just a marker prefix. *(reasoned)*

4. **The specified `down` checks do not detect either residual window they
   claim to detect — §10, §16, §18, §20.**

   **Issue.** `git add -f` puts a reserved path in the code repository's
   index; it does not create that path in the raw code working directory.
   *(read, measured)* Consequently the gate sees no raw upper/lower overlap,
   and §16's scan of *untracked* code paths does not see the path either.
   *(reasoned, measured)* Thus §20's statement that the force-add leak is
   detected at `down` is false for the report actually specified. *(read,
   measured)* Also, a new lower root entry with no upper counterpart is not a
   gate overlap, while §16 does not re-run the inventory comparison; that
   contradicts §10's promise that `down` names the new entry the same day.
   *(read, reasoned)*

   **How known.** In a pure scratch Git repository, an indexed
   `CLAUDE.md` with no working-tree path was constructed with
   `git update-index --add --cacheinfo`; this is the index/worktree state left
   after staging through the live view and then exposing the raw code tree.
   *(measured, reasoned)* It gave: *(measured)*

   ```text
   $ git ls-files --others --exclude-standard
   # no output
   $ git ls-files --stage CLAUDE.md
   100644 e9e5f7ce7bbda2d038b9cb8ae43a005492fd80ac 0	CLAUDE.md
   $ git diff --cached --name-status -- CLAUDE.md
   A	CLAUDE.md
   ```

   **Fix.** At pre-down, compare the current lower-root enumeration with the
   accepted inventory and the snapshot used at `up`, reporting additions and
   type changes without refusing teardown. *(reasoned)* In Git's default
   reporter, inspect index paths as well as raw untracked paths, and report any
   indexed path in the reserved lower-root/mount-target set that was not an
   allowed code path at `up`; `git ls-files --stage` is sufficient and is a
   read-only Git operation. *(read, reasoned)* Keep the raw untracked scan for
   copy-up residue. *(reasoned)*

5. **`mount_islands` rejects its own persistent attachment points on the
   next run — §7.9, §11, §12.**

   **Issue.** An empty store has no mount targets for its islands, so C10
   requires camp to create same-type attachment points in the store; §11 names
   the file placeholder explicitly and implies the corresponding directories.
   *(read, reasoned)* Storage persists and camp never removes it. *(read)* On
   the next `up`, those camp-created names are already in the water, but the
   collision rule says camp refuses whenever water already holds a name an
   island would cover. *(read)* The steady-state `.claude/settings.json` file
   island therefore makes the literal algorithm fail on its second session.
   *(reasoned)* The spec has no per-placeholder provenance with which an
   implementation could distinguish camp's attachment point from a user's
   colliding local file. *(read)*

   **Fix.** Specify an atomic, persistent scaffold manifest outside the
   exposed store subtree, recording every camp-created attachment point and
   its type. *(reasoned)* Check collisions before creation; on later runs,
   accept only a recorded, unchanged empty scaffold, and refuse an unrecorded
   or modified entry. *(reasoned)* When an island disappears, remove only a
   still-empty, manifest-owned scaffold or report it if it changed; this is
   deletion of camp's own object, not user work. *(reasoned)* Define the same
   lifecycle for directory and file islands and crash-test every transition.
   *(reasoned)*

6. **The `/tmp` acceptance test requires the locked-flags bug to remain —
   §4/C34, §14, §17, §22 step 4.**

   **Issue.** C34 and §14 require the read-only remount to preserve locked
   `nosuid`, `nodev`, `noexec`, and atime flags; the measured failure on this
   machine was caused by omitting them. *(read)* Yet §22 says acceptance *with
   the locked-flags fix* requires a `/tmp` environment to fail and ext4 to
   pass. *(read)* If all locked source flags are replicated and no other named
   incompatibility exists, that remount is expected to succeed, so the stated
   acceptance result tests the old bug rather than the fix. *(reasoned)*

   **Fix.** Make a preserved-flags case pass and a deliberately
   missing-flags case reproduce `EPERM`; if `/tmp` must still be refused for a
   different OverlayFS constraint, name that constraint, detection algorithm,
   and error separately. *(reasoned)* `doctor` may still report the inherited
   restrictions without calling a supported filesystem unusable. *(reasoned)*

## UNDERSPECIFIED — an implementer must stop and ask

1. **There is no representable total mount order or single final-plan
   algorithm — §6, §7, §15, §17.**

   **Issue.** `mount_ro`, `mount_rw`, and `mount_islands` are three separate
   YAML sequences, while the rule says all declared mounts run in “file
   order.” *(read)* YAML mappings do not provide a semantic interleaving of
   entries from three value sequences, and the example places `mount_rw`
   before `mount_islands` while §7 orders the islands mount before the registry
   rw mount. *(read, reasoned)* The code `.git` mount is simultaneously a
   declaration in §6 and an unconditional special step in §7; the spec does
   not say whether omission, duplication, a wrong source, or a linked-worktree
   `.git` file is refused or overridden. *(read)*

   Pre-mount validation also says targets already exist even though `live` is
   still empty, and deep targets may be supplied by an earlier declared mount
   or by a store placeholder. *(read)* No algorithm says how to resolve the
   would-be visible type through upper, lower, derived protections, stores,
   and preceding mounts. *(read)*

   **Fix.** Define one ordered mount AST—preferably one YAML sequence with an
   explicit kind—or a normative cross-kind order that can express every legal
   nesting. *(reasoned)* Define which Git mounts are reserved and derived; the
   safest rule is to derive `.git` and the exclude internally and forbid users
   from redeclaring them. *(reasoned)* Specify a sequential virtual-tree
   resolver used by `plan` and preflight, then validate the final concrete plan
   again after generation. *(reasoned)*

2. **Configured steps have phases but no configuration or process contract —
   §6, §14, §19.**

   **Issue.** §6 contains no key for configured steps. *(read)* §19 does not
   define whether a command is an argv vector or shell string, how several
   steps at one phase are ordered, cwd, environment inheritance and
   sanitisation, stdin/stdout/stderr, timeout/cancellation, exit-status
   handling, whether later phases run after partial failure, or whether the
   built-in is replaced or composed with configured steps. *(read)* A fresh
   implementation cannot make `plan` print the “exact commands” required by
   §16 without these choices. *(reasoned)*

   The privilege story is internally ambiguous too: §14 documents
   `sudo camp up`, under which the process is already root, while §19 says
   `prepare` runs before sudo is exercised and therefore has nothing to drop.
   *(read)* `camp down` is shown without sudo even though unmounting needs the
   same capability. *(read, reasoned)*

   **Fix.** Add the exact YAML schema and process contract, including failure
   semantics for every phase. *(reasoned)* Choose one elevation model. A
   coherent model is an unprivileged front end that validates and runs
   `prepare`, then invokes one narrow privileged helper for the validated
   mount plan; all later user steps run in a forked child with uid, gid,
   supplementary groups, environment, cwd, and capabilities explicitly reset.
   *(reasoned)* If literal `sudo camp up` remains, specify the equivalent
   privilege-drop-and-return protocol and how the invoking user's identity is
   authenticated. *(reasoned)*

3. **The path language is not defined tightly enough for a privileged
   pathname engine — §1, §6, §9, §12, §13, §15, §17.**

   **Issue.** The file does not state whether repository paths are relative to
   `$ENV`, the config directory, or cwd; whether absolute paths and `..` are
   accepted; what repository names may contain even though `/` separates name
   from subpath; whether `allow_overlap` contains root names or arbitrary
   paths; or whether `merged` may be absolute. *(read)* It does not define
   normalization before equal/nested comparisons, rejection of empty or `.`
   targets, treatment of symlinks in intermediate target/source components,
   duplicate repository realpaths, or unsupported root types such as sockets,
   FIFOs, and devices. *(read)* “Resolves under” and “is not a symlink” admit
   different implementations for all of these. *(reasoned)*

   **Fix.** Add a normative path table: base directory for every field,
   allowed lexical grammar, `openat2`/`realpath` policy, containment boundary,
   whether each comparison is lexical or inode-based, supported file types,
   and equality after normalization. *(reasoned)* Reserve `/`, empty
   components, `.`, and `..` in targets and generated entries; either reject
   all symlink components for mount operands or define a descriptor-based
   no-follow resolution rule. *(reasoned)*

4. **The inventory and generator list formats are not round-trippable —
   §18, §19.**

   **Issue.** Inventory lines use spaces and a literal ` -> ` delimiter while
   names and link targets may legally contain both; only newline in a *name*
   is refused, so a newline in a symlink target also breaks the record.
   *(read, reasoned)* Generator lists use tab delimiters while tabs in names
   remain legal, and the formats of `mount-targets.list` and
   `allow-overlap.list` are not given at all. *(read)* Linux filenames may
   also contain non-UTF-8 bytes, while the spec requires byte sorting but does
   not say whether such names are supported, escaped, or refused. *(read,
   reasoned)* Two valid filesystem states can therefore serialize to the same
   text or fail to parse consistently. *(reasoned)*

   §10 likewise does not define the byte separator between an existing exclude
   file that lacks a final newline and the generated marker; direct
   concatenation turns the marker into part of the last pattern. *(read,
   reasoned)*

   **Fix.** Choose one injective byte-level encoding for every record—NUL
   framing with explicit fields, or a specified escaping/base64 scheme—and
   define sorting over decoded raw name bytes. *(reasoned)* If human-reviewable
   text is mandatory, specify reversible C-style escaping and reject invalid
   encodings explicitly; cover names and symlink targets with spaces, tabs,
   arrows, backslashes, control bytes, and invalid UTF-8 in golden tests.
   *(reasoned)* Define exclude assembly byte-for-byte, including the empty-file
   and missing-final-newline cases, and verify the resulting full payload.
   *(reasoned)*

5. **The namespace supervisor's observable lifecycle is not specified —
   §7.11, §13, §14, §19.**

   **Issue.** Holding the flock in “camp as pid 1,” reparenting daemonised
   tmux, and reaping children requires a PID namespace and a resident
   supervisor that forks the workload; a user+mount namespace followed by
   `exec` is not that process topology. *(read, reasoned)* The spec does not
   say when the outer `camp run` waits or returns, how the detached supervisor
   reports successful setup, which exit code wins when the requested command
   exits while descendants remain, how signals and terminal ownership are
   forwarded, whether a namespace-local procfs is mounted, or when the
   supervisor drops all capability sets. *(read)* The claim that
   `camp run -- tmux new-session -d` returns while camp remains pid 1 requires
   an outer/inner handshake that is not described. *(reasoned)* Route B also
   lacks a selection/fallback rule and exact uid/gid maps. *(read)*

   **Fix.** Specify the process tree and state machine from outer launcher to
   PID-namespace init, including setup/error pipe, command child, reaping loop,
   signal policy, capability drop, procfs choice, detachment handshake, exit
   statuses, and route selection. *(reasoned)* Add the tmux-daemon case and a
   normal foreground command as acceptance traces, not only end-state checks.
   *(reasoned)*

6. **Privileged state, `list`, and `forget` are delegated to an unknown old
   behavior — §1, §7.11, §14, §15, §16.**

   **Issue.** “JSON under the state directory” supplies no filename/ID rule,
   schema, versioning, atomicity, ownership, permissions, or corrupt-record
   behavior. *(read)* “`list`/`forget` stay as-is” cannot be implemented from
   this file alone and does not say whether `forget` may discard an active or
   partly-up record. *(read, reasoned)* Under literal sudo, `$XDG_STATE_HOME`
   and `~` may name root's directories rather than the invoking user's, so
   `up`, unprivileged `list`, and `down` need an explicit common resolution
   rule. *(reasoned)*

   The current source records the actual mount list specifically so `down`
   does not derive teardown from a config edited while the composition is up;
   that valuable rule is absent from the spec. *(read,
   `internal/state/state.go` and `internal/composition/composition.go`)*

   **Fix.** Specify a versioned record containing at least invoking identity,
   canonical config/live/upper/workspace paths, config and inventory digests,
   the complete concrete mount plan and mount identities, camp-created paths,
   phase, tool version, and timestamps. *(reasoned)* Define atomic replacement,
   directory ownership/mode, lookup by ID/config/live, corrupt-record
   reporting, list ordering, and a refusal to forget while any recorded mount,
   lock guardian, or partial residue remains. *(reasoned)* `down`, `status`, and
   `explain` must use the recorded plan as the mounted truth and report config
   drift separately. *(reasoned)*

## GAP — a reachable state has no specified handling

1. **There is no write-ahead recovery protocol for a privileged crash —
   §7, §12–§16.**

   **State.** A privileged `up` can die after any mount but before §7.11 writes
   the up record. *(reasoned)* The next `up` refuses on residue, but if the
   config was edited, moved, deleted, or now derives a different plan, neither
   `down` nor `list` has a specified authoritative list of what camp mounted;
   the workspace self-bind is outside the live prefix as well. *(read,
   reasoned)* A crash during reverse rollback or `down` has the same ambiguity.
   *(reasoned)*

   **Fix.** Before the first privileged mount, atomically and durably write a
   `mounting` record containing the full validated plan; after each transition
   it may add actual mount IDs, but recovery must be possible from the original
   plan plus mountinfo identity checks. *(reasoned)* Change phase to `up` only
   after post-move verification; retain `partial` on failed rollback/down and
   remove or mark `down` only after absence is verified. *(reasoned)* Inject a
   kill at every mount, record-write, move, and unmount boundary and require
   `status`/`down` to converge without using the current config. *(reasoned)*

2. **Namespace sessions have no equivalent of the promised end-of-session
   reports — §10, §14, §16, §19, §20.**

   **State.** Namespace mode has no `down` and no state record, but the
   worktree-repair warning, inventory drift, and force-stage detection are all
   specified only under `down`. *(read)* The resident init is the only process
   positioned to run `pre-down`, yet §14 says teardown is simply kernel exit
   and §19 does not assign phase behavior by mode. *(read)* `post-down` cannot
   run inside a namespace whose destruction requires the last inside process
   to exit; a detached tmux session may also end long after the original
   terminal and stderr have gone away. *(reasoned)* Therefore §20's detection
   promise has no delivery path in the primary mode. *(reasoned)*

   **Fix.** Define automatic namespace shutdown: when only init remains, run
   the same pre-down inventory/index/worktree report as privileged `down`, then
   explicitly unmount or hand a result to the outside supervisor before exit;
   run post-down outside after namespace death. *(reasoned)* Specify where a
   detached session's report persists and how the next `camp` command surfaces
   it without turning it into a stale authority; a small generated report is
   compatible with “no privileged state record.” *(reasoned)*

3. **The disposable-work and live-mountpoint lifecycles have no executable
   ownership rule — §1, §7, §12, §15.**

   **State.** §7 takes the upper lock before generation, while §12 says the
   next namespace `up` sweeps work entries “whose lock is free”; the current
   entry's lock is then held by the sweeper itself. *(read)* Other
   `work/<live-hash>` names do not contain the upper identity needed to locate
   and test their lock, and unlike storage no marker is specified. *(read,
   reasoned)* An implementation cannot know which entries are stale or safely
   attributable. *(reasoned)*

   The live hash requires `realpath(live)`, but the spec never says whether
   `live` must pre-exist or camp creates it. *(read)* Nor does preflight refuse
   a nonempty live mountpoint: it checks mount residue, then the overlay can
   hide ordinary files until `down` reports them. *(read, reasoned)* That loses
   the evidence during the session and lets an implementation mount over user
   content. *(reasoned)*

   **Fix.** Give work directories a marker containing live, config, upper
   identity, and lock key; define that the holder may clean its own current
   hash after residue checks, while unrelated hashes are cleaned only after a
   successful nonblocking acquisition of their recorded global lock.
   *(reasoned)* Require `live` to be a real, non-symlink, empty directory before
   any mount, or define atomic creation, ownership, persistence, and hash timing
   if camp owns it; refuse nonempty content at `up` as well as reporting it at
   `down`. *(reasoned)*

4. **Path validation is separated from pathname use by an unchecked race —
   §7.2–§7.4, §15, §17, §19.**

   **State.** Repositories, storage, and config are writable by the invoking
   user while a privileged helper validates, generates, and mounts. *(read)* A
   source or target component can be renamed or replaced with a symlink after
   validation; a later `mount(2)` follows the pathname, and the post-mount
   identity check can still pass if it compares against the now-replaced
   source pathname. *(read C12, reasoned)* This is not about sandboxing the
   composed workload; it is about preventing a sudo-authorised camp operation
   from becoming a general root mount primitive. *(reasoned)*

   **Fix.** Resolve operands once with descriptor-relative, beneath/no-symlink
   semantics and carry stable handles into the privileged operation using the
   new mount API (`open_tree`/`move_mount`) where available; otherwise re-open
   and verify every component immediately in the helper and fail closed on any
   identity change. *(reasoned)* Keep the global upper lock across validation,
   generation, mount, move, and final verification. *(reasoned)*

5. **§23 is not a complete register of load-bearing unknowns — §14, §15,
   §22, §23.**

   **Missing.** The privileged path was explicitly unmeasured in the design
   review, but §23 lists only its `trusted.*` forensics, not a full privileged
   composition with private propagation, staging move, invoking-user
   ownership, state recovery, and outside-observer checks. *(read)* It also
   omits the PID-namespace/init/procfs/outer-handshake behavior on which the
   lock and tmux lifecycle depend, repeated-session island scaffold behavior,
   kill-point recovery, cross-environment upper locking, and source/target
   replacement races. *(read, reasoned)* The lazy-detach interaction was also
   absent; it is no longer merely unknown because the experiment above failed
   the invariant. *(measured)*

   **Fix.** Add these as named build-blocking measurements with expected
   outcomes, and move an item out of §23 only when the normative mechanism and
   acceptance test have both been written. *(reasoned)* In particular, “first
   privileged `up`” must validate the entire privileged lifecycle, not only
   xattr names. *(reasoned)*

## One-line non-findings

- The uncommitted `ply` → `camp` rename is expected and says nothing about the
  spec's implementability. *(read)*
- The present real-repository overlaps, root caches, lanes, and in-workspace
  `.registry` are pre-migration facts explicitly removed by §5, so they are not
  findings against the steady state. *(read, measured)*
- `.git` does not merge once the required code `.git` bind is in place, and
  gate exemption for a completely covered target is sufficient; the finding
  above is that the language does not make that bind structurally mandatory.
  *(read, reasoned)*
- The separate registry repository is a deliberate invariant-2 precondition,
  not optional hygiene under this spec. *(read)*
- Coarse, leading-slash exclude lines are the settled answer and avoid the
  measured deep-name bug; this review does not reopen file-level exclusion.
  *(read)*
- The absence of a camp pre-commit hook and the fact that `git add -f` cannot
  be prevented are settled; only the claimed but missing `down` detection is a
  finding. *(read)*
- Supplementary groups remaining effective for host permission checks closes
  the Docker objection to namespace mode; it is not re-raised here.
  *(measured in the source review, read here)*
- Worktrees created through `live` needing the printed repair command, the
  three-month prune window, and storage never being deleted are all carried
  into the spec. *(read)*
- One lower today, tool-merged read-only lowers later, the `mount_islands`
  name, the `camp` name, and the kernel-mount backend are settled and are not
  findings. *(read)*
- A directory island is explicitly one live directory bind, so nested
  untracked files under that selected directory are visible; the currently
  measured `.claude/agents/.pytest_cache` is pre-migration residue, and hiding
  nested junk would reopen the settled one-bind granularity rather than fill
  an omitted implementation step. *(read, measured, reasoned)*
- Identity route A, `trusted.overlay.*`, `--no-optional-locks`, GUI handoff,
  and tmux attach are already honestly deferred in §23; their mere
  unmeasured status is not a separate finding. *(read)*
