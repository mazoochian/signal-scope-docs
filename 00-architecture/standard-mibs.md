# Standard (RFC/IEEE) MIBs Shared Across Vendors

Before any vendor-specific enterprise MIB applies, every switch that speaks SNMP implements a common substrate of standard MIBs. SignalScope should treat these as the baseline read/write surface that works identically regardless of vendor, and only reach into a vendor's `mib-reference.md` for things these don't cover. Raw `.mib` text for all of these is vendored once under `signal-scope-docs/mibs/` rather than duplicated per vendor folder.

OID prefix note: all standard MIB-II objects live under `iso.org.dod.internet.mgmt.mib-2` = **`1.3.6.1.2.1`**. Exact numeric sub-indices below are given where well-established; treat any not given as "confirm against the vendored `.mib` file or a MIB browser" rather than assumed.

## SNMPv2-MIB (RFC 3418) — always poll first

The `system` group, `1.3.6.1.2.1.1`: `sysDescr` (.1), `sysObjectID` (.2, identifies vendor+model via a private enterprise OID — this is the fastest way to auto-detect vendor without CLI banner-scraping), `sysUpTime` (.3), `sysContact` (.4), `sysName` (.5), `sysLocation` (.6), `sysServices` (.7). This is what SignalScope's device-discovery flow should query first against any newly-added IP to confirm SNMP reachability and identify the device before anything else.

## IF-MIB (RFC 2863) — interface state and counters

The single most important MIB for SignalScope's per-interface GUI actions.
- `ifTable` (`1.3.6.1.2.1.2.2.1`): `ifIndex` (.1), `ifDescr` (.2), `ifType` (.3), `ifMtu` (.4), `ifSpeed` (.5, 32-bit — superseded by `ifHighSpeed` below), `ifPhysAddress` (.6, MAC), **`ifAdminStatus`** (.7, up/down/testing(1/2/3) — the object SignalScope's "enable/disable port" GUI action SETs), **`ifOperStatus`** (.8, read-only actual link state — used for GUI/telemetry read-back after any admin-status change), `ifLastChange` (.9), plus 32-bit octet/error/discard counters (.10-.20).
- `ifXTable` (`1.3.6.1.2.1.31.1.1.1`) extends the above: `ifName` (.1, the short interface name matching CLI syntax, e.g. `Gi0/1` — useful for correlating SNMP index to CLI-typed interface names), `ifHCInOctets`/`ifHCOutOctets` (64-bit counters, required for accurate throughput on modern link speeds), `ifHighSpeed` (.15, Mbps as a 32-bit value, avoids `ifSpeed` overflow), `ifAlias` (.18, the user-settable description string — maps to CLI `description` command), `ifConnectorPresent`, `ifPromiscuousMode`.
- Indexing note: `ifIndex` values are **not guaranteed stable across reboots** on all vendors (though most modern implementations persist them). SignalScope should resolve `ifIndex` from `ifName`/`ifDescr` at session start rather than caching it long-term.

## BRIDGE-MIB (RFC 4188) — bridging, spanning tree, MAC table

Rooted at `dot1dBridge`, `1.3.6.1.2.1.17`.
- `dot1dBase` group (`.1`): `dot1dBaseBridgeAddress`, `dot1dBaseNumPorts`, and `dot1dBasePortTable` mapping an internal bridge-port number to `ifIndex` via `dot1dBasePortIfIndex` — necessary because spanning-tree/bridging objects are indexed by bridge-port number, not `ifIndex` directly.
- `dot1dStp` group (`.2`): classic (802.1D) spanning-tree state — `dot1dStpProtocolSpecification`, `dot1dStpDesignatedRoot`, `dot1dStpRootCost`, `dot1dStpRootPort`, and per-port `dot1dStpPortTable` (`dot1dStpPortState`: disabled/blocking/listening/learning/forwarding/broken, `dot1dStpPortEnable`, `dot1dStpPortPathCost`). Most modern switches run RSTP/MSTP and expose extended objects in vendor MIBs or `P-BRIDGE-MIB`/`Q-BRIDGE-MIB` rather than pure 802.1D, but this group remains the read-only baseline for "what state is this port's spanning tree in" that works across nearly every vendor.
- `dot1dTp` group (`.4`): the transparent-bridging forwarding database — `dot1dTpFdbTable` (learned MAC address table: `dot1dTpFdbAddress`, `dot1dTpFdbPort`, `dot1dTpFdbStatus`). This is what backs a GUI "MAC address table" view without needing to scrape `show mac address-table` over CLI.

## Q-BRIDGE-MIB (RFC 4363) — VLANs

Rooted at `qBridgeMIB`, `mib-2.46` (`1.3.6.1.2.1.46`) — note this is a **separate arc from BRIDGE-MIB**, not a sub-tree of `dot1dBridge`, despite the close conceptual relationship.
- `dot1qVlanStaticTable`: the primary VLAN read/write surface — `dot1qVlanStaticName`, `dot1qVlanStaticEgressPorts` (bitmask of ports that are members, tagged or untagged), `dot1qVlanStaticUntaggedPorts` (bitmask of the subset that are untagged/access), `dot1qVlanStaticRowStatus` (RFC 2579 RowStatus SET semantics for creating/destroying a VLAN via SNMP — `createAndGo(4)`/`destroy(6)`). This is the object a GUI "create VLAN" / "add port to VLAN" action would SET, where a vendor supports SNMP-driven VLAN management at all (support varies — see per-vendor `mib-reference.md`).
- `dot1qPvid` (in `dot1qPortVlanTable`): the port's native/access VLAN (PVID) — maps to CLI `switchport access vlan <n>` (Cisco-family) or equivalent.
- `dot1qVlanCurrentTable`: read-only combined view of static+dynamic (GVRP/MVRP-learned) VLAN membership — used for read-back/polling rather than writes.

## LLDP-MIB (IEEE 802.1AB)

Rooted under the IEEE-assigned arc `1.0.8802.1.1.2` (not under `mib-2` — this is an IEEE 802.1 standard MIB, not IETF). This is the primary standards-based source for **topology discovery** — directly relevant to SignalScope's existing topology view.
- `lldpLocalSystemData` / `lldpLocPortTable`: what this device advertises about itself.
- `lldpRemTable`: neighbor information as heard from LLDP frames — `lldpRemChassisId`, `lldpRemPortId`, `lldpRemSysName`, `lldpRemSysDesc`, and `lldpRemManAddrTable` for the neighbor's management IP. Walking this table on every device is how SignalScope can build/validate topology edges without relying on manual entry, complementing (or replacing, where reliable) any CDP-equivalent vendor-proprietary discovery.

## ENTITY-MIB (RFC 6933, supersedes RFC 4133)

Rooted at `entityMIB`, `1.3.6.1.2.1.47`. `entPhysicalTable` describes the physical composition of a device — chassis, line cards/modules, ports, power supplies, fans — via `entPhysicalDescr`, `entPhysicalClass` (chassis/module/port/powerSupply/fan/…), `entPhysicalName`, `entPhysicalSerialNum`, `entPhysicalModelName`, `entPhysicalHardwareRev`/`entPhysicalFirmwareRev`/`entPhysicalSoftwareRev`. Directly relevant to SignalScope's existing `inventory_assets` table (serial/warranty/EoS tracking) — this is the standards-based source for populating serial number and model data automatically via SNMP instead of manual entry.

## RMON-MIB (RFC 2819)

Rooted at `rmon`, `1.3.6.1.2.1.16`. Four groups matter most for switch-level use (RMON1; full RMON2 with network/application-layer visibility is rare on access switches): `statistics` (`etherStatsTable`, per-interface packet/error/collision counters, an alternative/supplement to `ifTable` counters), `history` (`historyControlTable`/`etherHistoryTable`, device-side rolling sample storage), `alarm` (`alarmTable`, device-side threshold monitoring — the device itself watches an OID and fires when it crosses a threshold), `event` (`eventTable`, what happens when an alarm fires — log entry and/or trap). The alarm/event groups are notable for the **agentic polling** discussion in [connectivity-methods.md](connectivity-methods.md): a switch that supports RMON alarms can offload some threshold-watching to itself and only trap SignalScope on threshold crossing, rather than SignalScope having to poll continuously to catch transient conditions.

## Practical takeaway for SignalScope

The above seven MIBs cover: device identification (SNMPv2-MIB), interface admin/oper state and counters (IF-MIB), MAC/spanning-tree/bridging (BRIDGE-MIB), VLAN membership (Q-BRIDGE-MIB), topology neighbor discovery (LLDP-MIB), physical inventory (ENTITY-MIB), and device-side threshold alarms (RMON-MIB). Every vendor's `mib-reference.md` should be read as "here's what this vendor adds or restricts on top of this common baseline," not as a from-scratch OID list.
