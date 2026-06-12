---
title: "Building the IPv4 layer in Go"
date: 2026-05-22T10:00:00+00:00
description: "The fifth post in my TCP/IP stack series. ARP gave us IP→MAC resolution. Now we implement the protocol that finally crosses the link-layer boundary — IPv4 — with header parsing, the RFC 1071 checksum, longest-prefix-match routing, fragment reassembly, and the Send path that ties them all together."
tags: [golang, networking, tcpip, ipv4, rfc791]
---

## Where this fits

So far the stack handles raw frames
([loopback](/posts/building-a-loopback-device),
[TUN/TAP](/posts/building-a-tuntap-device)), parses Ethernet headers
([codec + Registry](/posts/building-ethernet-layer)), and resolves IP
addresses to MAC addresses ([ARP](/posts/building-arp-layer)). Every piece
so far has been *link-local*: it works inside one Ethernet segment and
breaks at the first router.

IPv4 is the protocol that crosses routers. It is the first layer in our
stack that does not care what is underneath it — Ethernet, Wi-Fi, a VPN
tunnel, doesn't matter. IPv4 says "here is a datagram with a source and a
destination address; somebody figure out how to get it there." That
indifference to the link layer is what made it the lingua franca of the
internet for forty-five years.

The code lives at
[rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack), in `pkg/ip`.

```
 ┌─────────────────────────────────────────┐
 │   Application (HTTP…)                   │
 ├─────────────────────────────────────────┤
 │   Transport (TCP/UDP)                   │
 ├─────────────────────────────────────────┤
 │   Network (IPv4) ◄──────────────────────│  ← this post
 ├─────────────────────────────────────────┤
 │   Link (Ethernet) + ARP                 │
 ├─────────────────────────────────────────┤
 │   Device (loopback / TAP)               │
 └─────────────────────────────────────────┘
```

The package is bigger than anything we have built so far. There are five
files: the header codec (`ip.go`), the ones'-complement checksum
(`checksum.go`), the longest-prefix-match routing table (`routing.go`),
the fragment reassembler (`fragment.go`), and the handler that ties them
together with both a receive path and a `Send` API (`handler.go`). Each
piece has its own concerns and its own tests; we will walk through them in
the order the receive and transmit paths exercise them.

---

## What is IPv4?

IPv4 is a datagram protocol. A *datagram* is a self-contained packet with
its own source, destination, and payload — no connection, no session, no
acknowledgement. Each datagram is routed independently. The protocol
provides **unreliable, connectionless, best-effort delivery**: datagrams
can be dropped, duplicated, reordered, or fragmented. If reliability
matters, the layer above (TCP) handles it. If not, the layer above (UDP)
embraces the unreliability.

Three things make IPv4 work across heterogeneous networks:

1. **A globally addressable namespace.** Every host has a 4-byte address.
   No two hosts on the public internet share an address (the local-network
   exceptions notwithstanding).
2. **Routing.** Each router along the path picks the next hop based on the
   destination address and its own routing table. No router needs to know
   the full path — only one hop at a time.
3. **Fragmentation.** If a datagram is larger than the next link's MTU,
   the router breaks it into fragments. The receiver reassembles them.

### The 20-byte header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |Type of Service|          Total Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (0–40 bytes, optional)             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Field by field:

- **Version (4 bits)** — `4` for IPv4. The very first nibble of every
  packet. Reading this lets a stack decide whether to dispatch to IPv4 or
  IPv6 parsing without any other context.
- **IHL (4 bits)** — Internet Header Length, in 32-bit words. Minimum
  `5` (20 bytes); maximum `15` (60 bytes). Anything above 5 means options
  are present.
- **TOS (1 byte)** — Type of Service. Originally meant to express
  precedence; today its top six bits hold the DSCP value used for QoS
  marking and the bottom two are ECN bits used by the transport layer.
- **Total Length (2 bytes)** — header + payload in bytes. This is what
  lets a parser separate the IP datagram from the Ethernet padding
  underneath.
- **Identification (2 bytes)** — used to group fragments of the same
  datagram. Set by the sender; constant across all fragments of a single
  datagram.
- **Flags (3 bits) + Fragment Offset (13 bits)** — packed into one
  16-bit word. The three flag bits are reserved, DF (Don't Fragment), and
  MF (More Fragments). The offset is in 8-byte units within the
  reassembled payload.
- **TTL (1 byte)** — Time To Live. Decremented by every router. When it
  reaches zero, the router drops the packet and sends back an ICMP "time
  exceeded." This is what makes `traceroute` work and what prevents
  routing loops from melting the internet.
- **Protocol (1 byte)** — the next-layer protocol number. `1` for ICMP,
  `6` for TCP, `17` for UDP. We define these three constants and
  dispatch on them.
- **Header Checksum (2 bytes)** — RFC 1071 ones'-complement checksum
  *of the header only*. Re-computed at every router because the TTL
  changes on each hop.
- **Source / Destination Addresses (4 bytes each)** — the only addresses
  IP recognises.
- **Options (0–40 bytes)** — historically used for source routing,
  timestamps, record route. In practice every modern firewall drops
  packets with options set. We parse them but produce datagrams without
  them.

That is the entire header. Twenty bytes of essential information; up to
forty bytes of options nobody uses.

---

## Exploring IPv4 from the command line

Every IP concern this package implements — routing, fragmentation, TTL,
checksums, protocol dispatch — corresponds to a Linux tool that lets you
exercise it directly. These are the tools to keep in your shell while
working on the implementation.

### Inspect IP addresses

```bash
# Show all IPv4 addresses on every interface
ip -4 addr show

# A single interface
ip addr show dev tap0

# Just the addresses (useful in scripts)
ip -4 -o addr show | awk '{print $4}'
```

```
2: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    inet 192.168.100.1/24 scope global tap0
       valid_lft forever preferred_lft forever
```

The `/24` is the CIDR notation for the subnet — bits 0-23 are the network
identifier, bits 24-31 are the host identifier within that network. CIDR
maps directly onto the `Network` struct we are about to build.

### The routing table

The kernel's IPv4 routing table is the canonical example of longest-prefix-
match dispatch. Reading it teaches you exactly what the `Table` we
implement needs to do:

```bash
# The main routing table — what userspace usually sees
ip route show

# Every table the kernel knows about (local, main, default, custom)
ip route show table all

# Just one table
ip route show table local
ip route show table main
```

```
default via 192.168.1.1 dev wlan0 proto dhcp metric 600
10.0.0.0/8 dev tap0 proto kernel scope link src 10.0.0.1
192.168.1.0/24 dev wlan0 proto kernel scope link src 192.168.1.100 metric 600
```

Three properties of this output match our implementation choices one-for-
one:

- The `metric` column — lower is better — is the tie-breaker when two
  routes have the same prefix length.
- `proto kernel scope link` means the route was inserted automatically
  when an IP address was added to the interface; the destination is
  directly reachable on the same link (no gateway).
- `default` is shorthand for `0.0.0.0/0` — the fallback route that
  matches everything.

The single most useful debugging command for routing is `ip route get`,
which asks the kernel which route *would be* used for a given destination
without actually sending anything:

```bash
# Which route would the kernel use to reach 8.8.8.8?
ip route get 8.8.8.8

# Force the lookup against a specific source IP
ip route get 8.8.8.8 from 10.0.0.1

# Through a specific interface
ip route get 8.8.8.8 dev tap0
```

```
8.8.8.8 via 192.168.1.1 dev wlan0 src 192.168.1.100 uid 1000
    cache
```

This single line tells you: the destination matched the default route, the
next hop is `192.168.1.1`, the outbound interface is `wlan0`, and the
source address will be `192.168.1.100`. That is precisely the information
`Table.Lookup` returns in our implementation, just wrapped in a different
output format.

### Add and remove routes

```bash
# Static route to a specific network via a gateway
sudo ip route add 10.0.0.0/24 via 192.168.100.1 dev tap0

# Direct (on-link) route — no gateway needed
sudo ip route add 10.0.0.0/24 dev tap0

# Default route
sudo ip route add default via 192.168.1.1

# Replace an existing route in place
sudo ip route replace 10.0.0.0/24 via 192.168.100.2 dev tap0

# Delete by network spec
sudo ip route del 10.0.0.0/24

# Flush all routes for a device (use with care)
sudo ip route flush dev tap0
```

The combination `add` + `del` is what lets you write integration tests
that don't pollute the host's permanent routing table.

### Watch packets on the wire

`tcpdump` with the `ip` filter and `-n` (no DNS) is what reveals the
header fields we care about:

```bash
# All IP traffic on tap0
sudo tcpdump -i tap0 -n ip

# With Ethernet headers too
sudo tcpdump -i tap0 -n -e ip

# Verbose — TTL, ID, flags, fragmentation, checksum status
sudo tcpdump -i tap0 -n -vv ip

# Filter by source / destination
sudo tcpdump -i tap0 -n src 10.0.0.1
sudo tcpdump -i tap0 -n dst 10.0.0.2

# Filter by protocol
sudo tcpdump -i tap0 -n icmp
sudo tcpdump -i tap0 -n tcp port 80
sudo tcpdump -i tap0 -n udp

# Filter by fragment status — useful when testing our fragment path
sudo tcpdump -i tap0 -n 'ip[6] & 0x20 != 0'   # MF flag set
sudo tcpdump -i tap0 -n 'ip[6:2] & 0x1fff != 0'  # non-zero fragment offset
```

`-vv` is the one to keep nearby while debugging the codec. A typical line
looks like this:

```
20:14:32.182814 IP (tos 0x0, ttl 64, id 1, offset 0, flags [DF],
   proto ICMP (1), length 84)
   10.0.0.1 > 10.0.0.2: ICMP echo request, id 5523, seq 1, length 64
```

Every field from the 20-byte header is right there: `ttl`, `id`,
`offset`, `flags`, `proto`, `length`. If `Marshal` ever produced wrong
bytes, this is the line that would show it.

### Test the TTL field with traceroute

`traceroute` works by sending packets with increasing TTL values: `TTL=1`,
then `TTL=2`, and so on. Each router along the path drops the packet when
TTL reaches zero and sends back an ICMP "time exceeded" — revealing one
hop per probe.

```bash
# ICMP-based (some servers block this)
sudo traceroute -I 8.8.8.8

# UDP probes (default on Linux)
traceroute 8.8.8.8

# TCP probes — gets through more firewalls
sudo traceroute -T -p 443 8.8.8.8

# More detail per hop
mtr 8.8.8.8
```

Watching `traceroute` next to `tcpdump -nn 'ip[8] < 5'` (TTL < 5) lets you
observe every probe leaving your machine. This is the cleanest way to
verify that our `Send` correctly sets `TTL=64` on every datagram — set
`64` and `traceroute` cannot reach more than 64 hops away.

### Test fragmentation

The Don't Fragment bit and the `ping` size flag together let you reproduce
every fragmentation case our implementation handles:

```bash
# Default ICMP echo — small enough to never fragment
ping -c 1 192.168.100.2

# Force fragmentation: 4000-byte payload over a 1500-byte MTU
ping -c 1 -s 4000 192.168.100.2

# Don't Fragment set — packet is rejected by the kernel if it exceeds MTU
ping -c 1 -M do -s 4000 192.168.100.2
# → "Frag needed and DF set (mtu = 1500)"

# Path MTU Discovery — probes for the largest non-fragmenting size
ping -c 4 -M do -s 1472 8.8.8.8

# Show the resulting fragments on the wire
sudo tcpdump -i tap0 -n -vv 'ip and (ip[6] & 0x20 != 0 or ip[6:2] & 0x1fff != 0)'
```

`ping -s 4000` is the easiest way to drive our reassembler from outside —
the kernel splits the 4028-byte datagram (4000 + 8-byte ICMP header + 20
ip header would be wrong — actually 4000 + 8 + 20 = 4028) into three
fragments at the 1500-byte MTU boundary, and our handler reassembles them.

### Path MTU and interface MTU

```bash
# Show the MTU of every interface
ip -4 link show | grep -E 'mtu|^[0-9]+:'

# Set jumbo frames on a TAP (both ends must agree)
sudo ip link set tap0 mtu 9000

# Query the kernel's Path MTU cache
ip route show cache

# Manually probe the path MTU to a destination
tracepath 8.8.8.8
```

Our routing table stores routes by destination network, not by Path MTU —
fragmentation decisions use the outbound device's MTU directly via
`route.Dev.MTU()`. If the path between two hosts has a smaller MTU
somewhere in the middle, our packets will be dropped by ICMP
"fragmentation needed" rather than fragmented further. This matches how
modern Linux behaves with `IP_PMTUDISC_DO` set by default.

### IP-level statistics

```bash
# Aggregate counters: received, forwarded, dropped, fragmented
nstat -a | grep -i ip

# The classic netstat
netstat -s -p ip

# Per-interface byte/packet counters
ip -s link show tap0
```

```
Ip:
    Forwarding: 2
    DefaultTTL: 64
    InReceives: 18432
    InHdrErrors: 0
    InAddrErrors: 0
    ForwDatagrams: 0
    InUnknownProtos: 0
    InDiscards: 0
    InDelivers: 18432
    OutRequests: 17094
    ...
    ReasmTimeout: 0
    ReasmReqds: 18
    ReasmOKs: 18
    ReasmFails: 0
    FragOKs: 0
    FragFails: 0
```

`InHdrErrors` is the counter that increments on bad checksum, wrong
version, or truncated header — i.e. everything our `Parse` returns an
error for. `ReasmTimeout` increments when the reassembler GC drops an
incomplete group. If our integration tests behave correctly, these
counters should move in lockstep with the equivalent paths in our code.

### Kernel tunables

```bash
# Default TTL for outbound packets
cat /proc/sys/net/ipv4/ip_default_ttl

# Reassembly timeout (seconds)
cat /proc/sys/net/ipv4/ipfrag_time

# Maximum bytes of memory the reassembler may consume
cat /proc/sys/net/ipv4/ipfrag_high_thresh

# Forwarding — must be 1 for the host to route between interfaces
cat /proc/sys/net/ipv4/ip_forward

# Strict source/dest checks
cat /proc/sys/net/ipv4/conf/all/rp_filter
```

Our defaults match the Linux defaults exactly: `TTL=64`, reassembly
timeout 30 seconds. Knowing these numbers and where they live in
`/proc` is what lets you reproduce a kernel behaviour in user-space code
without guessing.

### Generate or corrupt IP packets

`scapy` is the right tool for crafting test packets — it speaks the
fields by name, computes checksums automatically, and ships frames out a
TAP device with no privileges beyond `CAP_NET_RAW`:

```bash
sudo python3 - <<'PY'
from scapy.all import *
# Plain UDP datagram
sendp(Ether(dst='aa:bb:cc:dd:ee:01') /
      IP(src='10.0.0.2', dst='10.0.0.1', proto=17) /
      UDP(sport=4242, dport=8080) /
      b'hello', iface='tap0')

# Two fragments of one IPv4 datagram (id=42)
sendp(Ether(dst='aa:bb:cc:dd:ee:01') /
      IP(src='10.0.0.2', dst='10.0.0.1', id=42, flags='MF',
         frag=0, proto=6) /
      b'\x00' * 8, iface='tap0')

sendp(Ether(dst='aa:bb:cc:dd:ee:01') /
      IP(src='10.0.0.2', dst='10.0.0.1', id=42, flags=0,
         frag=1, proto=6) /
      b'\xff' * 4, iface='tap0')

# Deliberately broken checksum — chksum=0 forces a wrong value
sendp(Ether(dst='aa:bb:cc:dd:ee:01') /
      IP(src='10.0.0.2', dst='10.0.0.1', chksum=0x1234) /
      b'broken', iface='tap0')
PY
```

The bad-checksum case is what `TestParse_BadChecksum` covers in code;
running it on a live TAP confirms our handler silently drops the packet
the way it should.

---

## The implementation

Five files. We will walk through them in the order the receive path runs:
first the codec (`ip.go`), then the checksum it depends on
(`checksum.go`), then the routing table that the transmit path needs
(`routing.go`), then the reassembler that handles fragmented inputs
(`fragment.go`), and finally the handler that wires everything to the
`net.Registry` (`handler.go`).

### `Header` — the parsed view of a 20-byte structure

```go
type Header struct {
    Version    uint8
    IHL        uint8    // in 32-bit words; min 5 (20 bytes)
    TOS        uint8
    TotalLen   uint16
    ID         uint16
    Flags      uint8    // 3-bit field: FlagDF | FlagMF
    FragOffset uint16   // in 8-byte units
    TTL        uint8
    Protocol   Protocol
    Checksum   uint16
    Src        [4]byte
    Dst        [4]byte
    Options    []byte   // (IHL-5)*4 bytes; nil when IHL == 5
}
```

Twelve fields, none allocated except `Options`. Two value-type
optimisations to call out:

- `Src` and `Dst` are `[4]byte` arrays — comparable, usable as map
  keys, identical to the choice we made for the
  [ARP cache](/posts/building-arp-layer).
- `Flags` is an exposed `uint8` and accessed via bit constants
  (`FlagDF`, `FlagMF`), not a `struct { DF bool; MF bool }`. The bit-
  level representation matches the wire format so reading and writing the
  flags is a single mask operation.

`Options` is a slice because options are rare. Almost every datagram has
`IHL == 5` and the slice is `nil`, so the only allocation cost is in the
edge case.

### `Parse` — validate, decode, allocate-only-if-options

```go
func Parse(b []byte) (Header, error) {
    if len(b) < MinHeaderLen {
        return Header{}, ErrHeaderTooShort
    }
    version := b[0] >> 4
    if version != 4 {
        return Header{}, ErrBadVersion
    }
    ihl := b[0] & 0x0F
    headerLen := int(ihl) * 4
    if headerLen < MinHeaderLen || headerLen > len(b) {
        return Header{}, ErrHeaderTooShort
    }
    if Checksum(b[:headerLen]) != 0 {
        return Header{}, ErrBadChecksum
    }

    fo := binary.BigEndian.Uint16(b[6:8])
    var h Header
    h.Version = version
    h.IHL = ihl
    h.TOS = b[1]
    h.TotalLen = binary.BigEndian.Uint16(b[2:4])
    h.ID = binary.BigEndian.Uint16(b[4:6])
    h.Flags = uint8(fo >> 13)
    h.FragOffset = fo & 0x1FFF
    h.TTL = b[8]
    h.Protocol = Protocol(b[9])
    h.Checksum = binary.BigEndian.Uint16(b[10:12])
    copy(h.Src[:], b[12:16])
    copy(h.Dst[:], b[16:20])
    if headerLen > MinHeaderLen {
        h.Options = make([]byte, headerLen-MinHeaderLen)
        copy(h.Options, b[20:headerLen])
    }
    return h, nil
}
```

Four validations, in this order, all of which discriminate distinct
failure modes:

1. **Buffer at least 20 bytes.** Without this, indexing `b[0]` panics.
2. **Version field equals 4.** A version-6 packet on an IPv4 channel is
   not our problem and reading further fields would interpret them
   incorrectly.
3. **IHL produces a valid header length** (≥ 20, ≤ `len(b)`). A
   malicious or corrupted IHL could otherwise read past the buffer.
4. **Checksum is valid.** `Checksum(b[:headerLen])` over a *correct*
   header (including its own checksum field) sums to `0xFFFF`, whose
   ones'-complement is `0`. If the return value is non-zero, the
   header is corrupted.

The packed Flags/FragOffset word at offset 6–7 is the only awkward part
of the layout: three bits of flags in the top, thirteen bits of offset in
the bottom. The shift-and-mask pair (`fo >> 13`, `fo & 0x1FFF`)
extracts both.

Options are copied into a fresh slice, not aliased to the input buffer.
This is the only allocation on the parse path, and only when options are
present. The copy is necessary because the caller is allowed to reuse the
underlying buffer; without copying, the parsed `Options` would silently
mutate.

### `Marshal` — the inverse, including checksum

```go
func Marshal(dst []byte, h Header, payload []byte) error {
    headerLen := int(h.IHL) * 4
    need := headerLen + len(payload)
    if len(dst) < need {
        return fmt.Errorf("ip: marshal buffer too small: need %d have %d", need, len(dst))
    }
    dst[0] = (h.Version << 4) | h.IHL
    dst[1] = h.TOS
    binary.BigEndian.PutUint16(dst[2:4], h.TotalLen)
    binary.BigEndian.PutUint16(dst[4:6], h.ID)
    fo := (uint16(h.Flags) << 13) | h.FragOffset
    binary.BigEndian.PutUint16(dst[6:8], fo)
    dst[8] = h.TTL
    dst[9] = uint8(h.Protocol)
    dst[10], dst[11] = 0, 0 // zero before checksum computation
    copy(dst[12:16], h.Src[:])
    copy(dst[16:20], h.Dst[:])
    if len(h.Options) > 0 {
        copy(dst[20:headerLen], h.Options)
    }
    check := Checksum(dst[:headerLen])
    binary.BigEndian.PutUint16(dst[10:12], check)
    copy(dst[headerLen:], payload)
    return nil
}
```

The single subtle line is the explicit zero of the checksum field before
running `Checksum`. The RFC 1071 algorithm computes a sum over the entire
header *including the checksum field itself* — but at compute time, that
field must be zero. After the function returns the computed value, we
write it back into the same two bytes.

This is the standard ones'-complement checksum trick: because adding zero
contributes nothing to the sum, you can write the field afterward and
verification (which sums the entire header including the new checksum)
will produce zero.

### `Payload` — clip to TotalLen

```go
func Payload(b []byte, h Header) []byte {
    headerLen := int(h.IHL) * 4
    if len(b) <= headerLen {
        return nil
    }
    end := int(h.TotalLen)
    if end > len(b) {
        end = len(b)
    }
    if end <= headerLen {
        return nil
    }
    return b[headerLen:end]
}
```

`Payload` is small but essential. Ethernet pads short frames to a 60-byte
minimum at the hardware boundary; on receive, the TAP delivers those
padding bytes alongside the IP datagram. Without clipping to `TotalLen`,
the payload returned to the upper layer would include trailing zero bytes
that aren't part of the datagram. This is exactly the kind of bug that
silently corrupts a TCP segment by extending its length by 14 bytes.

### `Checksum` — RFC 1071 in twenty lines

```go
func Checksum(b []byte) uint16 {
    var sum uint32
    for i := 0; i+1 < len(b); i += 2 {
        sum += uint32(binary.BigEndian.Uint16(b[i : i+2]))
    }
    if len(b)%2 != 0 {
        sum += uint32(b[len(b)-1]) << 8
    }
    for sum>>16 != 0 {
        sum = (sum & 0xFFFF) + (sum >> 16)
    }
    return ^uint16(sum)
}
```

RFC 1071 in pure Go. The algorithm in three sentences:

1. Sum every 16-bit big-endian word into a 32-bit accumulator (which
   captures the carry bits the 16-bit format cannot hold).
2. If the input has an odd length, pad the last byte on the right with a
   zero — i.e. shift it left by 8 — and add it.
3. Fold the carry bits back into the low 16 bits repeatedly until no
   carry remains, then return the ones'-complement.

The fold loop almost always runs once or twice — the accumulator after
adding 1500 bytes of data cannot exceed `1500/2 * 0xFFFF ≈ 0x4AF82` —
but writing it as `for sum >> 16 != 0` is robust to any input size
without bound-checking.

We verify the implementation against the textbook RFC 1071 vector
(`00 01 f2 03 f4 f5 f6 f7 → 0x220d`) in `TestChecksum_RFC1071Vector` —
the same example that appears in §3 of the RFC.

### `Add16` and `PseudoHeaderChecksum` — for TCP/UDP

The header checksum covers only the IP header. The transport layer (TCP,
UDP) wants a checksum over a *pseudo-header* (a 12-byte fragment of the IP
header) plus the transport segment. Concatenating those two byte sequences
in memory would be slow, but ones'-complement arithmetic has a useful
property: the checksum of a concatenation equals the ones'-complement sum
of the partial checksums.

```go
func Add16(a, b uint16) uint16 {
    sum := uint32(a) + uint32(b)
    if sum > 0xFFFF {
        sum++ // ones'-complement carry
    }
    return uint16(sum)
}

func PseudoHeaderChecksum(src, dst [4]byte, proto Protocol, l4Len uint16) uint16 {
    var b [12]byte
    copy(b[0:4], src[:])
    copy(b[4:8], dst[:])
    b[8] = 0
    b[9] = uint8(proto)
    binary.BigEndian.PutUint16(b[10:12], l4Len)
    return Checksum(b[:])
}
```

`TestAdd16_CombinesChecksums` pins the property:
`Add16(Checksum(A), Checksum(B)) == Checksum(A || B)`. With this we can
compute the TCP checksum without ever materialising the pseudo-header +
segment in one buffer — a useful invariant when TCP arrives in the next
post.

### `Network` — CIDR as a value type

```go
type Network struct {
    IP     [4]byte // network address (host bits zeroed)
    Mask   [4]byte // subnet mask (e.g., 255.255.255.0)
    Prefix uint8   // prefix length in bits (0–32)
}

func (n Network) Contains(ip [4]byte) bool {
    for i := 0; i < 4; i++ {
        if ip[i]&n.Mask[i] != n.IP[i] {
            return false
        }
    }
    return true
}
```

A `Network` is the value-type counterpart to `net.IPNet`. The standard
library's type is `struct { IP IP; Mask IPMask }` with both fields as
`[]byte` slices — comparable only via `bytes.Equal`, not usable as a map
key, and prone to allocations on every construction. We get the same data
in 9 fixed bytes with `==` working directly.

`Contains` walks the four bytes and does an AND-and-compare. The compiler
on amd64 can unroll this into a `BSWAP + AND + CMP` pair on the underlying
uint32 representation — sub-nanosecond on hot paths.

### The longest-prefix-match `Table`

```go
type Route struct {
    Network Network
    Gateway [4]byte           // [4]byte{} means directly connected
    Dev     netpkg.LinkDevice
    Metric  int
}

type Table struct {
    mu     sync.RWMutex
    routes []Route // sorted by Prefix descending, then Metric ascending
}
```

The implementation is the simplest LPM strategy that works:

```go
func (t *Table) Add(r Route) {
    t.mu.Lock()
    defer t.mu.Unlock()
    t.routes = append(t.routes, r)
    sort.SliceStable(t.routes, func(i, j int) bool {
        pi, pj := t.routes[i].Network.Prefix, t.routes[j].Network.Prefix
        if pi != pj {
            return pi > pj // longest prefix first
        }
        return t.routes[i].Metric < t.routes[j].Metric
    })
}

func (t *Table) Lookup(dst [4]byte) (Route, error) {
    t.mu.RLock()
    defer t.mu.RUnlock()
    for _, r := range t.routes {
        if r.Network.Contains(dst) {
            return r, nil
        }
    }
    return Route{}, ErrNoRoute
}
```

Sorting on insert, linear scan on lookup. Production routers use a Patricia
trie (`O(log n)` per lookup at best), but our table has on the order of a
handful of entries — the linear scan finishes in nanoseconds. The
`sort.SliceStable` is important: if two routes have identical prefix and
identical metric, the one added first wins, which matches the kernel's
behaviour.

A note on the `Gateway` field: an empty `[4]byte{}` means "directly
connected — the destination is on the link, no gateway needed." The
sentinel-zero check `route.Gateway != ([4]byte{})` in `Send` distinguishes
these two cases.

### The `Reassembler`

Fragmented IP datagrams arrive as independent packets with the same
`(Src, Dst, ID, Protocol)` 4-tuple. The reassembler maintains a map keyed
by that tuple, accumulates fragments, and emits the reassembled payload
when all bytes are present.

```go
type fragKey struct {
    Src   [4]byte
    Dst   [4]byte
    ID    uint16
    Proto Protocol
}

type fragment struct {
    offset uint32
    data   []byte
}

type fragGroup struct {
    frags    []fragment
    deadline time.Time
    totalLen int
    known    bool // true once the last fragment (MF=0) arrives
}
```

The 4-tuple key is RFC 791-mandated: two datagrams in flight to the same
host must use different `ID` values or their fragments would mix.

```go
func (r *Reassembler) Add(h Header, payload []byte) ([]byte, bool) {
    key := fragKey{Src: h.Src, Dst: h.Dst, ID: h.ID, Proto: h.Protocol}
    offset := uint32(h.FragOffset) * 8 // 8-byte units → bytes
    hasMore := h.Flags&FlagMF != 0

    cp := make([]byte, len(payload))
    copy(cp, payload)

    r.mu.Lock()
    defer r.mu.Unlock()

    g, ok := r.groups[key]
    if !ok {
        g = &fragGroup{deadline: time.Now().Add(r.timeout)}
        r.groups[key] = g
    }
    g.frags = append(g.frags, fragment{offset: offset, data: cp})
    if !hasMore {
        g.totalLen = int(offset) + len(payload)
        g.known = true
    }

    if !g.known {
        return nil, false
    }

    result, complete := tryReassemble(g)
    if complete {
        delete(r.groups, key)
    }
    return result, complete
}
```

Three properties this implementation gets right:

- **Out-of-order arrival is fine.** Fragments can arrive in any order
  because we sort by offset inside `tryReassemble` before walking them.
  `TestReassembler_TwoFragmentsOutOfOrder` exercises this directly: the
  last fragment is injected first, then the first fragment, and
  reassembly still succeeds.
- **`totalLen` is only known once the last fragment (`MF=0`) arrives.**
  Until then, the reassembler does not even try to reassemble — there
  is no point checking for completeness if the upper bound is unknown.
- **The deadline is the *first* fragment's deadline.** RFC 791 §3.2
  specifies that the reassembly timer should not reset on every fragment
  — once started, it counts down regardless. Our GC enforces that
  rule.

```go
func tryReassemble(g *fragGroup) ([]byte, bool) {
    sort.Slice(g.frags, func(i, j int) bool {
        return g.frags[i].offset < g.frags[j].offset
    })

    var covered uint32
    for _, f := range g.frags {
        if f.offset > covered {
            return nil, false // gap
        }
        end := f.offset + uint32(len(f.data))
        if end > covered {
            covered = end
        }
    }
    if int(covered) < g.totalLen {
        return nil, false
    }

    out := make([]byte, g.totalLen)
    for _, f := range g.frags {
        end := f.offset + uint32(len(f.data))
        if int(end) > g.totalLen {
            end = uint32(g.totalLen)
        }
        copy(out[f.offset:end], f.data[:end-f.offset])
    }
    return out, true
}
```

Sort, walk, check for gaps, return the assembled byte slice when none
remain. The walk is the only interesting part — the `covered` variable
tracks the high-water mark of "bytes contiguously available from
offset 0," and a gap is detected the moment the next fragment's `offset`
exceeds `covered`.

The `end > g.totalLen` clamp handles a small adversarial case:
overlapping fragments where the last one extends past the declared
total length. Linux drops such packets entirely (the `RA-Guard`
mitigation for IPv6 originated here); we clip them, which is RFC-
compatible but less paranoid. For a test stack this is fine; a production
stack would reject.

### The receive event loop

```go
type Handler struct {
    localMAC    ethernet.Addr
    localIP     [4]byte
    arp         ARPResolver
    table       *Table
    reassembler *Reassembler
    rxCh        chan netpkg.Frame

    mu     sync.RWMutex
    uppers map[Protocol]UpperHandler

    nextID atomic.Uint32
}
```

The handler is the first piece in this package that talks to the rest of
the stack. It needs:

- `localMAC` and `localIP` to fill in outgoing headers.
- `arp` to resolve next-hop MAC addresses.
- `table` for routing decisions.
- `reassembler` for inbound fragmented datagrams.
- `rxCh` for the Registry to push received frames.
- `uppers` to dispatch by protocol number on receive.
- `nextID` to generate per-datagram identifiers.

The `arp ARPResolver` field is declared against an interface, not against
`*arp.Handler` directly:

```go
type ARPResolver interface {
    Resolve(ctx context.Context, ip [4]byte) (ethernet.Addr, error)
}
```

This is the inverse pattern to `net.LinkDevice` — the consuming package
defines the interface it needs, and the supplying package satisfies it
implicitly. `*arp.Handler` already has the right method shape, so the
test suite can substitute a `mockARP` without `pkg/ip` ever importing
`pkg/arp`.

```go
func (h *Handler) EtherType() ethernet.EtherType { return ethernet.EtherTypeIPv4 }
func (h *Handler) RxChan() chan netpkg.Frame     { return h.rxCh }

func (h *Handler) Start(ctx context.Context, errCh chan<- error) {
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        defer wg.Done()
        h.reassembler.Start(ctx)
    }()
    defer wg.Wait()

    for {
        select {
        case <-ctx.Done():
            return
        case frame, ok := <-h.rxCh:
            if !ok {
                return
            }
            h.process(frame)
        }
    }
}
```

Identical shape to the ARP handler's `Start`: launch the reassembler GC
goroutine, defer `wg.Wait()`, then loop on the receive channel until
shutdown. The `wg.Wait` is what makes
`TestHandler_Shutdown_NoLeak` pass cleanly.

### `process` — the receive path

```go
func (h *Handler) process(frame netpkg.Frame) {
    hdr, err := Parse(frame.Payload)
    if err != nil {
        return // drop malformed / non-IPv4 frames
    }

    payload := Payload(frame.Payload, hdr)

    // Handle fragmentation.
    if hdr.Flags&FlagMF != 0 || hdr.FragOffset > 0 {
        var complete bool
        payload, complete = h.reassembler.Add(hdr, payload)
        if !complete {
            return
        }
    } else {
        // Non-fragmented: copy so the caller can reuse the frame buffer.
        cp := make([]byte, len(payload))
        copy(cp, payload)
        payload = cp
    }

    h.mu.RLock()
    upper, ok := h.uppers[hdr.Protocol]
    h.mu.RUnlock()
    if !ok {
        return
    }
    upper.Deliver(hdr.Src, hdr.Dst, payload)
}
```

Four steps: parse, clip to total length, reassemble if fragmented,
dispatch by protocol. The fragmentation check uses the standard form:
**either MF is set (more fragments follow) or the offset is non-zero
(this is not the first fragment)**. A non-fragmented datagram has MF=0
and offset=0; everything else needs to go through the reassembler.

The non-fragmented branch still allocates a copy. This might look like
wasted work — the payload is already a slice into the frame buffer the
Registry handed us — but the Registry's contract says it may reuse that
buffer on the next iteration. Either we copy here, or the upper handler
has to copy in its `Deliver` implementation. Doing it here means every
upper handler gets a payload it can hold onto without ceremony.

Malformed frames are dropped silently, the same pattern as ARP.

### `Send` — route, ARP-resolve, fragment, frame

The transmit path is the most interesting code in the package. It pulls
together every other component:

```go
func (h *Handler) Send(ctx context.Context, dst [4]byte, proto Protocol, payload []byte) error {
    route, err := h.table.Lookup(dst)
    if err != nil {
        return fmt.Errorf("ip: send to %v: %w", dst, err)
    }

    // Resolve next-hop: gateway if set, else dst is on-link.
    nextHop := dst
    if route.Gateway != ([4]byte{}) {
        nextHop = route.Gateway
    }

    dstMAC, err := h.arp.Resolve(ctx, nextHop)
    if err != nil {
        return fmt.Errorf("ip: ARP resolve %v: %w", nextHop, err)
    }

    mtu := route.Dev.MTU()
    // maxPayload must be a multiple of 8 so non-last fragment offsets align.
    maxPayload := (mtu - MinHeaderLen) / 8 * 8

    id := uint16(h.nextID.Add(1))

    if len(payload) <= maxPayload || maxPayload <= 0 {
        return h.sendDatagram(route.Dev, dstMAC, id, dst, proto, 0, 0, payload)
    }

    // Fragmentation required.
    offset := 0
    for offset < len(payload) {
        chunkSize := maxPayload
        if remaining := len(payload) - offset; remaining < chunkSize {
            chunkSize = remaining
        }
        chunk := payload[offset : offset+chunkSize]

        var flags uint8
        if offset+chunkSize < len(payload) {
            flags = FlagMF
        }
        fragOffset := uint16(offset / 8)

        if err := h.sendDatagram(route.Dev, dstMAC, id, dst, proto, flags, fragOffset, chunk); err != nil {
            return err
        }
        offset += chunkSize
    }
    return nil
}
```

Six steps. The first three are the universal sequence for any IP send:
look up the route, decide whether to send via the gateway or directly to
the destination, and ask ARP to resolve the chosen next-hop IP to a MAC.

The `route.Gateway != ([4]byte{})` check is the standard sentinel — an
all-zero gateway means "destination is on-link," which is the case for
every directly-connected route in our table. Otherwise the gateway IP is
what we ARP for; the destination IP stays in the IP header but the
Ethernet header's destination MAC is the gateway's MAC. This is what
makes IP datagrams flow through routers.

`maxPayload := (mtu - MinHeaderLen) / 8 * 8` is the only arithmetic
trick in `Send`. Fragment offsets are in 8-byte units, so every fragment
except the last must contain a multiple-of-8 number of payload bytes. The
integer-divide-by-8-then-multiply-by-8 rounds the available payload down
to the nearest multiple of 8. For a 1500-byte MTU, this is
`(1500 - 20) / 8 * 8 = 1480` bytes — which is exactly what Linux uses.

`atomic.Uint32` on `nextID` is the second concurrency-relevant detail.
Two simultaneous calls to `Send` must produce two distinct datagram IDs,
or their fragments could be cross-pollinated by a receiver. The atomic
increment guarantees uniqueness without a lock.

The fragmentation loop is structurally the inverse of the reassembler:
walk the payload in `maxPayload` chunks, set MF=1 on every chunk except
the last, set the offset to `offset/8`, and call `sendDatagram` for
each. `TestHandler_Send_WithFragmentation` exercises this with a 24-byte
MTU and a 48-byte payload, expecting exactly two fragments.

### `sendDatagram` — one IP packet, one Ethernet frame

```go
func (h *Handler) sendDatagram(dev netpkg.LinkDevice, dstMAC ethernet.Addr, id uint16, dst [4]byte, proto Protocol, flags uint8, fragOffset uint16, payload []byte) error {
    hdr := Header{
        Version:    4,
        IHL:        MinHeaderLen / 4,
        TOS:        0,
        TotalLen:   uint16(MinHeaderLen + len(payload)),
        ID:         id,
        Flags:      flags,
        FragOffset: fragOffset,
        TTL:        64,
        Protocol:   proto,
        Src:        h.localIP,
        Dst:        dst,
    }

    frameLen := ethernet.HeaderLen + MinHeaderLen + len(payload)
    buf := make([]byte, frameLen)

    if err := Marshal(buf[ethernet.HeaderLen:], hdr, payload); err != nil {
        return fmt.Errorf("ip: marshal: %w", err)
    }

    ethHdr := ethernet.Header{
        Dst:       dstMAC,
        Src:       h.localMAC,
        EtherType: ethernet.EtherTypeIPv4,
    }
    if _, err := ethernet.Marshal(buf, ethHdr, buf[ethernet.HeaderLen:]); err != nil {
        return fmt.Errorf("ip: ethernet marshal: %w", err)
    }
    _, err := dev.Write(buf)
    return err
}
```

A single allocation per packet — the `buf` that holds the Ethernet
header + IP header + payload, contiguous. `ip.Marshal` writes the IP
header and payload at offset 14; `ethernet.Marshal` fills in the
14-byte Ethernet header in front of it.

Two defaults are hard-coded here that matter:

- `TTL: 64` — matches Linux's `/proc/sys/net/ipv4/ip_default_ttl`. A
  TTL of 64 reaches every part of the public internet (the longest known
  paths are ~30 hops) but eventually expires if a routing loop forms.
- `IHL: MinHeaderLen / 4` — we never produce datagrams with IP
  options, so IHL is always 5.

---

## The tests

Twenty-six tests cover the five files. Each file gets its own block; the
handler block is the largest because it exercises the receive and
transmit paths together.

```bash
# Plain run
go test ./pkg/ip/... -v

# With the race detector
CGO_ENABLED=1 go test ./pkg/ip/... -race -v
```

A clean run:

```
=== RUN   TestChecksum_RFC1071Vector
--- PASS: TestChecksum_RFC1071Vector (0.00s)
=== RUN   TestChecksum_Verification
--- PASS: TestChecksum_Verification (0.00s)
=== RUN   TestChecksum_OddLength
--- PASS: TestChecksum_OddLength (0.00s)
=== RUN   TestAdd16_CombinesChecksums
--- PASS: TestAdd16_CombinesChecksums (0.00s)
=== RUN   TestPseudoHeaderChecksum_NotZero
--- PASS: TestPseudoHeaderChecksum_NotZero (0.00s)
=== RUN   TestParse_ValidMinimal
--- PASS: TestParse_ValidMinimal (0.00s)
=== RUN   TestParse_TooShort
--- PASS: TestParse_TooShort (0.00s)
=== RUN   TestParse_BadVersion
--- PASS: TestParse_BadVersion (0.00s)
=== RUN   TestParse_BadChecksum
--- PASS: TestParse_BadChecksum (0.00s)
=== RUN   TestMarshal_RoundTrip
--- PASS: TestMarshal_RoundTrip (0.00s)
=== RUN   TestMarshal_FlagsMF
--- PASS: TestMarshal_FlagsMF (0.00s)
=== RUN   TestTable_LPM
--- PASS: TestTable_LPM (0.00s)
=== RUN   TestTable_NoRoute
--- PASS: TestTable_NoRoute (0.00s)
=== RUN   TestTable_Delete
--- PASS: TestTable_Delete (0.00s)
=== RUN   TestTable_MetricTieBreak
--- PASS: TestTable_MetricTieBreak (0.00s)
=== RUN   TestReassembler_NonFragment
--- PASS: TestReassembler_NonFragment (0.00s)
=== RUN   TestReassembler_TwoFragmentsInOrder
--- PASS: TestReassembler_TwoFragmentsInOrder (0.00s)
=== RUN   TestReassembler_TwoFragmentsOutOfOrder
--- PASS: TestReassembler_TwoFragmentsOutOfOrder (0.00s)
=== RUN   TestReassembler_GC_DropsExpiredGroups
--- PASS: TestReassembler_GC_DropsExpiredGroups (0.02s)
=== RUN   TestHandler_DeliverToUpper
--- PASS: TestHandler_DeliverToUpper (0.00s)
=== RUN   TestHandler_UnknownProtocol_Dropped
--- PASS: TestHandler_UnknownProtocol_Dropped (0.02s)
=== RUN   TestHandler_Send_NoFragmentation
--- PASS: TestHandler_Send_NoFragmentation (0.00s)
=== RUN   TestHandler_Send_WithFragmentation
--- PASS: TestHandler_Send_WithFragmentation (0.00s)
=== RUN   TestHandler_RegisterUpper_Duplicate
--- PASS: TestHandler_RegisterUpper_Duplicate (0.00s)
=== RUN   TestHandler_Shutdown_NoLeak
--- PASS: TestHandler_Shutdown_NoLeak (0.00s)
=== RUN   TestHandler_Reassembly_E2E
--- PASS: TestHandler_Reassembly_E2E (0.00s)
PASS
ok      github.com/rykth/tcp-ip-stack/pkg/ip   1.054s
```

### Checksum tests — RFC 1071 by the book

Five tests pin the algorithm:

- **`TestChecksum_RFC1071Vector`** — the canonical example from RFC
  1071 §3: `00 01 f2 03 f4 f5 f6 f7 → 0x220d`. If this passes, the
  basic implementation is correct.
- **`TestChecksum_Verification`** — feeds the checksum back through
  `Checksum`, including the just-computed value in the input. The
  result must be `0`, which is what `Parse` checks.
- **`TestChecksum_OddLength`** — the odd-length pad-with-zero case.
  Three bytes `0x01 0x02 0x03` should sum as `0x0102 + 0x0300` because
  the trailing odd byte is shifted left by 8.
- **`TestAdd16_CombinesChecksums`** — the algebraic identity
  `Add16(Checksum(A), Checksum(B)) == Checksum(A||B)`. The transport
  layers will lean heavily on this.
- **`TestPseudoHeaderChecksum_NotZero`** — sanity check that the
  function actually does something with its inputs.

### Header codec — six tests

`TestParse_ValidMinimal`, `TestParse_TooShort`, `TestParse_BadVersion`,
`TestParse_BadChecksum`, `TestMarshal_RoundTrip`,
`TestMarshal_FlagsMF`. Together they cover every error path in `Parse`
and verify that every field survives a marshal/parse round-trip.

`TestParse_BadChecksum` is the small one worth highlighting:

```go
hdr := ip.Header{Version: 4, IHL: 5, TotalLen: ip.MinHeaderLen,
                  TTL: 64, Protocol: ip.ProtocolICMP, Src: ipA, Dst: ipB}
buf := make([]byte, ip.MinHeaderLen)
ip.Marshal(buf, hdr, nil)
buf[10] ^= 0xFF // corrupt the checksum

_, err := ip.Parse(buf)
if !errors.Is(err, ip.ErrBadChecksum) { t.Errorf(...) }
```

Build a valid header, XOR one byte of the checksum field, parse — must
fail. Without this test a `Marshal` bug that zeroed the checksum field
and forgot to write the computed value back would slip past every other
test in the suite, because everything else uses well-formed inputs.

### Routing — four tests

- **`TestTable_LPM`** registers three routes (`/16`, `/24`, default)
  and verifies each destination matches the longest one. `10.0.0.5`
  matches the `/24`, `10.0.1.1` matches the `/16`, `192.168.1.1`
  falls back to the default.
- **`TestTable_NoRoute`** — empty table returns `ErrNoRoute`.
- **`TestTable_Delete`** — add a route, look it up, delete it by
  network spec, confirm subsequent lookups fail.
- **`TestTable_MetricTieBreak`** — add two routes for the same network
  with metrics 10 and 1, in that order. Lookup must return the metric-1
  route. This is what `sort.SliceStable`'s tie-break rule guarantees.

### Reassembler — four tests

- **`TestReassembler_NonFragment`** — a non-fragmented header (MF=0,
  offset=0) completes immediately on the first `Add` call. This is the
  fast path: if it didn't work, every non-fragmented datagram would be
  delayed waiting for fragments that will never arrive.
- **`TestReassembler_TwoFragmentsInOrder`** — two fragments arriving
  in offset order. The first call returns `(_, false)`; the second
  returns the assembled payload.
- **`TestReassembler_TwoFragmentsOutOfOrder`** — same scenario but the
  last fragment arrives first. Still has to complete reassembly. The
  test proves that order does not matter.
- **`TestReassembler_GC_DropsExpiredGroups`** — set a 10 ms timeout,
  add one fragment, sleep 20 ms, call `GC`, then try to complete the
  group with the missing fragment. Must return `(_, false)` because GC
  has dropped the in-progress group.

### Handler — seven tests

Seven tests exercise the full handler including receive, send, and
fragmentation:

- **`TestHandler_DeliverToUpper`** — inject a frame, expect the
  registered `UpperHandler` to receive it with `src`, `dst`, and
  payload intact.
- **`TestHandler_UnknownProtocol_Dropped`** — inject a frame with
  protocol number 253 (unassigned/experimentation range). The handler
  must not panic, deadlock, or deliver anywhere.
- **`TestHandler_Send_NoFragmentation`** — send a 4-byte payload
  through a 1500-byte MTU device. Exactly one Ethernet frame must
  appear on the spy device; parsing it must reveal an IP datagram with
  the right source, destination, protocol, and payload.
- **`TestHandler_Send_WithFragmentation`** — set the device MTU to
  `ethernet.HeaderLen + ip.MinHeaderLen + 24` (so that
  `maxPayload = 24` bytes per fragment), then send a 48-byte payload.
  Must produce exactly two fragments: the first with MF=1 and offset 0,
  the second with MF=0 and offset 3 (= 24/8). Both share the same ID.
  The test then feeds the two fragments back into a `Reassembler` and
  verifies the original payload comes back out. End-to-end fragmentation
  in one test.
- **`TestHandler_RegisterUpper_Duplicate`** — registering two upper
  handlers for the same protocol returns `ErrDuplicateProto`. Same
  pattern as `net.ErrDuplicateEtherType`.
- **`TestHandler_Shutdown_NoLeak`** — `goleak.VerifyNone` after
  `ctx.Cancel`. Both the receive loop and the reassembler GC goroutine
  must exit cleanly.
- **`TestHandler_Reassembly_E2E`** — the most realistic test: inject
  two fragments *out of order* through the receive channel, expect the
  upper handler to receive the fully reassembled payload. This is what
  `ping -s 4000` looks like to our handler on the wire.

The handler tests use a `spyDevice` (records every `Write`), a `mockARP`
(returns a hardcoded MAC), and a `mockUpper` (records every `Deliver`).
Each fixture is small and tightly focused on one role — together they
let the tests cover all the integration points without ever opening a
real device.

---

## Driving it on a real TAP device

The same three-terminal pattern. In one terminal, create the device, set
its IP, and run the stack:

```bash
sudo ip tuntap add dev tap0 mode tap user $USER
sudo ip link set tap0 up
sudo ip addr add 10.0.0.1/24 dev tap0
go run ./examples/ip
```

The driver:

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
)

// echoUpper prints every received datagram.
type echoUpper struct{}

func (echoUpper) Protocol() ip.Protocol { return ip.ProtocolICMP }
func (echoUpper) Deliver(src, dst [4]byte, payload []byte) {
    fmt.Printf("ICMP from %v to %v (%d bytes)\n", src, dst, len(payload))
}

func main() {
    dev, err := tuntap.New("tap0", tuntap.DeviceTAP)
    if err != nil {
        log.Fatal(err)
    }
    defer dev.Close()

    localMAC := ethernet.Addr{0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0x02}
    localIP  := [4]byte{10, 0, 0, 2}

    arpH := arp.NewHandler(localMAC, localIP, dev)
    table := ip.NewTable()
    onLink, _ := ip.ParseNetwork("10.0.0.0/24")
    table.Add(ip.Route{Network: onLink, Dev: dev})

    ipH := ip.NewHandler(localMAC, localIP, arpH, table)
    if err := ipH.RegisterUpper(echoUpper{}); err != nil {
        log.Fatal(err)
    }

    r := netpkg.New()
    if err := r.AddDevice(dev);    err != nil { log.Fatal(err) }
    if err := r.AddHandler(arpH);  err != nil { log.Fatal(err) }
    if err := r.AddHandler(ipH);   err != nil { log.Fatal(err) }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    log.Fatal(r.Start(ctx))
}
```

In a second terminal, watch the wire:

```bash
sudo tcpdump -i tap0 -n -e -vv 'ip or arp'
```

In a third, ping our stack:

```bash
ping -c 1 10.0.0.2
```

You will see this on `tcpdump`:

```
ARP, Request who-has 10.0.0.2 tell 10.0.0.1, length 28
ARP, Reply 10.0.0.2 is-at aa:bb:cc:dd:ee:02, length 28
IP (tos 0x0, ttl 64, id 12345, offset 0, flags [DF], proto ICMP (1),
    length 84) 10.0.0.1 > 10.0.0.2: ICMP echo request, id 7, seq 1, length 64
```

And our program prints:

```
ICMP from [10 0 0 1] to [10 0 0 2] (64 bytes)
```

The full path runs: kernel → ARP exchange (our ARP handler responds) →
ICMP echo request → our IP handler parses it, dispatches to `echoUpper`,
prints. The reply does not come back because we have not implemented ICMP
yet — but the receive path is fully alive.

To exercise fragmentation, ask the kernel to ping with a large payload:

```bash
# 4000 bytes — kernel will fragment into ~3 pieces at 1500 MTU
ping -c 1 -s 4000 10.0.0.2
```

`tcpdump` shows the fragments; our reassembler stitches them back; the
upper handler still sees one payload of the original size. This is the
behaviour `TestHandler_Reassembly_E2E` pins in code.

---

## Design decisions, revisited

### Why a separate `Reassembler` type?

The reassembler is a stand-alone type with its own constructor, options,
GC goroutine, and tests. The Handler embeds a `*Reassembler` rather than
implementing the fragment map inline. This split makes two things
possible:

1. **Test fragments without the handler.** Four of the 26 tests use
   `Reassembler` directly. Combining them with the handler would couple
   them to ARP, routing, and the receive loop.
2. **Configurable timeout.** `WithReassemblyTimeout(10 * time.Millisecond)`
   in the GC test sets a 10 ms deadline; the production default of 30
   seconds is unchanged. The handler does not need to know that the
   reassembler is configurable; the reassembler does not need to know
   that it lives inside a handler.

### Why is `ARPResolver` an interface, not a concrete type?

The handler depends on `Resolve(ctx, ip) → (MAC, error)`. That is the
whole surface it needs. Declaring an interface means:

- The test suite injects a `mockARP{mac: macB}` that returns a
  hardcoded MAC. No goroutines, no broadcasts, no timing.
- `pkg/ip` does not import `pkg/arp`. The two packages are siblings
  that can be developed and tested independently.
- A future implementation — say, a static-table resolver for hosts that
  never need to ARP, or a different ARP variant — can be substituted
  without modifying `pkg/ip` at all.

This is the same pattern `net.LinkDevice` and
`net.ProtocolHandler` use, applied one layer up.

### Why `nextID atomic.Uint32` and not a per-flow counter?

RFC 791 §3.2 says ID values must be unique per (Src, Dst, Protocol)
4-tuple "for the maximum time the datagram could live in the network."
Linux uses a global atomic counter and gets away with it because the ID
field is 16 bits — 65,536 packets between collisions, which at gigabit
speeds is about 50 ms. Receivers correctly handle the collision because
the reassembler timeout (30 s) is longer than any plausible network
lifetime.

We do the same. A more correct implementation would key the counter by
(Dst, Protocol), but at our scale and traffic profile the simpler version
behaves identically.

### Why does `process` always copy the payload?

The Registry's contract says the `Frame` is owned by the receiver — but
only until the handler returns. The IP handler returns immediately after
calling `upper.Deliver`, so if `Deliver` retained the payload slice, it
could outlive the Registry's read buffer.

Copying once in `process` means every upper handler can hold onto the
payload without ceremony. Alternatives — make the upper handler copy,
or change the Registry's contract to hand over ownership — both push
complexity to multiple call sites instead of solving it once at the
boundary.

---

## Limitations

| Limitation | Detail |
|------------|--------|
| **No IPv6** | Only `EtherTypeIPv4` is claimed. Adding IPv6 is a parallel package, not a modification |
| **No options on send** | Outgoing datagrams always have `IHL=5`. We can parse received options but never emit them |
| **No PMTUD** | Path MTU Discovery requires sending DF=1 and reacting to ICMP "fragmentation needed." Our `Send` does not set DF |
| **No source routing** | Loose and strict source-record routing options are recognised as bytes in `Options` but never honoured during dispatch |
| **No forwarding** | If a datagram arrives whose `Dst` is not `localIP`, we drop it. A real router would consult the routing table and forward |
| **Global nextID** | RFC 791 allows per-(Dst, Protocol) counters; we use a single atomic. Acceptable at low rates, suboptimal under sustained gigabit-class load |
| **Best-effort reassembly** | Overlapping fragments are clipped rather than rejected. A production stack should drop them |
| **No multicast or broadcast** | Destination addresses with the broadcast pattern (`255.255.255.255`) or in `224.0.0.0/4` are looked up in the routing table like unicast |
| **Linear-scan LPM** | Suitable for tens of routes; full Internet routing (~900k routes) needs a trie |

The "no forwarding" limitation is the boundary between a *host* stack and
a *router* stack. We are firmly the former. Adding forwarding would also
require handling the TTL decrement, sending ICMP "time exceeded" on TTL
expiry, and being careful about loops — substantial work that is out of
scope here.

---

## What is next

With IPv4 in place, the stack now reads datagrams, dispatches them by
protocol number, and sends datagrams that survive both routing and
fragmentation. Three protocols are now within reach, in the order they
fit on top of `UpperHandler`:

1. **ICMP** — the simplest. Receive an echo request, swap source and
   destination, send an echo reply. With a working ICMP handler our
   stack will reply to `ping`.
2. **UDP** — almost as simple as ICMP. A connectionless transport with a
   pseudo-header checksum (which is why we built `Add16` and
   `PseudoHeaderChecksum` already).
3. **TCP** — the substantial one. Connection state, congestion control,
   retransmission, and the segment-and-reassemble logic that makes TCP
   reliable.

The next post starts with ICMP, because the satisfaction of seeing the
kernel's `ping` get a reply from a stack we wrote ourselves is too good
to skip.

The full source is at
[rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack).
