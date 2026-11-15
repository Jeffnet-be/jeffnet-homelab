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
- **Observability** - the metrics stack, the network monitor and a second resolver. Alerts leave the
  estate for a chat bot, so the pager does not share a failure domain with anything it reports on.
- **Capacity** - the largest node, carrying the workloads that actually consume something, plus
  shared storage for the other two, and the mesh-VPN exit node that remote clients name.

Two of the three advertise the same route into the server network, and it is worth being precise
about what that does and does not buy, because the mesh VPN has no concept of a preferred router:
overlapping advertisements are **equal candidates**, and failover between them takes seconds and
needs nobody's attention. The **exit node** is the opposite - the client names one, so it does not
fail over at all. Two mechanisms, one product, and only one of them is redundant. Calling either of
them "primary" imports a model the software does not have.

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
or in the CI configuration to leak or rotate. CI's identity may **read** every state and write none:
it plans without taking the state lock, because the lock is a lease on the object - which is a write.
Applies, from the workstation, always take it.

**The hypervisor firewall is a project of its own**, with its own state and its own token. It reads
each guest's node and ID from the guest project's state rather than repeating them, and it generates
every guest's rules from the same service declarations the reverse proxy uses (see
[the services page](../services/README.md#the-contract)). A firewall that needs a privilege nothing
else needs should not share a token with things that don't.

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

### Things that reboot instead, which the plan does not say

Some in-place changes restart a container, and the plan shows them as an ordinary attribute change:
the network interface's firewall flag and the guest's DNS servers among them. That turned the
firewall rollout into a scheduling problem as much as a rules problem:

- **Never reboot both resolvers in one apply.** The provider applies changes in parallel, so two
  resolvers in one change means every client loses DNS at once. The second one waited for its own
  change until the first had run a night behind its new rules.
- **A host that runs your jobs cannot be restarted under them.** Rebooting the scheduler's guest
  killed the run it was executing, and the scheduler showed that run as *running* until someone
  looked, because nothing was left to record that it had ended.
- **Name every reboot in the pull request**, so the person applying picks the moment.

[The write-up](../findings/2026-11-14-the-reboot-the-plan-did-not-mention.md).

And one that is about the operator, not the provider: **apply from a stale checkout and the plan
says "No changes"**. It is right - about the code it was given. The merged change simply isn't there,
and the next pull request's plan is where it shows up. Pull before every apply.
[The write-up](../findings/2026-11-14-the-plan-that-said-no-changes.md).

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

## Five actors, five credential scopes, no overlap

This is the part I would keep in any environment, at any size.

| | Authors | Executes on demand | Executes on a schedule | Plans only |
|---|---|---|---|---|
| **Who** | The workstation | The controller | The scheduler | CI |
| **Repository access** | Write, one remote | Read-only clone | Read-only deploy key, one repository | Checkout per run |
| **Holds the vault?** | No | Yes, with the password | Delivered out of band, mounted read-only | No |
| **Can push?** | Yes | No | No | No |
| **SSH reach** | - | The estate | One inventory group, pinned by source address | - |
| **Credential lifetime** | Long-lived | Long-lived | Long-lived | **Nothing stored at all** |

**The fourth column was the last one to be got right, and it is the one worth copying.** CI only ever
plans, so it needs the least - and for a while it held the most awkward thing in the estate: a static
mesh-VPN key with a date on it, three lines below a cloud login that was already federated. One job,
two identity models, and the better one was already sitting above the worse one. **The inconsistency
inside the single file was a stronger signal than either half read on its own.**

It went through two steps. First an OAuth client scoped to mint one kind of short-lived key for one
tag - better, but still a stored secret. Now **workload identity federation for both**: the job's own
signed identity token is exchanged for an ephemeral VPN node with a single tag, and the VPN policy
lets that tag reach **one port on one host** - the hypervisor API, which proxies for the other nodes.
The policy carries tests that refuse a save if the tag could reach anything more. The node removes
itself when the run ends. Its hypervisor token can read the estate and create, start or delete
nothing; its cloud identity can read state and change none. **That column holds no credential at
all**, so there is nothing in it to leak, rotate or expire.

Two details from the OAuth-client stage that cost time, and still apply to any machine identity:

- **A permission written in terms of who *people* are grants a machine identity nothing.** The tag
  the client needed was owned by the administrators group - which looked correct, and had been
  correct for months, because every caller until then had been a person. An OAuth client is not a
  member of a user group; it authenticates *as the tags it was granted*, so the tag has to be owned
  by a tag. The error said "invalid or not permitted" - two different faults in one string, and only
  reading the policy file split them.
- **Scope the minting narrowly.** The first attempt granted the client the ability to create OAuth
  applications: a credential that can mint credentials is not a narrower version of the thing you
  wanted. Caught before it was saved.

**The fifth actor, with less than CI.** An AI coding assistant opens branches and pull requests from
a cloud session that holds no secrets and has no route to the lab. It cannot change the CI workflow
(a repository rule refuses its pushes there), it never merges, and it posts every project's plan
summary on the pull request - because a green check proves that planning *succeeded*, not what it
planned. A person reads the plans, merges, and applies.

**The machine that executes the code should not be able to rewrite it.** That used to be a
convention, which meant it existed only while a person was doing the work. It is now a read-only
deploy key, enforced by the git server.

Two supporting decisions:

- **The vault is not in git.** Git's purpose is to replicate a file to every copy of the repository,
  and a secret's requirement is that it doesn't. Those are opposite properties, and no amount of
  care about which remotes exist changes which one git is for. The vault is delivered out of band
  and mounted read-only.
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
