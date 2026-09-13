# The platform

Three hypervisor nodes in a cluster, every guest declared in Terraform, every host configured by
Ansible. The interesting parts are not the tools - they are the separation of who may author, who
may execute, and who may read what.

---

## The cluster

Three nodes, deliberately **asymmetric**. They were acquired at different times and have different
core counts and memory, and pretending otherwise would mean placing workloads against an average
that doesn't exist. Each node has a role, and every workload is placed against actual hardware:

- **Ingress and control plane** - the reverse proxy, identity, the automation controller, shared
  storage for one peer.
- **Observability** - the metrics stack, the network monitor, a second resolver, and the push
  service that every alert terminates on.
- **Capacity** - the largest node, carrying the workloads that actually consume something, plus
  shared storage for the other two and the primary remote-access route.

Three nodes is the smallest cluster that can hold quorum through the loss of one, which is the whole
reason for the third.

**Guests are containers except where the workload forbids it.** One service runs in a full VM
because it needs a kernel parameter that is not namespaced, and a container therefore cannot own it.
That is the general test: containers until something the workload needs belongs to the kernel rather
than to the guest.

### Failure domains are enumerated, not assumed

The design was sound and the placement was not. A single node once held ingress (and therefore every
hostname in the estate), identity, the automation controller, remote access - **and the monitoring
host, so nothing could report the outage.** One loose cable took all of it.

Placement includes the path a message takes. A detector that lives on the node it watches produces
excellent data about its own failure and no way to deliver it.

---

## Terraform

A **monorepo**, with a separate project folder per concern and **a separate state per project**.
Splitting by concern rather than by environment keeps the blast radius of an apply small, and it
means a mistake in one area cannot plan against the others.

The split matters for a reason that isn't obvious until it bites: **`terraform state list` covers one
state.** Ask "is everything under Terraform?" in a multi-state repo and you get an answer scoped to
whichever directory you happened to be standing in.

State lives in **remote object storage**, and CI authenticates to it with **OIDC** - a short-lived
federated token per run rather than a stored credential. There is no cloud secret in the repository
or in the CI configuration to leak or rotate.

### Things that force replacement, learned the hard way

Read every plan: **`# forces replacement` first, then the resource key, then the counts.** The ones
that have caught me:

- **SSH keys in the guest initialisation block.** Changing them forces replacement of *every*
  container. Terraform is the wrong layer for SSH keys - they belong to configuration management,
  which can converge them without rebuilding the guest.
- **Anything inside a clone block**, and the node assignment.
- **Purging on destroy** strips the guest's ID out of backup jobs and high-availability
  configuration as a side effect.

And one that isn't a replacement but is worse: **backup archives are keyed by guest ID.** A freed ID
reused by a new guest inherits the old one's archive series and prune policy, and a restore prompt
will offer both. Take the next free number.

**Move a workload while it is empty.** Migrating a service between nodes is trivial the day it is
deployed and a project three months later.

---

## Ansible

Roles live in the same repository as the Terraform, cloned to a dedicated controller. Two things
are worth copying:

**A role that must run everywhere gets its own play - and a play that must run everywhere gets a
schedule.** The baseline role runs against every host in the estate as a standalone play. The
endpoint-agent play does the same. The second one existed and was *correct* for weeks while never
having been scheduled, so nothing re-ran after a guest was rebuilt, and a host silently lost its
agent. Correct and never invoked is indistinguishable from absent.

**A scheduled check inherits its runner's reach.** The scheduler cannot check itself, and the
hypervisors are not in its inventory. That is written down next to the green result, because
otherwise a passing nightly is read as covering the estate.

---

## Three roles, three credential scopes, no overlap

This is the part I would keep in any environment, at any size.

| | Authors | Executes on demand | Executes on a schedule |
|---|---|---|---|
| **Who** | The workstation | The controller | The scheduler |
| **Repository access** | Write, both remotes | Read-only clone | Read-only deploy key, one repository |
| **Holds the vault?** | No | Yes, with the password | Delivered out of band, mounted read-only |
| **Can push?** | Yes | No | No |
| **SSH reach** | - | The estate | One inventory group, pinned by source address |

**The machine that executes the code should not be able to rewrite it.** That used to be a
convention, which meant it existed only while a person was doing the work. It is now a read-only
deploy key, enforced by the git server.

Two supporting decisions:

- **The vault is not in git**, Git's purpose is to replicate a file to every copy of the repository, and a secret's requirement is that it doesn't. Those are opposite properties, and no amount of care about which remotes exist changes which one git is for. The vault is delivered out of band and mounted read-only.
- **A deploy key beat a personal access token** on a documentation question. With two-factor
  enabled, the vendor's docs do not settle whether a token bypasses the second factor for
  git-over-HTTP, nor whether a read scope is sufficient to deny a push - two ambiguities on exactly
  the properties that decide whether a credential is really read-only. A deploy key has neither
  ambiguity: SSH never touches the account's password path, and read-only is enforced per repository.

---

## A fresh clone is a completeness test

The scheduler's first real run failed five times in a row, each on something that had worked for
months because one machine had always run the code:

- a key store that wanted the key in a different encoding
- command-line arguments that had to be passed one per field
- the Ansible configuration being read from the working directory, which was a different directory
- a task delegated to localhost inheriting privilege escalation, and finding no `sudo` in the
  container
- a file living in a directory that had never been committed

None of those were bugs in the code. They were **dependencies that existed on exactly one machine**,
invisible for as long as only that machine ran anything. Running the repository from a clean clone
on a different host found all five in an afternoon.

If you take one practice from this page: **run your own repository somewhere it has never run
before, on purpose, before something forces you to.**
