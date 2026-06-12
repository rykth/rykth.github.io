---
title: "Building the ARP layer in Go"
date: 2026-05-21T10:00:00+00:00
description: "The fourth post in my TCP/IP stack series. The Ethernet codec turned bytes into typed frames; the net.Registry routed them to per-protocol handlers. Now we implement the first real protocol — ARP — and answer the question every IPv4 packet eventually has to ask: what MAC address belongs to this IP?"
tags: [golang, networking, tcpip, arp, rfc826]
---

## Where this fits

We have a [loopback](/posts/building-a-loopback-device) device, a
[TUN/TAP](/posts/building-a-tuntap-device) device, an
[Ethernet codec, and a `net.Registry`](/posts/building-ethernet-layer) that
wires N devices to M `ProtocolHandler`s by EtherType. So far the stack can
read raw bytes off a wire, decode the 14-byte header, and ship a `Frame`
into a channel — but it cannot actually *do* anything with the result. There
are no handlers yet.

This post implements the first one. ARP — the Address Resolution Protocol,
specified by [RFC 826](https://datatracker.ietf.org/doc/html/rfc826) — is
the link-layer protocol that turns an IP address into a MAC address. Without
it, the IPv4 layer that comes next has no way to fill in the destination MAC
of an outgoing Ethernet frame. ARP is the prerequisite for every other
protocol we will implement.

The code lives at [rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack),
in `pkg/arp`.

```
 ┌─────────────────────────────────────────┐
 │   Application (HTTP…)                   │
 ├─────────────────────────────────────────┤
 │   Transport (TCP/UDP)                   │
 ├─────────────────────────────────────────┤
 │   Network (IP)                          │
 ├─────────────────────────────────────────┤
 │   Link (Ethernet) + ARP ◄───────────────│  ← this post
 ├─────────────────────────────────────────┤
 │   Device (loopback / TAP)               │
 └─────────────────────────────────────────┘
```

ARP is a strange protocol structurally. It lives at the link layer — its
packets are carried directly inside Ethernet frames, not on top of IP — but
it serves the layer above it. It is both a *user* of Ethernet and a
*dependency* of IP. That awkward positioning is the reason it gets its own
package alongside `pkg/ethernet`, rather than living inside `pkg/ip`.

---

## What is ARP?

When a host wants to send an IPv4 packet to another host on the same
network, it knows the destination IP address — that is what the caller
passed to the socket API. But the link layer underneath does not speak IP.
Ethernet only knows about MAC addresses. To put the IP packet on the wire,
the stack needs to know: *which MAC address corresponds to this IP?*

ARP answers that question with a two-packet exchange:

```
   Host A  (192.168.1.10, aa:aa:aa:aa:aa:aa)
       │
       │  1.  Broadcast ARP REQUEST:
       │      "Who has 192.168.1.20? Tell 192.168.1.10."
       │      Ethernet dst: ff:ff:ff:ff:ff:ff (broadcast)
       ▼
   ┌─────────────────────────────────────────────────────┐
   │           Local Ethernet segment                    │
   │           (every host sees the request)             │
   └─────────────────────────────────────────────────────┘
       ▲
       │  2.  Unicast ARP REPLY:
       │      "192.168.1.20 is at bb:bb:bb:bb:bb:bb"
       │      Ethernet dst: aa:aa:aa:aa:aa:aa (back to A)
       │
   Host B  (192.168.1.20, bb:bb:bb:bb:bb:bb)
```

Two structural properties define ARP:

1. The **request** is a broadcast Ethernet frame
   (`ff:ff:ff:ff:ff:ff`). Every host on the segment receives it, but only
   the host whose IP matches the target IP responds.
2. The **reply** is a unicast Ethernet frame sent directly back to the
   requester. Nobody else on the segment needs to see it.

This is why ARP only works on a single link-layer broadcast domain — a LAN,
a Wi-Fi network, a VLAN. The moment a router is in the path, ARP stops at
the router's interface. The router has its own MAC, replies with that, and
forwards the IP packet using its own routing logic.

### Why the cache exists

A naive ARP implementation would issue a fresh request before every IP
packet. That would multiply traffic by a factor of three (request + reply +
data) and add a full round-trip of latency to every send. Worse, broadcast
frames are processed by every host on the segment — so naive ARP would
scale linearly with both packet rate *and* segment size.

The fix is the **ARP cache**: a local map from IP to MAC, populated on every
exchange, with a finite TTL so stale entries eventually clear. Linux's
default TTL is 20 minutes. Ours matches.

### The packet format

ARP packets are flexible by design — the protocol was specified to carry
any combination of hardware and protocol address. In practice, every modern
network uses exactly one combination: Ethernet (hardware type 1) and IPv4
(protocol type `0x0800`). With those choices, the packet has a fixed 28-byte
layout:

```
 ┌────────────────────────────────────────────────────────────────────┐
 │                     ARP packet (RFC 826)                           │
 ├──────────────────────┬────────────────────────────────────────────┤
 │ Hardware type    (2) │  = 1  (Ethernet)                            │
 │ Protocol type    (2) │  = 0x0800  (IPv4)                           │
 │ HW addr len      (1) │  = 6                                        │
 │ Proto addr len   (1) │  = 4                                        │
 │ Operation        (2) │  = 1 (request) or 2 (reply)                 │
 │ Sender MAC       (6) │  the sender's hardware address              │
 │ Sender IP        (4) │  the sender's IP address                    │
 │ Target MAC       (6) │  request: all zeros; reply: requester's MAC │
 │ Target IP        (4) │  the IP being resolved                      │
 └──────────────────────┴────────────────────────────────────────────┘
                          Total: 28 bytes
```

Every field is fixed-size, which is what makes the packet zero-copy parseable.

---

## Exploring ARP from the command line

Before writing a single line of Go, it pays to spend time poking at the
kernel's ARP machinery. Linux exposes a rich set of tools — between `ip
neigh`, `arping`, `tcpdump`, and `scapy`, you can drive every code path our
implementation will need to handle.

### Read the ARP table

The kernel keeps the active ARP cache (technically the *neighbor table*) in
its routing subsystem. The modern way to read it is `ip neigh`:

```bash
# Full neighbor table — IPv4 ARP and IPv6 NDP together
ip neigh show

# IPv4 only (ARP entries)
ip -4 neigh show

# Only entries learned via a specific interface
ip neigh show dev eth0
```

```
192.168.1.1 dev eth0 lladdr aa:bb:cc:00:11:22 REACHABLE
192.168.1.5 dev eth0 lladdr de:ad:be:ef:ca:fe STALE
192.168.1.7 dev eth0  FAILED
```

The state column is what most casual users overlook — it encodes the
kernel's confidence in the entry:

| State        | Meaning                                                            |
|--------------|--------------------------------------------------------------------|
| `REACHABLE`  | Recently confirmed via traffic or an explicit reply. Trustworthy. |
| `STALE`      | Has not been used recently. Will be re-probed before next use.    |
| `DELAY`      | Pending re-probe — about to send a request.                       |
| `PROBE`      | A unicast probe has been sent; waiting for a reply.               |
| `FAILED`     | Probe timed out — host considered unreachable.                    |
| `PERMANENT`  | Manually added; never expires.                                    |

Our `Cache` is much simpler — there is only "valid" and "expired" — but
the Linux states are worth knowing because they show up in `tcpdump`
patterns: a `STALE` entry that transitions to `DELAY` means you are about
to see an ARP probe on the wire.

The legacy tool still works:

```bash
# Classic arp — same data, slightly different formatting
arp -n            # numeric addresses (no DNS lookup)
arp -e            # extended Linux format
arp -i eth0 -n    # filter by interface

# Look up the MAC for a specific IP without touching the kernel cache
getent hosts 192.168.1.5
```

### Add and remove entries

The kernel cache is writable, which is handy when you want to take ARP out
of the loop for a test:

```bash
# Add a permanent entry — survives forever, no GC
sudo ip neigh add 192.168.1.5 lladdr de:ad:be:ef:ca:fe dev eth0 nud permanent

# Replace whatever exists with a new value
sudo ip neigh replace 192.168.1.5 lladdr 11:22:33:44:55:66 dev eth0

# Remove a single entry
sudo ip neigh del 192.168.1.5 dev eth0

# Flush the entire neighbor table for one interface (forces re-resolution)
sudo ip neigh flush dev eth0

# Flush everything everywhere
sudo ip neigh flush all
```

`ip neigh flush dev tap0` is the single most useful command when testing
our handler. Run it between iterations to force the kernel to re-issue an
ARP request that our stack must answer — without flushing, the kernel will
silently use its cached entry and you will think the handler is broken when
it is actually never being asked.

### Trigger an ARP request

The most direct way to make an ARP request happen is `arping`:

```bash
# Basic — one request per second, prints replies as they arrive
sudo arping -I eth0 -c 4 192.168.1.5

# Source-spoofed — useful when our stack has not yet claimed an IP
sudo arping -I eth0 -S 192.168.1.99 -c 1 192.168.1.5

# Unsolicited (gratuitous) ARP — announces our mapping to the network
sudo arping -I eth0 -U -c 1 192.168.1.99

# Duplicate-address detection probe — does anyone else claim this IP?
sudo arping -I eth0 -D -c 1 192.168.1.99
```

`arping -D` is what DHCP clients use right after they get a lease but
before they start using the address: send an ARP request *for our own
proposed IP*, with sender IP set to `0.0.0.0`. If anyone replies, the
address is taken.

Gratuitous ARP (`-U`) is a request where the sender and target IPs are the
same — it carries no question, just an announcement. Hosts use it to
pre-populate each other's caches after a failover, so the new host's MAC
takes over without anyone needing to wait for a TTL expiry.

### Watch ARP on the wire

`tcpdump` filtered to `arp` shows every request and reply on the segment,
with `-e` adding the Ethernet header (which tells you whether the frame
was broadcast or unicast):

```bash
# Show ARP only, with Ethernet headers
sudo tcpdump -i eth0 -n -e arp

# Hex dump — every byte of the 14-byte Ethernet header + 28-byte ARP packet
sudo tcpdump -i eth0 -n -e -X arp

# Watch ARP for a specific host
sudo tcpdump -i eth0 -n arp host 192.168.1.5

# Save to a pcap for Wireshark
sudo tcpdump -i eth0 -w /tmp/arp.pcap arp
```

A typical request/reply pair looks like this in `tcpdump -e` output:

```
ff:ff:ff:ff:ff:ff > aa:aa:aa:aa:aa:aa, ethertype ARP (0x0806), length 42:
    Request who-has 192.168.1.20 tell 192.168.1.10, length 28
aa:aa:aa:aa:aa:aa > bb:bb:bb:bb:bb:bb, ethertype ARP (0x0806), length 42:
    Reply 192.168.1.20 is-at bb:bb:bb:bb:bb:bb, length 28
```

Length 42 is the smallest legal Ethernet frame size *as captured by libpcap
on Linux*: 14-byte header + 28-byte ARP packet. On the actual wire,
hardware pads to 60 bytes minimum (the so-called "runt frame" lower bound),
but the pad is stripped before `tcpdump` sees it.

### Craft a custom ARP packet

For integration testing, `scapy` is unbeatable — Python with a network
DSL, and it speaks raw frames over a TAP interface:

```bash
# Send a request from a fake source MAC for an arbitrary target IP
sudo python3 -c "
from scapy.all import *
sendp(Ether(dst='ff:ff:ff:ff:ff:ff', src='aa:bb:cc:dd:ee:01') /
      ARP(hwsrc='aa:bb:cc:dd:ee:01', psrc='192.168.100.2',
          pdst='192.168.100.1', op=1),
      iface='tap0')
"
```

This is the recipe for driving our handler: bring up `tap0`, register our
handler with the Registry, point a `scapy` request at it, and verify with
`tcpdump -i tap0 -e arp` that our reply lands on the wire correctly
formed.

### Inspect kernel ARP tuning

Most of the kernel's ARP behaviour is controlled by sysctls in
`/proc/sys/net/ipv4/neigh/<iface>/`. The ones worth knowing:

```bash
# Default TTL for resolved entries (in seconds)
cat /proc/sys/net/ipv4/neigh/eth0/base_reachable_time

# How long a STALE entry lives before being garbage-collected
cat /proc/sys/net/ipv4/neigh/eth0/gc_stale_time

# How many failed probes before declaring FAILED
cat /proc/sys/net/ipv4/neigh/eth0/ucast_solicit

# Number of broadcast probes before giving up
cat /proc/sys/net/ipv4/neigh/eth0/mcast_solicit

# Hard limit on the table size (per-interface)
cat /proc/sys/net/ipv4/neigh/default/gc_thresh1   # below this, no GC
cat /proc/sys/net/ipv4/neigh/default/gc_thresh2   # soft cap
cat /proc/sys/net/ipv4/neigh/default/gc_thresh3   # hard cap
```

On a busy server, hitting `gc_thresh3` (default 1024) is the most common
ARP-related production failure. Once the table is full, new entries cannot
be added — `Network unreachable` errors start appearing for hosts the
kernel cannot resolve. Our cache has no hard cap; it grows with the working
set, and stale entries are evicted by TTL alone.

---

## The implementation

The package is three files: `arp.go` for the wire codec, `cache.go` for
the TTL map, and `handler.go` for the protocol logic. We will walk through
them in that order — codec first, then state, then the goroutine that ties
them together.

### `Packet` — the 28-byte struct

```go
type Operation uint16

const (
    OperationRequest Operation = 1
    OperationReply   Operation = 2
)

type Packet struct {
    Operation Operation
    SenderMAC ethernet.Addr
    SenderIP  [4]byte
    TargetMAC ethernet.Addr
    TargetIP  [4]byte
}
```

Five fields. Every one is fixed-size:
[`ethernet.Addr`](/posts/building-ethernet-layer) is a `[6]byte` array, the
IPs are `[4]byte` arrays, and `Operation` is a `uint16`. The struct is
therefore *itself* a comparable value type — `p1 == p2` works without
`reflect.DeepEqual`, which is what makes the round-trip test in the
suite a one-line assertion.

Notably absent: hardware type, protocol type, hardware address length,
protocol address length. The wire format carries them, but they only ever
take one value combination in this stack. Hard-coding them at the codec
boundary means callers cannot accidentally produce malformed packets.

### `Parse` — zero-copy, strict on type

```go
const (
    hwTypeEthernet uint16 = 1
    protoTypeIPv4  uint16 = 0x0800
    hwAddrLen      uint8  = 6
    protoAddrLen   uint8  = 4
    PacketLen             = 28
)

func Parse(b []byte) (Packet, error) {
    if len(b) < PacketLen {
        return Packet{}, ErrPacketTooShort
    }
    if binary.BigEndian.Uint16(b[0:2]) != hwTypeEthernet {
        return Packet{}, ErrUnsupportedHardwareType
    }
    if binary.BigEndian.Uint16(b[2:4]) != protoTypeIPv4 {
        return Packet{}, ErrUnsupportedProtocolType
    }

    var p Packet
    p.Operation = Operation(binary.BigEndian.Uint16(b[6:8]))
    copy(p.SenderMAC[:], b[8:14])
    copy(p.SenderIP[:],  b[14:18])
    copy(p.TargetMAC[:], b[18:24])
    copy(p.TargetIP[:],  b[24:28])
    return p, nil
}
```

The function allocates nothing. `Packet` is a 28-byte struct returned on the
stack. The two `binary.BigEndian.Uint16` reads compile to a load plus a
byte-swap; the four `copy` calls compile to fixed-size memory moves.

Three distinct sentinel errors:

```go
var (
    ErrPacketTooShort          = errors.New("arp: packet too short")
    ErrUnsupportedHardwareType = errors.New("arp: unsupported hardware type (want Ethernet)")
    ErrUnsupportedProtocolType = errors.New("arp: unsupported protocol type (want IPv4)")
)
```

Discriminating these matters because they tell the caller different things:
"this is not an ARP packet at all" versus "this is ARP but not the flavor
we speak." A future IPv6 NDP implementation would log unsupported-protocol
drops as informational; truncation drops are usually worth a warning.

Note what `Parse` does *not* validate: it does not check
`b[4] == hwAddrLen` or `b[5] == protoAddrLen`. The packet length is fixed
at 28 bytes, and the address-length fields are positionally determined by
the hardware-type and protocol-type fields we already validated. A 28-byte
ARP packet with hardware type 1 and protocol type `0x0800` cannot have
mismatched length bytes without violating one of the earlier checks first.
Re-checking them would be defensive code with no failure mode it could
catch.

### `Marshal` — the inverse, equally strict

```go
func Marshal(dst []byte, p Packet) error {
    if len(dst) < PacketLen {
        return ErrPacketTooShort
    }
    binary.BigEndian.PutUint16(dst[0:2], hwTypeEthernet)
    binary.BigEndian.PutUint16(dst[2:4], protoTypeIPv4)
    dst[4] = hwAddrLen
    dst[5] = protoAddrLen
    binary.BigEndian.PutUint16(dst[6:8], uint16(p.Operation))
    copy(dst[8:14],  p.SenderMAC[:])
    copy(dst[14:18], p.SenderIP[:])
    copy(dst[18:24], p.TargetMAC[:])
    copy(dst[24:28], p.TargetIP[:])
    return nil
}
```

`Marshal` writes into a caller-supplied buffer and returns no length —
because the length is always 28. The signature returns only `error`, and
the only error it can return is `ErrPacketTooShort` (for the destination,
not the source). Hard-coding the constant fields here is what keeps the
caller from having to know that the hardware type field exists.

### `Cache` — TTL map with a background GC

ARP needs persistent state: the resolved entries we have seen. The cache is
a TTL map with the standard pattern of a background goroutine that sweeps
expired entries:

```go
const (
    defaultTTL        = 20 * time.Minute  // matches Linux's base_reachable_time
    defaultGCInterval = time.Minute
)

type Entry struct {
    MAC     ethernet.Addr
    Expires time.Time
}

type Cache struct {
    mu         sync.RWMutex
    entries    map[[4]byte]Entry
    ttl        time.Duration
    gcInterval time.Duration
}
```

The key is `[4]byte` — the IPv4 address — and the choice is identical to
the choice we made for `ethernet.Addr` in the previous post: a fixed-size
array, not a slice, because Go map keys must be comparable and an array of
bytes is the most compact comparable type available.

```go
func (c *Cache) Lookup(ip [4]byte) (ethernet.Addr, bool) {
    c.mu.RLock()
    e, ok := c.entries[ip]
    c.mu.RUnlock()
    if !ok || time.Now().After(e.Expires) {
        return ethernet.Addr{}, false
    }
    return e.MAC, true
}

func (c *Cache) Store(ip [4]byte, mac ethernet.Addr) {
    c.mu.Lock()
    c.entries[ip] = Entry{MAC: mac, Expires: time.Now().Add(c.ttl)}
    c.mu.Unlock()
}
```

`sync.RWMutex` over `sync.Mutex` because reads dominate writes by orders
of magnitude — every outgoing IP packet results in a `Lookup`, but only
ARP traffic and a freshly-resolved entry produce a `Store`. The read lock
is held only for the map access; the expiry comparison happens after
unlocking, so a slow `time.Now()` does not block other readers.

A subtle property: `Lookup` does not delete expired entries. It just
reports them as missing. The actual deletion happens in the GC goroutine.
This means a `Lookup` immediately followed by a `Store` for the same IP is
not racy — the `Store` either updates the existing entry (refreshing
`Expires`) or inserts a new one over a stale one. Either way the result is
correct, with no special-cased expiry path.

### The GC goroutine

```go
func (c *Cache) Start(ctx context.Context) {
    tick := time.NewTicker(c.gcInterval)
    defer tick.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-tick.C:
            c.gc()
        }
    }
}

func (c *Cache) gc() {
    now := time.Now()
    c.mu.Lock()
    for ip, e := range c.entries {
        if now.After(e.Expires) {
            delete(c.entries, ip)
        }
    }
    c.mu.Unlock()
}
```

Once a minute (the default), the goroutine wakes up, takes the write lock,
and deletes expired entries. The full sweep is intentional: a sorted-by-
expiry priority queue would be marginally more efficient at very large
sizes, but it would multiply the lock complexity. At ARP cache scales —
tens to a few thousand entries — the linear scan finishes in microseconds.

`Start` is blocking by contract. The caller runs it in a goroutine. We do
not return until `ctx` is cancelled. This pattern lets the parent
(`Handler.Start`) wait on the GC goroutine's exit with a `sync.WaitGroup`,
which is what makes `goleak.VerifyNone` pass cleanly across the whole
handler lifecycle.

### `Handler` — the `net.ProtocolHandler`

The Handler is what `pkg/net.Registry` invokes when it sees an EtherType
`0x0806` frame:

```go
type Handler struct {
    localMAC ethernet.Addr
    localIP  [4]byte
    dev      netpkg.LinkDevice
    cache    *Cache
    rxCh     chan netpkg.Frame

    mu      sync.Mutex
    pending map[[4]byte][]chan ethernet.Addr // waiters keyed by target IP
}
```

Seven fields, but only three of them are state machinery. `localMAC` and
`localIP` identify us — they are what we put in outgoing replies and what
we compare incoming targets against. `dev` is the device we transmit on
(needed because we send packets ourselves, not just receive them). `cache`
is the map. `rxCh` is the channel the Registry pushes frames into.

The last two — `mu` and `pending` — implement the **shared in-flight
request** optimisation, which we will get to in a moment.

```go
func (h *Handler) EtherType() ethernet.EtherType { return ethernet.EtherTypeARP }
func (h *Handler) RxChan() chan netpkg.Frame     { return h.rxCh }
```

These two methods satisfy `net.ProtocolHandler`. They are the contract
boundary: the Registry calls `EtherType()` once during `AddHandler` to key
the dispatch map, and `RxChan()` once per dispatched frame to push.
Everything else happens inside the handler.

### The receive loop

```go
func (h *Handler) Start(ctx context.Context, errCh chan<- error) {
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        defer wg.Done()
        h.cache.Start(ctx)
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
            if err := h.handle(frame); err != nil {
                _ = err // malformed packets are dropped, not fatal
            }
        }
    }
}
```

`Start` launches the cache GC goroutine, then loops on the receive channel
until shutdown. The `defer wg.Wait()` is what makes the shutdown
deterministic: when `ctx` is cancelled, both this loop *and* the cache
goroutine return, and `Start` does not exit until the cache goroutine is
fully joined. That ordering is what `TestHandler_Shutdown_NoLeak` checks.

Why are malformed packets dropped silently? Because the alternative is
either (a) write a log line per malformed packet, which a hostile sender
can use to fill the logs, or (b) push the error into `errCh`, which is for
fatal errors. ARP is best-effort by design — duplicate-detection probes,
gratuitous announcements, and stray traffic from misconfigured hosts all
produce packets we cannot or should not respond to. Dropping them is the
correct behaviour, not a code smell.

### `handle` — RFC 826 §6, then the operation switch

```go
func (h *Handler) handle(frame netpkg.Frame) error {
    p, err := Parse(frame.Payload)
    if err != nil {
        return err
    }

    // RFC 826 §6: always merge sender info into the cache.
    h.cache.Store(p.SenderIP, p.SenderMAC)

    switch p.Operation {
    case OperationRequest:
        if p.TargetIP != h.localIP {
            return nil
        }
        reply := Packet{
            Operation: OperationReply,
            SenderMAC: h.localMAC,
            SenderIP:  h.localIP,
            TargetMAC: p.SenderMAC,
            TargetIP:  p.SenderIP,
        }
        if err := h.sendPacket(p.SenderMAC, reply); err != nil {
            return fmt.Errorf("arp: send reply: %w", err)
        }

    case OperationReply:
        h.notifyWaiters(p.SenderIP, p.SenderMAC)
    }
    return nil
}
```

This is the heart of the handler, and it deserves a careful read because
two RFC 826 details are easy to miss:

**The cache update happens before the operation switch.** RFC 826 §6 is
explicit: "If the pair (sender protocol address, sender hardware address)
is not in the table, add it." Whether the packet is a request or a reply,
the *sender's* mapping is always valid information. So every incoming ARP
packet — even one targeting a different host — passively teaches our
cache about the sender. This means that on a busy LAN, our cache fills up
without us ever having to send a request.

**Requests not addressed to us are silently ignored after the merge.**
Returning `nil` rather than an error: the request was well-formed, we
processed the part we cared about (the cache update), and we have no reply
to send because the target IP is not ours. This is not a failure mode.

The reply branch calls `notifyWaiters` — which is the second half of the
Resolve flow.

### `Resolve` — the API the IP layer will call

`Resolve` is the externally visible API. Given a target IP, return the
target MAC, blocking until ARP succeeds or the context expires:

```go
func (h *Handler) Resolve(ctx context.Context, targetIP [4]byte) (ethernet.Addr, error) {
    if mac, ok := h.cache.Lookup(targetIP); ok {
        return mac, nil
    }

    // Slow path: register as a waiter, send a broadcast if first.
    ch := make(chan ethernet.Addr, 1)
    first := h.registerWaiter(targetIP, ch)
    if first {
        req := Packet{
            Operation: OperationRequest,
            SenderMAC: h.localMAC,
            SenderIP:  h.localIP,
            TargetMAC: ethernet.Addr{}, // unknown
            TargetIP:  targetIP,
        }
        if err := h.sendPacket(ethernet.Broadcast, req); err != nil {
            h.removeWaiter(targetIP, ch)
            return ethernet.Addr{}, fmt.Errorf("arp: send request: %w", err)
        }
    }

    select {
    case mac := <-ch:
        return mac, nil
    case <-ctx.Done():
        h.removeWaiter(targetIP, ch)
        return ethernet.Addr{}, ErrResolveFailed
    }
}
```

The fast path is one map lookup and one return. No locks beyond the
cache's `RWMutex`, no allocations, no syscalls. For cached entries this
takes a few dozen nanoseconds.

The slow path is the interesting one. Imagine ten goroutines simultaneously
call `Resolve(ctx, ipB)` on a cold cache. A naive implementation would have
each goroutine broadcast its own ARP request — ten identical broadcasts
hitting the segment, ten copies of the reply, and ten wasted ARP exchanges.
The waiter map prevents this: the first caller registers, sees `first ==
true`, and sends the broadcast. The other nine register, see `first ==
false`, and just block on their channel. When the reply arrives, every
waiter is unblocked with the same MAC.

The implementation is three short methods:

```go
func (h *Handler) registerWaiter(targetIP [4]byte, ch chan ethernet.Addr) bool {
    h.mu.Lock()
    defer h.mu.Unlock()
    first := len(h.pending[targetIP]) == 0
    h.pending[targetIP] = append(h.pending[targetIP], ch)
    return first
}

func (h *Handler) notifyWaiters(targetIP [4]byte, mac ethernet.Addr) {
    h.mu.Lock()
    waiters := h.pending[targetIP]
    delete(h.pending, targetIP)
    h.mu.Unlock()
    for _, ch := range waiters {
        ch <- mac
    }
}

func (h *Handler) removeWaiter(targetIP [4]byte, ch chan ethernet.Addr) {
    h.mu.Lock()
    defer h.mu.Unlock()
    waiters := h.pending[targetIP]
    for i, w := range waiters {
        if w == ch {
            h.pending[targetIP] = append(waiters[:i], waiters[i+1:]...)
            break
        }
    }
    if len(h.pending[targetIP]) == 0 {
        delete(h.pending, targetIP)
    }
}
```

Every waiter channel has capacity 1, which is what makes `notifyWaiters`
non-blocking even though it sends from inside a critical section above. We
release the lock before the sends only to keep the section as short as
possible — the sends themselves cannot block because each receiver is
either already blocked in `select`, or no longer cares.

`removeWaiter` is the timeout path: if `ctx.Done()` fires before a reply
arrives, the goroutine surgically removes its channel from the pending
list. Without this, a timed-out resolution would leave a stale channel in
the map; when the (delayed) reply eventually arrived, `notifyWaiters` would
send to a channel nobody was reading from, blocking the handler's main
loop forever.

### `sendPacket` — the ARP-over-Ethernet wrapper

```go
func (h *Handler) sendPacket(dstMAC ethernet.Addr, p Packet) error {
    buf := make([]byte, ethernet.HeaderLen+PacketLen)
    if err := Marshal(buf[ethernet.HeaderLen:], p); err != nil {
        return err
    }
    hdr := ethernet.Header{
        Dst:       dstMAC,
        Src:       h.localMAC,
        EtherType: ethernet.EtherTypeARP,
    }
    if _, err := ethernet.Marshal(buf, hdr, buf[ethernet.HeaderLen:]); err != nil {
        return err
    }
    _, err := h.dev.Write(buf)
    return err
}
```

The function allocates one 42-byte buffer (14 Ethernet header + 28 ARP),
writes the ARP packet at offset 14, then asks `ethernet.Marshal` to fill
in the 14-byte header in-place. The destination MAC is what differentiates
the two call sites: `Resolve` calls `sendPacket(ethernet.Broadcast, req)`
to broadcast the question, and `handle` calls
`sendPacket(p.SenderMAC, reply)` to send the answer back to exactly the
host that asked.

The one allocation per outgoing packet is acceptable because the outgoing
rate is bounded. ARP is *slow* by design — at most one request per IP per
TTL, usually far less. If profiling ever shows this allocation mattering,
a `sync.Pool` of 42-byte buffers wraps the call site without changing the
shape of the code.

---

## The tests

Twenty-one tests, split into three groups that mirror the three files.

```bash
# Plain run
go test ./pkg/arp/... -v

# With the race detector
CGO_ENABLED=1 go test ./pkg/arp/... -race -v
```

The interesting tests are not the codec round-trips — those are
mechanical — but the handler tests, which involve a `spyDevice`, injected
frames, and goroutines coordinating through channels. The whole suite runs
in just over a second.

### A clean run

```
=== RUN   TestParse_Valid
--- PASS: TestParse_Valid (0.00s)
=== RUN   TestParse_TooShort
--- PASS: TestParse_TooShort (0.00s)
=== RUN   TestParse_WrongHardwareType
--- PASS: TestParse_WrongHardwareType (0.00s)
=== RUN   TestParse_WrongProtocolType
--- PASS: TestParse_WrongProtocolType (0.00s)
=== RUN   TestMarshal_RoundTrip
--- PASS: TestMarshal_RoundTrip (0.00s)
=== RUN   TestMarshal_TooSmall
--- PASS: TestMarshal_TooSmall (0.00s)
=== RUN   TestCache_StoreAndLookup
--- PASS: TestCache_StoreAndLookup (0.00s)
=== RUN   TestCache_LookupMiss
--- PASS: TestCache_LookupMiss (0.00s)
=== RUN   TestCache_TTLExpiry
--- PASS: TestCache_TTLExpiry (0.02s)
=== RUN   TestCache_Delete
--- PASS: TestCache_Delete (0.00s)
=== RUN   TestCache_GC_RemovesExpiredEntries
--- PASS: TestCache_GC_RemovesExpiredEntries (0.05s)
=== RUN   TestCache_ConcurrentAccess
--- PASS: TestCache_ConcurrentAccess (0.00s)
=== RUN   TestHandler_RepliesTo_Request
--- PASS: TestHandler_RepliesTo_Request (0.00s)
=== RUN   TestHandler_Ignores_Request_NotForUs
--- PASS: TestHandler_Ignores_Request_NotForUs (0.02s)
=== RUN   TestHandler_UpdatesCache_OnRequest
--- PASS: TestHandler_UpdatesCache_OnRequest (0.00s)
=== RUN   TestHandler_UpdatesCache_OnReply
--- PASS: TestHandler_UpdatesCache_OnReply (0.02s)
=== RUN   TestHandler_Resolve_CacheHit
--- PASS: TestHandler_Resolve_CacheHit (0.00s)
=== RUN   TestHandler_Resolve_CacheMiss
--- PASS: TestHandler_Resolve_CacheMiss (0.00s)
=== RUN   TestHandler_Resolve_MultipleWaiters
--- PASS: TestHandler_Resolve_MultipleWaiters (0.00s)
=== RUN   TestHandler_Resolve_Timeout
--- PASS: TestHandler_Resolve_Timeout (0.05s)
=== RUN   TestHandler_Shutdown_NoLeak
--- PASS: TestHandler_Shutdown_NoLeak (0.00s)
PASS
ok      github.com/rykth/tcp-ip-stack/pkg/arp   1.179s
```

### The codec tests

Six tests cover `Parse` and `Marshal`:

- `TestParse_Valid` builds a packet via `Marshal`, decodes it via `Parse`,
  and checks every field. The mechanical correctness check.
- `TestParse_TooShort` confirms that a 27-byte buffer returns
  `ErrPacketTooShort`.
- `TestParse_WrongHardwareType` sets bytes 0–1 to `0x0006` (IEEE 802.2)
  and expects `ErrUnsupportedHardwareType`.
- `TestParse_WrongProtocolType` sets bytes 2–3 to `0x86DD` (IPv6) and
  expects `ErrUnsupportedProtocolType`.
- `TestMarshal_RoundTrip` runs two packets (one request, one reply) through
  `Marshal → Parse` and uses `got != want` directly — proving the
  `Packet` struct's value-type equality is wired up.
- `TestMarshal_TooSmall` confirms that marshalling into a 27-byte buffer
  returns `ErrPacketTooShort`.

The interesting one is `TestParse_WrongHardwareType`. ARP's wire format
permits any hardware type — Ethernet (1), 802.11 (6), Token Ring (4),
many others. Our `Parse` rejects everything except Ethernet, and the test
documents that with an explicit non-Ethernet value. If the codec ever drifts
toward leniency (say, accepting any hardware type and just propagating the
field), this test breaks first.

### The cache tests

Six tests cover the `Cache`:

- `TestCache_StoreAndLookup` — basic insert + read.
- `TestCache_LookupMiss` — unknown IP returns `(_, false)`.
- `TestCache_TTLExpiry` — set TTL to 10 ms, store, sleep 20 ms, lookup
  must miss.
- `TestCache_Delete` — explicit delete works.
- `TestCache_GC_RemovesExpiredEntries` — the most involved cache test:
  start the GC goroutine with a 10 ms tick and a 5 ms TTL, store one entry,
  sleep 50 ms, cancel ctx, wait for GC to exit, then verify the entry is
  gone.
- `TestCache_ConcurrentAccess` — 20 goroutines storing and 20 looking up
  the same key simultaneously. Run with `-race`, any unsynchronised map
  access is caught immediately.

The TTL and GC tests use timing-based assertions. That is usually a code
smell, but here it is justified: TTL semantics are inherently temporal,
and using a 5–10 ms TTL keeps the suite fast while leaving enough slack
that scheduler jitter does not cause flakes.

### The handler tests

Nine tests, each isolating one behaviour. They share a `spyDevice` fixture
that records every `Write` and exposes `waitWrite(t, n)` to block until at
least `n` frames have been recorded — which is what makes the tests
deterministic despite all the goroutine activity:

```go
type spyDevice struct {
    mu      sync.Mutex
    writes  [][]byte
    written chan struct{} // notified non-blocking on each Write
    ...
}
```

**`TestHandler_RepliesTo_Request`** — the canonical positive case. Inject
a request from `(macB, ipB)` asking for `ipA`. Wait for one device write.
Parse what came out: it must be an Ethernet frame with EtherType
`0x0806`, and the ARP packet inside must be an
`OperationReply` with `(SenderMAC=macA, SenderIP=ipA, TargetMAC=macB,
TargetIP=ipB)`. This test alone covers about half the protocol.

**`TestHandler_Ignores_Request_NotForUs`** — send a request asking for
`ipC` (an IP we have not registered). Sleep 20 ms. Assert the spy device
recorded *zero* writes. This is the test that pins the
`p.TargetIP != h.localIP` check; without it the handler would broadcast
replies for IPs it does not own, polluting the segment.

**`TestHandler_UpdatesCache_OnRequest`** and
**`TestHandler_UpdatesCache_OnReply`** — both inject a frame from `(macB,
ipB)`, then assert that `cache.Lookup(ipB) == macB`. This is the RFC 826 §6
property: cache updates happen *regardless* of operation. The test
explicitly proves that reply frames update the cache even though we send no
outgoing traffic in response.

**`TestHandler_Resolve_CacheHit`** — pre-populate the cache, call
`Resolve`, assert that (a) the right MAC comes back and (b) no packets
were broadcast. The fast path must be free of side effects.

**`TestHandler_Resolve_CacheMiss`** is the round-trip test, and it is the
clearest demonstration of how the protocol composes:

```go
// 1. Start the resolve in a goroutine.
go func() {
    mac, err := h.Resolve(ctx, ipB)
    resultCh <- result{mac, err}
}()

// 2. Wait for the broadcast request to land on the spy device.
dev.waitWrite(t, 1)

// 3. Parse it and verify it's the right shape.
ethFrame, _ := ethernet.Parse(dev.Recorded()[0])
req, _ := arp.Parse(ethFrame.Payload)
if req.Operation != arp.OperationRequest { t.Errorf(...) }
if req.TargetIP != ipB                   { t.Errorf(...) }
if ethFrame.Dst != ethernet.Broadcast    { t.Errorf(...) }

// 4. Simulate the remote host replying.
h.RxChan() <- buildReplyFrame(macB, ipB, macA, ipA)

// 5. Resolve unblocks with the right MAC.
r := <-resultCh
```

Five distinct synchronisation points, all deterministic, no `time.Sleep`
for correctness (only for negative assertions where we *want* to give the
handler time to do nothing).

**`TestHandler_Resolve_MultipleWaiters`** — the shared-in-flight test that
the waiter map exists to make work. Five goroutines call `Resolve(ipB)`
simultaneously. The test waits for *one* broadcast write, then injects
*one* reply, then verifies all five goroutines receive the same MAC. The
final assertion `len(dev.Recorded()) == 1` is what closes the loop:
exactly one broadcast was sent, not five.

**`TestHandler_Resolve_Timeout`** — call `Resolve` with a 50 ms timeout
and never reply. Must return `ErrResolveFailed`. The `removeWaiter` call
in the timeout path is what keeps this test from leaking goroutines on
shutdown.

**`TestHandler_Shutdown_NoLeak`** — start the handler, cancel the context,
assert `Start` returns within 2 seconds, assert no goroutines remain via
`goleak`. The cache GC goroutine and the receive loop must both exit
cleanly; the `defer wg.Wait()` in `Start` is what makes this test pass.

---

## Driving it on a real TAP device

The same pattern as the previous posts: bring up a TAP, wire up the
Registry, send a real ARP packet, watch the wire.

In one terminal, create the device and launch the stack:

```bash
sudo ip tuntap add dev tap0 mode tap user $USER
sudo ip link set tap0 up
sudo ip addr add 192.168.100.1/24 dev tap0
go run ./examples/arp   # ARP handler at 192.168.100.2 with macA
```

A minimal driver looks like this:

```go
package main

import (
    "context"
    "log"

    "github.com/rykth/tcp-ip-stack/pkg/arp"
    "github.com/rykth/tcp-ip-stack/pkg/ethernet"
    netpkg "github.com/rykth/tcp-ip-stack/pkg/net"
    "github.com/rykth/tcp-ip-stack/pkg/raw/tuntap"
)

func main() {
    dev, err := tuntap.New("tap0", tuntap.DeviceTAP)
    if err != nil {
        log.Fatal(err)
    }
    defer dev.Close()

    localMAC := ethernet.Addr{0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0x01}
    localIP  := [4]byte{192, 168, 100, 2}

    arpH := arp.NewHandler(localMAC, localIP, dev)

    r := netpkg.New()
    if err := r.AddDevice(dev);  err != nil { log.Fatal(err) }
    if err := r.AddHandler(arpH); err != nil { log.Fatal(err) }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    log.Fatal(r.Start(ctx))
}
```

In a second terminal, watch the traffic:

```bash
sudo tcpdump -i tap0 -n -e arp
```

In a third terminal, ask the kernel to resolve `192.168.100.2`:

```bash
# Flush any cached entry first so the kernel has to actually ask
sudo ip neigh flush dev tap0

# Trigger a resolution
ping -c 1 -W 2 192.168.100.2
# or, more directly:
sudo arping -I tap0 -c 1 192.168.100.2
```

`tcpdump` shows the request and our reply:

```
aa:bb:cc:dd:ee:00 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42:
    Request who-has 192.168.100.2 tell 192.168.100.1, length 28
aa:bb:cc:dd:ee:01 > aa:bb:cc:dd:ee:00, ethertype ARP (0x0806), length 42:
    Reply 192.168.100.2 is-at aa:bb:cc:dd:ee:01, length 28
```

And `ip neigh show dev tap0` now reports the kernel has learned our MAC:

```
192.168.100.2 dev tap0 lladdr aa:bb:cc:dd:ee:01 REACHABLE
```

That is the first end-to-end exchange between our userspace stack and the
Linux kernel: a real network protocol, running entirely through code we
wrote, talking to the kernel's neighbor subsystem as a peer.

---

## Design decisions, revisited

### Why hard-code Ethernet/IPv4?

The wire format permits other combinations: IPv4 over Token Ring, IPv6 over
Ethernet (though IPv6 uses NDP, not ARP), MAC-48 over various media. The
codec could be made generic, with the hardware/protocol-type fields
exposed.

We don't. Every other combination is dead in practice — IPv6 uses a
different protocol entirely, and non-Ethernet media types are vanishingly
rare on modern networks. Supporting them in `Parse` would require either
parameterising the `Packet` struct (turning the fixed-size fields into
slices) or accepting partial validation (storing fields whose meaning
varies). Both are worse than the current contract: "we speak Ethernet/IPv4
ARP, full stop."

### Why a separate cache type?

`Cache` is a stand-alone type with its own constructor, options, and tests.
The Handler embeds a pointer to one rather than implementing the map
directly. This separation makes two things possible:

1. **Testing the cache in isolation.** Six of the 21 tests use `Cache`
   directly without ever starting a handler. They would not be possible if
   the cache lived inside the handler.
2. **Substituting the cache.** `WithHandlerCache` lets a test inject a
   pre-seeded cache to drive the cache-hit path deterministically, without
   triggering a network exchange to populate it.

The factoring also gives the cache a clear contract: it is a TTL map and
nothing more. It does not know about packets, frames, ARP, or even
Ethernet (it depends on `ethernet.Addr` as a value type, but only as the
map's value).

### Why the shared-in-flight optimisation?

Five goroutines simultaneously calling `Resolve(ip)` is not a thought
experiment — it is what happens the moment we have a TCP layer with five
parallel `Dial` calls to the same host. Each `Dial` resolves the
destination IP separately, each one needs the MAC, each one calls
`Resolve`. Without deduplication we get five identical broadcasts on the
segment, five replies, five fully wasted protocol exchanges.

The waiter map is twelve lines of code in `registerWaiter` /
`notifyWaiters` and is exhaustively tested by
`TestHandler_Resolve_MultipleWaiters`. It is the kind of optimisation
worth doing early because it changes the shape of the public API in a way
that is impossible to retrofit later — a single `Resolve` call always
appears synchronous to its caller, regardless of how many concurrent
callers exist.

---

## Limitations

| Limitation | Detail |
|------------|--------|
| **Ethernet/IPv4 only** | The codec rejects other hardware and protocol types. IPv6 uses NDP, which is a separate protocol entirely |
| **No proxy ARP** | We do not reply on behalf of other hosts. Useful in bridged setups; not implemented |
| **No gratuitous ARP** | We do not send unsolicited announcements on startup or address change. Standard practice for DHCP clients; out of scope here |
| **No duplicate-address detection** | We do not probe for conflicts before claiming `localIP` |
| **No per-entry retry/probing** | Linux's `STALE`/`DELAY`/`PROBE` state machine refreshes entries before they expire; ours just lets them die at TTL |
| **No hard table cap** | The cache grows with the number of distinct seen IPs. On a quiet LAN this is bounded; on an open segment it is technically a DoS vector |
| **Single device per Handler** | Each `Handler` is bound to one `LinkDevice`. A multi-homed host needs one handler per interface |

The single-device-per-handler limitation is the most likely to be revisited
once the IP layer exists, because a multi-homed router needs ARP on every
interface simultaneously. The fix is not architecturally hard — the
Registry already supports multiple devices — but the per-interface MAC and
IP would need to be associated with each device.

---

## What is next

With ARP in place, the stack now has its first complete protocol
implementation: codec, state, handler, all integrated cleanly with
`pkg/net`'s `ProtocolHandler` contract. The shape of every subsequent
protocol — IPv4, ICMP, UDP, TCP — will follow this template: a `Parse`
and `Marshal` for the wire format, a state object if persistent state is
needed, and a handler goroutine that satisfies `net.ProtocolHandler`.

Next up is IPv4. The receive side parses the 20-byte IP header, verifies
the checksum, and dispatches by protocol number (ICMP, UDP, TCP). The
transmit side fills in the header, computes the checksum, *calls
`arp.Resolve` to learn the destination MAC*, and ships the frame to the
device. That call to `Resolve` is the reason we did ARP first — every
IPv4 send routes through it.

The full source is at [rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack).
