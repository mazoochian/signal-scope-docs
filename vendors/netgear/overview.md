# Netgear (ProSAFE / M-series / GS-series switches) — Overview

*Appendix-tier vendor doc — single file, less depth than Tier 1-3 vendors (per [`ROADMAP.md`](../../ROADMAP.md) Phase 0 decision, 2026-07-16). Confirm exact syntax/OIDs against the official CLI reference guide linked below before relying on them for a live implementation.*

## Product-line fragmentation — confirms the roadmap's "verify during research" flag

The roadmap tentatively assumed Netgear would split across CLI generations like D-Link; that assumption is confirmed, but the actual split is by **product tier**, not firmware generation:

| Line | CLI support | Management model |
|---|---|---|
| **"Smart Managed" / "Smart Managed Plus" / GS-series** (e.g. GS108T, GS724T, GS728T family) | **Web GUI only, officially.** Netgear's own documentation states these switches "don't support CLI access officially." An unofficial telnet workaround exists (Maintenance → Troubleshooting → Remote Diagnostics), explicitly framed by Netgear as diagnostic/development-only, not a supported management path. Select newer models/firmware (GS728TPv2/GS728TPPv2/GS752TPv2/GS752TPP 6.0.10.5+, GS308T/GS310TP 1.0.5.5+, GS108Tv3/GS110TPv3 7.0.9.5+, and others) have gained an official but reduced **"Smart Lite CLI."** |
| **ProSAFE M-series / "Intelligent Edge" (M4100, M4200, M4300, M5xxx)** | **Full, officially-supported CLI**, alongside SNMP v1/v2c/v3 and RMON. This is the line SignalScope should target for direct-CLI switch management — this doc's CLI/SNMP content below is scoped to this line. | |

This is a real structural difference from D-Link's split (two CLI *dialects*, both fully supported) — here it's "full CLI exists on the higher tier; the lower/cheaper tier is GUI-managed by design, with an unofficial or intentionally-limited CLI as a partial exception." SignalScope's vendor-capability detection for Netgear devices should check product line/model before assuming CLI reachability at all, not just before picking a dialect.

## CLI dialect (M-series / Intelligent Edge scope)

Mode-based, IOS-adjacent CLI (`configure`/interface sub-modes, `shutdown`/`no shutdown` presumed by convention though not independently re-confirmed this session) with a `<unit>/<slot>/<port>` three-part interface naming scheme confirmed for the M4300 (e.g. `interface 1/0/1`). The confirmed config-save command — `copy system:running-config nvram:startup-config` — uses a distinctive `system:`/`nvram:` filesystem-prefix notation rather than the bare `copy running-config startup-config` form used by Cisco/Arista/Huawei. This notation, combined with the `vlan participation include/exclude`-style VLAN command family referenced in Netgear community CLI cheat-sheets, is consistent with the same **Broadcom "FASTPATH" reference-CLI lineage** already identified in [`vendors/ubiquiti/overview.md`](../ubiquiti/overview.md) for Ubiquiti's EdgeSwitch-derived CLI — worth treating as a real cross-vendor pattern: several SMB/mid-market switch vendors license the same underlying Broadcom reference software stack, which explains structurally similar (but not identical) CLI shapes appearing under different vendor names. Not independently confirmed this session whether Netgear's `vlan` command family matches Ubiquiti's exactly — flag for a follow-up pass if this vendor is ever promoted to full tier.

## SNMP

- **Enterprise PEN confirmed: 4526** (`1.3.6.1.4.1.4526`), corroborated by multiple independent sources (community MIB mirrors, OID-lookup sites) this session.
- **v1, v2c, v3 confirmed supported on the M-series** (Netgear's own M4100 product documentation states "SNMP (v1, v2c and v3), RMON, Command Line Interface (CLI)" as a supported feature set).
- **Private MIBs are namespaced with a leading hyphen** per a community-sourced description of Netgear's MIB set ("all private MIBs beginning with a hyphen (-) prefix") — not independently re-confirmed against a primary Netgear source this session, but a distinctive-enough claim worth flagging for verification since no other vendor in this project has been described this way.
- **Practical write scope**: not independently confirmed this session beyond the standard cross-vendor baseline in [`standard-mibs.md`](../../00-architecture/standard-mibs.md). Treat as CLI-first per this docs tree's standing default, pending specific per-object confirmation.
- **MIB distribution**: Netgear hosts per-model MIB downloads on `support.netgear.com` (confirmed via Netgear's own "MIBs for Smart switches" KB article) as free downloads with **no explicit license or terms of use stated** on the distribution page — the same situation as Huawei, Extreme, and Dell (see those vendors' `mibs/README.md` files for the identical reasoning). No Netgear MIB files are vendored in this docs tree — link only, sources below.

## Illustrative GUI/CLI/SNMP mapping (M-series / Intelligent Edge scope)

| GUI Action | CLI | SNMP SET | Read-back | Notes |
|---|---|---|---|---|
| Enable/disable a port | `interface 1/0/1` → `no shutdown` / `shutdown` (convention assumed from IOS-adjacent shape; not independently re-confirmed this session) | Not confirmed | `show interface 1/0/1` (exact form not independently confirmed) | **CLI-only pending confirmation**, consistent with this docs tree's default posture. |
| VLAN configuration | `vlan participation include <n>` / `vlan tagging <n>` style, by analogy to the confirmed-shared FASTPATH lineage with Ubiquiti (see above) — **not independently confirmed for Netgear specifically** this session | Not confirmed | `show vlan` | Treat the exact command family as unconfirmed-by-analogy, not confirmed, until read directly from Netgear's own CLI reference PDF. |
| Persist configuration | `copy system:running-config nvram:startup-config` | Not confirmed | `show running-config` vs. stored startup-config diff | Confirmed command, distinctive `system:`/`nvram:` prefix notation not seen on any other vendor in this project. |

## Official sources (this session)

- [M4200 and M4300 Series ProSAFE/Intelligent Edge Managed Switches — CLI Command Reference Manual (PDF, downloads.netgear.com)](https://www.downloads.netgear.com/files/GDC/M4200/M4200-M4300_CLI_EN.pdf)
- [NETGEAR M4300 Intelligent Edge Series CLI Command Reference Manual (software versions through 12.0.11)](https://manuals.plus/m/ff9bc7ca221a674728173c5af6974089f9472b45e16063bf5e2a2fbd480f1638)
- [M4300 — want to save config from cli (tftp) — NETGEAR Communities](https://community.netgear.com/discussions/business-managed-switches/m4300---want-to-save-config-from-cli-tftp/2340778) — confirms the `copy system:running-config nvram:startup-config` save syntax.
- [MIBs for Smart switches — NETGEAR Support KB 24352](https://kb.netgear.com/24352/MIBs-for-Smart-switches) — MIB distribution, no stated license.
- [Which SNMP versions and MIBs does my NETGEAR GS728TPv2/GS728TPPv2/GS752TPv2/GS752TPP switch support? — NETGEAR Support](https://kb.netgear.com/000058225/Which-SNMP-versions-and-MIBs-does-my-NETGEAR-GS728TPv2-GS728TPPv2-GS752TPv2-or-GS752TPP-switch-support)
- [How to allow CLI (SSH & Telnet access) on NetGear Prosafe smart switches — Network Automator](https://networkautomator.com/2022/06/26/how-to-allow-cli-ssh-telnet-access-on-netgear-prosafe-smart-switches/) — corroborates the Smart-Managed-tier CLI-access limitation.
- [Manage Your Network Easier with NETGEAR Smart Switch CLI — netgear.com](https://www.netgear.com/hub/business/enterprise-it/smart-switch-lite-cli/) — "Smart Lite CLI" model/firmware list for the lower tier.
- [Netgear-MIB (PEN 4526) — OiDViEW MIB index](http://www.oidview.com/mibs/4526/Netgear-MIB.html)

**Uncertainty flags for implementers**: this vendor's syntax (beyond the confirmed save command and product-tier CLI-availability split) was researched via search-summary results rather than a full primary CLI-reference-PDF read — every command marked "not independently confirmed" above should be re-verified against the linked M4200/M4300 CLI Command Reference Manual PDF directly before production use. The shared-FASTPATH-lineage claim with Ubiquiti is an inference from structural similarity (interface-prefix notation style, VLAN command family naming), not a confirmed fact about Netgear's specific implementation.
