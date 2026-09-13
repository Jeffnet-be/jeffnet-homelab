# One service contract, two sources, seven guards

Most homelab repositories add a service in five places: a proxy route, a forward-auth rule, a
dashboard card, an uptime check, and a metrics target. Five places to edit, five to keep in sync,
and — the part that actually bites — five to remember on the way out.

This role builds one list, and every renderer consumes it. **No template ever touches `hostvars`.**

See [`tasks/main.yml`](tasks/main.yml) for the role.

---

## The declaration

In `inventory/host_vars/<host>.yml`, for anything Ansible manages:

```yaml
services:
  - name: docs
    dns_name: docs
    port: 6875
    auth: forward
    description: Documentation wiki
  - name: metrics
    dns_name: grafana
    port: 3000
    auth: internal
    health_path: /api/health
```

`name` is an **identifier** — it keys uptime history and derives the proxy matcher slug. `dns_name`
is what the FQDN is built from. They are separate on purpose, because the thing a service is called
and the name it answers to are not the same fact, and conflating them means you cannot rename either
one independently.

## The second source — the part worth copying

Not every service is a host you configure. Hypervisor UIs, a switch controller, a firewall's
management interface: all real services behind the same proxy, none of them Ansible hosts.

```yaml
# group_vars/all/proxy_external.yml
proxy_external_services:
  - name: hypervisor-a
    dns_name: node-a
    host: 10.0.0.11        # stated, because there is no ansible_host to read
    scheme: https
    port: 8006
    auth: forward
    tls_skip_verify: true
```

Same field contract, one difference: an external service **states its address**, where an inventory
service gets one from `ansible_host`. Pass 2 normalises both into `ip`, and after that line nothing
downstream knows or cares which source an entry came from.

The temptation is to keep two lists and have the proxy template loop over both. Don't — and the
reason is in the guards below.

## What gets derived

`fqdn`, `slug`, `scheme`, `port`, `upstream`, `auth`, `health_path`, `expect_status`,
`tls_skip_verify`. Derived **once**, here. Templates never re-derive, which is what stops the proxy
and the uptime check from disagreeing about a port.

One derivation worth stealing:

```jinja
'expect_status': item.expect_status | default(
                   'any(200, 302)' if (item.auth | default('none')) == 'forward' else '200'
                 )
```

A service behind forward auth answers an unauthenticated probe with a redirect, not a 200. Deriving
the expected status *from the auth mode* means the uptime check is right by construction. Written by
hand per service, this is the setting that gets copied wrong and leaves a permanent red dot on a
board — and **a monitor whose normal state includes red has already stopped working.**

## Seven guards, and why they need one list

| Guard | Catches |
|---|---|
| Contract not empty | A group with no members, or an unset external list — otherwise every renderer produces a valid, empty config |
| `name` unique | Silent collision in uptime history and proxy matcher slugs |
| `dns_name` unique | Two services claiming one FQDN. No error — the proxy serves whichever rendered last |
| `auth` value known | A typo rendering as **no protection at all** |
| Identity provider enabled if anything asks for forward auth | Publishing a service unauthenticated |
| Basic-auth hash present if anything asks for it | Same |
| External services have an address | An entry that would derive an upstream of `://:80` |

**The uniqueness guards are the argument for a single list.** A duplicate between two managed
services is findable by eye; a duplicate between a managed service and an unmanaged one is not
visible from either list, and no per-template loop could ever check it. Merging the sources is what
makes the check possible at all.

The auth guards are the same principle as a firewall's default deny: **a typo in a security-relevant
field must fail the run, not render as permissive.** `auth: forwrad` is not an unknown mode that
gets skipped — it's an unprotected service that looks configured.

## Two traps in the code

**`default([], true)` — the second argument matters.** A host with a `services:` key and nothing
under it yields `None`, not `[]`, and plain `default([])` does not substitute for `None`. Without
the boolean the aggregation fails on the first host that has the key and no entries, which is
usually a host somebody just added.

**`run_once: true` would be wrong here.** `set_fact` lands only on the host that ran the task, so a
contract built once would be undefined everywhere else. The role does no remote work at all, so
running it on every host costs nothing — the debug at the end carries the `run_once` instead, which
is the only part that should print once.

## A closing check

The final task prints the count by source and every slug. That exists so a run **states its own
coverage**: twenty-three services, nineteen inventory, four external. If that number drops, the diff
says so before anything renders — and a control that silently narrows its own scope is the failure
this whole repository is about.
