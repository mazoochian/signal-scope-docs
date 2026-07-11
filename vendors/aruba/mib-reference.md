# ArubaOS-CX MIB Reference

## Provenance note

HPE does not publish standalone `.mib`/`.my` files for direct download alongside its PDF/HTML SNMP guides in an easily scriptable way (the guides are prose/HTML documentation of the MIB objects, not the raw MIB source). The OIDs and access-level (`MAX-ACCESS`) facts below were confirmed this session by reading the **raw ARUBAWIRED-*.mib source text**, obtained via a third-party open-source mirror (LibreNMS's vendored MIB set, `github.com/librenms/librenms/tree/master/mibs/arubaos-cx`) rather than directly from HPE. Every `ARUBAWIRED-*.mib` file carries an explicit HPE proprietary-and-confidential copyright header (verbatim, from source):

> *(c) Copyright Hewlett Packard Enterprise Development LP. All Rights Reserved. The contents of this software are proprietary and confidential to the Hewlett-Packard Development Company, L.P. No part of this program may be photocopied, reproduced, or translated into another programming language without prior written consent of the Hewlett-Packard Development Company, L.P.*

This means the OIDs/object names cited below are believed accurate (read directly from vendor-authored source text) but the **files themselves are not suitable for vendoring into this repo** — see the licensing note in [overview.md](overview.md). Treat every OID below as "confirmed against vendor MIB source seen this session," not as independently cross-checked against a live device or the official PDF guide's prose (which was not directly fetchable this session — see overview.md's note on 403s).

## Standard (RFC/IEEE) MIBs supported

ArubaOS-CX implements the shared baseline documented in [../../00-architecture/standard-mibs.md](../../00-architecture/standard-mibs.md): **SNMPv2-MIB, IF-MIB, BRIDGE-MIB, Q-BRIDGE-MIB, LLDP-MIB, ENTITY-MIB, RMON-MIB**. This is standard for a modern enterprise switch NOS and consistent with what AOS-CX's own SNMP/MIB guides describe (system/interface/bridge/VLAN/LLDP/entity/RMON support), though exact per-object support (e.g. full RMON2 vs. RMON1-only groups) was not independently re-verified against the official guide text this session — treat that granularity as "very likely, not directly confirmed."

`sysObjectID` (`SNMPv2-MIB::sysObjectID`, `1.3.6.1.2.1.1.2`) for AOS-CX devices resolves under HPE's enterprise arc — confirmed sysObjectID pattern: **`1.3.6.1.4.1.47196.4.1.1.1`** (per search-summary reference to the AOS-CX SNMP guide; the enterprise-number component `47196` itself is independently confirmed below from MIB source). This is the fast vendor/model auto-detect path per the standard-mibs.md "poll first" guidance.

## HPE/Aruba enterprise MIBs — `ARUBAWIRED-*` family

**Enterprise PEN (Private Enterprise Number): `47196`** (HPE), confirmed directly from `ARUBAWIRED-NETWORKING-OID.mib` source: `hpe OBJECT IDENTIFIER ::= { enterprises 47196 }`.

**OID root for all AOS-CX wired-switch enterprise MIBs**, confirmed from source:

```
iso.org.dod.internet.private.enterprises.hpe(47196).hpeNetworking(4).wiredNetworkingDevices(1).arubaOS-CX(1).wndFeatures(3)
= 1.3.6.1.4.1.47196.4.1.1.3
```

(Note: `hpeNetworking` itself is `{ hpe 4 }`, confirmed from source — i.e. the full chain is `enterprises.47196` → `hpeNetworking = .4` → `wiredNetworkingDevices = .4.1` → `arubaOS-CX = .4.1.1` → `wndFeatures = .4.1.1.3`.)

Every `ARUBAWIRED-*-MIB` module attaches under `wndFeatures` at a fixed sub-arc, confirmed directly from `ARUBAWIRED-NETWORKING-OID.mib`:

| Module | OID (`wndFeatures.N` → full path `1.3.6.1.4.1.47196.4.1.1.3.N`) | Relevance to SignalScope |
|---|---|---|
| `ARUBAWIRED-LOOPPROTECT-MIB` | `.1` | Loop-protection feature state. |
| `ARUBAWIRED-MCLAG-MIB` | `.2` | Multi-chassis LAG state. |
| `ARUBAWIRED-MGMD-SNOOPING-MIB` | `.3` | IGMP/MLD snooping (multicast). |
| `ARUBAWIRED-MGMD-RMON-TRAP-MIB` | `.4` | Multicast-related RMON traps. |
| `ARUBAWIRED-RPVST-MIB` | `.5` | Rapid-PVST+ spanning-tree mode state. |
| `ARUBAWIRED-MVRP-MIB` | `.6` | MVRP (dynamic VLAN registration). |
| `ARUBAWIRED-VSX-MIB` | `.7` | VSX (Aruba's active-active MLAG-like chassis-redundancy feature) config-sync state. |
| `ARUBAWIRED-POE-MIB` | `.8` | Power-over-Ethernet per-port state/budget. |
| `ARUBAWIRED-LLDP-MIB` | `.9` | AOS-CX-specific LLDP extensions beyond standard `LLDP-MIB`. |
| `ARUBAWIRED-VSF-MIB` / `ARUBAWIRED-VSFv2-MIB` | `.10` / `.15` | VSF (Virtual Switching Framework, stacking) state — two module versions exist. |
| `ARUBAWIRED-CHASSIS-MIB` | `.11` | Chassis/fan/PSU/LED inventory — complements standard `ENTITY-MIB`. |
| `ARUBAWIRED-CIPT-MIB` | `.12` | Not investigated in depth this session — name suggests CIP/telephony-related; confirm before relying on it. |
| `ARUBAWIRED-MSTP-MIB` | `.13` | **Read-write** MSTP objects confirmed present (see below) — relevant to spanning-tree GUI actions beyond what `BRIDGE-MIB` covers. |
| `ARUBAWIRED-MDNS-MIB` | `.14` | mDNS relay/gateway feature. |
| `ARUBAWIRED-AAA-MIB` | `.16` | AAA (authentication) config/state — not investigated in depth. |
| `ARUBAWIRED-PORT-ACCESS-MIB` | `.17` | Port-access role/policy (802.1X-adjacent) state — not investigated in depth. |
| `ARUBAWIRED-PORTVLAN-MIB` | `.18` | **Read-only** port↔VLAN membership/mode (see below) — the read-back source for VLAN GUI sync, but **not** a write path. |
| `ARUBAWIRED-MACNOTIFY-MIB` | `.19` | MAC address change notifications. |
| `ARUBAWIRED-CONFIG-MIB` | `.20` | **Read-write** config-copy/checkpoint workflow (see below) — directly relevant to the "save/checkpoint" GUI action. |
| `ARUBAWIRED-PORTSECURITY-MIB` | `.21` | **Fully read-write** port-security config (see below) — directly relevant to the port-security GUI action. |
| `ARUBAWIRED-SYSTEMINFO-MIB` | `.22` | System-level info beyond `SNMPv2-MIB::system`. |
| `ARUBAWIRED-PROVIDER-BRIDGE-MIB` | `.23` | 802.1ad provider-bridging (QinQ) state. |
| `ARUBAWIRED-INTERFACE-MIB` | `.24` | **Read-write** interface autoneg/duplex/speed-list objects (see below) — beyond what standard `IF-MIB` exposes for physical-layer config. |
| `ARUBAWIRED-DIST-SERVICES-MIB` | `.25` | Not investigated in depth. |
| `ARUBAWIRED-SWITCH-IMAGE-MIB` | `.26` | Firmware image management. |
| `ARUBAWIRED-PM-MIB` | `.27` | Not investigated in depth (possibly "power management" or "port monitoring" — name not expanded in source comments seen). |

Additional modules exist in the mirrored MIB set but were not mapped to a `wndFeatures` sub-OID or examined this session: `ARUBAWIRED-FAN-MIB`, `ARUBAWIRED-FANTRAY-MIB`, `ARUBAWIRED-LED-LOCATOR-MIB`, `ARUBAWIRED-MODULE-MIB`, `ARUBAWIRED-POWER-STAT-MIB`, `ARUBAWIRED-POWERSUPPLY-MIB`, `ARUBAWIRED-TEMPSENSOR-MIB`, `ARUBAWIRED-NETWORKING-OID` (the root-OID definition module itself, not a feature MIB).

### `ARUBAWIRED-INTERFACE-MIB` (`.24`) — confirmed read-write objects

Table `arubaWiredInterfaceTable`, indexed by `arubaWiredInterfaceIndex` (an `InterfaceIndex` = same value as `IF-MIB::ifIndex`):
- `arubaWiredInterfaceAutoneg` (`INTEGER {on(1), off(2)}`, **read-write**)
- `arubaWiredInterfaceDuplex` (`INTEGER {full(1), half(2)}`, **read-write**)
- `arubaWiredInterfaceSpeeds` (`BITS`, one bit per speed from 10M through 800G, **read-write**) — the desired operating-speed set to offer/use.

### `ARUBAWIRED-PORTVLAN-MIB` (`.18`) — confirmed read-only

Table `arubaWiredPortVlanMemberTable`, indexed by `arubaWiredPortVlanMemberIndex` (`InterfaceIndex`):
- `arubaWiredPortVlanMemberMode` (`INTEGER {trunk(1), access(2)}`, **read-only**)
- `arubaWiredPortVlanMemberVid` (`VidList`, a 512-octet bitmap textual convention covering VLAN IDs 1-4096, **read-only**)

Both objects are explicitly `MAX-ACCESS read-only` in source — **this vendor MIB cannot be SNMP-SET to change VLAN membership.** Any SNMP-driven VLAN write on AOS-CX would have to go through standard `Q-BRIDGE-MIB::dot1qVlanStaticTable`/`dot1qPvid`, whose write support on this platform was not independently confirmed this session.

### `ARUBAWIRED-PORTSECURITY-MIB` (`.21`) — confirmed read-write

Global: `arubaWiredPortSecurityGlobalEnable` (`TruthValue`, **read-write**).

Per-port table `arubaWiredPortSecurityPortTable`, indexed by `arubaWiredifIndex` (read-only index):
- `arubaWiredPortSecurityEnable` (`TruthValue`, **read-write**)
- `arubaWiredClientLimit` (`Unsigned32 (1..64)`, **read-write**)
- `arubaWiredViolationAction` (`INTEGER {notify(1), shutdown(2)}`, **read-write**)
- `arubaWiredRecoveryTimer` (`Unsigned32 (10..600)`, **read-write**)
- `arubaWiredShutdownRecovery` (`TruthValue`, **read-write**)
- `arubaWiredStickyEnable` (`TruthValue`, **read-write**)
- Read-only companions: `arubaWiredCurrentSecureMacAddrCount`, `arubaWiredClientViolationStatus`, `arubaWiredClientViolationReason`, `arubaWiredClientLimitViolationCount`, `arubaWiredStickyClientMoveViolationCount` (all counters/status, read-only by design).

Plus `arubaWiredPortSecurityClientTable` (learned/configured client MAC entries per port) — read structure only inspected, full column list not exhaustively captured this session.

### `ARUBAWIRED-CONFIG-MIB` (`.20`) — confirmed read-write, config-copy/checkpoint workflow

This is AOS-CX's SNMP analogue of Cisco's `CISCO-CONFIG-COPY-MIB`. Core table: `arubaWiredConfigurationCopyTable`, created/driven via `RowStatus` (create a row with `createAndGo`, poll status, then delete) — confirmed structurally from source (`RowStatus` imported from `SNMPv2-TC`, standard row-creation SNMP pattern per `standard-mibs.md`'s Q-BRIDGE-MIB note on the same pattern).

Key row objects (indexed by `arubaWiredConfigurationCopyIndex`, a client-chosen `Unsigned32` serial number):
- `arubaWiredConfigurationCopySourceFileType` / `arubaWiredConfigurationCopyDestFileType` — both a `ConfigurationFileType`: `externalFile(1)`, `startupConfiguration(2)`, `runningConfiguration(3)`, `checkpoint(4)`. This is the SET that drives "save to startup," "load a checkpoint into running," "export running-config off-box," etc. — all as one generalized copy operation, source→dest.
- `arubaWiredConfigurationCheckpointName` (`DisplayString`) — the checkpoint name when either side is `checkpoint(4)`.
- `arubaWiredConfigurationCopyProtocol` — `scp(1)`/`sftp(2)`/`tftp(3)`, used when a side is `externalFile`.
- `arubaWiredConfigurationCopyFileFormat` — `cli(1)`/`json(2)`, for `externalFile` destinations.
- `arubaWiredConfigurationCopyServerAddressType`/`arubaWiredConfigurationCopyServerAddress` (`InetAddressType`/`InetAddress`) and `arubaWiredConfigurationCopyUserName`/`arubaWiredConfigurationCopyUserPassword` — for the external-file transfer case.
- Status objects (not individually enumerated with full names this session, confirmed present structurally): a `ConfigurationCopyState` (`waiting(1)/running(2)/successful(3)/failed(4)`) and a `ConfigurationCopyFailureCause` enum (`authenticationFailed`, `badFilename`, `busy`, `invalidConfiguration`, `invalidURL`, `systemNotReady`, `timeout`, `unknown`) for polling the outcome of an in-flight copy request.
- Also present (confirmed structurally, not enumerated in full): a scalar `read-write` object plus a `sysUpTime`-stamped "last running-config change" timestamp object, and a notification fired when the running configuration changes (`ConfigurationEventMedium` textual convention tags the change source as `checkpoint(1)/cli(2)/internal(3)/rest(4)/snmp(5)/ztp(6)` — directly useful for SignalScope's CLI↔GUI reconciliation, since a config-change notification can indicate whether the change came from the CLI session SignalScope itself opened vs. an out-of-band source).

**SignalScope relevance**: this MIB gives a fully-general SNMP path for "trigger a save," "trigger a checkpoint," and "restore a checkpoint" — a genuine SNMP-SET alternative to the CLI `copy running-config ...`/`checkpoint` commands in [cli-reference.md](cli-reference.md), for the case where a device is SNMP-only reachable (see the unification doc's SNMP-fallback rule).

### `ARUBAWIRED-MSTP-MIB` (`.13`) — read-write objects present

Confirmed structurally: several `read-write` scalar/table objects exist (5 distinct read-write objects observed in source) alongside a larger set of `accessible-for-notify` trap-only objects (MSTP topology-change notifications) and `not-accessible` index objects. Full object names/OIDs were not individually enumerated this session — before using this MIB for a GUI spanning-tree write path, re-fetch and read the full module (`https://raw.githubusercontent.com/librenms/librenms/master/mibs/arubaos-cx/ARUBAWIRED-MSTP-MIB` at time of writing) to get exact object names, or use the standard `BRIDGE-MIB`/`Q-BRIDGE-MIB` RSTP/MSTP-adjacent objects noted in `standard-mibs.md` where sufficient.

## Objects not confirmed this session

- Exact numeric OIDs for `ARUBAWIRED-CHASSIS-MIB`, `ARUBAWIRED-POE-MIB`, `ARUBAWIRED-VSX-MIB`, `ARUBAWIRED-VSF-MIB`/`VSFv2` internals — module locations (`wndFeatures.N`) are confirmed, but individual object tables inside were not read this session.
- Whether standard `Q-BRIDGE-MIB::dot1qVlanStaticTable`/`dot1qPvid` are writable on AOS-CX (the vendor-native VLAN MIB is read-only, as shown above, so this is the only remaining candidate SNMP-SET VLAN path and its write status is unresolved).
- The precise HPE PDF/HTML SNMP-MIB guide's own prose description of these objects (guide pages 403'd to automated fetch this session) — the facts above come from vendor MIB source text via a third-party mirror, not from reading HPE's guide narrative directly. Cross-check against the official guide (links in [overview.md](overview.md)) before treating any of the above as final for implementation.
