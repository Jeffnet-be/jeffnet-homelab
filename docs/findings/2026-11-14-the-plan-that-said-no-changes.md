# The plan that said No changes

**Found:** 14 November 2026 · **Class:** a true report about the wrong input.

> **In short.** A merged change was applied, the plan said `No changes`, and the change never reached
> the estate. The plan was right about the code it was given: the workstation had not pulled the
> merge. The next pull request's plan is what noticed.

---

## The symptom

A small fix to the firewall generator was merged after midnight. It stopped the generator opening a
port for a service that is not routed: an outbound-only worker that listens on nothing, which had
been given an allow rule anyway. Applying it should have removed exactly one rule.

The apply came back clean. The report was "applied, no changes", which reads as success. It is also
what a plan says when there is nothing to do, and the difference between those two is the whole
finding.

## The chain

**1. The next plan had one change too many.** The next firewall change went up as a pull request
shortly after. CI's plan showed the five guests it was meant to touch - and one more in-place
update that nothing in that pull request explained. The extra change was the rule the previous fix
should already have removed.

**2. The rule was still live, and the code was merged.** CI plans the merged code against the real
estate. Its plan proved both halves at once: `main` no longer wanted the rule, and the hypervisor
still had it. So the earlier apply had not removed it, and neither the code nor the estate was in
doubt.

**3. That left the input.** A plan compares the code you give it with the world. Given the code from
*before* the fix, it would compare a rule the code still declared with a rule the hypervisor still
held, find them identical, and say `No changes` - truthfully. A workstation that had not pulled
since the merge produces exactly that.

That is an inference, not a measurement. The workstation's plan output was not kept, and nobody
recorded which commit it was standing on. Nothing else explains all three facts, but the record
cannot prove it, and the write-up should not pretend otherwise.

## The cause

The apply ran from a checkout one merge behind `main`. The plan was correct about the code it was
given, and the code it was given was not the change.

Two things made it invisible:

- **`No changes` is the expected answer for a small change that has already been applied**, and the
  answer a stale checkout gives. The two are byte-for-byte the same message.
- **Nothing compares the estate with `main` after an apply.** CI plans pull requests, not merges.
  The only detector was the next pull request's plan, read by someone who reports every project's
  summary line rather than trusting the green check.

The harm was nil: the rule opened a port on which nothing listened. The same mistake with a change
that *closes* something - a revoked access, a removed exception - leaves the old exposure live while
every record says it is gone.

## What changed

- **Every apply step now starts with a pull**, written into every pull request's apply instructions
  rather than left as a habit.
- **The plan line for every project is posted on every pull request.** That convention is what
  turned an unexplained `1 to change` into a finding instead of a shrug.
- The missing fix went in with the next apply, after a pull, and the following plans were clean.

Still open as an idea, not done: make the apply refuse to run unless the checkout is at `main`'s
head. That would turn a written rule into one that does not depend on remembering it.

**Update, 24 November: done.** Applies now go through a small wrapper script that fetches, then
refuses unless the working tree is clean, the branch is `main` and it is exactly `main`'s head: not
behind, and not ahead with unmerged commits either. Only then does it plan to a file, ask, and apply
exactly that plan, with the state lock taken. Each refusal was tested on the workstation before it was
trusted: an untracked file, another branch, a local unpushed commit, and a checkout one merge behind
(the case of this finding). All four were refused with the reason, and a clean `main` planned normally.

The first real run also found a bug in the guard itself. Its question was meant to name the commit
being applied ("Apply this plan from `<commit>`?"), and printed nothing there. The shell in use treats
`?` as part of a variable name, so the commit variable followed by a question mark was read as a
different, empty variable. The one fact the question exists to show was the one missing. It was fixed
the same evening. **A guard is code too, and is tested like code: by running it, not by reading it.**

## The general lesson

**A plan from a stale checkout is right about the wrong code.** Any tool that reports a difference
reports it between the inputs it was given, and "no difference" is indistinguishable from "wrong
input". Before trusting an empty diff, ask what it was a diff *of*.

### The professional twin

A deployment from a branch that never got the hotfix. A migration run from a laptop one release
behind. A compliance scan against last month's baseline. Each reports success, accurately, about
something other than what you meant. **Name which copy you mean**, and check it before reading the
result.
