# LinkedIn posts — Building the ARP layer

Five posts derived from [`content/posts/building-arp-layer.md`](content/posts/building-arp-layer.md), each on a different angle of the ARP protocol and the Linux commands that exercise it.

---

## Post 1 — Linux's neighbor table has 6 states and most engineers know only one

`ip neigh show` reports one of six states on every line, and most of them have nothing to do with whether the entry currently has a MAC.

Linux's neighbor table (the kernel's name for the unified ARP + IPv6 NDP cache) runs a six-state machine on every entry. **REACHABLE** means the mapping was confirmed within the last `base_reachable_time` seconds — default 30s, not 20 minutes as commonly believed; that's the ageing-out threshold. **STALE** means the entry is still in the table but hasn't been used recently, so the kernel will probe before trusting it again. **DELAY** is the transient between STALE and PROBE: the kernel has decided to re-verify but hasn't sent the request yet. **PROBE** means a unicast ARP request is on the wire awaiting reply. **FAILED** means the probe timed out and the host is considered unreachable until something proves otherwise. **PERMANENT** means a human typed `ip neigh add ... nud permanent` and the GC will never touch it. The state column is exposed in `/proc/net/arp` as a bitmask (0x02 = REACHABLE, 0x04 = STALE, etc.). A connection that mysteriously stalls for ~3 seconds on first use is almost always a STALE→DELAY→PROBE transition happening silently.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The ARP post implements a simpler version of this cache — only "valid" and "expired" — but explains all six Linux states because they shape how kernels debug each other.

Find the full ARP layer breakdown in the comments below 👇

---

## Post 2 — Your ARP cache fills up from packets that aren't addressed to you

Your ARP cache populates from packets that have nothing to do with you. That is RFC 826 working as designed.

RFC 826 §6 has one of the most underread clauses in all of TCP/IP: *"If the pair (sender protocol address, sender hardware address) is not in the table, add it."* Every incoming ARP packet — request *or* reply, addressed to us or not — passively teaches our cache about the sender's IP→MAC mapping. A broadcast ARP request from host A asking for host C's MAC, which we have no business answering, still adds A's mapping to our table. This is what makes ARP scale: on a busy LAN with 200 hosts, your cache populates within minutes from background broadcast traffic alone, with zero outbound requests from you. The downside is that a hostile sender can poison your cache by broadcasting a spoofed `(IP, MAC)` pair — which is why `arpwatch` exists and why managed switches implement Dynamic ARP Inspection. The merge has to happen *before* the operation switch in any correct implementation, including ours and the Linux kernel's, because the spec puts it before the operation check.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The ARP post puts `cache.Store(p.SenderIP, p.SenderMAC)` ahead of the request/reply switch in `handle()` because RFC 826 §6 says so.

Find the full ARP layer breakdown in the comments below 👇

---

## Post 3 — The most common production ARP failure has a hard limit of 1024

The most common production ARP failure is `Neighbour table overflow` in `dmesg`. The hard limit is 1024 entries.

`/proc/sys/net/ipv4/neigh/default/gc_thresh3` caps the size of the kernel's neighbor table per address family. The default is 1024 on most distributions; once you hit it, new entries cannot be added and any IP the kernel can't resolve produces `EHOSTUNREACH`. This happens on three predictable infrastructure patterns: a Kubernetes node running 200+ pods that each open connections to several services (one entry per CIDR-distinct destination); a busy load balancer fronting thousands of distinct backends; a router whose LAN has more than ~1000 active hosts. The fix is `sysctl -w net.ipv4.neigh.default.gc_thresh3=8192` and matching `gc_thresh1` / `gc_thresh2` increases — but you have to make it permanent in `/etc/sysctl.d/` or the next reboot puts you back at 1024. The two related counters `gc_thresh1` (below this, no GC at all — default 128) and `gc_thresh2` (soft cap, GC starts being aggressive — default 512) shape the curve of how quickly Linux evicts STALE entries under pressure.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The ARP post calls out the absence of a hard cap as a deliberate non-goal — our cache grows with the working set and is evicted by TTL alone.

Find the full ARP layer breakdown in the comments below 👇

---

## Post 4 — `arping -U` is the entire mechanism behind every floating-IP failover

`arping -U` sends an ARP packet that nobody asked for. It is also the entire mechanism behind every IP-based failover system you've used.

A gratuitous ARP (GARP) is a request where the sender IP and the target IP are the same address — semantically, "I am announcing myself, no reply needed." The packet is broadcast to `ff:ff:ff:ff:ff:ff`, and per RFC 826 §6 every host on the segment updates its cache with the new mapping. This is how keepalived, Pacemaker, AWS Elastic IPs, and every "virtual IP that floats between two hosts" implementation works: when the standby host takes over, it sends a gratuitous ARP claiming the VIP, every peer on the segment instantly updates its cache, and traffic flows to the new host within milliseconds — no TTL wait, no per-peer cleanup. The command is `sudo arping -I eth0 -U -c 3 -s VIP DEST`, and you'll see `keepalived` send exactly this in `tcpdump -e arp` during a failover event. The downside: there's no authentication. A malicious host can send GARP for the gateway IP and intercept every peer's traffic to the gateway until somebody re-claims the mapping with a GARP of its own.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The ARP post lists gratuitous-ARP as one of the deliberate omissions — implementable in a few lines on top of the existing handler.

Find the full ARP layer breakdown in the comments below 👇

---

## Post 5 — DHCP clients send ARP requests with sender IP `0.0.0.0` and the zero is doing real work

DHCP clients send an ARP request with the sender IP set to `0.0.0.0`. That zero is doing a lot of work.

After a DHCP server offers a lease, the client doesn't immediately start using the IP. First it sends an `arping -D` style probe: an ARP request asking "who has X.X.X.X" with the *sender* IP set to `0.0.0.0` instead of the proposed lease address. This is mandated by RFC 5227. If any other host on the segment replies, that host already owns the lease and the client must decline the offer. The zero sender IP is critical: if the client populated the sender field with the proposed lease, every other host on the segment would write it into their ARP cache as a side effect of RFC 826 §6 merging — meaning the lease would effectively be claimed *just by probing for it*, defeating the entire detection scheme. With sender IP `0.0.0.0`, there is no `(IP, MAC)` pair to merge, so the probe is observable but doesn't pollute anyone's cache. The same trick powers IPv4 Link-Local autoconfiguration (RFC 3927) for the 169.254.0.0/16 range when DHCP is unavailable.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The ARP post walks through `arping -D` and the zero-sender semantics in the Linux command exploration section, just before the implementation walkthrough.

Find the full ARP layer breakdown in the comments below 👇
