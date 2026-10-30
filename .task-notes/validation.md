# Fake TCP validation reference

Task notes, not runtime contracts.

The earlier results below describe the initial UDP-header conversion. The final
full-inner-IPv4 carrier, exact TCP port filters and TUN support are covered in
the follow-up results at the end of this file. The earlier broad VMess deadline
failures remain recorded; that broad lane has not been repeated for this delta.

## Completed locally

- Windows MinGW GCC 14.2, `windows-ninja`, focused
  `waterwall.fake_tcp_header_unit` and `waterwall.rawsocket_config_unit`:
  both pass in Debug and optimized Release.
- The validation tree has `WW_BUILD_UNIT_TESTS=ON`, `WW_ENABLE_IPO=OFF`,
  `WW_PACK_RUNTIME=OFF`, and `OPENSSL_NO_ASM=ON` because NASM is absent.
- clang-format 14 passes for all new C/headers and modified legacy ranges.
- JSON parsing and named `next` references pass for both examples and the
  deterministic integration case. Notes inventory covers all 81 original node
  descriptions.
- Source checks passed: abort policy, orderly shutdown policy, worker identity
  policy, and tunnel callback dispatch policy. Python probe syntax passes.
- The full Windows Release application build is inconclusive: unchanged
  `tunnels/Internals/VmessCommon/src/crypto.c:103` fails GCC's array-bounds check
  in `kdfHash()` (`spans[6]`, inlined `memcpy`). The focused targets build and
  run independently of that failure. No VMess files were changed.

## Ubuntu validation location

The user supplied an Ubuntu test server. Source and pinned dependencies are in
`/tmp/waterwall-fake-tcp.NdHtbM/`; the complete extracted source is `linux-src`.
The temporary job script `run-tests.sh` writes phase updates to `linux-job.log`,
its enclosing exit status to `linux-job.exit`, and separate logs per build/test.
Credentials are not stored in this repository.

The server has two Xeon E5-2670 v2 CPU cores and about 3.8 GiB RAM. Build tools
were installed. Network namespace and iptables checks passed. clang-format 18
passed. GitHub HTTP requests timed out, so the already-fetched pinned sources
were transferred and selected through CMake cache/source overrides.

The `linux` and `linux-unit-tests` presets retain their separate trees. Both
use the existing older-CPU settings: `WW_ADVANCED_CPU_BASELINE=nehalem`,
`WW_ADVANCED_CPU_EXTRA_FLAGS=-mtune=generic`, `WW_DISABLE_AVX=ON`. Production
retains LTO. Build concurrency is two. QUIC client/server are disabled only in
the Ubuntu test configuration to keep the two-core build focused; their guarded
test registrations are consequently absent. Repository defaults are unchanged.

## Earlier Ubuntu results

- Production Release application built successfully with LTO and QUIC disabled.
- The direct `fake_tcp_header_udp_roundtrip` and existing
  `connection_to_packets_udp_bridge_roundtrip` passed.
- Both `capture_linux_protocols_bypass` and `capture_linux_protocols_tracked`,
  plus `raw_linux_notrack`, passed.
- The initial maximum-size raw exchange exposed inherited IpManipulator MTU
  segmentation. Explicit carrier `mtu: 1500` fixes the examples; the raw probe
  then passed, including 1456-byte payloads with four bytes of TCP options,
  both directions, checksums/flags, unrelated UDP and firewall-rule cleanup.
- All five focused native tests passed in both Debug and optimized Release:
  fake TCP, RawSocket configuration, Linux capture command construction,
  RawSocket startup failure and write-only startup.
- The broader functional pass ran 533 cases: 395 passed; 130 lacked the pinned
  independent proxy reference binaries and eight had CRLF expected-report files
  from the Windows source snapshot. The Linux fixture copies were normalized to
  canonical LF and exact checksum-pinned reference archives were fetched. Only
  the 138 affected cases were rerun: 131 passed and seven VMess longevity cases
  exceeded their existing 65-second workload deadlines. Thus **526 of 533
  functional cases passed**, with no feature-test failures. QUIC was excluded
  from this test build. The broad gate is not fully green on this host.

The seven remaining failures concern unchanged VMess paths, with intact
exchanges until the fixed deadline (completed / required):

| Case suffix after `waterwall.` | Exchanges |
| --- | ---: |
| `vmess_long_aes_128_gcm` | 50,757 / 65,540 |
| `vmess_long_chacha20_poly1305` | 53,966 / 65,540 |
| `vmess_strict_counter` | 50,422 / 65,535 |
| `vmess_long_zero_tls` | 53,260 / 65,540 |
| `vmess_client_long_aes_128_gcm` | 57,447 / 65,540 |
| `vmess_client_long_chacha20_poly1305` | 53,552 / 65,540 |
| `vmess_client_strict_counter` | 56,376 / 65,535 |

No deadlines or workloads were weakened, and these deadline failures were not
retried. They leave the VMess longevity gate unverified; they are not successful
counter-wrap evidence. A host that can complete those unchanged workloads is
needed for a clean broad result. No VMess source was modified.

The retained remote logs are under `/tmp/waterwall-fake-tcp.NdHtbM/`.
Local copies are in `E:/projects/WaterWall/build/ubuntu-validation/`.
`linux-functional.log` records the first run, `linux-functional-rerun.log` the
environment-corrected rerun, `linux-raw-fixed.log` the successful maximum-size
raw exchange, and `linux-unit-{debug,release}-test.log` the native results.
The initial job and final rerun exit codes are nonzero because of the failures
described above. Production and focused native builds succeeded.

The developer-guide checkout and the English/Persian RawSocket MDX mirrors
referenced by existing documentation are absent locally; those mirrors were not
updated. The new node description and source-node descriptions contain the
implemented feature's configuration and constraints.

## Follow-up: complete IPv4 carrier, exact ports and TUN

Final retained behavior wraps the complete inner IPv4 packet in a separate
IPv4/TCP carrier. Both receive endpoints use peer source IP plus protocol 6 and
configurable TCP source/destination ports. Kernel qualifiers also apply to
incoming NOTRACK, with identical predicates for insertion and deletion.

Confirmed results on the final production code:

- Windows focused codec/configuration tests pass in Debug and Release.
- All 17 changed C/header files pass clang-format 14 (modified ranges for legacy
  files). Example JSON, probe Python syntax and `git diff --check` pass.
- Ubuntu production Release builds with LTO, the same old-CPU settings and
  QUIC-disabled validation scope described above.
- All nine focused integration cases pass: direct fake UDP bridge, real raw
  fake-TCP UDP, real raw fake-TCP TUN, existing UDP bridge, protocol bypass and
  tracked modes, capture NOTRACK and tracked modes, and raw NOTRACK.
- Raw and TUN cases each pass three consecutive repetitions after fixing test
  readiness. Both verify ordinary real TCP request/reply from the same peer IPs
  on other ports, unrelated UDP delivery, outer/inner checksums, flags/options,
  multiple users, exact UDP bytes/reply tuples and firewall-rule cleanup.
- With four TCP option bytes, both cases carry 1428-byte UDP payloads in a
  1500-byte carrier. TUN reports MTU 1456 and rejects a 1429-byte UDP payload
  with EMSGSIZE and DF enabled. This exercises the actual kernel route MTU.
- Five focused native cases pass in both Linux Debug and optimized Release:
  fake TCP, capture commands/filter, RawSocket config, startup failure and
  write-only startup. The filter fixture executes generated cBPF through
  SO_ATTACH_FILTER, including either/both port qualifiers, wrong tuples,
  IPv4 options, malformed/truncated lengths, fragments, DF and exclusions.
- All nine source-policy checks pass. A read-only code review found no blocking
  defects. No real Windows WinDivert capture runtime was exercised.

Two test-harness issues were corrected without weakening workloads: archive
timestamps initially allowed stale objects, and readiness initially used a
host sysfs view for TUN and did not wait for the UDP listener bind. Extraction
now refreshes source times; TUN readiness queries namespace-local `ip -j`, and
UDP readiness waits for the bound-listener log before sending any request.

Final Ubuntu evidence is retained remotely under the same temporary directory
and copied to `build/ubuntu-validation/`: `followup-build.log`,
`followup-integration-final.log`, `followup-repeat.log`,
`followup-unit-{debug,release}-test.log`, and `followup-source-policy.log`.
The earlier broad VMess deadline failures and full Windows application build
limitation remain outside this focused delta; neither is claimed green.
