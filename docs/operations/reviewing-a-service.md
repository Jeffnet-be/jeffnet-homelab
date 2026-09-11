# Reviewing a service — the nine questions

The build list finishing is not the end of the work. Every service that was deployed was deployed
correctly *at the time*, by someone learning it, and then left alone. The review pass goes back
through each one and asks the same nine questions.

It has produced more findings than the build did.

## The questions

1. **Used, or vestigial?**
2. **In the backup scheme — and has it ever been restored from?**
3. **Monitored by what, and does that thing page anyone?**
4. **Is its own admin board green?**
5. **Credentials seeded from the vault, or set by hand?**
6. **Is it in the contract** — declared once, with its consumers derived?
7. **What is its retention policy?**
8. **What is its failure domain?**
9. **Is it pinned, and can it be upgraded?**

## What running it actually teaches

**The first question can end the review.** One service failed it in ten minutes: its database had
not been written to in ten days, its WAL was empty, it had no attachments and no configuration. It
had been deployed once to learn the deployment and never adopted. It was destroyed rather than
carried — and the code kept in the repository, because the knowledge was worth keeping and the
running instance was not.

**The second question is the one that fails.** The git service passed eight of nine and failed
*restorable*. Its application-consistent dump is on a timer, its contents verified — and it has
never been restored from, which is a different claim entirely. Expect this finding once per service
with a database inside a guest: a file-level copy of a live SQLite database is a copy of a file
being written to, and so is a guest backup of a running search index in a named volume. At the third
instance it stops being a per-service finding and becomes an estate rule.

**A review finds more in what a service leaves behind than in the service.** The ten-minute review
above led, via its own orphaned agent registration, to four further findings over the following
hours — including two defects in a guard that had been written the week before. Each was only
reachable through the one before it.

## Deploying a service — the pattern that came out of this

1. Read the vendor's documentation first, then the image. An image's defaults are part of your
   configuration, and its uid and gid are rarely what you assume.
2. Decide the config surface before the first `up`. A container's environment is fixed at creation.
3. Size and place the guest against actual hardware and failure domains — and move it while it is
   still empty.
4. Seed credentials from the vault *before* the first start, and check what any first-init guard is
   keyed on. Most of them are keyed on something that exists whether or not the seeding worked.
5. Enrol it in the endpoint agent deliberately, and confirm both guards.
6. Clear the application's own admin board, and write down whatever you are choosing to accept.
   Every new system arrives with its own permanent reds.
7. Prove the path before you populate the system.
8. After the play, wait a full monitoring interval and look at the board.
