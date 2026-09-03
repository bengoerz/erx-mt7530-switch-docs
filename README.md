# erx-mt7530-switch-docs

Unofficial documentation for `/sbin/switch` — an undocumented, unsupported
binary shipped on Ubiquiti EdgeRouter X (ER-X) class hardware (MT7621-based
boards) that talks directly to the board's switch ASIC, below EdgeOS's
normal `vyatta` configuration layer. On the hardware this was verified
against, the switch chip itself identifies as a MediaTek **MT7530**
(confirmed via the tool's own `phy mt7530` subcommand) — a common
switch-ASIC pairing on MT7621 SoC-based routers, distinct from the MT7621
SoC/CPU itself.

## Why this exists

`/sbin/switch` ships in the firmware image but has no man page, no
`--help` beyond a bare usage dump, and no official documentation anywhere.
It's the only way to do a few things EdgeOS's normal config simply has no
equivalent for — port mirroring (SPAN) chief among them — but using it
safely means understanding behavior that isn't written down anywhere:
what each subcommand actually does, how its changes relate (or don't) to
the normal `configure`/`commit`/`rollback` config system, and whether
anything it changes survives a reboot.

This repo exists to answer those questions from **live testing**, not
speculation — every claim here was verified against a real device, with
the actual command transcripts to back it up, and inference is called out
explicitly wherever the evidence is indirect.

## What's here

- **[`SWITCH_TOOL_MAN.md`](SWITCH_TOOL_MAN.md)** — a man-page-style quick
  reference: every subcommand from the tool's own usage output, grouped
  by function (VLAN/port config, mirroring, rate limiting, MAC table,
  ACLs, IGMP snooping, low-level PHY/register access), with notes on
  which ones have actually been exercised live versus documented from
  `--help` text alone.
- **[`SWITCH_TOOL.md`](SWITCH_TOOL.md)** — the narrative investigation:
  full command transcripts, the live reboot-persistence test, and
  functional proof of port mirroring across multiple VLANs.
- **[`SWITCH_TOOL_TESTRESULTS.md`](SWITCH_TOOL_TESTRESULTS.md)** —
  sustained-throughput proof for port mirroring: a single mirrored
  port/direction pushed to ≈974Mbps wire rate with zero measured packet
  loss, cross-checked two independent ways.

More may follow as more of this tool gets exercised and verified.

## Scope and disclaimer

This is a personal reverse-engineering project, not affiliated with,
endorsed by, or derived from any Ubiquiti or MediaTek source code or
official documentation. Findings are specific to the hardware/firmware
revision they were tested against (an EdgeRouter X 5-Port running EdgeOS
`v2.0.9-hotfix.7`, as of 2026) and may not hold on other revisions.
`/sbin/switch` writes take effect immediately against live hardware with
**no** relationship to EdgeOS's config-rollback safety net — read
`SWITCH_TOOL_MAN.md`'s CAVEATS section before running anything from here
against your own device.

## Credit

Inspired by gojimmypi's
["EdgeRouter-X Port Mirroring: Inspect ESP32 Network
Packets"](https://gojimmypi.github.io/Edgerouter-Port-Monitor/) (2022),
the earliest writeup found of the `mirror monitor`/`mirror target`
port-mirroring feature this repo also documents. Independently verified
here rather than copied from it, but it's the reason this looked worth
digging into further in the first place.
