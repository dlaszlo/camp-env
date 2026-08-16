# camp-env

The working directory camp is developed in: one configuration that
composes several repositories into a single tree, and camp itself as a
submodule.

```
git clone --recurse-submodules https://github.com/dlaszlo/camp-env
cd camp-env
(cd camp && go build -o camp ./cmd/camp && sudo install -m 755 camp /usr/local/bin/camp)
```

Read `.camp/config.yml` before anything else. It is commented, it is the
only file written by hand, and it is the whole of what camp needs to know.

## What is here, and what is not

```
camp-env/
├── .camp/
│   ├── config.yml        the composition — the only file written by hand
│   ├── inventory         the accepted snapshot of both roots
│   ├── work/             disposable, machine-local, not in git
│   └── storage/          persistent, machine-local, not in git
├── camp/                 the product → github.com/dlaszlo/camp (submodule)
├── camp-workspace/       the development environment — see below
├── camp-notes/           the design record — see below
└── camp-live/            the composed tree — not in git, see below
```

**`camp/` is the tool, and only the tool.** Nothing about this directory,
this machine, or the way the work is organised appears in it. That is the
attachment rule this whole arrangement follows: the environment may know
its product, the product may not know its environment. Every pointer here
runs outward, and nothing points back.

**`camp-workspace/` and `camp-notes/` are part of this repository.** The
first carries what a session needs and the product must not contain —
`CLAUDE.md`, the agent definitions, the skills, the output style. The
second is the design record: the specification, the measured constraints,
and the log of what the build found. Both were separate repositories
until they were folded in here with their history; a clone brings the
whole environment down, which is the point of there being one repository
and not four.

**`camp-live/` is where the work happens.** It is not in git and never
will be: it is not a directory of files but the composition itself, built
at every `camp run` and gone when the session ends. It has to exist and be
**empty** before a session starts, which is why the clone does not bring
it — git cannot record an empty directory, and a placeholder file in it
would make camp refuse. Create it once, after cloning.

## Making it work from a clone

```
mkdir camp-live                  # git cannot record an empty directory
$EDITOR .camp/config.yml         # set env: to this directory's real path
camp accept
camp plan                        # read it before the first run
camp run -- bash
```

`camp plan` names anything that would stop it, in sentences, with the way
out. Nothing is mounted while you read it.

## What the configuration says

Three points it makes concretely, all of them worth knowing before you
write one of your own:

**`.git` is declared, not derived.** Both the code repository and the
workspace have one at their root, and OverlayFS merges directories, so
without an explicit bind the two histories would union. camp never adds
that mount on its own; if it is missing, the overlap gate refuses the
composition before anything is mounted — by a rule that knows nothing
about git.

**`.notes` is a writable mount**, because the design record has to be
editable from inside a session while the workspace is read-only in there
on purpose. One thing follows from it living in this repository rather
than its own: `git` run in `.notes` from inside a session walks up to the
composed root and finds the **code** repository, so the design record is
committed from outside the composition. The generated exclude still
covers `/.notes`, so nothing lands in a commit by accident; what changed
is that the boundary is now detection rather than impossibility.

**`.claude` is an islands mount**: the entries the workspace tracks stand
read-only, and everything only this machine owns — `settings.local.json`,
the worktrees, the locks — lands in camp's machine-local storage
underneath.

## Working here

```
camp plan          what would be mounted, in order, and why
camp doctor        what this machine or this environment lacks
camp run -- <cmd>  the session
camp shell         a shell in the composed tree
```

A write inside `camp-live` lands in `camp/` — that is the point. A write
to anything the workspace provides fails with `EROFS`, loudly, because a
change that looks applied and exists in no repository is the failure this
tool is built against.

camp being developed inside its own composition is the useful part: every
change to the tool is used the same day, by the session that made it.

## License

MIT, in the repository this one composes. This one is configuration and
description.
