# Dell SmartFabric OS10 — MIB Reference

## Provenance note

Unlike the original seven Tier-1 vendors, this vendor's MIB research was conducted via search-summary results and targeted page fetches (Dell's own documentation portal and a third-party MIB-browser mirror) rather than a full primary MIB-source read. Treat OID values below as "confirmed to exist at this location" where a source directly stated it, and as "plausible, needs live-device or full-document confirmation" everywhere else — the same discipline used throughout this docs tree, just applied to thinner source material for this vendor. See [`overview.md`](overview.md)'s closing uncertainty note.

## A genuine PEN ambiguity, documented rather than resolved

Two different Dell-associated Private Enterprise Numbers surfaced during this research, and they are **not interchangeable** — a future reader should not assume one supersedes the other without checking which hardware/firmware generation is in play:

- **PEN 674 ("Dell Inc.")** — Dell's own long-standing enterprise number. OS10's own product/enterprise MIB tree is confirmed rooted here: `DELLEMC-OS10-PRODUCTS-MIB`'s `os10Products` object is at `1.3.6.1.4.1.674.11000.5000.100.2` (confirmed via a third-party MIB-browser mirror — see [`mibs/README.md`](mibs/README.md)), i.e. the chain is `enterprises.674` → `11000` (Dell EMC OS10 products arc) → `5000` → `100` → `2`. A separate reference surfaced during this session (`1.3.6.1.4.1.674.11000.5000.200.1.1`, BGP-related) suggests `11000.5000` is a broader OS10-enterprise-MIB parent arc with multiple sibling branches (`.100.*` for chassis/product objects, `.200.*` for BGP, plausibly others not investigated this session) — worth confirming the full sibling list before treating `.100.2` as the only entry point.
- **PEN 6027 ("Force10 Networks")** — Dell's acquired Force10 heritage number. Search results this session associated `sysObjectID` values for specific PowerSwitch models (S3048-ON, S4048-ON, S6010-ON) with enterprise number 6027, not 674 — meaning **`sysObjectID`-based vendor/model auto-detection for at least some OS10-capable hardware may resolve under the legacy Force10 PEN, not Dell's own 674**, even though the device is running current OS10 software. This was not fully reconciled this session (i.e., it's unclear whether *all* OS10 hardware reports `sysObjectID` under 6027, or only specific model families, or only devices that started life on OS9/FTOS before an OS10 conversion).

**Practical implication for SignalScope**: per [`standard-mibs.md`](../../00-architecture/standard-mibs.md)'s "poll `sysObjectID` first" guidance, do not hardcode a single expected PEN for Dell/OS10 vendor auto-detection — check for **both** `1.3.6.1.4.1.674.*` and `1.3.6.1.4.1.6027.*` prefixes, and treat resolving this cleanly (which PEN which model/firmware combination actually reports) as an open item for the verification backlog in [`ROADMAP.md`](../../ROADMAP.md).

## Standard (RFC/IEEE) MIBs supported

Per [`standard-mibs.md`](../../00-architecture/standard-mibs.md)'s shared baseline: **SNMPv2-MIB, IF-MIB** confirmed present (standard for any SNMP-speaking switch, not separately verified as an OS10-specific claim). **BRIDGE-MIB**: Dell's own OS10 User Guide MIBs page states **partial implementations of RFC 4188 (BRIDGE-MIB) and RFC 4363 (Q-BRIDGE-MIB)** are supported — this "partial" qualifier is an explicit vendor statement, not a hedge added by this docs project, and is worth treating as a real signal that some BRIDGE-MIB/Q-BRIDGE-MIB objects may return `noSuchObject` on OS10 even though the module is nominally supported. **LLDP-MIB, ENTITY-MIB, RMON-MIB**: not independently confirmed this session — treat as "expected, standard for the device class, unverified" per this docs tree's usual convention for gaps, and prioritize verifying BRIDGE-MIB/Q-BRIDGE-MIB's exact partial coverage before relying on VLAN or spanning-tree SNMP reads for OS10 devices.

## Confirmed vendor-specific behavior

- **Standard `BGP4-MIB` (`1.3.6.1.2.1.15`) returns "No such object" on OS10** — Dell's own documentation states OS10 uses the proprietary `DELLEMC-OS10-BGP4V2-MIB` instead. While BGP itself is a routing concern outside SignalScope's L2-switch-focused scope, this is a useful signal about OS10's general posture: **prefer checking for a Dell enterprise MIB before assuming standard-MIB coverage is complete**, more so than for the other six vendors researched with full primary-source access, where the standard-MIB baseline was more reliably confirmed protocol-by-protocol.
- `DELLEMC-OS10-PRODUCTS-MIB` (`os10Products`, `.674.11000.5000.100.2`) — existence and OID root confirmed; internal object/table structure (equivalent to `ENTITY-MIB::entPhysicalTable` for chassis/module/PSU/fan inventory, if that's what this module covers) **not enumerated this session** — a good target for a follow-up research pass given SignalScope's inventory-tracking use case (see [`data-model-notes.md`](../../00-architecture/data-model-notes.md)).

## SNMP write support

**Not independently confirmed this session for any object** — no equivalent of Cisco's `CISCO-VLAN-MEMBERSHIP-MIB` or Aruba's `ARUBAWIRED-PORTSECURITY-MIB`/`ARUBAWIRED-CONFIG-MIB` write workflows was located or ruled out. Per [`gui-cli-snmp-unification.md`](../../00-architecture/gui-cli-snmp-unification.md)'s standing rule, **treat every OS10 configuration action as CLI-only** (the `switchport`/`channel-group`/`spanning-tree`/`switchport port-security` commands in [`cli-reference.md`](cli-reference.md)) until a specific object is separately confirmed writable — this vendor should be added to [`comparison/snmp-write-support-matrix.md`](../../comparison/snmp-write-support-matrix.md) as "not yet researched" rather than assumed narrow-or-broad by analogy to any other vendor.

## Objects not confirmed this session

- The full sibling-arc list under `enterprises.674.11000.5000` (only `.100.*` and `.200.*` were glimpsed).
- Which PEN (674 vs. 6027) a given OS10-capable model actually reports in `sysObjectID` — see the PEN ambiguity section above, flagged for the [`ROADMAP.md`](../../ROADMAP.md) verification backlog.
- Exact `MAX-ACCESS` levels for any object in `DELLEMC-OS10-PRODUCTS-MIB` or any sibling module.
- LLDP-MIB, ENTITY-MIB, RMON-MIB support (assumed-but-unverified per the standard-baseline note above).
