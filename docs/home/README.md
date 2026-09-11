# The home side

The network does not stop at the server VLAN. Cameras, the automation platform, the IoT hub and the
client devices are part of the same design, and most of the segmentation decisions elsewhere in
these docs exist because of them.

## What is documented here

- **Why IoT and cameras are isolated**, and what "isolated" is actually enforcing — which direction
  is blocked, what the exceptions are, and how each one was verified rather than assumed.
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
