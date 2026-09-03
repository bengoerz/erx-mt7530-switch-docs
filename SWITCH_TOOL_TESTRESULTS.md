# `/sbin/switch` port mirroring — bidirectional throughput proof

Live measurement of `/sbin/switch mirror monitor`/`mirror target` (see
`SWITCH_TOOL.md`/`SWITCH_TOOL_MAN.md` for what the tool does and how it
was reverse-engineered) pushed to real, sustained near-gigabit
throughput on ER-X hardware — capturing the **full bidirectional**
conversation between two network segments, with **no duplicate frames**.

## Getting this right took two attempts — worth explaining why

The naive way to mirror "both directions of a conversation between two
ports" is to mirror each port in rx+tx (mode `3`):
```
/sbin/switch mirror target 1 3
/sbin/switch mirror target 2 3
```
This is wrong, and it's a non-obvious trap: a packet crossing the router
from port 1 to port 2 gets mirrored **twice** — once as it *enters* on
port 1 (RX) and again as it *leaves* on port 2 (TX). Same packet, two
frames in the capture. Measuring throughput this way silently doubles
(and then some, once both directions are summed) the real number.

The fix is to mirror **both** ports, but consistently in a **single**
direction — either RX on both, or TX on both, never mode `3`, never
mixed:
```
/sbin/switch mirror monitor 3     # port 3 = monitor/destination
/sbin/switch mirror target 1 1    # port 1, RX only
/sbin/switch mirror target 2 1    # port 2, RX only
```
Every packet crosses the router by entering on exactly one physical
port — a port-1→port-2 packet enters on port 1, a port-2→port-1 packet
enters on port 2. Mirroring RX on both ports therefore captures **every**
packet from **both** directions, **exactly once each**, at its unique
point of entry. No duplication, full bidirectional coverage.

## Setup

- **Topology**: two hosts on two different VLANs (`Host A` on VLAN 10 /
  switch port 1, `Host B` on VLAN 20 / switch port 2), routed through the
  ER-X. A third host (`Capture`) sits on switch port 3 with no L3
  address of its own — a bare port, used purely as the mirror
  destination.
- **Mirror config**: as above — port 1 and port 2 both mirrored RX-only
  to port 3.
- **Traffic generator**: `iperf3` in `--bidir` mode (simultaneous
  independent flows in both directions), 8 parallel TCP streams per
  direction, `Host A` ↔ `Host B`, 20-second run.
- **Measurement — three independent checks**:
  1. `iperf3`'s own reported throughput, both directions.
  2. `ethtool -S <capture-if>` on `Capture`, sampled immediately before
     and after the run — a **hardware MAC-level RX byte counter** on the
     capture NIC itself, plus `rx_missed_errors`/`rx_no_buffer_count` to
     check for capture-side drops.
  3. `tshark` against the raw `tcpdump` capture — checking which source
     addresses appear (both hosts should appear, confirming genuine
     bidirectional coverage) and cross-checking total captured bytes
     against the expected combined-flow size (to catch duplication, were
     it present).

## Results

| Metric | Value |
|---|---|
| `Host A → Host B` (iperf3, TCP payload) | 461.9 Mbps sent / 459.8 Mbps received |
| `Host B → Host A` (iperf3, TCP payload) | 462.8 Mbps sent / 460.0 Mbps received |
| Combined TCP payload, both directions | ≈924.7 Mbps |
| Mirror port (ethtool RX, wire rate) | **≈976.7 Mbps** |
| Capture-side drops (`rx_missed_errors`) | 0 |

`tshark` confirmed **both** hosts' addresses present as real traffic
sources in the capture — genuine bidirectional coverage, not one
direction masquerading as "the test."

**Duplication check**: combined TCP-payload bytes both directions ≈
2,312,110,080 bytes over the run. With normal Ethernet/IP/TCP framing
overhead (~4.6% for standard 1500-byte-MTU segments), expected wire-level
bytes ≈ 2,418,467,000. Measured mirror-side bytes (`ethtool -S` delta):
2,447,144,522 — within **1.2%** of the overhead-adjusted expectation, not
the ~2x a duplicated capture would show. This is the actual proof that
the RX-only/both-ports design avoids the duplication the naive rx+tx
approach falls into.

## What happens when the mirror is oversubscribed?

A natural question once you've seen the monitor port running close to its
own ceiling: if the segments being mirrored genuinely push more combined
traffic than the monitor port's 1Gbps can carry, does the mirror drop
frames, or does the *original* traffic between the segments slow down to
keep the mirror copy intact?

Tested directly: ran the identical bidirectional test **with mirroring
disabled entirely** and compared against the mirrored run above.

| | Combined bidirectional throughput |
|---|---|
| Mirror **off** | 924.5 Mbps |
| Mirror **on** (the run measured above) | 924.7 Mbps |

Statistically identical — within 0.2Mbps (<0.03%) across two independent
20-second runs, well inside normal run-to-run noise. **Enabling the
mirror had zero measurable effect on the real transfer**, even with the
monitor port already running near its own realistic ceiling. This also
shows the ~925Mbps combined figure isn't a mirroring artifact at all —
it's this router's own inherent forwarding capacity for this traffic
pattern, present whether or not anything is being mirrored (characterized
further below).

### Why doesn't bidirectional throughput just add up to ~2Gbps?

Full-duplex 1000BASE-T has physically separate wire pairs for each
direction — there's no *physical* reason two simultaneous ~930Mbps flows
in opposite directions across the same links couldn't coexist at close to
full rate each, for a combined ~1.8Gbps. The ~925Mbps combined figure
above is well short of that, which is worth actually explaining rather
than hand-waving as "some kind of forwarding limit."

Tested directly — each direction run **alone** (not simultaneously):

| Test | Throughput |
|---|---|
| `Host A → Host B` alone | 932.1 Mbps |
| `Host B → Host A` alone | 931.4 Mbps |
| Both simultaneously (bidir) | 461.9 + 462.8 = **924.7 Mbps combined** |

Each direction independently reaches line rate on its own — confirming
the full-duplex physical links themselves are not the constraint, exactly
as expected. But run at the same time, the two flows don't add up; they
split roughly evenly to a combined figure barely above what *either one
alone* achieves. The router's CPU stayed **97-99% idle** in every case
here too — unidirectional or simultaneous, no difference — ruling out
CPU contention as the explanation.

That points to a **shared aggregate forwarding-path ceiling** somewhere
in the router's datapath for this traffic pattern — most plausibly its
hardware NAT/flow-offload engine (`hnat`) or a shared internal bus
between the switch fabric and that engine, sized around ~925-950Mbps
**total** regardless of how many concurrent flows or directions are
using it, rather than a per-direction/per-port allowance the way
full-duplex Ethernet's physical layer would otherwise permit. Not
root-caused at the register/silicon level (out of scope, consistent with
this project's other CAVEATS) — but empirically clear: whatever this
ceiling is, it's shared across flows, not CPU-bound, and not a property
of the physical link.

**So: if combined traffic genuinely exceeds what the monitor port can
carry, the answer is "some traffic will not be mirrored" — not "the
transfer will slow down."** Mirroring is a passive copy at the switch
ASIC level, decoupled from the original traffic path with no flow-control
mechanism back to it (nothing in this architecture *could* signal "monitor
port is full, please slow down" to the original ports — that's not how a
hardware SPAN tap works on essentially any switch chip, this one
included). The original conversation between the two segments proceeds at
whatever rate the router can actually forward it at, entirely independent
of monitor-port congestion; only the mirrored *copy* is at risk of losing
frames once demand exceeds the monitor port's own line rate. This is
consistent with (though doesn't itself directly measure) the general
principle that mirroring happens below the CPU/netdev layer with no
software-visible drop counter on the monitor port — see `SWITCH_TOOL_MAN.md`
CAVEATS — so if frames were being dropped due to oversubscription, no
counter on either end would show it directly; you'd only infer it from
the mirrored capture's rate flattening out at the monitor port's own
physical ceiling (~940-980Mbps for 1000BASE-T) while the real,
independently-verified source traffic exceeded that.

### Will an IDS/NSM tool on the capture port notice if traffic is missing?

A natural next question if you're planning to point something like Zeek
or Suricata at the monitor port: will it tell you when the mirror is
dropping traffic? **Not this project's live-tested claim** — this wasn't
run against this rig — but worth being precise about, based on how these
tools' loss-detection mechanisms actually work, since the honest answer
is "partially, and it matters which layer of loss you mean."

**What their own drop counters won't show**: switch-ASIC-level drops (the
oversubscription scenario above) happen *before* a frame is ever
transmitted onto the wire toward the capture host — from that host's
point of view, a dropped frame simply never existed. This applies
uniformly to *everything* running on that host, including Zeek's own
`capture_loss.log` and Suricata's `stats.log` `capture.kernel_drops` —
both specifically measure loss **between the NIC and the tool's own
processing** (i.e. the tool couldn't keep up and its ring buffer
overflowed), which is a real and useful thing to monitor, but a different
failure mode than switch-level oversubscription. It's the same boundary
`ethtool -S`'s `rx_missed_errors` sits at, measured directly above (0
throughout this testing) — none of these counters can see past the point
where the frame was still inside the switch.

**What they likely *can* notice, indirectly**: for TCP connections a tool
is otherwise tracking, both Zeek and Suricata perform sequence-number-based
gap detection during stream reassembly. If a segment goes missing from an
otherwise-visible TCP stream — for *any* reason, including a switch-level
drop — the reassembler notices the discontinuity. Zeek surfaces this as
`missed_bytes` in `conn.log`, and ships a script
(`policy/misc/capture-loss.zeek`, commonly enabled) built specifically to
turn this into an ongoing capture-health metric. This works regardless of
*where* along the path the segment was actually lost — it's inferred from
what's missing in the reassembled stream, not measured at the drop point
itself.

**Where that safety net has gaps**: it only works for connections the
tool at least partially sees — if an entire flow (or one whole side of a
conversation) is dropped from the start, there's no "before" to compare
against, just a connection that appears never to have happened. And it
doesn't apply to UDP at all — no sequence numbers, no reassembly, no
generic structural way to notice a missing datagram (only possible via
protocol-specific parsing of an application-layer sequence field, if the
specific protocol carries one).

**Net**: likely yes for TCP streams it's otherwise tracking, via
stream-gap symptoms rather than a direct drop count — but that's an
inference from reassembly gaps, not a counter, and it won't cover UDP or
a flow dropped in its entirety. If precise loss accounting matters for
your use case, that's the actual boundary to design around — a
zero-loss capture on a hardware SPAN port isn't guaranteed by anything
here, only high-probability sustained line-rate throughput up to the
monitor port's own physical limit.

## Findings

1. **Port mirroring is real, functional, and line-rate-capable for a
   full bidirectional conversation** — not just a one-way slice. Two
   ports mirrored RX-only into one destination reproduced the true
   combined wire-level throughput of both flows (≈977Mbps, against a
   combined ≈925Mbps TCP-payload rate — consistent with normal framing
   overhead, not duplication), with **zero measured packet loss**, on a
   1Gbps monitor port carrying traffic close to its realistic practical
   ceiling (~940-980Mbps for 1000BASE-T).
2. **Getting "both directions, no duplicates" right requires picking a
   single side of the router (all-ingress or all-egress) across every
   mirrored port — not mode `3` on each.** Mirroring rx+tx on multiple
   ports simultaneously double-counts any packet that transits between
   those ports, since it gets mirrored once on entry and again on exit.
   This is easy to get wrong and not obvious from the tool's own
   `--help` text — worth calling out explicitly for anyone using this
   feature for real traffic analysis.
3. **Mirroring happens entirely in the switch ASIC, not the CPU.** The
   router's own CPU sat at 97-98% idle throughout testing, and the
   monitor port's own vyatta-visible interface counters stayed at 0
   bytes/0 packets despite carrying nearly a gigabit/sec of real
   mirrored traffic — the switch chip duplicates frames entirely below
   Linux's netdev/vyatta accounting layer, consistent with
   `SWITCH_TOOL_MAN.md`'s documented claims.
4. **No software-visible drop counter exists for oversubscription on the
   monitor port itself.** The capture NIC's own counters only report
   what its receive path did with whatever physically arrived; they
   can't distinguish "the switch ASIC dropped frames before
   transmitting them" from "nothing was oversubscribed." That
   distinction would need chip-register-level inspection — out of scope
   here (see `SWITCH_TOOL_MAN.md` CAVEATS on `reg`/`phy` — no
   documented-safe register map exists for this chip).
5. **Mirroring never throttles the original traffic, even near
   saturation.** Toggling the mirror off and rerunning the identical
   bidirectional test produced statistically identical throughput
   (924.5Mbps vs. 924.7Mbps) — confirming mirroring is a true passive
   copy with no backpressure to the source ports. If combined traffic
   ever genuinely exceeds the monitor port's own capacity, the expected
   behavior (architecturally, and consistent with everything measured
   here) is that excess **mirrored frames get dropped** — the real
   conversation between the segments is unaffected.

## Scope and open questions

- This tested RX-only mirroring on both ports. TX-only on both should be
  symmetric by the same logic (every packet also *exits* exactly one
  port), but wasn't separately verified.
- This used ordinary inter-VLAN routed traffic as the source flow, not
  NAT'd/WAN-bound traffic specifically. If your use case is mirroring
  traffic that's also undergoing NAT (e.g. actually watching what leaves
  toward the internet, post-NAT), that exact path wasn't tested here and
  could behave differently depending on how this chip's hardware-NAT
  offload interacts with the mirror feature.
- Mirroring more than two ports into one destination simultaneously
  wasn't tested — the same "pick one consistent direction per port"
  principle should still hold to avoid duplication, but combined demand
  from three or more full-rate ports would exceed the monitor port's own
  1Gbps capacity well before duplication becomes the limiting concern.
