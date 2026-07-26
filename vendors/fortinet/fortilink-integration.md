# Fortinet — FortiGate/FortiLink-Mediated Integration

*Appendix-tier vendor doc, second file. Confirm exact syntax/OIDs against the official references linked below before relying on them for a live implementation.*

## Why this doc exists, separate from `overview.md`

[`overview.md`](overview.md) documents the **standalone FortiSwitch CLI** (`config switch ...`, direct SSH/console/Telnet to the switch). That doc's own Phase 3 scope note (2026-07-22, see [`ROADMAP.md`](../../ROADMAP.md)) flagged that **FortiLink-managed is the dominant real-world deployment shape for FortiSwitch**, not a secondary mode — most fielded units are administered entirely through a parent FortiGate's `config switch-controller` tree, with the switch itself never taking a direct config-mode session in normal operation. This is the same **controller/GUI-primacy asymmetry** already documented for Ubiquiti (and, more mildly, Netgear/Zyxel) in [`00-architecture/gui-cli-snmp-unification.md`](../../00-architecture/gui-cli-snmp-unification.md)'s third asymmetry axis: for these devices, SignalScope's session should target the parent controller (FortiGate), not the managed device, as the actual system of record.

This doc covers that path directly: the FortiGate-side CLI a SignalScope session would actually use for a FortiLink-managed FortiSwitch fleet, and how it differs in shape from every other vendor in this project (including the standalone-Fortinet doc).

## The structural difference from every other vendor doc in this project

Every other vendor doc in this tree (standalone Fortinet included) documents a session opened **directly against the device being configured**. Here, the session target and the configuration target are different systems:

- **Session target**: the FortiGate (SSH/CLI or GUI/API), not the FortiSwitch.
- **Configuration target**: a specific managed FortiSwitch, referenced **by serial number** as the `edit` key inside `config switch-controller managed-switch` — not by hostname or IP the way a direct-CLI session would address it.
- **The FortiSwitch's own local CLI is not part of this workflow at all.** A FortiLink-managed switch can still be reached by console/direct-SSH (documented in `overview.md`), but changes made that way are a separate, out-of-band path from the FortiGate-managed configuration — this project has not independently confirmed whether FortiGate-pushed FortiLink config overwrites switch-local changes the way Ubiquiti's controller does (flagged as an open question below, not assumed either way).

This means SignalScope's "one session per device" model (see `gui-cli-snmp-unification.md`'s Session model section) needs a **fleet-of-one-controller** shape for FortiLink-managed Fortinet gear: one FortiGate session can configure N managed switches, each addressed by serial number within that single session — structurally closer to Ubiquiti's controller-API model than to any direct-CLI vendor's one-session-per-device model, even though the transport here is CLI/SSH rather than a REST API.

## CLI syntax — `config switch-controller managed-switch`

Same nested `config`/`edit`/`next`/`end` block grammar as standalone FortiOS (see `overview.md`'s CLI dialect summary) — the grammar doesn't change, only which device the resulting config applies to.

### Selecting a managed switch and its ports

```
config switch-controller managed-switch
    edit <FortiSwitch-serial-number>
        config ports
            edit <port-name>
                ...
            next
        end
    next
end
```

`<FortiSwitch-serial-number>` is the managed switch's serial (e.g. `S524DF4K15000024`), not a user-assigned name — confirmed across multiple FortiGate CLI reference versions (6.2.1 through 7.4.0). `<port-name>` follows the switch's own physical port naming (`port1`, `port5`, etc.).

### Port admin status, speed, description, PoE

| Action | Command (inside `config ports` → `edit <port>`) | Notes |
|---|---|---|
| Enable/disable a port | `set status {up\|down}` | Field name is `status`, same as the standalone-CLI doc's per-port `set status` — the FortiLink path reuses the same field name, just issued from the FortiGate against the switch-controller table instead of a direct switch session. |
| Set speed | `set speed <speed>` | e.g. `10000full`. |
| Set port description | `set description <text>` | |
| PoE enable/disable | `set poe-status {enable\|disable}` | No equivalent field exists in the standalone-CLI doc's curated table — PoE control is documented here, not there. |

### VLAN membership

| Action | Command | Notes |
|---|---|---|
| Set native/untagged (PVID-equivalent) VLAN | `set vlan <vlan-name>` | Fortinet's own FortiLink guide describes this as the "default VLAN for untagged incoming frames" — conceptually `dot1qPvid`, but the field name (`vlan`) is **different from the standalone CLI's `native-vlan`** field for the same concept. SignalScope's FortiLink-vs-standalone command-emission logic must not conflate the two field names. |
| Set tagged/allowed VLANs | `set allowed-vlans <vlan-list>` | Same conceptual role as standalone CLI's `allowed-vlans`, field name matches. |
| Allow all VLANs on a port | `set allowed-vlans-all enable` | No standalone-CLI equivalent found in this project's existing research. |
| Explicitly set untagged-VLAN list (separate from native) | `set untagged-vlans <vlan-list>` | Fortinet's FortiLink guide distinguishes this from `vlan` (native) — the exact semantic difference between "native VLAN" and "untagged VLANs" as separate fields was not fully disambiguated by the pages fetched this session; verify against the version-pinned CLI reference before assuming they're redundant. |

VLANs themselves (`<vlan-name>`) are Fortinet **named objects** (`config system interface` on the FortiGate defines VLAN interfaces used here), not bare VLAN IDs the way most other vendors in this project reference VLANs — a real structural difference worth flagging for SignalScope's VLAN-picker UI, which would need to resolve a VLAN ID to its FortiGate-side interface name first.

### Spanning tree

| Action | Command | Notes |
|---|---|---|
| Global STP parameters | `config switch-controller stp-settings` → `set name <name>` / `set revision <n>` / `set hello-time <n>` / `set forward-time <n>` / `set max-age <n>` / `set max-hops <n>` → `end` | A **separate top-level `config` block** from `managed-switch`, not nested under a specific switch — global STP timers apply FortiGate-wide across its managed-switch fleet, structurally distinct from every direct-CLI vendor's per-device global STP config. |
| Per-port STP enable | `set stp-state {enabled\|disabled}` (inside `config ports` → `edit <port>`) | Same field name as the standalone-CLI doc. |
| Edge port (PortFast-equivalent) | `set edge-port enable` | Referenced as a documented prerequisite for BPDU guard in Fortinet's own STP-settings page; the fetch this session did not show this field in a standalone full syntax block the way `stp-state`/`stp-bpdu-guard` were shown — **treat as real but not independently confirmed at the same tier as the other STP fields**, verify field name/enum spelling (`enable` vs `enabled`) against the version-pinned reference before generating this in production GUI-to-CLI logic. |
| BPDU Guard | `set stp-bpdu-guard {enabled\|disabled}` + `set stp-bpdu-guard-time <0-120>` | Confirmed field names and value range, from Fortinet's own "Configuring STP settings" FortiLink guide page. |
| Root Guard | `set stp-root-guard {enabled\|disabled}` | Confirmed field name exists; full enum/behavior not independently re-verified beyond the name itself this session. |

### LACP / link aggregation (trunk)

A FortiLink-managed trunk is configured as a **port entry of `type trunk`** inside the same `config ports` table as physical ports — not a separate top-level object the way Cisco/Dell/Aruba model a port-channel interface:

```
config switch-controller managed-switch
    edit <FortiSwitch-serial-number>
        config ports
            edit <trunk-name>
                set type trunk
                set mode {static | lacp-passive | lacp-active}
                set members <port1> <port2> ...
                set aggregator-mode {bandwidth | count}
                set bundle {enable | disable}
                set min-bundle <n>
                set max-bundle <n>
            next
        end
    next
end
```

Confirmed from Fortinet's own "Adding 802.3ad link aggregation groups (trunks)" FortiSwitch-managed-by-FortiOS guide page. `mode` selects static vs. LACP passive/active, matching the same three-way choice the standalone-CLI doc documents under `config switch trunk` — **field names differ between the two paths** (`mode`/`members` here vs. `mode`/`members` in the standalone doc's `config switch trunk`, which happen to match in this one case, but `type trunk` as a `ports`-table entry has no standalone-CLI equivalent at all).

**MCLAG** (multi-chassis LACP, spanning two physically separate FortiSwitches presented as one logical peer to a downstream device) is configured with an added `set mclag enable` on the trunk port entry, and requires an MCLAG peer group configured first — this is a FortiLink-specific capability with no equivalent in the standalone-CLI doc, since MCLAG is inherently a controller-coordinated construct spanning multiple physical switches.

## SNMP

**FORTINET-FORTISWITCH-MIB tables are specifically for this integration path.** The standalone-CLI doc already noted `FgSwDeviceEntry`/`FgSwPortEntry` are "oriented toward a FortiGate polling its managed FortiSwitches" without pursuing that further — this doc is the follow-through on that flag:

- **`fgSwDeviceEntry`** (`1.3.6.1.4.1.12356.101.24.1.1.1`) — per-managed-switch device info polled from the FortiGate: platform type, device ID, serial number, name, software version, connection/authorization status, IPv4/IPv6 addresses, join time, CPU/memory utilization.
- **`fgSwPortEntry`** (`1.3.6.1.4.1.12356.101.24.2.1.1`) — per-port info polled from the FortiGate for each managed switch's ports: port identification, status, speed/duplex, VLAN settings (native/allowed/untagged), PoE capability and power draw.
- Both tables were **added to the FortiOS Enterprise MIB in FortiOS 6.4.2** — confirmed via Fortinet's own "SNMP queries to the FortiGate Switch Controller for FortiSwitch and port information" release-notes page. Query target is **the FortiGate's own SNMP agent**, not the FortiSwitch's — consistent with everything else in this doc: SignalScope would poll the FortiGate for fleet-wide managed-switch state, the same system it would open a CLI session against for config changes.
- **Write access: not confirmed either way from primary source this session.** One search-summary result characterized the managed-FortiSwitch SNMP implementation as read-only for v1/v2c access, but a direct fetch of the FortiGate's own release-notes page describing these two tables did not itself state read-only vs. read-write explicitly — only `snmpwalk` (read) example queries were shown. Per this docs tree's standing discipline (never assert a write path without a documented workflow), **treat `fgSwDeviceEntry`/`fgSwPortEntry` as read-only/monitoring** until a primary source explicitly documents a SET workflow — consistent with the standalone-CLI doc's existing "treat FortiSwitch SNMP as read-oriented in practice" guidance, now extended to the FortiLink path specifically.

## Illustrative GUI/CLI/SNMP mapping (FortiLink-managed path)

| GUI Action | FortiGate CLI | SNMP | Notes |
|---|---|---|---|
| Enable/disable a port | `config switch-controller managed-switch` / `edit <serial>` / `config ports` / `edit <port>` / `set status up` / `next` / `end` | `fgSwPortEntry` read-back only; no confirmed SET path | Session target is the FortiGate, not the switch — see structural-difference section above. |
| Set port VLAN (native + tagged) | Same path, `set vlan <name>` (native) + `set allowed-vlans <list>` (tagged) | Read-back via `fgSwPortEntry` | VLANs referenced by FortiGate-side interface name, not bare VLAN ID — see VLAN section above. |
| Enable STP edge-port + BPDU Guard | Same path, `set edge-port enable` + `set stp-state enabled` + `set stp-bpdu-guard enabled` | Not confirmed | `edge-port` field confirmed to exist but at a lower confidence tier than the others — see STP section. |
| Create an LACP trunk | Same path, `edit <trunk-name>` / `set type trunk` / `set mode lacp-active` / `set members <ports>` | `fgSwPortEntry` would show the resulting trunk as a port entry, read-back only | No standalone-CLI equivalent shape (`type trunk` as a ports-table row rather than a separate `config switch trunk` object). |
| Persist configuration | *(same auto-save-on-`end` behavior as standalone FortiOS — no separate command)* | N/A | Confirmed unchanged from the standalone-CLI doc's finding: FortiOS/FortiSwitchOS's `end`-closes-and-persists model applies identically here. |

## Official sources

- [config switch-controller managed-switch — FortiGate/FortiOS 7.4.0 CLI Reference](https://docs.fortinet.com/document/fortigate/7.4.0/cli-reference/205620/config-switch-controller-managed-switch)
- [switch-controller managed-switch — FortiGate/FortiOS 6.2.1 CLI Reference](https://docs.fortinet.com/document/fortigate/6.2.1/cli-reference/174620/switch-controller-managed-switch)
- [Configuring VLANs — FortiSwitch 7.6.5 FortiLink Guide](https://docs.fortinet.com/document/fortiswitch/7.6.5/fortilink-guide/546342/configuring-vlans) — `vlan`/`allowed-vlans`/`untagged-vlans`/`allowed-vlans-all` field syntax.
- [Configuring STP settings — FortiSwitch 8.0.0 FortiLink Guide](https://docs.fortinet.com/document/fortiswitch/8.0.0/fortilink-guide/173292/configuring-stp-settings) — global `switch-controller stp-settings`, per-port `stp-state`/`stp-bpdu-guard`/`stp-bpdu-guard-time`/`stp-root-guard`/`edge-port`.
- [FortiSwitch port features — FortiSwitch 6.4.2 Devices Managed by FortiOS](https://docs.fortinet.com/document/fortiswitch/6.4.2/devices-managed-by-fortios/749330/fortiswitch-port-features) — `status`/`speed`/`description`/`poe-status` per-port fields.
- [Adding 802.3ad link aggregation groups (trunks) — FortiSwitch 7.0.8 Devices Managed by FortiOS](https://docs.fortinet.com/document/fortiswitch/7.0.8/devices-managed-by-fortios/801170/adding-802-3ad-link-aggregation-groups-trunks) — `type trunk`/`mode`/`members`/`aggregator-mode`/`bundle`/`min-bundle`/`max-bundle`/`mclag` field syntax.
- [SNMP queries to the FortiGate Switch Controller for FortiSwitch and port information — FortiOS 6.4.0 New Features](https://docs.fortinet.com/document/fortigate/6.4.0/new-features/87594/snmp-queries-to-the-fortigate-switch-controller-for-fortiswitch-and-port-information-6-4-2) — `fgSwDeviceEntry`/`fgSwPortEntry` introduction, OIDs, FortiOS 6.4.2 origin.
- [FortiGate switch MIBs — FortiGate/FortiOS 7.6.0](https://docs.fortinet.com/document/fortigate/7.6.0/fortigate-mib-information-overview/508932/fortigate-switch-mibs) — same tables, cited already in `overview.md`.

## Open questions for a future pass

- Whether FortiGate-pushed FortiLink config overwrites switch-local CLI changes the way Ubiquiti's controller does (this project's Ubiquiti finding) — not confirmed either way this session; if FortiLink behaves the same way, that reinforces treating the FortiGate as the sole authoritative session target for these devices, the same architectural treatment already applied to Ubiquiti.
- Exact semantic distinction between `vlan` (native) and `untagged-vlans` as separate fields in the VLAN table — Fortinet's own page distinguishes them but this session's fetch didn't fully disambiguate the difference in practice.
- `edge-port`'s exact enum spelling (`enable`/`disable` vs. `enabled`/`disabled` — inconsistent with the `stp-state`/`stp-bpdu-guard` fields' `enabled`/`disabled` pattern in what was fetched) — verify against the version-pinned CLI reference.
- Whether `fgSwPortEntry`/`fgSwDeviceEntry` genuinely have zero write-capable objects, or whether this session's inability to find a documented SET workflow just reflects thin source coverage (same "not confirmed" vs. "confirmed absent" distinction this docs tree tries to keep honest about elsewhere, e.g. Dell's MIB set — this Fortinet finding is currently the weaker, "not confirmed" tier, not Dell's stronger "confirmed absent" tier).
