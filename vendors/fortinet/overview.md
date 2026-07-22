# Fortinet (FortiSwitch) — Overview

*Appendix-tier vendor doc — single file, less depth than Tier 1-3 vendors. Confirm exact syntax/OIDs against the official references linked below before relying on them for a live implementation.*

## Scope note: standalone vs. FortiLink-managed

FortiSwitch units are most commonly deployed **FortiLink-managed** — physically a FortiSwitch, but administered entirely through the parent FortiGate's GUI/CLI (`config switch-controller managed-switch` on the FortiGate), with the FortiGate pushing config down over the FortiLink control channel. They also support a **standalone CLI** (direct SSH/console/Telnet to the switch itself, `config switch ...` tree) that is structurally the same FortiOS-style CLI grammar. This doc covers the **standalone CLI**, since that's the direct-session case relevant to SignalScope's connectivity model (see [connectivity-methods.md](../../00-architecture/connectivity-methods.md)); where a FortiSwitch is FortiLink-managed, SignalScope's session should target the FortiGate instead, using its `config switch-controller` equivalents — noted per-row below where it differs.

**Phase 3 scope note (2026-07-22):** FortiLink-managed is not just an alternate mode — it's the dominant real-world deployment shape for FortiSwitch, which makes this vendor structurally the same **controller/GUI-primacy** case already documented for Ubiquiti (see the third asymmetry axis in [`00-architecture/gui-cli-snmp-unification.md`](../../00-architecture/gui-cli-snmp-unification.md)): the standalone CLI documented below is real and directly usable, but for most fielded units the FortiGate, not the switch itself, is the actual system of record. This is why Fortinet stays appendix tier rather than being promoted to full tier despite genuine enterprise deployment — the higher-value depth investment, if SignalScope pursues Fortinet further, is a FortiGate/FortiLink-mediated integration doc, not a deeper standalone-CLI reference (see `ROADMAP.md` Phase 3 for the full decision record).

## CLI dialect summary — FortiOS-style block syntax

FortiSwitchOS/FortiOS CLIs are **not** Cisco-style line commands. They use a distinctive nested **`config` / `edit` / `next` / `end`** block grammar:

```
config switch interface
    edit "port5"
        set native-vlan 10
        set allowed-vlans 10,20,30
        set stp-state enabled
    next
end
```

| Token | Role |
|---|---|
| `config <table>` | Enters a configuration object/table context (e.g. `switch interface`, `system interface`, `switch trunk`). Closed with a matching `end`. |
| `edit <entry>` | Inside a `config` table, opens (or creates) one row/entry by name — analogous to Cisco's `interface GigabitEthernet0/1` sub-mode, but explicitly bounded by a matching `next`, not implicit until the next top-level command. |
| `set <field> <value>` | Sets a field on the currently-`edit`-ed entry, or on a scalar `config` block that has no `edit` layer (e.g. `config system snmp sysinfo`). |
| `next` | Closes the current `edit` block and returns to the `config` table context, ready for another `edit`. |
| `end` | Closes the current `config` table context, applying the block. |

This nesting can go several levels deep (e.g. `config switch interface` → `edit <port>` → `config allowed-vlans` sub-table). The `tree` command (from the CLI root) prints the full command hierarchy and field constraints for a session — useful for confirming exact field names/limits that aren't reproduced here. This is structurally quite different from Cisco/Arista/Huawei-style flat line commands where mode is tracked implicitly by prompt (`hostname(config-if)#`); here the block is explicit and self-closing, which SignalScope's session-state tracker should treat as a stack (each `config`/`edit` pushes a context, each `end`/`next` pops one).

## Config save model — auto-save, no separate "write memory"

**Confirmed**: FortiOS/FortiSwitchOS applies and persists configuration **as soon as the `end` of the enclosing `config` block executes** — there is no separate `write memory` / `copy running-config startup-config` step the way Cisco/Arista/Huawei require. The FortiSwitchOS CLI reference states configuration changes are written into the configuration database as they're made; the practical behavior documented across FortiOS/FortiSwitch admin guides is that a completed `edit ... next ... end` sequence is durable immediately. This is a **notable cross-vendor difference** SignalScope must model explicitly per [connectivity-methods.md](../../00-architecture/connectivity-methods.md)'s save-semantics point: a GUI "Save" action on a Fortinet device has **no corresponding CLI line to echo** beyond the `end` that already closed the block — there is no separate persist command to append, unlike Cisco (`write mem`) or D-Link (`save`). SignalScope's terminal-echo logic should not synthesize a fake save command for Fortinet; the last `end` already *is* the save.

## SNMP support level

| Version | Support | Notes |
|---|---|---|
| v1 | Yes | Community-string based, via `config system snmp community`. |
| v2c | Yes | Same community-config command family; adds GetBulk. |
| v3 | Yes, from FortiSwitchOS 7.0.0 onward | USM users configured via `config system snmp user` (standalone) or `config switch-controller snmp-user` (FortiLink-managed via FortiGate). Auth MD5/SHA family, priv DES/AES family — exact algorithm list varies by firmware release; confirm against the release-specific CLI reference. |

**Practical write scope**: treat FortiSwitch SNMP as **read-oriented in practice**. Fortinet's documented SNMP configuration surface (community/trap/user setup) is about making the switch pollable and trap-capable, not about exposing broad config-writable objects. No FortiSwitch-specific documentation was found this session confirming a general SNMP SET path for VLAN membership or port admin-status beyond the cross-vendor `IF-MIB`/`Q-BRIDGE-MIB` baseline in [standard-mibs.md](../../00-architecture/standard-mibs.md) — SignalScope should assume **CLI-only** for config changes on Fortinet devices unless a specific object is separately confirmed live.

## Curated CLI table

| Action | Standalone FortiSwitch CLI | Notes |
|---|---|---|
| Enable/disable a port | `config switch interface` → `edit <port>` → `set status {up\|down}` → `next` → `end` | Field name is `status`, not `admin-status`; confirm exact field name against the version-specific reference — some releases expose this under `config switch physical-port` instead. |
| VLAN tagged/untagged port membership | `config switch interface` → `edit <port>` → `set native-vlan <vlan-id>` (untagged/PVID) + `set allowed-vlans <list>` (tagged trunk membership) → `next` → `end` | `allowed-vlans` takes a comma-separated list/range; a port's untagged VLAN is `native-vlan`, distinct from the tagged `allowed-vlans` set — same conceptual split as `dot1qPvid` vs. `dot1qVlanStaticEgressPorts`/`UntaggedPorts` in Q-BRIDGE-MIB. |
| Spanning tree (STP/RSTP/MSTP, per-port) | `config switch interface` → `edit <port>` → `set stp-state {enabled\|disabled}` → optionally `set loop-guard {enabled\|disabled}`, `set edge-port {enabled\|disabled}` → `next` → `end` | Global STP mode config lives elsewhere in the tree (`config switch stp settings` family per release) — confirm against the CLI reference for the exact global-vs-per-port split on your firmware version. |
| LACP / link aggregation (LAG/trunk) | `config switch trunk` → `edit <trunk-name>` → `set mode {static\|lacp-active\|lacp-passive}` → `set members <port-list>` → optionally `set lacp-speed {fast\|slow}`, `set port-selection-criteria {src-ip\|dst-ip\|src-dst-ip\|src-mac\|dst-mac\|src-dst-mac}` → `next` → `end` | The task brief's `config system link-monitor` is a FortiGate SD-WAN/HA health-check construct, **not** the FortiSwitch LAG mechanism — the correct standalone-FortiSwitch construct is `config switch trunk`, confirmed via the FortiSwitch Administration Guide's "Link aggregation groups" section. |
| SNMP community (v1/v2c) | `config system snmp community` → `edit <id>` → `set name <string>` → `config hosts` → `edit <id>` → `set ip <addr> <mask>` → `set interface <name>` → `next` → `end` → `end` → `next` → `end` | Nested `hosts` sub-table per community entry restricts which NMS IPs may query that community — not a single flat community string like classic Cisco. |
| SNMP trap config | Same `config system snmp community` entry → `set events {cpu-high mem-low log-full intf-ip ent-conf-change ...}` (space-separated flag list) → also `config system snmp sysinfo` → `set trap-high-cpu-threshold <pct>` etc. for specific trap thresholds | Trap event selection is per-community, not a separate global trap-list command. |

## MIB support

| MIB | Support | Notes |
|---|---|---|
| SNMPv2-MIB, IF-MIB, BRIDGE-MIB, Q-BRIDGE-MIB, LLDP-MIB, ENTITY-MIB, RMON-MIB | Expected, standard baseline | Not independently walked against a live FortiSwitch this session; treat per [standard-mibs.md](../../00-architecture/standard-mibs.md) baseline assumption and confirm via MIB browser walk before depending on specific sub-OIDs. |
| **FORTINET-FORTISWITCH-MIB** | Confirmed to exist | Rooted at `fnFortiSwitchMib`, `1.3.6.1.4.1.12356.106` (Fortinet's IANA Private Enterprise Number is **12356**). Ships as part of the FortiOS enterprise MIB set; documented tables include `FgSwDeviceEntry` (info about FortiSwitches connected to a FortiGate via FortiLink) and `FgSwPortEntry` (per-port switch info) — i.e. this MIB is oriented toward a **FortiGate polling its managed FortiSwitches**, not necessarily a standalone FortiSwitch's own agent. Confirm scope (standalone-agent-walkable vs. FortiGate-only) against the downloadable `.mib` file before assuming SignalScope can walk it directly against a standalone unit. |
| FORTINET-FORTIGATE-MIB / FORTINET-CORE-MIB | Confirmed to exist, PEN 12356 | Broader FortiOS enterprise MIB umbrella; relevant if SignalScope ever polls the parent FortiGate rather than the switch directly. |

MIB text not vendored in this docs tree (per project convention — link only); obtain via Fortinet's [MIB download page](https://docs.fortinet.com/document/fortigate/7.6.0/fortigate-mib-information-overview) or a MIB-browser mirror (e.g. [Observium's FORTINET-FORTISWITCH-MIB mirror](https://mibs.observium.org/mib/FORTINET-FORTISWITCH-MIB/)) and confirm against a live device walk.

## GUI/CLI/SNMP mapping (illustrative — standalone FortiSwitch)

| GUI Action | CLI | SNMP SET | Read-back | Notes |
|---|---|---|---|---|
| Enable port | `config switch interface` / `edit <port>` / `set status up` / `next` / `end` | `IF-MIB::ifAdminStatus.<ifIndex> = up(1)` (standard baseline; not Fortinet-specific-confirmed) | `IF-MIB::ifOperStatus` or `get switch physical-port` | No separate save step — `end` persists immediately. |
| Add port to VLAN (tagged) | `config switch interface` / `edit <port>` / `set allowed-vlans <list>` / `next` / `end` | Not confirmed as an SNMP-writable Fortinet object this session | `diagnose switch vlan` / `get switch vlan` or re-read `allowed-vlans` field | Treat as CLI-only until an SNMP write path is separately verified. |
| Set port native/PVID | `config switch interface` / `edit <port>` / `set native-vlan <id>` / `next` / `end` | Not confirmed | Re-read `native-vlan` field | Same caveat as above. |
| Create LACP trunk | `config switch trunk` / `edit <name>` / `set mode lacp-active` / `set members <ports>` / `next` / `end` | Not confirmed | `get switch trunk` | No FortiSwitch-specific SNMP LAG-write object found this session. |
| Configure SNMP community | `config system snmp community` / `edit <id>` / `set name <str>` / (nested `hosts`) / `end` | N/A (this action configures SNMP itself) | `get system snmp community` | — |
| Persist config (GUI "Save") | *(no distinct command — already done by the `end` that closed the preceding block)* | N/A | N/A | The one meaningfully different row vs. every immediate-apply-then-separately-persist vendor: there is nothing to echo here. |

## Official sources

- [FortiSwitchOS CLI Reference — Introduction (7.6.4)](https://docs.fortinet.com/document/fortiswitch/7.6.4/fortiswitchos-cli-reference/608648/introduction) — `config`/`edit`/`next`/`end` grammar, `tree` command.
- [FortiSwitchOS CLI Reference — `config switch`](https://docs.fortinet.com/document/fortiswitch/7.6.4/fortiswitchos-cli-reference/511852/config-switch)
- [Link aggregation groups — FortiSwitch 7.2.10 Administration Guide](https://docs.fortinet.com/document/fortiswitch/7.2.10/administration-guide/352388/link-aggregation-groups) — `config switch trunk` LACP syntax.
- [Configuring SNMP — FortiSwitch 7.6.4 FortiLink Guide](https://docs.fortinet.com/document/fortiswitch/7.6.4/fortilink-guide/173288/configuring-snmp)
- [FortiGate switch MIBs — FortiGate/FortiOS 7.6.0](https://docs.fortinet.com/document/fortigate/7.6.0/fortigate-mib-information-overview/508932/fortigate-switch-mibs) — `FgSwDeviceEntry`/`FgSwPortEntry`, PEN confirmation.
- [OID repository: 1.3.6.1.4.1.12356.106 (fnFortiSwitchMib)](https://oid-base.com/get/1.3.6.1.4.1.12356.106)
- FortiSwitchOS CLI Reference PDFs (versioned): [7.4.3](https://fortinetweb.s3.amazonaws.com/docs.fortinet.com/v2/attachments/b24bbfd3-fc4c-11ee-8c42-fa163e15d75b/FortiSwitchOS-7.4.3_CLI_Reference.pdf), [6.2.0](https://fortinetweb.s3.amazonaws.com/docs.fortinet.com/v2/attachments/62d0ff3d-5fbd-11e9-81a4-00505692583a/fortiSwitchOS-6.2.0-CLI-ref.pdf) — useful for cross-checking exact field names/defaults per firmware version, since this doc does not pin a single version.

**Uncertainty flags for implementers**: exact field name for per-port admin status (`status` vs. a `physical-port` sub-tree field), the precise global-STP-mode command path, and any SNMP-writable object beyond the standard baseline are **not independently confirmed against a live device or a single authoritative version** this session — confirm against the version-pinned CLI reference for the target firmware before wiring up SignalScope's command-emission logic.
