# LinkedIn posts — Building the IPv4 layer

Five posts derived from [`content/posts/building-ip-layer.md`](content/posts/building-ip-layer.md), each on a different angle of IPv4 and the Linux commands that exercise it.

---

## Post 1 — `ip route get` is the dry-run lookup nobody uses

`ip route get 8.8.8.8` tells you exactly which route the kernel would use without sending a single packet.

It is the most underused command in Linux networking debugging. The kernel walks its routing table by longest-prefix-match, picks the winning entry, resolves the source address that would be used, looks up the outbound interface, and prints it all in one line — `8.8.8.8 via 192.168.1.1 dev wlan0 src 192.168.1.100 uid 1000 cache`. No syscall to a remote host, no ARP request fired, no firewall rule traversed. You can pin the lookup parameters with `from`, `dev`, `mark`, and `oif` to simulate what would happen under different policy-routing rules: `ip route get 8.8.8.8 from 10.0.0.1 mark 100` resolves against whichever table matches mark 100. Production use: before changing a routing rule, run `ip route get` against a sample of representative destinations and diff the output against the same command after the change. If the next-hop or interface field moves, you know exactly which traffic flows are about to be rerouted. The `cache` suffix appears because the result lives in the kernel's route cache; `ip route flush cache` clears it.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The IPv4 post implements a longest-prefix-match routing table whose `Lookup(dst)` is the exact analog of `ip route get` — same algorithm, same tie-breakers, fewer features.

Find the full IPv4 layer breakdown in the comments below 👇

---

## Post 2 — TTL=64 is the universal default and a quiet OS fingerprint

Every IP packet your Linux box sends starts with a TTL of 64. That number reaches every host on the public internet with room to spare.

`/proc/sys/net/ipv4/ip_default_ttl` reports 64 on modern Linux distributions. The TTL field is decremented by each router along the path; when it reaches zero, the packet is dropped and an ICMP "Time Exceeded" comes back to the sender — which is how `traceroute` discovers paths. 64 is a careful compromise: surveys of the public internet show the longest known paths are roughly 30 hops, so 64 leaves comfortable margin while still bounding the damage from a routing loop. Different OSes pick different defaults — classic Windows used 128, classic Solaris used 255, macOS uses 64 — which means a quick look at the `ttl=` field in `ping` output can reveal the remote OS family. To override, write a new value: `sudo sysctl -w net.ipv4.ip_default_ttl=128`. To send a packet with a custom TTL, `ping -t 1` produces an ICMP echo that dies at the first-hop router. The same TTL field is what makes `mtr --tcp` work; modern firewalls don't strip it because doing so would break the entire diagnostic ecosystem.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The IPv4 post hard-codes TTL=64 in `sendDatagram` to match Linux's default exactly — and explicitly lists not-decrementing-TTL-on-forwarding as a deliberate non-goal (we are a host stack, not a router stack).

Find the full IPv4 layer breakdown in the comments below 👇

---

## Post 3 — IP fragment offsets are in 8-byte units, and 1480 falls out of the math

IPv4 fragment offsets are in 8-byte units, which is why every non-final IP fragment carries a payload size that is a multiple of 8.

The fragment offset field is 13 bits, and it counts in 8-byte units rather than bytes. Why 8? Because 13 bits × 8 bytes = 65,535 bytes — exactly the maximum IP datagram size addressable by the 16-bit Total Length field. With a byte-granular offset, you could only address 8 KiB of payload; with an 8-byte unit, you can address the full 64 KiB. The consequence on the sender side: when fragmenting a 4000-byte payload over a 1500-byte MTU, the per-fragment payload size must be rounded down to the nearest multiple of 8. The arithmetic is `(MTU - 20) / 8 * 8` — on a 1500-byte MTU, that yields 1480 bytes per fragment, which is exactly what Linux uses. Try `ping -s 4000 some-host` and look at `tcpdump`: you'll see fragments at offsets 0, 185, 370 — offsets in the 8-byte unit, which translate to 0, 1480, 2960 bytes. The last fragment can be any size; only the non-final fragments are constrained. The same 8-byte rule applies to IPv6 fragment headers — the extension header was kept compatible.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The IPv4 post implements the fragmentation path with `maxPayload := (mtu - MinHeaderLen) / 8 * 8` — the exact same integer-arithmetic round-down that Linux uses.

Find the full IPv4 layer breakdown in the comments below 👇

---

## Post 4 — `ping -M do` is how you reproduce the most common silent internet failure

`ping -M do -s 4000 some-host` is how you provoke the most common silent networking failure on the public internet.

The `-M do` flag sets the Don't Fragment bit on every outgoing IP packet. If the path to the destination has a smaller MTU somewhere in the middle — a VPN tunnel, a PPPoE link, a misconfigured switch — and the local kernel can't fragment the 4000-byte payload itself, the packet is dropped and the kernel reports `Frag needed and DF set (mtu = 1500)`. The router that found the packet too big *should* send back an ICMP "Fragmentation Needed" (type 3 code 4) carrying the next-hop MTU, which is the basis of Path MTU Discovery: the sender shrinks subsequent packets to fit. The problem is that thousands of misconfigured firewalls drop the ICMP reply, silently breaking PMTUD and producing the classic "TCP connection establishes but stalls forever" symptom. `tracepath some-host` is the diagnostic: it walks the path with increasing DF-set probe sizes, reporting the MTU at each hop where a problem appears. The kernel exposes the discovered PMTU per destination in `ip route show cache`, which clears with `ip route flush cache`.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The IPv4 post implements outbound fragmentation against the device MTU but explicitly lists no-PMTUD as a deliberate non-goal — implementing it would require generating ICMP error messages, which is a project for the ICMP post.

Find the full IPv4 layer breakdown in the comments below 👇

---

## Post 5 — The IPv4 ID field wraps every 50 milliseconds at gigabit, and that's fine

The IPv4 Identification field is 16 bits. At gigabit Ethernet speeds, the entire ID space cycles in under a second.

The ID field exists for one reason: tying together the fragments of a single datagram during reassembly. The receiver groups fragments by the 4-tuple `(src IP, dst IP, protocol, ID)` — two datagrams in flight to the same host must use different IDs or their fragments would mix. With 65,536 possible values, a sender transmitting at line rate over gigabit Ethernet (~83,000 packets/s for 1500-byte frames) wraps the entire ID space in under a second. Reassemblers handle this gracefully because the per-datagram reassembly timeout — 30 seconds on Linux, exposed in `/proc/sys/net/ipv4/ipfrag_time` — is far longer than the time between collisions for any *individual* (src, dst, proto) flow at realistic rates. Linux's `nextID` counter is a single global atomic; RFC 791 §3.2 says counters should be keyed by `(dst, protocol)` so the wrap interval is per-flow rather than per-host, but every mainstream stack uses the simpler global form. The same field is what lets `nftables` and `tcpdump` correlate fragments — without it, the receiver couldn't tell which 1480-byte chunk belongs to which originating datagram.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The IPv4 post uses `atomic.Uint32` for the global ID counter and explains why the 16-bit collision interval is acceptable at our scale.

Find the full IPv4 layer breakdown in the comments below 👇
