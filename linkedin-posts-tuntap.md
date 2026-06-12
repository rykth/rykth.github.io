# LinkedIn posts — Building a TUN/TAP device

Five posts derived from [`content/posts/building-a-tuntap-device.md`](content/posts/building-a-tuntap-device.md), each on a different angle of the TUN/TAP device and the Linux commands that exercise it.

---

## Post 1 — `/dev/net/tun` is a clone device, not a singleton

There's only one `/dev/net/tun` on a Linux box, but every `open()` of it yields a brand new network interface.

It's a character device - major 10, minor 200, type `c` in `ls -la /dev/net/tun` - registered with the kernel's miscdevice subsystem. Crucially, it has no per-fd state until you call `ioctl(fd, TUNSETIFF, &ifreq)`. Before that ioctl runs, the descriptor is inert. After it runs, the kernel has either created a fresh `tap0`/`tap1`/`tun0` interface (if you passed an empty name) or attached to an existing one (if you passed a name that already exists). You can verify what's holding which interface open via `cat /proc/net/tun` - the kernel exposes `(count, socket pointer, name, flags)` for every active TUN/TAP fd. Closing the last fd that points to an interface created this way destroys the interface. Persistent interfaces created with `ip tuntap add dev tap0 mode tap` flip a flag that decouples lifetime from fd count - they survive until `ip tuntap del` runs explicitly. This is the cleanest example in Linux of "the device file is a factory, not a singleton."

I've been writing a "Building a TCP/IP stack" series where I implement each layer from scratch. The TUN/TAP device post walks through opening `/dev/net/tun`, running TUNSETIFF, wiring poll-based blocking I/O, and bringing the interface up - all from Go.

Find the full TUN/TAP device breakdown in the comments below 👇

---

## Post 2 — TUN vs TAP is one bit in one struct

Linux's TUN and TAP modes are the same kernel device with a different flag.

When you open `/dev/net/tun` and call `ioctl(fd, TUNSETIFF, &ifreq)`, the only thing that determines whether you get a Layer 3 packet interface or a Layer 2 frame interface is one bit in `ifreq.ifr_flags`: `IFF_TUN` (0x0001) for raw IP packets, `IFF_TAP` (0x0002) for full Ethernet frames. The same character device, the same ioctl, the same 40-byte struct — flip one bit and the semantics of every subsequent `read()` and `write()` change. TUN delivers IP packets with no MAC header; TAP delivers complete Ethernet frames including the 14-byte L2 header that ARP and bridging depend on. This is why VPNs (WireGuard, OpenVPN) use TUN — they only care about IP — while QEMU/KVM and container bridges use TAP, because they need to participate in the L2 broadcast domain. You can see which mode an interface is in with `ip tuntap list`: it prints `tap0: tap` or `tun1: tun` based on which flag was set when the fd was bound.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The TUN/TAP post walks through the exact ioctl flow — TUNSETIFF, the 40-byte ifreq layout, IFF_NO_PI, all of it — from a Go implementation.

Find the full TUN/TAP device breakdown in the comments below 👇

---

## Post 3 — IFF_NO_PI: the 4 mystery bytes at the start of every frame

By default, every read from a TUN/TAP file descriptor returns 4 mystery bytes before the actual frame. They're almost certainly the source of your bug.

The Linux TUN/TAP driver prepends a packet-info header to every frame: 2 bytes of flags (rarely used) followed by 2 bytes of EtherType in big-endian. So if you `read()` a SYN packet from `/dev/net/tun` expecting an IP header, you get four bytes of "metadata" first, and your parser interprets the IP version nibble as part of the wrong field. The fix is one line: set `IFF_NO_PI` (0x1000) in your `ifreq.ifr_flags` when calling TUNSETIFF. With it set, the descriptor delivers raw bytes starting at offset 0 — the IP header in TUN mode, the Ethernet header in TAP mode — exactly what a real NIC driver hands to the kernel network stack. The packet-info header existed to let userspace pre-route packets without parsing them, which made sense in 2001 but turns out to be a footgun for anyone writing a userspace stack. Every example on the internet that "mysteriously doesn't work" against a TUN device is forgetting this flag.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The TUN/TAP post explains the IFF_NO_PI behavior with the exact 4-byte layout and why my implementation sets it unconditionally.

Find the full TUN/TAP device breakdown in the comments below 👇

---

## Post 4 — `ip tuntap add ... user $USER` is how WireGuard runs without setuid

`sudo ip tuntap add dev tap0 mode tap user $USER` is the most underused command in Linux networking.

Opening `/dev/net/tun` requires CAP_NET_ADMIN, which is why every TUN/TAP example you read starts with `sudo`. But you don't need root for the process that *uses* the interface — you only need root for the user that *creates* it. The `user=` option on `ip tuntap add` records the UID that's allowed to open the interface afterward without privileges. After running the command once as root, your unprivileged Go program, test runner, or CI job can open `/dev/net/tun`, attach to `tap0` by name, and read/write frames freely. No setuid binary, no `capset()` calls, no `docker --privileged`. The interface stays around across `exec()` and across program restarts because it's persistent. This is how production VPN clients ship with no setuid: a privileged installer creates the interface once with `user=` (and optionally `group=`) and the daemon runs as a regular user for the rest of its life. The kernel's permission check on the device file enforces the restriction without any further plumbing.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch. The TUN/TAP post explains both the privileged setup path and how to drive the interface from an unprivileged Go process afterward.

Find the full TUN/TAP device breakdown in the comments below 👇

---

## Post 5 — `eventfd` is how you wake every goroutine blocked in `poll()` with one syscall

Linux's `eventfd` is what makes "wake up every waiter blocked in `poll()`" a one-write operation.

When a Go program does I/O against a TUN/TAP file descriptor, it has to sleep in `poll()` waiting for the interface to become readable. The problem: how do you wake that goroutine on shutdown without resorting to per-goroutine signaling or timeouts? The kernel answer is `eventfd(0, EFD_NONBLOCK|EFD_CLOEXEC)` — a file descriptor that holds an 8-byte counter and integrates with `poll()`. Include the eventfd in your `poll()` set alongside the TUN fd, and the moment something writes 8 bytes to it (incrementing the counter), every blocked `poll()` returns immediately with POLLIN on that fd. One write, N waiters woken. There's no goroutine targeting, no condvar broadcast, no mutex around state — just file-descriptor semantics. On close, write 8 bytes to the eventfd: every read or write currently parked in `poll()` returns `os.ErrClosed` without further coordination. The same trick is the backbone of how systemd's sd_event loop, nginx's worker shutdown, and Go's own `runtime/netpoll` wake their parked goroutines.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch. The TUN/TAP post shows the exact eventfd-plus-poll pattern in Go, plus the EINTR retry loop that catches signal interrupts.

Find the full TUN/TAP device breakdown in the comments below 👇
