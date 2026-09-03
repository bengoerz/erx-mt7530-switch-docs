% SWITCH(8) unofficial local documentation
% Reverse-engineered from live testing on Ubiquiti EdgeRouter ER-X-class
  hardware, not an official Ubiquiti or MediaTek document — see NOTES.
% 2026-09-03

# NAME

**switch** — control the MediaTek MT7530 switch ASIC directly, bypassing
EdgeOS/vyatta

# SYNOPSIS

```
/sbin/switch
/sbin/switch COMMAND [ARGS...]
```

Running with no arguments prints the built-in usage summary (the only
usage documentation the binary ships with — there is no man page, no
`--help`, and no online documentation for this tool anywhere; this page
exists because none of that does).

# DESCRIPTION

**switch** is an undocumented MIPS binary present on Ubiquiti EdgeRouter
ER-X-class hardware (confirmed at `/sbin/switch`, `-rwxr-xr-x`, dated
2014) that talks directly to the board's MediaTek MT7530 switch ASIC —
entirely below and independent of EdgeOS's normal `vyatta`
`configure`/`set`/`commit` config layer. It ships in the firmware image
as an internal hardware bring-up/debug tool and was never intended for
end-user use, but it is the *only* interface on this hardware for a
handful of switch-chip features vyatta exposes no equivalent for, port
mirroring chief among them.

Every command takes effect **immediately** against live hardware. There is
no dry-run, no confirmation prompt, and — critically — no relationship to
EdgeOS's `commit`/`commit-confirm`/`rollback`/`compare` machinery. See
**CAVEATS**.

Internal switch ports are numbered `0`-`4`, mapping 1:1 to `eth0`-`eth4`
on ER-X-class hardware (verified by cross-referencing `pvid dump` output
against each port's actual vyatta-assigned VLAN — see EXAMPLES). An
internal port `5` also exists in register/table output with no
corresponding `ethN`; believed to be an unused second CPU-facing lane,
unconfirmed.

# COMMANDS

Grouped below by function; this groups differs from the tool's own
`--help` ordering, which is unsorted.

## General / switch table

**dump**
: Dump the hardware MAC address table (learned MAC → port/VLAN/age).

**clear**
: Clear the MAC address table.

**add** *mac* *portmap* [*vlan-id* [*age*]]
: Add a static entry to the MAC table.

**del mac** *mac* **vid** *vid*
**del mac** *mac* **fid** *fid*
: Remove a MAC table entry, addressed by VLAN ID or filter ID.

**search mac** *mac* **vid** *vid*
**search mac** *mac* **fid** *fid*
: Look up a specific MAC table entry.

**filt** *mac* [*portmap* [*vlan-id* [*age*]]]
: Add a source-address (SA) filtering entry — default portmap `1111111`
  (drop from all ports) if omitted.

**mymac** *mac* *portmap*
: Register a "my-MAC" entry (addresses the switch/CPU itself should
  intercept rather than forward).

## VLAN / port configuration

**vlan dump**
: Dump the hardware VLAN table: VID, FID (filter ID), member portmap,
  S-tag state. Includes both explicit service VLANs (e.g. VLAN 20/40 on
  a `vlan-aware` box) and the internal per-port isolation VLANs
  (`4088`-`4095` range) EdgeOS auto-assigns to independent-mode ports —
  see NOTES.

**vlan set** [*idx*] *vid* *portmap*
: Create/modify a VLAN table entry — its ID, VID, and member portmap.
  *idx* may be `NULL` to auto-assign a table slot.

**pvid dump**
: Dump each port's PVID (the VLAN a port's untagged ingress traffic is
  assigned to).

**pvid** *port* *pvid*
: Set a port's PVID. Prints `Set port N pvid M.` on success — the only
  command in this tool observed to print a success confirmation; most
  others are silent.

**tag on** *port*
**tag off** *port*
: Keep (`on`) or strip (`off`) the 802.1Q tag on egress for a port.

**crossover** *port* **auto**\|**mdi**\|**mdix**
: Force or auto-negotiate MDI/MDI-X mode (crossover cable handling) for
  a port.

## Port mirroring (SPAN)

**mirror monitor** *port*
: Designate *port* as the mirror/SPAN destination (monitor port). Only
  one monitor port is active at a time — a second `mirror monitor` call
  replaces the first.

**mirror target** *port* *mode*
: Mark *port* as a mirror source, sending copies of its traffic to
  whatever port `mirror monitor` currently designates. *mode* is
  `0` (off — remove this port as a source), `1` (RX only), `2` (TX
  only), or `3` (both).

Mirroring is a **per-port** relationship — it has no concept of VLANs.
Mirroring two VLAN-tagged member ports each carrying a different VLAN
captures both VLANs' traffic simply because both physical ports were
named as targets, not because the tool applies any VLAN-aware filtering
of its own. There is no `mirror dump`/status command — the only way to
confirm current mirror state is a functional test (generate traffic,
capture on the monitor port) or `phy`/`reg` register inspection against
a chip datasheet, which this document does not provide (see BUGS).

Both `mirror` subcommands are silent on success — no confirmation text,
matching most of this tool's write commands.

## Rate limiting

**ingress-rate on** *port* *Kbps*
**egress-rate on** *port* *Kbps*
**ingress-rate off** *port*
**egress-rate off** *port*
: Enable/disable hardware-enforced per-port rate limiting, ingress or
  egress, in kbps. Not yet exercised live as of this writing — see
  SWITCH_TOOL.md for status.

## IGMP snooping

**igmpsnoop on** *query-interval* *default-router-portmap*
: Enable IGMP snooping and router-port learning. *query-interval* is
  1-255 (seconds, presumed).

**igmpsnoop off**
: Disable IGMP snooping and router-port learning.

**igmpsnoop enable** *port*
**igmpsnoop disable** *port*
: Enable/disable hardware IGMP leave/join/S-query/G-query handling on a
  specific port.

## ACL / security

**acl etype add** *ethertype* *portmap*
: Drop frames matching an EtherType, on the given ports.

**acl dip add** *dip* *portmap*
: Drop packets matching a destination IP, on the given ports.

**acl dip meter** *dip* *portmap* *meter-kbps*
: Rate-limit (rather than drop) packets matching a destination IP.

**acl dip trtcm** *dip* *portmap* *CIR* *CBS* *PIR* *PBS*
: Two-rate three-color (TrTCM) policing of a destination IP — CIR/PIR in
  kbps, CBS/PBS presumably in bytes (unconfirmed).

**acl port add** *sport* *portmap*
: Drop packets matching a source (layer-4) port, on the given ports.

**acl L4 add** *2-bytes* *portmap*
: Drop packets whose layer-4 payload starts with the given 2-byte
  pattern, on the given ports.

**dip add** *dip* *portmap*
**dip del** *dip*
**dip dump**
**dip clear**
: Manage a separate destination-IP table (distinct from `acl dip`, exact
  relationship between the two not investigated).

**sip add** *sip* *dip* *portmap*
**sip del** *sip* *dip*
**sip dump**
**sip clear**
: Manage a source-IP+destination-IP paired table.

None of the ACL/`dip`/`sip` families have been exercised live in this
environment; documented here from `--help` text only.

## Diagnostics / low-level

**phy**
: Dump all PHY registers, all ports.

**phy** *phy-addr*
: Dump PHY registers for one port.

**phy mt7530**
: Dump MT7530-specific chip registers. Confirms chip identity — this is
  how the ASIC was identified as an MT7530 in the first place.

**reg r** *offset*
: Read one register at *offset*.

**reg w** *offset* *value*
: Write one register at *offset*. No bounds-checking, no confirmation
  prompt, no known-safe register map documented anywhere official — see
  BUGS.

**reg d** *offset*
: Dump register(s) starting at *offset*.

# EXAMPLES

Cross-checking the tool's internal port numbering against real vyatta
config (a test setup with `eth1`=VLAN 10, `eth2`=VLAN 20 as
`vlan-aware` switch members, the rest independent):

```
$ /sbin/switch pvid dump
      PORT      PVID
         0      4088
         1        10
         2        20
         3      4091
         4      4092
         5         1
```

Port index matches `ethN` exactly (port 1 → `eth1` → PVID 10, etc.);
ports not in `vlan-aware` switch mode (here, ports 0/3/4 → `eth0`/`eth3`/
`eth4`) get an internal isolation VLAN instead of a real service VLAN —
see NOTES.

Mirroring two source ports to one destination (used for a SPAN/throughput
test — full writeup in `SWITCH_TOOL.md`):

```
$ /sbin/switch mirror monitor 3
$ /sbin/switch mirror target 1 3
$ /sbin/switch mirror target 2 3
```

No output on success. Verified functionally (not just "command exited
0") by pinging between two hosts on ports 1 and 2 and confirming the
mirrored frames appear on port 3's capture.

# FILES

None. This tool has no configuration file of its own and does not read
or write `/config/config.boot` — it talks directly to switch-chip
registers/tables in the kernel driver, which is precisely what makes its
changes invisible to the rest of EdgeOS (see CAVEATS).

# EXIT STATUS

Not formally characterized. No non-zero exit was observed during this
tool's use in this session, including for read-only commands; error
conditions (e.g. malformed `reg w`, an out-of-range port) have not been
deliberately tested.

# CAVEATS

- **No `commit`/`commit-confirm`/`rollback`/`compare` coverage of any
  kind.** Changes take effect immediately and are completely invisible
  to `compare` and `show configuration` — verified live: a change made
  here, followed immediately by `compare` in vyatta config mode, reports
  "No changes between working and active configurations." If a change
  made through this tool cuts off your own access, there is no
  automatic recovery — only a reboot or physical console access.
- **Confirmed does not survive a reboot** (verified live, 2026-09-03, by
  rebooting the test router mid-test and re-running a functional mirror
  check before/after — full writeup in `SWITCH_TOOL.md`). EdgeOS
  reinitializes switch-chip state from `config.boot` at boot; anything
  set through this tool and not also expressed in the vyatta config is
  gone after a reboot and must be reapplied.
- **Bit-position-to-physical-port mapping in `portmap` strings is not
  officially documented anywhere.** This document's port-numbering
  claims are pattern-matched from live `pvid dump`/`vlan dump` output
  cross-referenced against known vyatta config, not from a chip
  datasheet. Treat with appropriate skepticism on hardware revisions
  other than the one this was verified against.
- `reg r`/`reg w`/`reg d` are raw register access with **no
  bounds-checking and no documented-safe register map**. Do not use
  without a specific MT7530 register reference in hand.
- Most subcommands print nothing on success. Absence of an error is the
  only success signal for the majority of this tool's commands — a typo
  in a port number or VID may silently do nothing, or silently do the
  wrong thing.

# NOTES

This page documents behavior **reverse-engineered from live testing**
against one specific device (an EdgeRouter X 5-Port running EdgeOS
`v2.0.9-hotfix.7`) on 2026-09-03, cross-checked against the tool's own
`--help` output and observed command results. It is not an official
Ubiquiti or MediaTek document, was not derived from source code or a chip
datasheet, and should not be assumed accurate for other hardware
revisions or firmware versions without re-verification. Where this page
states something as confirmed/verified, it means confirmed live on that
specific device; where it states something as unconfirmed or inferred,
treat it as exactly that.

For the narrative investigation this page was distilled from — including
full command transcripts, the reboot-persistence test, and the live
mirror-functionality proof — see **SWITCH_TOOL.md**.

# SEE ALSO

`SWITCH_TOOL.md`
