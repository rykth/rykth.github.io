---
title: "Building the TCP layer in Go"
date: 2026-05-25T10:00:00+00:00
description: "The eighth post in my TCP/IP stack series — and the one the whole series has been pointing at. UDP gave us per-port multiplexing in one screen of code; TCP needs an 11-state RFC 793 machine, ring buffers with 32-bit wrapping sequence numbers, Jacobson/Karels RTT estimation, Karn's retransmission algorithm, and one goroutine per connection. After this post we can curl, ssh, nc, and socat against our stack."
tags: [golang, networking, tcpip, tcp, rfc793, rfc6298]
---

## Where this fits

We have a loopback and TUN/TAP device, an Ethernet codec on top of a
`net.Registry`, ARP for IP→MAC resolution, IPv4 with fragmentation and
routing, ICMP for `ping`, and a [UDP](/posts/building-udp-layer)
transport with Listen/Dial Conn semantics. Every protocol so far has
been an extension of the previous: same packet codec shape, same
handler abstraction, same `Sender` interface to push down into IP.

[TCP](https://datatracker.ietf.org/doc/html/rfc793) breaks that
pattern. TCP isn't a packet codec — it's a *protocol*, with state, with
time, with reliability semantics that the layer below cannot provide.
A UDP packet is delivered or it isn't, and either way the conversation
ends there. A TCP segment is delivered or it isn't, and either way the
conversation continues: lost segments are retransmitted, out-of-order
segments are buffered and reassembled, the receiver tells the sender
how fast to go, both sides agree on when they're done.

The package is the largest in the stack so far — seven files of
implementation plus a test file:

```
pkg/tcp/
├── tcp.go        — Header, Options, Parse/Marshal, ComputeChecksum/VerifyChecksum
├── state.go      — the 11 RFC 793 connection states
├── buffer.go     — SendBuffer + RecvBuffer with 32-bit wrapping sequence numbers
├── timer.go      — Jacobson/Karels RTT estimator + Karn's exponential-backoff timer
├── conn.go       — the per-connection run() goroutine that implements the state machine
├── listener.go   — passive open with a backlog channel
└── handler.go    — net.ProtocolHandler: 4-tuple demux, ephemeral port allocation, sendRST
```

The code lives at
[rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack), in `pkg/tcp`.

```
 ┌─────────────────────────────────────────┐
 │   Application (HTTP, SSH, curl…)        │
 ├─────────────────────────────────────────┤
 │   TCP ◄─────────────────────────────────│  ← this post
 ├─────────────────────────────────────────┤
 │   UDP    ICMP    Network (IPv4)         │
 ├─────────────────────────────────────────┤
 │   Link (Ethernet) + ARP                 │
 ├─────────────────────────────────────────┤
 │   Device (loopback / TAP)               │
 └─────────────────────────────────────────┘
```

---

## What is TCP?

TCP is the protocol that turns IP's "best-effort packet delivery"
into a reliable byte stream. Where UDP says "here is a datagram, take
it or leave it," TCP says "here is a byte stream, I will redeliver
whatever is lost until the bytes arrive in order, the receiver tells me
how fast to send, both sides explicitly agree on start and end."

Concretely, TCP layers six things on top of IP:

1. **Per-application demultiplexing.** Same 16-bit port numbers as
   UDP, but the keying is by full 4-tuple `(local IP, local port,
   remote IP, remote port)` rather than just by local port.
2. **Reliable delivery.** Every byte is identified by a 32-bit
   sequence number. The receiver acknowledges what it has received; the
   sender retransmits anything that goes unacknowledged before a
   timeout.
3. **In-order delivery.** Even when segments arrive out of order, TCP
   buffers them and presents the bytes to the application in the order
   the sender produced them.
4. **Connection setup and teardown.** The famous three-way handshake
   (`SYN`, `SYN+ACK`, `ACK`) lets both sides agree on initial
   sequence numbers and verify reachability before any data flows. The
   matching four-way close (`FIN`, `ACK`, `FIN`, `ACK`) lets either
   side end its half of the conversation independently.
5. **Flow control.** Each side advertises a receive window — how many
   bytes it can accept before the next acknowledgement. The sender
   never transmits more than the window allows. This is what prevents
   a fast sender from overwhelming a slow receiver.
6. **Congestion control.** The sender slows down when the network
   drops segments. Modern stacks use sophisticated algorithms (CUBIC,
   BBR); ours implements only the most rudimentary form — bound by
   the peer's advertised window and a fixed-cap retransmission timer.

What we are *not* implementing in this post: full congestion control,
selective acknowledgements (SACK), window scaling, persistent
connections, keepalives, urgent data, or any of the dozen RFC
extensions that production stacks accumulated over thirty years. The
result is a TCP implementation that interoperates with the kernel and
runs `curl`, `nc`, `socat`, and `ssh` correctly, but would not survive
in a high-throughput production environment.

### The 20-byte header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window             |
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (variable)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

A lot has happened in 20 bytes:

- **Source / Destination Port (2 bytes each)** — same as UDP. The
  difference is that TCP keys connections by the full 4-tuple, not just
  the destination port, which means two connections from different
  sources can both arrive at the same local port.
- **Sequence Number (4 bytes)** — the absolute position of the first
  byte of this segment's payload within the sender's byte stream.
  Wraps at 2^32 — about 4 GiB of data — which the sliding window
  comparison logic must handle correctly.
- **Acknowledgment Number (4 bytes)** — when the ACK flag is set,
  this is the next byte the receiver expects. Equivalently, every byte
  with a sequence number strictly less than this has been received.
- **Data Offset (4 bits)** — header length in 32-bit words. Minimum
  5 (= 20 bytes). Anything more means options follow.
- **Flags (6 bits)** — six control bits:
  - `URG` — urgent data; not used in modern TCP.
  - `ACK` — acknowledgment number is valid.
  - `PSH` — push: deliver this segment to the application immediately.
  - `RST` — reset: abort the connection.
  - `SYN` — synchronise sequence numbers (handshake start).
  - `FIN` — no more data from sender (graceful close).
- **Window (2 bytes)** — how many bytes the receiver will accept
  beyond the acknowledgment number. The maximum useful value without
  the window-scale option is 65,535.
- **Checksum (2 bytes)** — same RFC 1071 ones'-complement algorithm
  as UDP, over the same IPv4 pseudo-header structure.
- **Urgent Pointer (2 bytes)** — only meaningful when URG is set;
  ignored otherwise.
- **Options (0–40 bytes)** — MSS, window scale, SACK permitted,
  timestamps. We parse all of them; we set none.

### The three-way handshake

Connection setup is three segments, exchanged in a specific order:

```
Client                              Server
  │                                     │
  │  SYN seq=A                          │
  │ ─────────────────────────────────► │
  │                                     │
  │           SYN+ACK seq=B, ack=A+1    │
  │ ◄───────────────────────────────── │
  │                                     │
  │  ACK seq=A+1, ack=B+1               │
  │ ─────────────────────────────────► │
  │                                     │
  │   (both sides ESTABLISHED)          │
```

The state-machine moves are:

- Client: `CLOSED → SYN_SENT` on `Dial`, then `SYN_SENT →
  ESTABLISHED` when the SYN-ACK arrives.
- Server: `LISTEN` from the start, then `LISTEN → SYN_RECEIVED` when
  the SYN arrives, then `SYN_RECEIVED → ESTABLISHED` when the ACK
  arrives.

The initial sequence numbers (`A` and `B`) are randomly chosen by each
side — RFC 793 says any value works, RFC 6528 says it should be
unpredictable to prevent off-path attacks. Our `newISS()` calls
`crypto/rand`.

### The four-way close

Either side can close its half of the connection independently. The
canonical "client closes first" path is:

```
Client                              Server
  │                                     │
  │  FIN seq=A                          │
  │ ─────────────────────────────────► │
  │  (FIN_WAIT_1)                       │
  │                                     │
  │              ACK ack=A+1            │
  │ ◄───────────────────────────────── │
  │  (FIN_WAIT_2)              (CLOSE_WAIT)
  │                                     │
  │             (server still sending)
  │                                     │
  │              FIN seq=B              │
  │ ◄───────────────────────────────── │
  │                            (LAST_ACK)
  │                                     │
  │  ACK ack=B+1                        │
  │ ─────────────────────────────────► │
  │  (TIME_WAIT)              (CLOSED)
  │                                     │
  │  ... wait 2×MSL ...                 │
  │                                     │
  │  (CLOSED)
```

Eleven states total, with eleven transitions. Almost half the
implementation is the code that walks this graph.

---

## Exploring TCP from the command line

TCP has more tooling than any other protocol — both because it is the
most important, and because thirty-five years of debugging
half-failed connections has produced specialised tools for every
imaginable broken state.

### List TCP sockets

`ss` is the modern frontend:

```bash
# All TCP, numeric, with owning process
ss -tnp

# Listening sockets only
ss -tnlp

# Established connections
ss -tnp state established

# Sockets in TIME_WAIT (interesting for our 2×MSL state)
ss -tnp state time-wait

# Show retransmission timer + congestion control + RTT
ss -tnpo

# Full socket statistics: window, MSS, RTT, retransmits, CWND
ss -tnipo

# Filter by 4-tuple
ss -tnp 'src 10.0.0.2 and sport == 9000'
ss -tnp 'dst 10.0.0.1 and dport == 80'
```

`-i` is the one you reach for when debugging TCP performance — it
expands every connection's row to include the kernel's view of the
connection state:

```
ESTAB 0      0      10.0.0.2:9000     10.0.0.1:42178  users:(("tcp-stack",pid=12345,fd=4))
     cubic wscale:7,7 rto:204 rtt:0.067/0.012 ato:40 mss:1448 cwnd:10
     bytes_sent:418 bytes_acked:418 bytes_received:6 segs_out:5 segs_in:4
     send 1.728Gbps lastsnd:8 lastrcv:8 lastack:8 pacing_rate 3.456Gbps
     delivered:5 rcv_space:14600 rcv_ssthresh:64088 minrtt:0.057
```

Almost every number here corresponds to state in our implementation:
`rto:204` (milliseconds) is the analogue of our `RTTEstimator.RTO()`,
`rtt:0.067/0.012` is `srtt/rttvar`, `mss:1448` is the MSS option,
`cwnd:10` is the congestion window (we don't have one), `wscale:7,7`
is window scaling (we don't have one), `segs_out` / `segs_in` are
counters that map onto every `transmit()` and `Deliver()` in our
handler.

The legacy `netstat`:

```bash
netstat -tnp
netstat -tnlp
netstat -tn | awk '/^tcp/{print $6}' | sort | uniq -c   # state histogram
```

### Watch TCP on the wire

```bash
# All TCP, verbose, with relative sequence numbers
sudo tcpdump -i tap0 -n tcp

# Absolute sequence numbers (-S) — useful when debugging seq math
sudo tcpdump -i tap0 -n -S tcp

# Specific port either side
sudo tcpdump -i tap0 -n tcp port 9000

# By flag combinations — SYN only (handshake initiation)
sudo tcpdump -i tap0 -n 'tcp[tcpflags] == tcp-syn'

# SYN-ACK (handshake response)
sudo tcpdump -i tap0 -n 'tcp[tcpflags] & (tcp-syn|tcp-ack) == (tcp-syn|tcp-ack)'

# RST traffic (something went wrong)
sudo tcpdump -i tap0 -n 'tcp[tcpflags] & tcp-rst != 0'

# FIN traffic (close handshakes)
sudo tcpdump -i tap0 -n 'tcp[tcpflags] & tcp-fin != 0'

# Hex dump of the full segment, header included
sudo tcpdump -i tap0 -n -X tcp port 9000

# Save to pcap; Wireshark renders the state machine beautifully
sudo tcpdump -i tap0 -w /tmp/tcp.pcap tcp
```

A typical handshake on `tcpdump -n -S`:

```
14:02:01.182104 IP 10.0.0.1.42178 > 10.0.0.2.9000: Flags [S],
    seq 3735928559, win 65535, options [mss 1460], length 0
14:02:01.182125 IP 10.0.0.2.9000 > 10.0.0.1.42178: Flags [S.],
    seq 2882400000, ack 3735928560, win 65535, length 0
14:02:01.182148 IP 10.0.0.1.42178 > 10.0.0.2.9000: Flags [.],
    ack 2882400001, win 65535, length 0
```

`[S]` is SYN, `[S.]` is SYN+ACK (the period is ACK), `[.]` is ACK
alone. The ACK number in line 2 is `seq from line 1 + 1`, and the
ACK number in line 3 is `seq from line 2 + 1`. SYN consumes one
sequence number even though it carries no payload — same as FIN.

### Throughput and latency benchmarks

```bash
# iperf3 — the standard tool for TCP throughput
# Server side
iperf3 -s -p 5201

# Client side
iperf3 -c 10.0.0.2 -p 5201 -t 30 -P 4

# Pin the congestion control algorithm for comparison
iperf3 -c 10.0.0.2 -C cubic
iperf3 -c 10.0.0.2 -C bbr

# Reverse direction (server sends to client)
iperf3 -c 10.0.0.2 -R

# Latency probe via TCP
sudo nping --tcp -p 9000 --flags syn -c 10 10.0.0.2

# hping3 — old but flexible
sudo hping3 -S -p 9000 -c 5 10.0.0.2
```

Our implementation will not match production stacks on `iperf3` —
no congestion control, no window scaling, no GSO/GRO offload — but
it should produce stable single-digit-to-tens-of-megabits throughput
on loopback or a TAP, which is enough to exercise the receive and
retransmission paths.

### Crafted segments with scapy

```bash
sudo python3 - <<'PY'
from scapy.all import *
ip = IP(src='10.0.0.1', dst='10.0.0.2')

# Plain SYN — triggers passiveOpen on the server side
sendp(Ether(dst='aa:bb:cc:dd:ee:01')/ip/TCP(sport=12345, dport=9000, flags='S', seq=1000), iface='tap0')

# SYN with options — MSS + Window Scale (option parsing path)
sendp(Ether(dst='aa:bb:cc:dd:ee:01')/ip/TCP(sport=12345, dport=9000,
       flags='S', seq=1000,
       options=[('MSS', 1460), ('WScale', 7), ('SAckOK','')]), iface='tap0')

# Bare RST — should cause us to drop the connection
sendp(Ether(dst='aa:bb:cc:dd:ee:01')/ip/TCP(sport=12345, dport=9000, flags='R'), iface='tap0')

# Bad checksum — should be silently dropped by Deliver
sendp(Ether(dst='aa:bb:cc:dd:ee:01')/ip/TCP(sport=12345, dport=9000,
       flags='S', seq=1, chksum=0x1234), iface='tap0')

# Out-of-window ACK — should provoke a RST from us
sendp(Ether(dst='aa:bb:cc:dd:ee:01')/ip/TCP(sport=12345, dport=9000,
       flags='A', seq=1, ack=2), iface='tap0')

# Bare ACK for a 4-tuple we don't have — should provoke RST
sendp(Ether(dst='aa:bb:cc:dd:ee:01')/ip/TCP(sport=12345, dport=8080,
       flags='A', seq=1, ack=1), iface='tap0')
PY
```

The last two exercise `handler.go`'s `sendRST` path — the "I have no
record of this connection, send RST" behaviour mandated by RFC 793
§3.4 for any segment that arrives for an unknown 4-tuple.

### Block, RST-inject, and rate-limit TCP

```bash
# Drop all SYNs to port 9000 (simulates a closed port)
sudo iptables -A INPUT -p tcp --dport 9000 --syn -j DROP

# Send a TCP RST in response to any SYN to port 9000 (visible to client as ECONNREFUSED)
sudo iptables -A INPUT -p tcp --dport 9000 --syn -j REJECT --reject-with tcp-reset

# Rate-limit new connections (SYN floods)
sudo iptables -A INPUT -p tcp --syn -m limit --limit 10/s --limit-burst 20 -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -j DROP

# Block established connections (force the stack to retry from scratch)
sudo iptables -A INPUT -p tcp --tcp-flags ALL ACK -j DROP
```

`tcp-reset` is the rejection mode that most precisely tests our
"unexpected RST" handling: if the kernel-side `iptables` injects a
RST into the middle of a connection, our `Conn.run` should transition
the connection to `CLOSED`, signal `ErrConnReset` to `Read`/`Write`,
and return promptly without leaking the goroutine.

### Kernel TCP tunables and metrics

```bash
# Aggregate TCP counters
nstat -a | grep -i tcp
netstat -s -p tcp

# Current congestion control algorithm
sysctl net.ipv4.tcp_congestion_control

# Available algorithms (loadable via modprobe)
sysctl net.ipv4.tcp_available_congestion_control

# Per-route TCP metrics (smoothed RTT, congestion state)
ip tcp_metrics show

# Send/receive buffer sizing
cat /proc/sys/net/ipv4/tcp_rmem    # min default max
cat /proc/sys/net/ipv4/tcp_wmem

# SYN backlog
cat /proc/sys/net/ipv4/tcp_max_syn_backlog
cat /proc/sys/net/core/somaxconn

# SYN-cookie defence against SYN floods
cat /proc/sys/net/ipv4/tcp_syncookies

# Keepalive defaults
cat /proc/sys/net/ipv4/tcp_keepalive_time
cat /proc/sys/net/ipv4/tcp_keepalive_intvl
cat /proc/sys/net/ipv4/tcp_keepalive_probes

# Time-Wait recycling
cat /proc/sys/net/ipv4/tcp_tw_reuse
cat /proc/sys/net/ipv4/tcp_fin_timeout
```

```
Tcp:
    RtoAlgorithm: 1
    RtoMin: 200
    RtoMax: 120000
    MaxConn: -1
    ActiveOpens: 5234
    PassiveOpens: 8712
    AttemptFails: 12
    EstabResets: 4
    CurrEstab: 17
    InSegs: 482314
    OutSegs: 471092
    RetransSegs: 432       <—— the canonical retransmission counter
    InErrs: 0
    OutRsts: 87
    InCsumErrors: 0
```

`ActiveOpens` is the count of successful `Dial` calls;
`PassiveOpens` is the count of successful `Accept`s; `RetransSegs`
is incremented by our `handleRetransmit`. `InCsumErrors` is the
counter the `VerifyChecksum` failure path corresponds to. If our
counters and the kernel's diverged under identical traffic, we would
have a behavioural discrepancy to investigate.

### Inspect TCP offload features

```bash
# Show offload status for an interface
ethtool -k tap0

# Disable specific offloads (useful when debugging seq math against tcpdump)
sudo ethtool -K tap0 tso off gso off gro off lro off
```

When TSO (TCP Segmentation Offload) is on, the kernel hands a huge
"segment" to the NIC and the NIC fragments it on the wire. `tcpdump`
captures *before* TSO, so you see one giant segment that never
actually existed on the wire. Disabling TSO is the standard first step
in any TCP debugging session that involves comparing `tcpdump` output
against the seq/ack math your code is doing.

---

## The implementation

Seven files. We walk through them in dependency order: types and codec
first, then the buffers and timers, then the connection state machine,
then the handler that glues it all to the IP layer.

### `Header`, `Options`, `Parse`, `Marshal`

```go
type Header struct {
    SrcPort    uint16
    DstPort    uint16
    SeqNum     uint32
    AckNum     uint32
    DataOffset uint8 // in 32-bit words; min 5
    Flags      uint8 // FlagFIN | FlagSYN | FlagRST | FlagPSH | FlagACK | FlagURG
    Window     uint16
    Checksum   uint16
    Urgent     uint16
    Options    Options
}
```

Ten fields plus an embedded `Options` struct. The `Flags` field is
exposed as a `uint8` and accessed via constants:

```go
const (
    FlagFIN uint8 = 0x01
    FlagSYN uint8 = 0x02
    FlagRST uint8 = 0x04
    FlagPSH uint8 = 0x08
    FlagACK uint8 = 0x10
    FlagURG uint8 = 0x20
)

func (h Header) HasFlag(f uint8) bool { return h.Flags&f != 0 }
```

The bit layout matches the wire format — `0x01` is the FIN bit in the
control octet, etc. — so reading and writing the flags from the
serialised header is a single byte assignment, no shift-and-mask
gymnastics.

The Options struct holds the four extensions we recognise but never
emit:

```go
type Options struct {
    MSS           uint16
    WindowScale   uint8
    SACKPermitted bool
    Timestamps    bool
    TSVal         uint32
    TSEcr         uint32
}
```

`parseOptions` walks the variable-length options region as a
sequence of `(kind, length, value)` TLVs. Two option kinds (NOP=1
and End=0) have no length byte; everything else uses the
length-prefixed encoding. We recognise MSS (kind 2), Window Scale (3),
SACK Permitted (4), SACK (5, which we skip past but don't store),
Timestamp (8); unknown options have their length read and the parser
skips ahead.

```go
func parseOptions(opts []byte) Options {
    var o Options
    i := 0
    for i < len(opts) {
        switch opts[i] {
        case optEnd:
            return o
        case optNOP:
            i++
        case optMSS:
            if i+4 > len(opts) { return o }
            o.MSS = binary.BigEndian.Uint16(opts[i+2 : i+4])
            i += 4
        case optWScale:
            ...
        case optSACKPerm:
            ...
        case optTimestamp:
            ...
        default:
            if i+1 >= len(opts) { return o }
            length := int(opts[i+1])
            if length < 2 { return o }
            i += length
        }
    }
    return o
}
```

Two guards make this safe against malformed options: every kind that
takes a length checks `i+N > len(opts)` before reading, and the
default branch validates `length >= 2` to prevent infinite loops on a
zero-length unknown option.

`Parse` is the standard form: validate length, validate DataOffset,
read the fixed fields, recurse into `parseOptions` for the variable
region:

```go
func Parse(b []byte) (Header, error) {
    if len(b) < MinHeaderLen {
        return Header{}, ErrHeaderTooShort
    }
    do := b[12] >> 4
    if do < 5 {
        return Header{}, ErrBadDataOffset
    }
    headerLen := int(do) * 4
    if len(b) < headerLen {
        return Header{}, ErrHeaderTooShort
    }
    h := Header{
        SrcPort:    binary.BigEndian.Uint16(b[0:2]),
        DstPort:    binary.BigEndian.Uint16(b[2:4]),
        SeqNum:     binary.BigEndian.Uint32(b[4:8]),
        AckNum:     binary.BigEndian.Uint32(b[8:12]),
        DataOffset: do,
        Flags:      b[13],
        Window:     binary.BigEndian.Uint16(b[14:16]),
        Checksum:   binary.BigEndian.Uint16(b[16:18]),
        Urgent:     binary.BigEndian.Uint16(b[18:20]),
    }
    if headerLen > MinHeaderLen {
        h.Options = parseOptions(b[MinHeaderLen:headerLen])
    }
    return h, nil
}
```

`Marshal` is the inverse, writing only the fixed 20-byte portion. The
caller is responsible for any options region — we never emit options,
so the Marshal call is always exactly 20 bytes.

### `ComputeChecksum` and `VerifyChecksum`

```go
func ComputeChecksum(src, dst [4]byte, hdrBuf, payload []byte) uint16 {
    segLen := uint16(len(hdrBuf) + len(payload))
    pseudoCS := ip.PseudoHeaderChecksum(src, dst, ip.ProtocolTCP, segLen)
    hdrCS := ip.Checksum(hdrBuf)
    payCS := ip.Checksum(payload)
    return ip.Add16(pseudoCS, ip.Add16(hdrCS, payCS))
}

func VerifyChecksum(src, dst [4]byte, b []byte) bool {
    segLen := uint16(len(b))
    pseudoCS := ip.PseudoHeaderChecksum(src, dst, ip.ProtocolTCP, segLen)
    segCS := ip.Checksum(b)
    result := ip.Add16(pseudoCS, segCS)
    return result == 0xFFFF || result == 0
}
```

`ComputeChecksum` is the same three-partial-sums pattern as
[UDP](/posts/building-udp-layer): pseudo-header + TCP header + payload,
combined via `ip.Add16` to avoid materialising the concatenation.

`VerifyChecksum` exploits the standard RFC 1071 property: a complete
segment (including its own correct checksum field) sums to all-ones.
The accept condition is `result == 0xFFFF || result == 0` — the two
representations of ones'-complement zero are mathematically equivalent
and either can appear in practice depending on how the bytes happen to
arrange themselves.

### The 11 states

```go
type State uint8

const (
    StateClosed      State = iota
    StateListen
    StateSynSent
    StateSynReceived
    StateEstablished
    StateFinWait1
    StateFinWait2
    StateCloseWait
    StateClosing
    StateLastAck
    StateTimeWait
)
```

The full RFC 793 §3.2 set. The `iota`-derived `uint8` representation
means a State fits in a single byte and can be compared with `==`.

`String()` returns the canonical names (`"CLOSED"`, `"LISTEN"`,
`"SYN_SENT"`, …), which makes log output and test failures readable.
This is the implementation of the standard `fmt.Stringer` interface,
which means `fmt.Printf("%v", state)` produces the protocol-spec name
without per-call ceremony.

### Sequence number arithmetic

TCP sequence numbers are 32-bit and wrap. A naive `a < b` comparison
breaks the moment `b` wraps past zero — `0x00000001 < 0xFFFFFFFE` in
real arithmetic, but in sequence-number arithmetic the second number
is "behind" the first by 3 bytes.

The fix is well known: cast the difference to a signed `int32` and
check its sign:

```go
func seqLT(a, b uint32) bool { return int32(a-b) < 0 }
func seqGT(a, b uint32) bool { return int32(b-a) < 0 }
```

The trick is that `int32(a-b)` is interpreted as a signed 32-bit
number, which represents the *signed* offset from `b` to `a` within a
2^31 window. As long as the two sequence numbers are within 2 GiB of
each other (which they always are in any sane TCP connection), this
correctly orders them across wraparound.

Both functions are used throughout the buffer code for things like
"has the new ACK number advanced past the current una?" — the kind
of question that would otherwise need a half-page of comments to
explain why we are doing modular arithmetic.

### `SendBuffer` — the unacknowledged-data ring

```go
type SendBuffer struct {
    mu         sync.Mutex
    buf        []byte
    head       uint32 // write head (absolute seq of next byte to enqueue)
    una        uint32 // oldest unacknowledged sequence number
    nxt        uint32 // next byte to send
    peerWindow uint16 // receiver's advertised window
    notEmpty   *sync.Cond
    closed     bool
}
```

Three sequence-number cursors:

- **`una`** — the oldest byte the receiver has not yet acknowledged.
  Everything in `[una, nxt)` is on the wire, waiting for an ACK.
- **`nxt`** — the next byte that will be transmitted. Everything in
  `[nxt, head)` is in the buffer but has not been put on the wire
  yet.
- **`head`** — the position where the next `Write` call will enqueue.
  Everything in `[head, head + capacity)` is free space.

The invariant is `una ≤ nxt ≤ head`, with comparison in
sequence-number arithmetic (via `seqLT`/`seqGT`). A successful ACK
advances `una`. A `transmit` advances `nxt`. A `Write` advances
`head`.

`Write` is the standard cond-variable pattern: take the lock, sleep
while the buffer is full and not closed, copy as much as fits, signal
the cond, repeat. Blocking on a full send buffer is what implements
TCP flow control at the API boundary — application `Write`s block when
the peer's window is closed and the local buffer fills up.

```go
func (sb *SendBuffer) Write(data []byte) (int, error) {
    sb.mu.Lock()
    defer sb.mu.Unlock()
    total := 0
    for len(data) > 0 {
        for !sb.closed && sb.available() == 0 {
            sb.notEmpty.Wait()
        }
        if sb.closed {
            return total, ErrBufferFull
        }
        n := sb.available()
        if n > len(data) { n = len(data) }
        sb.enqueue(data[:n])
        data = data[n:]
        total += n
        sb.notEmpty.Broadcast()
    }
    return total, nil
}
```

`available()` returns the free space: `capacity - (head - una)`. The
subtraction works across wraparound because both are `uint32`s and
their difference is interpreted modulo 2^32.

`Peek` returns up to `n` bytes from `nxt`, *clipped to the peer's
window*. This is where flow control becomes visible: even if the local
buffer has 64 KiB of unsent data, `Peek` returns nothing if the peer
has advertised a window of zero.

```go
func (sb *SendBuffer) Peek(n int) []byte {
    sb.mu.Lock()
    defer sb.mu.Unlock()
    avail := int(sb.head - sb.nxt)
    win := int(sb.peerWindow) - int(sb.nxt-sb.una)
    if win < 0 { win = 0 }
    if avail > win { avail = win }
    if avail > n { avail = n }
    if avail == 0 { return nil }
    out := make([]byte, avail)
    for i := range out {
        out[i] = sb.buf[(sb.nxt+uint32(i))%uint32(len(sb.buf))]
    }
    return out
}
```

The `win := peerWindow - (nxt - una)` computation is the *effective
window*: how many bytes the receiver will still accept after the
bytes already in flight. If we have already sent `nxt - una = 5000`
bytes and the receiver advertised a window of `8000`, only 3000 more
bytes can be sent without buffer overrun.

The wraparound-safe `(sb.nxt + uint32(i)) % uint32(len(sb.buf))`
modular indexing is what turns the linear sequence number into a ring
buffer offset. This is the only place in the codebase where modular
indexing appears, and it is intentional that it appears here and not
in user code.

### `RecvBuffer` — in-order reassembly

```go
type RecvBuffer struct {
    mu       sync.Mutex
    buf      []byte
    head     uint32 // next expected sequence number
    tail     uint32 // write pointer
    ooo      []oooSegment
    notEmpty *sync.Cond
    closed   bool
}

type oooSegment struct {
    seq  uint32
    data []byte
}
```

Two pointers and a list. `head` is where the next byte will be read
from (= RCV.NXT in RFC 793 speak), `tail` is where the next byte will
be written. Bytes in `[head, tail)` are ready to read. The `ooo`
slice holds out-of-order segments that arrived before the bytes that
precede them.

`Receive` is the interesting one. Three cases:

```go
func (rb *RecvBuffer) Receive(seq uint32, data []byte) {
    if len(data) == 0 { return }
    rb.mu.Lock()
    defer rb.mu.Unlock()

    // 1. Trim already-received prefix.
    if seqGT(rb.head, seq) {
        trim := rb.head - seq
        if trim >= uint32(len(data)) { return }
        data = data[trim:]
        seq = rb.head
    }

    // 2. In-order — append directly and merge any OOO that now fits.
    if seq == rb.head {
        rb.writeInOrder(data)
        rb.mergeOOO()
        rb.notEmpty.Broadcast()
        return
    }

    // 3. Out-of-order — insert into the sorted OOO list.
    seg := oooSegment{seq: seq, data: append([]byte(nil), data...)}
    inserted := false
    for i, s := range rb.ooo {
        if seqLT(seq, s.seq) {
            rb.ooo = append(rb.ooo[:i], append([]oooSegment{seg}, rb.ooo[i:]...)...)
            inserted = true
            break
        }
    }
    if !inserted {
        rb.ooo = append(rb.ooo, seg)
    }
}
```

Three things to notice:

**Already-received data is trimmed, not dropped.** A segment that
overlaps with already-acknowledged bytes has its overlapping prefix
chopped off. This is RFC-mandated: senders sometimes retransmit
segments that have already been delivered (the ACK was lost on its way
back), and we need to ACK the *new* bytes within the retransmitted
segment while ignoring the redundant ones.

**The OOO list is sorted by sequence number.** Insertion sort is the
right complexity here — in a healthy connection the OOO list is
empty or has one or two entries. Quicksort would be overkill.

**`mergeOOO` runs after every in-order write.** When in-order data
arrives, it may fill the gap that was holding up earlier-arrived OOO
segments. `mergeOOO` walks the sorted list and writes everything that
is now contiguous with `tail`.

```go
func (rb *RecvBuffer) mergeOOO() {
    for len(rb.ooo) > 0 {
        s := rb.ooo[0]
        if seqGT(s.seq, rb.tail) {
            break
        }
        data := s.data
        if seqGT(rb.tail, s.seq) {
            trim := rb.tail - s.seq
            if trim >= uint32(len(data)) {
                rb.ooo = rb.ooo[1:]
                continue
            }
            data = data[trim:]
        }
        rb.writeInOrder(data)
        rb.ooo = rb.ooo[1:]
    }
}
```

The trim logic inside `mergeOOO` handles the case where an OOO segment
overlaps with what we have just received — a relatively common
situation under retransmission. The full receive path is covered by
`TestRecvBuffer_OutOfOrder`, which deliberately delivers segment 2
before segment 1 and verifies that `Read` returns `"hello world"`.

### `RTTEstimator` — Jacobson/Karels by the book

```go
type RTTEstimator struct {
    mu      sync.Mutex
    srtt    time.Duration
    rttvar  time.Duration
    rto     time.Duration
    hasRTT  bool
}

const (
    minRTO = time.Second
    maxRTO = 60 * time.Second
)
```

The
[Jacobson/Karels algorithm](https://datatracker.ietf.org/doc/html/rfc6298)
maintains two state variables — the smoothed RTT (`srtt`) and the
RTT variance (`rttvar`) — and derives the retransmission timeout
from them:

```
RTO = SRTT + 4 × RTTVAR
```

On the *first* sample, RFC 6298 §2.2 specifies:

```
SRTT  = R
RTTVAR = R / 2
RTO   = SRTT + max(G, 4 × RTTVAR)
```

On *subsequent* samples (§2.3):

```
RTTVAR = (1 - β)·RTTVAR + β·|SRTT - R'|   where β = 1/4
SRTT   = (1 - α)·SRTT   + α·R'             where α = 1/8
```

The α and β constants give the estimator a long memory and a
short-but-not-too-short response time. β = 1/4 means RTTVAR mostly
remembers historical jitter but updates quickly when traffic patterns
change; α = 1/8 means SRTT is slow to drift, which prevents one
unusually fast or slow ACK from skewing the estimator.

```go
func (r *RTTEstimator) Update(sample time.Duration) {
    r.mu.Lock()
    defer r.mu.Unlock()
    if !r.hasRTT {
        r.srtt = sample
        r.rttvar = sample / 2
        r.hasRTT = true
    } else {
        diff := r.srtt - sample
        if diff < 0 { diff = -diff }
        r.rttvar = (3*r.rttvar + diff) / 4
        r.srtt = (7*r.srtt + sample) / 8
    }
    rto := r.srtt + 4*r.rttvar
    if rto < minRTO { rto = minRTO }
    if rto > maxRTO { rto = maxRTO }
    r.rto = rto
}
```

`(3*rttvar + diff) / 4` is the integer-arithmetic equivalent of
`(1 - 1/4)·rttvar + (1/4)·diff`. Similarly for SRTT with α = 1/8.
The clamp to `[1 s, 60 s]` is RFC 6298 §2.4–§2.5.

### `RetransmitTimer` — Karn's algorithm

```go
type RetransmitTimer struct {
    mu       sync.Mutex
    est      *RTTEstimator
    backoff  int
    timer    *time.Timer
    fired    chan struct{}
    stopped  bool
    maxRetry int
}
```

The timer is driven by `time.AfterFunc`, which pushes a token into
`fired` when the timeout elapses. The `Conn.run` goroutine watches
`Fired()` in its main select and reacts to retransmission events.

[Karn's algorithm](https://en.wikipedia.org/wiki/Karn%27s_algorithm)
says two things:

1. Do not measure RTT on retransmitted segments. If an ACK arrives
   after a retransmit, you cannot tell whether it is acknowledging the
   original or the retransmit.
2. On retransmission, *double* the RTO. Each successive retransmit
   doubles it again, up to a maximum.

Both are implemented by the `Backoff` method and the `Reset` method
working together:

```go
func (rt *RetransmitTimer) Reset() {
    rt.mu.Lock()
    defer rt.mu.Unlock()
    rt.backoff = 0
    rt.restart()
}

func (rt *RetransmitTimer) Backoff() bool {
    rt.mu.Lock()
    defer rt.mu.Unlock()
    rt.backoff++
    if rt.maxRetry > 0 && rt.backoff > rt.maxRetry {
        return false
    }
    rt.restart()
    return true
}
```

`Reset` is called after a fresh (non-retransmitted) transmission;
`backoff` is zeroed and the timer fires at the base RTO. `Backoff` is
called on timeout: increment the counter, restart the timer at
`base_rto * 2^backoff`, return false when we have hit `maxRetry`.

The `restart` helper applies the exponential:

```go
func (rt *RetransmitTimer) restart() {
    rto := rt.est.RTO()
    for i := range rt.backoff {
        _ = i
        rto *= 2
        if rto > maxRTO {
            rto = maxRTO
            break
        }
    }
    if rt.timer != nil { rt.timer.Stop() }
    rt.timer = time.AfterFunc(rto, func() {
        select {
        case rt.fired <- struct{}{}:
        default:
        }
    })
}
```

`maxRetry = 5` in our handler. After five backoffs, the
`Conn.run` loop treats the timeout as a terminal failure and
transitions to `CLOSED`, signalling `ErrTimeout` to any blocked
`Read`/`Write`.

### `Conn` — the connection goroutine

```go
type Conn struct {
    key     ConnKey
    handler *Handler

    state   State
    iss     uint32
    irs     uint32

    sendBuf *SendBuffer
    recvBuf *RecvBuffer
    rtt     *RTTEstimator
    retx    *RetransmitTimer

    inbound  chan segment
    closeCh  chan struct{}
    resetCh  chan struct{}
    dataKick chan struct{}

    connectErr  error
    connectDone chan struct{}

    readErr  error
    writeErr error
    errMu    sync.RWMutex

    once sync.Once
}
```

This is the largest struct in the stack. It has to be — every
connection has its own state machine, its own send and receive
buffers, its own RTT estimator, its own retransmission timer, its own
goroutine.

Three groups of fields:

1. **State** — `state` (current RFC 793 state), `iss` (our initial
   sequence number, randomly chosen), `irs` (the peer's initial
   sequence number).
2. **Buffers and timers** — `sendBuf`, `recvBuf`, `rtt`, `retx`.
3. **Communication channels** — four channels that other goroutines
   use to push events at `run`: `inbound` (segments from the
   handler), `closeCh` (`Close()` was called), `resetCh` (RST
   received), `dataKick` (`Write()` enqueued new data).

The `errMu` plus `readErr` and `writeErr` are the mechanism by which
terminal errors (RST, timeout, peer close) get surfaced to in-flight
`Read` and `Write` calls — the `run` goroutine sets them, the
user-facing methods check them.

### `run` — the event loop that drives the state machine

```go
func (c *Conn) run(ctx context.Context) {
    defer func() {
        c.retx.Stop()
        c.handler.removeConn(c.key)
        c.setReadErr(ErrConnClosed)
        c.setWriteErr(ErrConnClosed)
    }()

    if c.state == StateSynSent {
        c.sendSYN()
    }

    var timeWaitTimer <-chan time.Time

    for {
        // Opportunistic send while we have data in ESTABLISHED or CLOSE_WAIT.
        if c.state == StateEstablished || c.state == StateCloseWait {
            if data := c.sendBuf.Peek(1460); len(data) > 0 {
                c.sendData(data)
                c.sendBuf.Advance(uint32(len(data)))
                c.retx.Reset()
            }
        }

        select {
        case <-ctx.Done():
            c.sendRST(c.sendBuf.Nxt(), c.recvBuf.NextExpected())
            return

        case <-c.closeCh:
            c.handleClose()
            if c.state == StateClosed {
                return
            }

        case seg := <-c.inbound:
            if c.handleSegment(seg) {
                return
            }

        case <-c.dataKick:
            // Fall through to the send check at top of loop.

        case <-c.retx.Fired():
            if !c.handleRetransmit() {
                c.setReadErr(ErrTimeout)
                c.setWriteErr(ErrTimeout)
                return
            }

        case <-timeWaitTimer:
            return
        }

        if c.state == StateTimeWait && timeWaitTimer == nil {
            timeWaitTimer = time.After(2 * msl)
        }
    }
}
```

The structural pattern is identical to every other `Start`-style
goroutine in the stack: a `for/select` that multiplexes events
from multiple sources, all with channel-based delivery.

Six event sources:

- `ctx.Done()` — handler shutdown. Send RST, exit cleanly.
- `closeCh` — user called `Close()`. Transition into the close
  half of the state machine.
- `inbound` — a segment arrived from the handler. Process it through
  the state-specific handler function.
- `dataKick` — `Write()` enqueued bytes. Fall through to the top of
  the loop where the opportunistic send picks them up.
- `retx.Fired()` — retransmission timer expired. Back off and
  retransmit, or give up.
- `timeWaitTimer` — we have been in `TIME_WAIT` for 2×MSL. Clean
  up and exit.

The opportunistic send at the top of the loop is the data-transfer
path. Every iteration of the loop checks whether we have unsent data
and whether the peer's window allows us to send it; if so, we send,
advance `nxt`, reset the retransmit timer. The check happens at the
top of every iteration because *any* event can change the eligibility
(an ACK opens up window space; a `dataKick` adds new data; a state
transition might unblock sending).

### Per-state segment handling

`handleSegment` is a dispatch table from state to per-state handler
function. The full set is in `conn.go`:

- `handleSynSent` — looking for a SYN+ACK. Validate the ack number
  matches our ISS+1, install the peer's ISN, send the final ACK,
  transition to `ESTABLISHED`.
- `handleSynReceived` — looking for the final ACK. Validate, set
  peer window, transition to `ESTABLISHED`.
- `handleEstablished` — the main data-transfer path. Acknowledge
  newly-acked bytes, deliver new payload, ACK new data, transition on
  incoming FIN.
- `handleFinWait1` — we sent FIN; awaiting ACK and/or peer FIN. Three
  outgoing transitions: `FIN_WAIT_2` (our FIN was ACK'd),
  `CLOSING` (peer sent simultaneous FIN), `TIME_WAIT` (peer
  ACK'd and FIN'd in one segment).
- `handleFinWait2` — our FIN was ACK'd; waiting for peer FIN. One
  transition: `TIME_WAIT`.
- `handleClosing` — simultaneous close; waiting for ACK of our FIN.
  One transition: `TIME_WAIT`.
- `handleLastAck` — final state before close. One transition:
  `CLOSED` (return true from `handleSegment` to terminate the
  goroutine).

RST handling is hoisted out of the per-state switch and run first,
because it is state-independent:

```go
if hdr.HasFlag(FlagRST) {
    switch c.state {
    case StateSynSent, StateSynReceived, StateEstablished,
        StateFinWait1, StateFinWait2, StateCloseWait:
        c.connectErr = ErrConnReset
        c.signalConnect()
        c.setReadErr(ErrConnReset)
        c.setWriteErr(ErrConnReset)
        return true
    }
    return false
}
```

A RST in any active state immediately tears down the connection. The
states left out of the list (`Closing`, `LastAck`, `TimeWait`,
`Closed`, `Listen`) either already have a close in progress or are
not user-visible, so the RST is treated as redundant.

### `handleEstablished` — the data-transfer engine

```go
func (c *Conn) handleEstablished(hdr Header, payload []byte, src, dst [4]byte) bool {
    if hdr.HasFlag(FlagACK) {
        if seqGT(hdr.AckNum, c.sendBuf.Una()) {
            c.sendBuf.Acknowledge(hdr.AckNum)
            c.sendBuf.SetPeerWindow(hdr.Window)
        }
    }

    if len(payload) > 0 {
        c.recvBuf.Receive(hdr.SeqNum, payload)
        c.sendACK()
    }

    if hdr.HasFlag(FlagFIN) {
        c.recvBuf.Receive(hdr.SeqNum+uint32(len(payload)), nil)
        c.sendACK()
        c.setReadErr(ErrConnClosed)
        if c.state == StateEstablished {
            c.state = StateCloseWait
        }
    }
    return false
}
```

Three independent checks: ACK, payload, FIN. A single segment can
carry all three at once (a piggybacked ACK on data with a FIN), and
each is handled independently.

The `seqGT(hdr.AckNum, c.sendBuf.Una())` guard is what filters out
duplicate ACKs — if the ACK number is not strictly greater than what
we already had, there is no new information.

The FIN-handling path is particularly subtle: the FIN itself
*consumes a sequence number* even though it carries no payload, so
the receive-buffer call is
`c.recvBuf.Receive(hdr.SeqNum + uint32(len(payload)), nil)` — we
advance `tail` by one past the end of the payload, accounting for the
phantom FIN byte. This is what makes our ACK number on the reply
correct.

The transition to `CLOSE_WAIT` only happens if we were in
`ESTABLISHED` — re-entering the function from `FIN_WAIT_1` or
`FIN_WAIT_2` (which both call through to `handleEstablished` for the
data half) does not move us to `CLOSE_WAIT`.

### `Listener` — passive open with a backlog

```go
type Listener struct {
    port    uint16
    handler *Handler
    backlog chan *Conn
    done    chan struct{}
    once    sync.Once
}

func (l *Listener) Accept(ctx context.Context) (*Conn, error) {
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    case <-l.done:
        return nil, ErrListenerClosed
    case c := <-l.backlog:
        return c, nil
    }
}
```

A bounded channel (`backlog`) of size 128 holds connections that have
completed their three-way handshake but have not yet been pulled by
the application's `Accept` call. This is the equivalent of the
kernel's accept queue, controlled by the `backlog` parameter of the
`listen(2)` syscall.

If the application doesn't `Accept` fast enough, the channel fills and
`passiveOpen` drops the new connection on the floor (see below). In a
production stack you would want a real overflow policy (sometimes
called "syn cookies" or "accept queue overflow"), but for our purposes
the drop is fine.

### `Handler` — the 4-tuple demultiplexer

```go
type Handler struct {
    sender  Sender
    localIP [4]byte

    mu        sync.RWMutex
    conns     map[ConnKey]*Conn
    listeners map[uint16]*Listener
    nextEph   uint32

    ctx    context.Context
    cancel context.CancelFunc
}

type ConnKey struct {
    LocalIP    [4]byte
    LocalPort  uint16
    RemoteIP   [4]byte
    RemotePort uint16
}
```

Unlike UDP, which keyed connections by destination port only, TCP
keys by the full 4-tuple `(local IP, local port, remote IP, remote
port)`. This is what allows multiple connections from different
clients to share a single server port.

`Deliver` does both kinds of lookup — first the 4-tuple connection
map, then (if no connection match) the listener map for SYN segments
that are starting a new connection:

```go
func (h *Handler) Deliver(src, dst [4]byte, payload []byte) {
    hdr, err := Parse(payload)
    if err != nil { return }
    if !VerifyChecksum(src, dst, payload) { return }

    p := Payload(payload, hdr)
    seg := segment{src: src, dst: dst, hdr: hdr, payload: p}

    key := ConnKey{
        LocalIP:    dst,
        LocalPort:  hdr.DstPort,
        RemoteIP:   src,
        RemotePort: hdr.SrcPort,
    }

    h.mu.RLock()
    conn, connOK := h.conns[key]
    lst, lstOK := h.listeners[hdr.DstPort]
    h.mu.RUnlock()

    if connOK {
        conn.deliver(seg)
        return
    }

    if lstOK && hdr.HasFlag(FlagSYN) && !hdr.HasFlag(FlagACK) {
        h.passiveOpen(src, dst, hdr, lst)
        return
    }

    // No match — send RST.
    if !hdr.HasFlag(FlagRST) {
        go h.sendRST(dst, src, hdr)
    }
}
```

The last branch is RFC 793 §3.4 verbatim: any segment that arrives for
an unknown 4-tuple gets a RST in reply, *unless* the incoming segment
was itself a RST (in which case we would loop forever ping-ponging
RSTs).

`passiveOpen` is what handles the server side of the three-way
handshake:

```go
func (h *Handler) passiveOpen(src, dst [4]byte, hdr Header, lst *Listener) {
    key := ConnKey{
        LocalIP: dst, LocalPort: hdr.DstPort,
        RemoteIP: src, RemotePort: hdr.SrcPort,
    }

    c := newConn(key, h, StateSynReceived)
    c.irs = hdr.SeqNum
    c.recvBuf = NewRecvBuffer(recvBufSize, hdr.SeqNum+1)
    c.sendBuf.SetPeerWindow(hdr.Window)

    h.mu.Lock()
    if _, exists := h.conns[key]; exists {
        h.mu.Unlock()
        return
    }
    h.conns[key] = c
    h.mu.Unlock()

    go func() {
        c.sendSYNACK()
        go c.run(h.ctx)
        if err := c.waitConnect(h.ctx); err != nil {
            c.Close()
            return
        }
        lst.deliver(c)
    }()
}
```

The choreography: create a Conn in `SYN_RECEIVED` state, register it
in the 4-tuple map, send SYN+ACK, start the `run` goroutine, wait for
`run` to signal `connectDone` (which happens when the final ACK
arrives and the state transitions to `ESTABLISHED`), then push the
Conn into the listener's backlog channel. From the application's point
of view, the connection appears in `Accept` only *after* the handshake
is complete.

### `Dial` — active open

```go
func (h *Handler) Dial(ctx context.Context, localPort uint16, remote ConnKey) (*Conn, error) {
    h.mu.Lock()
    var port uint16
    if localPort == 0 {
        port, _ = h.allocEphemeralLocked()
    } else {
        port = localPort
    }

    key := ConnKey{
        LocalIP:    h.localIP,
        LocalPort:  port,
        RemoteIP:   remote.RemoteIP,
        RemotePort: remote.RemotePort,
    }
    if _, exists := h.conns[key]; exists {
        h.mu.Unlock()
        return nil, ErrPortInUse
    }
    c := newConn(key, h, StateSynSent)
    h.conns[key] = c
    h.mu.Unlock()

    go c.run(h.ctx)
    if err := c.waitConnect(ctx); err != nil {
        c.Close()
        h.mu.Lock()
        delete(h.conns, key)
        h.mu.Unlock()
        return nil, err
    }
    return c, nil
}
```

The mirror image of `passiveOpen`: allocate a port (ephemeral if
`localPort == 0`, otherwise the requested one), register the new
Conn in `SYN_SENT` state, start `run`, wait for it to signal
`connectDone` (which happens on receiving SYN+ACK and sending the
final ACK).

The `waitConnect(ctx)` is what makes `Dial` honour the caller's
context. If the context cancels before the handshake completes, we
tear down the Conn and return the context error.

---

## The tests

Twenty-three tests in four blocks: codec, state, buffers and timers,
handler. The handler block includes integration tests that wire two
handlers together in-process to exercise the full state machine.

```bash
# Plain run
go test ./pkg/tcp/... -v

# With the race detector
CGO_ENABLED=1 go test ./pkg/tcp/... -race -v
```

A clean run:

```
=== RUN   TestParse_Valid
--- PASS: TestParse_Valid (0.00s)
=== RUN   TestParse_TooShort
--- PASS: TestParse_TooShort (0.00s)
=== RUN   TestParse_BadDataOffset
--- PASS: TestParse_BadDataOffset (0.00s)
=== RUN   TestMarshal_RoundTrip
--- PASS: TestMarshal_RoundTrip (0.00s)
=== RUN   TestMarshal_BufferTooSmall
--- PASS: TestMarshal_BufferTooSmall (0.00s)
=== RUN   TestPayload
--- PASS: TestPayload (0.00s)
=== RUN   TestComputeChecksum_Verify
--- PASS: TestComputeChecksum_Verify (0.00s)
=== RUN   TestVerifyChecksum_Corruption
--- PASS: TestVerifyChecksum_Corruption (0.00s)
=== RUN   TestState_String
--- PASS: TestState_String (0.00s)
=== RUN   TestSendBuffer_WriteAndAck
--- PASS: TestSendBuffer_WriteAndAck (0.00s)
=== RUN   TestSendBuffer_PeerWindowThrottles
--- PASS: TestSendBuffer_PeerWindowThrottles (0.00s)
=== RUN   TestRecvBuffer_InOrder
--- PASS: TestRecvBuffer_InOrder (0.00s)
=== RUN   TestRecvBuffer_OutOfOrder
--- PASS: TestRecvBuffer_OutOfOrder (0.00s)
=== RUN   TestRecvBuffer_FreeSpace
--- PASS: TestRecvBuffer_FreeSpace (0.00s)
=== RUN   TestRTTEstimator_FirstSample
--- PASS: TestRTTEstimator_FirstSample (0.00s)
=== RUN   TestRTTEstimator_MultipleSamples
--- PASS: TestRTTEstimator_MultipleSamples (0.00s)
=== RUN   TestHandler_Protocol
--- PASS: TestHandler_Protocol (0.00s)
=== RUN   TestHandler_Listen_PortInUse
--- PASS: TestHandler_Listen_PortInUse (0.00s)
=== RUN   TestListener_CloseUnblocksAccept
--- PASS: TestListener_CloseUnblocksAccept (0.00s)
=== RUN   TestHandshake_Established
--- PASS: TestHandshake_Established (0.00s)
=== RUN   TestDataTransfer
--- PASS: TestDataTransfer (0.00s)
=== RUN   TestHandler_BadChecksum_Drop
--- PASS: TestHandler_BadChecksum_Drop (0.00s)
=== RUN   TestConcurrentDial
--- PASS: TestConcurrentDial (0.00s)
PASS
ok      github.com/rykth/tcp-ip-stack/pkg/tcp   1.020s
```

### The codec and state tests

Nine tests: header parse/marshal in both happy and unhappy paths,
`Payload` length, `ComputeChecksum`/`VerifyChecksum` with
corruption check, and the `String()` mapping for all 11 states.

`TestParse_BadDataOffset` is the small one worth highlighting:
DataOffset = 3 is structurally impossible (the field's value is in
32-bit words, minimum 5), and the test sets `b[12] = 0x30` to verify
that `Parse` rejects it. A buggy parser that accepted DataOffset < 5
would compute a negative header length and read past the buffer —
exactly the kind of failure mode our explicit validation prevents.

`TestVerifyChecksum_Corruption` is the integrity check: compute a
correct checksum, embed it in the segment, then XOR one byte of the
header and re-verify. The expected result is `false` — the corruption
must be detected.

### The buffer and timer tests

Seven tests covering `SendBuffer`, `RecvBuffer`, and the RTT
estimator.

**`TestSendBuffer_PeerWindowThrottles`** — set the peer window to 3
bytes, write 11 bytes to the send buffer, and verify that `Peek`
returns no more than 3. This is the flow-control invariant: even if
the local buffer has plenty of unsent data, we never transmit beyond
what the peer told us they can accept.

**`TestRecvBuffer_OutOfOrder`** — the most representative buffer
test. Two segments are delivered in reverse order: segment-at-offset-5
arrives first, then segment-at-offset-0. After both arrive,
`Read` must return `"hello world"` — the in-order concatenation.
This proves the OOO list, the sort-on-insert, and the merge-after-fill
all work correctly.

**`TestRTTEstimator_FirstSample`** — submit a 500 ms RTT sample to
a fresh estimator, check that the RTO has updated. The first-sample
formula is special-cased (`SRTT = R`, `RTTVAR = R/2`), and getting
that wrong would mean every connection starts with an unstable
RTO.

**`TestRTTEstimator_MultipleSamples`** — submit ten samples with
increasing values (50 ms, 55 ms, 60 ms, …). The final RTO must still
be within `[1 s, 60 s]` — proving the min/max clamps work.

### The handler tests

Seven tests, including the most important integration tests in the
package: full handshake and data transfer with two handlers wired
together.

**`TestHandshake_Established`** — the canonical positive case. A
client and a server share a `mockSender` pair that forwards every
`Send` into the peer's `Deliver`. The server `Listen`s on port 9000,
the client `Dial`s the server, and the test asserts that the server's
`Accept` returns a Conn. This exercises every state transition in the
three-way handshake.

**`TestDataTransfer`** — extends the handshake test with a `Write`
on the client and a `Read` on the server. The data string `"hello,
TCP!"` must round-trip in full. This is the test that proves the
data-transfer engine works end-to-end.

**`TestHandler_BadChecksum_Drop`** — craft a SYN segment with the
checksum field left at zero, deliver it, and verify no Conn was
created. The handler must drop the bad segment silently and not panic.

**`TestConcurrentDial`** — launch 5 simultaneous `Dial`s to the same
server. Each one must succeed, each must get a unique ephemeral local
port, and the listener must hand back all 5 connections via
`Accept`. The lock around `allocEphemeralLocked` and the
4-tuple-keyed conn map together guarantee uniqueness; this test pins
that guarantee.

**`TestListener_CloseUnblocksAccept`** — start an `Accept` in a
goroutine, then call `Close` on the listener from the main goroutine.
`Accept` must return promptly with `ErrListenerClosed`. This is the
broadcast-close pattern that all our Listen-style APIs use.

### The mock sender pair

The integration tests are made possible by a thin in-process bridge
that delivers each `Send` directly into the peer's `Deliver`:

```go
type mockSender struct {
    mu      sync.Mutex
    peer    *tcp.Handler
    localIP [4]byte
    peerIP  [4]byte
}

func (m *mockSender) Send(_ context.Context, dst [4]byte, _ ip.Protocol, payload []byte) error {
    m.mu.Lock()
    peer := m.peer
    m.mu.Unlock()
    if peer == nil { return nil }
    cp := make([]byte, len(payload))
    copy(cp, payload)
    peer.Deliver(m.localIP, dst, cp)
    return nil
}
```

This is a faithful simulator of a perfectly reliable IP layer: every
`Send` arrives at the peer's `Deliver` with a brief Go goroutine
scheduling delay. By wiring two handlers together with paired
senders, we exercise both ends of the TCP protocol against itself —
no kernel involvement, no TAP device, no ARP, no fragmentation. Pure
state-machine traffic.

---

## Driving it on a real TAP device

The full client/server demo with `nc` on the kernel side. The Go
program is the longest of the series because the wiring traverses
every layer:

```go
package main

import (
    "context"
    "fmt"
    "io"
    "log"

    "github.com/rykth/tcp-ip-stack/pkg/arp"
    "github.com/rykth/tcp-ip-stack/pkg/ethernet"
    "github.com/rykth/tcp-ip-stack/pkg/ip"
    netpkg "github.com/rykth/tcp-ip-stack/pkg/net"
    "github.com/rykth/tcp-ip-stack/pkg/raw/tuntap"
    "github.com/rykth/tcp-ip-stack/pkg/tcp"
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
    tcpH := tcp.NewHandler(ipH, localIP)
    defer tcpH.Shutdown()
    ipH.RegisterUpper(tcpH) //nolint:errcheck

    r := netpkg.New()
    r.AddDevice(dev); r.AddHandler(arpH); r.AddHandler(ipH)

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    go r.Start(ctx) //nolint:errcheck

    // Echo server on TCP port 9000.
    lst, err := tcpH.Listen(9000)
    if err != nil { log.Fatal(err) }
    defer lst.Close()
    fmt.Println("echo server listening on 10.0.0.2:9000")

    for {
        c, err := lst.Accept(ctx)
        if err != nil { return }
        fmt.Println("accepted connection")
        go func(c *tcp.Conn) {
            defer c.Close()
            buf := make([]byte, 4096)
            for {
                n, err := c.Read(ctx, buf)
                if err != nil || n == 0 {
                    if err != nil && err != io.EOF {
                        fmt.Printf("read: %v\n", err)
                    }
                    return
                }
                fmt.Printf("got %d bytes: %q\n", n, buf[:n])
                if err := c.Write(ctx, buf[:n]); err != nil {
                    fmt.Printf("write: %v\n", err)
                    return
                }
            }
        }(c)
    }
}
```

Three terminals as usual:

```bash
# Terminal 1 — TAP + stack
sudo ip tuntap add dev tap0 mode tap user $USER
sudo ip link set tap0 up
sudo ip addr add 10.0.0.1/24 dev tap0
go run ./examples/tcp

# Terminal 2 — wire capture
sudo tcpdump -i tap0 -n -e -S 'tcp or arp'

# Terminal 3 — TCP client
nc 10.0.0.2 9000
```

The session looks like:

```
[terminal 3]
$ nc 10.0.0.2 9000
hello                 ← you type
hello                 ← echoed back
ping                  ← you type
ping
^D                    ← you EOF
$
```

`tcpdump` shows the full handshake, the data round-trip, and the
four-way close:

```
ARP, Request who-has 10.0.0.2 tell 10.0.0.1
ARP, Reply 10.0.0.2 is-at aa:bb:cc:dd:ee:02

# Three-way handshake
IP 10.0.0.1.42178 > 10.0.0.2.9000: Flags [S],  seq 3735928559, win 65535
IP 10.0.0.2.9000  > 10.0.0.1.42178: Flags [S.], seq 2882400000, ack 3735928560, win 65535
IP 10.0.0.1.42178 > 10.0.0.2.9000: Flags [.],   seq 3735928560, ack 2882400001, win 65535

# Data: "hello\n"
IP 10.0.0.1.42178 > 10.0.0.2.9000: Flags [P.], seq 3735928560:3735928566, ack 2882400001, win 65535, length 6
IP 10.0.0.2.9000  > 10.0.0.1.42178: Flags [.],  seq 2882400001, ack 3735928566, win 65535
IP 10.0.0.2.9000  > 10.0.0.1.42178: Flags [P.], seq 2882400001:2882400007, ack 3735928566, win 65535, length 6
IP 10.0.0.1.42178 > 10.0.0.2.9000: Flags [.],   seq 3735928566, ack 2882400007, win 65535

# Close
IP 10.0.0.1.42178 > 10.0.0.2.9000: Flags [F.], seq 3735928566, ack 2882400007, win 65535
IP 10.0.0.2.9000  > 10.0.0.1.42178: Flags [.],   seq 2882400007, ack 3735928567, win 65535
IP 10.0.0.2.9000  > 10.0.0.1.42178: Flags [F.], seq 2882400007, ack 3735928567, win 65535
IP 10.0.0.1.42178 > 10.0.0.2.9000: Flags [.],   seq 3735928567, ack 2882400008, win 65535
```

And the stack prints:

```
echo server listening on 10.0.0.2:9000
accepted connection
got 6 bytes: "hello\n"
got 5 bytes: "ping\n"
```

Every layer is participating: the kernel's `nc` is talking to our
stack's TCP. The handshake exercised the SYN-RECEIVED state and the
final ACK. The data exchange exercised `handleEstablished` for both
sides. The close exercised `FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT` on
the client side and `CLOSE_WAIT → LAST_ACK` on the server side.

`ss -tnp` on the kernel side during the session shows the connection
in `ESTABLISHED`; immediately after `nc` exits, the kernel's side
enters `TIME_WAIT` for 60 seconds. Our stack's `TIME_WAIT` lasts
2×MSL = 60 seconds as well (`msl = 30 * time.Second` in `conn.go`).

You can also test with `curl` if you put a small HTTP responder behind
the same listener:

```go
// Replace the echo loop with a fixed HTTP response.
go func(c *tcp.Conn) {
    defer c.Close()
    buf := make([]byte, 4096)
    c.Read(ctx, buf) //nolint:errcheck  // discard the request
    body := "<html><body>hello from a userspace TCP stack</body></html>"
    resp := fmt.Sprintf("HTTP/1.0 200 OK\r\nContent-Length: %d\r\n\r\n%s",
                        len(body), body)
    c.Write(ctx, []byte(resp)) //nolint:errcheck
}(c)
```

```bash
$ curl http://10.0.0.2:9000/
<html><body>hello from a userspace TCP stack</body></html>
```

That HTML round-trips through every layer of code in this series —
Ethernet to TAP to ARP to IP to TCP — and back the other way. It is
the closing acceptance test of the whole project so far.

---

## Design decisions, revisited

### Why one goroutine per connection?

The alternative would be a single event loop in the handler that
muxes events across all connections by 4-tuple. That is closer to how
the Linux kernel works internally (the BH context handles many
sockets), and it would have lower per-connection memory overhead.

The single-goroutine-per-connection design has two big advantages
that outweigh the memory cost:

1. **State transitions are sequential.** Each connection's state
   machine runs in exactly one goroutine, so there is no need for
   per-state locks. The `state`, `iss`, `irs` fields can be plain
   `uint`s without atomics.
2. **`Read`/`Write` block naturally.** The user-facing API is built
   around blocking calls that wait on a buffer or a context. With one
   goroutine per connection, blocking is the goroutine's natural state
   — no need for an `epoll`-style readiness machinery to surface "you
   can read N bytes now" to the user.

At a million concurrent connections, this design would be a problem.
At the scales we are exercising, it is the simplest correct
implementation.

### Why does `Conn.run` ignore the `dataKick` payload?

The channel is `chan struct{}` rather than `chan []byte` because the
data lives in the `SendBuffer`, not in the channel. `dataKick`'s only
job is to *wake* the run goroutine — once awake, the loop's
opportunistic-send check at the top picks up whatever is in the
buffer.

The channel has capacity 1 and the send is non-blocking with `default:`:

```go
select {
case c.dataKick <- struct{}{}:
default:
}
```

This means multiple `Write`s in quick succession produce *at most one*
wake-up event. The run goroutine sees the event and drains the buffer,
which is fine — we don't need one wake per Write, only one wake to
trigger a drain.

### Why is `connectDone` a channel and not a `sync.WaitGroup`?

Both work, but `chan struct{}` integrates with `select`. The caller
of `Dial`/`Accept` needs to wait on *either* the handshake completing
*or* the context cancelling, which is a textbook `select` use case:

```go
func (c *Conn) waitConnect(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-c.connectDone:
        return c.connectErr
    }
}
```

A `WaitGroup` would force the caller to drive a separate goroutine
just to wait, defeating the purpose.

### Why is the SendBuffer a ring and the RecvBuffer also a ring?

Both buffers are bounded — 64 KiB each by default. A linear buffer
would grow unbounded if the consumer fell behind; the ring's fixed
capacity is the natural place to apply back-pressure.

For the SendBuffer, "full" means `Write` blocks until ACKs free up
space. For the RecvBuffer, "full" means the advertised window in our
outgoing ACKs shrinks, which throttles the peer's transmission rate.
Both behaviours are essential to TCP flow control, and both are
easier to implement on top of a ring than a linear buffer.

### Why `crypto/rand` for the ISS?

```go
func newISS() uint32 {
    var b [4]byte
    if _, err := rand.Read(b[:]); err != nil {
        return 0
    }
    return binary.BigEndian.Uint32(b[:])
}
```

RFC 793 originally specified a clock-derived ISN (a counter
incrementing 250,000 times per second). RFC 6528 updated this to a
secret-keyed cryptographic derivation, to prevent off-path attackers
from guessing the next ISN of a connection they cannot observe.

We do not implement the keyed derivation — we just use 32 bytes of
random data per ISN. This is sufficient for an attacker outside our
LAN to be unable to guess; an inside attacker who can sniff our
traffic does not benefit from the keyed derivation anyway.

---

## Limitations

| Limitation | Detail |
|------------|--------|
| **No congestion control** | The sender is bound only by the peer's advertised window. No slow-start, no CUBIC, no BBR. This is fine on a LAN; it would melt under congestion on a long-fat-pipe |
| **No SACK** | Selective Acknowledgments would let us retransmit only the missing fragments of a window. We send everything from `una` on timeout |
| **No window scaling** | Maximum effective window is 65,535 bytes. Limits throughput on links with high bandwidth-delay product |
| **No timestamps** | We parse the timestamp option but never emit it. This limits RTT estimation precision and rules out PAWS (Protection Against Wrapped Sequences) |
| **No keepalive** | An idle connection stays open indefinitely. Production stacks send keepalive probes after a configurable idle period |
| **No urgent data** | The URG flag and Urgent Pointer field are ignored. Nobody actually uses urgent data |
| **No PMTU discovery** | We assume the path MTU equals the device MTU. A bottleneck in the middle that drops large packets would manifest as connection stalls, not MSS adjustment |
| **No half-open detection** | If a peer crashes without sending RST, our connection stays in `ESTABLISHED` until the next data attempt and subsequent retransmission timeout (~30 seconds) |
| **Fixed buffer sizes** | 64 KiB for send and receive, hard-coded. A `WithBufferSize` option would be a one-line extension |
| **Single retransmit timer per connection** | We arm one timer for the oldest unacknowledged segment. RFC 6298 §5 allows per-segment timers; we choose the simpler form |
| **Best-effort SYN flood protection** | The listener backlog drops on overflow but does not implement SYN cookies. A SYN flood would exhaust the backlog channel and stop accepting new connections |
| **No graceful peer-closed signal** | `Read` returns `ErrConnClosed` indistinguishably from local close vs. peer FIN. A real socket returns `io.EOF` only for peer-initiated close |

The biggest functional gap is **congestion control**. Implementing
even basic slow-start + congestion-avoidance would be a few hundred
lines and would let our stack survive real-world traffic patterns.
It is the natural next step beyond the scope of this post.

---

## What is next

TCP is the substantial milestone the series has been pointing at. With
it in place, the stack is "feature complete" in the sense that any
mainstream client — `curl`, `ssh`, a web browser, any application
binary that speaks BSD sockets — can talk to it without modification.
The remaining work is in two directions:

1. **Production hardening** — congestion control, SACK, window
   scaling, timestamps, PMTUD, keepalives. Most of these are well-
   documented RFC extensions that bolt onto the state machine
   without restructuring it.
2. **Application protocols on top** — DNS over UDP, HTTP/1.1 over
   TCP, possibly TLS via an external library. With UDP and TCP both
   working, we have the substrate for any text-based or
   length-prefixed application protocol.

The next post in this series will pick one of these and dig in. The
satisfaction of running `curl http://10.0.0.2/` and watching a
userspace stack we wrote serve real HTML is hard to match, and that
is the demonstration the rest of the series will build toward.

The full source is at
[rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack).
