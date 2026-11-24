# Principles

Every line here exists because it happened. None of them are aspirational, and none were learned
from a book - each one is the residue of a specific fault on real hardware, most of which are
written up in [../findings](../findings/).

## On verification

**Verify effective configuration, not configuration.** `sshd -T`, `exportfs -v`, `docker inspect`,
the served response, the environment as well as the config file. A correct SNMP user on a disabled
agent reads as configured.

**"Configured", "running", "declared" and "has ever worked" are four separate facts.** A backup
storage can be defined, mounted, scheduled, green - and have never carried a byte.

**Verify a backup by restoring from it**, and know which storage the restore came from. Anything
else is a claim about existence, not about recovery.

**A backup you have made safe to store may no longer be safe to rely on.** Stripping secrets out of
a configuration backup is correct, and it removes part of what a restore needs. *A dated record of
what changed* and *an artefact you can restore from* are two jobs, and one control usually cannot
hold both - so finishing the first one quietly ends the second unless the second is given its own
artefact.

**Verify a control's coverage, not just its existence.** A nightly backup job that protects under a
third of the estate is not a smaller version of a working backup job. It looks identical from the
dashboard.

**Read the machine, not your notes.** A dozen times the notes have been wrong and the system right.
A file's mtime has settled in one command what three rounds of inference could not.

**A configuration read cannot settle an outcome question.** "Is this control working?" is answered by
the artefact it produces, never by the file that produces it. A template that says an account has
insufficient permission is evidence about the template; on anything configured by hand, what the
account may do today is a separate fact, changed in a separate place, recorded nowhere. Three
documents once stated a control had never worked, from a correct reading of a correct file, while it
had been working for twelve days.

**Read the noise before you suppress it.** A diff that changes constantly while changing nothing of
substance is not untidiness, it is a description of what the artefact contains. The instinct to
silence it is right; acting on it without reading it first removes the evidence and keeps the defect.

**Count what you already have before inferring.** A repository's commit count divided by its
collection interval dated an undocumented change to within a day. Artefacts carry their own
timestamps, and arithmetic on them beats three rounds of reasoning about them.

**Check that the test could have produced the signal** - including your own probe. Three zeroes from
a `curl` that never connected are not three measurements.

**A permit is a capability, not a measurement.** A firewall rule, a granted scope, an open port: each
says what *may* happen, never what does. Reading a rule and reporting behaviour is the same error as
reading a container's exposed ports and calling them published - and a rule covering several
services cannot have its counter attributed to one of them. The behaviour is in the session log, and
nowhere else.

**A granular rule sitting above a catch-all is a logging category, not a control.** Every careful rule
below a permissive one classifies traffic that was already allowed. Read the table in evaluation
order, find the widest thing in it, and judge everything above it against that - because an audit
that walks the list rule by rule will find each individual one defensible and miss that none of them
decide anything. Narrowing the catch-alls is the work; the granular rules only start being controls
afterwards.

**An export is a summary, and a summary can drop a qualifier.** A report of a policy table rendered a
rule's destination as a plain list where the device held a *negated* one - which reads as the exact
opposite of what is configured, in a document that otherwise looks authoritative. Confirm anything
that looks nonsensical against the device's own output. A rule that makes no sense is usually a
reporting artefact, and occasionally it is a real misconfiguration; the export cannot tell you which.

## On guards

**A guard that cannot fail is not a guard.** Observed variants: a string the software never emits; a
task that `--check` skips; a placeholder silently dropped; a `| default([])` that empties the list
being compared *from*; a pipeline whose last command always succeeds; an error handler wrapped around
a file read, which turns "the file is missing" into "the file is empty".

**Prove a guard denies by breaking what it compares against** - and re-prove it after you change the
guard. An old proof does not carry over a rewrite.

**A pipeline reports its last command's status.** `set -o pipefail`, and give every read an explicit
return-code contract. Otherwise a command that errors produces an empty result, and an empty result
passes every comparison it is subtracted from and fails every one it is subtracted into.

**A default is only safe in the direction that fails loudly.** A `| default()` on a variable nothing
ever defines is a guess, not a default - require it and let it fail. The same applies to a feature
disabled by *omission*: an empty block is valid, silent, and reads as intentional, so the behaviour it
selects is whichever one the software defaults to. An explicit `false` is reviewable - somebody chose
it, and a reviewer can ask why. An absent key looks like nothing at all.

**A check behind a login checks the login.** Decide which side of it the check must see, and make the
other side a failure: an uptime check that follows the redirect, or a login probe that also accepts
`200`, is green whether the login is there or not.

**The tool you test with is not the client you deploy.** An identity provider redirected `curl` and
answered the exporter with `401`, because one sent an `Accept` header and the other did not. Ask the
probe what it receives.

**A guard is only as reliable as the least-converged host in its group.** The host outside the
nightly is where guard defects surface, months late.

## On alerting

**Many detectors, one pager.** Ask "where does the alert go" of every subsystem separately. Five
detectors with four pagers means one of them notices things and tells nobody.

**Recording is not alerting, and a control is finished when a person is interrupted** - not when the
notification is sent. Seven correct alerts into a channel nobody was subscribed to on a device that
interrupts produced the same outcome as having no monitoring at all.

**Noise and silence are not symmetric failures.** Clients report noise. Nobody reports silence.

**Alert on the outcome, not on the job.** A mirror whose token has expired keeps running, exits
cleanly and serves yesterday's code; only "is the copy behind its source?" catches it. The same goes
for a nightly that succeeds: a check-mode run that *would change something* is drift, and deserves
its own page next to the one for failure.

**Never let a monitor's normal state include red.** One permanently wrong dot trains everybody to
read past the board.

## On declaration

**Render, don't reconcile.** Declare a service once; derive the proxy route, the dashboard card, the
uptime check and the metric targets from it. Deleting the declaration should delete all four.

**Deleting a guest leaves behind everything that *enrolled* it.** Derived consumers drop by
themselves; enrolled ones persist and show as broken rather than absent - which reads as a fault
rather than a deletion.

**Generate the allow-list from the declaration, and apply the declaration's own filter.** A firewall
rule written from memory drifts from the service it protects; one derived from the same entry that
routes the service cannot. The generator must ask the same question the router asks - *is this
service routed?* - or it will open ports for things that listen on nothing.

**Roll out default-deny from a canary to the ingress.** One guest first, with checks from outside
(the direct port times out, the name still loads); then waves; the thing everything depends on last,
alone. Two of anything redundant never go in the same change.

**A variable's name is not its value.** Read what it resolves to on the target. In a
one-stack-per-host estate, a path named after a service is a category error.

**One fact, one definition** - and know where a fact came from. Two sources for one fact agree until
the day they don't, and which one wins depends on which task you happen to be reading.

## On writing things down

**Write down what you chose not to have, and name the consequences.** An undocumented deliberate
absence is indistinguishable from an oversight, including to the person who made it. Mail is
deliberately not a notification path here; that sentence exists precisely so nobody "fixes" it.

**A hardening step that closes a path must say what it will cost later.**

**A stopgap needs a written end condition**, or it is a permanent decision taken by accident. And
the end condition has to be executed when it arrives - one here was written correctly, arrived, and
was still sitting in place three days later.

**An incidental signal still has to be recorded as deliberate, and a deliberate one is better.** A
control can emit proof of its own liveness as a side effect - a timestamp line that changes on every
successful read, so a commit appears only when the collection worked. That makes silence the failure
signal, which is worth alerting on, and it survives only if somebody writes down that the ugly line is
load-bearing. Here it was written down, and was tidied away two months later anyway, by someone
changing the code without reading the record. Recording it is the minimum; **the better fix is a
signal that exists only to be one**, such as a direct report to an external heartbeat on every run,
so that tidying cannot remove it by accident.

**Record what a test cannot prove.** A scheduled check inherits its runner's reach; enumerate what
it cannot see and write that beside the green.

**An undocumented control is never reviewed.** Not misconfigured, not broken - absent from every
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

**The machine that executes the code should not be able to rewrite it** - enforced by a read-only
deploy key, not by convention. A convention only a person enforces stops existing the moment a
machine does the work.

**A credential that has lived somewhere it should not is spent**, whether or not it was used.

**Git is the wrong transport for a file that must not reach one of its remotes.**

**Prefer an existing certificate to a new password**, and prefer a mechanism with no ambiguity to
one whose documentation leaves the ambiguity open.

## On the work itself

**A proof of concept that was never retired is indistinguishable from production to every system
that watches it** - the backup job, the uptime check, the dashboard card describing it as something
it no longer is.

**A fresh clone is a completeness test for the repository.** Five dependencies had lived on exactly
one machine for months, invisible, because one machine always ran the code.

**Move a workload while it is empty.** Migrating a service between nodes is trivial the day it is
deployed and a project three months later.

**A host that runs your jobs cannot be restarted under them.** The job dies, and the scheduler keeps
showing it as running, because nothing is left to record the end.

**A plan from a stale checkout is right about the wrong code.** "No changes" from a copy that never
pulled the merge is a true statement about nothing that matters. Pull before you apply.

**Name which copy you mean.** "I fixed it" is not a complete statement when three machines hold a
copy and two remotes hold the truth.

**An observation can be true and still not bear on the question.** Time correlation is the weakest
evidence available; an outage that coincided with a configuration run was caused by a loose cable.

**Know when to park something.**
