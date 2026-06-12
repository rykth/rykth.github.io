# LinkedIn posts — Building the TCP layer

Five posts derived from [`content/posts/building-tcp-layer.md`](content/posts/building-tcp-layer.md), each on a different angle of TCP and the Linux commands that exercise it.

---

## Post 1 — TIME_WAIT exists for a 60-second reason, and ignoring it breaks production

A TCP connection that closes cleanly spends 60 seconds in TIME_WAIT before the kernel releases the 4-tuple. That number is RFC-mandated and not negotiable.

The duration is **2×MSL** — Maximum Segment Lifetime, which RFC 793 fixed at 30 seconds, doubled to allow for the worst-case round-trip of a delayed segment. The reason: when the active-close side sends its final ACK, the receiver might not get it; the receiver re-sends its FIN; the closer must still be around to ACK the re-sent FIN. If the 4-tuple were reclaimed immediately, a brand-new connection on the same port pair could receive that stray FIN and mistake it for its own peer hanging up. Sixty seconds is enough that any in-flight segment from the previous incarnation has died on TTL by the time the port is reused. The cost: at high outbound connection rates to a single destination, the ephemeral port pool empties into TIME_WAIT and `connect(2)` starts returning `EADDRNOTAVAIL`. The fix on the active-close side is `sysctl -w net.ipv4.tcp_tw_reuse=1`, which lets new outbound connections reuse a TIME_WAIT entry as long as the TCP timestamp option proves the new connection is later. `ss -tn state time-wait | wc -l` is the count to watch.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The TCP post implements the full 11-state RFC 793 machine including TIME_WAIT with a 2×MSL timer — `msl = 30 * time.Second` matches Linux exactly.

Find the full TCP layer breakdown in the comments below 👇

---

## Post 2 — `ss -tnipo` shows the kernel's hidden TCP state in one line

`ss -tnipo` shows the entire hidden state machine of a TCP connection in one line — RTT, congestion window, retransmission timeout, scaling factor, pacing rate.

The `-i` flag is the one to remember: it tells `ss` to dump the kernel's per-connection TCP state in human-readable form. A typical line looks like `cubic wscale:7,7 rto:204 rtt:0.067/0.012 mss:1448 cwnd:10 bytes_sent:418 bytes_acked:418 segs_out:5 segs_in:4 send 1.728Gbps pacing_rate 3.456Gbps lastsnd:8 delivered:5 minrtt:0.057`. Read field by field: `cubic` is the congestion control algorithm; `wscale:7,7` is the window scale exponent both sides agreed during the handshake (window × 2⁷); `rto:204` is the current retransmission timeout in milliseconds; `rtt:0.067/0.012` is the smoothed RTT and variance (Jacobson/Karels); `mss:1448` is the effective MSS after option negotiation; `cwnd:10` is the congestion window in segments; `pacing_rate` is what BBR-style pacing would emit. Every one of these is read from the kernel's `tcp_info` struct, the same one `getsockopt(TCP_INFO)` exposes. If you're debugging a TCP performance problem and not running `ss -tnipo` against the affected connection, you're missing 90% of the visible state.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The TCP post implements the Jacobson/Karels RTT estimator and Karn's exponential-backoff timer — the same `srtt`, `rttvar`, and `rto` numbers Linux exposes through `tcp_info`.

Find the full TCP layer breakdown in the comments below 👇

---

## Post 3 — TCP initial sequence numbers are cryptographic because of a 1994 attack

Every modern TCP connection's initial sequence number is produced by a cryptographic RNG. There's a specific 1990s attack that's the reason.

RFC 793 originally specified the ISN as a counter incrementing 250,000 times per second — predictable enough that Kevin Mitnick in 1994 famously used it to spoof a TCP connection from a trusted host without ever seeing a single reply packet, by guessing the next ISN the target would generate. RFC 1948 in 1996 added a host-and-port-specific hash; RFC 6528 in 2012 made it mandatory and recommended a per-host secret. Modern Linux derives the ISN from MD5 over `(source IP, dest IP, source port, dest port)` keyed by a kernel secret generated at boot, with a 4-microsecond-resolution timestamp mixed in. Any off-path attacker who can't observe the connection cannot guess the next ISN within the lifetime of a probe — the prediction space is 2³² and the per-host secret rotates. You can sanity-check your own implementation by capturing 100 outbound SYNs and verifying the seq numbers are statistically uniform — if they cluster or increment monotonically, the kernel ISN generator is broken.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The TCP post implements `newISS()` as a four-byte read from `crypto/rand` — not the full RFC 6528 keyed derivation, but uniformly random per connection, which defeats the same class of attack.

Find the full TCP layer breakdown in the comments below 👇

---

## Post 4 — `ethtool -K eth0 tso off` is the first move in any serious TCP debugging session

`sudo ethtool -K eth0 tso off gso off gro off lro off` is the first command in any serious TCP debugging session. The default state makes `tcpdump` lie.

**TSO** — TCP Segmentation Offload — lets the kernel hand a huge "segment" (sometimes 64 KB) to the NIC, and the NIC fragments it into MTU-sized packets on the wire. The reason: pushing the per-packet work to hardware frees the CPU. The problem: `tcpdump` captures *before* segmentation, so you see one giant "segment" that never existed on the wire as a single packet. Compare against the receiver's `tcpdump` which sees the post-fragmentation reality, and the seq/ack arithmetic doesn't match. **GSO** is the software equivalent for NICs without hardware TSO. **GRO/LRO** are the receive-side mirror: the NIC coalesces multiple incoming segments into one before delivering to the kernel, so tcpdump on the receiver sees fewer-and-larger frames than were actually transmitted. Turning all four off makes the wire visible exactly as the protocol intends: every TCP segment is exactly one packet, both sides agree on byte counts, and the `cwnd` math on the sender matches the `segs_in` counter on the receiver. The performance penalty is real (~10–15% on bulk transfers) but the debugging clarity is worth it. Re-enable with `ethtool -K eth0 tso on gso on gro on lro on` when you're done.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The TCP post deliberately doesn't implement TSO/GSO — every `Send` produces exactly one segment, which makes the receive side's behaviour testable against a stock `tcpdump`.

Find the full TCP layer breakdown in the comments below 👇

---

## Post 5 — A TCP FIN consumes a sequence number even though it has no payload

A TCP FIN consumes one sequence number even though it carries no payload. That phantom byte is what makes the four-way close arithmetic work.

When a peer is done sending, it sets the FIN flag in the next outgoing segment. The segment carries no data — the byte stream is empty by definition. But the ACK that comes back from the other side increments the acknowledgment number by *one* anyway, as if a single byte had been transmitted. This is not a hack; it's specified by RFC 793. The reason: the close itself is a logical event in the byte stream that needs to be acknowledged. Without consuming a sequence number, the closing side couldn't reliably tell whether its FIN had been received — the peer's ACK number wouldn't move. The same trick applies to SYN: the SYN flag at the start of a connection also consumes a sequence number, which is why the three-way handshake's final ACK is `ISN+1` rather than `ISN`. So a TCP byte stream has *two* virtual bytes beyond its real payload: one for SYN (at the start, seq = ISN) and one for FIN (at the end). Both are visible in `tcpdump -S` output as the `[S]` and `[F.]` flags on length-zero segments that nonetheless move the ack pointer forward by 1.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The TCP post's `handleEstablished` explicitly accounts for the FIN sequence with `c.recvBuf.Receive(hdr.SeqNum + uint32(len(payload)), nil)` — advancing the receive cursor one byte past the end of the payload to capture the phantom FIN byte.

Find the full TCP layer breakdown in the comments below 👇
