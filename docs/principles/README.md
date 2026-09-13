# Principles

Every line here exists because it happened. None of them are aspirational, and none were learned
from a book — each one is the residue of a specific fault on real hardware, most of which are
written up in [../findings](../findings/).

## On verification

**Verify effective configuration, not configuration.** `sshd -T`, `exportfs -v`, `docker inspect`,
the served response, the environment as well as the config file. A correct SNMP user on a disabled
agent reads as configured.

**"Configured", "running", "declared" and "has ever worked" are four separate facts.** A backup
storage can be defined, mounted, scheduled, green — and have never carried a byte.

**Verify a backup by restoring from it**, and know which storage the restore came from. Anything
else is a claim about existence, not about recovery.

**Verify a control's coverage, not just its existence.** A nightly backup job that protects under a
third of the estate is not a smaller version of a working backup job. It looks identical from the
dashboard.

**Read the machine, not your notes.** A dozen times the notes have been wrong and the system right.
A file's mtime has settled in one command what three rounds of inference could not.

**Check that the test could have produced the signal** — including your own probe. Three zeroes from
a `curl` that never connected are not three measurements.

**A permit is a capability, not a measurement.** A firewall rule, a granted scope, an open port: each
says what *may* happen, never what does. Reading a rule and reporting behaviour is the same error as
reading a container's exposed ports and calling them published — and a rule covering several
services cannot have its counter attributed to one of them. The behaviour is in the session log, and
nowhere else.

## On guards

**A guard that cannot fail is not a guard.** Observed variants: a string the software never emits; a
task that `--check` skips; a placeholder silently dropped; a `| default([])` that empties the list
being compared *from*; a pipeline whose last command always succeeds.

**Prove a guard denies by breaking what it compares against** — and re-prove it after you change the
guard. An old proof does not carry over a rewrite.

**A pipeline reports its last command's status.** `set -o pipefail`, and give every read an explicit
return-code contract. Otherwise a command that errors produces an empty result, and an empty result
passes every comparison it is subtracted from and fails every one it is subtracted into.

**A default is only safe in the direction that fails loudly.** A `| default()` on a variable nothing
ever defines is a guess, not a default — require it and let it fail.

**A guard is only as reliable as the least-converged host in its group.** The host outside the
nightly is where guard defects surface, months late.

## On alerting

**Many detectors, one pager.** Ask "where does the alert go" of every subsystem separately. Five
detectors with four pagers means one of them notices things and tells nobody.

**Recording is not alerting, and a control is finished when a person is interrupted** — not when the
notification is sent. Seven correct alerts into a channel nobody was subscribed to on a device that
interrupts produced the same outcome as having no monitoring at all.

**Noise and silence are not symmetric failures.** Clients report noise. Nobody reports silence.

**Never let a monitor's normal state include red.** One permanently wrong dot trains everybody to
read past the board.

## On declaration

**Render, don't reconcile.** Declare a service once; derive the proxy route, the dashboard card, the
uptime check and the metric targets from it. Deleting the declaration should delete all four.

**Deleting a guest leaves behind everything that *enrolled* it.** Derived consumers drop by
themselves; enrolled ones persist and show as broken rather than absent — which reads as a fault
rather than a deletion.

**A variable's name is not its value.** Read what it resolves to on the target. In a
one-stack-per-host estate, a path named after a service is a category error.

**One fact, one definition** — and know where a fact came from. Two sources for one fact agree until
the day they don't, and which one wins depends on which task you happen to be reading.

## On writing things down

**Write down what you chose not to have, and name the consequences.** An undocumented deliberate
absence is indistinguishable from an oversight, including to the person who made it. Mail is
deliberately not a notification path here; that sentence exists precisely so nobody "fixes" it.

**A hardening step that closes a path must say what it will cost later.**

**A stopgap needs a written end condition**, or it is a permanent decision taken by accident.

**Record what a test cannot prove.** A scheduled check inherits its runner's reach; enumerate what
it cannot see and write that beside the green.

**An undocumented control is never reviewed.** Not misconfigured, not broken — absent from every
document describing the system, and therefore never examined by anyone auditing those documents.
The review finds what the notes mention. Anything the notes omit is invisible to it by construction,
which is why a periodic export of the real configuration is worth more than a careful re-read.

**A control that expires on a date nobody is watching is worse than not having it.** A trial licence,
a certificate, a token with a lifetime: each becomes a protection you believe you have, on a schedule
you are not tracking. Accept the gap in writing instead, with the condition that would close it.

**A line that cannot do anything still costs attention.** A rule pointing at an interface that
accepts nothing, a disabled deny-all, a variable nothing reads: each is read by every person who
reviews the configuration, and each has to be understood before it can be dismissed. Delete them.

**Keep an explicit list of what is configured by hand.** Every estate has one. Anything configured
outside the repository goes on it, and the list is a roadmap rather than a confession.

## On credentials

**Every credential gets its own identity, scoped as narrowly as the provider allows.**

**The machine that executes the code should not be able to rewrite it** — enforced by a read-only
deploy key, not by convention. A convention only a person enforces stops existing the moment a
machine does the work.

**A credential that has lived somewhere it should not is spent**, whether or not it was used.

**Git is the wrong transport for a file that must not reach one of its remotes.**

**Prefer an existing certificate to a new password**, and prefer a mechanism with no ambiguity to
one whose documentation leaves the ambiguity open.

## On the work itself

**A proof of concept that was never retired is indistinguishable from production to every system
that watches it** — the backup job, the uptime check, the dashboard card describing it as something
it no longer is.

**A fresh clone is a completeness test for the repository.** Five dependencies had lived on exactly
one machine for months, invisible, because one machine always ran the code.

**Move a workload while it is empty.**

**Name which copy you mean.** "I fixed it" is not a complete statement when three machines hold a
copy and two remotes hold the truth.

**An observation can be true and still not bear on the question.** Time correlation is the weakest
evidence available; an outage that coincided with a configuration run was caused by a loose cable.

**Know when to park something.**
