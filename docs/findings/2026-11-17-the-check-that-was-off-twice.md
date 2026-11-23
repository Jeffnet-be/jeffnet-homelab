# The check that was off twice

**Found:** 16-17 November 2026 · **Class:** a control that was off, and stayed off after it was switched on.

> **In short.** A manual SSH session refused to connect: the server's host key had changed. The
> automation had connected to the same machines the same day without a word. Host key checking was
> off in the automation's configuration. Switching it on there was not enough, because the scheduler
> that runs the nightly jobs switches it off again for every run.

---

## The symptom

Two new virtual machines had just been built on addresses that older machines used to have. A
manual `ssh` to one of them stopped with `REMOTE HOST IDENTIFICATION HAS CHANGED`. That was expected:
new machine, old address, old key on file.

What was not expected was the silence before it. The configuration tool had already connected to
all of them and had said nothing about a key. A third host, the
automation server itself, turned out to present a key that no longer matched the one on file either.
Three mismatches in two days, and the automation had noticed none of them.

## The chain

**1. The easy reading was "nothing is wrong".** Two of the three had an innocent explanation: new
machines on reused addresses always present new keys. That explained the manual warning. It did not
explain why the automation, which connects over the same SSH, had accepted keys it had never seen
before. A check that cannot tell a rebuilt machine from an impersonated one has not checked either.

**2. The setting was off.** The automation's configuration file said `host_key_checking = False`, a
common lab default that saves a failed run after every rebuild. With it off, the automation connects to whatever answers on an address.

**3. Switching it on was not enough.** The obvious fix was one line in that file. The fix also had
to hold for the way the scheduled runs start, so that was read too. The scheduler that runs the nightly jobs sets
`ANSIBLE_HOST_KEY_CHECKING=False` in the environment of **every** run it starts. That is in its
source code, not its documentation. An environment variable outranks the configuration file, so
the nightly would have kept connecting blind while the file said otherwise. Worse, with the variable
set, the tool puts `StrictHostKeyChecking=no` in front of any SSH options I add, and SSH keeps the
first value it reads. So that route was closed too.

**4. What outranks the environment.** An inventory variable does. Setting host key checking there,
for every host, wins over both the file and the scheduler. It is also the one place a later change
to either cannot quietly undo.

**5. Then: which keys are the right ones?** Checking is only as good as the list it checks against.
The usual way to build that list, `ssh-keyscan`, asks the network what each machine's key is, which
is trust on first use from exactly the position an attacker would be in. Instead, each guest's key
is read **inside the guest, through its hypervisor**, which never touches that network, and must
match what the network says. The three physical hosts are confirmed by hand against their own
console. The collection script refuses to write a line it could not verify both ways.

**6. A dead end in the script.** Its first run on the controller ended silently with no output at
all. One guest is a canary that is normally powered off; its scan failed, and under the script's
strict error handling that one failure ended the whole run. Each host is now reported and skipped
on its own, and the run ends with a summary and a non-zero exit.

## The cause

Host key checking was disabled in the configuration file, and disabled again, independently, by
the scheduler for every nightly run. Fixing only the first would have left the second in place, and
the file would have said the check was on.

## What changed

- **Checking is on for every host, set in the inventory**, where it outranks the file and the
  scheduler's environment.
- **One tracked file of host keys is the only one the automation trusts.** Personal `known_hosts`
  files are not read.
- **Every line in it is verified two ways** (inside the guest, and from the network), and only the
  collection script writes it. No key is typed by hand or copied from a scan alone.
- **CI checks the file against the inventory:** one line per host, no extras, no duplicates, none
  hashed. A machine added or retired without its key fails the pull request, not the nightly run.
- **A new machine has a named waiting state** in the inventory until its key is collected after it
  is built, so it is excluded deliberately rather than by accident.
- **A mismatch is now an incident, not a chore.** The nightly fails on that host, and the first
  question is why the key changed, before the line is replaced.

Not seen yet: the new check catching a real mismatch on its own. The next rebuild is that test.

## The general lesson

**A security setting is the value that wins, not the value you wrote.** A configuration file is one
layer among several: environment variables, command-line flags, the defaults a wrapper adds on the
way in. Before calling a control on, find out what the running process actually received.

### The professional twin

A Group Policy setting overridden by another GPO later in the processing order. A CI runner image
that exports `GIT_SSL_NO_VERIFY` or sets `StrictHostKeyChecking no` in a global SSH config. A
container that sets `NODE_TLS_REJECT_UNAUTHORIZED=0` "for the build". Each leaves the visible
setting correct and the effective one off. **Check the effective value, where it is used.**
