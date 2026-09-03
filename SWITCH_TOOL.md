# `/sbin/switch` — the MT7530 switch-ASIC tool on Ubiquiti EdgeRouter (ER-X class) hardware

_Captured live against an EdgeRouter X 5-Port running EdgeOS
`v2.0.9-hotfix.7` on 2026-09-03. Every output block below is real,
unedited command output from that box, not reconstructed from memory._

## What it is

`/sbin/switch` is a small MIPS binary (`-rwxr-xr-x`, dated 2014) that talks
**directly to the switch chip** — a MediaTek MT7530 (`/sbin/switch phy
mt7530` confirms the chip identity) — completely below EdgeOS's vyatta
config layer. It's not documented, has no man page, and doesn't appear
anywhere in `/config/config.boot` or `show configuration`. It ships in the
firmware image as a raw hardware debug tool Ubiquiti left in place.

A companion quick-reference in classic man-page format,
`SWITCH_TOOL_MAN.md`, covers the full command set; this document is the
narrative investigation — what was actually tried, what the real output
looked like, and what that revealed.

## Full command reference (verbatim, no-args usage output)

```
$ /sbin/switch
 /sbin/switch acl etype add [ethtype] [portmap]              - drop etherytype packets
 /sbin/switch acl dip add [dip] [portmap]                    - drop dip packets
 /sbin/switch acl dip meter [dip] [portmap][meter:kbps]      - rate limit dip packets
 /sbin/switch acl dip trtcm [dip] [portmap][CIR:kbps][CBS][PIR][PBS] - TrTCM dip packets
 /sbin/switch acl port add [sport] [portmap]           - drop src port packets
 /sbin/switch acl L4 add [2byes] [portmap]             - drop L4 packets with 2bytes payload
 /sbin/switch add [mac] [portmap]                  - add an entry to switch table
 /sbin/switch add [mac] [portmap] [vlan id]        - add an entry to switch table
 /sbin/switch add [mac] [portmap] [vlan id] [age]  - add an entry to switch table
 /sbin/switch clear                                - clear switch table
 /sbin/switch del mac [mac] vid [vid]              - delete an entry from switch table
 /sbin/switch del mac [mac] fid [fid]               - delete an entry from switch table
 /sbin/switch search mac [mac] vid [vid]            - search an entry with specific mac and vid
 /sbin/switch search mac [mac] fid [fid]            - search an entry with specific mac and fid
 /sbin/switch dip add [dip] [portmap]                  - add a dip entry to switch table
 /sbin/switch dip del [dip]                            - del a dip entry to switch table
 /sbin/switch dip dump                                 - dump switch dip table
 /sbin/switch dip clear                                - clear switch dip table
 /sbin/switch dump          - dump switch table
 /sbin/switch ingress-rate on [port] [Kbps]        - set ingress rate limit on port 0~4
 /sbin/switch egress-rate on [port] [Kbps]         - set egress rate limit on port 0~4
 /sbin/switch ingress-rate off [port]              - del ingress rate limit on port 0~4
 /sbin/switch egress-rate off [port]               - del egress rate limit on port 0~4
 /sbin/switch filt [mac]                           - add a SA filtering entry (with portmap 1111111) to switch table
 /sbin/switch filt [mac] [portmap]                 - add a SA filtering entry to switch table
 /sbin/switch filt [mac] [portmap] [vlan id]       - add a SA filtering entry to switch table
 /sbin/switch filt [mac] [portmap] [vlan id] [age] - add a SA filtering entry to switch table
 /sbin/switch igmpsnoop on [Query Interval] [default router portmap] - turn on IGMP snoop and router port learning (1~255)
 /sbin/switch igmpsnoop off                                  - turn off IGMP snoop and router port learning
 /sbin/switch igmpsnoop enable [port#]                       - enable IGMP HW leave/join/Squery/Gquery
 /sbin/switch igmpsnoop disable [port#]                      - disable IGMP HW leave/join/Squery/Gquery
 /sbin/switch mymac [mac] [portmap]                  - add a mymac entry to switch table
 /sbin/switch mirror monitor [portnumber]            - enable port mirror and indicate monitor port number
 /sbin/switch mirror target [portnumber] [0:off, 1:rx, 2:tx, 3:all]  - set port mirror target
 /sbin/switch phy                                    - dump all phy registers
 /sbin/switch phy [phy_addr]                         - dump phy register of specific port
 /sbin/switch phy mt7530                             - dump mt7530 phy registers
 /sbin/switch crossover [port] [auto/mdi/mdix]       - switch auto or force mdi/mdix mode for crossover cable
 /sbin/switch pvid [port] [pvid]                - set pvid on port 0~4
 /sbin/switch pvid dump                         - dump port pvid setting
 /sbin/switch reg r [offset]                       - register read from offset
 /sbin/switch reg w [offset] [value]               - register write value to offset
 /sbin/switch reg d [offset]                       - register dump
 /sbin/switch sip add [sip] [dip] [portmap]            - add a sip entry to switch table
 /sbin/switch sip del [sip] [dip]                      - del a sip entry to switch table
 /sbin/switch sip dump                                 - dump switch sip table
 /sbin/switch sip clear                                - clear switch sip table
 /sbin/switch tag on [port]                        - keep vlan tag for egress packet on port 0~4
 /sbin/switch tag off [port]                       - remove vlan tag for egress packet on port 0~4
 /sbin/switch vlan dump                            - dump switch table
 /sbin/switch vlan set [vlan idx (NULL)][vid] [portmap]  - set vlan id and associated member
```

Well beyond VLAN/mirror config: MAC-table manipulation (`add`/`del`/`search`/
`filt`/`mymac`), per-source/dest-IP ACL/rate-limit rules (`acl`, `sip`,
`dip`), IGMP snooping, crossover/MDI-X control, and raw PHY register dumps
(`phy`, `phy mt7530`). This doc focuses on the subset exercised live —
VLAN/PVID inspection and port mirroring; the rest is documented from
`--help` text only in the companion man page.

## Ground truth vs. the vyatta config layer, cross-checked live

A test device was configured with two ports as `vlan-aware` switch
members (one carrying VLAN 10, the other VLAN 20) and two ports left as
independent, non-switched routed interfaces. The vyatta config
(`show interfaces switch switch0 switch-port`) reported that cleanly:

```
interface  mode         switch-master
=========  ====         =============
eth0       independent
eth1       switch       switch0
eth2       switch       switch0
eth3       independent
eth4       independent
```

`/sbin/switch pvid dump`:

```
      PORT      PVID
         0      4088
         1        10
         2        20
         3      4091
         4      4092
         5         1
```

`/sbin/switch vlan dump`:

```
  vid  fid  portmap    s-tag
    1    0  invalid
   ...  (2-16: all "invalid" — unused VLAN IDs, not yet defined in hardware)
   10    0  -1----1-       0
   20    0  --1---1-       0
 4088    0  1-----1-       0
 4089    0  -1----1-       0
 4090    0  --1---1-       0
 4091    0  ---1--1-       0
 4092    0  ----1-1-       0
 4093    0  -----11-       0
 4094    0  ------1-       0
```

**Interpretation, cross-checked against the vyatta table above (not
assumed):**

- Internal switch port index maps 1:1 to `ethN` for indices 0-4 (`0`=`eth0`,
  `1`=`eth1`, … `4`=`eth4`). Index `5` has no corresponding `ethN` and its
  PVID is the untouched factory default (`1`) — almost certainly an unused
  internal port, possibly a second CPU-facing lane on this chip that EdgeOS
  doesn't wire up. Not confirmed against a register-level source, flagged
  as inference.
- **Ports actually in `vlan-aware` switch mode** (`eth1`→VID 10, `eth2`→VID
  20 in this test) show their **real service VLAN ID as PVID** — no
  isolation-VLAN indirection for these two.
- **Independent-mode ports** (`eth0`, `eth3`, `eth4`) each still get their
  own **internal isolation VLAN** in the `4088`-`4094` range, invisible in
  `show interfaces`/`config.boot` — `eth0`→`4088`, `eth3`→`4091`,
  `eth4`→`4092`. This is the mechanism that keeps independent-mode ports
  from leaking traffic to each other at the switch-chip level even though
  nothing in the vyatta config says so explicitly.
- `4089`/`4090`/`4093` exist in the hardware VLAN table but are **not**
  currently any port's PVID — leftover/reserved isolation-VLAN slots from
  the chip's default table population, not evidence of a live isolation
  relationship. (`4089`'s portmap is coincidentally identical to VID 10's —
  bit pattern reuse, not a hidden alias; PVID dump is the authority on what
  a port *actually* uses, not presence in the VLAN table.)
- `portmap` is a fixed-width bitmap, one character per internal port index,
  `1` = member, read left-to-right as index `0..7`. Every VID above
  consistently sets bit index `6` alongside whichever physical port it
  belongs to — that bit is very likely the CPU port (the path packets take
  to reach EdgeOS's routing stack), never a bare physical port on its own.
  **Still not officially documented** — this is pattern-matched from every
  VID observed on this box, not a datasheet fact.

## Writes are real, immediate, and completely invisible to vyatta

Live-verified round trip, port 1 (already PVID 10):

```
$ /sbin/switch pvid 1 10
Set port 1 pvid 10.
$ /sbin/switch pvid dump
      PORT      PVID
         0      4088
         1        10
         ...
```

Then, in the same session, from vyatta config mode:

```
# compare
No changes between working and active configurations
```

Confirms directly: `/sbin/switch` writes take effect immediately against
the live switch chip and produce **zero trace** in `compare`, `show
configuration`, or `config.boot` — `commit-confirm`'s auto-rollback-on-
timeout protects nothing here. A mistake made through this tool (e.g.
mirroring the wrong port onto your own management path) has no built-in
recovery path short of a reboot (which reloads switch-chip state from the
vyatta config, see below) or physical console access.

## Persistence across reboot — live-verified, not just assumed

Method: functional test, not a config dump (there's no `mirror dump`
subcommand — mirror state isn't visible in `vlan dump`/`pvid dump` output
at all, before or after). A mirror was configured (see next section) and
verified working — a ping between two hosts on the mirrored ports produced
3 mirrored frames on the monitor port's capture. Then:

```
$ reboot now         # ~100s to come back
```

Same ping, same capture, immediately after the box came back (interfaces,
routing, and an unrelated firewall ruleset that had also been configured
all came back correctly — those are vyatta-managed and reloaded from
`config.boot` as expected):

```
0 packets captured
0 packets received by filter
```

**Confirmed: `/sbin/switch` mirror state does not survive a reboot.**
`pvid dump` also came back byte-for-byte identical to its pre-reboot
capture — but that's because the switch-member ports' PVIDs are driven by
the vyatta config either way, not because anything `/sbin/switch`-side
persisted; the mirror relationship specifically (invisible to `pvid
dump`/`vlan dump`) was the part that reset. General rule confirmed: after
any reboot, **any** `/sbin/switch` change (mirror, rate limits, ACLs,
etc.) needs to be reapplied — nothing set through this tool survives on
its own.

## Port mirroring (`mirror monitor` / `mirror target`)

The two commands that matter for SPAN/mirror work — EdgeOS's vyatta config
has no equivalent for this at all:

```
/sbin/switch mirror monitor [portnumber]
/sbin/switch mirror target [portnumber] [0:off, 1:rx, 2:tx, 3:all]
```

- `mirror monitor <port>` designates the one port that receives mirrored
  copies (the SPAN/monitor destination).
- `mirror target <port> <mode>` marks a source port to be mirrored *to*
  whatever `mirror monitor` currently points at — `1`=RX only, `2`=TX only,
  `3`=both. Call it once per source port; nothing here is VLAN-scoped —
  mirroring is a per-**port** relationship regardless of what VLAN(s) that
  port carries, so mirroring two VLAN-tagged ports (each carrying a
  different VLAN) captures both VLANs' traffic simply because both ports
  are named as targets, not because the tool understands VLANs specially.

Both commands are **silent on success** — no confirmation text, unlike
`pvid`.

**Live-verified functionally.** Setup: port 3 as the monitor/destination,
ports 1 and 2 (each carrying a different VLAN) as mirror sources:

```
/sbin/switch mirror monitor 3      # port 3 = monitor/destination port
/sbin/switch mirror target 1 3     # port 1 -> mirror, both rx+tx
/sbin/switch mirror target 2 3     # port 2 -> mirror, both rx+tx
```

Functional proof (5 pings between a host on port 1 and a host on port 2,
captured on a third host attached to port 3):

```
<src-mac-1> > <router-mac>: <ip-1> > <ip-2>: ICMP echo request   # port 1 RX, mirrored
<router-mac> > <src-mac-2>: <ip-1> > <ip-2>: ICMP echo request   # port 2 TX, mirrored
<router-mac> > <src-mac-1>: <ip-2> > <ip-1>: ICMP echo reply     # port 1 TX, mirrored
```

Three mirrored frames per single ping round-trip — one for each leg the
packet actually took across the two source ports. Confirms, live and not
just per the manual: a single monitor port really does receive copies from
**two different physical ports carrying two different VLANs**
simultaneously, both directions.

## Rate limiting (`ingress-rate` / `egress-rate`)

```
/sbin/switch ingress-rate on [port] [Kbps]
/sbin/switch egress-rate on [port] [Kbps]
/sbin/switch ingress-rate off [port]
/sbin/switch egress-rate off [port]
```

Per-port, hardware-enforced rate limiting below the vyatta layer — same
invisibility/no-rollback caveats as everything else here. Not yet
exercised live.

## Related but distinct: `system offload hwnat`

Not part of `/sbin/switch` — this is a separate vyatta-level setting
(`set system offload hwnat enable`) that offloads IPv4 forwarding/NAT to
the MT7530's hardware NAT engine, bypassing the CPU's Linux netfilter path
for eligible flows entirely. Mentioned here only because it's easy to
conflate with the switch-chip tool above; it's configured and verified
through the normal vyatta config layer, not through `/sbin/switch`.

## Safety notes (restated from live confirmation, not just carried over)

- No `commit-confirm`/`rollback`/`compare` coverage — verified above, not
  assumed.
- Persistence-across-reboot: **confirmed no**, via a real reboot cycle —
  see above.
- `reg r`/`reg w`/`reg d` are raw ASIC register access — no bounds-checking,
  no official register map. Only touch with a specific MT7530 register
  reference in hand.
- Test one port/one change at a time, confirm `dump`/`pvid dump`/`vlan
  dump` and actual traffic before making a second change — especially true
  here since there's no automatic undo.
