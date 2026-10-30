# WaterWall architecture reference

Source map from all 81 existing node `description.md` files. The complete reading
inventory and per-node responsibility tables are in [A](node-docs-a.md),
[B](node-docs-b.md), and [C](node-docs-c.md). These notes support this task; they
do not add contracts. The developer-guide checkout referenced by AGENTS.md is
absent from this workspace.

## Runtime and boundaries

`core/imported_tunnels.c` registers compiled node factories. The root CMake file
selects nodes; each node has its own static library, public `interface.h`, private
`structure.h`, and instance/upstream/downstream/common implementations. NodeManager
solves layer boundaries before indexing and preparing chains.

Upstream travels to `next`; downstream travels to `prev`. Normal lines represent
streams or UDP associations. Packet lines belong to the chain, one per worker,
and survive individual packet drops. Callbacks transfer sbuf ownership and may
destroy normal lines. A transform owns neither kind of line. Worker pools must
only be accessed on their owning worker.

Listeners own incoming normal lines; connectors borrow them. Layer-4 protocol
nodes negotiate, frame, encrypt, multiplex or route those lines. RawSocket and
TunDevice instead publish full IP packets on worker packet lines. Bridge connects
branches without changing the exact line; it does not perform transport decoding.
Stream framing alone does not create actual TCP/IP packets.

## Fake TCP composition

Client, upstream encoding:

```
UdpListener -> ConnectionToPackets -> FakeTcpHeader(client)
            -> IpManipulator(up-tcp-bit-*) -> RawSocket
```

Server, upstream decoding (return encoding travels downstream):

```
RawSocket -> IpManipulator(dw-tcp-bit-*) -> FakeTcpHeader(server)
          -> PacketsToConnection -> UdpConnector
```

UdpListener owns each client association's normal line. ConnectionToPackets
borrows that line and creates inner UDP packets using configured virtual IPv4
endpoints. Alternatively, TunDevice publishes the original IPv4 packet directly
into FakeTcpHeader. FakeTcpHeader and IpManipulator use only chain-owned
packet lines. PacketsToConnection owns the server's reconstructed UDP normal
lines; UdpConnector borrows them and sends payload to the configured destination.
Replies traverse the same mapping in reverse. No fake TCP handshake, reliable
delivery or real TCP stream is introduced.

IpManipulator already owns flag editing. Its `protoswap` excludes native TCP/UDP
replacement numbers and does not construct headers, so it cannot implement this
conversion by itself. RawSocket's protocol predicate is applied in kernel capture
and incoming NOTRACK rules (Linux cBPF) or WinDivert (Windows). Its allowlist can
reuse the same exclusion bitmap by computing the complement once at startup.

FakeTcpHeader prepends outer IPv4/TCP headers to the complete inner IPv4 packet,
preserving its addresses, transport ports, options and payload without copying.
Outer local/peer IPv4 addresses are required; outer ports are configurable and
independent of the inner transport tuple. Both RawSocket endpoints constrain
capture to protocol 6, peer source IP, peer TCP source port and local destination
port. A bounded cBPF predicate filters before NFQUEUE and NOTRACK; WinDivert uses
TCP port field predicates. Ordinary TCP on other ports remains with the host.

The device packet ceiling is 1500 bytes. Overhead is 40 bytes plus configured TCP
options. Inner MTU is carrier MTU minus that overhead: 1460 without options,
1456 with four option bytes. Subtract another 28 for ordinary IPv4/UDP payloads.
Set core/ConnectionToPackets/TunDevice MTUs explicitly. Inner fragments survive
encapsulation for a TUN peer, but local-stack consumers require complete IPv4
datagrams; device reassembly differs from bridge output fragmentation.

Set carrier-side IpManipulator `mtu: 1500` explicitly when the bridge/core MTU
reserves fake-header overhead. Its normal TCP egress segmentation otherwise
splits a maximum-size carrier and destroys the original UDP datagram boundary.
