# Arista EOS — MIB Reference

Source: [Arista SNMP MIBs page](https://www.arista.com/en/support/product-documentation/arista-snmp-mibs) (41 Arista enterprise MIBs + standard IETF/IEEE MIB list, verified this session), [EOS SNMP user manual](https://www.arista.com/en/um-eos/eos-snmp).

## Read-only-by-default posture — read this first

Arista's own MIB documentation page states plainly: **"all MIB support is read-only unless otherwise noted."** This applies across both the standard (IETF/IEEE) MIBs and Arista's own enterprise MIBs. The narrow set of confirmed exceptions found this session:

- `IF-MIB::ifAdminStatus` — writable (bring an interface up/down via SNMP SET).
- `IF-MIB::ifAlias` — writable (interface description, maps to CLI `description`).

No other standard-MIB write path (VLAN membership, spanning-tree state, port-security, config-save trigger) was confirmed as writable in Arista's documentation this session. This is a significant cross-vendor data point: **Arista leans read-only-by-default for SNMP**, in contrast to vendors that expose broader vendor-MIB write surfaces for VLAN/config-save automation. SignalScope should treat any Arista SNMP SET beyond `ifAdminStatus`/`ifAlias` as unconfirmed and default to CLI-only for GUI actions — see [gui-cli-snmp-mapping.md](gui-cli-snmp-mapping.md), which is mostly CLI-only rows as a direct consequence.

Note the practical gap this leaves vs. the agent config surface: `snmp-server community ... rw` and `snmp-server group ... write <view>` let an operator *grant* write access, but the number of objects actually implementing SET behind that grant is small. A `rw` community configured on an Arista switch is not evidence of broad SNMP-driven configurability — it should not be treated as such by SignalScope's capability-detection logic.

## Standard MIBs — EOS support

| MIB | EOS support | Notes |
|---|---|---|
| SNMPv2-MIB | Read-only | `system` group (`sysDescr`, `sysObjectID`, `sysUpTime`, etc.) — standard baseline, use for device identification/discovery per [standard-mibs.md](../../00-architecture/standard-mibs.md). |
| IF-MIB (RFC 2863) | Read-only, with confirmed exceptions | `ifAdminStatus` and `ifAlias` are writable (see above); everything else (`ifOperStatus`, counters, `ifTable`/`ifXTable` generally) read-only. |
| BRIDGE-MIB (RFC 4188) | Read-only | Listed among supported standard MIBs; no write exception noted. |
| Q-BRIDGE-MIB (RFC 4363) | Read-only | Listed among supported standard MIBs; **no confirmed SNMP write path for VLAN membership/PVID** (`dot1qVlanStaticRowStatus`, `dot1qPvid`) was found — treat VLAN config as CLI-only on Arista pending further verification. |
| LLDP-MIB (IEEE 802.1AB) | Read-only | Topology/neighbor discovery use case unaffected by read-only posture — SignalScope's LLDP-based topology discovery works identically to other vendors. |
| ENTITY-MIB (RFC 6933) + ARISTA-ENTITY-SENSOR-MIB | Read-only | Physical inventory (chassis/module/serial) and Arista's own entity-sensor extension for environmental (temperature/fan/power) readings. |
| RMON-MIB (RFC 2819) + RMON2-MIB | Read-only | Both RMON1 and RMON2 listed as supported. |

Arista's page lists 44 standard IETF/IEEE MIBs total (broader than SignalScope's cross-vendor baseline seven) — the above table covers only the seven baseline MIBs from [standard-mibs.md](../../00-architecture/standard-mibs.md); the remainder are out of scope for this document unless a specific SignalScope feature needs them.

## Arista enterprise MIBs (arc `1.3.6.1.4.1.30065`)

Confirmed from `ARISTA-SMI-MIB.txt` (fetched directly this session):

| Node | OID |
|---|---|
| `arista` | `1.3.6.1.4.1.30065` |
| `aristaProducts` | `1.3.6.1.4.1.30065.1` (root for `sysObjectID` values — i.e. this is how SignalScope's SNMPv2-MIB-based vendor auto-detection identifies "this is an Arista switch" and potentially which model) |
| `aristaModules` | `1.3.6.1.4.1.30065.2` (MODULE-IDENTITY values) |
| `aristaMibs` | `1.3.6.1.4.1.30065.3` (parent arc for all the individual `ARISTA-*-MIB` modules below) |
| `aristaExperiment` | `1.3.6.1.4.1.30065.4` |
| `aristaInternalUse` | `1.3.6.1.4.1.30065.5` |

Individual enterprise MIBs relevant to switch management, listed on Arista's MIB download page (module-level names confirmed; full numeric leaf OIDs **not** independently verified this session — treat as "confirm against the actual MIB file before hard-coding an OID" per the standard-mibs.md convention):

| MIB | Relevance to SignalScope |
|---|---|
| `ARISTA-SMI-MIB` | Root SMI definitions (enterprise arc), confirmed above. |
| `ARISTA-PRODUCTS-MIB` | `sysObjectID` value assignments per Arista model — useful for precise model detection beyond generic vendor detection. |
| `ARISTA-IF-MIB` | Arista-specific interface extensions beyond standard IF-MIB/IF-XMIB. |
| `ARISTA-BRIDGE-EXT-MIB` | Bridging extensions beyond standard BRIDGE-MIB. |
| `ARISTA-SW-IP-FORWARDING-MIB` (IPv4 only, per Arista's own listing) | IP forwarding table state. |
| `ARISTA-CONFIG-COPY-MIB` / `ARISTA-CONFIG-MAN-MIB` | Named similarly to Cisco's `CISCO-CONFIG-COPY-MIB` (referenced in [connectivity-methods.md](../../00-architecture/connectivity-methods.md) as a config-drift/export mechanism) — **potentially relevant to SignalScope's config-drift polling tier**, but write/trigger support was not confirmed this session; given the platform-wide read-only-unless-noted posture, do not assume this MIB provides an SNMP-triggerable config export/save without independent confirmation. |
| `ARISTA-ENTITY-SENSOR-MIB` | Environmental sensor readings (temp/fan/power), complements standard ENTITY-MIB. |
| `ARISTA-VXLAN-MIB`, `ARISTA-VRF-MIB`, `ARISTA-BGP4V2-MIB`, `ARISTA-ACL-MIB`, `ARISTA-QOS-MIB` | Feature-specific MIBs (overlay networking, routing, ACLs, QoS) — out of scope for SignalScope's current Layer-2 switch-management focus, noted for completeness. |

Full list (41 enterprise MIBs) is available at the [Arista SNMP MIBs page](https://www.arista.com/en/support/product-documentation/arista-snmp-mibs); not all are reproduced here since most are outside SignalScope's current switch-management scope.

## `snmp-server view` / `show snmp view` — MIB-view access control

A distinctive EOS feature worth documenting explicitly: EOS supports **named MIB views** that restrict which objects/subtrees a given community string or SNMPv3 user/group can see or modify, independent of the object's own read-only/read-write nature:

```
snmp-server view <view-name> <mib-family> {include|exclude}
show snmp view
```

Confirmed example from Arista's documentation:

```
switch(config)# snmp-server view sys-view system include
switch(config)# snmp-server view sys-view system.2 exclude
switch(config)# show snmp view
sys-view system - included
sys-view system.2 - excluded
```

Views are then referenced by name in `snmp-server community <string> view <view-name> [ro|rw]` or `snmp-server group ... read <view-name> write <view-name>` to scope what a given credential can reach. This is an access-control layer, not a mechanism that makes additional objects writable — a `write` view naming an object that isn't implemented as SET-able in the underlying agent still won't accept a SET. SignalScope should treat `snmp-server view` purely as a **visibility/access-control fact to surface in the device's SNMP config summary** (e.g. "this community can only see the `system` subtree"), not as evidence of expanded write capability.
