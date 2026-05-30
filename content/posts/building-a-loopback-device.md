---
title: "Building a loopback device in Go"
date: 2026-05-29T10:00:00+00:00
description: "I am building a userspace TCP/IP stack in Go. This post walks through the first piece: an in-process loopback device that lets every higher layer be tested without root privileges, raw sockets, or a real network interface."
tags: [golang, networking, tcpip, loopback]
---

## Where this fits

I am not a networking expert. I am documenting my practical journey of building
a userspace TCP/IP stack in Go, layer by layer, from the bottom up.

The code lives at [rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack).
This post covers `pkg/loopback` - the very first piece: a virtual network
device that talks to itself.

A TCP/IP stack is built in layers. Each layer knows nothing about the layers
above it and makes only one assumption about the layer below: that it can send
and receive raw bytes. Before we can build Ethernet framing, ARP resolution, IP
routing, or TCP segments, we need something to send bytes into and read bytes
out of. That is the loopback device.


![loopback layer](/images/loopback_post_1.png "loopback layer")

---

## What is a loopback?

A loopback device is a virtual network interface that routes traffic back to
the same host that sent it, without ever touching physical hardware or leaving
the kernel's memory. Any frame written to the loopback interface is immediately
readable from the same interface.

On Linux the loopback interface is named `lo` and is assigned `127.0.0.1/8`
for IPv4 and `::1/128` for IPv6. The entire `127.0.0.0/8` subnet is reserved
for loopback traffic by [RFC 5735](https://datatracker.ietf.org/doc/html/rfc5735).

![loopback flow](/images/loopback_post_2.png "loopback flow")

---

## Exploring the kernel loopback

Before writing a single line of Go it is worth understanding what we are
modelling. Every Linux system has a loopback interface. The following commands let you inspect it from every angle.

### Inspect the interface

```bash
ip link show lo
```

Shows the link-layer state of the interface: flags, MTU, and the hardware
address (all zeros - loopback has no physical NIC, so there is no MAC address).

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

The flags tell the full story: `LOOPBACK` marks it as a loopback device,
`UP` means the interface is administratively enabled, and `LOWER_UP` means
the kernel considers the link layer active.

```bash
ip addr show lo
```

Adds the assigned IP addresses to the output. You will see both IPv4 and IPv6
loopback addresses:

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
```

The `scope host` annotation is important: it tells the kernel that this address
is only reachable from the local machine. No routing entry for it will ever be
advertised to neighbors.

### Bring the interface up or down

```bash
ip link set lo up
ip link set lo down   # breaks ALL local socket communication - use with care
```

Taking `lo` down will immediately break every service on the host that
communicates over `127.0.0.1`, including many database connections and
inter-process sockets. This is a useful experiment in a VM; avoid it on a
shared machine.

### Add an extra loopback address

```bash
# Temporary - lost on reboot
ip addr add 127.0.0.2/8 dev lo

# Verify it was added
ip addr show lo

# Remove it when done
ip addr del 127.0.0.2/8 dev lo
```

Any address in `127.0.0.0/8` is valid. This is commonly used to run two
instances of the same service on different ports while keeping their bind
addresses distinct, or to test software that expects to communicate with a
remote host without actually having one.

### Inspect traffic statistics

```bash
ip -s link show lo
```

Shows bytes, packets, errors, and drops on both RX and TX sides:

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    RX:  bytes packets errors dropped  missed   mcast
      1847204    8923      0       0       0       0
    TX:  bytes packets errors dropped carrier collsns
      1847204    8923      0       0       0       0
```

RX and TX byte counts are always identical on loopback: every byte written is
read back, so nothing is ever lost.

```bash
# Alternative using the proc filesystem
cat /proc/net/dev | grep lo
```

`/proc/net/dev` is updated in real time by the kernel and is readable by any
user - useful for quick scripting without iproute2.

### Watch the routing table

```bash
ip route show table local | grep 127
```

This reveals how the kernel decides to send traffic addressed to `127.x.x.x`
to `lo`:

```
local 127.0.0.0/8 dev lo proto kernel scope host src 127.0.0.1
local 127.0.0.1 dev lo proto kernel scope host src 127.0.0.1
broadcast 127.255.255.255 dev lo proto kernel scope link src 127.0.0.1
```

Three entries: a host route for the entire /8, a more specific host route for
`127.0.0.1` itself, and a broadcast route. The kernel installs all three
automatically when `lo` is brought up; they are never written to a config file.

```bash
# Show only the main routing table
ip route show

# Show all tables (local, main, default)
ip route show table all | grep lo
```

### Capture traffic

```bash
# Watch all frames on the loopback interface
sudo tcpdump -i lo -n

# Filter to a specific port
sudo tcpdump -i lo -n port 8080

# Include a hex dump of each frame
sudo tcpdump -i lo -n -X port 8080

# Capture to a file for later analysis 
sudo tcpdump -i lo -n -w /tmp/lo.pcap
```

`tcpdump` on `lo` is invaluable for verifying that higher-layer code is
producing correctly structured packets. Since loopback traffic never leaves the
kernel, you cannot sniff it from another machine - this is the only way to see
it.

### Ping via loopback

```bash
# IPv4
ping -c 4 127.0.0.1

# IPv6
ping6 -c 4 ::1

# Any address in the /8 block works
ping -c 4 127.0.0.2
```

Pinging the loopback address is the most basic sanity check for the kernel
network stack. If `ping 127.0.0.1` fails, the kernel networking subsystem has
a serious problem.

### Check active connections

```bash
# Established TCP connections to/from localhost
ss -tnp dst 127.0.0.1

# All TCP sockets listening on localhost
ss -tlnp src 127.0.0.1

# UDP sockets on localhost
ss -unlp src 127.0.0.1
```

`ss` (socket statistics) is the modern replacement for `netstat`. The `-t`
flag filters to TCP, `-u` to UDP, `-l` to listening sockets, `-n` suppresses
DNS lookups, and `-p` shows the owning process. Use it to verify that a server
is actually bound before trying to connect.

### Measure round-trip latency

```bash
# Simple RTT measurement - expect sub-millisecond results
ping -c 10 -q 127.0.0.1

# Measure with higher frequency to catch jitter
ping -c 100 -i 0.01 127.0.0.1
```

Loopback latency is effectively zero under normal conditions - typically under
100 microseconds. Any result above a few milliseconds indicates kernel
scheduling pressure or CPU throttling, not network congestion.

---

## Why build our own instead of using the kernel's?

The kernel `lo` is a real interface with real state. Using it from a Go test
suite introduces several problems:

- Creating test interfaces requires `root` or `CAP_NET_ADMIN`.
- CI pipelines often run in unprivileged containers with no access to raw sockets.
- Interface state leaks between tests unless explicitly cleaned up.
- Tests that depend on real network interfaces are inherently slower and less
  deterministic than pure in-process tests.

What we actually need is much simpler: something that satisfies a `LinkDevice`
interface so that every layer above it (Ethernet, ARP, IP, TCP) can be
unit-tested without any OS interaction.

![loopback overview](/images/loopback_post_3.png "loopback overview")

The contract is simple: `Write` enqueues a frame, `Read` dequeues one, `Close`
shuts everything down. That is the entire interface our protocol stack needs
from a device.

---

## The implementation

### The struct

```go
type Device struct {
    name   string
    mtu    int
    ch     chan []byte
    closed chan struct{}
}
```

Four fields. `ch` is the frame queue; `closed` is the shutdown signal. `name`
and `mtu` are metadata - they do not participate in the data path. That is all
the state the device needs.

### Construction with functional options

Rather than a constructor that grows a new parameter every time someone needs a
different MTU or buffer size, the device uses the functional options pattern:

```go
type Option func(*Device)

func WithMTU(mtu int) Option {
    return func(d *Device) { d.mtu = mtu }
}

func WithBufferSize(n int) Option {
    return func(d *Device) { d.ch = make(chan []byte, n) }
}

func WithName(name string) Option {
    return func(d *Device) { d.name = name }
}
```

Each option is just a function that mutates the device. `New` initialises the
struct with defaults and then applies options in order:

```go
func New(opts ...Option) *Device {
    d := &Device{
        name:   "lo",
        mtu:    defaultMTU,                          // 65535
        ch:     make(chan []byte, defaultBufferSize), // 256 frames
        closed: make(chan struct{}),
    }
    for _, o := range opts {
        o(d)
    }
    return d
}
```

Callers only specify what they want to override. Everything else stays at its
documented default:

```go
dev := loopback.New()                               // defaults: "lo", 65535 MTU, 256-frame buffer
dev := loopback.New(loopback.WithMTU(1500))         // Ethernet MTU for fragmentation tests
dev := loopback.New(loopback.WithBufferSize(1024))  // large buffer for throughput benchmarks
dev := loopback.New(loopback.WithName("tap0"))      // named device for multi-device test rigs
```

The device is ready to use immediately after `New`. There is no separate
`Start` call, no background goroutine, nothing to initialise asynchronously.

### Write

```go
func (d *Device) Write(p []byte) (int, error) {
    select {
    case <-d.closed:
        return 0, ErrClosed
    default:
    }

    frame := make([]byte, len(p))
    copy(frame, p)

    select {
    case <-d.closed:
        return 0, ErrClosed
    case d.ch <- frame:
        return len(p), nil
    }
}
```

There are two distinct `select` blocks, which might look odd at first. They
serve different purposes.

The first `select` is a fast pre-check. If the device is already closed, we
return immediately without allocating. This avoids a `make` + `copy` that would
be discarded a nanosecond later.

Then we allocate and copy. The caller's buffer `p` belongs to the caller. After
`Write` returns, the caller may reuse or overwrite it. If we stored the
original slice, the receiver would observe the mutation: a data race. Copying
before enqueuing eliminates the race with zero synchronisation overhead, at the
cost of one allocation per frame.

The second `select` blocks until either the frame is enqueued or the device is
closed. Both branches are handled correctly: enqueue succeeds, or we detect a
concurrent `Close` and return `ErrClosed`.

### Read

```go
func (d *Device) Read(p []byte) (int, error) {
    select {
    case <-d.closed:
        return 0, ErrClosed
    case frame, ok := <-d.ch:
        if !ok {
            return 0, ErrClosed
        }
        if len(p) < len(frame) {
            return 0, fmt.Errorf("loopback: read buffer too small: need %d, have %d", len(frame), len(p))
        }
        return copy(p, frame), nil
    }
}
```

`Read` blocks until either a frame is available or the device is closed. The
`select` gives us both conditions simultaneously - no polling, no sleep loops,
no timers.

The `ok` check on the channel receive handles the case where `ch` itself was
closed (rather than `d.closed` being signalled). In practice `Write` never
closes `ch`, but defensive code is worth writing here.

The buffer size check protects callers who pass an undersized buffer. Passing
`make([]byte, dev.MTU())` as the receive buffer guarantees this check never
triggers for valid frames.

### Close

```go
func (d *Device) Close() error {
    select {
    case <-d.closed:     // already closed: no-op
    default:
        close(d.closed)  // first close: broadcast stop
    }
    return nil
}
```

Closing a Go channel wakes every goroutine blocked on a receive from it,
simultaneously. This makes `d.closed` a broadcast signal: one `close()` call
unblocks any number of concurrent `Read` or `Write` callers without any
per-goroutine coordination.

The `select` idiom makes `Close` idempotent. Calling `close()` on an already-
closed channel panics in Go. The select checks first: receiving from a closed
channel is immediate, so if `d.closed` is already closed, we take the no-op
branch silently. Safe to call from `defer`, from cleanup handlers that fire in
unpredictable order, from multiple goroutines.

---

## Design decisions, explained

### Why a buffered channel?

A buffered Go channel is the simplest correct FIFO for frame delivery. It is
lock-free for the common case (enqueue/dequeue without blocking), handles
concurrent access correctly without any mutex, and integrates naturally into
`select` for multiplexing with the shutdown signal.

The default capacity of 256 frames is large enough that a burst of test traffic
never blocks the writer, but small enough that a runaway producer will
eventually apply back-pressure rather than growing the queue unboundedly. For
high-throughput benchmarks, `WithBufferSize` lets you increase it:

```go
dev := loopback.New(loopback.WithBufferSize(4096))
```

### Why copy on write?

Real hardware NIC drivers use DMA ring buffers - the NIC writes directly into
kernel memory and the driver hands a pointer to the upper layer. No copy
needed, but the buffer must remain pinned until the NIC is done with it.

We have no DMA and no pinning mechanism. The caller's `p` slice is heap memory
they own. The moment `Write` returns, the caller may free, reuse, or modify it.
If we stored the slice without copying, the reader could observe mid-mutation
state from another goroutine - a data race that the Go race detector would flag
immediately.

One allocation per frame is the cost. For a test device, correctness takes
priority over allocation avoidance.

### Why 65535 as the default MTU?

The system loopback uses 65536 bytes. Since there is no physical medium, there
is no reason to simulate the fragmentation imposed by Ethernet's 1500-byte
limit. A large MTU lets upper-layer tests send jumbo datagrams without
triggering the IP fragmentation and reassembly path. When you specifically
*want* to test fragmentation, override with `WithMTU(1500)`:

```go
// Force the IP layer to fragment at Ethernet boundaries
dev := loopback.New(loopback.WithMTU(1500))
```

### Why a signal channel instead of a mutex-protected bool?

The idiomatic Go way to signal cancellation to multiple goroutines is a closed
channel. A `sync.Mutex` protecting a `closed bool` would also work, but it
requires each `Read` and `Write` caller to poll the flag separately from
blocking on the channel - you cannot `select` on a mutex. The signal channel
integrates cleanly into every `select` block.

### No MAC address

The real `lo` reports `00:00:00:00:00:00`. Loopback has no hardware identity -
frames never leave the process, so there is no need to address a remote NIC.
Our Ethernet layer treats the all-zeros address as the loopback sentinel and
skips hardware address resolution entirely.

---

## How it will plug into the rest of the stack

The device has a `Read`, `Write`, `Close`, `Name`, and `MTU` - exactly the
surface a link-layer device needs to expose. The plan for the next layers is
to define a `LinkDevice` interface with that shape and let everything above
the device - Ethernet, ARP, IP, TCP - depend only on that interface. 

Concretely, once we have an Ethernet handler the wiring will look something
like this:

```go
dev := loopback.New()
defer dev.Close()

// pretend these exist in the next post
eth := ethernet.New(dev)
go eth.Start(ctx)

eth.Send(dstMAC, etherTypeARP, payload)
```

A test that exercises an Ethernet round trip looks identical whether the
underlying device is `loopback.New()` or a real TAP interface bound to a
kernel `tun` file - that is the whole point of putting the `LinkDevice`
abstraction at this layer. For test setups that need two distinct devices
talking to a shared switch fabric, named devices keep the diagnostics
readable:

```go
client := loopback.New(loopback.WithName("client-lo"))
server := loopback.New(loopback.WithName("server-lo"))
```

The fabric itself (copying frames from one device's TX into the other's RX)
is a tiny goroutine we will write when we need it. The device intentionally
does not know about it.

---

## What is next

The loopback device gives us the foundation: a `Write`/`Read`/`Close` surface
that the rest of the stack can be built on top of.

Next up is the Ethernet layer - parsing 802.3 frame headers, dispatching by
EtherType, and wiring the ARP handler into the stack. With a loopback device
underneath, the entire Ethernet layer can be tested with raw byte slices and
zero network access.

The full source is at [rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack).
