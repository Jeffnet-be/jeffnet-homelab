# The guard that could not fail

**Found:** 2 September 2026 · **Class:** the control was the defect.

A guard had been added to catch a real problem: variables declared in a stack's environment file and
never actually delivered to the container. It worked. It had been green for days. Then it produced
a **false positive** on one host, and pulling that thread found that it had never been capable of
producing a true one on several others.

This was the second time in four days that a guard on this estate turned out to be unable to fail.

---

## Why it surfaced where it did

On the one host that sits **outside** the nightly convergence run. That host was the least-converged
in the estate, so it was the first to run the guard as it had been written rather than as it had
been patched - and it produced a finding within the hour of an unrelated investigation.

> **A guard is only as reliable as the least-converged host in its group** - and that host is where
> its defects surface, late.

## The symptom

The guard reported three variables as declared-but-never-delivered. They *were* delivered: the
container carried the environment file and had all three. The complaint was false.

## Three defects, all inside the guard

**1. The target was a guess.** The guard inspected a container whose name came from a variable
referenced only as `| default(stack_name)` - and **nothing in the repository ever set that
variable.** So it always inspected a container named after the stack. That is correct by coincidence
wherever a stack happens to run a container of the same name, and wrong everywhere else. On this
host the containers were named after components, and nothing was called after the stack at all.

> **A `| default()` on a variable nothing ever defines is a guess, not a default.** A default is only
> safe in the direction that fails loudly; this one picked a plausible-looking target and then
> reported confidently on whatever it found.

**2. The read could not report its own failure.** The inspect command was the first stage of a
pipeline ending in a sort:

```
docker inspect <container> | python3 -c "…" | sort -u
```

The inspect failed. The python stage died on empty input. `sort` succeeded on nothing. **A
pipeline's exit status is its last command's**, so the return code was zero and the task reported a
clean success with empty output.

And an empty list is not neutral:

> **An empty result passes every comparison it is subtracted from, and fails every one it is
> subtracted into.** Which produced a silent false *pass* on some stacks and this noisy false *alarm*
> on the one where the empty list happened to land on the other side of the comparison.

**All four reads in the role shared the defect.** Three ended in `sort`, one in `cut`.

**3. The assert and its message were two expressions that had to agree.** The same four-term
comparison was written twice - once in the condition, once in the failure message. Two copies of one
fact, kept in sync by hand.

## The precedent, three days earlier

A different guard, same class. Its comparison included a `| default([])` on the list it compared
*from*, so whenever that list was empty - which was the failing case it existed to catch - the
difference was empty too and the assert passed. It had been green for three runs and had never been
able to do anything else.

Two guards, four days, one failure mode: **the control was structurally incapable of reporting the
condition it was written for.**

## What changed

- **`set -o pipefail`** with an explicit bash executable on every read, and **an explicit return-code
  contract per read** - because the correct contracts genuinely differ. One read is healthy when it
  finds nothing and exits non-zero. Another is defective if it returns nothing at all. A single rule
  would have been wrong for at least one of them.
- **The primary container is now required per stack**, and set explicitly even where it equals the
  stack name - a fact rather than a coincidence.
- **The declaration check runs before the stack is brought up**, because it checks a declaration
  rather than state, and should fail before any work is done.
- **The comparison is computed once** and used by both the condition and the message.
- **Every `| default([])` was removed from the assert.** They had only ever been load-bearing because
  the reads could silently produce nothing. With the reads honest, a default could only hide
  something.

## Proving it

A canary key was added to one stack's environment template - a variable no container would ever
receive - and the guard failed and named it. Then the canary was removed.

The previous proof did **not** carry over. The reads, the return-code contracts and the comparison
had all been rewritten; what had been proven was a different control that happened to share a name.

> **Prove a guard denies by breaking what it compares against - and re-prove it after you change the
> guard.** Proving that a control *allows* is not proving that it denies.

## The general lesson

Every control in this estate is now assumed guilty until it has been observed refusing something.
The specific questions that come out of these two:

- **Does every stage of this check report its own failure**, or only the last one?
- **What does this check do when its input is empty** - and is empty a valid answer here, or a symptom?
- **Is any value in this check a guess dressed as a default?**
- **When did this last deny something**, and was it a real fault or a fabricated test?

### The professional twin

Audits and compliance checks fail this way constantly, and they fail *quietly* - a script that
enumerates non-compliant machines, an API call whose credential silently expired, an empty result
set, and a report reading zero findings. **An audit that finds nothing deserves exactly the
scepticism of a control that reports success.**
