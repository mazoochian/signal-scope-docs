# MikroTik RouterOS — MIB Reference

## Provenance note

Unlike most other vendors in this docs tree, MikroTik's enterprise MIB **is vendored directly** in this folder (`mibs/MIKROTIK-MIB.txt`) — see [`mibs/README.md`](mibs/README.md) for the licensing basis (a LibreNMS/GPLv3 mirror, since MikroTik's own download page carries no separate MIB redistribution grant). Every object/OID below was read directly from that file, not inferred from documentation prose — treat this as a first-hand source read, the reverse of the situation for Aruba/Huawei/Extreme where the vendor MIB text itself wasn't vendorable and OIDs came from documentation instead.

## Standard (RFC/IEEE) MIBs supported

RouterOS implements the shared baseline in [`../../00-architecture/standard-mibs.md`](../../00-architecture/standard-mibs.md): **SNMPv2-MIB, IF-MIB, BRIDGE-MIB, Q-BRIDGE-MIB, LLDP-MIB, RMON-MIB** are all present per MikroTik's own SNMP documentation (see [overview.md](overview.md)). `ENTITY-MIB` support was not independently confirmed this session — RouterOS's system/hardware-inventory objects are more commonly exposed via the vendor's own `mtxrSystem`/`mtxrHealth` groups below than via `entPhysicalTable`, so confirm `ENTITY-MIB` support specifically before relying on it for MikroTik inventory polling.

`sysObjectID` resolves under MikroTik's enterprise arc, PEN **14988**, confirmed directly from `MIKROTIK-MIB.txt`: `mikrotik OBJECT IDENTIFIER ::= { enterprises 14988 }`. The exact `sysObjectID` value for a given hardware model was not enumerated this session — use `print oid` on a live device (see [overview.md](overview.md)) to confirm rather than assuming a single value covers all RouterOS hardware.

## `MIKROTIK-MIB` structure (PEN 14988)

The whole enterprise MIB is one module (`mikrotikExperimentalModule`, confirmed still named "Experimental" in the module identity despite two decades of production use — not a signal of instability, just an unchanged module name). Every feature area attaches under `mtXRouterOs`, `1.3.6.1.4.1.14988.1.1`:

| Group | OID (`mtXRouterOs.N`) | Relevance to SignalScope |
|---|---|---|
| `mtxrWireless` | `.1` | Wireless-specific — out of scope for switch management. |
| `mtxrQueues` | `.2` | QoS queue (simple/tree) byte/packet counters — read-only. |
| `mtxrHealth` | `.3` | **Voltage, temperature, fan-speed, PSU-state sensors** — directly relevant to SignalScope's environmental/health monitoring. All 19 scalar objects confirmed `read-only` (see below). |
| `mtxrLicense` | `.4` | License level/features — not investigated in depth. |
| `mtxrHotspot` | `.5` | RouterOS Hotspot captive-portal feature — out of scope for switch management. |
| `mtxrDHCP` | `.6` | DHCP server lease table — relevant to broader network-visibility features, not core switch config. |
| `mtxrSystem` | `.7` | **System identity + two genuinely read-write operational objects** (see below) — the most SignalScope-relevant scalar group. |
| `mtxrScripts` | `.8` | Named user-scripts + a read-write "run" trigger per script (see below) — a distinctive SNMP-driven automation surface unique to this vendor. |
| `mtxrTraps` | `.9` | Trap-related config. |
| `mtxrNstremeDual` | `.10` | Legacy proprietary wireless protocol — irrelevant to switches. |
| `mtxrNeighbor` | `.11` | MikroTik's proprietary neighbor-discovery table (MNDP) — a supplement to standard `LLDP-MIB` for topology discovery, MikroTik-to-MikroTik only. |
| `mtxrGps` | `.12` | GPS module state — irrelevant to switches. |
| `mtxrWirelessModem` | `.13` | Cellular modem state — irrelevant to switches. |
| `mtxrInterfaceStats` | `.14` | **Extended per-interface RMON-style counters** (frame-size histogram, FCS/alignment/jabber/fragment error breakdown) — a superset of what `IF-MIB`/RMON-MIB's `etherStatsTable` provide standardly; confirmed all `read-only` in a table indexed by `mtxrInterfaceStatsIndex`. |
| `mtxrPOE` | `.15` | **Per-port PoE status/voltage/current/power table** — directly relevant to SignalScope's inventory/health views for PoE-capable switch models (e.g. CRS series with PoE-out ports). All objects confirmed `read-only`; `mtxrPOEStatus` is a rich enum (`disabled(1)`, `waitingForLoad(2)`, `poweredOn(3)`, `overload(4)`, `shortCircuit(5)`, `voltageTooLow(6)`, `currentTooLow(7)`, `powerReset(8)`, plus more not enumerated this session) — a good source for a PoE-fault GUI indicator. |
| `mtxrLTEModem` | `.16` | Irrelevant to switches. |
| `mtxrPartition` | `.17` | Storage-partition state (routers with NAND partitioning) — not typically relevant to switch-only hardware. |
| `mtxrScriptRun` | `.18` | Related to the script-run mechanism above. |
| `mtxrOptical` | `.19` | SFP/optical-transceiver diagnostics (DDM-style: temperature, voltage, bias current, Tx/Rx power) — relevant to SignalScope's interface/optics health views on switches with SFP ports. Not individually enumerated this session. |
| `mtxrIPSec` | `.20` | IPSec tunnel state — irrelevant to switch-only deployments. |
| `mtxrWifi` / (Wifi Registration Table, Wifi Interfaces) | `.21` + others | Wireless — out of scope. |
| `mtxrCT` | `.22` | Connection-tracking table — routing/firewall feature, irrelevant to L2 switch management. |

## Confirmed read-write objects (the entire write surface of this MIB)

Grepped directly from source — these are the **only three** `MAX-ACCESS read-write` objects in the entire `MIKROTIK-MIB` module:

- **`mtxrSystemReboot`** (`mtxrSystem.1` → `1.3.6.1.4.1.14988.1.1.7.1.0`) — `Integer32`, "set non zero to reboot." Confirms the claim in [overview.md](overview.md).
- **`mtxrUSBPowerReset`** (`mtxrSystem.2` → `1.3.6.1.4.1.14988.1.1.7.2.0`) — `Integer32`, "switches off usb power for specified amount of seconds."
- **`mtxrScriptRunCmd`** (inside `mtxrScriptTable`, `mtxrScripts.1` → per-row under `1.3.6.1.4.1.14988.1.1.8.1.1.3`) — `Integer32`, "set non zero to run" the named RouterOS script at that table row. This is a previously-uncatalogued (relative to `overview.md`, which mentioned it only in passing as "running a pre-configured script") **general-purpose escape hatch**: since RouterOS scripts can contain arbitrary CLI-equivalent logic, this object is functionally "SNMP-triggered execution of administrator-predefined RouterOS command sequences" — the closest thing MikroTik has to a general SNMP-driven config-write mechanism, but only for logic an administrator has pre-staged as a named script, not for arbitrary ad-hoc SignalScope-generated changes.

Every other object in every group listed above — including all 19 `mtxrHealth` scalars, the entire `mtxrInterfaceStats` and `mtxrPOE` tables, and all `mtxrSystem` identity scalars (`mtxrSerialNumber`, `mtxrFirmwareVersion`, `mtxrBoardName`, etc.) — is confirmed `read-only`.

**Standard MIB-II identity/config objects** (`SNMPv2-MIB::sysName`/`sysContact`/`sysLocation`) are writable per ordinary SNMPv2-MIB semantics, as on any vendor — not part of `MIKROTIK-MIB` itself but confirmed functional on RouterOS per [overview.md](overview.md).

## Practical takeaway for SignalScope

This MIB is built for **monitoring**, not configuration. Beyond system-identity scalars and the two/three write objects above, there is no SNMP path to interface admin state, VLAN/bridge membership, spanning-tree, or LACP — consistent with the CLI-only conclusion in [gui-cli-snmp-mapping.md](gui-cli-snmp-mapping.md) and [comparison/snmp-write-support-matrix.md](../../comparison/snmp-write-support-matrix.md). Where this MIB *is* uniquely strong relative to other vendors documented in this project is **environmental/PoE/optical health telemetry** (`mtxrHealth`, `mtxrPOE`, `mtxrOptical`) — richer per-sensor detail than the standard baseline or most other vendors' enterprise MIBs expose, and worth prioritizing for SignalScope's device-health GUI on MikroTik hardware specifically.

## Objects not confirmed this session

- `mtxrOptical` and `mtxrWifi`/wireless-registration table internals were not individually enumerated (out of scope — wireless irrelevant, optics lower priority than health/PoE for this pass).
- Exact per-model `sysObjectID` values (use live `print oid`, per [overview.md](overview.md), rather than assuming one value across all RouterOS hardware).
- `ENTITY-MIB` support was asserted in [overview.md](overview.md)'s standard-baseline list by convention but not independently re-confirmed against the vendored file (`ENTITY-MIB` is not part of `MIKROTIK-MIB` itself — it would be a separate standard-MIB implementation question).
