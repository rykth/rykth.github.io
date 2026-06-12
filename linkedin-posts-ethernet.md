# LinkedIn posts — Building the Ethernet layer

Five posts derived from [`content/posts/building-ethernet-layer.md`](content/posts/building-ethernet-layer.md), each on a different angle of the Ethernet frame format and the Linux commands that exercise it.

---

## Post 1 — The Ethernet broadcast address is technically a multicast address

`ff:ff:ff:ff:ff:ff` is, strictly speaking, just a special multicast address.

The IEEE 802 spec assigns two structural bits to the first octet of every MAC address. Bit 0 (the LSB) marks whether the address is multicast: set means "group destination, every host on this segment subscribed to this group should accept the frame," clear means "individual destination, only this one host." Since `0xff = 0b11111111`, the broadcast address has bit 0 set, which makes it a multicast address with every host on the segment as a member. This is why `ip maddr show` lists `ff:ff:ff:ff:ff:ff` alongside IPv4 multicast groups like `224.0.0.1`. The same bit governs all `01:00:5e:xx:xx:xx` IPv4 multicast addresses (the entire `01:00:5e` prefix has bit 0 set in the first octet) and `33:33:xx:xx:xx:xx` IPv6 multicast. An Ethernet driver's multicast filter is a Bloom filter over the bottom 23 bits of any address with bit 0 set; broadcast falls through it for free.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The Ethernet post implements the 14-byte header parser and the MAC Addr type with `IsMulticast` and `IsBroadcast` methods that test exactly this bit.

Find the full Ethernet layer breakdown in the comments below 👇

---

## Post 2 — The EtherType field is also a length, depending on its value

The 13th and 14th bytes of every Ethernet frame are simultaneously a protocol ID and a length, depending on their value.

Original 1980 Ethernet didn't have an EtherType field — bytes 13–14 carried the length of the payload, in bytes, ranging from 46 (the minimum legal payload) to 1500 (the maximum). When TCP/IP took over the wire and needed to distinguish protocols, the IEEE picked a clever extension: any value at or above 0x0600 (1536 decimal) — which can't possibly be a legitimate payload length because it exceeds 1500 — is reinterpreted as a protocol identifier. So 0x0800 is IPv4, 0x0806 is ARP, 0x86DD is IPv6, 0x8100 is a 802.1Q VLAN tag. The disambiguation happens at reception: if the value is ≤ 0x05DC (1500), the frame is parsed as classic 802.3 with a length-followed-by-LLC-header; if it's ≥ 0x0600, the frame is Ethernet II with raw payload following. Modern stacks never use the length form. Watch `tcpdump -e` and every line reports `ethertype IPv4 (0x0800)` — pure protocol IDs, length implied by the IP header.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The Ethernet post implements the parser as a strict Ethernet II decoder — values below 0x0600 are explicitly listed as a non-goal.

Find the full Ethernet layer breakdown in the comments below 👇

---

## Post 3 — 802.1Q VLAN tags insert 4 bytes in the worst possible place

A 802.1Q VLAN tag adds 4 bytes to an Ethernet frame between the source MAC and the EtherType — exactly where every parser least wants them.

Original Ethernet II frames are `[dst MAC | src MAC | EtherType | payload]`, 14 bytes of header. With VLAN tagging (managed switches, virtio-net, OpenStack tenant networks), the frame becomes `[dst MAC | src MAC | 0x8100 | TCI | EtherType | payload]`, 18 bytes of header. The EtherType field is overwritten with 0x8100 (the VLAN sentinel), and the *real* EtherType is pushed back 4 bytes behind the TCI — Tag Control Information: 3 bits of priority (PCP), 1 bit DEI, 12 bits of VLAN ID. A parser written for plain Ethernet II sees 0x8100 and either skips 4 bytes and re-reads the EtherType, or drops the frame. Hardware does this in silicon; userspace stacks have to handle it explicitly. Linux exposes per-VLAN interfaces as `eth0.10` (VLAN ID 10 on eth0), and `ip -d link show` reports the VLAN ID for tagged interfaces. The 12-bit VID gives you 4094 usable VLANs (0 and 4095 are reserved).

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The Ethernet post implements a strict Ethernet II decoder — VLAN-tagged frames are listed as a deliberate non-goal in the limitations table.

Find the full Ethernet layer breakdown in the comments below 👇

---

## Post 4 — Bit 1 of the first octet says "I made this MAC up"

Bit 1 of the first octet of a MAC address says whether the address was assigned by a vendor or made up on the spot.

Cleared (0): the address is globally unique, registered with the IEEE to a specific vendor via the 24-bit OUI in the first three octets. You can look up the manufacturer with any OUI database — Apple's 14:7d:da, Intel's 00:1c:42, VirtualBox's 08:00:27. Set (1): the address is locally administered, meaning whoever assigned it made it up and is responsible for uniqueness within their broadcast domain. This is the bit that the conventional `02:xx:xx:xx:xx:xx` "crafted MAC" prefix sets — `02 = 0b00000010` has bit 1 set and bit 0 (multicast) clear, marking it as a unicast, locally administered address. Docker uses `02:42:xx:xx:xx:xx` for container interfaces; QEMU uses `52:54:00:xx:xx:xx`; Linux bridges use a locally-administered address derived from their UUID. Changing your laptop's MAC for privacy on public WiFi — `sudo ip link set wlan0 address 02:00:00:11:22:33` — should always set this bit, or you risk colliding with a real vendor address.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The Ethernet post implements the MAC Addr type as a `[6]byte` array so it's a comparable value type and works as a map key for the ARP cache.

Find the full Ethernet layer breakdown in the comments below 👇

---

## Post 5 — The FCS is 4 bytes you never see

Every Ethernet frame on the wire ends with 4 bytes of CRC32 that your software will never see.

The Frame Check Sequence (FCS) is computed by the transmitting NIC over the entire frame and appended to the end. Every receiver — including switches — recomputes it on arrival; if the computed and received FCS disagree, the frame is dropped at the hardware level before any driver, kernel, or `tcpdump` sees it. The dropped count surfaces in `ip -s link show` as `errors` on the RX side, or in `ethtool -S eth0 | grep -i crc` as `rx_crc_errors` if your NIC exposes per-counter detail. On a healthy LAN with good cables, you should see zero CRC errors over weeks; a non-zero count is the first signal of a flaky transceiver, a damaged cable, or an EMI source near the run. NICs strip the FCS *before* delivering the frame to the kernel, which is why every Linux network stack treats frames as 14-byte header + payload with no trailer, and why `tcpdump`'s per-frame byte count is always 4 less than what the wire actually carried. The 4 missing bytes are real — they just lived for less than a microsecond.

I've been writing a "Building a TCP/IP stack in Go" series where I implement each layer from scratch on top of TUN/TAP. The Ethernet post calls out the FCS as a deliberate non-concern of the codec — the NIC and TUN/TAP driver both strip it before any byte reaches our parser.

Find the full Ethernet layer breakdown in the comments below 👇
