# D-Link (DGS-series / xStack managed switches) — Overview

*Appendix-tier vendor doc — single file, less depth than Tier 1-3 vendors. Confirm exact syntax/OIDs against the official model-specific CLI reference guide linked below before relying on them for a live implementation — D-Link's managed-switch CLI is **not one dialect**, see below.*

## CLI dialect summary — two distinct families, not one

Unlike most vendors covered elsewhere in this docs tree, D-Link's managed-switch line spans **two structurally different CLI grammars** depending on product generation/series. SignalScope must detect which family a given device speaks (e.g. via `sysDescr`/`sysObjectID` or a banner probe) rather than assuming one:

| Family | Example series | Shape | Command style |
|---|---|---|---|
| **Classic xStack / native D-Link CLI** | DGS-3000, DGS-3400, DGS-3600, DGS-3100, DES-3xxx | Flat, verb-first, no persistent sub-mode context — every command is a single line naming its own scope (port list, VLAN name, group ID) as an explicit argument, similar in spirit to Extreme's EXOS (no `interface X` context to enter/exit) | `create vlan <name> tag <id>`, `config ports <list> state {enable\|disable}`, `create link_aggregation group_id <n> type lacp`, `config stp ports <list> state {enable\|disable}`, `create snmp community <string> view <name> {read_only\|read_write}` |
| **Newer "Cisco-like" CLI** | DGS-1210/ME, DGS-1510, DGS-3130, DGS-3620, DGS-6600 (metro/enterprise-oriented lines) | Context/sub-mode grammar much closer to Cisco IOS — `configure terminal`, `interface range`, persistent per-interface context | `interface range ethernet 1/0/5-1/0/16`, `switchport mode access`, `switchport access vlan 2`, `switchport trunk allowed vlan tagged 1-3`, `copy running-config startup-config` |

The task brief's framing ("often Cisco-like config/interface-range syntax") matches the **second** family; the classic xStack DGS-3xxx line that the brief also names is actually the **first**, non-Cisco-like family. Both are in active use across the DGS-3xxx naming space depending on exact model/firmware generation — **confirm which dialect a specific model speaks against its own model-specific CLI Reference Guide** before assuming either syntax below.

## SNMP support level

| Version | Support | Notes |
|---|---|---|
| v1 | Yes | Community-string based across both CLI families. |
| v2c | Yes | Same community-string mechanism, adds GetBulk/64-bit counters. |
| v3 | Yes, across current DGS-3xxx/DGS-1xxx lines | USM user/group model — confirmed present in DGS-3000 series (SNMP User Table / SNMP Group Table) and DGS-3100 documentation. Exact auth/priv algorithm list (MD5/SHA, DES/AES) not independently confirmed per-model this session — confirm against the specific model's CLI/Web UI reference guide. |

**Practical write scope**: not independently confirmed this session beyond the standard cross-vendor baseline in [standard-mibs.md](../../00-architecture/standard-mibs.md) (`IF-MIB::ifAdminStatus`, `Q-BRIDGE-MIB` VLAN tables). Treat D-Link SNMP as primarily a **read/monitoring** surface for SignalScope purposes, with CLI as the default write path, pending device-specific confirmation of any broader SNMP SET support.

## Config save model

Both dialect families require an **explicit persist step** — config is applied to the running configuration immediately on each command (immediate-apply, same family as Cisco/Arista/Huawei), but is **not** durable across reboot until separately saved:

| Family | Save command | Notes |
|---|---|---|
| Classic xStack CLI | `save` (bare command, from the config-privileged prompt) | Persists running config to NVRAM; documented across DGS-3000-series CLI reference guides as the standard end-of-session step. |
| Newer Cisco-like CLI | `copy running-config startup-config` | Prompts for destination filename confirmation (`Destination filename startup-config? [y/n]`), then reports `Saving all configurations to NV-RAM ... Done`. A `copy running-config flash:<filename>` variant also exists for saving to a named file rather than the default startup-config. |

This is the same GUI-consequence pattern documented for Cisco/Arista/Huawei in [gui-cli-snmp-unification.md](../../00-architecture/gui-cli-snmp-unification.md): a D-Link "Save" GUI action must emit the model-appropriate persist command as a **visible, distinct terminal line** after the config change(s) — it is never implicit the way Fortinet's `end`-applies-and-persists model is (see `vendors/fortinet/overview.md` for that contrast).

## Curated CLI table

| Action | Classic xStack CLI | Newer Cisco-like CLI |
|---|---|---|
| Enable/disable a port | `config ports <port-list> state {enable\|disable}` (e.g. `config ports 1:1-1:3 state disable`) | `interface range ethernet 1/0/5-1/0/16` → `shutdown` / `no shutdown` |
| VLAN tagged/untagged port membership | `create vlan <name> tag <vlan-id>` then `config vlan <name> add {tagged\|untagged} <port-list>` (e.g. `config vlan Voice1 add tagged 1,23,24`) | `switchport mode {access\|trunk\|hybrid}`, `switchport access vlan <id>` (untagged/access), `switchport trunk allowed vlan tagged <list>` / `switchport hybrid allowed vlan tagged <list>` + `switchport hybrid native vlan <id>` |
| Spanning tree | `config stp ports <port-list> state {enable\|disable}`; global root priority via `config stp priority <value> instance_id 0` | Model-specific — expected under a `spanning-tree` command family analogous to Cisco; **not independently confirmed this session**, confirm against the model's CLI reference. |
| LACP / link aggregation (LAG) | `create link_aggregation group_id <n> type lacp` → `config link_aggregation group_id <n> master_port <port> ports <list> state enabled` → `config lacp_port <port-list> mode {active\|passive}` | Expected under an `interface port-channel`/`channel-group` equivalent per the Cisco-like family; **not independently confirmed this session**, confirm against the model's CLI reference. |
| SNMP community/trap config | `create snmp community <string> view <view-name> {read_only\|read_write}`; trap-host config via a separate `create snmp host` / `config snmp traps` command family (exact syntax not independently confirmed this session) | Expected under an `snmp-server community <string> {ro\|rw}` / `snmp-server host <addr>` equivalent per the Cisco-like family; **not independently confirmed this session**. |

## MIB support

| MIB | Support | Notes |
|---|---|---|
| SNMPv2-MIB, IF-MIB, BRIDGE-MIB, Q-BRIDGE-MIB, LLDP-MIB, RMON-MIB | Expected, standard baseline | Not independently walked against a live D-Link switch this session; confirm per-model via MIB browser walk. ENTITY-MIB support is plausible but not confirmed. |
| **D-Link enterprise MIBs** | Confirmed to exist, PEN **171** | D-Link's IANA Private Enterprise Number is `1.3.6.1.4.1.171`. Confirmed enterprise MIB modules referenced by third-party MIB mirrors include **`DLINK-AGENT-MIB`** (device/agent management objects) and **`DLINKSW-SNMP-MIB`** (switch-specific objects); model-specific MIB files are also published per-product on D-Link's own FTP mirror (e.g. `ftp.dlink.ru/pub/Switch/<model>/.../MIBs/`) rather than a single unified enterprise MIB across the whole DGS line. **Exact object/table names for VLAN or port-admin SNMP write support within these MIBs were not confirmed this session** — treat as "confirm against the model-specific `.mib` file or a MIB browser walk" per this doc's stated uncertainty policy. |

MIB text not vendored in this docs tree (per project convention — link only). Sources: [D-Link MIB browser OID list (PEN 171)](https://mibbrowser.online/mibdb_search.php?search=1.3.6.1.4.1.171.&vendor=D-LINK), [OiDViEW D-Link MIB index](http://www.oidview.com/mibs/171/md-171-1.html), [D-Link's own per-product FTP MIB directories](https://ftp.dlink.ru/pub/Switch/), [LibreNMS vendored D-Link MIBs](https://github.com/librenms/librenms/tree/master/mibs/dlink).

## GUI/CLI/SNMP mapping (illustrative — classic xStack dialect shown; substitute Cisco-like equivalents for DGS-1210/1510/3130/3620/6600)

| GUI Action | CLI | SNMP SET | Read-back | Notes |
|---|---|---|---|---|
| Enable port | `config ports 1:1 state enable` then `save` | `IF-MIB::ifAdminStatus.<ifIndex> = up(1)` (standard baseline; D-Link-specific confirmation not found this session) | `IF-MIB::ifOperStatus` or `show ports 1:1` | Save step required — not auto-persisted. |
| Add port to VLAN (tagged) | `create vlan Sales tag 20` then `config vlan Sales add tagged 1:1` then `save` | `Q-BRIDGE-MIB::dot1qVlanStaticEgressPorts`/`dot1qVlanStaticRowStatus` (standard baseline; D-Link write support not independently confirmed) | `show vlan Sales` | Two-step CLI (create then add ports) unlike single-line Cisco `switchport access vlan`. |
| Set port PVID (untagged/native VLAN) | `config vlan Sales add untagged 1:1` then `save` | `Q-BRIDGE-MIB::dot1qPvid` (standard baseline; not confirmed) | `show vlan Sales` / `show ports 1:1` | — |
| Create LACP trunk | `create link_aggregation group_id 1 type lacp` → `config link_aggregation group_id 1 master_port 1 ports 1-4 state enabled` → `config lacp_port 1-4 mode active` → `save` | Not confirmed | `show link_aggregation group_id 1` | Three-command sequence (create group, assign ports, set LACP mode) rather than one interface-level command. |
| Configure SNMP community | `create snmp community consult view ShowAll read_only` then `save` | N/A (this action configures SNMP itself) | `show snmp community` | — |
| Persist config (GUI "Save") | `save` (classic) / `copy running-config startup-config` (Cisco-like) | N/A | N/A | Must always be echoed as its own terminal line — never implicit, unlike Fortinet. |

## Official sources

- [DGS-3000 Series CLI Reference Guide (v4.00)](https://support.dlink.com/resource/products/DGS-3000-10TC/REVB/DGS-3000-SERIES_REVB_CLI_REFERENCE_GUIDE_v4.00_WW.pdf) — classic xStack dialect, primary source for `config ports`, `create vlan`, `save`.
- [DGS-3400 Series CLI Manual](https://www.dlink.com/-/media/business_products/dgs/dgs-3450/manuals/dgs_3400_series_cli_manual_2_6_en_uk.pdf) — `config ports` port-state/speed syntax example.
- [DGS-3100 Series CLI Manual](https://support.dlink.com/resource/products/dgs-3100-48/reva/DGS-3100-48_CLI_MANUAL_2.20_EN.PDF) — link aggregation command family (`create link_aggregation`, `config lacp_port`).
- [DGS-1510 Series CLI Reference Guide (v1.70)](https://www.dlink-jp.com/product/switch/pdf/DGS-1510_Series_CLI_Reference_Guide_v1.70.pdf) — Cisco-like `interface range`/`switchport` dialect example.
- [DGS-3130 Series CLI Reference Guide](https://www.dlink-jp.com/product/switch/pdf/DGS-3130_Series_A1_CLI-Reference-Guide_v1.10.pdf) — Cisco-like dialect, `copy running-config startup-config` save confirmation.
- [How to Configure VLANs — Example (HTTP and CLI), DGS-1510-Series (D-Link support FAQ)](https://www.dlink.com/uk/en/support/faq/switches/layer-2-gigabit/dgs-series/es_dgs_1510_escenario_config_vlan_por_gui_y_cli)
- [How to Configure Link Aggregation LACP — DGS-1510-28 Series (D-Link support FAQ)](https://www.dlink.com/en/support/faq/switches/layer-2-gigabit/dgs-series/uk_dgs_1510_28_configure_link_aggregation_lacp)
- [Configuring SNMP on D-Link switches (Esia community wiki, third-party but consistent with vendor docs)](https://wiki.esia-sa.com/en/snmp/snmp_dlink_switches)

**Uncertainty flags for implementers**: spanning-tree and LACP command syntax for the newer Cisco-like family, exact SNMP trap-host command syntax for both families, and any SNMP-writable object beyond the standard baseline are **not independently confirmed against a live device or a single authoritative model** this session — confirm against the specific model's CLI Reference Guide (D-Link publishes one per product/revision, not one per series) before wiring up SignalScope's command-emission logic.
