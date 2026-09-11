# The door that was closed on purpose

**Found:** 2 September 2026 · **Caused:** 20 August 2026, between 11:07 and 15:50 · **Class:**
correct hardening with an unwritten consequence.

For twelve days the estate could not enrol a new host into its own intrusion-detection system. Every
control involved was working exactly as designed, and nothing anywhere produced a signal.

---

## The symptom

It started as something else entirely. A service was being retired, and deleting its orphaned agent
registration from the SIEM manager meant looking at the agent list — where a *different* host showed
**disconnected**, with a last keep-alive dated the day before the container had been replaced.

Disconnected reads as a network problem. It was not one.

## The chain

Each step was only reachable through the one before it.

**1. The agent was not installed.** `systemctl status` on the host returned `Unit could not be
found` — not *inactive*, not *failed*. The service had never existed on the rebuilt container. The
estate's most privileged host had been without host-based intrusion detection for two days.

**2. The play was right.** The estate-wide agent play *does* target that host; the host pattern was
correct. The play had simply never been run since the rebuild, because at that point it had no
schedule. A role that must run everywhere needs its own play — and a play that must run everywhere
needs a schedule.

**3. The guard fired correctly, on a fact that looks like the opposite.** Running the play failed on
its enrolment guard: the agent's key file **existed and was zero bytes**. The vendor's package ships
an empty key file, so a guard keyed on existence would have passed here. The guard was keyed on
content, which is why this was visible at all.

**4. Nothing was listening.** The agent log gave the actual error: unable to connect to the
enrolment service. Not a duplicate agent name, not a rejected password — a connection that went
nowhere. Name the error class before you reason about the cause; "enrolment failed" covers at least
four unrelated faults.

**5. The port was never published.** `docker inspect` on the manager returned `null` for the
enrolment port's host binding. `docker ps` had been showing that port all along, in a range with no
host address — which is `EXPOSE` metadata from the image, not a publication. The two are printed in
the same column.

## The cause

Two timestamps settled it, after three rounds of inference had not:

- The key file on the oldest enrolled guest was written at **11:07** on 20 August — the batch
  enrolment run.
- The manager container was created at **15:50** the same day — enrolment closed, exactly as the
  compose template's own comment prescribed: *close this port once the estate is enrolled*.

**Everything that existed inside those four hours is enrolled and fine. Nothing created or replaced
since could enrol.** Twelve days later the first replacement happened, and the gap appeared with no
signal anywhere, because "a thing that cannot be added" produces no event.

The hardening was correct. The sentence that should have followed it — *from now on, any guest that
gets replaced cannot re-enrol* — was never written down. An undocumented deliberate absence is
indistinguishable from an oversight, including to the person who made it.

## A correction worth keeping

The first pass at scoping this concluded that three hosts were unmonitored. It was one. The error
was dating agents by when their *service* was deployed rather than when their *guest* was built — an
agent enrols the first time the guest runs the play, which for one host was ten days before the
application it now runs ever existed. Reasoning from the wrong date produced a false escalation.
Dating the events from file mtimes settled in one command what inference had got wrong twice.

## What changed

The mechanism was already in the code and had simply never been exercised — the port publication was
already behind a conditional defaulting to closed:

```jinja
{% if enrollment_open | default(false) | bool %}
      - "1515:1515"
{% endif %}
```

So enrolment is closed by default and opened by a declared parameter rather than an edit:

```bash
ansible-playbook playbooks/siem.yml -e enrollment_open=true    # open
ansible-playbook playbooks/siem.yml                            # close again
```

Each toggle recreates the manager container, so there are two recreations and a short ingestion gap
per new host — an argument for batching enrolments rather than for leaving the door open.

Three things were added:

1. **The consequence, in writing**, next to the decision that caused it.
2. **A procedure**, now part of deploying anything: build the guest → open enrolment → run the agent
   play limited to that host → confirm both guards → close enrolment.
3. **A schedule** for the agent play, with alerting. Its first scheduled run failed on every host —
   a genuine misconfiguration, not a test — the alert reached a phone, and it was fixed. A fabricated
   test proves formatting; only a real fault proves detection.

The schedule's *coverage* is written down beside its green, because it inherits its runner's reach:
the scheduler cannot check itself (it cannot SSH to its own host's LAN address — a container is not
NAT'd to it, so the source-address pin correctly denies), and the hypervisors are not in its
inventory. The complete answer is a manager-side coverage assert that compares the agent list against
the hosts the play targets, run from elsewhere. That is queued.

## The general lesson

**A hardening step that closes a path must say what it will cost later.** The sentence after the win
is the one that gets skipped, because at the moment you write it the cost is hypothetical and the
win is real.

Two supporting rules came out of the same evening:

- **Enrolment and reporting are separate facts on separate ports.** One proves a registration was
  accepted; the other proves events are flowing. Check both, deliberately.
- **Verify the published port map, not the summary column.** `docker ps` prints exposed and
  published ports together; only the entries with a host address are reachable.

### The professional twin

This is the handover question, generalised. "Do you have monitoring?" is answerable yes here — every
agent that exists is healthy, the dashboard is green, the coverage was 100% of everything that had
been enrolled. The question that finds this is **"show me the last host you onboarded, and when."**
