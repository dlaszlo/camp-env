# ssh from inside a composition, against a real peer

Date: 2026-08-19, at a terminal, by the owner. This closes the
measurement the whole `session:` feature was built for: the ssh refusal
that started it (`log/2026-08-16-handoff-ssh-inside-a-session.md`) is what
the environment map and the workspace-owned launchers exist to repair, and
until now the repair had only been measured up to the point where a real
peer was needed.

## What was run

A `camp run` session in this environment, eleven mounts, and then:

```
$ command -v ssh scp sftp
/home/dlaszlo/dev/camp-env/camp-live/.workspace/bin/ssh
/home/dlaszlo/dev/camp-env/camp-live/.workspace/bin/scp
/home/dlaszlo/dev/camp-env/camp-live/.workspace/bin/sftp

$ ssh librechat
Welcome to Ubuntu 26.04 LTS ...
librechat@librechat:~$ exit

$ scp /etc/hostname librechat:/tmp/camp-check && ssh librechat rm /tmp/camp-check
hostname                        100%    7     9.6KB/s   00:00

$ sftp -b /dev/null librechat

$ git ls-remote git@github.com:dlaszlo/camp.git | head -3
0255c6ed...    HEAD
bb407ec0...    refs/heads/fix/review-8ddf464
0255c6ed...    refs/heads/main
```

## What it settles

**All three entry points reach the real program through the launcher.**
`command -v` answers with `<live>/.workspace/bin/` for each, which is the
composition's own PATH doing what it was declared to do -- and every one
of the three then completed a connection.

**No "Bad owner or permissions".** That refusal is what a session used to
meet: a session maps only the invoking uid, so a system-wide ssh
configuration owned by root reads as owned by `nobody`, and ssh refuses a
configuration it cannot attribute. The launcher's `-F` names the user's
own file and, in doing so, makes ssh skip the system-wide one. Four
programs, no refusal.

**The user's own configuration still applies.** `librechat` is a host
alias from `~/.ssh/config` -- it resolved, the key was accepted, and the
connection went to the address the alias names. A session that could
only reach hosts spelled out in full would have been a repair in name
only.

**git over ssh works, against the real remote.** `git ls-remote` answered
with this repository's actual refs, which is `GIT_SSH_COMMAND` covering
the case where no shell is involved at all.

## And the same terminal, after the session ended

```
(camp) dlaszlo@ubuntu:~/dev/camp-env/camp-live$ exit
dlaszlo@ubuntu:~/dev/camp-env$ command -v ssh
/usr/bin/ssh
dlaszlo@ubuntu:~/dev/camp-env$ ssh -G librechat >/dev/null && echo "kint is tiszta"
kint is tiszta
```

This is the half that matters most and is easiest to forget. Everything
above was arranged by prepending a directory to a PATH and declaring an
environment -- and both of those are the kind of arrangement that leaks
onto a machine if it is done in the wrong place. It did not: `ssh`
resolves to `/usr/bin/ssh` again, and `ssh -G` parses the configuration
it always parsed. **Nothing camp did survived the session.**

The prompt says the same thing in passing. The `(camp)` mark is gone the
moment the shell is the outside one again, because it was declared in the
composition's `session.environment` rather than written into a startup
file on the machine. A marker that outlived its session would be the same
leak in a smaller form.

## What is left, and it is a different question

The keyring: whether libsecret is reachable across the namespace
boundary. A `git push` to an https remote whose credentials come from it
is the shortest test. It matters for the same reason this did -- a push
that cannot reach the keyring is the same user-visible failure as the ssh
refusal -- and no configuration key repairs it.
