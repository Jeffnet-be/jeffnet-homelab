# jeffnet-homelab

A homelab run like a small production estate, and a record of what went wrong in it. Most of what
fails in a small estate does not look broken.

The build took weeks. Going back through it service by service found more than the build did, and
almost all of it was in what things leave behind rather than in the things themselves. Those
findings are the point of this repository.

> **Scope note.** This is documentation, not the estate. The infrastructure code lives in a private
> repository; nothing here is generated from it and nothing here is applied to anything. Internal
> names, addresses and identifiers are substituted throughout - see [SETUP.md](SETUP.md).

---

## Each finding, in one line

| Found | Finding | What it taught |
|---|---|---|
| 24 Nov 2026 | [The backup that trusted whoever answered](docs/findings/2026-11-24-the-backup-that-trusted-whoever-answered.md) | Switching a check on is the easy half; the hard half is the first thing it trusts. |
| 17 Nov 2026 | [The check that was off twice](docs/findings/2026-11-17-the-check-that-was-off-twice.md) | A security setting is the value that wins, not the value you wrote. |
| 16 Nov 2026 | [The reply that never left](docs/findings/2026-11-16-the-reply-that-never-left.md) | A shortcut that works can still be wrong in two ways at once. |
| 14 Nov 2026 | [The reboot the plan did not mention](docs/findings/2026-11-14-the-reboot-the-plan-did-not-mention.md) | A plan describes state, not uptime. |
| 14 Nov 2026 | [The plan that said No changes](docs/findings/2026-11-14-the-plan-that-said-no-changes.md) | A plan from a stale checkout is right about the wrong code. |
| 26 Sep 2026 | [The backup that was too complete](docs/findings/2026-09-26-the-backup-that-was-too-complete.md) | A backup that is too complete is a secret store you did not know you were running. |
| 22 Sep 2026 | [The checks that could not see the login](docs/findings/2026-09-22-the-checks-that-could-not-see-the-login.md) | An allow-list is not authentication, whatever it is called. |
| 2 Sep 2026 | [The guard that could not fail](docs/findings/2026-09-02-the-guard-that-could-not-fail.md) | A control is untested until it has been seen refusing something. |
| 2 Sep 2026 | [The door that was closed on purpose](docs/findings/2026-09-02-the-enrolment-window.md) | A hardening step that closes a path must say what it will cost later. |
| 24 Aug 2026 | [The backup that never existed](docs/findings/2026-08-24-the-backup-that-never-existed.md) | "We have backups" is a claim about existence, not about coverage or recovery. |

The rest of this page is the longer argument. The [findings index](docs/findings/README.md) has
the full list and the format each one follows.

---

## The thesis

Here is what failure looked like in this lab:

- A backup storage that had never carried a byte was *fully declared*.
- A mail notification path was *trying the whole time*, bouncing to an address nobody read.
- A monitoring stack fired seven real alerts into a channel nobody was subscribed to.
- A delivery task copied nothing, twice, from two different wrong paths, and reported success both
  times.
- A guard compared a list against an empty one, produced by a pipeline whose last command could
  not fail.
- An enrolment port was closed on purpose, correctly, and nothing recorded that the estate had
  stopped being able to grow.
- A config backup committed, for weeks, one file whose entire contents were the device's error
  message.

Exactly one of those looks like an outage. The rest look like green.

The inverse needs the same attention. Reading the private repository end to end, rather than the
documents describing it, produced twelve findings. Three of them said a document was wrong, and two
of those said the estate was *better* than recorded. A document wrong in the safe direction is not
harmless: work gets queued that is already finished, and a control believed absent is a control
nobody thinks to test.

Every rule in [the principles](docs/principles/README.md) is there because one of these happened
here, and was found by reading the machine instead of the documentation.

---

## Where to start

| If you want | Read |
|---|---|
| **What went wrong, and how it was found** | [docs/findings/](docs/findings/README.md) |
| The rules that came out of it | [docs/principles/](docs/principles/README.md) |
| The cluster, Terraform, Ansible, credential scopes | [docs/platform/](docs/platform/README.md) |
| The services, and how one is declared | [docs/services/](docs/services/README.md) |
| How a service is deployed and reviewed | [docs/operations/](docs/operations/reviewing-a-service.md) |
| The home side - IoT, cameras, automation | [docs/home/](docs/home/README.md) |
| Code you can lift | [examples/](examples/README.md) |

---

## The environment

**Network.** Seven VLANs behind a firewall that is both the gateway and the only inter-VLAN policy
point, with a managed switch carrying tagged trunks. The segmentation write-up is withheld until
each boundary has been read off the device rather than off notes; [the home page](docs/home/README.md)
says why.

**Platform.** Three Proxmox nodes in a cluster. Every guest is declared in Terraform, and Ansible
configures them from a dedicated controller. Guests are containers unless the workload needs
something that belongs to the kernel. [More](docs/platform/README.md).

**Ingress.** One reverse proxy, one wildcard certificate, one split-horizon wildcard DNS record. A
service is declared once, and its proxy route, dashboard card, uptime check, metric targets and
firewall rules are derived from that entry. Adding a service is an entry, never a block; deleting
the entry removes all of them. [More](docs/services/README.md#the-contract).

**Guest firewall.** Every service guest drops unsolicited inbound traffic at the hypervisor. Its
allow-list is generated from the same declaration, and only DNS and the ingress are open to every
client. It rolled out from one canary guest to the ingress, a wave at a time.

**Observability.** Probes at three layers, SNMP network monitoring, config backup, log viewing, and
host-based intrusion detection on every guest. Every detector ends in one chat through one bot.
The nightly runs in check mode and pages on drift as well as on failure.

**Credentials.** Five actors, five scopes, no overlap: the workstation authors, the controller
executes, the scheduler runs on a timer, CI only plans and stores no credential at all, and an AI
coding assistant opens pull requests from a session with no secrets and no route to the lab. A
person reads every plan, merges and applies. [More](docs/platform/README.md#five-actors-five-credential-scopes-no-overlap).

**Git.** The hosted repository is canonical. An on-prem server keeps a read-only mirror of every
repository for the day the hosted one is unreachable, and alerts when a mirror falls behind its
source rather than when the sync job fails.

**Home.** Automation, cameras and IoT on their own VLANs, documented as a design rather than a floor
plan. [More](docs/home/README.md).

---

## The stack

Named so the design can be read. Versions and addresses are not published, and network-device vendors
are not named; a careful reader can still infer some from the details, so that is a courtesy, not a control.

| Layer | Software |
|---|---|
| Hypervisor | Proxmox VE, three-node cluster |
| Infrastructure as code | Terraform (`bpg/proxmox` provider), remote state in cloud object storage |
| Configuration | Ansible, run from a controller and on a schedule by Semaphore |
| CI | GitHub Actions: plan only, workload identity federation, actions pinned by commit |
| Ingress and identity | Caddy, Authelia, two Pi-hole resolvers |
| Remote access | Tailscale |
| Observability | Prometheus, Grafana, Gatus, Homepage, LibreNMS, Oxidized, Dozzle |
| Security | Wazuh, Proxmox guest firewall |
| Alerting | One Telegram bot; Healthchecks.io for scheduled runs and drift |
| Records and data | NetBox, BookStack, Paperless-ngx, Nextcloud, Jellyfin |
| Git | GitHub (canonical), Gitea (read-only mirror) |
| Home | Home Assistant |

---

## How a service is declared

![A service is declared once, in its host's inventory entry, and everything else is derived from it: the reverse-proxy route, the forward-auth policy, the dashboard card, the uptime check, the metric and probe targets and the guest firewall rules. Deleting the entry removes the derived pieces by itself; the intrusion-detection registration, host keys and backup archives are not derived and outlive it.](diagrams/service-derivation.svg)

## How an alert reaches a person

![Six kinds of detector (metrics alerting, uptime checks, hypervisor notifications, network monitoring, intrusion detection and the nightly playbook runs) all end in one chat on one phone, hosted outside the estate. The uptime checker has its own dead-man heartbeat, scheduled jobs are watched for failure, silence and drift, and every path has been proven end to end.](diagrams/detectors-to-pager.svg)

---

## About

Built, run and written up by [Jeffrey](https://github.com/Jeffnet-be), as hands-on practice for
infrastructure as code and security operations. The curated case study, and my other work, are at
[jeffnet.be](https://jeffnet.be/homelab). Corrections and questions are welcome as issues.

## Licence

Documentation under CC BY 4.0. Code in `examples/` under the MIT License. See [LICENSE](LICENSE).
