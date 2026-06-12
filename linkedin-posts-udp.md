# LinkedIn posts — Building the UDP layer

Five posts derived from [`content/posts/building-udp-layer.md`](content/posts/building-udp-layer.md), each on a different angle of UDP and the Linux commands that exercise it.

---

## Post 1 — UDP is the only protocol in the IP family that lets the sender opt out of the checksum

UDP is the only protocol in the IP family with a documented way for the sender to say "I didn't compute a checksum." That escape hatch is the exact 16-bit value `0x0000`.

RFC 768 carves out a special meaning: a Checksum field of `0x0000` on the wire means the sender opted out of integrity protection. The algorithm itself cannot legitimately produce that value — if the ones'-complement sum happens to be all-zeros, the sender rewrites it to `0xFFFF` (the alternate representation of zero in ones'-complement arithmetic) before transmission. That makes `0x0000` an unambiguous sentinel that no honest checksum can collide with. Receivers see `0x0000` and skip verification entirely; any other value is a real checksum to verify. This was a 1980 compromise for hosts that couldn't afford the per-byte computation overhead. It is gone in IPv6 — UDP checksums are mandatory there. Watch for it with `tcpdump -i tap0 'udp[6:2] == 0'`: every matching datagram is a sender announcing "trust me on this one," usually a real-time codec, an embedded device, or a misbehaving load generator.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The UDP post implements this opt-out in `Deliver`: `if hdr.Checksum != 0 { verify } else { accept }` — three lines that match exactly what RFC 768 mandates.

Find the full UDP layer breakdown in the comments below 👇

---

## Post 2 — The UDP checksum covers 12 bytes that are never on the wire

The UDP checksum protects bytes that never travel on the wire.

UDP's 16-bit checksum covers three things: the 12-byte IPv4 pseudo-header, the 8-byte UDP header, and the variable payload. The first of those — the pseudo-header — is synthesised at compute time and discarded immediately: source IP (4), destination IP (4), one zero byte, IP protocol number (1, value 17 for UDP), and the UDP length (2). None of those 12 bytes occupy any space in the transmitted frame. They exist solely so the checksum will detect a class of failure modes that header-only protection misses. Picture a corrupted router that rewrites the IP destination address but leaves the UDP header intact: without the pseudo-header, the receiver would happily accept a misdelivered datagram because the UDP checksum still validates. With the pseudo-header, the receiver computes the checksum using *its own* IP (the dst it actually received) and any divergence from what the sender computed reveals the corruption. TCP uses the same trick with a slightly different layout — same defence against mis-routed segments. The shared algorithm is why protocol-agnostic primitives like `ip.PseudoHeaderChecksum` and `ip.Add16` exist: UDP and TCP both call them.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The UDP post implements `ComputeChecksum` as three partial sums combined via `ip.Add16` — never materialising the pseudo-header + header + payload as a contiguous buffer.

Find the full UDP layer breakdown in the comments below 👇

---

## Post 3 — `connect(2)` on a UDP socket doesn't send anything; it installs a filter

Calling `connect(2)` on a UDP socket does not send a packet. It tells the kernel which remote to accept replies from.

UDP is connectionless on the wire — there is no SYN, no handshake, no per-flow state on routers. But the BSD socket API lets you bind a UDP socket to a specific remote by calling `connect(srv_ip, srv_port)`. After that call, two things change at the local kernel: outgoing `send(2)` calls no longer need to specify the destination (the socket remembers), and incoming datagrams from *any other source* are silently dropped before reaching the receive queue. The drop is implemented in the kernel's `udp_v4_lookup` path — once a socket has a peer set, the lookup demands a full 4-tuple match `(src ip, src port, dst ip, dst port)` rather than just a destination port. This is what lets a DNS client safely race two queries to two resolvers in parallel — each socket only sees the reply from its own peer. The flip side: you can call `connect(2)` again with `AF_UNSPEC` to *unbind* the peer, returning the socket to listen-from-everyone mode. The socket type never changes; only the kernel's per-receive filter does.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The UDP post implements both modes — `Listen` creates an unconnected `Conn` that accepts from any source; `Dial` creates a connected `Conn` that filters per-receive against a bound remote.

Find the full UDP layer breakdown in the comments below 👇

---

## Post 4 — `SO_REUSEPORT` is how N processes share one UDP port and the kernel load-balances

`SO_REUSEPORT` is how a single UDP port can be shared by N processes, with the kernel doing the load balancing.

Set the socket option before `bind(2)`, and multiple processes can all bind to the same `(ip, port)` pair without `EADDRINUSE`. When a UDP datagram arrives, the kernel hashes the 4-tuple `(src ip, src port, dst ip, dst port)` and picks one of the bound sockets based on that hash. The selection is deterministic per flow — every datagram from the same source ends up at the same backend socket — which is what makes it usable for stateful protocols (DNS resolvers, syslog collectors, QUIC servers). Each process gets its own receive queue, eliminating the contention of a single-socket `recvfrom()` loop. The flag was added in Linux 3.9 specifically to scale memcached and nginx. It is also the easiest way to do a graceful restart of a UDP server: bind a new process to the same port with SO_REUSEPORT, let the kernel start hashing new flows to it, drain the old process, kill it. No connection loss, no port collision. `ss -ulnp` shows multiple owners on the same port when SO_REUSEPORT is in play — the `users:` column lists all of them.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The UDP post calls out lack-of-SO_REUSEPORT as a deliberate limitation — our `conns map[uint16]*Conn` allows exactly one Conn per port today.

Find the full UDP layer breakdown in the comments below 👇

---

## Post 5 — `ip_local_port_range` is two numbers, and they cap your outbound flow count

`/proc/sys/net/ipv4/ip_local_port_range` is two numbers, and they control how many concurrent outbound flows your machine can have to a single destination.

By default on Linux it's `32768 60999` — 28,232 ports available for the kernel to assign as the *source port* when a process calls `connect(2)` without binding. The pool is consumed by every TCP and UDP connection out of the box, plus by every `getsockname()`-style ephemeral allocation. Hit the ceiling on outbound connections to a single `(dst_ip, dst_port)` 4-tuple — say, a microservice hammering one downstream — and `connect(2)` starts returning `EADDRNOTAVAIL` even though the destination is healthy. The fix is `sudo sysctl -w net.ipv4.ip_local_port_range='10000 65535'`, which gives you 55,536 ports. `tcp_tw_reuse=1` helps too, because TIME_WAIT sockets count against the pool. Look at the current setting with `cat /proc/sys/net/ipv4/ip_local_port_range`; look at the actively-used count with `ss -tan | awk 'NR>1 {print $4}' | grep -oE ':[0-9]+$' | sort -u | wc -l`. The kernel's allocator walks this range looking for an unused port — the same shape of algorithm any userspace UDP stack ends up implementing.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The UDP post's `allocEphemeralLocked` is a rotating linear scan over `[40000, 65535]` — the same algorithm Linux uses, scaled down.

Find the full UDP layer breakdown in the comments below 👇
