# Session handoff — where the work stands, 2026-08-16 evening

Written because the next session starts with no memory of this one. It
records what was done, what was decided, what is still open, and the
handful of facts about this environment that a new session would otherwise
have to rediscover.

Two documents came out of today. This one is the state of the work. The
other, `2026-08-16-handoff-ssh-inside-a-session.md`, is a design brief for
a single problem and is meant to be handed to a design session on its own.

## Where the product stands

The tool is built, tested and public.

  - **Repository:** `github.com/dlaszlo/camp`, branch `main`, 15 commits.
  - **`go build ./... && go vet ./... && gofmt -l . && go test ./...`** —
    all clean, 170 tests, **none skipped**. The install-gated tests run
    here because this session itself is inside a camp namespace whose
    AppArmor profile leaves it unconfined, so nested namespaces work.
    Whether they would run from a bare terminal was not checked.
  - **No coding work was found outstanding.** No TODO, no unimplemented
    branch, no placeholder. What §23 of the specification defers is
    deliberately not built.
  - **Testing work remains, and it needs a person.** The privileged mode
    (`camp up` / `camp down`) has never been run end to end on this
    machine: seven checks in `reference/remaining-checks.md` need sudo on
    a real terminal, and four more need a person at a keyboard.

## What was done today, and why

**The git frame correction** (`e255e45`). A participating directory need
not be a repository root. `gitwire.Open` asked `--is-inside-work-tree`,
which says true however far up the `.git` was found, so a source inside a
larger repository was taken for a root; the pathspec then anchored at the
real root, found nothing, and camp mounted an islands store with **no
islands in it and said nothing**. Open now also reads `--show-prefix` and
scopes every question into the repository's frame. Measured in a real
composition whose workspace is a subdirectory of a larger repository: the
islands are correct, and the ignored machine-local file is not among them.
The raw-listing fallback is unchanged and now says so — the note the
specification promised had no caller at all.

**A documentation correction** (`993b032`). `install.md` listed `ls-tree`,
which camp never runs, and omitted `rev-parse`, which it does. The README
presented git as an unconditional requirement. One heading was an English
idiom. `how-it-works` learned that a source may sit inside a larger
repository.

**The ssh finding** (`566782f`). See the other handoff. The commit's
*facts* are right; its *recommendation* — configure the host's git — the
owner has since rejected, and that text has to be replaced by whatever the
design settles on.

**The history was rewritten twice, and the repository recreated.**

  - Every `Co-Authored-By: Claude` trailer was removed from every commit.
    The reason the owner gave: much of what is published now is AI slop,
    and somebody who might carry the idea further should not meet that as
    a first impression. Attribution, if any, is the owner's sentence to
    write in the README, in their own words, naming the several models
    that worked on it — **do not add it, and do not offer to**.
  - Commit bodies were shortened from 175–580 words to 45–163. Subjects
    were kept: they were already one line, imperative, about what changed
    for the reader. What went out of the bodies is what the code carries
    anyway — stage numbers, file lists, test names. What stayed is the
    *why* and what was measured.
  - Local traces were swept: `refs/original`, the reflog, the stale
    remote-tracking ref, then `gc --prune=now`. Verified by hash that no
    old commit is reachable.
  - A force push does **not** remove what GitHub already has, so the
    repository was deleted and recreated, and the rewritten history pushed
    into an empty one.
  - `~/.claude/settings.json` now sets `attribution.commit` and
    `attribution.pr` to empty strings, so no trailer is added again.

## What is open

### The original task, not started

**Rewrite `reference/spec.md` into the present tense.** This is what the
day was supposed to be about and it has not been touched. The instructions
are in `.notes/README.md` and still stand: describe what camp *is*, keep
every "why", remove the process (finished build stages, dated decisions,
"owner decision" tags, deferred items that are done, and the whole section
about migrating one specific pair of repositories that has nothing to do
with camp). `reference/constraints.md` is mostly measurement and should
survive nearly intact, but wants the same eye.

Two things from today have to be folded in while doing it:

  - the git frame correction above — the specification still implies a
    participating directory is a repository root;
  - whatever the ssh design settles, because it changes the configuration
    language.

And the build measurements in `2026-08-16-build-measurements.md` correct
the specification in two places (the capability set an overlay mount needs,
and the locked flags); that fold-in was also part of the original task.

### Publishing the environment

The owner decided camp-env becomes a public repository. The URL exists:
`git@github.com:dlaszlo/camp-env.git`. **Nothing has been pushed and
camp-env is not yet a git repository.**

What is already written, in `/home/dlaszlo/dev/camp-env/`:

  - `README.md` — describes the environment. **It is out of date**: it
    still says camp-workspace and camp-notes are submodules.
  - `.gitignore` — ignores `camp-live/`, and camp's `work/`, `storage/`,
    `reports/`.

What is undecided. The owner wants **one** repository, with the workspace
and the design record inside it rather than as submodules, and camp as the
only submodule. That is now possible — before today's fix it would have
silently produced zero islands. Two things still have to be settled:

  - **How to fold them in.** `git subtree`, keeping both repositories'
    two commits each, or a plain copy that drops them. The owner has not
    chosen.
  - **The design record's boundary.** With `camp-notes` as its own
    repository, `git` run in `.notes` from inside a session stops there
    and commits land where they belong. Folded into camp-env, it walks up
    to the composed root and finds the **code repository** — so the notes
    would have to be committed from outside the composition. The exclude
    still covers `/.notes`, so nothing leaks accidentally, but the
    boundary that makes it impossible becomes detection after the fact.

**A constraint on doing it:** the workspace cannot be restructured from
inside a session. Measured: `touch /home/dlaszlo/dev/camp-env/camp-workspace/.probe`
returns `Read-only file system`. That is the design working. The
camp-workspace half of the fold has to happen from a terminal outside, and
`camp accept` has to be re-run afterwards, because the workspace's root
entries change.

### The ssh problem

Handed off in the other document. It is a design question, not a coding
one, and it changes the configuration language.

### Two smaller findings, not acted on

  - **`camp doctor` run from inside a composition** reports "this
    composition would not start", because the live directory is not empty
    — it is mounted. True, and misleading in that context. Worth deciding
    whether doctor should notice it is inside.
  - **`camp explain` and the docs** currently recommend the host-side ssh
    repair the owner rejected. Must be replaced.

## Facts about working in this environment

  - **You are inside a camp composition.** `camp-live` is the composed
    tree; the code repository is `camp`, writable; `.notes` is this
    repository, writable; `CLAUDE.md` and the other workspace names are
    read-only by design and editing them fails with `EROFS`.
  - **`git push` over ssh needs `GIT_SSH_COMMAND='ssh -F ~/.ssh/config'`**
    until the design lands. Without it, ssh refuses to start. This is the
    subject of the other handoff.
  - **The scratch root for tests** is not `/tmp` by default; it is
    `~/overlayfs/.camp-tests`, overridable with `CAMP_TEST_ROOT`.
  - **Bash working directory persists between tool calls.** A `cd` in one
    command changes where the next one runs; absolute paths avoid an hour
    of confusion.
  - **Run the full check before saying anything is finished:**
    `go build ./... && go vet ./... && gofmt -l . && go test ./...`

## The working style the owner asked for

Recorded because it shaped every commit today and should shape the next
ones.

  - **Commit subjects**: one line, imperative, saying what changed for the
    reader. **Bodies**: short, and only the *why* and what was measured —
    not what the code already says. Two to six lines is the shape now.
  - **Plain language in the documents too**, not only in conversation: no
    idiom standing where a literal statement belongs, no shorthand without
    its half-sentence, precision counted as part of plainness.
  - **The code is the truth.** Where a document and the code disagree, the
    document is wrong — verify, do not assume. Today that rule caught
    `ls-tree` in the install guide and a dead function the specification
    had promised would be called.
  - **Do not wire the host.** camp is something you are inside. A solution
    that installs into `~/.local/bin`, a shell startup file or a global
    git configuration is the wrong shape, however well it works.
