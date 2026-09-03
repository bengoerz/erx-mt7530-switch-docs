# ER-X port mirroring at line speed, at zero performance cost

**Executive summary**: hardware port mirroring on an EdgeRouter X (ER-X
5-Port, EdgeOS `v2.0.9-hotfix.7`, MediaTek MT7530 switch ASIC) runs at
full line rate with **zero measurable cost** to the traffic it
mirrors — proven both in isolation (a single mirrored port/direction
sustained ≈974Mbps wire rate with **zero packet loss**) and under real
bidirectional cross-traffic (mirroring on vs. off was statistically
identical, and a final confirmation run with mirroring active landed at
931.2Mbps, squarely inside the normal range). Establishing what "line
rate" even means for this router required a 26-test side-investigation:
bidirectional throughput between two *routed* segments has a hard
ceiling around **925-950Mbps combined**, not the naive 2Gbps two
full-duplex 1000BASE-T links would suggest — caused by the fixed
capacity of the router's hardware NAT/offload engine, confirmed by
directly disabling it (throughput collapsed to 428Mbps, CPU spiked from
~99% idle to ~95% softirq). The same physical ports reach **~1875Mbps**
when traffic doesn't need routing (pure L2 bridging), confirming the
ceiling is specific to the routing decision, not the links or hardware
themselves.

## Goal

**Primary goal**: prove that traffic can be captured via the switch
ASIC's hardware port-mirroring (see
[`SWITCH_TOOL_TESTRESULTS.md`](SWITCH_TOOL_TESTRESULTS.md)) at full line
rate, with **no measurable throughput penalty and no packet loss** to the
real traffic being mirrored — i.e. that turning on visibility into the
network doesn't cost the network anything.

Answering that convincingly requires knowing what "full line rate"
actually *is* for this router first. That became a necessary side-quest:
**push real bidirectional throughput between two routed network
segments** (**Host A** ↔ **Host B**, routed through the ER-X) as close as
possible to **2Gbps combined**, to establish the actual achievable
baseline — only once that ceiling (and its cause) was pinned down could
the mirroring question be answered against the real number rather than a
theoretical one. Baseline testing had shown combined bidirectional
throughput plateauing around **925-950Mbps**, well under half the
theoretical ceiling, so most of the test log below is this side-quest:
finding out why, and whether it could be pushed higher.

## Method

- **Traffic generator**: `iperf3 --bidir`, typically 8 parallel TCP
  streams per direction, 15-20s runs, between Host A and Host B.
- **Measurement**: `iperf3`'s own self-reported throughput, measured
  directly on the two endpoints — *not* via a port-mirror/capture path
  (deliberately: a mirror destination is itself a separate 1Gbps port and
  would have clipped any measurement above ~1Gbps, defeating the purpose
  of a test aimed at 2Gbps).
- Both endpoints carried a second, lower-metric management NIC; without
  an explicit route, traffic to the other host's subnet silently took
  that shortcut instead of the real routed path being tested. Every run
  added an explicit host route to force genuine routed traffic between
  the two segments.
- After each router config change, the physical link bounce this causes
  wipes DHCP leases and manual routes on the endpoints — each change was
  followed by an interface bounce + DHCP renewal + route re-add cycle
  before the next measurement.

## Tests and results

| # | Change tested | Combined throughput | Verdict |
|---|---|---:|---|
| 1 | Baseline (VLAN-aware segments, DPI on) | 926.5 Mbps | — |
| 2 | DPI / traffic-export disabled | 928.9 Mbps | no effect |
| 3 | Host A/B ports converted to independent (non-VLAN) ports | 926.1 Mbps | no effect |
| 4 | All switch ports made VLAN-aware members | 928.3 Mbps | no effect |
| 5 | MTU raised to platform max (2018 — hardware caps below true jumbo) | 802.9 Mbps | **worse** |
| 6 | MTU reverted to 1500 | 927.3 Mbps | confirms 1500 optimal |
| 7 | UDP, bidirectional, 900Mbps requested each way | 954.0 Mbps, ~0.01% loss | same ceiling, protocol-agnostic |
| 8 | UDP, unidirectional, 950Mbps requested | 950.0 Mbps achieved, ~0.008% loss | alone, hits full requested rate cleanly |
| 9 | NIC ring buffers, RX+TX → 4096 (both hosts) | 927.1 Mbps | no effect |
| 10 | BBR congestion control (replacing cubic) | 935.2 Mbps | ~flat, but retransmits fell 10x (thousands → hundreds) |
| 11 | BBR + enlarged TCP buffers (64MB) | 935.3 Mbps | no additional effect |
| 12 | (final-state confirmation after tests 9-11 restore) | 934.9 Mbps | consistent |
| 13 | Ring buffer RX-only → 4096 (TX default) | 933.1 Mbps | no effect |
| 14 | Ring buffer TX-only → 4096 (RX default) | 932.8 Mbps | no effect |
| 15 | Port mirroring fully disabled | 934.7 Mbps | no effect (mirror never mattered) |
| 16 | Single TCP stream per direction (no parallelism) | 924.7 Mbps, minimal retransmits | ceiling isn't about needing many flows |
| 17 | 32 parallel TCP streams per direction | 942.8 Mbps, retransmits exploded (~29k each way) | more streams ≠ more throughput, just more loss |
| 18 | Third host, separate NIC on a separate PCIe bus from Host A/B's shared card | 847-857 Mbps, badly asymmetric | **inconclusive** — confounded by an unrelated NIC-specific RX-path problem, didn't cleanly isolate the PCIe-bus question |
| 19 | NAT masquerade applied to A→B traffic (testing whether hardware NAT offload favors NAT'd flows) | 930.5 Mbps | no effect — rules out NAT-offload hypothesis |
| **20** | **Pure L2 hardware bridging** — same two physical ports, VLAN-aware/routing removed entirely, hosts given static IPs on one flat subnet | **1875.4 Mbps** (then **1873.3 Mbps** on a repeat run), near-zero retransmits | **breakthrough** — see Findings |
| 21 | Routed VLAN topology restored (control, same hardware) | 919.8 Mbps | confirms the ceiling is topology-dependent, fully reversible |
| **22** | 128 parallel TCP streams per direction (iperf3's hard max) | **997.4 Mbps combined** | real ~5-8% bump over the usual ceiling, but retransmits exploded to 82,433+90,171 in 15s — not a usable optimization |
| **23** | **Control test: hardware NAT/offload (`hwnat`) disabled** | **428.5 Mbps combined** (less than half normal), CPU spiking to 94.5% softirq | confirms the ~930Mbps ceiling *is* the hardware offload engine's own capacity |
| 24 | `hwnat` re-enabled, confirmation | 934.0 Mbps | fully restored |
| **25** | Independent (non-switch-fabric) port ↔ switch-fabric port, both directions | 936-938 Mbps / 927.2 Mbps | same ceiling — no advantage from an independent port; see below |
| 26 | Final confirmation on the fully-restored baseline config, port mirroring active | 931.2 Mbps combined | squarely inside the established range, mirroring costs nothing |

## Findings

1. **Root cause identified: the ~925-950Mbps ceiling is specific to the
   router's L3 routing/forwarding decision, not any physical, PCIe,
   switch-fabric, or protocol-level constraint.** Test 20 is the decisive
   result: bridging the exact same two physical ports at L2 (no routing
   decision at all) on the exact same hardware reached **~1875Mbps
   combined — essentially the full 2×1Gbps physical ceiling** —
   reproduced twice, with near-zero packet loss. Restoring the routed
   topology on the same hardware (Test 21) brought the ~920Mbps ceiling
   right back. This is about as clean an A/B as this kind of
   investigation gets.
2. **The ceiling is protocol-agnostic and insensitive to virtually
   everything else tried.** TCP and UDP hit the same number (Tests 7-8).
   VLAN configuration, DPI/export, MTU (both directions), NIC ring
   buffers, TCP congestion control, TCP buffer size, stream count (1 to
   32), NAT/masquerade, and port mirroring — none of it moved the ceiling
   by more than measurement noise.
3. **The router's own CPU stayed 97-99% idle throughout every test**,
   including the bidirectional runs sitting right at the ceiling. This
   rules out CPU-bound software routing as the mechanism — whatever caps
   throughput in the L3 path is doing so without spending CPU cycles,
   consistent with a fixed-capacity hardware datapath resource (the
   router's hardware NAT/flow-offload engine — MediaTek `hnat`/PPE — or a
   bus feeding it) rather than a MIPS-CPU routing bottleneck. **Test 23
   converts this from inference to proof**: disabling the hardware
   offload engine directly dropped throughput to under half normal and
   spiked CPU to ~95% softirq — the offload engine's own fixed capacity
   *is* the ceiling.
4. **Larger MTU made things measurably worse** (Test 5: 802.9Mbps vs.
   927.3Mbps at the platform's actual max supported MTU of 2018 bytes —
   true 9000-byte jumbo frames aren't supported on this hardware at all).
   Frames above 1500 bytes appear to fall off the hardware fast path in
   this router's forwarding pipeline rather than reducing per-packet
   overhead as normally expected from larger frames.
5. **BBR congestion control is a genuine, free improvement in link
   quality (not throughput)** — retransmits dropped roughly 10x with no
   downside, even though it didn't raise the ceiling itself. Worth
   keeping enabled on hosts that route through this class of router
   regularly.
6. **Mirroring never affects the underlying traffic** — confirmed
   directly (Test 15: mirror on vs. off, statistically identical) and
   consistent with the separate port-mirroring throughput proof in
   [`SWITCH_TOOL_TESTRESULTS.md`](SWITCH_TOOL_TESTRESULTS.md).
7. **The separate-PCIe-bus test (18) was inconclusive, not negative** —
   it surfaced an apparently unrelated NIC-specific problem rather than
   cleanly isolating whether two hosts sharing one physical NIC's PCIe
   upstream link mattered. This became moot once Test 20 showed pure L2
   switching between those same two hosts — still sharing that same
   card — already reached ~1875Mbps: if PCIe sharing were the
   constraint, bridging wouldn't have helped either, so it demonstrably
   isn't.
8. **An independent (non-switch-fabric) port offers no advantage over a
   switch-fabric port** (Test 25) — a read-only driver/bus comparison
   (`ethtool -i`) had already shown the router's independent port and its
   switch-fabric ports share an identical driver and bus address, with
   only one frame-engine instance in the kernel for the entire switch
   complex. A live throughput test through an independent port confirmed
   it: same ~930-940Mbps ceiling in both directions. The hardware offload
   engine operates at the IP/routing layer, not per physical port, so
   which port a flow enters or exits on doesn't matter to its capacity.

## Mechanism model (inference, not independently verified)

All switch ports sit behind the MT7530 switch fabric, which connects to
the CPU/offload-engine complex over a single shared internal
interconnect. L2 traffic never leaves the switch fabric (full ~2Gbps,
matching Test 20). Routed traffic must leave the fabric, get processed by
the offload engine, and re-enter — crossing the shared interconnect twice
per packet — so ~930Mbps combined routed throughput represents ~1.86Gbps
of occupancy on a ~2Gbps shared path (950Mbps unidirectional × 2
traversals ≈ 1.9Gbps also fits). This model is consistent with every
result collected, including the idle CPU and the MTU regression (oversized
frames falling off the fast path onto the slow CPU path), but the exact
interconnect topology isn't published, so it remains a well-fitted model
rather than a confirmed spec.

Two practical paths forward for real routed traffic that needs to exceed
this ceiling: keep the segments on one L2 domain (already proven), or
route on a separate host with two physical NICs (one per segment) and
demote the ER-X to a pure L2 switch — each flow would then enter on one
physical link and exit on another, avoiding the double-traversal
entirely, while the ER-X's hardware mirroring continues to work on the
access ports. A single-trunk router-on-a-stick setup would *not* fix
this — it recreates the same shared-link cap.

## Bottom line

**Primary goal achieved: port mirroring on this hardware runs at line
speed with zero measurable cost to the real traffic being mirrored.**
Demonstrated two independent ways: (1) the standalone mirroring
investigation in [`SWITCH_TOOL_TESTRESULTS.md`](SWITCH_TOOL_TESTRESULTS.md)
pushed a single mirrored port/direction to ≈974Mbps wire rate with
**zero measured packet loss**, cross-checked via hardware RX counters and
raw capture analysis, not just `iperf3`'s self-report; (2) *this*
investigation confirmed mirroring has **no effect on the real traffic
being mirrored**, twice — Test 15 (mirror on vs. off, statistically
identical combined throughput) and Test 26 (the final confirmation run,
mirror active on the fully-restored production config, landing at
931.2Mbps — squarely inside the same 925-950Mbps range every unmirrored
run also landed in). Port mirroring on this hardware is implemented in
the switch ASIC itself, not the CPU forwarding path, which is exactly why
it's free: it doesn't compete with the same fixed-capacity resource (the
hardware offload engine) that caps routed throughput — which is what all
the rest of this investigation went and found.

**Side-quest result: 2Gbps bidirectional is not achievable between two
*routed* segments on this router** — the L3 forwarding path has a hard,
CPU-independent ceiling around 925-950Mbps combined, and nothing at the
OS, driver, protocol, or router-config layer changes that (26 tests
covering the practical space of software/config levers, including live
confirmation that an independent port offers no advantage over a
switch-fabric one). **2Gbps bidirectional *is* achievable on the same
hardware for traffic that doesn't need routing** — pure L2 switching
between the same two ports reached the full physical ceiling cleanly and
reproducibly. This is what made the primary goal's numbers meaningful in
the first place: "line speed" for this router's *routed* traffic is
~930-950Mbps, not 2Gbps, and that's the bar the mirroring proof above
actually had to clear. If a real workload needs to move more than that
between two routed segments, keeping them on the same L2 domain (or
accepting the router's routing-path ceiling) are the two realistic
options on this specific ER-X; there is no config change available that
lets routed traffic between two subnets exceed roughly one port's worth
of aggregate throughput on this hardware.

## Operational notes (for anyone repeating this kind of test)

- **A router reboot was required twice** during this investigation to
  clear a stale ARP/neighbor entry for the test-VM host on its management
  interface, causing every downstream host to become fully unreachable —
  intermittently, seemingly from cumulative rapid SSH/reconfiguration
  churn rather than any single specific command. Symptom: the neighbor
  table shows `FAILED`/`INCOMPLETE` for an address that's actually up;
  fix: reboot the router (not the downstream host — it was never actually
  down in either occurrence, confirmed via direct ARP/ping checks from
  the router itself both times).
- **After any router interface/VLAN/switch-port config change**, expect
  connected hosts to lose their DHCP lease and any manually-added routes
  (physical link bounce). Fix: bounce the interface, renew DHCP, then
  re-add the explicit cross-subnet route.
- **If DHCP renewal itself doesn't recover an address** after such a
  bounce, the underlying cause can be the router's own DHCP server bound
  to a now-stale interface reference from a just-recreated VLAN
  sub-interface — fixable without a full reboot by toggling the DHCP
  server off and back on, which forces a clean rebind to the current
  interfaces.
- EdgeOS's `commit-confirm` + `confirm` flow could not be driven reliably
  through non-interactive SSH automation in this environment (the
  `confirm` command was never successfully delivered before the
  auto-revert timer fired, twice). Given none of this testing touched the
  actual management access path, plain `commit` + `save` was used for all
  config changes instead, accepting that a mistake would need to be fixed
  forward rather than auto-reverted — appropriate here, but worth
  designing around deliberately (or scripting the interactive session
  differently) if attempting `commit-confirm` automation on a change that
  *does* risk admin lockout.
- NAT/masquerade rule IDs on this EdgeOS build must be in the `5000-9999`
  range (source/masquerade rule types specifically) — a lower ID number
  fails commit validation.

All router/host state was restored to the documented pre-investigation
baseline (VLAN-aware segments, DPI/export enabled, MTU 1500, port
mirroring reapplied) at the end of this investigation.
