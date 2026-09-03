# `/sbin/switch` port mirroring — sustained throughput proof

Live measurement of `/sbin/switch mirror monitor`/`mirror target` (see
`SWITCH_TOOL.md`/`SWITCH_TOOL_MAN.md` for what the tool does and how it
was reverse-engineered) pushed to real, sustained near-gigabit throughput
on ER-X hardware, to answer the question those two docs leave open: does
this actually work at line rate, or just for light/incidental traffic
like a ping?

## Setup

- **Topology**: two hosts on two different VLANs (`Host A` on VLAN 10 /
  switch port 1, `Host B` on VLAN 20 / switch port 2), routed through the
  ER-X — the same cross-VLAN inter-port path any "monitor what's
  transiting this router" use case would actually capture. A third host
  (`Capture`) sits on switch port 3, with no L3 address of its own — a
  bare port, used purely as the mirror destination.
- **Mirror config** — one source port, one direction, no doubling:
  ```
  /sbin/switch mirror monitor 3     # port 3 = monitor/destination
  /sbin/switch mirror target 1 1    # port 1 (Host A's VLAN), RX only
  ```
  `mirror target <port> 1` = mirror only traffic *entering* the router on
  that port (mode `1` = RX). This models the realistic case — e.g.
  watching everything a given LAN segment sends toward the router (and
  from there, potentially on to the internet) — rather than an
  artificially doubled rx+tx-of-multiple-ports scenario that isn't
  representative of a normal single-SPAN deployment.
- **Traffic generator**: `iperf3`, 8 parallel TCP streams, `Host A` →
  `Host B`, run for both 20s and 30s windows.
- **Measurement — two independent methods, cross-checked**:
  1. `iperf3`'s own reported throughput (`Host A`↔`Host B`) — the
     underlying flow's real TCP-payload rate.
  2. `ethtool -S <capture-if>` on `Capture`, sampled immediately before
     and after each run — a **hardware MAC-level RX byte counter** on
     the capture NIC itself, plus its `rx_missed_errors`/
     `rx_no_buffer_count` fields to check for capture-side drops.
     This is the authoritative number for what the mirror actually put
     on the wire, independent of anything the capture software did with
     it.
  3. `tshark`/`capinfos` against the raw `tcpdump` capture, as a sanity
     check on which source addresses actually appear (confirming a
     single-port capture, not a mix).

## Results

| Run | Duration | Source flow (iperf3, TCP payload) | Mirror port (ethtool RX, wire rate) | Capture-side drops |
|---|---|---|---|---|
| 1 | 20s | 931.8 Mbps sent / 929.1 Mbps received | **≈973.9 Mbps** | 0 |
| 2 | 30s | 932.1 Mbps sent / 929.7 Mbps received | **≈974.5 Mbps** | 0 |

Both runs: `rx_missed_errors` and `rx_no_buffer_count` deltas on the
capture NIC were **zero** — every byte that arrived was actually
counted, nothing was silently lost to a swamped capture. `tshark`
confirmed the pcap's only real traffic source was `Host A`'s address —
a clean single-port capture, not summed traffic from multiple ports.

**Why the mirror rate (≈974Mbps) is slightly higher than iperf3's own
number (≈930Mbps)**: `iperf3` reports TCP-payload bytes only; the
mirror captures full Ethernet frames off the wire, including Ethernet/
IP/TCP headers — roughly 4.6% overhead for standard 1500-byte-MTU
segments. 931.8 × 1.046 ≈ 974.4Mbps, matching the hardware-counter
figure almost exactly. This is the expected relationship between
app-level and wire-level throughput, not an anomaly — good independent
confirmation the two measurement methods agree with each other.

The two runs (20s and 30s) landed within 0.6Mbps of each other — this is
a **sustained**, reproducible number, not a short-lived burst.

## Findings

1. **Port mirroring is real, functional, and line-rate-capable.** A
   single mirrored port/direction reproduces its source flow's true
   wire-level throughput (≈974Mbps, matching the source's ≈930Mbps
   TCP-payload rate plus normal framing overhead), with **zero measured
   packet loss**, sustained over a 30-second window. This is close to
   the realistic practical ceiling for 1000BASE-T (~940-980Mbps, below
   the 1000 nominal) — not a synthetic/incidental result, an actual
   near-saturation throughput test.
2. **Mirroring happens entirely in the switch ASIC, not the CPU.** The
   router's own CPU sat at 97-98% idle throughout every run, and the
   monitor port's own vyatta-visible interface counters stayed at 0
   bytes/0 packets despite carrying nearly a gigabit/sec of real
   mirrored traffic — the switch chip duplicates frames entirely below
   Linux's netdev/vyatta accounting layer. Consistent with (and now
   throughput-proven, not just structurally inferred from)
   `SWITCH_TOOL_MAN.md`'s documented claims about how this feature
   works.
3. **No software-visible drop counter exists for oversubscription on
   the monitor port itself.** The capture NIC's own counters only
   report what its receive path did with whatever physically arrived;
   they can't distinguish "the switch ASIC dropped frames before
   transmitting them" from "nothing was oversubscribed." That
   distinction would need chip-register-level inspection — out of scope
   here (see `SWITCH_TOOL_MAN.md` CAVEATS on `reg`/`phy` — no
   documented-safe register map exists for this chip).

## Scope and open questions

- This tested **one port mirrored in one direction** (RX). Whether TX-only
  behaves identically wasn't separately verified — reasonable to assume
  symmetry given the architecture, but not empirically checked.
- This used ordinary inter-VLAN routed traffic as the source flow, not
  NAT'd/WAN-bound traffic specifically. If your use case is mirroring
  traffic that's also undergoing NAT (e.g. actually watching what leaves
  toward the internet, post-NAT), that exact path wasn't tested here and
  could behave differently depending on how this chip's hardware-NAT
  offload interacts with the mirror feature.
- Mirroring *multiple* ports/directions into one destination
  simultaneously was not the scenario measured here, and shouldn't be
  assumed to scale linearly from this single-port result — funneling
  more than one port's traffic into one monitor port will contend for
  that monitor port's own physical bandwidth (a straightforward
  consequence of it still being a normal 1Gbps port), separate from
  whatever internal cost the mirroring feature itself may have under
  heavier configurations.
