---
title: "Building the ICMP layer in Go"
date: 2026-05-23T10:00:00+00:00
description: "The sixth post in my TCP/IP stack series. IPv4 gave us routed datagrams; now we implement ICMP — the protocol behind ping, traceroute, and every 'Destination Unreachable' you have ever seen. After this post, the Linux kernel's ping command will get a reply from a stack we wrote ourselves."
tags: [golang, networking, tcpip, icmp, rfc792]
---

## Where this fits

We have a loopback and a TUN/TAP device, an Ethernet codec on top of a
`net.Registry`, ARP to resolve IP→MAC mappings, and an
[IPv4 handler](/posts/building-ip-layer) that delivers reassembled
datagrams to `UpperHandler`s by protocol number. Everything that
follows in this series — ICMP, UDP, TCP — is an upper handler that
plugs into the IP layer through that single `Deliver(src, dst, payload)`
method.

[ICMP](https://datatracker.ietf.org/doc/html/rfc792) is the smallest of
the three, and the one with the most satisfying acceptance test. After
the code in this post is wired in, you can type `ping 10.0.0.2` against
our TAP device and watch a reply come back from a stack with zero kernel
networking code in it.

The code lives at
[rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack), in
`pkg/icmp`.

```
 ┌─────────────────────────────────────────┐
 │   Application (HTTP…)                   │
 ├─────────────────────────────────────────┤
 │   Transport (TCP/UDP)                   │
 ├─────────────────────────────────────────┤
 │   ICMP ◄────────────────────────────────│  ← this post
 ├─────────────────────────────────────────┤
 │   Network (IPv4)                        │
 ├─────────────────────────────────────────┤
 │   Link (Ethernet) + ARP                 │
 ├─────────────────────────────────────────┤
 │   Device (loopback / TAP)               │
 └─────────────────────────────────────────┘
```

ICMP sits *next to* the transport layer rather than under it — it
travels inside IP datagrams with protocol number 1, just like TCP travels
with 6 and UDP with 17. But unlike TCP and UDP, ICMP has no port numbers
and no notion of an application binding to a "listening" address.
Every ICMP message is for the *host*, not for a process on the host. The
kernel deals with them directly.

Two files, two concerns. `icmp.go` is the wire codec — a `Packet`
struct and the `Parse` / `Marshal` pair. `handler.go` is everything
above that: the `ip.UpperHandler` that replies to echo requests, plus
the `Ping` API that the caller side uses.

---

## What is ICMP?

ICMP is the *diagnostic and error reporting* protocol of IPv4. When
something goes wrong inside the network — a router has no route for a
destination, a packet's TTL hits zero, a host is unreachable — the
device that noticed the problem sends an ICMP message back to the
original source explaining what happened. When something needs to be
*tested* — is this host reachable? what is the round-trip time? what
is the path? — ICMP echo request/reply is the universal probe.

RFC 792 defines about a dozen message types. The ones you encounter in
practice are:

| Type | Name                     | What it means                                           |
|------|--------------------------|---------------------------------------------------------|
| 0    | Echo Reply               | Response to a type-8 echo request                       |
| 3    | Destination Unreachable  | Router cannot deliver — see Code for why                |
| 5    | Redirect                 | "Use a different gateway next time"                     |
| 8    | Echo Request             | "Are you there? Reply with the same payload"            |
| 11   | Time Exceeded            | TTL hit zero — used by `traceroute`                     |
| 12   | Parameter Problem        | Malformed IP header — rare in practice                  |

Our implementation covers only **echo request (8) and echo reply (0)**.
That is the subset that `ping` exercises and the subset that lets us
test the stack end-to-end without implementing a full ICMP error-
reporting machine. Other types are silently ignored.

### The echo request/reply pair

```
Sender (10.0.0.1)                          Target (10.0.0.2)
       │                                          │
       │   IP { proto=1 } / ICMP {                │
       │     Type=8 (EchoRequest),                │
       │     Code=0,                              │
       │     Checksum=…,                          │
       │     ID=A, Seq=N,                         │
       │     Data="hello"                         │
       │   }                                      │
       │─────────────────────────────────────────►│
       │                                          │
       │                                          │  (just send it back,
       │                                          │   flip Type to 0)
       │                                          │
       │   IP { proto=1 } / ICMP {                │
       │     Type=0 (EchoReply),                  │
       │     Code=0,                              │
       │     Checksum=…,                          │
       │     ID=A, Seq=N,                         │
       │     Data="hello"                         │
       │   }                                      │
       │◄─────────────────────────────────────────│
```

The protocol is *almost* a no-op: take the incoming packet, change Type
from 8 to 0, recompute the checksum, send it back to the source. The
target does no interpretation of the Data — it copies the bytes
verbatim. That is the property `ping` exploits to measure round-trip
time: it stamps the *current time* into the Data on transmit, then
reads the same bytes back on reply and subtracts to get the RTT.

### The packet format

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Type     |      Code     |          Checksum             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Identifier          |        Sequence Number        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Data (variable length)                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Eight bytes of header, plus a variable payload. Field by field:

- **Type (1 byte)** — `8` for echo request, `0` for echo reply.
- **Code (1 byte)** — sub-type. For echo this is always `0`.
  Destination Unreachable (Type=3) uses Code to distinguish "network
  unreachable" (0) from "host unreachable" (1), "protocol unreachable"
  (2), "port unreachable" (3), and so on.
- **Checksum (2 bytes)** — RFC 1071 ones'-complement checksum over the
  *entire* ICMP message (header + data). Unlike IP, ICMP does not have a
  pseudo-header — the checksum is over the bytes as they sit on the
  wire.
- **Identifier (2 bytes)** — set by the sender, copied verbatim into
  the reply. Used to match replies to requests when multiple
  concurrent pings are in flight.
- **Sequence Number (2 bytes)** — usually incremented by the sender on
  each successive echo request. Also copied verbatim into the reply.
- **Data (variable)** — opaque bytes. The receiver echoes them back
  exactly.

Note what is *not* in the header: there are no source or destination
fields. ICMP relies entirely on the IP header that wraps it to know who
sent the message and who should receive the reply. ID and Sequence are
the only fields that let a sender demultiplex multiple in-flight
exchanges.

---

## Exploring ICMP from the command line

Every concern our handler implements has a corresponding Linux tool that
drives it from the outside. These are the commands worth keeping nearby
while working on the implementation.

### `ping` — the canonical exercise

The single most useful command:

```bash
# Default: send one echo request per second, forever
ping 10.0.0.2

# Stop after N requests
ping -c 4 10.0.0.2

# Custom interval (seconds; root needed for <1)
sudo ping -i 0.1 10.0.0.2

# Custom payload size (default is 56 bytes, producing 64-byte ICMP messages)
ping -s 32 10.0.0.2     # 40-byte ICMP message
ping -s 1472 10.0.0.2   # fills a 1500-byte MTU exactly

# Custom TTL — every router decrements by 1
ping -t 1 10.0.0.2      # only reaches first-hop router

# Hard timeout (per-reply)
ping -W 2 10.0.0.2

# Set Don't Fragment in the IP header — forces PMTUD
ping -M do -s 1472 10.0.0.2

# Use a specific source interface
ping -I tap0 10.0.0.2

# Flood-ping (root only) — useful for stress-testing the receive path
sudo ping -f 10.0.0.2
```

Default `ping` output:

```
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.052 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.041 ms
^C
--- 10.0.0.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1006ms
rtt min/avg/max/mdev = 0.041/0.046/0.052/0.005 ms
```

The numbers map directly onto fields in our `Packet`:

- `icmp_seq` is `p.Seq` — incremented per request by `ping`.
- `ttl=64` is the IP header's TTL field on the reply. Our IP layer
  hardcodes 64, which `ping` will report as `ttl=64` on every reply it
  receives from us.
- `time=` is the RTT, computed by `ping` using a timestamp it embedded
  in the Data of each request. Our `Ping` does exactly the same thing
  internally.

### Watch ICMP on the wire

```bash
# All ICMP traffic, verbose
sudo tcpdump -i tap0 -n -vv icmp

# Only echo requests
sudo tcpdump -i tap0 -n 'icmp[icmptype] == icmp-echo'

# Only echo replies
sudo tcpdump -i tap0 -n 'icmp[icmptype] == icmp-echoreply'

# Destination Unreachable messages
sudo tcpdump -i tap0 -n 'icmp[icmptype] == icmp-unreach'

# Hex dump of every ICMP packet (header + data)
sudo tcpdump -i tap0 -n -X icmp

# Save to pcap for Wireshark analysis
sudo tcpdump -i tap0 -w /tmp/icmp.pcap icmp
```

A typical request/reply pair with `-vv`:

```
20:11:08.182452 IP (tos 0x0, ttl 64, id 5523, offset 0, flags [DF],
                    proto ICMP (1), length 84)
    10.0.0.1 > 10.0.0.2: ICMP echo request, id 5523, seq 1, length 64
20:11:08.182471 IP (tos 0x0, ttl 64, id 17829, offset 0, flags [none],
                    proto ICMP (1), length 84)
    10.0.0.2 > 10.0.0.1: ICMP echo reply, id 5523, seq 1, length 64
```

The reply's `id 5523, seq 1` echoes the request's `id 5523, seq 1` byte
for byte. The IP header's `id` field (17829) is unrelated — that is
the IPv4 datagram identifier our `Send` allocates from
`nextID.atomic.Uint32`, separate from the ICMP `Identifier`.

The bpf filter `icmp[icmptype] == icmp-echo` looks cryptic but is worth
remembering: `icmp[N]` reads the Nth byte of the ICMP payload, and
`icmptype` is just the offset alias for byte 0 (the Type field). The
constant `icmp-echo` resolves to 8.

### Traceroute via ICMP

`traceroute` works by sending packets with increasing TTL values and
collecting the ICMP "Time Exceeded" responses from each router along the
path. The default Linux `traceroute` uses UDP; `-I` switches to ICMP
echo requests, which is the variant Windows' `tracert` uses by default
and which is sometimes the only one that survives modern firewalls:

```bash
# ICMP-based traceroute
sudo traceroute -I 8.8.8.8

# Mix of TCP, UDP, ICMP with continuous probing
mtr 8.8.8.8

# Show ICMP message types at each hop
sudo mtr --report-mode --report-cycles 10 8.8.8.8
```

Our stack does *not* send Time Exceeded messages on TTL expiry — that
is part of the ICMP error-reporting subset we deferred. So our TAP-
hosted IP will appear as `* * *` in any `traceroute` whose path passes
through our stack. Implementing it would be a few dozen lines in the IP
handler; it's on the limitations list.

### Block, rate-limit, and rewrite ICMP with iptables / nftables

ICMP is what every adversarial firewall blocks first, and replicating
those conditions in a test environment is useful when proving that the
stack handles drops correctly:

```bash
# Block every incoming echo request
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j DROP

# Block outgoing echo replies
sudo iptables -A OUTPUT -p icmp --icmp-type echo-reply -j DROP

# Rate-limit incoming ICMP to 5 per second
sudo iptables -A INPUT -p icmp -m limit --limit 5/s -j ACCEPT
sudo iptables -A INPUT -p icmp -j DROP

# Show current rules including counters
sudo iptables -L -v -n

# Remove all the test rules
sudo iptables -F
```

The nftables equivalent:

```bash
sudo nft 'add table inet filter'
sudo nft 'add chain inet filter input { type filter hook input priority 0; }'
sudo nft 'add rule inet filter input icmp type echo-request drop'
```

When our stack is sending pings *from* the TAP toward the host, an
`iptables -A INPUT -p icmp --icmp-type echo-request -j DROP` rule on
the host is the easiest way to drive the `ctx`-timeout path in
`TestHandler_Ping_Timeout` against real kernel behaviour.

### Kernel ICMP tunables

```bash
# Globally disable echo reply (the kernel won't respond to ping)
echo 1 | sudo tee /proc/sys/net/ipv4/icmp_echo_ignore_all

# Ignore broadcast pings (default 1 — for security)
cat /proc/sys/net/ipv4/icmp_echo_ignore_broadcasts

# Rate-limit outgoing ICMP errors (the "X ms" between identical errors)
cat /proc/sys/net/ipv4/icmp_ratelimit

# Mask: which ICMP types are rate-limited (bitfield, default covers errors)
cat /proc/sys/net/ipv4/icmp_ratemask

# Allow ICMP echo without root (Linux ICMP socket feature)
cat /proc/sys/net/ipv4/ping_group_range
```

`icmp_echo_ignore_all=1` is the kernel-side equivalent of removing our
echo-request handler — useful when comparing what the kernel does
versus what our stack does for the same incoming traffic. If both
return silence, both are configured to drop; if our stack replies and
the kernel doesn't, the difference is visible immediately on
`tcpdump -e icmp`.

### Craft ICMP packets with scapy

For driving every code path our handler implements — including the
malformed-input path — `scapy` is unbeatable:

```bash
sudo python3 - <<'PY'
from scapy.all import *

# Plain echo request
sendp(Ether(dst='aa:bb:cc:dd:ee:01') /
      IP(src='10.0.0.1', dst='10.0.0.2') /
      ICMP(type=8, code=0, id=0x4242, seq=1) /
      b'hello',
      iface='tap0')

# Echo reply (might be useful to drive Ping receive logic)
sendp(Ether(dst='aa:bb:cc:dd:ee:01') /
      IP(src='10.0.0.1', dst='10.0.0.2') /
      ICMP(type=0, code=0, id=0x4242, seq=1) /
      b'hello',
      iface='tap0')

# Bad checksum (chksum=1234 overrides scapy's auto-computation)
sendp(Ether(dst='aa:bb:cc:dd:ee:01') /
      IP(src='10.0.0.1', dst='10.0.0.2') /
      ICMP(type=8, code=0, chksum=0x1234) /
      b'broken',
      iface='tap0')

# Type 13 (Timestamp request) — a type we should ignore
sendp(Ether(dst='aa:bb:cc:dd:ee:01') /
      IP(src='10.0.0.1', dst='10.0.0.2') /
      ICMP(type=13, code=0, id=1, seq=0) /
      b'\x00'*12,
      iface='tap0')
PY
```

The bad-checksum case exercises `TestParse_BadChecksum` via the live
wire. The Type=13 case verifies that our handler silently ignores types
we don't claim — there should be no reply on `tcpdump -i tap0 icmp`
when this packet is injected.

### ICMP statistics

The kernel maintains per-protocol counters that map onto every code path
in our handler:

```bash
nstat -a | grep -i icmp
```

```
IcmpInMsgs                      438                0.0
IcmpInErrors                    2                  0.0
IcmpInEchoReps                  219                0.0
IcmpInEchos                     217                0.0
IcmpOutMsgs                     436                0.0
IcmpOutErrors                   0                  0.0
IcmpOutEchoReps                 217                0.0
IcmpOutEchos                    219                0.0
```

`IcmpInEchos` is the count of received echo requests — what our
handler's `Deliver` increments. `IcmpOutEchoReps` is what `sendReply`
produces. `IcmpInErrors` rolls up checksum failures and malformed
packets — our `Parse` error returns. If these numbers do not match
between the kernel and our stack when running the same test load, we
have a behavioural discrepancy to investigate.

---

## The implementation

Two files. We will go through them in receive order: codec first
(`icmp.go`), then handler (`handler.go`).

### `Packet` — the 8-byte header plus opaque data

```go
type Type uint8

const (
    TypeEchoReply   Type = 0
    TypeEchoRequest Type = 8
)

type Packet struct {
    Type     Type
    Code     uint8
    Checksum uint16
    ID       uint16
    Seq      uint16
    Data     []byte
}
```

Six fields, exactly the wire shape. `Type` is its own named type so
that misusing a `uint8` where a `Type` is expected (or vice versa) is a
compile error.

`Data` is a slice because the payload is variable-length. For an echo
request from `ping -s 56`, this is 56 bytes; for `ping -s 1472`, it
fills a 1500-byte MTU. The slice is the only allocation on the parse
path.

### `Parse` — checksum first, then decode

```go
func Parse(b []byte) (Packet, error) {
    if len(b) < HeaderLen {
        return Packet{}, ErrPacketTooShort
    }
    // Checksum covers the full ICMP message (header + data).
    if ip.Checksum(b) != 0 {
        return Packet{}, ErrBadChecksum
    }
    var p Packet
    p.Type = Type(b[0])
    p.Code = b[1]
    p.Checksum = binary.BigEndian.Uint16(b[2:4])
    p.ID = binary.BigEndian.Uint16(b[4:6])
    p.Seq = binary.BigEndian.Uint16(b[6:8])
    if len(b) > HeaderLen {
        p.Data = make([]byte, len(b)-HeaderLen)
        copy(p.Data, b[HeaderLen:])
    }
    return p, nil
}
```

Two things to call out.

**The checksum is computed across the whole buffer**, not just the
header. ICMP differs from IP here: the IP checksum covers only the 20-
byte header, and the upper layer is on its own. ICMP says "include
everything, header and payload alike." That choice makes the checksum
also act as an integrity check on the echo's Data field — which matters
because the Data is unstructured bytes, with no upper-layer parser to
catch corruption.

**The checksum verifies in one call by checking against zero**. RFC
1071's property — a correct checksum included in its own input sums
to all-ones, whose ones'-complement is zero — applies just as much
here as in IP. `ip.Checksum(b) != 0` is the test for "this packet is
intact." That single line is doing as much work as a full re-derivation
would, with no buffer copy and no exclusion logic.

The codec dependency on `pkg/ip` is purely for `Checksum` — ICMP does
not pull in any other IP type. The dependency is documented in the
`import` block:

```go
import "github.com/rykth/tcp-ip-stack/pkg/ip"
```

We could duplicate the 20-line `Checksum` function into `pkg/icmp` to
remove the dependency, but the round-trip tests in
`TestChecksum_RFC1071Vector` already verified the IP implementation
against the RFC vector. Reusing it is correct by construction.

`Data` is *copied* into a fresh slice on parse, not aliased to the
input buffer. That decision matches what the IP handler does at its
own boundary: the Registry owns the receive buffer; once `Parse`
returns, the caller is free to do anything with `b`. The copy is the
cost of letting the upper layer hold onto its payload.

### `Marshal` — the inverse

```go
func Marshal(dst []byte, p Packet) (int, error) {
    need := HeaderLen + len(p.Data)
    if len(dst) < need {
        return 0, ErrPacketTooShort
    }
    dst[0] = uint8(p.Type)
    dst[1] = p.Code
    dst[2], dst[3] = 0, 0 // zero before checksum computation
    binary.BigEndian.PutUint16(dst[4:6], p.ID)
    binary.BigEndian.PutUint16(dst[6:8], p.Seq)
    copy(dst[8:], p.Data)
    check := ip.Checksum(dst[:need])
    binary.BigEndian.PutUint16(dst[2:4], check)
    return need, nil
}
```

The same zero-the-checksum-field-then-overwrite trick as IP. Before
computing, dst[2:4] is zero, so the checksum sums in nothing for itself.
After computing, we write the result back into the same two bytes — and
any subsequent `Parse` will verify because the new checksum is now part
of the sum that produces zero.

`copy(dst[8:], p.Data)` *also* makes Marshal copy-safe — that is what
`TestMarshal_PayloadIsCopied` checks. After `Marshal` returns, mutating
the caller's `p.Data` slice does not affect the encoded bytes in
`dst`. This isn't a deliberate guarantee — `copy` works that way for
free — but pinning it in a test prevents a future "optimisation" that
aliases the slice from breaking the contract silently.

The function returns `(int, error)` rather than just `error` because
the caller needs to know how many bytes were actually written into
`dst`. The caller frequently passes a buffer larger than the message
(for example, a stack-allocated `[1500]byte`) and slices it back to
`buf[:n]` before passing to the IP `Send`.

### The `Handler` and the `Sender` interface

```go
type Sender interface {
    Send(ctx context.Context, dst [4]byte, proto ip.Protocol, payload []byte) error
}

type Handler struct {
    sender Sender
    nextID atomic.Uint32

    mu      sync.Mutex
    pending map[uint16]chan time.Time
}
```

The `Sender` interface is the same pattern as
[`ARPResolver`](/posts/building-ip-layer) in `pkg/ip`: declare exactly
the surface this package needs, let the consumer satisfy it implicitly.
`*ip.Handler` satisfies `Sender` because its `Send` method has the
matching signature — but `pkg/icmp` never imports `pkg/ip` for this
purpose, only for `ip.Checksum` and `ip.Protocol`.

The reason matters more here than at the IP/ARP boundary: ICMP's tests
exercise a real `Ping` flow with a real `Send` invocation, and
substituting a `mockSender{}` lets the suite verify the bytes that
*would* go on the wire without ever booting an IP handler, an ARP
handler, a device, or a Registry.

`pending` is the demultiplexing map. Each `Ping` allocates a unique
`uint16` Identifier and registers a channel keyed by that ID. When an
echo reply arrives in `Deliver`, the handler looks up the channel by
the reply's Identifier and signals it. Without this map, two concurrent
pings to the same destination would race for whichever reply arrived
first; with it, each `Ping` only receives the reply that carries its
own ID.

### `Deliver` — receive both halves of the conversation

```go
func (h *Handler) Deliver(src, dst [4]byte, payload []byte) {
    p, err := Parse(payload)
    if err != nil {
        return
    }

    switch p.Type {
    case TypeEchoRequest:
        go h.sendReply(src, p)

    case TypeEchoReply:
        h.mu.Lock()
        ch, ok := h.pending[p.ID]
        h.mu.Unlock()
        if ok {
            select {
            case ch <- time.Now():
            default:
            }
        }
    }
}
```

`Deliver` is the entry point that `ip.Handler` invokes when a datagram
with protocol number 1 has been parsed and reassembled. It does three
things: parse the ICMP packet (silently dropping garbage), classify by
type, and act.

The **`go h.sendReply(src, p)`** line is the most consequential one in
the file. The reply has to be sent through `h.sender.Send`, which
ultimately calls `arp.Resolve` for the next hop. On a cold ARP cache,
that resolve will broadcast and wait for a reply — potentially seconds.
If we did this synchronously, we would block the IP receive loop, and
every other datagram (UDP, TCP, ARP, anything) would be delayed behind
us. Spawning a goroutine lets the receive loop continue immediately
while the reply path waits on ARP in the background.

The cost is that an echo request triggers an unbounded goroutine: there
is no semaphore limiting how many concurrent replies are in flight. For
a low-rate protocol like ICMP this is fine; for something like UDP this
would need a worker pool. We will revisit the pattern when UDP arrives.

The **`select { case ch <- time.Now(): default: }`** pattern in the
reply branch is identical to the non-blocking dispatch we use in
`net.Registry`. The channel has capacity 1; if it is already full, the
reply is a duplicate (the kernel sometimes generates these under
flood-ping conditions), and dropping it is the right behaviour. Without
the `default`, a duplicate would block `Deliver` forever and stall the
IP layer.

### `sendReply` — the echo response

```go
func (h *Handler) sendReply(dst [4]byte, req Packet) {
    reply := Packet{
        Type: TypeEchoReply,
        Code: 0,
        ID:   req.ID,
        Seq:  req.Seq,
        Data: req.Data,
    }
    buf := make([]byte, HeaderLen+len(reply.Data))
    if _, err := Marshal(buf, reply); err != nil {
        return
    }
    h.sender.Send(context.Background(), dst, ip.ProtocolICMP, buf) //nolint:errcheck
}
```

The reply construction has the form RFC 792 mandates: keep the original
`ID` and `Seq` so the requester can match the response to its request,
keep the `Data` verbatim so the requester can verify integrity and
compute RTT against an embedded timestamp.

`context.Background()` here is deliberate, and worth explaining. We
don't have a context from the caller — `Deliver` doesn't take one,
because it is invoked by `ip.Handler`'s receive loop, which has its own
shutdown context that doesn't propagate to individual upper-layer
calls. The reply is a short, self-contained operation: ARP cache hit
(very likely, since the requester just successfully reached us), one IP
`Send`, done. Using `context.Background` keeps this goroutine
independent of caller lifetimes — even if a hypothetical caller's
context cancels, the reply still goes out, which is the right
behaviour for a stateless echo.

The `//nolint:errcheck` is a deliberate concession: there is nowhere
useful to surface a send error from a spawned goroutine. Logging is
better than swallowing — but the project's logging story is light, and
the kernel itself doesn't surface ARP failures from its own ICMP reply
path. The error is dropped on the floor and the next ICMP request will
get another shot.

### `Ping` — the demultiplexer in action

```go
func (h *Handler) Ping(ctx context.Context, dst [4]byte) (time.Duration, error) {
    id := uint16(h.nextID.Add(1))

    // Register the pending channel BEFORE sending to avoid a race where
    // the reply arrives before we've registered.
    ch := make(chan time.Time, 1)
    h.mu.Lock()
    h.pending[id] = ch
    h.mu.Unlock()
    defer func() {
        h.mu.Lock()
        delete(h.pending, id)
        h.mu.Unlock()
    }()

    var data [8]byte
    sent := time.Now()
    binary.BigEndian.PutUint64(data[:], uint64(sent.UnixNano()))

    pkt := Packet{Type: TypeEchoRequest, ID: id, Seq: 0, Data: data[:]}
    buf := make([]byte, HeaderLen+len(pkt.Data))
    if _, err := Marshal(buf, pkt); err != nil {
        return 0, fmt.Errorf("icmp: marshal echo request: %w", err)
    }

    if err := h.sender.Send(ctx, dst, ip.ProtocolICMP, buf); err != nil {
        return 0, fmt.Errorf("icmp: send echo request: %w", err)
    }

    select {
    case received := <-ch:
        return received.Sub(sent), nil
    case <-ctx.Done():
        return 0, ErrPingTimeout
    }
}
```

Five steps. The order of the first two — generate a unique ID, then
register the pending channel — is what makes this function correct
under concurrent calls:

1. **Allocate a unique ID with `atomic.Uint32.Add(1)`.** Concurrent
   callers cannot collide. The 16-bit truncation gives 65,536 IDs
   before wrap — far more than any plausible number of in-flight pings,
   and at wrap time the previously-registered channel will have been
   deleted by its own `defer`.
2. **Register the pending channel *before* sending.** If we sent first,
   a very fast reply could arrive before our `pending[id] = ch`
   assignment ran, and `Deliver` would find no waiting channel and
   silently drop the reply. By registering first, the reply path is
   guaranteed to find us.
3. **Embed the send timestamp in the Data field.** The 8 bytes are the
   nanosecond Unix time at send. The kernel's `ping` does exactly this.
   It's redundant with the in-process `sent` variable, but it leaves
   the wire-level RTT measurable by `tcpdump` and confirms the Data
   bytes were echoed back unchanged.
4. **Marshal and send via the injected `Sender`**. The Sender does
   the route lookup, ARP resolution, fragmentation, and Ethernet
   framing. ICMP just hands it bytes and a destination.
5. **Wait on either the channel or the context.** Standard cancellable
   blocking. The `defer` cleans up the `pending` map entry whether
   `Ping` returns success or timeout.

The `defer` deletion is what `TestHandler_Ping_Timeout` exercises: a
timed-out `Ping` must not leave its channel in the map, or a delayed
reply minutes later would try to send to a goroutine that no longer
exists. `goleak.VerifyNone` catches this — if the cleanup were broken,
the spawned goroutines from earlier tests would still be parked on
unread channels.

---

## The tests

Fourteen tests in three blocks: `Parse` codec (5), `Marshal` codec
(3), and `Handler` (6).

```bash
# Plain run
go test ./pkg/icmp/... -v

# With the race detector
CGO_ENABLED=1 go test ./pkg/icmp/... -race -v
```

A clean run:

```
=== RUN   TestParse_ValidEchoRequest
--- PASS: TestParse_ValidEchoRequest (0.00s)
=== RUN   TestParse_ValidEchoReply
--- PASS: TestParse_ValidEchoReply (0.00s)
=== RUN   TestParse_TooShort
--- PASS: TestParse_TooShort (0.00s)
=== RUN   TestParse_BadChecksum
--- PASS: TestParse_BadChecksum (0.00s)
=== RUN   TestParse_EmptyData
--- PASS: TestParse_EmptyData (0.00s)
=== RUN   TestMarshal_RoundTrip
--- PASS: TestMarshal_RoundTrip (0.00s)
=== RUN   TestMarshal_TooSmall
--- PASS: TestMarshal_TooSmall (0.00s)
=== RUN   TestMarshal_PayloadIsCopied
--- PASS: TestMarshal_PayloadIsCopied (0.00s)
=== RUN   TestHandler_RepliesTo_EchoRequest
--- PASS: TestHandler_RepliesTo_EchoRequest (0.00s)
=== RUN   TestHandler_Ignores_Malformed
--- PASS: TestHandler_Ignores_Malformed (0.01s)
=== RUN   TestHandler_Ping_CacheMiss
--- PASS: TestHandler_Ping_CacheMiss (0.00s)
=== RUN   TestHandler_Ping_Timeout
--- PASS: TestHandler_Ping_Timeout (0.03s)
=== RUN   TestHandler_Ping_ConcurrentIDs
--- PASS: TestHandler_Ping_ConcurrentIDs (0.00s)
=== RUN   TestHandler_Protocol
--- PASS: TestHandler_Protocol (0.00s)
PASS
ok      github.com/rykth/tcp-ip-stack/pkg/icmp  1.054s
```

### The codec tests

Five `Parse` tests and three `Marshal` tests. Each one is structured
the same way: build a packet via `Marshal`, then either decode it via
`Parse` (positive cases) or perturb the bytes and confirm `Parse`
returns the right sentinel error (negative cases).

The interesting ones:

**`TestParse_EmptyData`** — a `ping`-style packet with zero payload
bytes. `len(b) == HeaderLen == 8`. The check `len(b) > HeaderLen`
before allocating `p.Data` is what avoids a zero-length allocation here.
This test catches the kind of off-by-one bug that would crash on
`make([]byte, 0)` if the bounds check were wrong.

**`TestMarshal_PayloadIsCopied`** — the inverse property to
`TestParse_payloadAliasesInput` from the Ethernet post. After `Marshal`
returns, mutating the source `p.Data` slice must not affect the encoded
bytes in `dst`. This pins the "encode-once" contract: the caller can
recycle their data buffer immediately after `Marshal` returns.

```go
data := []byte{1, 2, 3}
p := icmp.Packet{Type: icmp.TypeEchoRequest, ID: 1, Data: data}
buf := make([]byte, icmp.HeaderLen+len(data))
icmp.Marshal(buf, p)
data[0] = 0xFF
got, _ := icmp.Parse(buf)
if got.Data[0] == 0xFF { t.Error("...") }
```

If a future change ever optimised `Marshal` by aliasing the slice
header rather than calling `copy`, this test breaks first.

### The handler tests

Six tests that together cover both directions of the protocol:

**`TestHandler_RepliesTo_EchoRequest`** — inject a synthetic echo
request via `Deliver`, wait for the spawned reply goroutine to call
`Send`, and parse the bytes the mock recorded. Every field of the reply
must match the request: same `ID`, same `Seq`, same `Data`. The reply
must have `Type == TypeEchoReply`. This is the entire server-side
protocol in one test.

**`TestHandler_Ignores_Malformed`** — `Deliver` a 1-byte payload (too
short to be ICMP at all). The handler must not panic, deadlock, or
call `Send`. The sleep-and-check pattern verifies *absence* of sends:
the only correct behaviour for malformed input is silence.

**`TestHandler_Ping_CacheMiss`** — the full round trip. Start `Ping`
in a goroutine; wait for the request to be written to the mock sender;
parse it to extract the auto-generated `ID`; build a matching reply;
inject it via `Deliver`. `Ping` must unblock and return a positive
`time.Duration`. This is the test that proves the request/reply
demultiplexing actually works.

```go
go func() {
    rtt, err := h.Ping(ctx, ipB)
    rttCh <- rtt
    errCh <- err
}()

sender.waitSent(t, 1)

pkts := sender.Recorded()
req, _ := icmp.Parse(pkts[0].payload)
replyPayload := buildReply(t, req.ID, req.Seq, req.Data)
h.Deliver(ipB, ipA, replyPayload)

rtt := <-rttCh
```

Five distinct synchronisation points, all deterministic. The
`waitSent(1)` pattern is the same one we used in
[ARP](/posts/building-arp-layer) — drives all of these tests without a
single `time.Sleep` for positive assertions.

**`TestHandler_Ping_Timeout`** — call `Ping` with a 30 ms context, do
not reply. Must return `ErrPingTimeout`. This is the test that the
`defer` map cleanup makes pass: without it, the leaked channel would
trip `goleak.VerifyNone`.

**`TestHandler_Ping_ConcurrentIDs`** — the demultiplexing stress test.
Four goroutines simultaneously call `Ping(ipB)`. Each must allocate a
distinct `ID` (the test checks indirectly: replies are matched per-ID,
and each `Ping` must resolve with the right reply). Crucially, the
test injects the replies *in reverse order* — the latest request gets
its reply first, the earliest gets its reply last. Every `Ping` must
still unblock correctly. If two requests shared an ID, one would never
unblock and the test would deadlock.

**`TestHandler_Protocol`** — one-liner: `h.Protocol() == ProtocolICMP`.
This is the contract the IP handler keys on when dispatching by
protocol number. Pinning it explicitly catches the case where a future
change accidentally returned `ip.ProtocolUDP` and made ICMP frames
disappear into the UDP handler.

---

## Driving it on a real TAP device

Wiring the ICMP handler into the stack means giving the IP handler an
`UpperHandler` for protocol 1, and giving the ICMP handler a `Sender`
(which is the IP handler itself). The whole composition fits in about
forty lines:

```go
package main

import (
    "context"
    "log"

    "github.com/rykth/tcp-ip-stack/pkg/arp"
    "github.com/rykth/tcp-ip-stack/pkg/ethernet"
    "github.com/rykth/tcp-ip-stack/pkg/icmp"
    "github.com/rykth/tcp-ip-stack/pkg/ip"
    netpkg "github.com/rykth/tcp-ip-stack/pkg/net"
    "github.com/rykth/tcp-ip-stack/pkg/raw/tuntap"
)

func main() {
    dev, err := tuntap.New("tap0", tuntap.DeviceTAP)
    if err != nil { log.Fatal(err) }
    defer dev.Close()

    localMAC := ethernet.Addr{0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0x02}
    localIP  := [4]byte{10, 0, 0, 2}

    arpH  := arp.NewHandler(localMAC, localIP, dev)
    table := ip.NewTable()
    onLink, _ := ip.ParseNetwork("10.0.0.0/24")
    table.Add(ip.Route{Network: onLink, Dev: dev})
    ipH   := ip.NewHandler(localMAC, localIP, arpH, table)
    icmpH := icmp.NewHandler(ipH)
    if err := ipH.RegisterUpper(icmpH); err != nil { log.Fatal(err) }

    r := netpkg.New()
    if err := r.AddDevice(dev);  err != nil { log.Fatal(err) }
    if err := r.AddHandler(arpH); err != nil { log.Fatal(err) }
    if err := r.AddHandler(ipH);  err != nil { log.Fatal(err) }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    log.Fatal(r.Start(ctx))
}
```

In one terminal, create the TAP and run the stack:

```bash
sudo ip tuntap add dev tap0 mode tap user $USER
sudo ip link set tap0 up
sudo ip addr add 10.0.0.1/24 dev tap0
go run ./examples/icmp
```

In a second, watch the traffic:

```bash
sudo tcpdump -i tap0 -n -e -vv 'icmp or arp'
```

In a third, ping our stack:

```bash
ping -c 3 10.0.0.2
```

The first `ping` triggers a full ARP exchange, then echo request/reply
cycles:

```
ARP, Request who-has 10.0.0.2 tell 10.0.0.1
ARP, Reply 10.0.0.2 is-at aa:bb:cc:dd:ee:02
IP 10.0.0.1 > 10.0.0.2: ICMP echo request, id 5523, seq 1, length 64
IP 10.0.0.2 > 10.0.0.1: ICMP echo reply,   id 5523, seq 1, length 64
IP 10.0.0.1 > 10.0.0.2: ICMP echo request, id 5523, seq 2, length 64
IP 10.0.0.2 > 10.0.0.1: ICMP echo reply,   id 5523, seq 2, length 64
IP 10.0.0.1 > 10.0.0.2: ICMP echo request, id 5523, seq 3, length 64
IP 10.0.0.2 > 10.0.0.1: ICMP echo reply,   id 5523, seq 3, length 64
```

`ping` reports:

```
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.83 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.41 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.38 ms

--- 10.0.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss
rtt min/avg/max/mdev = 0.379/0.540/0.831/0.207 ms
```

That is the moment all the work pays off. Three layers of code we
wrote — Ethernet, ARP, IPv4, plus the Registry that glues them
together — are responding to the Linux kernel exactly as if they were
a real host on the segment. The kernel cannot tell the difference.

To exercise the **`Ping`** direction (our stack sending pings outward),
add a small RPC to the program — read a destination IP from stdin,
call `icmpH.Ping(ctx, dst)`, print the RTT:

```go
go func() {
    var ip4 string
    for {
        if _, err := fmt.Scanln(&ip4); err != nil { return }
        var dst [4]byte
        for i, s := range strings.Split(ip4, ".") {
            n, _ := strconv.Atoi(s); dst[i] = byte(n)
        }
        ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
        rtt, err := icmpH.Ping(ctx, dst)
        cancel()
        if err != nil { fmt.Println("ping:", err); continue }
        fmt.Printf("reply in %v\n", rtt)
    }
}()
```

Typing `10.0.0.1` (the kernel's TAP address) at the prompt now
produces an ARP request (our ARP handler doing the resolve), a real
echo request to the kernel, and an echo reply from the kernel back into
our stack:

```
ARP, Request who-has 10.0.0.1 tell 10.0.0.2
ARP, Reply 10.0.0.1 is-at <kernel's TAP MAC>
IP 10.0.0.2 > 10.0.0.1: ICMP echo request, id 1, seq 0, length 16
IP 10.0.0.1 > 10.0.0.2: ICMP echo reply,   id 1, seq 0, length 16
```

The `id 1` is from our `nextID.Add(1)`. The Data length is 16 bytes:
the 8-byte timestamp our `Ping` embeds, plus 8 more zeros from the IP
layer's minimum frame padding.

---

## Design decisions, revisited

### Why goroutine-spawn `sendReply`?

The receive path on the IP layer is a single goroutine — `ip.Handler`'s
`process` loop reads from its `RxChan` and calls `upper.Deliver`
inline. If `Deliver` blocks (waiting on ARP, waiting on a slow device
write, waiting on anything), the entire receive path stalls. Every
other ICMP request behind this one waits; every TCP segment, every UDP
datagram, every ARP frame on the same device waits.

Spawning a goroutine for the reply makes the protocol stack progressive
under load. The cost is unbounded concurrency — a flood ping could spawn
thousands of reply goroutines simultaneously — but the size of each one
is tiny (~2 KB stack) and the work is bounded (one ARP call, one IP
send, exit).

For the upper layers that follow (UDP, TCP), the same problem will
demand a different solution because the *work* per packet is heavier
and the *rate* is higher. A `sync.Pool` of worker goroutines, fed by
a channel, is the standard answer. ICMP doesn't need it yet.

### Why is `Sender` an interface?

The same reason `ARPResolver` in `pkg/ip` is an interface: we need a
test fixture that records sends without booting an IP stack. The
`mockSender` in the test file is twenty lines of code. Without the
interface, every test would need a full ARP+IP+Registry+device
construction to exercise a single ICMP behaviour.

The interface also gives us the *direction* of the dependency we want.
`pkg/icmp` does not need to import `*ip.Handler` to call its `Send`.
The two packages stay siblings. If we ever wanted to run ICMP over a
different lower layer — say, a UDP-tunnelled IP-in-UDP encapsulation —
substituting a different `Sender` would be the entire change.

### Why `atomic.Uint32` and not a per-destination counter?

Each `Ping` allocates a fresh ID and registers it in the global
`pending` map. Concurrent `Ping`s to the same destination must get
different IDs, or `Deliver` could not disambiguate replies. A monotonic
atomic counter — same pattern as `ip.Handler.nextID` — is the
minimum mechanism that satisfies the constraint.

A per-destination counter would let the ID space be reused per
destination (65,536 concurrent pings per peer rather than per host),
but the global form is simpler, and 65,536 concurrent in-flight pings
is well beyond any plausible use case. If we ever hit that ceiling we
have other problems first.

### Why store `time.Time` in the pending channel?

The channel could carry `struct{}` and let `Ping` compute the RTT
against its own `sent` variable. We do that *already* — `received.Sub(sent)` uses the in-process `sent`, not the channel-delivered time. The
`time.Now()` value sent through the channel is currently unused.

It is there as future-proofing. If we ever want to surface the kernel's
*kernel-receive-time* for RTT measurement — for example, by using
`SO_TIMESTAMPNS` style hardware-receive timestamps when we wire the
device to a real NIC — the channel already carries the right type.
Removing it later if we never use it is a one-line change; adding it
later would mean revisiting every call site.

---

## Limitations

| Limitation | Detail |
|------------|--------|
| **Echo only** | We handle types 8 and 0. Types 3 (Destination Unreachable), 5 (Redirect), 11 (Time Exceeded), 13 (Timestamp Request), and others are silently ignored |
| **No outbound errors** | When our IP layer drops a packet — bad checksum, no route, TTL=0 on forwarding — it does not send back an ICMP error. The sender sees a black hole |
| **No PMTUD support** | We don't generate `Fragmentation Needed (DF set)` errors. A peer doing Path MTU Discovery will time out instead of receiving an explicit "too big" |
| **No reply rate limiting** | `sendReply` spawns a goroutine per request unconditionally. A flood ping can saturate device-write throughput |
| **No raw API** | Upper-layer code cannot send arbitrary ICMP types. Only echo requests via `Ping` |
| **Best-effort `sendReply` errors** | The reply goroutine swallows errors. A persistent ARP failure would not surface to logs |
| **Identifier wraparound** | The 16-bit ID space wraps at 65,536. At ~10k pings/s this collides every few seconds, although the channel-cleanup `defer` prevents the worst case |
| **No source-address selection** | Outgoing replies use whatever `localIP` was configured at construction. Multi-homed hosts need one handler per local IP |

The biggest missing piece is **ICMP error generation**. Implementing it
is the right next step if we ever want our stack to be a good network
citizen — currently, peers that send us packets we can't deliver get
no feedback at all, which makes debugging the *peer's* problem
impossible. The work is well-scoped: add an `ICMPError` method to the
handler, call it from `ip.Handler.process` when various drop paths
hit, and craft the standard "IP header + 8 bytes of original payload"
structure that error messages carry.

---

## What is next

ICMP completes the diagnostic plumbing. The stack can now be probed
(`ping` from outside) and can probe outward (`icmpH.Ping(...)` from
inside). The next two protocols are the heavier ones:

1. **UDP** — connectionless, datagram-oriented, almost as simple as
   ICMP but with port numbers. The pseudo-header checksum we set up in
   `pkg/ip` already (with `Add16` and `PseudoHeaderChecksum`) is what
   the UDP and TCP layers will use to verify their segments.
2. **TCP** — the substantial one. Connection state, three-way handshake,
   reliable delivery via sequence numbers and retransmission, sliding-
   window flow control, congestion control. A full TCP implementation
   is roughly the size of everything else we have built so far,
   combined.

UDP is next because its surface is small and it gives us a working
transport layer to host higher-level demos against (DNS clients, a tiny
HTTP-over-UDP, etc.) while we build the TCP machinery in parallel.

The full source is at
[rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack).
