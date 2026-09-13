# Network and ingress

## Segmentation

Six VLANs behind a single firewall that is both the gateway and the inter-VLAN policy point, with a
managed switch carrying tagged trunks to the three nodes.

| VLAN | Purpose |
|---|---|
| 300 | Management - hypervisors' out-of-band, switch controller, firewall |
| 400 | Guest |
| 500 | Clients |
| 700 | IoT |
| 800 | Cameras |
| 900 | Servers - every guest in the estate |

Seven firewall rules govern the server VLAN in total:

- **In from clients:** DNS to the two resolvers; HTTP, HTTPS and HTTP/3 to the reverse proxy.
- **Out to management, proxy only:** the proxy reaches the switch controller and the firewall's
  management interface, nothing else does.
- **Out to management, monitoring:** the network monitor reaches the management range on SNMP and
  ICMP.
- **Out to management, config backup:** the config-backup host reaches the firewall on SSH.
- **In from management:** the switch controller sends syslog to the log collector.

Two rules that were expected and are not needed, each verified rather than assumed:

- **NFS between nodes needs no rule** - the nodes share a VLAN, so access control is entirely in
  `/etc/exports`.
- **Endpoint agents need no rule** - every agent is in the server VLAN. The compose template's
  header comment records that a rule becomes necessary the moment an agent is placed outside it.

### Things that are true about this firewall and are not obvious

- **`allowaccess` is not a firewall rule.** It is evaluated against the interface that owns the
  destination address, which is a different question from whether policy permits the traffic.
- **The mesh VPN is not policed by any of these rules.** A subnet router is a hole in the
  segmentation by design - written down as a deliberate absence rather than tolerated as one.
- **No IPv6 routing anywhere**, link-local only. Also deliberate, also written down.
- **A policy drop is a timeout. A reset means the packet arrived. A refusal means nothing was
  listening.** Three different failures that get described with the same word.
- **A container reaching its own host's LAN address is not source-NAT'd to that address** - sshd
  sees the docker bridge, so a source-address pin correctly denies it. This is why the scheduler
  cannot check itself, which is also the right answer: the check for a thing must not depend on the
  thing.

## DNS, TLS and ingress

Settled early and deliberately not re-litigated:

- **One wildcard DNS record**, split-horizon, resolving `*.lab.example.internal` to the reverse
  proxy. **Never a per-service record.**
- **One wildcard certificate**, issued over DNS-01 against the public zone and renewed by the proxy
  itself. No per-service certificate, no HTTP-01, nothing exposed inbound to obtain it.
- **One site block** covering the whole wildcard - a site-level directive therefore covers every
  service at once, which is how HSTS got applied estate-wide in a single line and confirmed in the
  served response rather than in the config file.
- **Services are declared once** in host vars and derived into proxy routes by a `contract` role.
  Adding a service is an entry, never a block.

Two consequences worth stating:

- **Identity is the string you registered with.** A service reachable at `cloud.` will reject
  requests for `nextcloud.` at the application layer, not the proxy - and the rejection is a 400,
  not a 403, because it is a string comparison rather than an authorisation decision.
- **A wildcard cannot express an SSH endpoint.** The git service's SSH port is therefore addressed
  directly in configuration - the one deliberate exception, recorded as such so that nobody
  "corrects" it later.

## Failure domains

The design was sound and the placement was not. Everything that mattered on a single node meant one
loose cable took ingress (and therefore every hostname), the identity provider, the Ansible
controller, remote access - **and the monitoring host, so nothing could report the outage.**

Placement includes the path a message takes. Detectors that live on the node they watch produce
excellent data about their own failure and no way to deliver it.

The diagnosis is worth keeping too: `Destination host unreachable` **from the firewall** means it
could not ARP. Every failing address was a guest on one node, and executing a ping from inside a
container on that node succeeded - which put the fault northbound of the physical interface and
ended the investigation in about a minute. A failed ARP entry is not proof of a broken path; a
successful ping is proof of a working one. Nothing logged a link change at any layer.
