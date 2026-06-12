# LinkedIn posts — Building a loopback device

Five posts derived from [`content/posts/building-a-loopback-device.md`](content/posts/building-a-loopback-device.md), each on a different angle of the loopback device and the Linux commands that exercise it.

---

## Post 1 — The whole /8 is loopback

127.0.0.1 is the famous one, but the entire 127.0.0.0/8 is loopback.

RFC 5735 reserves the full /8 — 16,777,216 addresses from 127.0.0.0 through 127.255.255.255 — for loopback traffic. Any IP in that block routes back to `lo` on the same host. The kernel installs a single catch-all entry: `local 127.0.0.0/8 dev lo proto kernel scope host src 127.0.0.1`, which you can see with `ip route show table local | grep 127`. There's no /32 per address — one route covers all 16 million. This is what lets you run a dozen services bound to different 127.x.x.x addresses without port collisions: bind one daemon to 127.0.0.2:80 and another to 127.0.0.3:80, and the kernel disambiguates by destination IP at the routing layer rather than forcing you to use different ports. Adding an alias is one command: `sudo ip addr add 127.0.0.2/8 dev lo`. No interface configuration, no firewall change, no port conflict — just a free address.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The first post walks through a userspace loopback device that satisfies the same LinkDevice contract as `lo`, in-process, no root needed.

Find the full loopback device walkthrough in the comments below 👇

---

## Post 2 — The 65,536 MTU

`ip link show lo` reports an MTU of 65536. That's 43x the standard Ethernet MTU.

Real interfaces top out at 1500 bytes per frame because Ethernet was designed in the 1970s for media that couldn't transmit larger payloads reliably. Loopback has no media — every "frame" is a memory copy inside the kernel — so the historical MTU constraint never applied. The kernel sets `lo`'s MTU to 65536 (the maximum addressable by a 16-bit IP length field, plus one) so the IP layer's fragmentation path never fires on loopback. A 64 KiB response from a local Redis to a local web server is one IP datagram, one TCP segment, one memcpy — no reassembly, no segment boundaries, no per-fragment overhead. You can change it — `sudo ip link set lo mtu 1500` — and watch your local socket throughput collapse as the kernel suddenly has to fragment and reassemble traffic that used to flow as a single datagram. The 65536 default is the kernel's way of saying "this interface has no wire; stop pretending it does."

I've been writing a "Building a TCP/IP stack in Go" series. The first post implements a userspace loopback device — same idea, MTU defaults to 65535, override with WithMTU(1500) when you specifically want to exercise the fragmentation path.

Find the full loopback device walkthrough in the comments below 👇

---

## Post 3 — RX always equals TX

Run `ip -s link show lo` and the RX and TX byte counters will always be identical. This is a property the kernel cannot violate.

Every byte written to the loopback interface must be read back from it, because loopback is its own destination. There's no other side to lose packets to, no buffer to overflow, no link to disconnect. The two counters incrementing in lockstep is the visible invariant of "this device has no external receiver." If you ever see them diverge, the kernel has a bug. The counters also reveal traffic you might not realise is happening: every connection your daemons make to 127.0.0.1 — PostgreSQL clients to a local Postgres, your shell talking to systemd, DNS lookups against systemd-resolved — adds to both columns simultaneously. On a busy server, `lo`'s counter is often higher than the real NIC's, because internal IPC dominates external traffic. The `/proc/net/dev` file exposes the same numbers without iproute2, which makes it trivial to graph: `watch -n 1 'cat /proc/net/dev | grep lo'`.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch. The first post — a userspace loopback device — is a Go channel pretending to be a NIC, and it has the same RX-equals-TX invariant for the same reason.

Find the full loopback device walkthrough in the comments below 👇

---

## Post 4 — `scope host` is a kernel safety feature

`ip addr show lo` reports `scope host` on the 127.0.0.1 entry. Those two words are an explicit kernel safety boundary.

The kernel uses four address scopes: global (routable to the public internet), site (deprecated), link (reachable on the same physical segment), and host (reachable only from the local machine). When an address is `scope host`, the kernel refuses to advertise it anywhere — no ARP responses on other interfaces, no inclusion in routing announcements, no exposure to remote peers asking "who has this IP?" This is what prevents a hostile neighbor from claiming 127.0.0.1 and intercepting your traffic to localhost: the kernel would never have told them the address existed in the first place. The scope is set automatically when an address belongs to the loopback subnet, but you can pin it manually with `ip addr add 192.168.1.1/32 dev eth0 scope host` to create a "private" address that the kernel will never leak to the wire. It's also what makes `ip route show table local` interesting: every host-scoped address gets a route there, and nowhere else.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The loopback device is the first piece — same isolation guarantee as `scope host`, achieved by simply never crossing a goroutine boundary.

Find the full loopback device walkthrough in the comments below 👇

---

## Post 5 — `ip link set lo down` breaks your machine

Running `sudo ip link set lo down` is one of the fastest ways to wreck a Linux server. Don't try it on anything you care about.

`lo` is not just for `ping 127.0.0.1`. Almost every long-running daemon on a modern Linux box uses loopback for inter-process communication: PostgreSQL's TCP listener on 127.0.0.1:5432, Redis on 127.0.0.1:6379, systemd-resolved's DNS on 127.0.0.53:53, the Docker daemon's API on 127.0.0.1:2375 if you've configured it that way, X11 auth over localhost. The moment `lo` goes down, every one of those connections drops, and any new connection attempt to a 127.x.x.x address returns ENETUNREACH. systemd will start respawning services that depend on local DNS; cron jobs that talk to a local database will fail; even `sudo` itself sometimes breaks if it's configured to log via syslog over a Unix socket that the kernel routes through the loopback path. Bringing it back with `ip link set lo up` recovers most of these, but some daemons cache failures and need a restart. The lesson: `lo` is mission-critical infrastructure on every Linux box, even one with no network cable.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch. The first post — a userspace loopback device — is a useful mental model for what the kernel's `lo` actually does, minus the existential risk of accidentally bringing it down.

Find the full loopback device walkthrough in the comments below 👇
