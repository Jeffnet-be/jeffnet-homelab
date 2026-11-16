# About this repository

## What it is

A documentation repository, written by hand. It is **not** a mirror, export or scrub of the private
infrastructure repository, and shares no git history with it. That is deliberate: a sanitised copy
of a private repo is one bad rebase away from publishing something it shouldn't, and git history is
permanent. Everything here was written into this repository directly.

## Substitutions

The following are substituted consistently throughout. The *shape* of the design is accurate; the
identifiers are not.

| Real | Here |
|---|---|
| Internal wildcard domain | `*.lab.example.internal` |
| Per-service hostnames | `svc-<role>` |
| Addresses | none, except documentation addresses (`192.0.2.0/24`) where an example needs one |
| Storage account, resource group, state key names | generic placeholders |
| Screenshots | regenerated against placeholder data, or omitted |

Software is named, in the README's stack table, because the design cannot be read without it.
Versions, network-device vendors and models are not: those are what an attacker would look up.

No VLAN numbers or address ranges are published in the current tree. A service-to-address inventory
is a target list and is not published either.

## What is not here, and why

- **Open findings.** A finding is written up when it is closed. A public list of your own
  unremediated weaknesses, attached to your name, is a different document with a different audience.
- **Anything under active remediation**, for the same reason.
- **Secrets, obviously** - but also credential *names*, key-store entry names, and the contents of
  any dashboard screenshot, which is the one place nobody remembers to look.

## What is still in the history, and why it stays

The network page withdrawn in September (`docs/architecture/network.md`) is gone from the tree and
still in the git history, including two closed pull requests' refs, which the host does not let an
owner delete. It holds no addresses or hostnames. It does contain the VLAN numbering in use at the time, a summary
of the server-VLAN rules and one firewall configuration keyword. It predates the firewall clean-up and the guest
firewall, and it was withdrawn because it was inaccurate. A history rewrite would not reach the
pull-request refs or existing clones, and recreating the repository would trade its history for
very little, so it was **accepted rather than rewritten** (November 2026).

This is the rule from the top of this page, applied to itself: history is permanent, so the review
that matters is the one before the commit.

## Guards on this repository

Secret scanning runs at two boundaries, on the theory that a control you have only ever seen pass is
not a control:

1. **Authoring side** - `gitleaks` as a pre-commit hook on the authoring workstation, installed per
   machine: it does not travel with a clone, so a new workstation starts without it.
2. **Receiving side** - GitHub secret scanning and push protection, and a CI workflow that scans the
   full history on every push and weekly. This is the boundary that does not depend on which machine
   committed.

Both were proven to deny before being trusted: a canary string shaped like a live credential was
staged, the commit was blocked, and the canary removed. Proving a guard allows is not proving it
denies.

**What each one cannot prove**, because a green check that is read as wider than it is becomes its
own problem:

- The hook scans the **index**, not history - it reports zero commits scanned, and it cannot see
  anything already committed. The full-history scan is a separate command, run once before the first
  commit.
- The hook lives in `.git/hooks/`, which is **not version-controlled**. It protects this machine and
  does not survive a clone. The CI workflow exists because of that gap, and runs weekly as well as
  on push, because upstream rules get added and a repository that passed in March is not one that
  passes today.
