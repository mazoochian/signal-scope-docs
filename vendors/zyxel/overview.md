# Zyxel (GS1900 / XGS/GS-standalone managed switches) — Overview

*Appendix-tier vendor doc — single file, less depth than Tier 1-3 vendors (per [`ROADMAP.md`](../../ROADMAP.md) Phase 0 decision, 2026-07-16). Confirm exact syntax/OIDs against the official CLI reference guide linked below before relying on them for a live implementation.*

## Product-line fragmentation — same pattern as Netgear, one tier more extreme

Confirms the roadmap's "verify during research" flag, and lands on the identical structural split already documented for [Netgear](../netgear/overview.md) — entry-tier "smart managed" hardware is GUI-first with limited-to-no CLI, full CLI exists only on the higher tier:

| Line | CLI support | Management model |
|---|---|---|
| **GS1900 series** ("GbE Smart Managed Switch") | **No configuration CLI at all — confirmed directly, not inferred.** SSH access exists but is limited to read-only diagnostic commands (`show running-config`, `ping`); attempting `configure terminal` returns `Unknown command`, and community reports confirm **the GS1900 cannot enter configuration mode via CLI under any command spelling** — every config change must go through the web GUI. This is a stricter limitation than Netgear's equivalent entry tier, which at least has an unofficial telnet workaround and an emerging "Smart Lite CLI" on some models — Zyxel's GS1900 has no config-write CLI path whatsoever. |
| **XGS4600 / GS19x0 / GS2210 / GS3700 "standalone-managed" line** | **Full CLI**, IOS-adjacent (`configure`, `vlan <n>`, `interface port-channel <n>`, `write memory`), alongside the web GUI. This is the line SignalScope should target for direct-CLI Zyxel switch management — this doc's CLI/SNMP content below is scoped to this line. |
| **NebulaFlex-capable models** (e.g. some XS1930 units) | Can switch between **standalone** (local CLI/GUI, as above) and Zyxel's **Nebula cloud management platform** (license-free cloud controller) | A lighter-weight version of the same controller-vs-direct-device fork already documented in depth for [Ubiquiti](../ubiquiti/overview.md) — not investigated further this session, but if SignalScope encounters a Nebula-managed Zyxel device, expect the same class of "durable state lives in the controller, not necessarily the device" consideration flagged there, and treat it as unconfirmed-by-default rather than assuming Nebula behaves identically to UniFi's controller model. |

**SignalScope implication**: exactly as with Netgear, capability detection for a Zyxel device must check product line before assuming CLI reachability — and for GS1900 specifically, must not offer a CLI fallback path for configuration actions **at all**, only for read-only diagnostics.

## CLI dialect (standalone-managed line scope)

Mode-based, IOS-adjacent: `configure` (not `configure terminal` — the shorter form is what's confirmed for this vendor) enters config mode; `vlan <n>` creates/enters a VLAN; `interface port-channel <n>` enters a port-channel's interface context, where per-interface settings like `pvid` (port VLAN ID) are set. `write memory` persists to non-volatile storage — same shorthand-save convention as Cisco/Arista/Dell OS10 rather than Huawei's longer `save` (interactive) or Aruba's `copy running-config startup-config`. Exact physical-interface naming convention (whether it's a bare port number, `port <n>`, or a slot/port scheme) was **not independently confirmed this session** — verify against the full CLI reference manual before generating for a GUI action.

## SNMP

- **Enterprise PEN confirmed: 890** (`zyxel OBJECT IDENTIFIER ::= { enterprises 890 }`), corroborated across multiple independent MIB-mirror sources this session (Observium, OiDViEW, a GitHub-hosted `ZYXEL-MIB` mirror).
- Multiple named enterprise MIB modules confirmed to exist by name: `ZYXEL-MIB`, `ZYXEL-SNMP-MIB`, `ZYXEL-AS-MIB`, `ZYXEL-ZYWALL-MIB`, and per-model modules (e.g. `ZYXEL-GS2200-24-MIB`) — internal object/table structure of any of these **not independently examined this session**. Zyxel also ships device-specific MIB files directly with some firmware bundles (community reference to `<device-id>-enterprise.mib`/`<device-id>-private.mib` naming) — a per-device rather than per-product-line distribution model not seen on any other vendor in this project.
- SNMP version support (v1/v2c/v3) and write scope **not independently confirmed this session** for either product tier — given the GS1900's confirmed total absence of a config-write CLI path, its SNMP write scope (if any) would be architecturally significant (potentially the *only* programmatic config-write path for that tier) and is worth prioritizing in a follow-up research pass over the standalone-managed line's SNMP, where CLI is already the more natural target.
- **MIB distribution**: multiple third-party mirrors exist (Observium, OiDViEW, LibreNMS, a standalone `Poil/MIBs` GitHub repo); **no first-party Zyxel MIB portal with stated redistribution terms was located this session** — same no-explicit-license situation as Huawei/Extreme/Dell/Netgear. No Zyxel MIB files are vendored in this docs tree — link only, sources below.

## Illustrative GUI/CLI/SNMP mapping (standalone-managed line scope)

| GUI Action | CLI | SNMP SET | Read-back | Notes |
|---|---|---|---|---|
| Enable/disable a port | Not independently confirmed this session (exact keyword/interface-naming pattern unresolved) | Not confirmed | `show running-config` (confirmed to exist as a read-back command, per the GS1900 diagnostic-CLI finding above — likely available on the full-CLI tier too) | **CLI-only pending confirmation**, consistent with this docs tree's default posture. |
| VLAN configuration | `configure` → `vlan <n>` → (per-interface `pvid <n>` inside `interface port-channel <n>` context, for the port-channel case at least) | Not confirmed | Not independently confirmed | Confirmed command family exists; exact physical (non-port-channel) interface VLAN-assignment syntax not independently confirmed this session. |
| Persist configuration | `write memory` | Not confirmed | N/A | Confirmed command. |
| Any configuration action on GS1900-tier hardware | **No CLI path exists — web GUI only** | Unconfirmed, but architecturally important if it exists (see SNMP section above) | Web GUI / possibly SNMP GET | The one case in this docs tree (alongside Ubiquiti's controller-durability finding) where "CLI-only" is not an option SignalScope can fall back to — a GS1900-tier device may need to be GUI-scraped or SNMP-driven exclusively, or simply flagged as a device class SignalScope cannot fully manage via its CLI-first architecture. |

## Official / primary sources (this session)

- [\[Switch\] How to configure VLAN on GS1900-xx switches — Zyxel Support Campus USA](https://mysupport.zyxel.com/hc/en-us/articles/360008607580--Switch-How-to-configure-VLAN-on-GS1900-xx-switches-firmware-2-40-and-newer) — GS1900 web-GUI-only VLAN configuration.
- [Configure Terminal Switch GS1900 — Zyxel Community](https://community.zyxel.com/en/discussion/10877/configure-terminal-switch-gs1900) — confirms `configure terminal` fails with "Unknown command" on GS1900.
- [GS1900-8 Switch CLI Reference Guide — Zyxel Community](https://community.zyxel.com/en/discussion/6861/gs1900-8-switch-cli-reference-guide) — confirms the GS1900's SSH access is diagnostic-only (`show running-config`, `ping`).
- [Network Switch - Configure VLAN GS/XGS-Series — Zyxel Support Campus EMEA](https://support.zyxel.eu/hc/en-us/articles/4407129230738-Network-Switch-Configure-VLAN-GS-XGS-Series)
- [Zyxel switch basic setup — Mervin Praison](https://mer.vin/2020/02/zyxel-switch-basic-setup/) — `configure`/`vlan`/`interface port-channel`/`write memory` command sequence.
- [ZYXEL COMMUNICATIONS ETHERNET SWITCH CLI REFERENCE MANUAL — ManualsLib](https://www.manualslib.com/manual/771667/Zyxel-Communications-Ethernet-Switch.html) — full CLI reference for the standalone-managed line; not directly fetched this session, recommended as the primary source for a follow-up pass.
- [Zyxel XGS4600-52F — guide to Accessing and Using the Command Line Interface — mans.io](https://mans.io/files/viewer/1246345/10)
- [ZYXEL-MIB — Observium MIB Browser](https://mibs.observium.org/mib/ZYXEL-MIB/) and [OiDViEW Zyxel MIB index (PEN 890)](https://www.oidview.com/mibs/890/md-890-1.html)
- [Zyxel Switch Quick Sales Guide: Portfolio and Features](https://manuals.plus/m/9a0a3e1ca053880c2bbcb762de3ef9a12928a82b4847178add9302f47b70e9db) — product-line positioning (GS1900 vs. XGS4600 vs. NebulaFlex/XS1930).

**Uncertainty flags for implementers**: this vendor's standalone-managed-line CLI syntax (beyond the confirmed `configure`/`vlan`/`write memory` command family) was researched via search-summary results rather than a full primary CLI-reference-manual read — the linked ManualsLib CLI reference should be read directly before implementation. The GS1900's total CLI-config-absence finding is the strongest-confirmed claim in this doc (multiple independent community sources agree); its SNMP write-scope status, which would materially change SignalScope's options for that device tier, is the single highest-priority open question for this vendor.
