# A delivery guard for `.env`-driven compose stacks

Two facts that look like one:

- **`.env` substitutes.** Docker Compose reads it to expand `${VAR}` *in the compose file itself*.
- **`env_file:` injects.** Only that puts variables inside the container's environment.

A stack can therefore render thirty-nine variables into a `.env` that the application never sees —
which is exactly what happened here, with eleven consumed beside thirty-nine rendered, for weeks.

This guard compares what the template *declares* against what the running container *received*, and
fails the play when the difference is non-empty.

## What it is guarding against, and what it is guarding itself against

The first version of this guard spent three runs unable to fail, and a later version reported a
false positive on the one host that was outside the nightly convergence. Three defects, all of them
in the guard rather than in what it measured:

1. **The target was a guess.** The container to inspect was referenced only as
   `| default(stack_name)` — and nothing in the repository ever *set* it. That is correct by
   coincidence wherever the container happens to share the stack's name, and wrong everywhere else.
   A `| default()` on a variable nothing ever defines is a guess, not a default.
2. **The read could not fail.** `docker inspect x | python3 -c '...' | sort -u` returns **`sort`'s**
   exit status. The inspect failed, python died on empty input, `sort` succeeded on nothing, and
   Ansible saw a clean task with empty output. An empty list then passes every comparison it is
   subtracted *from* and fails every one it is subtracted *into* — hence a false pass on some stacks
   and a false alarm on another.
3. **The assert and its message were two expressions that had to agree.** The same four-term
   difference chain, written twice.

## The shape of the fix

- `set -o pipefail` with `executable: /bin/bash` on **every** read, and an explicit return-code
  contract per read, because the contracts genuinely differ:
  - well-formedness: rc 0 **or 1** — `grep -v` exits 1 when it finds nothing, which is the healthy
    case
  - declared keys: rc 0 or 1, **and non-empty** — a template that declares nothing is a defect
  - substitution keys: rc 0 or 1, **empty is valid** — zero `${}` is normal for an `env_file` stack
  - container keys: **rc 0 and non-empty** — so "could not read" is never mistaken for "received
    none"
- The difference computed **once** with `set_fact`, used by both the assert and its message.
- **Every `| default([])` removed from the assert.** They were only load-bearing because the reads
  could silently produce nothing; with the reads honest, a default could only hide something.
- The primary-container name **required per stack** and set explicitly even where it equals the
  stack name — a fact rather than a coincidence.
- The declaration assert runs **before** the stack is brought up: it checks a declaration, not
  state, so it should fail before any work is done.
- Two `block:`s, each gated once on the template's existence, rather than a `when:` repeated per
  task. The original read tasks had no `when:` at all, which is why removing the bad default broke
  every `.env`-less stack with *undefined variable*.

## Proving it

Add a key to one stack's `.env` template that no container will ever receive — `GUARD_CANARY` — and
confirm the assert fails and names it. Then remove it.

Do this **again after any change to the guard**: the old proof does not carry over when the reads,
the return-code contracts and the comparison have all been rewritten. Proving a control allows is
not proving it denies.

> The code below is an illustrative reconstruction of the pattern, with estate-specific variable
> names generalised. Read it as a shape to pattern-match, not as a drop-in role.

## One more thing the reconstruction turned up

`failed_when:` given a **list** is an AND, not an OR. Written as

```yaml
failed_when:
  - _received.rc != 0
  - _received.stdout_lines | length == 0
```

the task only fails when the command errored **and** returned nothing — so a read that errors while
still printing something passes, and so does a read that succeeds while printing nothing. Two
return-code contracts silently become one. Write the disjunction explicitly:

```yaml
failed_when: >-
  _received.rc != 0
  or _received.stdout_lines | length == 0
```

Another shape of the same class: a guard whose failure condition is narrower than the one you meant
to write.
