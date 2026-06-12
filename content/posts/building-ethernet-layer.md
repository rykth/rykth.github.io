---
title: "Building the Ethernet layer in Go"
date: 2026-05-20T10:00:00+00:00
description: "The third post in my TCP/IP stack series. After building a loopback device and a TUN/TAP device, we now have raw bytes flowing in and out. This post is about parsing and producing Ethernet II frames — the codec layer that every higher protocol depends on."
tags: [golang, networking, tcpip, ethernet]
---

## Where this fits

So far we have two devices: a [loopback](/posts/building-a-loopback-device) for
in-process unit tests, and a [TUN/TAP device](/posts/building-a-tuntap-device)
for real kernel-routed traffic. Both deliver raw bytes to a `Read` call and
accept raw bytes from `Write`. But raw bytes are not yet useful — we need to
understand what is in them.

The bytes that come off a TAP device are Ethernet II frames. Before any IP
routing, ARP resolution, or TCP handshake can happen, something has to decode
that 14-byte header and hand the payload to the right upper layer. That
something is `pkg/ethernet`.

```
 ┌─────────────────────────┐
 │   Application (HTTP…)   │
 ├─────────────────────────┤
 │   Transport (TCP/UDP)   │
 ├─────────────────────────┤
 │   Network (IP)          │
 ├─────────────────────────┤
 │   Link (Ethernet/ARP) ◄─│  ← this post
 ├─────────────────────────┤
 │   Device (loopback/TAP) │
 └─────────────────────────┘
```

This layer is different from the previous two. The loopback and TUN/TAP devices
are about I/O — moving bytes between goroutines or between userspace and the
kernel. `pkg/ethernet` is a pure codec: no OS calls, no goroutines, no
privileges. It parses byte slices into structured types and marshals structured
types back into byte slices. The entire package is 75 lines of logic.

---

## What is Ethernet?

Ethernet is the dominant Layer 2 (data link) protocol for local area networks.
It defines how data is packaged into *frames* and delivered between nodes that
share the same physical or virtual segment.

The standard most implementations follow today is **Ethernet II** (also called
DIX Ethernet, after its DEC/Intel/Xerox authors). The frame format is fixed: a
14-byte header followed by a variable-length payload.

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │                       Ethernet II Frame                             │
 ├──────────────┬──────────────┬───────────────┬────────────────────── ┤
 │  Dst MAC     │  Src MAC     │  EtherType    │       Payload         │
 │  6 bytes     │  6 bytes     │  2 bytes      │    46–1500 bytes      │
 ├──────────────┴──────────────┴───────────────┴───────────────────────┤
 │◄─────────────── 14-byte header ──────────────►◄──── variable ──────►│
 └─────────────────────────────────────────────────────────────────────┘
```

### The MAC address

A **MAC address** (Media Access Control address) is a 48-bit (6-byte) hardware
identifier assigned to every network interface. It identifies who is speaking
at the link layer, the same way an IP address identifies a routing destination.

```
  aa:bb:cc:dd:ee:ff
  ──────── ────────
       │        │
       │        └── Device identifier (vendor-assigned)
       └─────────── Organizationally Unique Identifier (OUI, IEEE-assigned)
```

Two structural bit flags live in the first octet:

| Bit | Meaning when set |
|-----|-----------------|
| bit 0 (LSB) | **Multicast** — frame is addressed to a group, not one host |
| bit 1 | **Locally administered** — address was overridden, not factory-burned |

The all-ones address `ff:ff:ff:ff:ff:ff` is the **broadcast** address —
every node on the segment receives the frame. Because bit 0 of `0xff` is set,
broadcast is technically a subset of multicast.

### The EtherType field

The 2-byte EtherType field at offset 12 tells the receiving stack which
protocol owns the payload:

| Value | Protocol |
|-------|----------|
| `0x0800` | IPv4 |
| `0x0806` | ARP |
| `0x86DD` | IPv6 |
| `0x8100` | 802.1Q VLAN tag |
| `0x88CC` | LLDP |

Values below `0x0600` are lengths (classic 802.3 Ethernet). Values `0x0600`
and above are EtherTypes. Every modern stack, including ours, uses EtherType.

### Why Ethernet exists

Ethernet solves three link-layer problems. IP handles routing between networks.
MAC addresses handle delivery within a single segment. ARP bridges the two by
translating IP addresses to MAC addresses within a segment.

```
  Application (TCP/UDP)
       │
  IP layer — knows destination IP, resolves it to a MAC via ARP
       │
  Ethernet layer — wraps the IP packet in a frame, addresses it to the MAC
       │
  NIC / TUN/TAP driver — transmits the raw frame bytes
```

---

## Exploring Ethernet from the command line

### Show all Ethernet interfaces

```bash
# All interfaces
ip link show

# Only Ethernet-type (excludes loopback, TUN/TAP, bridges)
ip link show type ether
```

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.100/24 brd 192.168.1.255 scope global eth0
```

The `link/ether` line shows the MAC address. The `brd` field is the broadcast
address — always `ff:ff:ff:ff:ff:ff` for standard Ethernet.

```bash
# Classic ifconfig — still useful for reading MAC and MTU at a glance
ifconfig eth0
```

### Read the MAC address

```bash
# From ip link
ip link show eth0 | grep link/ether

# From sysfs — scriptable, no output parsing needed
cat /sys/class/net/eth0/address
```

```
aa:bb:cc:dd:ee:ff
```

`/sys/class/net/<iface>/address` is the authoritative source for the kernel's
view of the MAC. It is updated immediately when the address changes, and
readable by any user — no root or iproute2 needed.

### Change the MAC address

```bash
# The interface must be DOWN before changing its MAC
sudo ip link set eth0 down
sudo ip link set eth0 address 02:00:00:00:00:01
sudo ip link set eth0 up
```

Setting bit 1 of the first octet (`0x02`) marks the address as locally
administered — a useful convention to distinguish crafted addresses from
factory-burned ones. Setting bit 0 would make the address multicast, which
is typically unintentional and confuses ARP.

```bash
# Classic approach via ifconfig
sudo ifconfig eth0 hw ether 02:00:00:00:00:01
```

### Change the MTU

```bash
# Enable jumbo frames (requires switch and NIC support)
sudo ip link set eth0 mtu 9000

# Restore standard Ethernet MTU
sudo ip link set eth0 mtu 1500
```

Both ends of a link must agree on an MTU. Mismatched MTUs cause silent
fragmentation on one side and potential drops on the other.

### View the ARP table

ARP is what maps IP addresses to MAC addresses at the link layer. The kernel
maintains a cache of recently learned mappings:

```bash
# All learned MAC-to-IP mappings
ip neigh show

# For a specific interface
ip neigh show dev eth0
```

```
192.168.1.1 dev eth0 lladdr aa:bb:cc:00:11:22 REACHABLE
192.168.1.5 dev eth0 lladdr de:ad:be:ef:ca:fe STALE
```

`REACHABLE` means the entry was confirmed recently. `STALE` means it has not
been used and may be re-probed on next use. `FAILED` means ARP got no reply
and the host is considered unreachable.

```bash
# Classic arp tool
arp -n

# Force re-resolution by flushing the cache
sudo ip neigh flush dev eth0
```

Flushing the ARP cache is useful when testing our stack — it forces the kernel
to re-issue ARP requests, which our implementation must respond to correctly.

### Capture Ethernet frames

`tcpdump` with the `-e` flag shows the Ethernet header on every line, making
it possible to inspect MAC addresses directly:

```bash
# Show Ethernet headers on every captured frame
sudo tcpdump -i eth0 -n -e

# ARP frames only — useful for debugging address resolution
sudo tcpdump -i eth0 -n arp

# Frames to or from a specific MAC address
sudo tcpdump -i eth0 ether host aa:bb:cc:dd:ee:ff

# Only broadcast frames
sudo tcpdump -i eth0 ether broadcast

# Only multicast (includes broadcast)
sudo tcpdump -i eth0 ether multicast

# Full hex dump — see the raw bytes, including the 14-byte header
sudo tcpdump -i eth0 -n -X

# Save to a file and open in Wireshark for the full decode tree
sudo tcpdump -i eth0 -w /tmp/capture.pcap
sudo tcpdump -r /tmp/capture.pcap -n -e
```

### Filter by EtherType

```bash
# IPv4 frames only
sudo tcpdump -i eth0 -n 'ether proto 0x0800'

# ARP frames only
sudo tcpdump -i eth0 -n 'ether proto 0x0806'

# IPv6 frames only
sudo tcpdump -i eth0 -n 'ether proto 0x86dd'

# Count frames by EtherType distribution (requires tshark)
sudo tshark -i eth0 -T fields -e eth.type | sort | uniq -c | sort -rn
```

### Send a raw Ethernet frame

```bash
# Craft and send an arbitrary frame with scapy (Python)
sudo python3 -c "
from scapy.all import *
frame = Ether(dst='ff:ff:ff:ff:ff:ff', src='aa:bb:cc:dd:ee:01', type=0x0806) / b'\x00' * 28
sendp(frame, iface='tap0')
"
```

This is invaluable for integration testing: you can inject a crafted ARP
request directly at our TAP device and verify that our stack responds with
a correctly formatted ARP reply — visible in `tcpdump` on the same interface.

### Show NIC hardware details

```bash
# Driver name, version, and bus slot
sudo ethtool -i eth0

# Offload capabilities — which checksums and segmentation the NIC handles in hardware
sudo ethtool -k eth0

# Supported link speeds and auto-negotiation state
sudo ethtool eth0
```

---

## The implementation

### `Addr` — a value type for MAC addresses

The first design decision is the type used to represent a MAC address. The Go
standard library provides `net.HardwareAddr`, but it is defined as:

```go
type HardwareAddr []byte
```

A slice. That has three concrete problems for a network stack:

```go
// net.HardwareAddr
a := net.HardwareAddr{0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0xff}
b := net.HardwareAddr{0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0xff}

a == b               // compile error: slice cannot be compared with ==
bytes.Equal(a, b)    // works, but verbose
m := map[net.HardwareAddr]string{}  // compile error: slice not comparable
```

Our implementation uses a `[6]byte` array instead:

```go
type Addr [6]byte
```

Array types in Go are value types. They are comparable, usable as map keys,
and copied by value rather than by slice header. All three problems go away:

```go
a := ethernet.Addr{0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0xff}
b := ethernet.Addr{0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0xff}

a == b                              // true — direct 6-byte comparison
cache := map[ethernet.Addr]net.IP{} // valid — Addr is comparable
```

The ARP cache will be a `map[ethernet.Addr]Entry`. That only works because
`Addr` is an array. A slice-based type would require a wrapper with a custom
hash function to achieve the same result.

### `Broadcast` and multicast detection

```go
var Broadcast = Addr{0xff, 0xff, 0xff, 0xff, 0xff, 0xff}

func (a Addr) IsMulticast() bool { return a[0]&0x01 != 0 }
func (a Addr) IsBroadcast() bool { return a == Broadcast }
```

`IsMulticast` tests bit 0 of the first octet — the IEEE-defined multicast bit.
This single check correctly identifies both group multicast addresses (e.g.
`01:00:5e:xx:xx:xx` for IPv4 multicast) and the broadcast address, because all
bits of `0xff` are set including bit 0.

`IsBroadcast` narrows to exactly the all-ones case. Because `Addr` is a value
type, `a == Broadcast` is a single 6-byte comparison. On amd64, the compiler
can emit this as a 48-bit integer comparison rather than a loop.

### `Parse` — zero-copy frame decoding

```go
func Parse(b []byte) (Frame, error) {
    if len(b) < HeaderLen {
        return Frame{}, ErrFrameTooShort
    }
    var f Frame
    copy(f.Dst[:], b[0:6])
    copy(f.Src[:], b[6:12])
    f.EtherType = EtherType(binary.BigEndian.Uint16(b[12:14]))
    f.Payload = b[14:]
    return f, nil
}
```

The function allocates nothing. `Frame` is a struct value returned on the
stack. The two `copy` calls for `Dst` and `Src` are 6 bytes each — 12 bytes
total, fast enough that no benchmark has ever flagged them.

`f.Payload = b[14:]` is a slice header assignment, not a copy. `Payload` points
directly into the caller's buffer `b`. No allocation, no memcpy for the payload.

This matters in a receive loop:

```go
var buf [1514]byte   // stack-allocated, one buffer per device goroutine
for {
    n, _ := dev.Read(buf[:])
    f, _ := ethernet.Parse(buf[:n])
    // f.Payload aliases buf[14:n] — zero heap allocation
    dispatch(f)
    // buf is reused on the next iteration; dispatch must have copied f.Payload
    // before returning if it needs to keep it
}
```

The caller is responsible for copying `f.Payload` before the next `Read` call
overwrites `buf`. This is a deliberate contract: we push the allocation decision
to the layer that knows whether a copy is needed. Most paths — ARP resolution
and IP forwarding — copy the payload exactly once into their own structures and
never need a second copy.

### `Marshal` — zero-alloc frame encoding

```go
func Marshal(dst []byte, h Header, payload []byte) (int, error) {
    need := HeaderLen + len(payload)
    if len(dst) < need {
        return 0, fmt.Errorf("ethernet: marshal buffer too small: need %d, have %d", need, len(dst))
    }
    copy(dst[0:6], h.Dst[:])
    copy(dst[6:12], h.Src[:])
    binary.BigEndian.PutUint16(dst[12:14], uint16(h.EtherType))
    copy(dst[14:], payload)
    return need, nil
}
```

The caller supplies the destination buffer. `Marshal` writes into it and
returns the byte count. No allocations on the happy path.

The alternative — `return []byte` — would force a heap allocation per outgoing
frame. The upper layers (IP, ARP) pre-allocate a transmit buffer and call
`Marshal` into it. The buffer is reused across frames.

### `binary.BigEndian` — network byte order

Ethernet puts the EtherType field in big-endian order (network byte order).
We read it with:

```go
f.EtherType = EtherType(binary.BigEndian.Uint16(b[12:14]))
```

And write it with:

```go
binary.BigEndian.PutUint16(dst[12:14], uint16(h.EtherType))
```

Both calls compile to a pair of byte loads/stores with a byte-swap instruction
(`bswap` on amd64). They do not allocate and do not use `encoding/binary.Read`,
which goes through reflection. This is the idiomatic zero-alloc binary encoding
pattern in Go.

### `Frame` embeds `Header`

```go
type Header struct {
    Dst       Addr
    Src       Addr
    EtherType EtherType
}

type Frame struct {
    Header
    Payload []byte
}
```

Embedding `Header` into `Frame` promotes its fields. Callers write `f.Dst`,
`f.EtherType` instead of `f.Header.Dst`, `f.Header.EtherType`. The dispatch
switch at the top of the receive path reads naturally:

```go
switch f.EtherType {
case ethernet.EtherTypeIPv4:
    ipHandler.Deliver(f.Src, f.Payload)
case ethernet.EtherTypeARP:
    arpHandler.Deliver(f.Src, f.Payload)
}
```

`Header` is also passed directly to `Marshal`:

```go
ethernet.Marshal(txBuf[:], ethernet.Header{
    Dst:       peerMAC,
    Src:       myMAC,
    EtherType: ethernet.EtherTypeARP,
}, arpPayload)
```

### Sentinel errors

```go
var ErrFrameTooShort = errors.New("ethernet: frame too short")
var ErrInvalidAddr   = errors.New("ethernet: invalid MAC address")
```

Both errors have exactly one cause. Wrapping them with context (`fmt.Errorf`
with `%w`) would add nothing — the string already identifies the package and
the problem. Sentinel errors allow callers to branch without parsing strings:

```go
f, err := ethernet.Parse(buf[:n])
if errors.Is(err, ethernet.ErrFrameTooShort) {
    dropCounter++
    continue
}
```

The `"package: description"` format is consistent with the convention used
throughout the stack.

### `ParseAddr` — not on the data path

```go
func ParseAddr(s string) (Addr, error) {
    parts := strings.Split(s, ":")
    if len(parts) != 6 {
        return Addr{}, ErrInvalidAddr
    }
    var a Addr
    for i, p := range parts {
        v, err := strconv.ParseUint(p, 16, 8)
        if err != nil {
            return Addr{}, ErrInvalidAddr
        }
        a[i] = byte(v)
    }
    return a, nil
}
```

`ParseAddr` is for startup-time configuration parsing — reading a MAC address
from a config file or a command-line flag. It allocates (via `strings.Split`)
and makes no attempt to be fast. That is fine because it is never called in
the receive or transmit loop.

`strconv.ParseUint(p, 16, 8)` accepts both lowercase and uppercase hex digits,
so the parser is case-insensitive without any extra work — `aa:bb:...` and
`AA:BB:...` both decode to the same `Addr`.

The error returned is always `ErrInvalidAddr`, never a wrapped error with
string context. There is only one failure mode — the string is not a valid MAC
address — and extra context would not help the caller decide what to do about it.

### `String` — the inverse of `ParseAddr`

```go
func (a Addr) String() string {
    return fmt.Sprintf("%02x:%02x:%02x:%02x:%02x:%02x",
        a[0], a[1], a[2], a[3], a[4], a[5])
}
```

Lowercase, colon-separated, zero-padded. The format matches what `ip link`,
`tcpdump -e`, and `ParseAddr` all expect, which means a MAC printed by the
stack can be copy-pasted back into a config file and round-trip cleanly. The
method also makes `Addr` satisfy `fmt.Stringer`, so `t.Logf("%v", a)` and
`log.Printf("%s", a)` produce readable output without per-call formatting at
the call site.

---

## The tests

```bash
# Plain run — no privileges, no cgo
go test ./pkg/ethernet/... -v

# With the race detector (needs cgo)
CGO_ENABLED=1 go test ./pkg/ethernet/... -race -v

# Force a fresh run (Go caches passing test results by default)
go test ./pkg/ethernet/... -count=1
```

No root. No OS calls. No network. The entire suite runs in memory.

The suite is ten tests plus two benchmarks. The first half exercises the
binary layout (`Parse`, `Marshal`, their error paths, and the zero-copy
guarantee); the second half exercises the `Addr` type (`IsMulticast`,
`IsBroadcast`, `String`, `ParseAddr`) and the `EtherType` constants.

### Round-trip: Parse then Marshal

```go
func TestParse_roundtrip(t *testing.T) {
    payload := []byte{0x01, 0x02, 0x03, 0x04}
    raw := buildFrame(addrA, addrB, ethernet.EtherTypeIPv4, payload)

    f, err := ethernet.Parse(raw)
    // verify f.Dst, f.Src, f.EtherType, f.Payload byte-by-byte
}

func TestMarshal_roundtrip(t *testing.T) {
    h := ethernet.Header{Dst: addrA, Src: addrB, EtherType: ethernet.EtherTypeARP}
    payload := []byte{0x10, 0x20, 0x30}

    dst := make([]byte, ethernet.HeaderLen+len(payload))
    n, _ := ethernet.Marshal(dst, h, payload)

    f, _ := ethernet.Parse(dst[:n])
    // verify f matches h and payload
}
```

The round-trip test is the most basic correctness check: build a raw frame
manually, parse it, verify every field. Then go the other direction: marshal
a header and payload into a buffer, parse the result, verify it matches the
input. If these two tests pass, the byte layout is correct.

### Zero-copy: payload aliases the input buffer

```go
func TestParse_payloadAliasesInput(t *testing.T) {
    raw := buildFrame(addrA, addrB, ethernet.EtherTypeIPv4, []byte{0xde, 0xad})

    f, _ := ethernet.Parse(raw)

    // Mutate the original buffer after Parse
    raw[14] = 0xff

    // Payload must reflect the mutation — proves no copy was made
    if f.Payload[0] != 0xff {
        t.Errorf("Parse must not copy payload")
    }
}
```

This is the zero-copy contract test. It mirrors the copy-on-write test in the
loopback package, but in the opposite direction: here we *want* the mutation to
propagate, proving that `Payload` genuinely aliases the input rather than
silently copying it.

### Error cases

```go
func TestParse_tooShort(t *testing.T) {
    cases := [][]byte{
        nil,
        {},
        make([]byte, ethernet.HeaderLen-1),
    }
    for _, b := range cases {
        _, err := ethernet.Parse(b)
        if !errors.Is(err, ethernet.ErrFrameTooShort) {
            t.Errorf("len=%d: got %v, want ErrFrameTooShort", len(b), err)
        }
    }
}
```

Three inputs that are all too short: nil, empty, and one byte short of a full
header. All three must return `ErrFrameTooShort`. The test uses `errors.Is`
rather than string comparison, so it stays correct even if the error message
is ever reworded.

### `Marshal` rejects undersized buffers

```go
func TestMarshal_bufferTooSmall(t *testing.T) {
    h := ethernet.Header{EtherType: ethernet.EtherTypeIPv4}
    payload := []byte{0x01, 0x02}

    dst := make([]byte, ethernet.HeaderLen)  // exactly 14 bytes — no room for payload
    _, err := ethernet.Marshal(dst, h, payload)
    if err == nil {
        t.Fatal("expected error for too-small buffer, got nil")
    }
}
```

`Marshal` writes header + payload, so `dst` must be at least
`HeaderLen + len(payload)` bytes. The test sizes `dst` to exactly the header
length and passes a 2-byte payload, deliberately one of the easiest off-by-one
mistakes the upper layers could make. A silent partial write here would
produce truncated frames that pass higher-level length checks but get rejected
by every receiver — exactly the kind of bug that takes hours to find via
`tcpdump`.

### `IsMulticast` — the bit rule

```go
cases := []struct {
    addr ethernet.Addr
    want bool
}{
    {ethernet.Addr{0x01, ...}, true},   // multicast bit set
    {ethernet.Broadcast, true},          // ff:ff:ff:ff:ff:ff — bit 0 of 0xff is set
    {ethernet.Addr{0x00, ...}, false},  // unicast
    {ethernet.Addr{0xaa, ...}, false},  // 0xaa = 0b10101010 — bit 0 is clear
    {ethernet.Addr{0x03, ...}, true},   // 0x03 = 0b00000011 — bit 0 set
}
```

The table tests both the happy path and the easy-to-miss case: `0xaa` looks
"multicast-ish" with its high bit set, but bit 0 is clear, so it is unicast.
`0x03` has both bits set. The test documents exactly what the IEEE spec says
about the bit position.

### `IsBroadcast` — the narrow case

```go
func TestAddr_IsBroadcast(t *testing.T) {
    if !ethernet.Broadcast.IsBroadcast() {
        t.Error("Broadcast.IsBroadcast() = false, want true")
    }
    if addrA.IsBroadcast() {
        t.Errorf("%v.IsBroadcast() = true, want false", addrA)
    }
}
```

Two assertions: the broadcast constant identifies as broadcast, and an
arbitrary unicast address does not. The discriminator is the full 6-byte
equality check, so any address that is "almost ff:ff:ff:ff:ff:ff" — for
example `ff:ff:ff:ff:ff:fe` — correctly reports `false`.

### `ParseAddr` — every malformed case

```go
cases := []struct {
    input   string
    want    ethernet.Addr
    wantErr bool
}{
    {"aa:bb:cc:dd:ee:ff", ethernet.Addr{0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0xff}, false},
    {"00:00:00:00:00:00", ethernet.Addr{},                                  false},
    {"ff:ff:ff:ff:ff:ff", ethernet.Broadcast,                                false},
    {"AA:BB:CC:DD:EE:FF", ethernet.Addr{0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0xff}, false},  // case-insensitive
    {"aa:bb:cc:dd:ee",    ethernet.Addr{},                                  true},   // too few groups
    {"aa:bb:cc:dd:ee:ff:00", ethernet.Addr{},                               true},   // too many groups
    {"aa:bb:cc:dd:ee:gg", ethernet.Addr{},                                  true},   // invalid hex
    {"",                  ethernet.Addr{},                                  true},
}
```

Eight cases covering every realistic failure mode plus three happy paths
(including uppercase input, since `strconv.ParseUint` is case-insensitive).
The `wantErr bool` design means each row asserts only one thing — the result
matches the expected `Addr`, or an error is returned — which keeps each
failure message specific to a single input.

### `String` round-trips with `ParseAddr`

```go
func TestAddr_String(t *testing.T) {
    a := ethernet.Addr{0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0xff}
    if got := a.String(); got != "aa:bb:cc:dd:ee:ff" {
        t.Errorf("String() = %q, want %q", got, "aa:bb:cc:dd:ee:ff")
    }
}
```

Pinned format: lowercase, colon-separated, zero-padded. Combined with
`TestParseAddr`'s lowercase-and-uppercase cases above, this proves that
`ParseAddr(a.String()) == a` for every well-formed address.

### EtherType constants

```go
func TestEtherType_constants(t *testing.T) {
    if ethernet.EtherTypeIPv4 != 0x0800 { ... }
    if ethernet.EtherTypeARP  != 0x0806 { ... }
    if ethernet.EtherTypeIPv6 != 0x86DD { ... }
}
```

This looks obvious — of course the constants are correct. But they are the
values that determine which handler processes each frame. If they ever drift
from the IANA-assigned values (say, because someone typed `0x0860` instead of
`0x0806`), every ARP frame would be silently discarded. Locking them down
explicitly means the tests catch the typo before it reaches a running stack.

### A clean test run

```
=== RUN   TestParse_roundtrip
--- PASS: TestParse_roundtrip (0.00s)
=== RUN   TestParse_tooShort
--- PASS: TestParse_tooShort (0.00s)
=== RUN   TestParse_payloadAliasesInput
--- PASS: TestParse_payloadAliasesInput (0.00s)
=== RUN   TestMarshal_roundtrip
--- PASS: TestMarshal_roundtrip (0.00s)
=== RUN   TestMarshal_bufferTooSmall
--- PASS: TestMarshal_bufferTooSmall (0.00s)
=== RUN   TestAddr_IsMulticast
--- PASS: TestAddr_IsMulticast (0.00s)
=== RUN   TestAddr_IsBroadcast
--- PASS: TestAddr_IsBroadcast (0.00s)
=== RUN   TestParseAddr
--- PASS: TestParseAddr (0.00s)
=== RUN   TestEtherType_constants
--- PASS: TestEtherType_constants (0.00s)
=== RUN   TestAddr_String
--- PASS: TestAddr_String (0.00s)
PASS
ok      github.com/rykth/tcp-ip-stack/pkg/ethernet     1.008s
```

### Benchmarks

```bash
# Run only the benchmarks, skip the regular tests
go test ./pkg/ethernet/... -bench=. -benchmem -run=^$ -count=1
```

The result on a 12th-gen Intel laptop:

```
goos: linux
goarch: amd64
pkg: github.com/rykth/tcp-ip-stack/pkg/ethernet
cpu: 12th Gen Intel(R) Core(TM) i7-1255U
BenchmarkParse-12      178106964     6.663 ns/op    0 B/op    0 allocs/op
BenchmarkMarshal-12     87471046    13.72  ns/op    0 B/op    0 allocs/op
```

Zero allocations per operation on both paths. At 6.7 ns per parse, the
Ethernet layer adds under 1 µs of overhead at 100 000 frames per second. The
`0 allocs/op` result is a regression gate: if any future change accidentally
introduces an allocation on the data path — say, by replacing the `[6]byte`
array with a slice — the benchmark will flag it immediately.

`Marshal` is slower than `Parse` because it does three `copy` calls plus a
`PutUint16` (writing the header out), whereas `Parse` only does two `copy`
calls for the MAC addresses and a slice-header assignment for the payload.
The asymmetry is fundamental and not worth trying to remove.

---

## Using it on top of a real device

The Ethernet layer sits between the device and every higher protocol. Until
the ARP and IP layers exist, the most useful thing we can build on top of it
is a frame sniffer — a Go program that does what `tcpdump -i tap0 -e` does,
using only the code we have so far. Against a TAP device it looks like this:

```go
dev, err := tuntap.New("tap0", tuntap.DeviceTAP)
if err != nil {
    log.Fatal(err) // typically: missing CAP_NET_ADMIN
}
defer dev.Close()

var buf [1514]byte // 14-byte header + 1500-byte payload
for {
    n, err := dev.Read(buf[:])
    if err != nil {
        return
    }

    f, err := ethernet.Parse(buf[:n])
    if err != nil {
        continue // drop malformed frames
    }

    fmt.Printf("%s → %s  type=%#04x  len=%d\n",
        f.Src, f.Dst, uint16(f.EtherType), len(f.Payload))
}
```

The `f.Src` / `f.Dst` calls use the `String` method we just covered, so the
log lines look like `aa:bb:cc:dd:ee:ff → ff:ff:ff:ff:ff:ff  type=0x0806
len=28` — readable without any extra formatting.

The transmit side is the inverse. Once an ARP package exists in a future post,
sending a broadcast ARP request will look like this — every symbol below
already exists today, except `arp.MarshalRequest`:

```go
arpPayload := arp.MarshalRequest(myMAC, myIP, targetIP) // future package

h := ethernet.Header{
    Dst:       ethernet.Broadcast,
    Src:       myMAC,
    EtherType: ethernet.EtherTypeARP,
}

var txBuf [1514]byte
n, _ := ethernet.Marshal(txBuf[:], h, arpPayload)
dev.Write(txBuf[:n])
```

One `Parse` call per frame on the receive side, one `Marshal` call per frame
on the transmit side, zero heap allocations on either path. The
`ethernet.Broadcast` constant is the `ff:ff:ff:ff:ff:ff` address — using it
explicitly makes the broadcast intent obvious at the call site, instead of
hiding it in a literal.

To see the receive loop work, run the sniffer against a freshly created TAP
device in one terminal and inject a crafted frame in another:

```bash
# Terminal 1 — create the device and run our sniffer
sudo ip tuntap add dev tap0 mode tap user $USER
sudo ip link set tap0 up
sudo ip addr add 192.168.100.1/24 dev tap0
go run ./examples/loopback   # placeholder path; the sniffer above

# Terminal 2 — pump a broadcast ARP request into tap0
sudo python3 -c "
from scapy.all import *
sendp(Ether(dst='ff:ff:ff:ff:ff:ff', src='aa:bb:cc:dd:ee:01', type=0x0806)
      / b'\x00'*28, iface='tap0')
"
```

The sniffer prints exactly one line — the ARP frame, EtherType `0x0806`,
28-byte payload. That round trip (scapy → kernel → TAP fd → `unix.Read` →
`ethernet.Parse` → `fmt.Printf`) is the first time our stack produces output
from real wire-format input.

---

## Lifting it to a registry

The sniffer above is fine for one device and one protocol. The moment we
have ARP plus IP running on a TAP plus a loopback, the receive loop becomes
a tangle of switch statements, goroutines, and `defer` calls that close
things in the right order. That is the problem `pkg/net` exists to solve. It
is the structural counterpart to the `pkg/ethernet` codec: where ethernet
turns bytes into typed values, `pkg/net` turns typed values into dispatched
events on the right handler's channel.

```
 ┌───────────────────────────────────────────────────────────┐
 │                       net.Registry                        │
 │                                                           │
 │   ┌──────────┐    rxLoop      ┌───────────────────────┐   │
 │   │ loopback │ ──────────────►│  ARP   ProtocolHandler│   │
 │   │          │   ethernet.    │                       │   │
 │   │ tuntap   │   Parse +      │  IPv4  ProtocolHandler│   │
 │   └──────────┘   dispatch     └───────────────────────┘   │
 └───────────────────────────────────────────────────────────┘
       N devices, one rxLoop each       M handlers, keyed by EtherType
```

The package introduces three building blocks. None of them are large
individually; the value is in how cleanly they compose.

### `LinkDevice` — the device interface

```go
type LinkDevice interface {
    Name() string
    MTU() int
    Read(p []byte) (int, error)
    Write(p []byte) (int, error)
    Close() error
}
```

Five methods. The method set is identical to the public API of
`pkg/loopback.Device` and `pkg/raw/tuntap.Device`, which means both types
satisfy `LinkDevice` without any modification — no `var _ LinkDevice =
(*loopback.Device)(nil)` declaration, no factory wrapper, nothing. That is
structural typing earning its keep.

The directionality of the dependency is what makes this work:
`pkg/loopback` and `pkg/raw/tuntap` import nothing from `pkg/net`.
`pkg/net` imports nothing from them. Each device package stands alone and
remains testable in isolation; the upper stack consumes them through an
interface that lives in the package that needs the abstraction.

### `ProtocolHandler` — the per-EtherType processor

```go
type ProtocolHandler interface {
    EtherType() ethernet.EtherType
    RxChan() chan Frame
    Start(ctx context.Context, errCh chan<- error)
}
```

Every Ethernet protocol the stack will implement (ARP, IPv4, IPv6) satisfies
this interface. The three methods carry the three things the Registry needs
to know:

- `EtherType()` — which protocol the handler claims. Used as the dispatch
  key.
- `RxChan()` — the channel the Registry pushes frames to. The handler
  *owns* the channel — it creates it in its constructor and returns the
  same channel from every call. The Registry never closes it.
- `Start(ctx, errCh)` — the handler's main loop, started by the Registry
  in a fresh goroutine. The handler reads from its own `RxChan()`, does its
  protocol work, and reports errors via `errCh`. It must return promptly
  when `ctx` is cancelled.

The frame delivered to handlers is *not* the same type as `ethernet.Frame`:

```go
type Frame struct {
    Dst     ethernet.Addr
    Src     ethernet.Addr
    Payload []byte       // owned copy — safe to retain or mutate
    Dev     LinkDevice   // the device the frame arrived on
}
```

The differences from `ethernet.Frame` are deliberate. `EtherType` is gone —
by the time the handler sees the frame, the Registry has already used it for
dispatch. `Payload` is now a *copy* the handler owns, not a slice into the
device's read buffer. `Dev` is added, so a multi-homed handler (one ARP
handler serving both a TAP and a loopback) can tell which interface the
request came in on. Each change reflects a contract shift at the boundary
between the codec and the dispatcher.

### `Registry` — the wiring

```go
type Registry struct {
    mu       sync.Mutex
    devices  []LinkDevice
    handlers map[ethernet.EtherType]ProtocolHandler
    drops    atomic.Int64
}
```

Four fields. `devices` is an ordered slice — every registered device gets
its own rxLoop goroutine. `handlers` is a map keyed by EtherType, enforcing
the "at most one handler per protocol" rule at the type level. `drops` is
an atomic counter; we will return to why exactly that counter exists in a
moment. `mu` protects the two collections during registration and during the
per-frame handler lookup.

Registration is straightforward and rejects duplicates:

```go
func (r *Registry) AddDevice(dev LinkDevice) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    for _, d := range r.devices {
        if d.Name() == dev.Name() {
            return ErrDeviceAlreadyRegistered
        }
    }
    r.devices = append(r.devices, dev)
    return nil
}

func (r *Registry) AddHandler(h ProtocolHandler) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    if _, exists := r.handlers[h.EtherType()]; exists {
        return ErrDuplicateEtherType
    }
    r.handlers[h.EtherType()] = h
    return nil
}
```

Two sentinel errors, the same pattern as `ethernet.ErrFrameTooShort`:

```go
var (
    ErrDeviceAlreadyRegistered = errors.New("net: device already registered")
    ErrDuplicateEtherType      = errors.New("net: duplicate EtherType")
)
```

Both let callers branch on the specific failure mode without parsing error
strings — important because both errors are recoverable (rename the device,
remove the duplicate handler, retry).

### `Start` — the event loop

`Start` is the only blocking method on the Registry. It launches a goroutine
for every handler, an rxLoop for every device, and a watchdog that closes
everything when the context is cancelled:

```go
func (r *Registry) Start(ctx context.Context) error {
    // snapshot under the mutex so subsequent Add* calls cannot mutate
    // the slices we are iterating
    r.mu.Lock()
    devices := append([]LinkDevice(nil), r.devices...)
    handlers := make([]ProtocolHandler, 0, len(r.handlers))
    for _, h := range r.handlers {
        handlers = append(handlers, h)
    }
    r.mu.Unlock()

    errCh := make(chan error, len(devices)+len(handlers))
    var wg sync.WaitGroup

    // handler goroutines first — every handler must be reading from its
    // RxChan before any frame can be dispatched to it
    for _, h := range handlers {
        wg.Add(1)
        go func(h ProtocolHandler) {
            defer wg.Done()
            h.Start(ctx, errCh)
        }(h)
    }

    // one rxLoop per device
    for _, dev := range devices {
        wg.Add(1)
        go func(dev LinkDevice) {
            defer wg.Done()
            r.rxLoop(ctx, dev, errCh)
        }(dev)
    }

    // watchdog: when ctx is cancelled, close every device to unblock its Read
    go func() {
        <-ctx.Done()
        for _, dev := range devices {
            dev.Close()
        }
    }()

    wg.Wait()
    close(errCh)

    var errs []error
    for err := range errCh {
        if err != nil {
            errs = append(errs, err)
        }
    }
    return errors.Join(errs...)
}
```

The startup order matters. If we launched the rxLoops first, an early
incoming frame could arrive at a handler whose `Start` has not yet executed
— and the frame would either block the dispatch or (worse, with the
non-blocking send we describe below) be silently counted as a drop. By
spinning up handlers first, every handler is reading from its channel before
the first `dev.Read` returns.

The `errCh` is sized to `len(devices) + len(handlers)`. Every goroutine
either reports zero or one error before returning, so the channel cannot
fill — no goroutine ever blocks trying to send an error. After
`wg.Wait()` we close the channel and drain it into `errors.Join`. The
returned error is `nil` only when every goroutine returned cleanly.

### The rxLoop — where codec meets dispatch

Each device runs in its own copy of this loop:

```go
func (r *Registry) rxLoop(ctx context.Context, dev LinkDevice, errCh chan<- error) {
    buf := make([]byte, dev.MTU()+ethernet.HeaderLen)
    for {
        n, err := dev.Read(buf)
        if err != nil {
            select {
            case <-ctx.Done():
            default:
                errCh <- fmt.Errorf("net: device %s: %w", dev.Name(), err)
            }
            return
        }

        f, err := ethernet.Parse(buf[:n])
        if err != nil {
            continue // drop malformed frames
        }

        r.mu.Lock()
        h, ok := r.handlers[f.EtherType]
        r.mu.Unlock()
        if !ok {
            continue // no handler registered for this EtherType
        }

        payload := make([]byte, len(f.Payload))
        copy(payload, f.Payload)

        frame := Frame{Dst: f.Dst, Src: f.Src, Payload: payload, Dev: dev}
        select {
        case h.RxChan() <- frame:
        default:
            r.drops.Add(1)
        }
    }
}
```

Six steps per frame: read, parse, look up the handler, copy the payload,
send, account for drops. A few details are worth pausing on.

**The buffer lives outside the loop.** `dev.MTU() + ethernet.HeaderLen` is
the largest legal frame on this device. Allocating it once and reusing it
across iterations means the only per-frame allocation in the hot path is
the payload copy. The codec's zero-copy `Parse` is what makes this work —
the `Frame` returned aliases `buf`, so the rxLoop is free to overwrite `buf`
on the next iteration *after* it has copied the payload it needs.

**The non-blocking send is the key correctness property.** A slow ARP
handler must never stall the rxLoop, because that would also stall every
other handler that depends on frames arriving on the same device. The
`select { case h.RxChan() <- frame: default: drops.Add(1) }` pattern
guarantees the rxLoop progresses regardless of handler state, at the cost of
shedding frames the handler cannot keep up with. The back-pressure problem
is pushed to the handler's design — pick a channel size, distribute work,
batch.

**Unknown EtherType drops are silent.** Frames with no registered handler
are discarded without incrementing `Drops`. The drop counter is reserved for
*back-pressure* drops — full handler channel — not for "protocol not
implemented." Mixing the two would make the counter useless when debugging a
production stack: "are we losing IPv4 frames to a slow handler, or are we
seeing IPv6 frames we don't claim to handle?" One number cannot answer both
questions.

**The read-error / ctx.Done dance.** When the watchdog closes the device,
the goroutine in `dev.Read` returns an error. At that point we don't yet
know whether the error is a real fault or just shutdown — but the context
disambiguates: if `ctx.Done()` is already closed, the error is expected and
silenced; otherwise it is reported through `errCh`. The pattern is small,
but it is the difference between a clean shutdown and an alarming
`device closed` line in the logs every time the program exits.

### Shutdown without goroutine leaks

The end-to-end shutdown sequence ties the abstraction together:

1. The caller cancels the context.
2. The watchdog's `<-ctx.Done()` returns and it closes every registered
   device.
3. Each device's `Close()` unblocks the corresponding `dev.Read` call.
4. Each rxLoop sees the read error, peeks at `ctx.Done`, and returns.
5. Each handler's `Start` returns (its contract requires it to honour
   `ctx`).
6. The WaitGroup reaches zero, `Start` collects errors and returns.

Every goroutine has a deterministic exit. The test suite uses
`go.uber.org/goleak` to assert nothing is left behind:

```go
func TestRegistry_Shutdown_NoLeak(t *testing.T) {
    defer goleak.VerifyNone(t)

    dev := loopback.New(loopback.WithName("lo-shutdown"))
    h := newMockHandler(ethernet.EtherTypeIPv4, 8)

    cancel, done := startRegistry(t, dev, h)
    cancel()

    select {
    case err := <-done:
        if err != nil {
            t.Fatalf("Start returned non-nil error: %v", err)
        }
    case <-time.After(2 * time.Second):
        t.Fatal("Start did not return after ctx cancel")
    }
}
```

`goleak.VerifyNone` runs at the end of the test and fails if any non-test
goroutines are still running. If the watchdog or any rxLoop ever forgot to
return, this test would catch it immediately. It is the regression gate for
the entire shutdown protocol — far more robust than asserting on a side
effect like "no log line was printed."

### The Registry tests — eight scenarios

```bash
CGO_ENABLED=1 go test ./pkg/net/... -race -v
```

Eight tests cover the abstraction end-to-end. Each one isolates a single
property:

- **`TestRegistry_DispatchToHandler`** — a frame written to a loopback
  device is parsed and delivered to the matching handler with `Src`,
  `Dst`, `Payload`, and `Dev` all set correctly.
- **`TestRegistry_UnknownEtherType_Dropped`** — a frame with no
  registered handler is dropped without incrementing `Drops` (the property
  we argued for above).
- **`TestRegistry_Backpressure_Drop`** — when a handler's channel is
  full, subsequent frames are dropped *and* `Drops` increments.
- **`TestRegistry_Shutdown_NoLeak`** — `Start` returns within two
  seconds of `ctx.Cancel`, and `goleak` reports no surviving goroutines.
- **`TestRegistry_AddDevice_Duplicate`** — registering two devices with
  the same name returns `ErrDeviceAlreadyRegistered`.
- **`TestRegistry_AddHandler_Duplicate`** — registering two handlers for
  the same EtherType returns `ErrDuplicateEtherType`.
- **`TestRegistry_MultipleDevices`** — frames written to two distinct
  loopback devices are both delivered to a single shared handler, with
  `f.Dev` correctly identifying which device each frame came from.
- **`TestRegistry_PayloadIsCopied`** — the payload delivered to the
  handler survives a subsequent `dev.Read` overwriting the rxLoop's
  internal buffer.

That last test is the structural complement to
`TestParse_payloadAliasesInput` from earlier in this post.
`ethernet.Parse` deliberately *aliases* the input buffer for zero-copy
decoding. The Registry then *copies* on the boundary into a `Frame` owned
by the handler. Both tests together pin the contract: zero-copy at the
codec, owned copy at the dispatch.

A clean run:

```
=== RUN   TestRegistry_DispatchToHandler
--- PASS: TestRegistry_DispatchToHandler (0.00s)
=== RUN   TestRegistry_UnknownEtherType_Dropped
--- PASS: TestRegistry_UnknownEtherType_Dropped (0.02s)
=== RUN   TestRegistry_Backpressure_Drop
--- PASS: TestRegistry_Backpressure_Drop (0.05s)
=== RUN   TestRegistry_Shutdown_NoLeak
--- PASS: TestRegistry_Shutdown_NoLeak (0.00s)
=== RUN   TestRegistry_AddDevice_Duplicate
--- PASS: TestRegistry_AddDevice_Duplicate (0.00s)
=== RUN   TestRegistry_AddHandler_Duplicate
--- PASS: TestRegistry_AddHandler_Duplicate (0.00s)
=== RUN   TestRegistry_MultipleDevices
--- PASS: TestRegistry_MultipleDevices (0.00s)
=== RUN   TestRegistry_PayloadIsCopied
--- PASS: TestRegistry_PayloadIsCopied (0.02s)
PASS
ok      github.com/rykth/tcp-ip-stack/pkg/net   1.103s
```

### Putting it together

The manual sniffer from the previous section becomes this:

```go
import (
    "context"
    "log"

    "github.com/rykth/tcp-ip-stack/pkg/loopback"
    netpkg "github.com/rykth/tcp-ip-stack/pkg/net"
    // these next two packages will exist in upcoming posts; the interface
    // they will satisfy — ProtocolHandler — already exists today
    "github.com/rykth/tcp-ip-stack/pkg/arp"
    "github.com/rykth/tcp-ip-stack/pkg/ip"
)

func main() {
    dev := loopback.New()

    arpH := arp.NewHandler(/* ... */)
    ipH  := ip.NewHandler(/* ... */)

    r := netpkg.New()
    if err := r.AddDevice(dev); err != nil { log.Fatal(err) }
    if err := r.AddHandler(arpH); err != nil { log.Fatal(err) }
    if err := r.AddHandler(ipH);  err != nil { log.Fatal(err) }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    if err := r.Start(ctx); err != nil {
        log.Fatal(err)
    }
}
```

The rxLoop is gone. The switch on EtherType is gone. The shutdown
coordination is gone. The frame buffer is gone. Every concern that was
tangled in the sniffer now lives in exactly one of three places: the
*device* knows how to read and write bytes, the *handler* knows how to
process its protocol, and the *Registry* knows how to wire them together.

That separation is the entire point. The next post will introduce an
`arp.Handler` whose entire job is to read `Frame`s from a channel, look at
the opcode, and write replies. It will not contain a single mention of file
descriptors, Ethernet headers, or goroutine lifecycle. The Registry already
solved those problems.

---

## Limitations

| Limitation | Detail |
|-----------|--------|
| **Ethernet II only** | Classic 802.3 frames (length < `0x0600`) are not decoded; the EtherType field would be misread as a length |
| **No 802.1Q VLAN tags** | A VLAN-tagged frame has EtherType `0x8100` followed by a 4-byte tag before the real EtherType; we do not unwrap it |
| **No FCS** | The 4-byte frame check sequence is stripped by the NIC or TUN/TAP driver; our layer never validates it |
| **No minimum payload enforcement** | Real hardware requires 46-byte minimum payload; we allow any length ≥ 0 |
| **One frame at a time** | `Parse` and `Marshal` process one frame per call; a batch API could reduce dispatch overhead at high packet rates |

The VLAN limitation is the most likely to surface in practice. If your TAP
device is attached to a VLAN-aware bridge, you will see frames with EtherType
`0x8100` that our dispatcher does not know how to handle. The fix is a
pre-parse step that strips the 802.1Q tag before calling `ethernet.Parse`.

---

## What is next

With the Ethernet codec and the `pkg/net` Registry in place, the stack can
now read a raw frame off any registered device, decode the header, and route
it to a per-protocol handler with zero coupling between the two sides. The
next step is ARP — the protocol that resolves IP addresses to MAC addresses
within a subnet. ARP will fit into the abstraction as a single
`ProtocolHandler`: it claims `EtherType == 0x0806`, owns a receive channel,
runs a goroutine, and uses `ethernet.Broadcast` and `ethernet.Addr` exactly
as defined here. Nothing about devices, file descriptors, or shutdown
coordination needs to appear in its code — the Registry already owns those.

The full source is at [rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack).
