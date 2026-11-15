# The checks that could not see the login

**Found:** 22 September 2026 · **Closed:** 15 November 2026 · **Class:** the control was the defect,
twice.

> **In short.** Three admin consoles were published behind an allow-list whose name made it sound like
> authentication. Putting a real login in front of them was the easy part. The checks meant to prove
> it working could not fail - one would have stayed green with the login gone, the other with the
> monitor locked out - and fixing them broke every login probe in the estate for an evening, because
> the test tool and the probe were not the same client.

---

## The symptom

Reading the private repository end to end turned up a line nobody had questioned: the reverse proxy
published the hypervisor, switch-controller and firewall consoles to every trusted network - the VPN
and the client network included - under an access mode called `internal`.

`internal` was a source-address allow-list. It decided which networks could reach the console's own
login page, and nothing else. The name read like a protection level; it was a routing decision. The
consoles' own logins were the only thing in front of them.

## The fix that was easy

The identity provider already sat in front of most services. The three consoles moved behind it,
and the allow-list stayed in front of that, so a console now needs a trusted network *and* a login.
That part worked first time: all three asked for the login.

## The part that was not

**The uptime checks would have stopped checking anything.** Two of the three consoles cannot be
probed directly, by design - management access is narrow on purpose, and widening it for a monitor
would undo the point. So the checker reaches them through the proxy, and through the proxy it would
now meet the login page. A check that only ever sees the login page is a check of the identity
provider, three times over.

The design answer was a bypass at the identity provider for exactly one source address - the
monitoring host - so its checks reach the consoles themselves. That is safe only because the proxy
does not trust a client's forwarded-for header: the address the identity provider sees is the
address that connected, and it cannot be claimed by writing a header. That is now written down
beside the proxy configuration, because the bypass depends on it.

A second probe, from a host *without* the bypass, was meant to prove the other half: that the login
is enforced.

## Re-reading the checks before writing this up

Neither probe could fail on the thing it was there for.

- **The uptime check** accepted `200` or `302` and followed redirects. With the bypass broken, it would
  have received the identity provider's redirect - or followed it to the login page's `200` - and
  stayed green either way.
- **The login probe** accepted `200` as well as the redirect. With the login removed, the console
  answers `200`, and the probe would have reported success.

Both had passed. Both were reported as proof. This is the same failure as
[the guard that could not fail](2026-09-02-the-guard-that-could-not-fail.md), two months later, in a
change whose author knew that write-up well.

The fix: the uptime check no longer follows redirects and accepts only `200`, so the identity
provider's redirect is red. The login probe accepts only a redirect to the identity provider, so a
console answering `200` is red.

## The fix that broke everything else

The stricter login probe went live and **every** login-protected service turned red - ten of them.
The status code said why: `401`, not `302`.

The identity provider redirects clients that look like browsers and answers everything else with
`401` and no redirect at all. The probe sent no `Accept` header. The command used to design the
check - `curl`, from the same host - sends `Accept: */*`, and was redirected. Same host, same URL,
different client, different answer. The old probe had accepted `401` too, which is the only reason
it had ever passed; nobody had written down why.

The probe now asks like a browser. All ten went back to green - this time on the one answer that
means the login is there.

Along the way, a comment placed inside a template expression broke every run of the configuration
role until the next fix merged; the converge's check mode is what caught it. A YAML parse and a rendering of the *other* templates had passed; running
the real role against the real inventory would have caught it in seconds, and is what proved the
fix.

## Proving it

- From the monitoring host, all three consoles answer `200` with no redirect: the bypass works.
- From another host, all three answer a redirect to the identity provider: the login is enforced.
- The login probe, pointed at a service that has no login, reports the `200` and **fails**. That is
  the denial, observed.
- The bypass was then broken on purpose: its rules turned into ordinary login rules by hand. The
  monitoring host started getting the redirect, and all three uptime checks went red on their next
  run. Restoring the configuration with the playbook turned them green again. That is the uptime
  check's denial, observed.

## The general lesson

**An allow-list is not authentication, whatever it is called.** Read what an access mode *does*, not
what its name suggests.

**A check that sits behind a login checks the login.** Decide which side of it the check is meant to
see, and make the other side a failure.

**The tool you test with is not the client you deploy.** Headers, redirects and user agents all change
the answer. Before trusting a probe, ask it - not `curl` - what it receives.

**Run the real thing.** A file that parses is not a configuration that renders.

### The professional twin

Synthetic monitoring behind single sign-on, reporting a hundred per cent availability of the login
page. A health endpoint behind an authenticating gateway that answers `200` from the gateway itself. A
penetration test's "all endpoints require authentication" measured with a client the gateway treats
differently from a browser. Each is green, and each is measuring something other than what its
dashboard says.
