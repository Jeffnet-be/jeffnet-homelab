# The reply that never left

**Found:** 16-17 November 2026 · **Class:** a shortcut that two security layers broke between them, and a trust it had been granting all along.

> **In short.** Remote SSH to one server stopped working, and only for servers on one physical host.
> The host was also the gateway that remote traffic entered through. Its VPN client translated the
> traffic's addresses once on the way in, and again on the server's freshly enabled firewall bridge.
> The server answered; the answer never found its way back. Moving the gateway off the hypervisors fixed
> it, and showed that the old setup had made every remote device look like an admin host.

---

## The symptom

Remote access to the lab runs over a mesh VPN. One device on the internal network advertises the lab's
subnet to the VPN, and traffic from a laptop elsewhere enters the lab through it. Until mid-November,
that device was one of the three hypervisors.

On 16 November, SSH over the VPN to one server timed out. The same server answered normally from inside
the lab. Servers on the other two hypervisors answered over the VPN as usual. Only the guests on the
hypervisor that was doing the routing were affected, and the problem had started when those guests got
their own firewall.

## The chain

**1. The obvious suspect was the new firewall.** Each server had just been given a default-deny
firewall at the hypervisor, with an allow-list for SSH from admin hosts. A rule missing for the VPN
would look exactly like this: a connection that times out instead of being refused. But servers on
the other hypervisors, behind the same generated rule set, were fine. Whatever was wrong was specific
to the one hypervisor, so the rules were not the place to keep looking.

**2. Watching the packets, not the rules.** A capture on that hypervisor showed the connection request
arriving from the VPN, being translated to the hypervisor's own address (the VPN client masquerades
routed traffic, as it should), and reaching the server. The server answered. The answer reached the
hypervisor's firewall bridge for that server, and stopped there: it never left through the VPN
interface.

**3. Translated twice.** The capture also showed the address translation firing a second time, on the
firewall bridge in front of the server, with the source port remapped. With a guest firewall enabled,
the hypervisor passes that server's traffic through an extra bridge, and the VPN client's masquerade
rule matched there as well. A reply has to be translated back through every step it was translated
through, on the host that holds the records. With two translations of the same connection on one host,
the reply did not make it back: it died in connection tracking, at the bridge, exactly where the capture
lost it.

**4. Why only this host.** On the other hypervisors, VPN traffic arrived already translated, from
outside, and was simply routed to their guests: one translation, done elsewhere. Only on the hypervisor
that was both the VPN gateway and the host of the firewalled guests did both translations happen on
one machine.

**5. The trust nobody had chosen.** Because the gateway translated every VPN connection to its own
address, every remote device arrived in the lab looking like that hypervisor. And the hypervisors were
on the admin allow-list. So any device on the VPN, not only the admin laptop, was treated as an admin
host by every firewall rule that trusted that list. It had worked for months, which is why nobody had
asked what it implied.

## The cause

The VPN gateway ran on a hypervisor that also hosted firewalled guests. Its masquerade rule applied
twice to traffic for those guests, and the replies died in connection tracking. Separately, the same
translation made every VPN device indistinguishable from a trusted hypervisor.

## What changed

- **Two small dedicated gateway VMs** took over the VPN route, one primary and one standby, on
  different hypervisors. They do one thing: forward the lab subnet, with translation on. No hypervisor
  advertises a route or acts as an exit node any more.
- **The VPN policy says who may carry the subnet:** only devices tagged as subnet routers can advertise
  it, and the hypervisors' own tag is granted nothing. The policy carries tests that refuse a change
  which would let the hypervisors' tag reach anything.
- **The hypervisors left the admin allow-list.** The two gateways are on it instead, as the address
  remote admin traffic now arrives from. Confirmed on the server: an SSH session over the VPN now shows
  the gateway as its source.
- **A written rule:** never route the VPN on a hypervisor that hosts firewalled guests.

Still a trade-off, and written down as one: the gateways translate addresses too, so every VPN device
still arrives as a gateway address. The difference is that the gateways are single-purpose, and the
VPN policy, not the lab's firewall, decides which devices may use them.

## The general lesson

**A shortcut that works can still be wrong in two ways at once.** Running the VPN on a hypervisor saved
a machine and worked for months. It failed only when a second, correct control (a per-guest firewall)
met it on the same host, and the investigation of that failure exposed what it had been doing all
along: turning every remote device into a trusted one. **When two controls touch the same packet, test
them together, and when you put a translating device in front of an allow-list, ask who the list now
trusts.**

### The professional twin

A NAT gateway or proxy whose address sits on an allow-list, so everything behind it inherits the trust.
A load balancer that rewrites source addresses in front of an IP-restricted admin page. A firewall
appliance and a host firewall that each translate the same flow. Each passes every test run on its own,
and breaks, or quietly widens access, only in combination.
