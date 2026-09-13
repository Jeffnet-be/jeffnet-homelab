# The services

Around twenty services, each in its own guest, each deployed the same way. The value is not in the
list — it's in the fact that adding one is a single declaration, and that removing one is the same
declaration deleted.

For *how* a service gets reviewed once it exists, see
[the nine questions](../operations/reviewing-a-service.md).

---

## What runs

Grouped by what they're for, which is also how the dashboard is organised — on meaning rather than
on convenience.

**Ingress and identity** — reverse proxy, identity provider, two DNS resolvers.

**Automation** — git, an automation controller, a scheduler running plays on a timer, a CI runner.

**Observability** — metrics and alerting, uptime checks, a network monitor, a log viewer, a push
notification service, network-device config backup.

**Security** — host-based intrusion detection across every guest.

**Documentation and data** — a documentation wiki, a source of truth for infrastructure records, a
document management system, file sync and share, media.

**Home** — the automation platform, bridging to the isolated IoT and camera VLANs.

Deliberately absent: a mail relay, which means account mail from one application does not work and
that is recorded rather than discovered; and an in-repo secrets manager, because credentials live in
a commercial password manager that already has a mobile client and a recovery story.

---

## The contract

![One declaration, four consumers](../../diagrams/service-derivation.svg)

A service is declared **once**, in the host's variables, with its name, port and a one-line
description. A `contract` role derives everything else: the proxy route, the dashboard card, the
uptime check, and the metrics and probe targets.

**Add an entry, never a block.** The rule that makes this work is that no consumer is ever written
by hand — if the dashboard needs a card, the card comes from the declaration or it doesn't exist.

The payoff shows up on deletion. Retiring one service removed its proxy route, its dashboard card,
its uptime check and its metric targets from a single deleted entry, with no other change anywhere.
**Render, don't reconcile.**

What *didn't* drop is the more useful half: the intrusion-detection agent registration, host-key
entries, and the backup archives — which now outlive the guest they protected, with nothing deciding
when they should stop. **Derived consumers drop by themselves; enrolled ones persist and show as
broken rather than absent.**

---

## Conventions that took a mistake to learn

**One compose stack per host, and the stack directory is named for the host's role, not the
service.** Two services that share a host share a stack and a directory. A path variable named after
a service is therefore a category error, and reading one cost two deliveries that reported success
while copying nothing. **Read what a variable resolves to on the target, not what its name implies.**

**`.env` substitutes; `env_file:` injects.** The first expands `${VAR}` inside the compose file. Only
the second puts variables into the container's environment. A stack once rendered thirty-nine
variables into an `.env` that the application never read — eleven were being consumed because they
happened to be substituted, and the rest were documentation.

**A container's environment is fixed at creation.** A stale container carries the environment it was
born with, however many times you rewrite the file.

**Prefer the module that shells out to the vendor's CLI over the one that imports its SDK.** The
SDK-backed container modules break against a common HTTP library upgrade with an error that names a
URL scheme rather than a dependency conflict. The CLI-backed module has no such coupling.

**Mount configuration directories, not single files.** A single-file bind mount breaks the moment
the application rewrites the file rather than editing it in place.

**Read the image before writing the template.** User and group IDs differ between images, and an
image's defaults are part of your configuration whether or not you chose them. One image ships
without timezone data, so a timezone variable is accepted and ignored — which is why the estate runs
UTC, written down rather than tolerated.

---

## What a retirement looks like

One service was deployed once to learn how to deploy it and then never adopted. Its database had not
been written to in ten days; there were no attachments, no exports, no configuration. It had a
backup slot, an uptime check, an image to keep current, and a dashboard card describing it as
something it was not.

It was destroyed with `terraform destroy`, and **the code stayed in the repository** — the knowledge
was worth keeping, the running instance was not.

> **A proof of concept that was never retired is indistinguishable from production to every system
> that watches it.** It consumes backup capacity, a monitoring slot, upgrade attention and a line on
> the dashboard, and it does all of that while providing nothing.

The first of the nine questions — *used, or vestigial?* — can end a review in ten minutes. It is
worth asking first for exactly that reason.
