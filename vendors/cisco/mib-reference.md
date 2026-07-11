# Cisco IOS / IOS-XE — MIB Reference

This document covers what Cisco adds or restricts on top of the cross-vendor standard MIBs in [`standard-mibs.md`](../../00-architecture/standard-mibs.md), plus Cisco's own enterprise MIBs relevant to switch management. Per that doc's baseline: SNMPv2-MIB/IF-MIB/BRIDGE-MIB/Q-BRIDGE-MIB/LLDP-MIB/ENTITY-MIB/RMON-MIB are assumed present; this file only calls out Cisco-specific behavior on top of them.

## Standard MIB support and Cisco-specific quirks

| Standard MIB | Cisco IOS-XE support | Notes |
|---|---|---|
| SNMPv2-MIB | Full | `sysObjectID` under a Cisco Catalyst will resolve to an OID under `1.3.6.1.4.1.9.1.<model-specific>` (Cisco's enterprise arc, PEN 9) — useful for SignalScope's vendor auto-detect. |
| IF-MIB | Full, including `ifXTable`/64-bit counters | `ifName` matches the short CLI interface form (`Gi1/0/1`), `ifDescr` the longer form (`GigabitEthernet1/0/1`) — both present, useful for correlating SNMP index to CLI-typed names per the standard-MIBs doc's note. |
| BRIDGE-MIB | Present, but classic 802.1D `dot1dStp` group reflects **PVST/Rapid-PVST per-VLAN state only for VLAN 1** (or whichever instance the agent context defaults to) unless SNMP **context-based** access (`vlan-<id>@community` community-string indexing, or SNMPv3 contexts) is used to reach other VLANs' STP instances. This is a well-known Cisco-specific gotcha: a naive walk of `dot1dStpPortTable` without a VLAN-scoped community/context only shows one VLAN's tree. |
| Q-BRIDGE-MIB | Present; `dot1qVlanStaticTable`/`dot1qPvid` readable and, per Cisco's own documented workflow, writable (VLAN create/destroy via `dot1qVlanStaticRowStatus`) — see Cisco's SNMP VLAN how-to cited in `overview.md`. In practice Cisco's own `CISCO-VLAN-MEMBERSHIP-MIB` (below) is the more commonly used/documented path for per-port VLAN assignment specifically. |
| LLDP-MIB | Supported, but **not enabled by default** — requires `lldp run` in global config first (LLDP is off out of the box on Cisco switches, unlike CDP). Once enabled, standard `lldpRemTable` etc. populate normally. |
| ENTITY-MIB | Full — `entPhysicalTable` covers chassis/supervisor/line-card/PSU/fan on modular platforms (9400/9500/9600) and the single-chassis case on fixed platforms (9200/9300). Reliable source for serial/model/firmware per SignalScope's inventory tracking. |
| RMON-MIB | `statistics`/`history` groups present; `alarm`/`event` groups present but require explicit configuration (`rmon alarm`/`rmon event` CLI commands, or SNMP SET into `alarmTable`/`eventTable`) — nothing is monitored by RMON out of the box. |

### CDP vs LLDP

Cisco switches ship with **CDP (Cisco Discovery Protocol) enabled by default**, which predates and (on an all-Cisco topology) is generally considered more reliable/richer than LLDP for Cisco-to-Cisco neighbor discovery — it carries Cisco-specific fields (native VLAN, duplex mismatches, IOS version, platform capabilities) that LLDP's standard TLVs don't. LLDP is supported and standards-based (interoperates with non-Cisco neighbors) but **must be explicitly enabled** (`lldp run`) and is not on by default.

- **`CISCO-CDP-MIB`** (root `1.3.6.1.4.1.9.9.23`, confirmed via search): `cdpGlobalRun` (`.1.3.1`, read-write `TruthValue`, whether CDP is running at all — mirrors CLI `cdp run`/`no cdp run`) and `cdpCacheTable` (neighbor cache, keyed by local `ifIndex` + a per-neighbor device index) — the CDP analogue of `LLDP-MIB::lldpRemTable`.
- SignalScope's topology discovery should walk **both** where available on Cisco gear: LLDP for cross-vendor edges, CDP for higher-fidelity Cisco-to-Cisco edges, and reconcile/prefer CDP data when both report the same edge on an all-Cisco segment.

## Vendor (enterprise) MIBs

All under Cisco's enterprise arc `1.3.6.1.4.1.9` (PEN 9). Per-MIB OID confirmation below is from Cisco's own SNMP how-to docs where cited, or cross-checked against multiple independent MIB-browser mirrors (`oidref.com`, `mibs.observium.org`) where a direct Cisco doc citation wasn't available — flagged accordingly.

### CISCO-VLAN-MEMBERSHIP-MIB

- Root: `ciscoVlanMembershipMIB` = `1.3.6.1.4.1.9.9.68` (confirmed).
- Key objects (confirmed via search + local vendored copy in `mibs/CISCO-VLAN-MEMBERSHIP-MIB.my`): `vmMembershipEntry` = `.1.2.2.1`, `vmVlanType` = `.1.2.2.1.1`, **`vmVlan`** = `.1.2.2.1.2` (`INTEGER 0..4095`, `MAX-ACCESS read-write`) — the per-port access-VLAN (PVID) assignment object, keyed by `ifIndex`.
- **SNMP SET support**: Yes — this is Cisco's own documented mechanism for assigning a switchport's access VLAN via SNMP (see [How To Add, Modify, and Remove VLANs Using SNMP](https://www.cisco.com/c/en/us/support/docs/ip/simple-network-management-protocol-snmp/45080-vlans.html)). Realistic write path on modern IOS-XE, subject to the SNMP agent being configured read-write and (historically) VTP mode considerations for trunk/VLAN database changes.

### CISCO-VTP-MIB

- Root: `ciscoVtpMIB` = `1.3.6.1.4.1.9.9.46` (confirmed via search); `managementDomainName` at `.1.2.1.1.2` (`MAX-ACCESS read-create`, confirmed via search).
- Covers VTP domain name/mode/version and the VTP-synchronized VLAN database (`vtpVlanTable` at `.1.3.1.1`).
- **SNMP SET support**: Objects are defined `read-create`/`read-write` in the MIB, but Cisco's own guidance for VLAN management via SNMP centers on `CISCO-VLAN-MEMBERSHIP-MIB` for port assignment and the standard `Q-BRIDGE-MIB` static-VLAN table for VLAN existence — VTP-domain-level SNMP writes (changing VTP mode/domain via SNMP) are not something we found a documented Cisco-supported workflow for. Treat as **read-only in practice** for SignalScope purposes; VTP mode/domain changes should go through CLI (`vtp mode`, `vtp domain`). Not vendored locally (see `overview.md`).

### CISCO-STACK-MIB / CISCO-STACKWISE-MIB

- **`CISCO-STACK-MIB`**: root `1.3.6.1.4.1.9.5.1` (confirmed via search). This is a large, old (1995-origin) MIB built for CatOS-era chassis/module/port management on Catalyst 4000/5000/6000-class hardware; vendored locally (`mibs/CISCO-STACK-MIB.my`) for reference but **not the recommended primary source** for modern Catalyst 9000 fixed-stack management.
- **`CISCO-STACKWISE-MIB`**: root `1.3.6.1.4.1.9.9.500` (confirmed via search) — the modern MIB for Cisco's StackWise physical-stacking technology (stack member state, stack cabling, stack power). Confirmed to exist and be supported on stackable fixed switches (3750-X and successors); we did **not** obtain a confirmed sub-OID for specific objects (e.g. per-member role/state) this session — treat individual StackWise object OIDs as unconfirmed pending a MIB-file pull, rather than guessing them. Not vendored locally this session.
- **SNMP SET support**: Both MIBs are realistically **read-only** for SignalScope's purposes — stack membership/role changes are a CLI-driven operation (`switch renumber`, `switch priority`, physical cabling) on IOS-XE, not something Cisco documents as SNMP-settable.

### CISCO-CONFIG-COPY-MIB

- Root: `ciscoConfigCopyMIB` = `1.3.6.1.4.1.9.9.96` (confirmed via search and local vendored copy `mibs/CISCO-CONFIG-COPY-MIB.my`).
- `ccCopyTable`/`ccCopyEntry` = `.1.1.1`/`.1.1.1.1`, with `ccCopyProtocol` (`.1.1.1.1.2`, `MAX-ACCESS read-create`), `ccCopySourceFileType`/`ccCopyDestFileType` (`.1.1.1.1.3`/`.1.1.1.1.4`, both `read-create`), and `ccCopyEntryRowStatus` (RFC 2579 `RowStatus`, creates/triggers the copy operation).
- Confirmed from the MIB text itself (see local vendored copy): setting `ccCopySourceFileType = runningConfig(4)` and `ccCopyDestFileType = startupConfig(3)` and creating the row is the **documented SNMP equivalent of `write memory`/`copy running-config startup-config`** — the MIB's own description text explicitly notes the protocol object "may be ignored by the implementation" when the copy is local (running→startup), i.e. no TFTP server involved for that specific case. The same table also drives copy-to/from-network (TFTP/SCP-style export for backup, using `networkFile`/`ccCopyServerAddress`/`ccCopyFileName`) and copy-to/from-`iosFile` (local flash).
- **SNMP SET support**: **Yes, and this is the one enterprise-MIB write path Cisco explicitly documents end-to-end** (see [How To Copy Configurations To and From Cisco Devices Using SNMP](https://www.cisco.com/c/en/us/support/docs/ip/simple-network-management-protocol-snmp/15217-copy-configs-snmp.html)) — directly relevant to SignalScope's "save config" GUI action and to the config-drift-detection tier's SNMP-triggered backup path mentioned in `connectivity-methods.md`.

### CISCO-PROCESS-MIB / CISCO-MEMORY-POOL-MIB

- **`CISCO-PROCESS-MIB`**: root region confirmed via search — `cpmCPUTotal5minRev` = `1.3.6.1.4.1.9.9.109.1.1.1.1.8` (`INTEGER 0..100`, 5-minute average CPU busy %, the object Cisco's own CPU-collection how-to recommends over the older `cpmCPUTotal5min`, which was range-limited and could misreport near 100%). Not vendored locally this session (see `overview.md`).
- **`CISCO-MEMORY-POOL-MIB`**: root `ciscoMemoryPoolMIB` = `1.3.6.1.4.1.9.9.48` (confirmed via search). Exposes per-pool (Processor, I/O, etc.) used/free memory.
- Why these matter: **IF-MIB and the other standard MIBs have no CPU/memory objects at all** — this is exactly the "some vendors only expose CPU/memory via CLI, not SNMP" case flagged in `connectivity-methods.md`'s agentic-polling section, except Cisco is actually the good case here — it *does* expose both via SNMP, so SignalScope's passive-telemetry poller should prefer these over CLI-scraping `show processes cpu`/`show memory` for Cisco devices specifically.
- **SNMP SET support**: None — both are read-only telemetry MIBs by nature.

### CISCO-PORT-SECURITY-MIB

- Root: `ciscoPortSecurityMIB` = `1.3.6.1.4.1.9.9.315` (confirmed via search and local vendored copy `mibs/CISCO-PORT-SECURITY-MIB.my`).
- Key objects (confirmed from vendored MIB text): `cpsIfPortSecurityEnable` (`.1.2.1.1.1`, `TruthValue`, `MAX-ACCESS read-write`), `cpsIfMaxSecureMacAddr` (max secure MACs, `read-write`), `cpsIfViolationAction` (`read-write`), `cpsIfViolationCount` (`.1.2.1.1.9`, read-only counter).
- **SNMP SET support**: The MIB itself defines these as `read-write`, but **Cisco does not publish a documented SNMP-driven port-security configuration workflow** (unlike the VLAN-membership and config-copy cases above, which have Cisco how-to docs specifically walking through `snmpset` usage). SignalScope should treat port-security configuration as **CLI-only in practice** for Cisco devices — this is the concrete example flagged in `overview.md` of "MIB says read-write, but that's not the same as a supported write path"; a naive implementation that trusts `MAX-ACCESS` alone would incorrectly assume an SNMP fallback exists here.

## Summary table (feeds the cross-vendor SNMP-write-support matrix)

| MIB / object | Realistic SNMP SET support on IOS-XE |
|---|---|
| `IF-MIB::ifAdminStatus`, `ifAlias` | Yes |
| `CISCO-VLAN-MEMBERSHIP-MIB::vmVlan` | Yes (Cisco-documented) |
| `Q-BRIDGE-MIB` static VLAN table | Yes (Cisco-documented, less commonly used than the above) |
| `CISCO-CONFIG-COPY-MIB` (save/backup) | Yes (Cisco-documented end-to-end) |
| `CISCO-VTP-MIB` (domain/mode) | No — CLI only |
| `BRIDGE-MIB`/STP per-port cost/priority/portfast | No — CLI only |
| `CISCO-STACK-MIB` / `CISCO-STACKWISE-MIB` | No — CLI/physical only |
| LACP/EtherChannel membership | No — CLI only (no MIB path found) |
| `CISCO-PORT-SECURITY-MIB` | No in practice, despite `read-write` MIB definition — CLI only |
| `CISCO-PROCESS-MIB`, `CISCO-MEMORY-POOL-MIB` | N/A — read-only telemetry |
| `CISCO-CDP-MIB::cdpGlobalRun` | Technically `read-write` per MIB; no confirmed Cisco-documented SNMP-driven workflow found — treat as CLI-only (`cdp run`/`no cdp run`) pending further confirmation |
