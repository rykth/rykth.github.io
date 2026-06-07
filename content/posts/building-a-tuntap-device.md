---
title: "Building a TUN/TAP device in Go"
date: 2026-06-07T10:00:00+00:00
description: "The second post in my TCP/IP stack series. The loopback device gave us a fully in-process pipe for unit tests. TUN/TAP gives us a real virtual NIC that the kernel routes traffic through - bridging our userspace stack to the wider network."
tags: [golang, networking, tcpip, tuntap, linux]
---

## Where this fits

In the [previous post](https://rykth.github.io/posts/building-a-loopback-device/) we built a loopback
device: an in-process `Write`/`Read` pipe backed by a Go channel. It requires
no privileges, runs on any OS, and makes unit tests fast and deterministic.

But at some point we need to talk to the real network. We need a device the
kernel's routing table knows about, one that `tcpdump` can sniff and that other
processes on the same host can reach. That is what this post is about.

The code lives at [rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack),
in `pkg/raw/tuntap`.


![tuntap layer](/images/tuntap_post_1.png "tuntap layer")

---

## What is TUN/TAP?

TUN/TAP is a Linux kernel feature that creates a virtual network interface
backed by a userspace file descriptor instead of physical hardware. When the
kernel decides to send a packet out through a TUN/TAP interface, the packet
appears in userspace as data readable from a file descriptor. When userspace
writes data to that file descriptor, the kernel receives it as if it had
arrived from a real NIC.

![what is tuntap](/images/tuntap_post_2.png "what is tuntap")

The kernel exposes both TUN and TAP through a single *clone device*:
`/dev/net/tun`. Every `open()` on that path, followed by a `TUNSETIFF` ioctl,
yields an independent virtual interface.

---

## TUN vs TAP

The name encodes the two operating modes:

| Mode | Layer | What you read/write | Ethernet header | Primary use |
|------|-------|---------------------|-----------------|-------------|
| **TUN** | 3 (Network) | Raw IP packets | No | VPNs (WireGuard, OpenVPN), IP tunnels |
| **TAP** | 2 (Link) | Full Ethernet frames | Yes, 14 bytes | VM networking, bridges, **this stack** |

**TUN** gives you raw IP packets. The kernel strips the Ethernet header before
delivery, and you must provide a valid IP packet when writing. This suits VPNs
that only care about routing IP traffic between endpoints.

**TAP** gives you complete Ethernet frames including the 6+6-byte MAC header
and the 2-byte EtherType field. The kernel treats a TAP interface exactly like
a real Ethernet NIC - it participates in ARP, can be added to bridges, and can
carry any EtherType payload (IP, ARP, IPv6, etc.).

We use TAP. Our Ethernet layer needs to read and write full frames. Stripping
the MAC header would hide the information our ARP and Ethernet code must handle.

In Go, the distinction is a single constant:

```go
type DeviceType uint16

const (
    DeviceTAP DeviceType = unix.IFF_TAP  // 0x0002
    DeviceTUN DeviceType = unix.IFF_TUN  // 0x0001
)
```

---

## Exploring TUN/TAP from the command line

Before writing a single line of Go it is worth understanding what the kernel
gives us. All commands in this section require root or `CAP_NET_ADMIN`.

### Verify the clone device exists

```bash
ls -la /dev/net/tun
```

```
crw-rw-rw- 1 root root 10, 200 May 18 08:00 /dev/net/tun
```

The `c` means character device. Major 10, minor 200 - the kernel registers
it as a *miscdevice*. Every open gives you an independent file descriptor and
there is no per-process state before the `TUNSETIFF` ioctl.

```bash
# See which processes currently hold open TUN/TAP file descriptors
cat /proc/net/tun
```

```
ct  sk               name  flags
1   ffff9a3c4e20e000  tap0  0002
```

`ct` is the open-fd count, `sk` is the kernel socket pointer, `name` is the
interface name, and `flags` encode the mode (`0002` = `IFF_TAP`).

### List active TUN/TAP interfaces

```bash
# Modern iproute2 - dedicated subcommand
ip tuntap list
```

```
tap0: tap MULTI_QUEUE
tun1: tun
```

```bash
# Filter ip link output by type
ip link show type tun
ip link show type tap
```

### Create and delete interfaces

```bash
# Create a persistent TAP interface (survives until deleted or reboot)
sudo ip tuntap add dev tap0 mode tap

# Create a persistent TUN interface
sudo ip tuntap add dev tun0 mode tun

# Assign to a specific user so non-root processes can open it
sudo ip tuntap add dev tap0 mode tap user $USER

# Delete them when done
sudo ip tuntap del dev tap0 mode tap
sudo ip tuntap del dev tun0 mode tun
```

A *persistent* interface created with `ip tuntap add` stays in the kernel even
after the process that created it exits. An interface created by opening
`/dev/net/tun` and calling `TUNSETIFF` in code disappears when the last
file descriptor for it is closed.

### Inspect a live interface

```bash
# Link-layer view: flags, MTU, MAC address
ip link show tap0
```

```
5: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
```

Unlike the loopback (`00:00:00:00:00:00`), a TAP interface has a real MAC
address - the kernel assigns a random one. This matters for ARP: other hosts
on the same subnet will ARP for this address.

```bash
# IP address view
ip addr show tap0
```

```
5: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.100.1/24 scope global tap0
```

### Bring up and assign an address

When our code calls `New`, the interface starts in the `DOWN` state - the
kernel creates it but does not route traffic to it yet. Our `New` implementation
calls `SIOCSIFFLAGS` to bring it `UP` automatically. But assigning an IP
address still needs to be done externally:

```bash
sudo ip link set tap0 up
sudo ip addr add 192.168.100.1/24 dev tap0
```

Now the kernel routing table has an entry for `192.168.100.0/24` via `tap0`,
and our stack receives any traffic the kernel routes to that subnet.

### Add routes

```bash
# Route a specific subnet through the TAP interface
sudo ip route add 192.168.100.0/24 dev tap0

# Add a default route - all unmatched traffic goes through tap0
sudo ip route add default via 192.168.100.1 dev tap0

# Verify the routing table
ip route show
```

```
default via 192.168.100.1 dev tap0
192.168.100.0/24 dev tap0 proto kernel scope link src 192.168.100.1
```

### Traffic statistics

```bash
# Byte and packet counters with error breakdown
ip -s link show tap0
```

```
5: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    RX:  bytes packets errors dropped  missed   mcast
      48532     312      0       0       0       2
    TX:  bytes packets errors dropped carrier collsns
      31044     198      0       0       0       0
```

```bash
# Same data from /proc - scriptable without iproute2
cat /proc/net/dev | grep tap0
```

### Change the MTU

```bash
# Jumbo frames for benchmarking
sudo ip link set tap0 mtu 9000

# Back to standard Ethernet
sudo ip link set tap0 mtu 1500
```

MTU changes take effect immediately. The kernel will drop frames larger than
the configured MTU, so both ends must agree.

### Capture traffic

This is where TUN/TAP becomes invaluable for debugging. Unlike the loopback
device, every frame our stack sends or receives is visible to `tcpdump`:

```bash
# All traffic
sudo tcpdump -i tap0 -n

# Only ARP - useful when debugging address resolution
sudo tcpdump -i tap0 -n arp

# Only TCP to/from port 8080, with protocol decode
sudo tcpdump -i tap0 -n -v tcp port 8080

# Hex dump - see the raw bytes our stack is producing
sudo tcpdump -i tap0 -n -X tcp port 8080

# Save for Wireshark analysis
sudo tcpdump -i tap0 -w /tmp/tap0.pcap
```

Run `tcpdump` in one terminal while our stack runs in another. Every ARP
request, every IP datagram, every TCP SYN is visible. This is the fastest way
to diagnose why the stack is not reaching a peer.

### Check active connections

```bash
# TCP connections going through tap0
ss -tnp dst 192.168.100.0/24

# Check what is listening on the TAP-connected subnet
ss -tlnp src 192.168.100.0/24
```

### Ping through the interface

```bash
# Verify the kernel can reach the address we assigned
ping -c 4 192.168.100.1

# If our stack is running and responding to ARP, this should get replies
ping -c 4 192.168.100.2
```

---

## The implementation

### Three file descriptors

Every `Device` manages exactly three file descriptors:

```
tfd.fd          - open("/dev/net/tun") + TUNSETIFF
                  Frame I/O: read incoming frames, write outgoing frames

tfd.shutdownFd  - eventfd(0, EFD_NONBLOCK|EFD_CLOEXEC)
                  Shutdown signal: written by Close, watched by Poll

ctlFd           - socket(AF_INET, SOCK_DGRAM, IPPROTO_IP)
                  Control socket: SIOCSIFMTU, SIOCSIFFLAGS ioctls
```

The split between `tfd.fd` and `ctlFd` is a kernel constraint. The TUN/TAP
fd supports `TUNSETIFF` but not the network interface ioctls like `SIOCSIFMTU`
or `SIOCSIFFLAGS`. Those require a socket. The conventional choice is a
`SOCK_DGRAM` socket on `AF_INET` - it is never used for actual communication,
only as a handle for ioctl calls.

### The struct

```go
type Device struct {
    tfd    tunFd
    name   string
    kind   DeviceType
    mtu    int
    ctlFd  int
    closed uint32  // atomic; 0=open, 1=closed
}

type tunFd struct {
    fd         int
    shutdownFd int
    readPoll   [2]unix.PollFd  // {tun POLLIN,  shutdownFd POLLIN}
    writePoll  [2]unix.PollFd  // {tun POLLOUT, shutdownFd POLLIN}
}
```

The `PollFd` arrays are pre-built at construction time so `Read` and `Write`
never allocate on the hot path. `closed` is a `uint32` rather than a `bool`
because `sync/atomic` operates on integer types.

### Defaults and the lone option

Unlike the loopback device, which exposes three functional options, the
TUN/TAP device exposes only one - `WithMTU` - because almost everything else
is fixed by the kernel:

```go
const defaultMTU = 1500 // matches a standard Ethernet interface

func WithMTU(mtu int) Option {
    return func(d *Device) { d.mtu = mtu }
}
```

The interface name is a required positional argument to `New` (passing `""`
lets the kernel pick one), and the device type - TAP or TUN - is selected by
the second positional argument. Everything else is internal.

```go
dev, _ := tuntap.New("tap0", tuntap.DeviceTAP)                  // default 1500 MTU
dev, _ := tuntap.New("",     tuntap.DeviceTAP, tuntap.WithMTU(9000))  // kernel picks a name, jumbo MTU
```

### `New` - step by step

```go
func New(name string, kind DeviceType, opts ...Option) (*Device, error) {
    // 1. Open the clone device
    fd, err := unix.Open("/dev/net/tun", unix.O_RDWR|unix.O_CLOEXEC, 0)

    // 2. Configure the interface via TUNSETIFF
    var req ifReq
    req.Flags = uint16(kind) | unix.IFF_NO_PI
    copy(req.Name[:], name)
    ioctlTUNSETIFF(uintptr(fd), &req)
    d.name = strings.TrimRight(string(req.Name[:]), "\x00")

    // 3. Wrap the fd with poll-based blocking I/O
    tfd, _ := newTunFd(fd)
    d.tfd = tfd

    // 4. Open a control socket for ioctl calls
    ctlFd, _ := unix.Socket(unix.AF_INET, unix.SOCK_DGRAM|unix.SOCK_CLOEXEC, unix.IPPROTO_IP)
    d.ctlFd = ctlFd

    // 5. Set the MTU via SIOCSIFMTU
    d.setMTU(d.mtu)

    // 6. Bring the interface UP via SIOCGIFFLAGS + SIOCSIFFLAGS
    d.setUp()

    return d, nil
}
```

Each step can fail, and each failure closes all previously opened fds before
returning. `O_CLOEXEC` and `SOCK_CLOEXEC` ensure the fds are not inherited
by child processes.

Step 2 deserves more attention. After `TUNSETIFF`, the kernel writes the
final interface name back into `req.Name`. If we passed `""`, the kernel
assigns the next available name (`tap0`, `tap1`, …). We read it back with
`strings.TrimRight` to strip the NUL padding.

`IFF_NO_PI` suppresses the kernel's 4-byte *packet info* header. Without it,
every frame would start with:

```
Offset  Size  Field
0       2     Flags  (e.g. TUN_PKT_STRIP)
2       2     Protocol  (big-endian EtherType, e.g. 0x0800 = IPv4)
4       ...   Actual frame
```

With `IFF_NO_PI` set, raw bytes start at offset 0, matching what a real NIC
driver delivers to the upper layer.

### Why not `os.File.Read`?

The intuitive approach is to wrap the fd in an `os.File` and call `Read`:

```go
f := os.NewFile(uintptr(fd), "tap0")
n, err := f.Read(buf)
```

This does not work correctly. The Go runtime's network poller integrates with
`os.File` for network sockets, but TUN/TAP fds behave differently. In
particular, calling `Close` on a TUN/TAP fd does not reliably unblock a
goroutine sleeping in `Read` via the Go runtime poller - the goroutine may
leak or return the wrong error.

The solution is to take full control of blocking using `unix.Poll`.

### Non-blocking fd + `unix.Poll`

```go
func newTunFd(fd int) (tunFd, error) {
    unix.SetNonblock(fd, true)

    shutdownFd, _ := unix.Eventfd(0, unix.EFD_NONBLOCK|unix.EFD_CLOEXEC)

    return tunFd{
        fd:         fd,
        shutdownFd: shutdownFd,
        readPoll: [2]unix.PollFd{
            {Fd: int32(fd),         Events: unix.POLLIN},
            {Fd: int32(shutdownFd), Events: unix.POLLIN},
        },
        writePoll: [2]unix.PollFd{
            {Fd: int32(fd),         Events: unix.POLLOUT},
            {Fd: int32(shutdownFd), Events: unix.POLLIN},
        },
    }, nil
}
```

Setting the tun fd non-blocking means `unix.Read` and `unix.Write` return
`EAGAIN` immediately when no data is available or the kernel buffer is full,
instead of sleeping inside the syscall. We then call `unix.Poll` explicitly,
passing both the tun fd and the shutdown eventfd in the same `PollFd` array.

`Poll` sleeps until one of the fds becomes ready. If the tun fd is ready, we
retry the `Read` or `Write`. If the shutdown eventfd is ready, `Close` was
called and we return `os.ErrClosed`. Both conditions are handled in one
blocking call, with no polling loop or busy-wait.

The full Read loop:

```go
func (d *Device) Read(p []byte) (int, error) {
    for {
        n, err := unix.Read(d.tfd.fd, p)
        if err == nil {
            return n, nil
        }
        switch err {
        case unix.EAGAIN:
            if pollErr := d.tfd.blockOnRead(); pollErr != nil {
                return 0, pollErr
            }
        case unix.EINTR:
            // Signal interrupted the syscall - retry immediately.
        case unix.EBADF:
            return 0, os.ErrClosed
        default:
            return 0, fmt.Errorf("tuntap: read: %w", err)
        }
    }
}
```

`Write` is identical in structure, using `blockOnWrite` instead.

### `eventfd` - a zero-overhead shutdown signal

A Linux `eventfd` is a file descriptor that holds a 64-bit counter. Writing
8 bytes increments the counter; reading 8 bytes drains it. Crucially, it
participates in `poll`/`select`/`epoll` - when the counter is non-zero, the
fd is readable (`POLLIN`).

```go
func (t *tunFd) wakeForShutdown() {
    var buf [8]byte
    binary.NativeEndian.PutUint64(buf[:], 1)
    _, _ = unix.Write(t.shutdownFd, buf[:])
}
```

When `Close` calls `wakeForShutdown`, every goroutine sleeping in `unix.Poll`
returns immediately - they all see `POLLIN` on `shutdownFd`. No mutex,
no channels, no per-goroutine signalling. One write, all goroutines wake.

- `EFD_NONBLOCK` - reads and writes on the eventfd never block
- `EFD_CLOEXEC` - the fd is closed automatically on `exec`

### EINTR - signals can interrupt `Poll`

On Linux, any signal delivered to a process (a `SIGCHLD` from a subprocess,
a `SIGUSR1` from a monitoring tool, even `SIGWINCH` from terminal resize)
causes blocking syscalls to return `EINTR`. Without a retry loop, a single
signal would permanently break I/O.

```go
func (t *tunFd) blockOnPoll(fds []unix.PollFd) error {
    const problemFlags = unix.POLLHUP | unix.POLLNVAL | unix.POLLERR
    for {
        _, err := unix.Poll(fds, -1)
        if err == unix.EINTR {
            continue  // signal interrupted - retry
        }

        tunRevents  := fds[0].Revents
        shutRevents := fds[1].Revents
        fds[0].Revents = 0
        fds[1].Revents = 0

        if err != nil {
            return err
        }
        if shutRevents&(unix.POLLIN|problemFlags) != 0 || tunRevents&problemFlags != 0 {
            return os.ErrClosed
        }
        return nil
    }
}
```

The `problemFlags` - `POLLHUP`, `POLLNVAL`, `POLLERR` - indicate the fd is
unusable. Rather than returning a syscall-specific error the caller cannot
meaningfully handle, we treat them the same as a shutdown and return
`os.ErrClosed`.

Note the `Revents` reset after reading. `unix.Poll` writes into the `Revents`
fields of the slice we pass in. If we loop back (on `EINTR`), those fields
from the previous iteration must be cleared or the next `Poll` call may
misread stale data.

### Idempotent close with `atomic.CompareAndSwap`

```go
func (d *Device) Close() error {
    if !atomic.CompareAndSwapUint32(&d.closed, 0, 1) {
        return nil  // already closed
    }
    d.tfd.wakeForShutdown()
    d.closeAll()
    return nil
}
```

`atomic.CompareAndSwapUint32` checks that `closed == 0` and sets it to `1` in
a single CPU instruction. Only the first caller wins the swap. Subsequent
callers see `closed == 1`, the swap fails, and they return immediately without
closing fds a second time- which would be a use-after-close bug.

This is different from the loopback device, which uses a closed channel as its
shutdown signal. Here we use an atomic flag because we are coordinating with
the kernel's poll machinery rather than Go's scheduler.

### The `ifreq` struct - getting the size exactly right

The kernel's `TUNSETIFF`, `SIOCSIFMTU`, and `SIOCSIFFLAGS` ioctls all expect
a pointer to an `ifreq` union - a 40-byte C struct. The Go representation must
match that layout exactly or the kernel returns `EINVAL`.

```go
type ifReq struct {
    Name  [unix.IFNAMSIZ]byte  // 16 bytes - NUL-terminated interface name
    Flags uint16               //  2 bytes - IFF_TAP | IFF_NO_PI | ...
    _     [22]byte             // 22 bytes - padding (union unused half)
}                              // total: 40 bytes

type ifreqMTU struct {
    Name [unix.IFNAMSIZ]byte   // 16 bytes
    MTU  int32                 //  4 bytes
    _    [20]byte              // 20 bytes padding
}                              // total: 40 bytes

type ifreqFlags struct {
    Name  [unix.IFNAMSIZ]byte  // 16 bytes
    Flags int16                //  2 bytes
    _     [22]byte             // 22 bytes padding
}                              // total: 40 bytes
```

Each variant overlaps the same 40-byte region. The active field lives at
offset 16; everything after that is padding. The unit tests assert all three
structs are exactly 40 bytes:

```go
func TestIfReqSize(t *testing.T) {
    const want = 40
    if got := unsafe.Sizeof(ifReq{}); got != want {
        t.Errorf("ifReq size = %d, want %d", got, want)
    }
}
```

If Go's struct layout rules ever padded a field differently, or if a future
change introduced an extra field, this test catches it immediately -
long before a cryptic `EINVAL` from the kernel would.

### Bringing the interface UP

When the kernel creates a TUN/TAP interface it is in the `DOWN` state. No
traffic is routed to it until it transitions to `UP`. Our `setUp` reads the
current flags first to avoid clobbering bits the kernel may have set:

```go
func (d *Device) setUp() error {
    req := ifreqFlags{}
    copy(req.Name[:], d.name)

    ioctlSIOCGIFFLAGS(uintptr(d.ctlFd), &req)  // read current flags
    req.Flags |= unix.IFF_UP                     // set the UP bit
    ioctlSIOCSIFFLAGS(uintptr(d.ctlFd), &req)   // write back
    return nil
}
```

A simpler implementation would hardcode `req.Flags = unix.IFF_UP`, but that
would clear any other flags the kernel set during interface creation. The
read-modify-write is safer.

---

## How it will plug into the rest of the stack

`Device` exposes exactly the same five methods as the loopback device -
`Read`, `Write`, `Close`, `Name`, `MTU` - so once the next post defines a
`LinkDevice` interface with that shape, swapping the loopback for a TAP is a
single-line change at the test boundary:

```go
// Unit test setup - no privileges, all in-process
dev := loopback.New()

// Integration / real networking setup - requires CAP_NET_ADMIN
dev, err := tuntap.New("tap0", tuntap.DeviceTAP)
```

A minimal end-to-end driver against the current code looks like this:

```go
dev, err := tuntap.New("tap0", tuntap.DeviceTAP)
if err != nil {
    log.Fatal(err) // most commonly: missing CAP_NET_ADMIN
}
defer dev.Close()

log.Printf("listening on %s (MTU %d)", dev.Name(), dev.MTU())

buf := make([]byte, dev.MTU()+14) // MTU + Ethernet header
for {
    n, err := dev.Read(buf)
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("got frame: % x", buf[:n])
}
```

After bringing the process up, assign an IP address to the interface from
another terminal and ping it:

```bash
sudo ip addr add 192.168.100.1/24 dev tap0
ping -c 1 192.168.100.2
```

The kernel routes traffic for `192.168.100.0/24` through `tap0`, delivering
raw Ethernet frames to our `Read` call. The first frame we observe will be an
ARP request (`who-has 192.168.100.2`). Until the next post wires up an ARP
handler, the ping will not get a reply - but `tcpdump -i tap0 -n arp` running
in a third terminal will show the kernel's request arriving on the wire.

That observable round trip - kernel routing -> our `Read` -> printed by Go - is
the first time the stack is doing something a real NIC could do, and it is the
acceptance test the rest of this series builds on.

---

## Loopback vs TUN/TAP

Both devices satisfy `net.LinkDevice`. The choice between them is a testing
strategy choice, not an API choice.

| Property | `pkg/loopback` | `pkg/raw/tuntap` |
|----------|----------------|-----------------|
| Privileges required | None | `CAP_NET_ADMIN` |
| OS support | Any (pure Go) | Linux only |
| Kernel-visible | No | Yes (`ip link show`) |
| `tcpdump`-able | No | Yes |
| Frame delivery | Go channel (in-process) | Kernel routing (real) |
| IPC between processes | No | Yes |
| Latency | Nanoseconds (channel) | Microseconds (syscall + kernel) |
| Ideal for | Unit tests, CI | Integration tests, real networking |

Use the loopback when testing protocol logic in isolation. Use TUN/TAP when
testing how the stack interacts with the kernel, with other processes, or with
the real network.

---

## Limitations

| Limitation | Detail |
|-----------|--------|
| **Linux only** | Uses `//go:build linux`; `/dev/net/tun`, `eventfd`, and the ioctl constants are Linux-specific |
| **`CAP_NET_ADMIN` required** | Opening `/dev/net/tun` requires elevated privileges |
| **Single-queue** | We do not set `IFF_MULTI_QUEUE`; one goroutine drives `Read` at a time |
| **No persistent interfaces** | The interface is destroyed when the process exits or `Close` is called |
| **No VLAN offload metadata** | `IFF_VNET_HDR` is not set; VLAN frames can be received but not synthesised with correct offload hints |
| **No packet loss simulation** | Unlike the loopback, there is no wrapper for fault injection; real kernel buffering applies |

---

## What is next

With both a loopback device and a TUN/TAP device satisfying the same
`LinkDevice` interface, the stack can now be tested at two levels: pure
in-process unit tests with the loopback, and real kernel-routed integration
tests with TAP.

Next up is the Ethernet layer - parsing 802.3 frame headers, demultiplexing by
EtherType, and handling ARP. With TAP underneath, we can write an integration
test that sends a real ARP request and watches `tcpdump` confirm the response.

The full source is at [rykth/tcp-ip-stack](https://github.com/rykth/tcp-ip-stack).
