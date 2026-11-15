# The reboot the plan did not mention

**Found:** 14 November 2026 · **Class:** an in-place change with an unannounced side effect.

> **In short.** Turning a container's firewall flag on - or changing its DNS servers - reboots it, and
> the plan shows only an attribute changing in place. Read in advance, that turned a firewall
> rollout into a scheduling problem. The one time the precondition was not checked, it rebooted the
> scheduler under its own running job, which then showed as *running* until someone looked.

---

## The symptom

A Terraform plan for the guest firewall rollout read like a list of harmless edits: on each guest,
one attribute of the network interface going from `false` to `true`, marked *update in-place*. No
`# forces replacement`, no destroy. Nothing in the plan said any guest would restart.

## Reading the provider instead of the plan

In-place is a statement about the resource's identity, not about its uptime. The provider's update
path for containers collects a *reboot required* flag while it applies changes, and restarts the
container at the end when it is set. Two of the attributes that set it mattered here:

- the network interface's **firewall flag** - the very thing the rollout was turning on, guest by
  guest;
- the guest's **DNS servers**.

So every guest joining the firewall would reboot on apply, and the plan would never say so. Once that
was confirmed in the provider's source, it shaped the rollout:

- **The two resolvers went in separate changes, a day apart.** The provider applies changes in
  parallel, so both resolvers in one apply would have dropped DNS for every client at the same
  moment.
- **Every pull request named the guests that would reboot**, so the person applying could pick the
  moment.
- **The scheduler's guest was to be applied outside a scheduled run.** The pull request that
  included it said so.

## The one that was missed

The scheduler's guest joined the firewall in the same wave as the home-automation host, and that pull
request said to apply it outside a scheduler run. The apply happened while the scheduler was busy
with a manual re-run of the previous night's failed job.

The guest rebooted under it. The run died with the container. When the scheduler came back, it
showed the run as **running**, and kept showing it: the process that would have recorded the end was
gone, and nothing else knew there had been one. Stopping it in the UI could only mark it stopped in
the scheduler's database, which was all there was left to do. Re-run, it completed clean.

A smaller instance of the same shape, the same night: the first run of a new mirror job was cut off
because the play that started it also recreated the container it was running in. Moving the first
run to after the stack was up, behind a readiness check, fixed it.

## The cause

Two gaps, one in the tool and one in the procedure:

- **"In-place" reads as "no restart", and is not.** The plan's vocabulary describes what happens to
  the resource in state. What the provider does to the running guest to get there is not part of it.
- **The precondition was written, not checked.** "Apply outside a scheduler run" was in the pull
  request. Nothing made anyone look at the scheduler's task list before applying, and a run started
  by hand is not on any schedule to remember.

## What changed

- **A change that reboots a guest is named in its pull request**, with the reason, for every guest.
- **Any apply that reboots the scheduler's guest runs when the scheduler is idle** - checked in its
  task list, not inferred from the schedule.
- **Never both resolvers in one apply** stays a standing rule, not a rollout note.

## The general lesson

**A plan describes state, not uptime.** Ask of every in-place change what the provider has to do to
the running thing to make it true. The answer is in the provider's code or documentation, not in
the plan.

And: **a host that runs your jobs cannot be restarted under them.** The job does not fail - it
simply stops, and the system that should have recorded the end is the system that was restarted.
Expect a ghost, and know how to clear it, before you reboot the thing that keeps the records.

### The professional twin

A configuration push that bounces a service as a side effect. A patch window that reboots the job
scheduler mid-batch. A "non-disruptive" setting that restarts an agent. Ask of every change: **what
restarts to make this true, and what was running there at the time?**
