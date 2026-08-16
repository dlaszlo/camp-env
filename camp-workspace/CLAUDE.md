# Working on camp

`camp` composes several git repositories into one working directory using
OverlayFS and bind mounts, without any of them learning about the others.
Writes land in the code repository or in machine-local storage — never in
the workspace, and never silently in the wrong place.

You are working inside a composition camp built. **Read
[ENVIRONMENT.md](ENVIRONMENT.md) before your first write** — it says what
a composed tree is, which parts of it refuse writes and why, and what to
do when one does. `camp explain` answers the same questions for the
composition that is actually running.

What it means here:

| where | what it is |
|---|---|
| everything not listed below | the **code repository** — the product. Every ordinary write lands here. |
| `.notes/` | the **design record**, its own repository, writable. The specification lives at `.notes/reference/spec.md`. |
| `.claude/` | machine-local storage with the workspace's tracked entries standing read-only in it |
| `CLAUDE.md`, and the other workspace names | read-only. Editing them here fails with `EROFS`, by design. |

**Read `.notes/reference/spec.md` before changing behaviour.** It is the
single source the implementation was built from, and it records why each
decision is what it is. `constraints.md` beside it records what the
kernel, git and the measuring instruments actually do — an argument that
contradicts something there is wrong rather than merely different.

## How to write code here

Write Go the way a senior Go developer writes it, and prefer the boring
answer.

**KISS.** The simplest thing that is honest. This tool refuses rather
than guesses, so a clever fallback is usually a bug: when camp does not
recognise something, it says so and stops.

**YAGNI.** No abstraction without a second caller that exists today. The
package has no interfaces at all, because none was needed; do not add one
speculatively. If you export something, something has to use it.

**DRY, but not at the cost of clarity.** Two similar validations that
refuse for different reasons stay two functions with two messages. Shared
*mechanism* gets factored; shared *wording* usually should not be.

**SOLID where it earns its keep.** One package, one responsibility — the
plan decides, `mountx` acts, `verify` checks, `report` renders, and
nothing that decides anything reaches for a terminal. The dependency rule
that matters most here is the write door, below.

**Clean code.** Small functions, names that say what the thing is, no
comment that repeats the code. Comments here carry *reasons* — why this
order, why this refusal, what was measured — because that is the part a
reader cannot recover from the code. Keep that density; it is deliberate.

## Rules you can break without noticing

Each of these has a test that fails the build. Do not work around the
test; the test is the rule.

1. **Every filesystem write goes through `internal/fsx`.** An `fsx.Area`
   cannot be constructed from a repository path, which is what makes
   "camp never modifies a repository" a property of the source rather
   than a promise. `internal/fsx/writesites_test.go` scans for writes
   anywhere else.

2. **No lazy unmount, ever.** `MNT_DETACH`, `umount -l` and any
   detach flag are absent from the source and stay absent. A detached
   mount leaves the kernel's table while it is still alive and still
   being written through. `internal/mountx/source_test.go` enforces it.

3. **No `--force`, anywhere.** The escape hatch is configuration — the
   same decision, recorded and diffable.

4. **Never trust a mount call; inspect the result.**
   `MS_BIND|MS_RDONLY` in one `mount(2)` silently ignores the read-only
   flag. `statvfs` is the authority, mountinfo is the cross-check, and a
   covered mount stays listed while being reachable by nothing.

5. **Compare bytes, not locale.** Every name comparison — the gate, the
   inventory, the exclude — sorts by bytes, or two of them will quietly
   describe different sets.

6. **Follow no symlink.** Every path operand is resolved
   descriptor-relative with `openat2(RESOLVE_NO_SYMLINKS |
   RESOLVE_BENEATH)`. A component the user owns can be swapped between a
   check and a mount, and refusing the whole class is the only check
   without a race in it.

7. **Ask `/proc`, never a program's output.** Command output is
   translated; anything parsed runs under `LC_ALL=C`, and state comes
   from `/proc`.

## Messages are part of the product

Every refusal and report speaks to somebody who has not read the
specification: name the path, say what is true on each side, say which
side matters, give the exact command that repairs it, and say whose move
it is — the user acts, camp checks. "Overlap detected" is not an
acceptable message. Refusals go through `internal/refusal`, which carries
a stable rule identifier for tests and the prose for people.

## Tests

`internal/testenv` builds real directory trees with real git
repositories; the checks that read git have to meet git and not a
directory that looks like one. The scratch root deliberately avoids
`t.TempDir()` — that lands in `/tmp`, and some tests need a filesystem
whose locked flags a namespace can replicate. `CAMP_TEST_ROOT` overrides
it.

Tests that build a real composition need permission to create a user
namespace, which is granted by AppArmor profile to one installed binary
path. They skip from a checkout with the install commands in the message,
and pass when run through an installed camp. Do not weaken them into
passing without a namespace: an unmeasured guarantee is not one.

Run `go build ./... && go vet ./... && gofmt -l . && go test ./...`
before saying anything is finished.

## Commits

One commit per coherent change, message in the imperative, first line
saying what changed for the reader rather than which file moved. The body
says *why*, and what was measured if anything was. Look at `git log` —
the existing messages are the standard.

Nothing internal goes into the product repository: no design notes, no
review material, no personal paths, no references to whatever else is
running on this machine. That material belongs here or in `.notes`.
