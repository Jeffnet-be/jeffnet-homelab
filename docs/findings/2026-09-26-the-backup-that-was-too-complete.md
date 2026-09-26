# The backup that was too complete

**Found:** 26 September 2026
**Class:** The control was producing too much
**Subject:** Network-device configuration backup

A config backup that had already failed in two different ways, and was on its third, which was the
one that looked like success.

---

## The symptom

The collector pulls the firewall's running configuration on a timer and commits it to a local git
repository, so that every change to the device is dated and diffable.

It had spent its first three weeks not doing that at all. It authenticated as a deliberately
read-only account, that account was not permitted to render the configuration, and the collector
committed the device's refusal as though it were the configuration: one commit, whose entire body was
an error string, never updated again, because an unchanging error produces no second commit. That is
a finding in its own right - **the artefact exists, is dated, is in version control, and is the
failure text** - and it is worse than a backup that never ran, because it produces evidence.

This write-up is about what happened after that stopped being true. Looking at the same repository
later, every single collection had produced a commit, and every commit's diff was about twenty-odd
lines long.

At a glance this is a working control being noisy. A config backup that commits on every run is
storing more revisions than it needs to, which is untidy rather than dangerous, and there is always
something more urgent than tidy. It had been read that way for weeks - *a backup that churns is
still a backup*, which is true, and which is not the question the churn was answering.

The reason it got looked at properly was unrelated: the diff was annoying enough to want silenced,
and silencing it meant finding out what was in it.

---

## The chain

**1. Two of the changing lines were bookkeeping.** The device renders its own clock and a
configuration version counter into what it hands out, and both differed between reads. That accounts
for two lines out of twenty-odd, and it accounts for why a commit appears every hour: the artefact
genuinely is different every time, so there is always something to commit.

This is the point at which the "cosmetic" reading is most defensible, and it is wrong, because two
lines had been explained and the rest had not.

**2. The rest of the changing lines were the device's stored credentials.** The configuration includes
every secret the device holds, in the obfuscated form the vendor stores them in - and the device
**re-obfuscates them every time it renders the configuration**, so each read produces different
ciphertext for the same unchanged secret. That is the churn. It was not a formatting quirk. Every
line of it was a secret being written into version control, again, on a schedule.

The obfuscation is not encryption at rest in any meaningful sense. It is reversible without a key
unless a separate platform feature is switched on, and it is best read as *encoded*, not protected.

**3. The collector already knew how to strip them.** Its per-platform module has a redaction path
that replaces exactly those values before the configuration is written out. It is gated behind a
flag, and the flag was not set - not set to `false`, but absent, because the block it lives in was
rendered as empty:

```yaml
vars: {}
```

A block with no keys in it is valid, silent, and reads as intentional. There is no error and no
warning; the redaction path simply never runs, and the backup works perfectly at everything except
the one thing the flag controls.

**4. And it should not have been collecting at all.** Four days earlier, three separate documents
said this control had *never once* retrieved a configuration. That claim had been written after
reading the collector's template, seeing that it authenticated as a deliberately read-only account,
and reasoning - correctly, for the configuration as it stood when the template was written - that
such an account cannot render a configuration at all.

The template was still accurate. The device was not. The account's permissions had been widened **on
the device**, by hand, during an unrelated incident twelve days before, and nothing in the repository
could show that, because nothing in the repository had changed.

**The repository's own history dated it.** Dividing the number of commits by the collection interval
put the first successful collection within a day of that incident. A count nobody had thought of as
evidence settled in one arithmetic step a question that three documents had got wrong by inference.

---

## The cause

Two causes, in two different places, and neither is a bug:

- **One empty configuration block**, which disabled a redaction feature by omitting it rather than by
  refusing it.
- **A permission change made on the device**, outside the repository, which turned a control that had
  been failing loudly into one that was failing quietly.

Either alone is harmless. The first one only matters once the second one happens, and the second one
was invisible to every document describing the system.

---

## What changed

- The flag is set. The redaction path runs.
- **Verified across two consecutive collections**, not one - the whole point is a difference between
  successive reads, so a single clean artefact proves nothing. The diff is now the clock and version
  lines, and nothing else.
- The repository was re-initialised, so the history is gone rather than rewritten.

And two things were written down rather than fixed, because writing them down is the fix:

**The clock line that survives is now a heartbeat.** It was the obvious next thing to silence, and
silencing it would have been a mistake: because the device renders its clock into every read, a commit
appears on every successful collection and only on a successful collection. That makes "no commit
since the last interval" the precise liveness condition for this control: the collector stopping, the
device refusing and the credential failing all produce the same silence, and one check catches all
three.

It is also a far tighter condition than the one originally written down, which was the repository
going a week without a commit. At hourly collection that is a hundred and sixty-eight missed reads
before anybody hears - a threshold chosen for a control that was believed to commit only when the
device changed, and wrong for one that commits every time it succeeds. **The right staleness
threshold is a property of the collection interval, not of the thing being collected.**

And the line it rests on looked exactly like noise. **An incidental signal still has to be recorded
as deliberate**, or the next person tidies it away - correctly, by every standard except the one
nobody wrote down.

**A redacted configuration backup is not a restorable one.** Stripping the secrets is correct and it
is not free: the artefact that is now safe to keep in version control can no longer rebuild the
device, because the thing that was removed is part of what a restore needs. Those are two different
jobs - *a dated record of what changed*, and *an artefact you can restore from* - and one control
cannot do both. Finishing the first one quietly ends the second, so the restore path has to be a
separate, deliberate artefact held somewhere appropriate to what it contains.

---

## The general lesson

**Read the noise before you suppress it.** A diff that changes constantly while changing nothing of
substance is not a cosmetic problem, it is a description of what the artefact contains. The instinct
to quiet it is the right instinct; doing so without reading it first deletes the evidence and keeps
the defect. Ten minutes of reading was the entire investigation.

**A backup that is too complete is a secret store you did not know you were operating.** Every
question you would ask about a credential store - who can read it, is it encrypted, is it backed up
somewhere else, how long does it retain - applies to it, and none of them have been asked, because it
is filed in your head as a backup.

**"Configured" and "running" are separate facts, and a configuration read cannot settle an outcome
question.** The wrong conclusion here was reached by reading the right file. The template said the
account could not do the job, and the template was accurate; what it could not say was what the
account had been granted since, on a device that is configured by hand. Read the artefact, not the
thing that produces the artefact.

**A feature disabled by an empty block fails silently in the unsafe direction.** An explicit `false`
is reviewable - someone chose it, and a reviewer can ask why. An absent key looks like nothing at
all, and the behaviour it selects is whichever one the software defaults to. A default is only safe
in the direction that fails loudly, and a redaction default that fails towards *not redacting*
fails in the other one.

**Three failures of one control, none of which looked like an outage.** It produced an artefact that
was its own error message; then it worked and nobody knew; then it worked and stored too much. The
only thing that distinguished them was opening the artefact.
