# The backup that never existed, and the alert that was trying the whole time

**Found:** 24 August 2026 · **Class:** fully declared, and fully unheard.

Two failures, found within an hour of each other, and the second is why nobody knew about the first.

The estate had a nightly backup job, defined in the hypervisor's own scheduler, targeting a storage
that was configured, mounted in the UI and visible in the storage list. Eight guests were in its
scope. It had never written a byte, and had never been able to.

---

## The symptom

None. That is the finding.

The investigation did not start from an alert or a failure - it started from an unrelated question
about whether a metrics exporter was installed everywhere. Checking one node's storage showed a
status of `inactive`, which is not a word anyone had gone looking for.

## The chain

**1. The storage was inactive.** The hypervisor's storage list is configuration, not state. A
storage can be defined against an export that nothing serves, and the definition is perfectly valid.

**2. The export didn't exist.** From the client side, an export listing returned
`RPC: Program not registered`. That is the server-side portmapper answering that nothing is
registered for NFS - not a permissions error, not a timeout. Different failures, different words:
a policy drop is a timeout, a reset means the packet arrived, and this was a host that answered to
say it had nothing listening.

**3. The server was never installed.** On the node meant to be serving the export,
`systemctl status nfs-server` returned **`Unit could not be found`**. Not *inactive*, not *failed* -
the package had never been installed. Which means the export had never existed, the storage had
never been reachable, and the nightly job had been failing since the day it was created.

Eight guests on one node had never been backed up at any point in their lives.

## The second failure

The backup job had been reporting its failures. The hypervisor mails job results to `root`, and
`root` had an address configured - in the cluster user database rather than in the mail aliases
file, which is where two rounds of looking had gone first.

The address was dead. **The mail was not unread; it was bouncing.** It had been bouncing nightly for
as long as the job had been failing, into a queue nobody had opened.

> "Nobody reads it" and "it bounces" look identical from the inside. Both produce silence. Only one
> of them is visible in a queue, and only if you check the queue rather than the configuration.

## What changed

- The NFS server was installed, the export defined, and the storage confirmed **active** - then
  verified from the other direction, on the destination filesystem, by looking at the archives
  themselves rather than at a green job.
- Every guest in the estate is now backed up cross-node.
- **Mail was removed as a notification path**, deliberately, and that decision was written down -
  because an undocumented deliberate absence is indistinguishable from an oversight. Notifications
  go to a push service that reaches a phone.
- The backup collector grew a rule to distinguish *never backed up* from *storage unreachable*, which
  had previously both reported as zero.

## The general lesson

**"We have backups" is a claim about existence, not about coverage, and certainly not about
recovery.** Three separate facts, and a dashboard shows the first one only:

1. Is it **configured**? - the storage was.
2. Is it **running**? - it wasn't, and hadn't ever been.
3. Has it been **restored from**? - a different question again, and the one this estate went on to
   fail elsewhere.

The corollaries that came out of this evening and have been applied since:

- **Verify effective configuration, not configuration.** `Unit could not be found` is not the same
  as *stopped*, and a defined storage is not a reachable one.
- **Verify coverage, not existence.** A job protecting under a third of the estate looks identical
  to a working one from the summary.
- **A control whose success is invisible needs something that proves it, from outside.** The proof
  here is the archive on the destination filesystem, not the job's own exit status.
- **Check the queue, not the config.** A notification path that is trying and failing looks exactly
  like one nobody has configured.

### The professional twin

This is the single most transferable finding in this repository, because every organisation has said
this sentence. The handover questions that would have caught it in one minute:

- **When was the last restore, and from which storage?** Not "do you have backups."
- **Show me the last notification you received from the backup system**, on the device you received
  it on.
- **How many guests are in scope, and how many exist?** The gap is the answer.
