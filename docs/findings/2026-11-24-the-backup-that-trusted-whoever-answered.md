# The backup that trusted whoever answered

**Found:** 24 November 2026 (the setting was older: it was flagged in September and left) · **Class:** a
control that authenticated to its target without checking which target it was.

> **In short.** The configuration backup for the edge firewall logged in over SSH every hour without
> checking the firewall's host key. Anything that answered on that address would have been handed the
> backup account's password. Turning the check on meant first deciding which key to trust, from a
> place where the network could not be the one answering. Once the backups were trustworthy, they
> turned out to be noise: every hour produced a "change" that was only the firewall's clock.

---

## The symptom

There was no outage and no alert. The lab's configuration backup logs into the edge firewall every
hour, reads the configuration and commits it to a git repository, so that any change on the firewall
has a dated, diffable record.

Its SSH client ran with host-key checking off. I noted it in September and put it off: the firewall
sat inside the lab, and that felt like enough. It came back up because the lab had just finished doing the opposite
everywhere else: a week earlier, the automation's own host-key check had turned out to be off twice
over, and had been fixed so that every server key is pinned in a tracked file. The backup tool was
the one SSH client left that would talk to anything.

## The chain

**1. What "off" actually meant.** Without host-key checking, the client accepts whatever key the
other side presents. The backup account is read-only, but it is a real credential, sent on every
connection. Anything able to answer on the firewall's address, or to sit between the backup server
and the firewall, would receive it every hour, and the backup would happily commit whatever
configuration that impostor chose to show.

**2. Turning it on needs a trust decision first.** A host-key check is only as good as the key it
compares against. The obvious way to get that key, asking the network for it, is exactly the thing the
check exists not to trust. The key had to be confirmed some other way. The change was built to stop
until that happened: the tracked key file shipped empty, and the pull request's own check failed on
purpose until a person added a verified line.

**3. The first attempt returned nothing.** Scanning the firewall's key from the automation controller
came back empty: no key, no error. The firewall only accepts SSH from the few machines that need it,
and the controller is not one of them. So the scan was run from the backup server, the one place
whose view of the key matters.

**4. The firewall could not confirm its own key.** The plan was to compare the fingerprint with the
firewall's own console. The firewall's SSH client, pointed at itself, picked a different key type,
accepted it silently, and showed no fingerprint at all. That was the second wrong turn: there was no
second path to the key.

**5. Confirming the path instead of the key.** The question was narrower than it looked. To fake the
key, something on the backup server's own network would have to pose as its gateway at the moment of
the scan. So the check became whether the gateway the backup server was talking to was the firewall's
own interface. Its hardware address matched. The key was pinned on that basis, and the commit message says how it was verified.

**6. The check held.** With the key pinned and checking on, the backup ran straight away and on demand,
with no host-key errors in its log.

**7. Then the history became readable, and it was wrong.** With the backup now worth reading, the
repository showed five commits in a row titled "update". Each changed one line: the firewall's
system clock, printed in the status block the backup records above the configuration. The firewall's
newer firmware labels that line differently from the label the backup tool knows to strip, so every
fetch differed from the last, and every hour committed a "change" that was not one. A real change
would have been one commit among twenty-four a day that all looked the same.

**8. Fixing the noise without forking the tool.** The tool lets a local model file extend its built-in
one. A small extension now drops that single line, from that single command's output, and leaves
everything else to the upstream model. The first fetch afterwards committed once, removing the line;
the next fetch committed nothing.

**9. The line had been kept on purpose.** A review the same day pointed out that a September finding
had already met this clock line and decided to keep it: because it changed on every read, a commit
proved that a collection had worked, and "no commit for an hour" was the liveness alert. That decision
was written down, in a finding and in the principles, and was reversed anyway, because the fix was
made from the code without the record being read. The decision itself stands, for the reason in step
7, but liveness could not be left to chance: every collection is being made to report straight to an
external heartbeat service, success or failure, so that the service alerts when it hears nothing. That
is not closed until the alert has fired once on purpose.

## The cause

The backup's SSH client had host-key checking switched off. I had put it off because the target sat
inside the lab, and inside the lab is exactly where a compromised machine would be.

Separately, the backup's record of change was drowned by a clock line that the tool's model did not
recognise on newer firmware, so the record could not show a real change, or the absence of one.

## What changed

- **The backup checks the firewall's host key** against a tracked file. A device without a verified
  line is not backed up at all, which is a visible failure, instead of being backed up from whoever
  answered. CI fails if a device is added without its line, or if a line has no device.
- **The first trust is written down,** including how it was established and why that was enough. The
  next time the key changes, after a firmware upgrade or a hardware swap, the backups stop on purpose,
  and the same verification is repeated before the line is replaced.
- **A backup now commits only when the configuration changes.** That makes the planned staleness
  alert possible: "no commit for a day" can now mean "nothing changed", and "no successful fetch for a
  day" is a separate, meaningful signal.

## The general lesson

**Switching a check on is the easy half. The hard half is the first thing it trusts.** Host-key
checking, certificate pinning and allow-lists all reduce to "compare against a known-good value", and
the known-good value has to come from somewhere the attacker you are defending against does not
control. When no such source exists, narrow the question: here, not "is this the right key?" but "was
the path to the key the one I think it was?"

**And a record full of noise is not a record**, but before removing the noise, find out whether
anyone was listening to it. A backup that commits every hour regardless hides real changes; that same
hourly commit was also its only proof of life, and the reason was written down in a place nobody
checked before changing the code. **Read the record before you tidy.**

### The professional twin

Network backup and monitoring tools that log into switches and firewalls with host-key checking off
because "it's the management network". A trust-on-first-use prompt clicked through on a jump host.
A configuration archive whose daily diff is a timestamp, so nobody reads it, so nobody sees the night
someone added a rule. Each looks like a working control until the day it is the only witness.
