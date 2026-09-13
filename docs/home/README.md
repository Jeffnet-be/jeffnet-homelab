# The home side

The network does not stop at the server VLAN. Cameras, the automation platform, the IoT hub and the
client devices are part of the same design, and most of the segmentation decisions elsewhere in
these docs exist because of them.

> **Status, 11 Sep 2026.** This page is a scope, not yet a write-up. An export of the live firewall
> policy did not match my own notes about what these VLANs are permitted to do — and it diverged in
> *both* directions: some boundaries turned out tighter in practice than the rules require, others
> looser than the notes claimed. Nothing here asserts a boundary until it has been read off the
> machine rather than off a document. That reconciliation is the next piece of work, and it is the
> reason this section is short.

## What will be documented here

- **What separating IoT and cameras is actually enforcing** — which direction is blocked, what the
  exceptions are, and how each one was verified rather than assumed. Separate VLAN and *isolated*
  are two different claims, and only the first one is free.
- **The gap between what a rule permits and what the devices do.** These are different measurements
  taken from different places, and each can be the flattering one. A rule that permits something
  nothing uses is a latent capability; a device doing something no rule mentions is a gap in the
  notes. Both are worth writing down, and neither can be read off the other.
- **The automation platform's failure domain.** It runs on the same cluster as everything else,
  which is a decision with consequences worth stating.
- **How the home side is backed up**, and whether that has ever been restored from.
- **What a device is trusted to do**, as opposed to what its vendor assumes it may do. Most consumer
  kit expects a flat network and a permanent path to a vendor cloud; the interesting work is in
  deciding which half of that to grant.

## What is deliberately not documented here

Camera **placement** and coverage, the layout of the building, which entrances have access control,
and any automation that encodes when the house is occupied.

Those are physical-security facts about a specific address rather than technical ones, and they do
not become safe by being interesting. The rule is the same one applied to the service inventory
elsewhere in this repository: **publish the pattern, not the map.** A design that only works while
its diagram is secret is not a design worth writing up anyway.
