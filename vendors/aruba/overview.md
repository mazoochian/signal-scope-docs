# HPE Aruba Networking — ArubaOS-CX Overview

Scope of this vendor folder: **ArubaOS-CX** only — the modern, current-generation CLI/NOS running on the 6xxx (6100/6200/6300/6400) and 8xxx (8320/8325/8360/8400/9300/10000) switch series. This is what SignalScope should target for HPE Aruba wired-switch support.

## Legacy dialects (not covered in depth here)

HPE/Aruba's switch line has three distinct CLI histories a future reader should not confuse with ArubaOS-CX:

- **ArubaOS-Switch (formerly "ProVision"/"ProCurve")** — the CLI on the older 2530/2540/2620/2930/3810/5400zl families. Menu-driven config resembles Cisco IOS loosely (`configure`, `vlan 10`, `interface`) but with materially different syntax and a flat `show run` model with no checkpoint concept. Still shipping/supported on some SKUs as of this writing, but a separate research pass.
- **Comware-based HPE switches** (ex-3Com/H3C heritage, e.g. some HPE FlexNetwork/FlexFabric and older HPE A-series/5500/7500 series) — Comware CLI (`system-view`, `vlan batch`, `display` instead of `show`) is a completely different command grammar, closer to Huawei VRP than to IOS.
- **ArubaOS (wireless controller OS)** — unrelated to ArubaOS-CX despite the name; this is the WLAN controller/AP OS, out of scope for switch management entirely.

None of the above are documented further in this folder. If SignalScope needs to support them later, they warrant their own `cli-reference.md`/`mib-reference.md` pass rather than being folded into these ArubaOS-CX files.

## CLI dialect

ArubaOS-CX has a traditional line-mode CLI (SSH/Telnet/console, `configure terminal` → mode contexts) that is IOS-like in shape (privileged vs. config vs. config-if contexts, `show` commands) but has its own command grammar throughout — it is **not** a Cisco IOS clone and commands should never be assumed to transliterate directly. It also exposes a first-class **REST API** (v1/v10.xx depending on firmware) and an **NAE (Network Analytics Engine)** Python scripting environment for condition-based automation. SignalScope's differentiator is the literal CLI/terminal path, so this documentation set focuses on the CLI; the REST API and NAE are out of scope for this pass but worth flagging as a possible alternate/future integration path distinct from both CLI and SNMP.

## SNMP version support

ArubaOS-CX supports **SNMPv1, SNMPv2c, and SNMPv3** concurrently (multiple community strings / SNMPv3 users can coexist). SNMPv3 supports the standard USM auth (MD5/SHA) and privacy (DES/AES) protocols, plus `snmpv3 context`/VRF-scoped access. See [cli-reference.md](cli-reference.md) for exact config commands.

**Practical write scope**: unlike many vendors where SNMP is read-only by default or by policy, ArubaOS-CX ships several enterprise MIBs with explicit `read-write` objects — most notably `ARUBAWIRED-PORTSECURITY-MIB`, parts of `ARUBAWIRED-MSTP-MIB`, and `ARUBAWIRED-CONFIG-MIB` (a full SNMP-driven config-copy/checkpoint workflow, analogous to Cisco's `CISCO-CONFIG-COPY-MIB`). Confirmed directly from vendor MIB source in this session — see [mib-reference.md](mib-reference.md) for specifics and per-object access levels. Standard `IF-MIB::ifAdminStatus` is writable as on virtually every vendor. Notably, the vendor's own **VLAN-membership MIB** (`ARUBAWIRED-PORTVLAN-MIB`) is **read-only** (`arubaWiredPortVlanMemberMode`/`arubaWiredPortVlanMemberVid` are both `MAX-ACCESS read-only`) — confirmed from source — so SNMP-driven VLAN assignment, if it exists at all on this platform, would have to go through standard `Q-BRIDGE-MIB::dot1qVlanStaticTable`/`dot1qPvid`, whose write support on ArubaOS-CX was **not** independently confirmed this session (state uncertain — verify against a live device or the SNMP/MIB guide before assuming it's writable).

## Config-apply model

ArubaOS-CX applies configuration lines **immediately** to the running configuration as they're typed, the same as Cisco/Arista/Huawei — there is no Juniper-style candidate/`commit` requirement for ordinary config changes to take effect. However, ArubaOS-CX layers a distinct **checkpoint** system on top of this for safety/rollback that is worth SignalScope modeling explicitly:

- The running config is continuously the "live" state (`show running-config`), same as always.
- `startup-config` is a separate, explicitly-saved copy used at boot — persisted only via `copy running-config startup-config` (aliased as `write memory`), exactly like the Cisco pattern.
- **Checkpoints** are named, timestamped snapshots of a full config (`copy running-config checkpoint <name>`) that can be diffed (`checkpoint diff <name1> <name2>`) and restored (`copy checkpoint <name> running-config`). The switch also auto-creates a checkpoint a few minutes after any config change (default name pattern `CPC<timestamp>`).
- **`checkpoint auto <minutes>`** implements a Juniper-`commit confirmed`-like guarded-apply workflow: it arms an automatic rollback timer, and near the end of the window the CLI prompts for confirmation — if not confirmed, the switch reverts to the pre-change checkpoint automatically. This is the closest ArubaOS-CX gets to a "candidate config" safety net, but it is opt-in per change, not the default apply path (ordinary commands still take effect immediately, unguarded, unless the operator explicitly wraps the session in `checkpoint auto`).
- This checkpoint mechanism is also SNMP-drivable via `ARUBAWIRED-CONFIG-MIB`'s `arubaWiredConfigurationCopyTable` (source/dest file types include `checkpoint`, `runningConfiguration`, `startupConfiguration`, `externalFile`) — see [mib-reference.md](mib-reference.md).

SignalScope implication: the GUI's "Save" affordance should map to `write memory`/`copy running-config startup-config` (persist-only, like Cisco), but a *separate* "safe apply with rollback" affordance is a natural fit for `checkpoint auto <n>` — distinct from ordinary immediate-apply and worth surfacing as its own GUI concept per the checkpoint/rollback semantics above, not conflated with plain Save.

## Official documentation sources (verified this session)

- [AOS-CX 10.13 SNMP/MIB Guide (PDF, all switch series)](https://arubanetworking.hpe.com/techdocs/AOS-CX/10.13/PDF/snmp_mib.pdf)
- [AOS-CX 10.14 SNMP/MIB Guide (PDF, all switch series)](https://arubanetworking.hpe.com/techdocs/AOS-CX/10.14/PDF/snmp_mib.pdf)
- [AOS-CX 10.15 SNMP/MIB Guide (HTML)](https://arubanetworking.hpe.com/techdocs/AOS-CX/10.15/HTML/snmp_mib/Content/fir-int2.htm)
- [AOS-CX 10.14 Command-Line Interface Guide, 8320/8325 series (PDF)](https://arubanetworking.hpe.com/techdocs/AOS-CX/10.14/PDF/cli_832x.pdf)
- [AOS-CX 10.14 Link Aggregation Guide (LACP/LAG)](https://arubanetworking.hpe.com/techdocs/AOS-CX/10.14/HTML/link_aggregation/Content/Chp_LAG/lac-cnf-set-10.htm)
- [Configuring SNMP — AOS-CX 10.14 Fundamentals Guide, 8400 series](https://arubanetworking.hpe.com/techdocs/AOS-CX/10.14/HTML/fundamentals_8400/Content/Chp_SNMP/cnf-snm.htm)
- [Securing Spanning Tree — AOS-CX 10.15 Hardening Guide](https://arubanetworking.hpe.com/techdocs/AOS-CX/10.15/HTML/hardening/Content/Chp_hard-ctrl/sec-span-tree.htm)
- [MSTP use case: Spanning tree on edge ports — AOS-CX 10.16 L2 Bridging Guide](https://arubanetworking.hpe.com/techdocs/AOS-CX/10.16/HTML/l2_bridging_8400/Content/Chp_stp/use-caseMSTP_SpanEdgePorts.htm)
- [Port security — AOS-CX 10.11 Security Guide, 8360 series](https://www.arubanetworks.com/techdocs/AOS-CX/10.11/HTML/security_8360/Content/Chp_Port_sec/por-sec-fl-10.htm)
- [ArubaOS-CX Checkpoint CLI commands — CLI reference bank (10.07, various models)](https://arubanetworking.hpe.com/techdocs/AOS-CX/10.07/HTML/5200-7839/Content/Chp_Cfg_FW_mgt/Chk_cmds/cop-che-che-nam-ru-con-sta-con.htm)
- [ArubaOS-CX Configuration Checkpoints and Auto Rollback — HPE Airheads community writeup](https://airheads.hpe.com/discussion/arubaos-cx-configuration-checkpoints-and-auto-rollback)

Note: HPE's AOS-CX documentation is versioned per firmware release and per switch-model family (separate PDF/HTML trees for e.g. 6200 vs 8400 vs 832x), and URLs shift between releases (e.g. `techdocs/AOS-CX/10.07/HTML/5200-7839/...` vs the newer `AOSCX-CLI-Bank/cli_8400/...` path scheme). Several exact CLI-bank command pages (e.g. `vlan trunk allowed`, `snmpv3 user`) returned HTTP 403 to direct automated fetch in this session even though they are indexed and readable via search-engine cache/summary — treat command syntax sourced only via search summary (flagged inline below) as "very likely correct, not directly re-verified against the live page" rather than a first-hand read.

## MIB licensing / redistribution note

HPE/Aruba's enterprise MIB source files (the `ARUBAWIRED-*` modules) carry an explicit proprietary header:

> *(c) Copyright Hewlett Packard Enterprise Development LP. All Rights Reserved. The contents of this software are proprietary and confidential to the Hewlett-Packard Development Company, L.P. No part of this program may be photocopied, reproduced, or translated into another programming language without prior written consent...*

confirmed directly from the raw MIB text (retrieved from a third-party redistribution — see [mib-reference.md](mib-reference.md) for provenance). This is materially different from the RFC/IEEE standard MIBs vendored under `signal-scope-docs/mibs/` (those are IETF/IEEE documents with open redistribution norms). **No Aruba enterprise MIB files are vendored in this repo's `mibs/` folder** — the `aruba/mibs/` subfolder is intentionally left empty, and this doc set links to official HPE PDFs/HTML and cites specific OIDs/object names verified against source, rather than copying MIB text. If SignalScope's implementation needs the actual `.mib`/`.my` files for a compiler/browser, obtain them from HPE's own [SNMP/MIB guide downloads](https://arubanetworking.hpe.com/techdocs/AOS-CX/10.14/PDF/snmp_mib.pdf) or HPE's software support portal for the specific firmware version in use, not from this repo.
