# jeffnet-homelab

A three-node Proxmox estate, built with Terraform and Ansible, then **reviewed** — service by
service, against nine questions. This repository is the write-up: the architecture, the operating
patterns, the reusable code shapes, and the findings.

The findings are the point. The build took a few weeks; the review pass has produced more than the
build did, and almost all of it in what services leave behind rather than in the services
themselves.

> **Scope note.** This is documentation, not the estate. The infrastructure code lives in a private
> repository; nothing here is generated from it, and nothing here is applied to anything. Internal
> names, addresses and identifiers are substituted throughout — see [SETUP.md](SETUP.md).

---

## The estate in one page

**Platform** — three Proxmox VE nodes in a cluster, all guests declared in Terraform across two
states (VMs, services). Node-local VM templates. Guests are LXC except where the workload forbids
it — the SIEM needs `vm.max_map_count`, which is not namespaced, so it is a VM.

**Configuration** — Ansible, roles cloned from git on a dedicated controller. Two plays run
estate-wide on a schedule: a baseline (`common`) and endpoint-agent enrolment. Everything else is
per-host and derived from a single declaration.

**Ingress** — one Caddy instance, one site block, one wildcard certificate issued over DNS-01,
one split-horizon wildcard DNS record. Services are declared once in host vars and *derived* into
proxy routes, dashboard cards, uptime checks and metric targets. Adding a service is one entry,
never a block. Deleting one removes all five consumers by itself.

**Observability** — Prometheus with three probe layers, a metrics stack, network monitoring by
SNMP, config backup by SSH, log aggregation, and a SIEM with agents on every guest. Detectors
publish to a single self-hosted push service with one write-only identity each, and one phone
subscription. Many detectors, one pager.

**Credentials** — three roles, three scopes, no overlap. The workstation authors and can write to
both remotes. The controller executes: read-only clone, holds the vault, pushes nothing. The
scheduler executes on a timer: read-only deploy key scoped to one repository, vault delivered out
of band, SSH reaching one inventory group and pinned by source address.

**Network** — six VLANs behind a firewall doing all inter-VLAN policy, with seven rules total into
and out of the server VLAN. A mesh VPN provides remote access and a subnet route, which is
deliberately a hole in the segmentation and is written down as one.

---

## Where to start

| If you want | Read |
|---|---|
| The design and why it landed there | [docs/architecture/network.md](docs/architecture/network.md) |
| How a service gets deployed, and reviewed | [docs/operations/reviewing-a-service.md](docs/operations/reviewing-a-service.md) |
| The interesting part | [docs/findings/](docs/findings/) |
| The rules that came out of it | [docs/principles/README.md](docs/principles/README.md) |
| Code you can lift | [examples/](examples/) |

---

## The thesis

Most of what fails in a small estate does not look broken.

A backup storage that had never carried a byte was *fully declared*. A mail notification path was
*trying the whole time*, bouncing to an address nobody read. A monitoring stack fired seven real
alerts into a channel nobody was subscribed to on a device that interrupts. A delivery task copied
nothing, twice, from two different wrong paths, and reported success both times. A guard compared
a list against an empty list produced by a pipeline whose last command could not fail.

Exactly one of those looks like an outage. The rest look like green.

The habits in [docs/principles](docs/principles/README.md) all exist because one of them happened
here, on real hardware, and was found by reading the machine instead of the documentation.

---

## Licence

Documentation under CC BY 4.0. Code in `examples/` under the MIT Licence. See [LICENSE](LICENSE).
