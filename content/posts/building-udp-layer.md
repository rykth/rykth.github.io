---
title: "Building the UDP layer in Go"
date: 2026-05-24T10:00:00+00:00
description: "The seventh post in my TCP/IP stack series. ICMP let us answer ping; UDP is the first real transport layer — it adds per-application port numbers, an optional checksum, and the Listen/Dial Conn abstraction that all subsequent protocols build on. After this post we can run socat, netcat, and other UDP clients against our stack."
tags: [golang, networking, tcpip, udp, rfc768]
---

## Where this fits

By now the stack has a device, an Ethernet codec, ARP, IPv4 with
fragmentation, a `net.Registry` that routes by EtherType, and an
[ICMP handler](/posts/building-icmp-layer) that answers `ping`. The
shape of every upper-layer protocol has been established: implement
`ip.UpperHandler`, claim a protocol number, parse a header, dispatch.

[UDP](https://datatracker.ietf.org/doc/html/rfc768) is the first
protocol where dispatching by protocol number is not enough. IPv4
delivers all UDP traffic to the same handler, but inside that handler,
multiple *applications* on the same host need to be told apart. UDP
solves that by adding 16-bit port numbers to each datagram — and once
you have ports, you suddenly need an API for binding to them, an API
for sending from them, and a state machine for connecting two endpoints
together. That API is the bulk of this post.

The code lives at
[rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack), in
`pkg/udp`. Three files:

- `udp.go` — the 8-byte header codec and the pseudo-header checksum.
- `conn.go` — the `Conn` type: receive buffer, send shortcut, close
  semantics. The shape every UDP application talks to.
- `handler.go` — the `ip.UpperHandler` that demultiplexes inbound
  datagrams by port to the right `Conn`, plus `Listen` and `Dial`.

```
 ┌─────────────────────────────────────────┐
 │   Application (DNS, syslog, QUIC…)      │
 ├─────────────────────────────────────────┤
 │   UDP ◄─────────────────────────────────│  ← this post
 ├─────────────────────────────────────────┤
 │   ICMP            Network (IPv4)        │
 ├─────────────────────────────────────────┤
 │   Link (Ethernet) + ARP                 │
 ├─────────────────────────────────────────┤
 │   Device (loopback / TAP)               │
 └─────────────────────────────────────────┘
```

---

## What is UDP?

UDP is the minimum-viable transport layer. It does three things on top
of IP, and only three:

1. **Multiplexes by port number.** Two 16-bit ports — source and
   destination — let many independent flows of UDP traffic share a
   single IP address.
2. **Adds an optional integrity check.** A 16-bit checksum covers the
   header, payload, and a pseudo-header containing the IP addresses.
   The checksum is optional in IPv4 — a value of 0 means "the sender
   chose not to compute one."
3. **Carries the length.** The header records the length of the
   header+payload, redundant with the IP header's length but useful when
   UDP is carried over a transport that doesn't.

What UDP explicitly does *not* do:

- No connection setup or teardown.
- No retransmission. If the network drops your datagram, it stays
  dropped.
- No ordering. Two datagrams sent in sequence can arrive in reverse.
- No flow control. The sender can blast the receiver as fast as it
  wants; if the receiver can't keep up, datagrams are dropped.
- No congestion control. UDP traffic does not back off in response to
  network congestion. This is famously the source of "bufferbloat"
  patterns in misconfigured networks.

These omissions are the point. UDP is the right transport for
applications that either don't need reliability (a real-time video
codec would rather drop a frame than wait for it) or that implement
their own reliability layer on top (QUIC, DNS-over-UDP retries,
application-layer RPC framings). When TCP arrives in the next post we
will pay for every one of these features in code; UDP gets the same
underlying IP layer for free.

### The 8-byte header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Length             |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Data (variable length)                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Four fields, all 16 bits, all big-endian:

- **Source Port** — the sender's port. Echoed by the receiver as the
  destination of any reply.
- **Destination Port** — the receiver's port. This is what the kernel
  uses to find which application gets the datagram.
- **Length** — total UDP segment length in bytes, including the 8-byte
  header. Minimum value is 8 (a header with no payload). Maximum is
  65,535, though in practice this is further constrained by the IP MTU.
- **Checksum** — the RFC 1071 ones'-complement checksum over the IPv4
  pseudo-header, the UDP header (with the checksum field zeroed), and
  the payload. A value of 0 indicates the sender opted out.

### The pseudo-header

UDP's checksum does not cover only UDP bytes. It covers a synthesised
*pseudo-header* that includes the IP source, destination, protocol
number, and UDP length:

```
 0                   1                   2                   3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source IP (4 bytes)                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination IP (4 bytes)                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Zero     |  Protocol=17  |          UDP Length            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

These bytes are never put on the wire. They exist solely to be summed
into the checksum so that a receiver can verify the datagram was
delivered to the right endpoint. Without the pseudo-header, a
mis-routed UDP datagram (delivered to the wrong IP by a corrupted
router) would pass checksum validation. With it, any divergence between
"what the sender addressed" and "what the receiver saw" is caught.

This is the same algorithm TCP uses with its own pseudo-header. It is
why we built `ip.Add16` and `ip.PseudoHeaderChecksum` in the
[IP post](/posts/building-ip-layer) — UDP is the first protocol that
actually uses them.

---

## Exploring UDP from the command line

UDP has fewer dedicated tools than TCP or ICMP, but the ones that exist
are sharp. Between `socat`, `nc -u`, `tcpdump`, `ss`, and the iptables
`-p udp` rules, every state and code path in our handler can be driven
from the outside.

### Send and receive raw UDP datagrams

```bash
# Listen on UDP port 9999 (run in one terminal)
nc -ul 9999
socat -u UDP-LISTEN:9999,fork -          # newer alternative

# Send a datagram (run in another terminal)
echo "hello" | nc -u -w1 10.0.0.2 9999
echo "hello" | socat - UDP-DATAGRAM:10.0.0.2:9999

# Interactive UDP session: type, press enter, read replies
nc -u 10.0.0.2 9999

# Send a datagram from a specific source port
nc -u -p 12345 10.0.0.2 9999
```

`nc -u` is the workhorse. `socat -u` (uppercase U means "unidirectional")
is the alternative when you want to bind a long-running listener that
processes many connections. `socat` is more verbose but also more
flexible — it can mux UDP onto stdin/stdout, a TCP socket, a file, or
another UDP endpoint.

For our stack, `nc -u 10.0.0.2 9999` is the canonical client probe:
when our UDP handler `Listen`s on port 9999 and we send a datagram from
the kernel, the bytes should flow through our handler to the bound
`Conn`'s `ReadFrom`.

### Watch UDP on the wire

```bash
# All UDP on the interface, verbose
sudo tcpdump -i tap0 -n -vv udp

# Specific port (matches source OR destination)
sudo tcpdump -i tap0 -n udp port 9999

# Source port or destination port specifically
sudo tcpdump -i tap0 -n udp src port 9999
sudo tcpdump -i tap0 -n udp dst port 9999

# By address + port pair
sudo tcpdump -i tap0 -n "udp and host 10.0.0.2 and port 9999"

# Hex dump including the full UDP header
sudo tcpdump -i tap0 -n -X udp port 9999

# Match datagrams with the don't-checksum sentinel (checksum field = 0)
sudo tcpdump -i tap0 -n 'udp[6:2] == 0'
```

A typical verbose line:

```
20:42:10.182104 IP (tos 0x0, ttl 64, id 17829, offset 0, flags [DF],
                    proto UDP (17), length 33)
    10.0.0.1.52341 > 10.0.0.2.9999: UDP, length 5
        0x0000:  4500 0021 45a5 4000 4011 c4f2 0a00 0001
        0x0010:  0a00 0002 cc75 270f 000d 1d77 6865 6c6c
        0x0020:  6f
```

`length 33` is the IP-level length (20-byte IP header + 8-byte UDP
header + 5-byte payload). `UDP, length 5` reports the payload size.
At offset `0x16` you can read the UDP header bytes: `cc75` is the
source port (52341), `270f` is the destination port (9999), `000d` is
the UDP length (13 = 8 + 5), and `1d77` is the checksum.

### List bound UDP sockets

The kernel exposes its UDP socket table through `ss` (the modern
`netstat`):

```bash
# UDP only, numeric ports, show owning process
ss -ulnp

# Same plus all states (UDP only has UNCONN; this is a habit-forming
# flag worth keeping)
ss -ulnap

# Only listening sockets on a specific port
ss -ulnp 'sport == 9999'

# Connected UDP sockets (those that called connect(2))
ss -unp state established
```

```
State    Recv-Q  Send-Q  Local Address:Port   Peer Address:Port
UNCONN   0       0       0.0.0.0:53           0.0.0.0:*           users:(("systemd-resolve",pid=541,fd=18))
UNCONN   0       0       0.0.0.0:9999         0.0.0.0:*
UNCONN   0       0       127.0.0.1:38123      0.0.0.0:*
```

The `Local Address:Port` column maps directly onto the keys in our
handler's `conns map[uint16]*Conn`. Every bound port in our stack would
show up here if we were the kernel.

`Recv-Q` is the byte count of buffered datagrams not yet read by the
application. Our `Conn`'s `rxCh` channel of capacity 64 is the
equivalent — it can hold 64 in-flight datagrams before back-pressure
kicks in and the handler starts dropping.

### Generate, replay, and corrupt UDP traffic

`nping` (from the nmap suite) is a Swiss army knife for crafted UDP:

```bash
# Send 5 UDP datagrams with a custom payload
sudo nping --udp -p 9999 -c 5 --data-string "hello" 10.0.0.2

# Send from a specific source port
sudo nping --udp -p 9999 --source-port 12345 -c 1 10.0.0.2

# Set the IP TTL
sudo nping --udp -p 9999 --ttl 1 -c 1 10.0.0.2

# Custom data length (random bytes)
sudo nping --udp -p 9999 --data-length 1000 -c 1 10.0.0.2

# Send to a port that has no listener — expect ICMP Port Unreachable in reply
sudo nping --udp -p 65000 -c 1 10.0.0.2
```

And `scapy` for full byte-level control:

```bash
sudo python3 - <<'PY'
from scapy.all import *

# Standard datagram
sendp(Ether(dst='aa:bb:cc:dd:ee:01') /
      IP(src='10.0.0.1', dst='10.0.0.2') /
      UDP(sport=12345, dport=9999) /
      b'hello', iface='tap0')

# Checksum = 0 (sender opted out — must still be accepted)
sendp(Ether(dst='aa:bb:cc:dd:ee:01') /
      IP(src='10.0.0.1', dst='10.0.0.2') /
      UDP(sport=12345, dport=9999, chksum=0) /
      b'no-checksum', iface='tap0')

# Deliberately bad checksum
sendp(Ether(dst='aa:bb:cc:dd:ee:01') /
      IP(src='10.0.0.1', dst='10.0.0.2') /
      UDP(sport=12345, dport=9999, chksum=0x1234) /
      b'broken', iface='tap0')

# Length field smaller than the actual payload
sendp(Ether(dst='aa:bb:cc:dd:ee:01') /
      IP(src='10.0.0.1', dst='10.0.0.2') /
      UDP(sport=12345, dport=9999, len=4) /
      b'garbage', iface='tap0')
PY
```

The first two test the happy path and the RFC 768 "zero checksum =
opted out" path. The third exercises `TestHandler_BadChecksum_Drop`'s
real-wire counterpart — the datagram must arrive but produce no
delivery and no error. The fourth tests
`TestParse_LengthFieldTooSmall`.

### Block, rate-limit, and rewrite UDP

```bash
# Drop incoming UDP on port 9999
sudo iptables -A INPUT -p udp --dport 9999 -j DROP

# Reject (sends ICMP Port Unreachable back to sender)
sudo iptables -A INPUT -p udp --dport 9999 -j REJECT --reject-with icmp-port-unreachable

# Rate-limit incoming UDP to 5 datagrams per second per source
sudo iptables -A INPUT -p udp -m hashlimit \
    --hashlimit-mode srcip --hashlimit-name udp \
    --hashlimit-above 5/s -j DROP

# Show byte and packet counts
sudo iptables -L INPUT -v -n

# Clear the test rules
sudo iptables -F
```

`-j REJECT --reject-with icmp-port-unreachable` is the standard "no
listener on this port" reply. Our stack does *not* generate these
right now — when a UDP datagram arrives for a port with no `Conn`
bound, we silently drop. This is on the limitations list.

### Kernel UDP statistics and tunables

```bash
# Counters that map onto our handler's code paths
nstat -a | grep -i udp
netstat -s -p udp

# Receive buffer ceilings (per-socket)
cat /proc/sys/net/core/rmem_default
cat /proc/sys/net/core/rmem_max
cat /proc/sys/net/ipv4/udp_mem        # global "low pressure high" triple

# Ephemeral port range used by the kernel for connect(2)
cat /proc/sys/net/ipv4/ip_local_port_range

# UDP send/receive memory accounting
cat /proc/sys/net/ipv4/udp_wmem_min
cat /proc/sys/net/ipv4/udp_rmem_min
```

```
Udp:
    InDatagrams: 4521
    NoPorts: 12
    InErrors: 0
    OutDatagrams: 4519
    RcvbufErrors: 0
    SndbufErrors: 0
```

`NoPorts` is the count of UDP datagrams that arrived for a port with no
listener. `InErrors` is checksum failures and malformed segments — our
`Parse` and checksum verification both increment counters in this
column. `OutDatagrams` is the equivalent of every `h.sender.Send` call
our handler makes.

`ip_local_port_range` is the kernel's ephemeral range — usually
`32768 60999`. Our handler uses `40000–65535`, deliberately overlapping
with the upper half of the kernel's range so that a single host running
both our stack and kernel UDP traffic does not have port collisions
(although the two namespaces are separate, so technically no collision
is possible).

---

## The implementation

Three files, three concerns. We will walk through them in the order an
incoming datagram traverses them: codec first, handler second, Conn
third. The transmit path goes the other direction, so we will revisit
the codec at the end through the `send` lens.

### `Header` — four 16-bit fields

```go
const HeaderLen = 8

type Header struct {
    SrcPort  uint16
    DstPort  uint16
    Length   uint16
    Checksum uint16
}
```

That's it. Every field is fixed-size, every field is comparable, the
struct itself is 8 bytes and copies cheaply. The smallest header of
any protocol in the stack — much smaller than IP's 20 bytes or
Ethernet's 14.

### `Parse` — header decode without checksum verification

```go
func Parse(b []byte) (Header, error) {
    if len(b) < HeaderLen {
        return Header{}, ErrHeaderTooShort
    }
    var h Header
    h.SrcPort = binary.BigEndian.Uint16(b[0:2])
    h.DstPort = binary.BigEndian.Uint16(b[2:4])
    h.Length = binary.BigEndian.Uint16(b[4:6])
    h.Checksum = binary.BigEndian.Uint16(b[6:8])
    if int(h.Length) < HeaderLen {
        return Header{}, ErrHeaderTooShort
    }
    return h, nil
}
```

Two validations. Buffer must be at least 8 bytes (or indexing panics).
The declared `Length` field must be at least 8 (or the segment claims
to contain no header at all, which makes the rest of the bytes
uninterpretable).

Notably absent: any check that `len(b)` matches `h.Length`. The IP
layer has already clipped the payload to the IP `TotalLen`, so the
UDP segment we receive should already be the right size. Re-checking
here would catch a bug in the IP handler at the cost of an extra
comparison on every datagram, and our handler treats length mismatches
the same way it would treat checksum mismatches — by dropping
silently.

Also notably absent from `Parse`: any checksum verification. This is
deliberate. The checksum depends on the IP source and destination —
information `Parse` doesn't have. Checksum verification happens in the
handler's `Deliver`, which gets the IP addresses passed in.

### `Payload` — aliasing slice

```go
func Payload(b []byte) []byte {
    if len(b) <= HeaderLen {
        return nil
    }
    return b[HeaderLen:]
}
```

`Payload` returns a slice into `b`. The caller is responsible for
copying it before reusing the underlying buffer. This is the same
zero-copy contract as `ethernet.Parse`'s `Payload` field: codec stays
allocation-free, the user of the slice decides whether it needs to be
retained.

In practice, the handler always copies the payload before handing it
to a `Conn`. The aliasing is only visible to code that calls `Parse`
and `Payload` directly without going through the handler — which is
exactly what the tests do, and what gives them clean assertions about
"the same bytes."

### `ComputeChecksum` — three checksums combined

```go
func ComputeChecksum(src, dst [4]byte, h Header, payload []byte) uint16 {
    var hdrBuf [HeaderLen]byte
    binary.BigEndian.PutUint16(hdrBuf[0:], h.SrcPort)
    binary.BigEndian.PutUint16(hdrBuf[2:], h.DstPort)
    binary.BigEndian.PutUint16(hdrBuf[4:], h.Length)
    // hdrBuf[6:8] = 0 (checksum field zeroed for computation)

    pseudoCS := ip.PseudoHeaderChecksum(src, dst, ip.ProtocolUDP, h.Length)
    udpCS    := ip.Add16(ip.Checksum(hdrBuf[:]), ip.Checksum(payload))
    return ip.Add16(pseudoCS, udpCS)
}
```

This is the most algorithmically dense function in the package, and the
cleanest illustration of why we built `ip.Add16` in the IP post. The
checksum is computed over three byte sequences that are *not contiguous
in memory*:

1. The 12-byte pseudo-header (synthesised in `ip.PseudoHeaderChecksum`).
2. The 8-byte UDP header, with the checksum field zeroed.
3. The variable-length payload.

If we wanted a single `Checksum(bytes)` call, we would have to
materialise all three concatenated into one buffer — extra allocation
and copy. Instead, we compute three partial checksums (the pseudo-
header, the header, the payload) and combine them with `Add16`. The
mathematical identity is `Add16(C(A), C(B)) == C(A || B)`, so this
produces the same result as the concatenated version, with zero extra
allocation.

The header buffer is stack-allocated as `[HeaderLen]byte`, which means
the entire function runs without touching the heap.

The trick that makes the zeroed-checksum-field work is the same as
in the IP and ICMP layers: `hdrBuf[6:8]` is implicitly zero because Go
zero-initialises stack arrays, so the checksum field contributes
nothing to the partial sum.

### `Marshal` — fixed-shape encoding

```go
func Marshal(dst []byte, h Header) error {
    if len(dst) < HeaderLen {
        return ErrHeaderTooShort
    }
    binary.BigEndian.PutUint16(dst[0:2], h.SrcPort)
    binary.BigEndian.PutUint16(dst[2:4], h.DstPort)
    binary.BigEndian.PutUint16(dst[4:6], h.Length)
    binary.BigEndian.PutUint16(dst[6:8], h.Checksum)
    return nil
}
```

`Marshal` writes only the 8-byte header — the caller is responsible
for appending the payload. The function takes the `Checksum` field
verbatim, on the assumption that the caller has either set it to
`ComputeChecksum(...)` or to `0` to opt out. This is the inverse of
`Parse`, which decoded the checksum without verifying it.

### The `Handler` — port demultiplexing

```go
type Handler struct {
    sender  Sender
    localIP [4]byte

    mu      sync.RWMutex
    conns   map[uint16]*Conn
    nextEph uint32 // protected by mu; offset into ephemeral range
}
```

Four fields. The `Sender` interface (which `*ip.Handler` satisfies) is
the upward dependency, identical in pattern to ICMP. `localIP` is the
source address for every outbound datagram. `conns` is the demuxer's
state: a map from destination port to the `Conn` listening on it.
`nextEph` tracks the next port to try when allocating an ephemeral
port for `Dial`.

The lock is `sync.RWMutex` because the read pattern dominates — every
`Deliver` does one read, while only `Listen`, `Dial`, and `Close`
mutate the map.

### `Deliver` — the receive path

```go
func (h *Handler) Deliver(src, dst [4]byte, payload []byte) {
    hdr, err := Parse(payload)
    if err != nil {
        return
    }
    data := Payload(payload)

    // RFC 768: a checksum of 0 means the sender did not compute one — accept.
    if hdr.Checksum != 0 {
        if ComputeChecksum(src, dst, hdr, data) != hdr.Checksum {
            return
        }
    }

    h.mu.RLock()
    conn, ok := h.conns[hdr.DstPort]
    h.mu.RUnlock()
    if !ok {
        return
    }

    // Connected Conns only accept datagrams from their specific remote.
    if conn.remote != nil {
        if conn.remote.IP != src || conn.remote.Port != hdr.SrcPort {
            return
        }
    }

    cp := make([]byte, len(data))
    copy(cp, data)
    conn.deliver(Datagram{
        Src:     Addr{IP: src, Port: hdr.SrcPort},
        Dst:     Addr{IP: dst, Port: hdr.DstPort},
        Payload: cp,
    })
}
```

Six steps: parse, payload, checksum, port lookup, connected-conn
filter, deliver. Each step is a discrete drop reason — bad parse, bad
checksum, no listener, wrong remote — and each one is silent. UDP is
best-effort; the senders cannot tell what happened to their datagram.

Three details are worth pausing on.

**The zero-checksum opt-out is explicit.** RFC 768 specifies that a
checksum of 0 on the wire means "the sender did not compute one." The
*on-the-wire* value `0` is reserved as a sentinel because the actual
checksum can never legitimately be 0 — a checksum of 0 in the math
is rewritten to `0xFFFF` (the ones'-complement of 0 in a 16-bit field).
`TestHandler_ZeroChecksum_Accepted` pins this behaviour.

**Connected-Conn filtering is per-receive.** A `Conn` created via
`Dial` (remote != nil) accepts datagrams only from its bound remote.
Frames from any other source are dropped at this layer, even though
the destination port matches. This is the UDP equivalent of
`connect(2)` semantics in the BSD socket API. `TestHandler_Dial_ConnectedFilter`
exercises both branches.

**The payload is copied before delivery.** The IP layer's contract says
the buffer is reusable after `Deliver` returns. The Conn might hold
onto the payload indefinitely (an application might buffer received
datagrams in a queue), so the boundary copy here is the same pattern
we used in the IP and ICMP layers. The cost is one allocation per
received datagram; the alternative is forcing every application code
path to copy on its own, which is worse.

### `Listen` — bind to a known port

```go
func (h *Handler) Listen(port uint16) (*Conn, error) {
    h.mu.Lock()
    defer h.mu.Unlock()
    if _, exists := h.conns[port]; exists {
        return nil, ErrPortInUse
    }
    c := newConn(Addr{IP: h.localIP, Port: port}, nil, h)
    h.conns[port] = c
    return c, nil
}
```

Trivial. Take the lock, check uniqueness, allocate the `Conn` with
`remote = nil` (the server-side, listen-mode signal), register, return.
The sentinel error pattern is the same as
`net.ErrDeviceAlreadyRegistered` and `ip.ErrDuplicateProto` — every
registration in this stack uses the same shape.

### `Dial` — bind to an ephemeral port, target a remote

```go
func (h *Handler) Dial(_ context.Context, remote Addr) (*Conn, error) {
    h.mu.Lock()
    defer h.mu.Unlock()
    port, err := h.allocEphemeralLocked()
    if err != nil {
        return nil, err
    }
    c := newConn(Addr{IP: h.localIP, Port: port}, &remote, h)
    h.conns[port] = c
    return c, nil
}

func (h *Handler) allocEphemeralLocked() (uint16, error) {
    n := h.nextEph
    h.nextEph++
    for i := uint32(0); i < ephemeralRange; i++ {
        port := uint16(uint32(minEphemeralPort) + (n+i)%ephemeralRange)
        if _, exists := h.conns[port]; !exists {
            return port, nil
        }
    }
    return 0, ErrNoEphemeralPort
}
```

The ephemeral allocator is a rotating linear scan over the range
`[40000, 65535]`. `nextEph` is the hint — start the search there —
and the modulo means the scan wraps cleanly past 65535 back to 40000.
The full range is 25,536 ports, which can be exhausted in principle
but never in practice for a stack at our scale.

The context parameter is currently unused. It is part of the signature
for forward compatibility — once we have a Bind-style API that might
block on contention (think: SO_REUSEPORT in the kernel), the context
will let callers cancel the wait. For now, `Dial` is non-blocking and
the parameter is `_`. `TestHandler_ConcurrentDial` runs 16 concurrent
`Dial` calls and verifies every one gets a unique port — the lock
serialises the allocations.

### `Conn` — the user-facing API

```go
type Conn struct {
    rxCh    chan Datagram
    done    chan struct{}
    local   Addr
    remote  *Addr // nil if unconnected (Listen mode)
    handler *Handler
    once    sync.Once
}
```

Six fields. `rxCh` is a buffered channel of received datagrams; the
handler pushes into it from `Deliver`, the user pulls from it via
`ReadFrom`. `done` is the close signal — a closed channel, the same
pattern we used in the
[loopback device](/posts/building-a-loopback-device) — that
unblocks any in-flight `ReadFrom`. `local` and `remote` are the
addressing endpoints. `handler` is a back-pointer for `Close` (to
deregister the port) and `WriteTo` (to invoke `send`). `once` makes
`Close` idempotent.

### `deliver` — the receive side enqueue

```go
func (c *Conn) deliver(dg Datagram) {
    select {
    case <-c.done:
        return // conn closed; discard
    default:
    }
    select {
    case c.rxCh <- dg:
    case <-c.done:
    default:
        // backpressure: drop
    }
}
```

Two `select`s, same as the loopback device's `Write`. The first is the
fast pre-check — if the Conn is already closed, drop immediately
without even attempting the send. The second is the actual send, with
three branches: enqueued, closed during enqueue, or no buffer space.

The "no buffer space" `default` is what gives this a back-pressure
drop semantics. The `rxCh` has capacity 64; if the user isn't reading,
the 65th datagram gets dropped. This is exactly what UDP applications
expect — if you don't drain your receive socket, the kernel drops
incoming packets.

### `ReadFrom` — the receive side dequeue

```go
func (c *Conn) ReadFrom(ctx context.Context) (Datagram, error) {
    select {
    case dg := <-c.rxCh:
        return dg, nil
    case <-c.done:
        return Datagram{}, ErrConnClosed
    case <-ctx.Done():
        return Datagram{}, ctx.Err()
    }
}
```

Three-way `select`: datagram arrives, conn is closed, context cancels.
This is the canonical Go pattern for cancellable blocking dequeue.

The three branches are exhaustive and orthogonal — every blocked
`ReadFrom` exits through exactly one of them. The
`TestConn_ReadFrom_ContextCancel` test exercises the context branch
with a 30 ms timeout. `TestConn_Close_UnblocksReadFrom` exercises the
`done` branch by closing the Conn while a goroutine is blocked in
`ReadFrom`.

### `Read` and `Write` — the connected-only shortcuts

```go
func (c *Conn) Read(ctx context.Context) ([]byte, error) {
    if c.remote == nil {
        return nil, ErrNotConnected
    }
    dg, err := c.ReadFrom(ctx)
    if err != nil {
        return nil, err
    }
    return dg.Payload, nil
}

func (c *Conn) Write(ctx context.Context, payload []byte) error {
    if c.remote == nil {
        return ErrNotConnected
    }
    return c.WriteTo(ctx, payload, *c.remote)
}
```

For connected Conns (those created via `Dial`), these wrappers expose
a simpler byte-oriented API: just bytes in, bytes out, no addressing
ceremony. The connected-only enforcement is the explicit
`ErrNotConnected` return, pinned by `TestConn_Read_NotConnected` and
`TestConn_Write_NotConnected`.

The split between `Read`/`Write` (connected) and `ReadFrom`/`WriteTo`
(unconnected) is the same pattern as Go's standard `net.UDPConn`. It
keeps the connected-mode API stripped down while letting unconnected
Conns still send to or receive from arbitrary peers.

### `Close` — deregister, broadcast, no double-close

```go
func (c *Conn) Close() error {
    c.once.Do(func() {
        c.handler.deregister(c.local.Port)
        close(c.done)
    })
    return nil
}
```

`sync.Once` makes the close idempotent without any conditional logic.
The `Do` closure runs exactly once; subsequent calls are no-ops and
return nil. This is the cleanest idiom for "execute exactly once
forever after."

The close order matters slightly: deregister from the handler's map
*first*, then close the `done` channel. Doing it in this order means
that any `Deliver` call already in flight when `Close` runs will either
(a) acquire the map read lock before our deregister, find the entry,
and successfully enqueue (followed by the user reading it before
shutting down) or (b) acquire after, find nothing, and drop. Either is
fine; what we avoid is the race where the channel is closed while a
`Deliver` is mid-send to it (which would panic).

### `send` — the transmit path

```go
func (h *Handler) send(ctx context.Context, src, dst Addr, payload []byte) error {
    length := uint16(HeaderLen + len(payload))
    hdr := Header{
        SrcPort: src.Port,
        DstPort: dst.Port,
        Length:  length,
    }
    hdr.Checksum = ComputeChecksum(src.IP, dst.IP, hdr, payload)

    buf := make([]byte, int(length))
    if err := Marshal(buf, hdr); err != nil {
        return fmt.Errorf("udp: marshal header: %w", err)
    }
    copy(buf[HeaderLen:], payload)

    return h.sender.Send(ctx, dst.IP, ip.ProtocolUDP, buf)
}
```

Five steps: compute the segment length, build the header (with
checksum), allocate the buffer, marshal the header, append the
payload, hand to IP. The header is built with `Checksum` set to its
real value before `Marshal` runs, which is what differentiates this
from the IP and ICMP layers — there, the checksum field had to be
zeroed first because `Marshal` is what computed it. Here, we compute
the checksum ourselves (the function needs the IP addresses, which
`Marshal` doesn't have) and just write the result into the field.

Like ICMP, `send` does one allocation per segment. Reusing buffers
through a `sync.Pool` is a low-effort optimisation if profiling ever
shows it mattering.

---

## The tests

Twenty-five tests in three blocks: codec (7), handler (8), and Conn
(10). The Conn block is the largest because the user-facing API has
the most surface to cover — context cancellation, close idempotency,
connected vs unconnected filtering.

```bash
# Plain run
go test ./pkg/udp/... -v

# With the race detector
CGO_ENABLED=1 go test ./pkg/udp/... -race -v
```

A clean run:

```
=== RUN   TestParse_Valid
--- PASS: TestParse_Valid (0.00s)
=== RUN   TestParse_TooShort
--- PASS: TestParse_TooShort (0.00s)
=== RUN   TestParse_LengthFieldTooSmall
--- PASS: TestParse_LengthFieldTooSmall (0.00s)
=== RUN   TestMarshal_RoundTrip
--- PASS: TestMarshal_RoundTrip (0.00s)
=== RUN   TestMarshal_TooSmall
--- PASS: TestMarshal_TooSmall (0.00s)
=== RUN   TestComputeChecksum_Verify
--- PASS: TestComputeChecksum_Verify (0.00s)
=== RUN   TestComputeChecksum_Corruption
--- PASS: TestComputeChecksum_Corruption (0.00s)
=== RUN   TestHandler_Protocol
--- PASS: TestHandler_Protocol (0.00s)
=== RUN   TestHandler_Listen_Deliver
--- PASS: TestHandler_Listen_Deliver (0.00s)
=== RUN   TestHandler_Listen_PortInUse
--- PASS: TestHandler_Listen_PortInUse (0.00s)
=== RUN   TestHandler_Listen_UnknownPort_Drop
--- PASS: TestHandler_Listen_UnknownPort_Drop (0.02s)
=== RUN   TestHandler_BadChecksum_Drop
--- PASS: TestHandler_BadChecksum_Drop (0.02s)
=== RUN   TestHandler_ZeroChecksum_Accepted
--- PASS: TestHandler_ZeroChecksum_Accepted (0.00s)
=== RUN   TestHandler_Dial
--- PASS: TestHandler_Dial (0.00s)
=== RUN   TestHandler_Dial_ConnectedFilter
--- PASS: TestHandler_Dial_ConnectedFilter (0.02s)
=== RUN   TestHandler_ConcurrentDial
--- PASS: TestHandler_ConcurrentDial (0.00s)
=== RUN   TestConn_ReadFrom_ContextCancel
--- PASS: TestConn_ReadFrom_ContextCancel (0.03s)
=== RUN   TestConn_Close_UnblocksReadFrom
--- PASS: TestConn_Close_UnblocksReadFrom (0.01s)
=== RUN   TestConn_Close_Idempotent
--- PASS: TestConn_Close_Idempotent (0.00s)
=== RUN   TestConn_ReadFrom_AfterClose
--- PASS: TestConn_ReadFrom_AfterClose (0.00s)
=== RUN   TestConn_Write_NotConnected
--- PASS: TestConn_Write_NotConnected (0.00s)
=== RUN   TestConn_Read_NotConnected
--- PASS: TestConn_Read_NotConnected (0.00s)
=== RUN   TestConn_WriteTo_Sends
--- PASS: TestConn_WriteTo_Sends (0.00s)
=== RUN   TestConn_Connected_ReadWrite
--- PASS: TestConn_Connected_ReadWrite (0.00s)
=== RUN   TestConn_LocalAddr_String
--- PASS: TestConn_LocalAddr_String (0.00s)
PASS
ok      github.com/rykth/tcp-ip-stack/pkg/udp   1.111s
```

### The codec tests

Seven tests cover `Parse`, `Marshal`, and `ComputeChecksum`. The
interesting ones:

**`TestParse_LengthFieldTooSmall`** — manually craft a header with the
Length field set to 7. Bytes-on-the-wire say "this is a UDP segment of
total length 7," which is impossible because the header alone is 8
bytes. `Parse` must return `ErrHeaderTooShort`. This is the only
internal consistency check the codec does, and it catches a
particularly nasty class of malformed input that would otherwise look
plausible.

**`TestComputeChecksum_Verify`** — build a frame, parse it back,
recompute the checksum with the same IP addresses, assert that the
recomputed value equals the stored value. This is the round-trip
invariant for the pseudo-header algorithm.

**`TestComputeChecksum_Corruption`** — build a valid frame, then XOR
one byte of the payload, then recompute. The recomputed checksum must
differ from the stored one. This is the property that lets `Deliver`
detect tampering or transmission errors.

### The handler tests

Eight tests cover `Deliver`, `Listen`, `Dial`. Highlights:

**`TestHandler_Listen_Deliver`** — the canonical positive case: bind
port 9999, inject a frame from a remote, call `ReadFrom` and verify
the `Datagram` has the right `Src`, `Dst`, and `Payload`.

**`TestHandler_Listen_PortInUse`** — second `Listen` on the same port
returns `ErrPortInUse`. The whole point of having the sentinel error
is that integration code can branch on it (e.g., retry with a
different port).

**`TestHandler_BadChecksum_Drop`** — build a valid frame, XOR one byte
of the checksum field, deliver, then call `ReadFrom` with a 20 ms
timeout. The expected error is `context.DeadlineExceeded`, proving
that the bad-checksum frame produced *no* delivery. This is the test
that pins the silent-drop policy.

**`TestHandler_ZeroChecksum_Accepted`** — build a frame with checksum
explicitly set to 0 (the RFC 768 opt-out signal). Deliver, then call
`ReadFrom`. The datagram must arrive even though the checksum field
"doesn't match" what we would have computed — because per RFC 768, 0
means "do not check."

**`TestHandler_Dial_ConnectedFilter`** — call `Dial` to get a
connected Conn bound to `remote`. Deliver a frame *from* `remote` — it
must arrive. Then deliver a frame from a *different* source IP with
the same source port — it must be filtered out (drop, no delivery). The
test uses a 20 ms timeout on the second `ReadFrom` to assert absence.

**`TestHandler_ConcurrentDial`** — launch 16 goroutines, each calling
`Dial` simultaneously. Every Conn returned must have a unique local
port. The lock around `allocEphemeralLocked` is what makes this work;
without it, two goroutines could both pick `40000` and `conns[40000]`
would silently overwrite.

### The Conn tests

Ten tests, the most of any block. Highlights:

**`TestConn_ReadFrom_ContextCancel`** — `ReadFrom` with a 30 ms
context-deadline. Must return `context.DeadlineExceeded`. Verifies the
context branch of the three-way select.

**`TestConn_Close_UnblocksReadFrom`** — start a `ReadFrom` in a
goroutine, sleep 5 ms (to ensure the goroutine has actually blocked on
the channel), close the Conn from the main goroutine. The `ReadFrom`
must return `ErrConnClosed` within one second.

**`TestConn_Close_Idempotent`** — two consecutive `Close` calls. Both
must succeed. The `sync.Once` makes this trivially correct.

**`TestConn_Write_NotConnected`** and **`TestConn_Read_NotConnected`** —
both API errors. A `Listen`-created Conn must reject `Write` and
`Read` because there is no remote to send to or filter from. The two
errors are not symmetric — `Write` has nowhere to send; `Read` has
nowhere to filter — but the sentinel is shared.

**`TestConn_Connected_ReadWrite`** — the full round-trip:
- `Dial` to get a connected Conn.
- `Write` a request payload. Wait for the mock sender to record it.
- Inject a reply frame via `Deliver`.
- `Read` the reply.

This is the test that proves a UDP "client" round-trip works
end-to-end. It is the closest thing in the suite to a real application
pattern.

---

## Driving it with `nc` over TAP

The full client/server demo is twenty lines of Go on the stack side
and one `nc` invocation on the kernel side. The stack:

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/rykth/tcp-ip-stack/pkg/arp"
    "github.com/rykth/tcp-ip-stack/pkg/ethernet"
    "github.com/rykth/tcp-ip-stack/pkg/ip"
    netpkg "github.com/rykth/tcp-ip-stack/pkg/net"
    "github.com/rykth/tcp-ip-stack/pkg/raw/tuntap"
    "github.com/rykth/tcp-ip-stack/pkg/udp"
)

func main() {
    dev, _ := tuntap.New("tap0", tuntap.DeviceTAP)
    defer dev.Close()

    localMAC := ethernet.Addr{0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0x02}
    localIP  := [4]byte{10, 0, 0, 2}

    arpH  := arp.NewHandler(localMAC, localIP, dev)
    table := ip.NewTable()
    onLink, _ := ip.ParseNetwork("10.0.0.0/24")
    table.Add(ip.Route{Network: onLink, Dev: dev})
    ipH  := ip.NewHandler(localMAC, localIP, arpH, table)
    udpH := udp.NewHandler(ipH, localIP)
    ipH.RegisterUpper(udpH) //nolint:errcheck

    r := netpkg.New()
    r.AddDevice(dev);   r.AddHandler(arpH); r.AddHandler(ipH)

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    go r.Start(ctx) //nolint:errcheck

    // Echo server on UDP port 9999.
    srv, err := udpH.Listen(9999)
    if err != nil { log.Fatal(err) }
    defer srv.Close()
    fmt.Println("echo server listening on 10.0.0.2:9999")

    for {
        dg, err := srv.ReadFrom(ctx)
        if err != nil { return }
        fmt.Printf("got %q from %v — echoing back\n", dg.Payload, dg.Src)
        if err := srv.WriteTo(ctx, dg.Payload, dg.Src); err != nil {
            log.Printf("WriteTo: %v", err)
        }
    }
}
```

Bring up the TAP, run the stack, and probe with `nc -u`:

```bash
# Terminal 1 — TAP setup + stack
sudo ip tuntap add dev tap0 mode tap user $USER
sudo ip link set tap0 up
sudo ip addr add 10.0.0.1/24 dev tap0
go run ./examples/udp

# Terminal 2 — watch the wire
sudo tcpdump -i tap0 -n -e -vv 'udp or arp'

# Terminal 3 — probe with netcat
echo "hello world" | nc -u -w1 10.0.0.2 9999
```

The first `nc` triggers a full ARP resolution (our ARP handler
replies), then a UDP datagram from a kernel-assigned ephemeral port to
our `10.0.0.2:9999`. Our server echoes it back. `nc` prints
`hello world` and exits.

```
ARP, Request who-has 10.0.0.2 tell 10.0.0.1, length 28
ARP, Reply 10.0.0.2 is-at aa:bb:cc:dd:ee:02, length 28
IP 10.0.0.1.40123 > 10.0.0.2.9999: UDP, length 12
IP 10.0.0.2.9999 > 10.0.0.1.40123: UDP, length 12
```

The stack:

```
echo server listening on 10.0.0.2:9999
got "hello world\n" from 10.0.0.1:40123 — echoing back
```

For an interactive test, run `nc -u 10.0.0.2 9999` (no `-w1`) and type
lines. Each line is sent as its own datagram and echoed back; the
session continues until you hit Ctrl-D.

To exercise `Dial` outward — our stack sending UDP to a kernel-side
server — add a tiny client to the program:

```go
go func() {
    time.Sleep(time.Second)
    cli, _ := udpH.Dial(ctx, udp.Addr{IP: [4]byte{10, 0, 0, 1}, Port: 7777})
    defer cli.Close()
    cli.Write(ctx, []byte("ping from stack")) //nolint:errcheck
    ctx2, cancel := context.WithTimeout(ctx, time.Second)
    defer cancel()
    reply, err := cli.Read(ctx2)
    if err != nil { return }
    fmt.Printf("reply from kernel: %q\n", reply)
}()
```

And on the kernel side, run a `nc -u -l 7777` listener that echoes:

```bash
nc -u -l 7777 | while read line; do echo "$line back"; done
```

The stack's `Dial` allocates an ephemeral port in `[40000, 65535]`,
the `Write` becomes a UDP segment from that port to `10.0.0.1:7777`,
and the kernel's `nc -u -l 7777` receives it. The reply from `nc`
flows back through ARP-cached MAC → kernel → TAP → our IP handler →
UDP demuxer → the connected Conn's `rxCh`. The whole cycle takes
about 200 microseconds.

---

## Design decisions, revisited

### Why the connected/unconnected distinction?

A `Listen` Conn is bound to a local port but accepts datagrams from
*any* source. A `Dial` Conn is bound to a local port *and* filtered to
one specific remote. The distinction maps onto how BSD sockets work:
`bind(2)` without `connect(2)` is unconnected; both is connected.

The benefit is per-receive filtering. A client that has `Dial`ed a
specific server doesn't want to see traffic from other senders — those
would only be possible via attacks or misconfigurations, and silently
dropping them is the right safety policy. Without `Dial`, the
application code would have to inspect every received datagram's `Src`
field and decide whether to accept or drop.

The shorthand `Read`/`Write` API is only available on connected Conns
because it elides the addressing entirely. For unconnected mode you
must use `ReadFrom`/`WriteTo` and address explicitly.

### Why is `nextEph` a `uint32` and not a `uint16`?

The ephemeral range is 16-bit (`40000–65535`), so a `uint16` would
technically suffice. But the modulo-based scan would overflow on
increment if `nextEph` were a `uint16` near the boundary, and
correcting that with `(n + i) % ephemeralRange` requires the wider
type. Using `uint32` avoids the entire class of arithmetic bugs at
the cost of two extra bytes of state — a clear trade.

### Why is `rxCh` capacity 64?

It is the same as the Registry's per-handler RxChan capacity. Sixty-
four is large enough that bursty traffic (e.g., a flood of DNS
responses, or a syslog forwarder catching up) does not drop datagrams
under normal application read rates. It is small enough that a slow
or hung consumer applies back-pressure before consuming gigabytes of
memory — important because UDP is the one transport where there is no
sender-side flow control.

For applications that need a larger buffer (think: streaming media
ingest), exposing a `WithRxBuffer(n int)` option would be the
obvious extension. We don't have it yet because no caller has needed
it.

### Why does `Conn.Close` deregister before closing `done`?

The order matters under concurrent `Deliver` and `Close`. With
deregister-first, an in-flight `Deliver` either:

1. Acquires the read lock before our `deregister` runs, finds the
   Conn, calls `deliver()` — which checks `done` (still open),
   enqueues, returns. The user reads the datagram, then sees `done`
   close.
2. Acquires the read lock *after* our `deregister`, finds nothing, no
   delivery happens.

Both outcomes are fine. What we avoid: a `deliver()` mid-call when
`done` has already been closed and the channel send would not race
(channels are safe to receive-on-closed, but the `select { case
c.rxCh <- dg: }` would still proceed normally and the consumer would
see the datagram via the closed-channel "drain" path before the `done`
signal — slightly surprising but not unsafe). Doing deregister first
shortens the window where this happens.

If we ever switched the order, no test in the suite would fail
deterministically — the issue is purely about predictability of
behaviour under tight races.

---

## Limitations

| Limitation | Detail |
|------------|--------|
| **No fragmentation by us** | We hand the segment to the IP layer in one piece. If `len(payload) + 8 > MTU`, the IP layer fragments. This is correct, but means UDP cannot enforce a "max datagram size" check at our boundary |
| **No ICMP Port Unreachable** | Datagrams arriving for unbound ports are silently dropped. RFC compliance would have us return ICMP Type 3 Code 3. Same gap as `pkg/icmp` |
| **Drop-on-back-pressure** | The Conn's 64-frame channel drops on overflow with no signal to the application. A real socket reports `EAGAIN`; ours gives only silence |
| **No IP_PKTINFO** | A multi-homed Listen Conn cannot tell which *local* IP a datagram was addressed to. The `Datagram.Dst` field always reports the handler's `localIP`, even for broadcast or multicast traffic |
| **No port reuse** | A single port can hold exactly one Conn. The kernel's `SO_REUSEPORT` would let multiple Conns share one port for load distribution |
| **No multicast** | `Listen(port)` only handles unicast addressed to our `localIP`. Multicast group membership is not implemented |
| **No traffic class / TOS** | The IP TOS field is always 0. QoS-marked traffic is not distinguishable |
| **Single local IP** | A multi-homed host needs one handler per IP. The handler does not select a source address based on routing |

The biggest missing feature is **ICMP Port Unreachable generation**.
Implementing it would require both (a) a clean way for UDP to signal
"no listener" to the ICMP layer and (b) ICMP code that can construct
the Type 3 Code 3 packet. Both are well-scoped pieces of work; we will
revisit them when we round out the ICMP error subset.

---

## What is next

UDP is the easier of the two transport protocols. With it in place, the
stack can run any UDP-based application: DNS clients, syslog senders,
NTP probes, anything QUIC-shaped. The Conn/Listen/Dial abstractions
established here are the template every higher-level API in the stack
will mimic.

Next is **TCP** — the substantial one. The wire format is denser (the
header is at least 20 bytes), but the real work is the state machine:
SYN → SYN-ACK → ACK for handshake, sequence numbers for reliable
delivery, retransmission timers, sliding-window flow control, and at
least one of the standard congestion control algorithms. TCP will
probably be larger than every other layer combined, and split across
half a dozen files.

The full source is at
[rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack).
