# LinkedIn post — Series announcement

I shipped a small networking project called `neth`, and the questions I got back surprised me.

They weren't about my code. They were about the network itself.

These came from good engineers. People who ship features every day, call APIs over HTTPS, debug timeouts, scale services. And yet the layer underneath all of it - the thing every request rides on - was a black box. `connect()` goes in, bytes come out, and the middle is magic.

I'm writing a series: **Building a TCP/IP stack**, layer by layer, from the bottom up. Not a textbook - a practical, code-first walkthrough where every layer is something you can read, run, and break.

The plan, bottom to top:
→ Loopback device - a virtual NIC that talks to itself
→ TUN/TAP - getting real packets into userspace
→ Ethernet - framing raw bytes into something addressable
→ ARP - how IP addresses find MAC addresses
→ IP - routing and fragmentation
→ ICMP - what `ping` is really doing
→ UDP - the minimal transport
→ TCP - connections, ordering, retransmission, the hard part

I'm not a networking expert, and that's the point. 

First post is live: a userspace **loopback device** - the very first piece, the thing every higher layer sends bytes into before any of the fancy stuff exists. No root, no raw sockets, just a Go channel pretending to be a network card.

Link: https://rykth.github.io/posts/building-a-loopback-device/

#golang #networking #tcpip #softwareengineering
