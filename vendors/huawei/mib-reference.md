# Huawei VRP — MIB Reference

Covers standard (RFC/IEEE) MIB support confirmed for Huawei VRP S-series switches, the vendor enterprise MIBs relevant to switch management, and VRP's distinctive MIB-view access-control model. Baseline standard-MIB object definitions (`IF-MIB`, `BRIDGE-MIB`, `Q-BRIDGE-MIB`, `LLDP-MIB`, `ENTITY-MIB`, `RMON-MIB`) are documented once in [../../00-architecture/standard-mibs.md](../../00-architecture/standard-mibs.md) and vendored in [../../mibs/](../../mibs/) — this file only notes what Huawei confirms/restricts on top of that baseline.

## Standard MIB support

| MIB | Huawei S-series support | Notes |
|---|---|---|
| SNMPv2-MIB | Confirmed | Huawei publishes a dedicated support page for it per device family, e.g. [SNMPv2-MIB — S-series MIB Reference](https://support.huawei.com/enterprise/en/doc/EDOC1100212421/7c6ad5b4/snmpv2-mib), implying full `system` group support (`sysDescr`, `sysObjectID`, `sysUpTime`, `sysContact`, `sysName`, `sysLocation`, `sysServices`). `sysObjectID` for Huawei devices resolves under the Huawei enterprise arc `1.3.6.1.4.1.2011`. |
| IF-MIB | Supported (baseline for `display interface`/`ifAdminStatus`/`ifOperStatus` parity) | Not independently re-verified via a dedicated Huawei doc page in this pass — treat as high-confidence given VRP's interface model maps directly onto `ifTable`/`ifXTable`, but confirm exact counter coverage (`ifHCInOctets` etc.) against the specific device's MIB reference before relying on it. |
| BRIDGE-MIB | Supported for classic (802.1D) bridging/STP baseline | VRP defaults to MSTP; expect Huawei's own STP-extension objects (not confirmed by name in this pass) to carry MSTP-specific detail beyond what `dot1dStp` exposes. |
| Q-BRIDGE-MIB | Supported for VLAN read-back; SNMP-driven VLAN creation/membership write support **not independently confirmed** in this pass | Given Huawei also exposes `HUAWEI-L2VLAN-MIB` (below) for VLAN management, the vendor MIB may be the more complete write path — verify against `dot1qVlanStaticRowStatus` SET behavior on real hardware/a MIB browser before assuming Q-BRIDGE-MIB write works. |
| LLDP-MIB | Supported (VRP switches implement 802.1AB LLDP) | Standard IEEE MIB, used for SignalScope topology discovery same as other vendors. |
| ENTITY-MIB | Supported | Standard chassis/module/port inventory MIB; expected to back Huawei's serial/model/hardware-rev read-back the same as other vendors. |
| RMON-MIB | Supported | Standard RMON1 groups (`statistics`/`history`/`alarm`/`event`) — VRP switches commonly support RMON alarm/event groups for device-side threshold monitoring per the architecture doc's agentic-polling discussion. |

## Huawei enterprise MIBs (switch management)

All rooted under the Huawei enterprise PEN, `1.3.6.1.4.1.2011` (`enterprises.huawei`).

| MIB | Root OID | Purpose | Source |
|---|---|---|---|
| `HUAWEI-PORT-MIB` | `1.3.6.1.4.1.2011.5.25.157` (`...huaweiMgmt(5).hwDatacomm(25).hwPortMib(157)`) | Extended interface/port attributes beyond `IF-MIB` — transmission rate, duplex mode, auto-negotiation mode, up/down debounce timing. Confirmed to exist and documented per S-series MIB reference. | [HUAWEI-PORT-MIB — S1720/S2700/S5700/S6720 MIB Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000178181/69628d40/huawei-port-mib) |
| `HUAWEI-L2VLAN-MIB` | `1.3.6.1.4.1.2011.5.25.42.3` (`...hwDatacomm(25).hwL2Mgmt(42).hwL2Vlan(3)`) | Layer-2 VLAN management — basic VLAN attributes and the VLAN-to-port relationship (Huawei's own VLAN table, ~440 objects per the Observium MIB browser summary, spanning basic VLAN config, QinQ/stacking, voice VLAN, MAC/protocol-based VLAN assignment, port isolation, and per-VLAN statistics). Imports `IF-MIB`, `Q-BRIDGE-MIB`, `P-BRIDGE-MIB`. This is likely the more complete SNMP write surface for VLAN membership on Huawei devices vs. plain `Q-BRIDGE-MIB` — **not independently write-tested in this pass**. | [HUAWEI-L2VLAN-MIB — S1720/S2700/S5700/S6720 MIB Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000178181/62fde089/huawei-l2vlan-mib), [Observium MIB browser mirror](https://mibs.observium.org/mib/HUAWEI-L2VLAN-MIB/) |
| `HUAWEI-CONFIG-MAN-MIB` | `1.3.6.1.4.1.2011.5.6.10` (`...huaweiUtility(6).hwConfig(10)`) | Configuration-management MIB — the closest Huawei analogue to Cisco's `CISCO-CONFIG-COPY-MIB`, relevant to the config-drift-detection tier in [connectivity-methods.md](../../00-architecture/connectivity-methods.md) (triggering a config export/backup via SNMP rather than SSH `display current-configuration`). **Object-level detail (specific config-copy/backup trigger OIDs) not confirmed in this pass** — only the root OID and general purpose were recoverable from search; consult the linked MIB reference page directly (JS-rendered, requires a browser or a MIB-aware fetch) or a MIB browser walk against a live device before relying on specific table/scalar names. | [HUAWEI-CONFIG-MAN-MIB — S1720/S2700/S5700/S6720 MIB Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000178181/434f229f/huawei-config-man-mib) |

Other Huawei enterprise MIBs exist (`HUAWEI-DEVICE-MIB`, `HUAWEI-ACL-MIB`, `HUAWEI-XQoS-MIB`, `HUAWEI-STACK-MIB`, `HUAWEI-L2MAM-MIB`, etc. — see the [LibreNMS huawei mibs directory listing](https://github.com/librenms/librenms/tree/master/mibs/huawei) for a fuller inventory) but were not individually researched here as out of scope for SignalScope's port/VLAN/STP/LACP/SNMP-config feature set — revisit if a future feature needs QoS, stacking, or ACL SNMP control.

## `snmp-agent mib-view` — VRP's MIB access-control model

This is VRP's distinctive feature relative to most other vendors in this documentation set, and is the mechanism SignalScope must model explicitly for Huawei devices rather than treating "community has write access" as sufficient.

**Model**: VRP implements a named, composable MIB-view mechanism (conceptually a simplified VACM — View-based Access Control Model, the same idea RFC 3415 formalizes for SNMPv3, but here applied uniformly across v1/v2c/v3 access control in the VRP CLI) sitting *between* a community-string/SNMPv3-group and the actual MIB tree:

1. **Define a view**: `snmp-agent mib-view { excluded | included } <view-name> <oid-tree>` — a view is built incrementally by adding `included`/`excluded` OID-subtree rules under a shared `view-name`. `oid-tree` may be given as a dotted OID (`1.3.6.1.2.1.2`) or an object name (`ifTable`, `system.7`).
2. **Bind the view to an identity**:
   - v1/v2c: `snmp-agent community { read | write } <community-name> mib-view <view-name>` — ties a community string to a specific view for either read or write access (separately — a community can be read-bound to one view and a different community write-bound to a narrower one).
   - v3: `snmp-agent group v3 <group-name> { authentication | noauth | privacy } read-view <view> write-view <view> notify-view <view>` — same idea per-group, with the added v3 security-level dimension (`authentication`/`noauth`/`privacy`) and a separate `notify-view` controlling which objects can appear in traps sent to that group's users.
3. **Default view**: `ViewDefault` exists out of the box, covers the standard MIB-2 tree, and is immutable (cannot be edited or deleted) — a safe fallback but not a substitute for scoping down write access.
4. **Read-back**: `display snmp-agent mib-view [ <view-name> ]` shows configured views and their include/exclude rules.

**Why this matters for SignalScope**: unlike vendors where SNMP write access is a flat read/write toggle on the community, a Huawei SET can fail for two independent reasons — insufficient read/write grant on the community/group, *or* the target OID falling outside the bound view — and the GUI's "why did this SNMP action fail" diagnostic needs to check both, ideally by having the device-onboarding flow read back `display snmp-agent mib-view` and `display snmp-agent community`/`display snmp-agent group` up front so SignalScope knows the actual writable OID subtree before ever attempting a SET, rather than discovering the restriction via a failed SET at action time.

Sources: [snmp-agent mib-view — Command Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000128405/9c1bc6fd/snmp-agent-mib-view), [display snmp-agent mib-view — Command Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000128405/c74a608d/display-snmp-agent-mib-view), [snmp-agent community — Command Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000128405/37b6f32c/snmp-agent-community), [SNMP Configuration Commands — CloudEngine S3700/S5700/S6700 Command Reference](https://support.huawei.cn/enterprise/en/doc/EDOC1100368578/30eb7700/snmp-configuration-commands).

## MIB file availability

No raw `.mib` files are vendored in this repo for Huawei enterprise MIBs — see [overview.md](overview.md#mib-licensingredistribution-note) and [mibs/README.md](mibs/README.md) for the licensing rationale. Use the linked `support.huawei.com` pages (human-readable object docs) or the third-party mirrors noted there (LibreNMS, Observium) for OID-level lookups pending a clearer redistribution basis.
