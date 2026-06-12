# LinkedIn posts — Building the ICMP layer

Five posts derived from [`content/posts/building-icmp-layer.md`](content/posts/building-icmp-layer.md), each on a different angle of ICMP and the Linux commands that exercise it.

---

## Post 1 — ICMP has no port numbers, yet concurrent pings don't collide

ICMP has no port numbers. Every message arrives at the host, not at a process — yet `ping` somehow keeps concurrent pings from the same machine straight.

The trick is two 16-bit fields in the ICMP echo header: **Identifier** and **Sequence Number**. The sender picks a unique Identifier per process — Linux's `ping(8)` traditionally uses the process PID truncated to 16 bits — and increments the Sequence per echo request within that process. The Identifier and Sequence are both copied verbatim into the reply by the responder. When the reply comes back, the kernel hands it to the right `ping` process by matching the Identifier; that process pairs it to the right outstanding request by matching the Sequence. This is why you can run `ping 8.8.8.8 &`, `ping 1.1.1.1 &`, and `ping 9.9.9.9 &` in three terminals without their replies crossing — each process gets a different ICMP Identifier from its PID. There's no port-based bind, no `select()` on a socket per port; the kernel routes by Identifier alone. Watch with `tcpdump -n -vv icmp` — you'll see `id 5523, seq 1` on every line, and the reply will echo both fields exactly.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The ICMP post implements exactly this demultiplexer: a `map[uint16]chan` keyed by Identifier, so concurrent `Ping()` calls in Go are correctly routed without ever colliding.

Find the full ICMP layer breakdown in the comments below 👇

---

## Post 2 — `ping` doesn't need root anymore, and a sysctl decides who can use it

On modern Linux, `ping` doesn't need root and doesn't need the setuid bit. There's a kernel feature for that.

Historically `ping` was a setuid binary because opening a raw ICMP socket required CAP_NET_RAW. Linux 3.0 introduced a new socket type — `SOCK_DGRAM` with protocol `IPPROTO_ICMP` — that lets unprivileged processes send and receive ICMP echo messages without raw-socket privileges. The kernel implementation only allows echo requests (Type 8), prevents spoofed source addresses, and assigns the Identifier from the socket binding rather than letting userspace choose it. Access is gated by `/proc/sys/net/ipv4/ping_group_range`, which is two numbers — `1 0` on locked-down distros (no GID can use it) or `0 2147483647` on permissive ones (any GID can). Set it permissively with `sudo sysctl -w net.ipv4.ping_group_range="0 2147483647"` and you can ship a `ping` binary with no special bits set. The iputils `ping` on Ubuntu 20.04+ tries this socket type first and only falls back to raw if it fails — which is why `ls -l $(which ping)` no longer shows the `s` setuid mode on modern systems.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The ICMP post implements the same kind of unprivileged echo path — but from inside a userspace stack talking to a TAP device, where the privilege boundary lives on the TUN/TAP fd open, not on the ICMP socket.

Find the full ICMP layer breakdown in the comments below 👇

---

## Post 3 — `ping`'s `ttl=` field is a passive OS fingerprint

`ping -c 1 some-host` lets you guess the remote OS with surprisingly high accuracy. Look at the `ttl=` field.

When an IP packet leaves a host, its TTL is set to the local kernel default. Each router along the path decrements it by 1. So if a reply comes back with `ttl=58`, you know two things: (a) the original TTL was probably the next standard default above 58 — almost certainly 64 — and (b) there were 64 − 58 = 6 routers on the return path. The standard defaults are an OS fingerprint: Linux and macOS use **64**, recent Windows uses **128**, Cisco IOS and classic Solaris use **255**. A `ttl=255` reply almost always means a router or a Solaris-era host; `ttl=128` is Windows; `ttl=64` is *nix. nmap's OS detection uses this as one signal alongside TCP option ordering, timestamps, and window sizes. You can change your machine's default to confuse this with `sudo sysctl -w net.ipv4.ip_default_ttl=128`, which would make Linux look like Windows from the outside — and is, incidentally, one of the simplest forms of OS-level privacy hygiene against passive fingerprinting.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The ICMP post relies on the IP layer's hard-coded `TTL=64` — which means `ping` against our stack reports `ttl=64`, indistinguishable from a real Linux host on the same segment.

Find the full ICMP layer breakdown in the comments below 👇

---

## Post 4 — `ping`'s `time=` is read out of bytes inside the packet itself

When `ping` shows `time=0.41 ms`, that number was extracted from bytes inside the ICMP packet itself.

Look at any echo request with `tcpdump -i lo -X icmp`. The first 8 bytes of the Data field on Linux contain the `gettimeofday()` result at the moment of transmission, encoded as two 32-bit ints (seconds, microseconds). The receiver echoes the Data field back verbatim — RFC 792 mandates it — so when the reply arrives, `ping` reads those 8 bytes back out, calls `gettimeofday()` again, and prints the difference. The 8 bytes never *meant* anything to the protocol — they're treated as opaque user data — but `ping` repurposes them as a sender-side timestamp piggy-backed on the wire. The cleverness is that no per-request state has to live in the sender process: even if `ping` were stateless between requests, it could still compute RTT from the bytes it's holding. Fill those 8 bytes with garbage instead — for example by writing a custom ICMP sender — and the round-trip *still works*, but `ping`-style RTT measurement only works if the responder is stock Linux echoing back exactly what arrived. Wireshark decodes the timestamp automatically when it sees the first 8 bytes look plausibly time-shaped.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The ICMP post's `Ping()` does exactly this trick — embeds a `time.Now().UnixNano()` in the first 8 bytes of Data, then reads it back on reply to compute RTT.

Find the full ICMP layer breakdown in the comments below 👇

---

## Post 5 — The ICMP checksum covers the payload too, unlike IP's

The ICMP checksum covers the entire packet — header *and* payload together. That makes it different from every other protocol in the IP family.

IPv4's header checksum covers only the 20-byte header (and is recomputed at every router because the TTL changes). TCP and UDP both cover a synthesised pseudo-header + their own header + the payload, but the pseudo-header is *not* part of the wire bytes — it's a calculation trick. ICMP is unique: its 16-bit checksum covers the literal bytes of the ICMP message exactly as they appear on the wire, including the variable-length Data section. This matters because the Data is unstructured: there's no inner parser to catch corruption, no length field to cross-check, no upper-layer protocol to notice a flipped bit in the payload. The checksum is the only integrity guard on those bytes. The computation trick is the same one IP uses for its header: zero the 2-byte checksum field, sum the entire ICMP message in 16-bit big-endian words using ones'-complement, take the bitwise NOT, write the result back into the cleared bytes. Verification is the same sum over the same bytes including the now-populated checksum — a correct packet sums to all-ones (`0xFFFF`), whose ones'-complement is zero.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The ICMP post implements `Parse` so that the checksum check is a single line: `if ip.Checksum(b) != 0 { return ErrBadChecksum }` — reusing the RFC 1071 implementation from the IP layer.

Find the full ICMP layer breakdown in the comments below 👇
