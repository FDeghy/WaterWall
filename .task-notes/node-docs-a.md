# Node reading notes A

Read from the 27 `description.md` files listed below, sorted by path, indexes
0–26. These are task reference notes, not additional runtime contracts.

WaterWall composes nodes through directional callbacks. L4 normal lines carry
streams or datagrams; L3 packet lines carry complete IP packets and remain owned
by the chain. A byte stream protocol header does not become an IP/TCP header.
Many stream setup nodes permanently bypass themselves once parsing and replay
finish; packet transforms and nodes mapping separate lines remain active.

| Node | Responsibility and boundaries | Pairing / constraints |
| --- | --- | --- |
| APIServer | L4 terminal control gateway; registered node operations over length-prefixed JSON. Borrows lines; no socket. | Requires reliable stream; exact operator-token permissions independent from traffic-user authentication. UDP requires TcpOverUdp. |
| AuthenticationClient | L4 autonomous control-plane head; owns worker-0 control line, caches authoritative users and reports local usage. Not data pass-through. | AuthenticationServer peer; account consumers use durable ID handles/copy-outs, never raw user pointers. Cached offline admission after successful initial sync. |
| AuthenticationServer | L4 terminal database/control service; owns authoritative users table, revisions, producer sessions, bounded sync snapshots and database saves. | No next or sockets; AuthenticationClient transport must supply reliable stream, and security wrappers for remote secrets. API administration is separately registered through APIServer. |
| Bgp4Client | L4 stream disguise: upstream BGP-like frame encoder with first OPEN, downstream decoder. | Bgp4Server; password accepted but ignored; 39-byte prepend padding. Framing persists, no bypass. |
| Bgp4Server | L4 inverse: validates first OPEN then strips upstream frame envelopes, encodes downstream. | Bgp4Client; 19-byte prepend padding. Both have bounded retention and transparent Pause/Resume. |
| BlackHole | Terminal layer-agnostic sink adapter; passive establishes and discards, active rejects during Init. | Cannot have next; borrows line, never owns/destroys packet line. |
| Bridge | Layer-equivalent relay pairing two named chain locations, passing the same exact line while mapping direction through its peer. | Pair in same config. Reverse-specific adjacency is Bridge -> ReverseClient and ReverseServer -> Bridge, with no inserted gap nodes. |
| ConnectionFisherClient | L4 association from borrowed main line to multiple owned candidate lines; probes FISH?, keeps first validated FISH!. | ConnectionFisherServer; max bounded startup/order queues. Child valid only toward next; original valid only toward prev. |
| ConnectionFisherServer | L4 probe decoder/responder; delays backend Init until valid probe and first app data. | ConnectionFisherClient; after startup/FIFO completion, destroys local state and bypasses. Losing candidates create no backend. |
| ConnectionToPackets | L4 -> L3 active-open bridge through worker-local lwIP; borrowed normal line becomes TCP/UDP IPv4 packets on chain packet line. | source-ipv4 required; target/port may come from context. Reverse PacketsToConnection. IPv4 only by design, UDP outgoing fragmentation supported; return-side fragments must already be reassembled. |
| Disturber | Same-layer test middle node generating loss, corruption, duplicates, delays, disorder, hangs and closes. | Test-only; no disturbance by default, blocks splice. |
| EncryptionClient | L4 upstream AEAD encryption/downstream decryption in TLS-shaped records. | EncryptionServer with same algorithm/password/KDF; 17-byte padding, blocks splice. |
| EncryptionServer | L4 upstream AEAD decryption/downstream encryption. | EncryptionClient; authentication failure closes connection, blocks splice. |
| HalfDuplexClient | L4 association splitting borrowed main into owned upload/download children; 17-byte role + random pair-ID intro. | HalfDuplexServer; first app input triggers intros; pair ID is metadata, not authentication. Remains active to map lines and pressure. |
| HalfDuplexServer | L4 association matching borrowed half transports, owning reconstructed main line toward next. | HalfDuplexClient; PipeTunnel brings halves to one worker, bounded waiting upload and startup FIFO. |
| HeaderClient | L4 one-time metadata prefix: 2-byte port, 1-byte TCP/UDP selector, or PROXY v1/v2. | HeaderServer or PROXY-aware backend; protocol selection snapshots during Init; datagram boundaries require separate framing. 108-byte padding, bypass after first send. |
| HeaderServer | L4 metadata reader applying destination port/protocol or PROXY source metadata before backend Init. Numeric port override consumes nothing. | Corresponding HeaderClient mode. PROXY is trusted-input IPv4 TCP only. Bypass after ordered initial replay. |
| HttpClient | L4 HTTP/1.1, HTTP/2, h2c and WebSocket body transport; encodes requests, decodes response bodies. | HttpServer or compatible body peer. TLS separate; explicit HTTP version must agree with following TlsClient ALPN. Split mode owns upload/download children. Blocks splice. |
| HttpProxyClient | L4 outgoing HTTP/1.1 proxy interface; CONNECT opaque relay or configured single HTTP body exchange. | Transport connector addresses proxy; target-address/port address origin. TLS after node protects proxy link. No UDP, HTTP/2, pooling or retry. |
| HttpProxyServer | L4 incoming HTTP forward proxy/CONNECT; borrows source and owns outbound child, selecting origin from request. | TcpListener before; compatible outbound chain after. Authentication optional via no-auth/local users/tracked AuthenticationClient; tracked mode inserts UserController. |
| HttpServer | L4 inverse HTTP body transport; parses requests, forwards bodies, formats replies. | HttpClient; TLS separate. Single mode body end is not line Finish; split upload end is. Blocks splice. |
| DomainResolver | Same-layer middle helper resolving dest_ctx domain before delayed Init; no new lines or payload mutation. | Often internal to connectors/protocol clients; bounded FIFO during async DNS, then bypass on normal lines. |
| StreamFragmenter | L4 timed/counter-limited upstream range delays and optional valid TLS ClientHello record splitting. | Does not promise TCP wire packet boundaries. Default wait-for-est; one bounded FIFO/timer, then bypass when shaping settles. |
| UserController | L4 normal-line user admission, quota, expiry, per-user bandwidth and local connection/IP reservations; stream or datagram mode. | Needs AuthenticationClient and already attached user handle; protocol servers insert internal controllers. Never packet lines. Remains active for enforcement. |
| IpManipulator | L3 IPv4 packet tricks: reversible protocol swap, TLS packet transcript tricks, TCP flag rewriting, port ghost trailers, duplication. | Existing flag setters must be reused. Protoswap replacements explicitly forbid 6 and 17, so protoswap-udp:6 cannot implement fake TCP. No normal-flow Init/Finish; packet owner is chain. |
| IpOverrider | L3 IPv4 source/destination rewrites independently upstream/downstream; chance/size gate, incremental checksums. | No sockets/lines. IPv6 values parse but runtime rewriting is disabled. Preserves existing recalc requests. |
| JunkDatagramSender | Same-layer normal-line noise injector before first configured payloads; protocol-shaped junk plus best-effort delayed copies. | Skips junk generation on packet lines; does not construct raw IP or TCP headers. Normal lifecycle passthrough, bounded scheduling ownership. |

## Relevant fake-TCP seams

- `ConnectionToPackets` gives UdpListener data a valid IPv4/UDP envelope while
  retaining datagram boundaries and per-flow return mapping. `PacketsToConnection`
  is the inverse receiver/UDP-flow creator.
- `IpManipulator` exposes `up-tcp-bit-*` / `dw-tcp-bit-*` for cwr/ece/urg/ack/psh/
  rst/syn/fin; values on/off/toggle/flip/switch/packet->flag. Simple rewrites update
  TCP checksums incrementally. Reusing this after fake-TCP construction avoids a
  second flag configuration language.
- Existing ProtoSwap deliberately excludes native TCP/UDP replacement numbers.
  Fake TCP therefore needs an actual TCP envelope and protocol-6 handling, not
  merely changing UDP's IP protocol field.
- `preserve-tcp-bitflags` appends/restores one flags byte according to configured
  action directions. That trailer feature is independent from fake-header
  construction; omit when only shaping the fabricated header.
- All these packet transforms consume/forward buffers, not normal connections;
  the original normal-line owner and chain packet-line ownership must stay intact.

## Source inventory

Each table row corresponds to its exact file:

```
tunnels/APIServer/description.md
tunnels/AuthenticationClient/description.md
tunnels/AuthenticationServer/description.md
tunnels/Bgp4Client/description.md
tunnels/Bgp4Server/description.md
tunnels/BlackHole/description.md
tunnels/Bridge/description.md
tunnels/ConnectionFisherClient/description.md
tunnels/ConnectionFisherServer/description.md
tunnels/ConnectionToPackets/description.md
tunnels/Disturber/description.md
tunnels/EncryptionClient/description.md
tunnels/EncryptionServer/description.md
tunnels/HalfDuplexClient/description.md
tunnels/HalfDuplexServer/description.md
tunnels/HeaderClient/description.md
tunnels/HeaderServer/description.md
tunnels/HttpClient/description.md
tunnels/HttpProxyClient/description.md
tunnels/HttpProxyServer/description.md
tunnels/HttpServer/description.md
tunnels/Internals/DomainResolver/description.md
tunnels/Internals/StreamFragmenter/description.md
tunnels/Internals/UserController/description.md
tunnels/IpManipulator/description.md
tunnels/IpOverrider/description.md
tunnels/JunkDatagramSender/description.md
```
