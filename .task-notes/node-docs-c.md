# Node description reading notes: transport, adapters, and tests

Read completely from the 27 source descriptions listed below. These are navigation
notes, not new contracts. Source descriptions remain authoritative. L3 means full
IP packets on persistent worker packet lines; L4 means normal connection lines.

| Node | Responsibility and boundary | Pairing / lifecycle constraints |
| --- | --- | --- |
| SpeedTestClient | L4 synthetic chain head; owns configured normal test lines, emits framed upload/download/bidirectional tests and reports throughput. | Pair SpeedTestServer across TCP/UDP; transport Est starts HELLO; UDP waits for ACK and reports loss; not an application proxy. |
| SpeedTestServer | L4 borrowed-line chain end; responds Est, validates HELLO/DATA/END, returns reports and optional downloads. | Only SpeedTestClient peer; does not destroy borrowed line; supports optional deterministic verification. |
| StreamToPackets | L4 previous side to L3 next side; parses back-to-back IPv4 total lengths, emits one packet at a time; return packets select a validated stream. | Pair PacketsToStream; IPv4 only; globally newest validated source IP wins, same-source line pool uses per-flow rendezvous; paused carriers excluded; never destroy borrowed or packet lines. |
| TcpConnector | L4 outbound tail; destination selectors/weighted choices, async DomainResolver, TCP socket, downstream Est on connect. | Borrows line; target may come from src/dest context; 60s connect deadline, ~600s idle; bounded pre-connect/write backlog. |
| TcpListener | L4 inbound head; accepts sockets, owns one normal line each, fills source metadata, forwards upstream. | Source context port is local listener port; real peer port has separate peer_source_port; ACLs/balancing/interface scope; initial/active idle deadlines and bounded output backlog. |
| TcpOverUdpClient | L4 stream-to-datagrams using KCP, optional Reed-Solomon FEC; adds internal data/ping/close flag. | Pair TcpOverUdpServer across datagram-preserving transport; FEC settings must agree; no raw TCP header disguise; MTU is IPv4-budget based; blocks splice. |
| TcpOverUdpServer | Inverse KCP datagram-to-stream transform with keepalive/timeout and optional FEC. | Pair client; previous side must preserve datagrams; soft KCP queue pressure is advisory, not a hard byte cap; borrowed normal lines. |
| TcpUdpConnector | L4 chain-end wrapper containing TcpConnector and UdpConnector, sharing settings. | Exact destination TCP/UDP protocol wins, otherwise source protocol; stores selected child during Init; never destroys line. |
| TcpUdpListener | L4 chain-head wrapper with both TCP and UDP listeners sharing settings, forwards to common next. | Children own normal lines; source protocol selects return child; standard SOCKS5 UDP entry because it delegates typed dynamic endpoint capability. |
| Template | No-op L4 middle relay and standard instance/upstream/downstream/common/include skeleton. | Developer scaffold; no real state/settings; constructors, padding, finish and callback-reference rules apply when extended. |
| TesterClient | Synthetic integrity-test head; normal mode owns a line per worker; packet mode uses persistent worker packet lines. | Pair TesterServer; deterministic 11 chunks, optional raw IPv4/TCP/UDP/ICMP generation, stateless reordered packet checking; 30s watchdog; packet lines never close. |
| TesterServer | Synthetic integrity-test tail; verifies requests and returns deterministic responses, optionally overlaps streams. | Borrows normal lines; packet mode can accept either direction; packet finish fatal; supports matching IPv4 transport test generation. |
| TlsClient | L4 client TLS transform using prefixed bundled BoringSSL; real certificate validation, SNI, ALPN, record encoding/decoding, optional ECH/shaping/fragment helper. | No sockets; ordinarily before TcpConnector; transport Est is not TLS-ready; preceding HttpClient must match configured ALPN; retained ordinary buffers and whole-chain splice block. |
| TlsServer | L4 server TLS transform using OpenSSL; cert/key, ALPN/SNI, optional pre-ServerHello fallback, record shaping. | No HTTP awareness or sockets; normally after TcpListener; downstream Init forbidden; fallback only before TLS commitment; server clean close drains close_notify. |
| TrojanClient | L4 Trojan request/auth transform; TCP passes opaque after setup, UDP borrows application line and owns one TCP carrier with per-datagram address framing. | Add TlsClient separately for ordinary deployment; connector targets server, request target is final destination; first payload may precede Est; no request replay. |
| TrojanServer | Authenticates SHA224 password, parses CONNECT or UDP ASSOCIATE; local allowlist or AuthenticationClient plus internal UserController. | Add TlsServer separately; first callback must contain full 56-byte hash; fallback only before authentication; owns UDP backend per requested destination, borrows carrier. |
| TunDevice | L3 TUN adapter; full IPv4 packets enter/leave chain, worker packet lines; native route/DNS/egress-pin support. | Edge either side; fragment reassembly default; L3/L4 stack bridges require complete datagrams; Linux direct PTC pair has private checksum/GSO/enqueue optimization; packet Finish fatal. |
| UdpConnector | Normal-L4 outbound tail; worker-local sockets, destination selection/balancing, async DNS, registered-peer reply routing. | Cannot accept packet lines directly; explicit bridge required; emits Est when local resources ready; paused inbound replies drop; destination source ports are not line identifiers. |
| UdpListener | L4 UDP head; owns pseudo-connection line per peer, forwards each datagram upstream and replies to remembered endpoint. | First packet creates/Init line; paused inbound datagrams drop; idle ~30s initial/~300s active; dynamic endpoint provider used by SOCKS5 via TcpUdpListener. |
| UdpOverTcpClient | L4 UDP payload-to-stream transform with two-byte big-endian datagram length prefix. | Pair UdpOverTcpServer; next must carry a byte stream; not fake TCP; borrowed normal lines; zero lengths invalid; outbound max 65,505 bytes. |
| UdpOverTcpServer | L4 inverse length-prefixed stream-to-UDP payload transform; marks destination UDP. | Previous is stream; next ordinarily UdpConnector; each decoded frame stays one datagram; malformed frames/retention exhaustion close borrowed carrier through owner. |
| UdpStatelessSocket | Head-only UDP adapter; owns inbound peer normal lines and supports outbound routing-context sends on borrowed/packet lines. | Intended transport edge for WireGuardDevice; must have next and no prev; one bound socket also initiates peer traffic; cross-worker sends post to I/O worker. |
| VlessClient | L4 plain VLESS v0 UUID request/response transform; TCP and fixed-target UDP over owned TCP carrier. | TLS/REALITY/HTTP/Mux separate; UDP uses two-byte lengths including empty datagrams; bypass TCP only after both request and response setup settle; borrows application line. |
| VlessServer | L4 VLESS v0 UUID auth/request parser; TCP or fixed-destination UDP backend; local allowlist or managed internal UserController. | Full version+UUID must be first callback; no addons/Vision/mux; fallback only unauthenticated; first real reply combines response header; owns UDP backend, borrows carrier. |
| VmessClient | L4 modern VMess AEAD request/response/body codec; TCP or fixed-target UDP/packetaddr over TCP carrier. | TLS/HTTP separate; codec ordinary-only and blocks splice; explicit security, masking/padding/counter policy; default wrap repeats AEAD nonces for interoperability; no reconnect/replay. |
| VmessServer | L4 VMess AEAD auth/codec server; local/managed UUID accounts, replay tables, bounded auth work, native or per-destination packetaddr UDP backends. | TLS/HTTP separate; owned UDP backends, borrowed input carrier; unknown credentials may fallback, authenticated invalid requests never do; whole-chain splice block. |
| WireGuardDevice | Packet/transport bridge performing WireGuard encryption, peer routing, AllowedIPs checks, handshakes/rekey/keepalive; owns normal per-worker transport companions. | No socket/OS WG interface; layer solver requires exactly one L3 side and one L4 side; head UdpStatelessSocket is supported transport, UdpListener is not; inner packet lines remain persistent. |

## Reusable implications for fake TCP

- UDP application payloads start naturally at UdpListener normal owned peer lines.
- UdpConnector needs normal lines on the destination side; packets need
  PacketsToConnection conversion before it. Full IP packets cannot be sent straight
  into UdpConnector as if they were application datagrams.
- UdpOverTcp is real stream-carried length framing, and TcpOverUdp is KCP. Neither
  constructs fake TCP wire headers.
- Packet nodes must retain packet line lifetime; transforms are not line owners.
- A fake TCP path must keep complete-datagram/fragment behavior compatible with
  local-stack bridges, and preserve one application UDP datagram per decoded body.

## Files read

1. `tunnels/SpeedTestClient/description.md`
2. `tunnels/SpeedTestServer/description.md`
3. `tunnels/StreamToPackets/description.md`
4. `tunnels/TcpConnector/description.md`
5. `tunnels/TcpListener/description.md`
6. `tunnels/TcpOverUdpClient/description.md`
7. `tunnels/TcpOverUdpServer/description.md`
8. `tunnels/TcpUdpConnector/description.md`
9. `tunnels/TcpUdpListener/description.md`
10. `tunnels/template/description.md`
11. `tunnels/TesterClient/description.md`
12. `tunnels/TesterServer/description.md`
13. `tunnels/TlsClient/description.md`
14. `tunnels/TlsServer/description.md`
15. `tunnels/TrojanClient/description.md`
16. `tunnels/TrojanServer/description.md`
17. `tunnels/TunDevice/description.md`
18. `tunnels/UdpConnector/description.md`
19. `tunnels/UdpListener/description.md`
20. `tunnels/UdpOverTcpClient/description.md`
21. `tunnels/UdpOverTcpServer/description.md`
22. `tunnels/UdpStatelessSocket/description.md`
23. `tunnels/VlessClient/description.md`
24. `tunnels/VlessServer/description.md`
25. `tunnels/VmessClient/description.md`
26. `tunnels/VmessServer/description.md`
27. `tunnels/WireGuardDevice/description.md`
