# Huawei VRP — CLI Reference

Curated command tables for VRP on S-series switches (S1720/S2700/S5700/S6700/S6720 and CloudEngine S3700/S5700/S6700 "V600" trains). See [overview.md](overview.md) for the broader dialect notes (prompts, `display` vs `show`, `undo` vs `no`, immediate-apply model). Sources are linked inline and rolled up in overview.md.

## Entering system-view and interface view

| Task | Command(s) | Notes |
|---|---|---|
| Enter system-view (global config) | `system-view` | Prompt changes from `<hostname>` to `[hostname]`. No separate `enable`/privileged-EXEC step is required by default on most S-series deployments, unlike Cisco IOS — though VRP does support user privilege levels (0-15) that can gate this. |
| Enter interface view | `interface GigabitEthernet0/0/1` | Three-part slot/port numbering (`stack-id/slot/port` or `chassis/slot/port` depending on model) — **not** the two-part `0/1` Cisco uses. Prompt becomes `[hostname-GigabitEthernet0/0/1]`. |
| Enter VLAN view | `vlan 10` | Prompt becomes `[hostname-vlan10]`; used for VLAN-scoped config (e.g. `description`, `port` membership shortcut) as an alternative to per-interface config. |
| Return to system-view from interface/VLAN view | `quit` | One level up per `quit`; `return` jumps directly to user view from any depth. |
| Exit system-view to user view | `quit` (from `[hostname]`) or `return` (from any depth) | |

## Interface admin state & description

| Task | Command(s) | Notes |
|---|---|---|
| Disable a port (admin down) | `system-view` → `interface GigabitEthernet0/0/1` → `shutdown` | Maps to `IF-MIB::ifAdminStatus` = `down(2)`. |
| Enable a port (admin up) | `system-view` → `interface GigabitEthernet0/0/1` → `undo shutdown` | Maps to `IF-MIB::ifAdminStatus` = `up(1)`. Negation keyword is `undo`, not `no`. |
| Set interface description | `system-view` → `interface GigabitEthernet0/0/1` → `description <text>` | Maps to `IF-MIB::ifAlias`. Description string length limits vary by device/version (commonly up to 242 or 255 chars) — confirm against the specific device's command reference rather than assuming a fixed max. |

## VLAN configuration

| Task | Command(s) | Notes |
|---|---|---|
| Create a VLAN | `system-view` → `vlan 10` | Creates and enters VLAN 10's view in one step. |
| Create multiple VLANs in one command | `vlan batch 10 15 to 19 25` | Individual IDs and `to`-ranges can be mixed; max 10 non-contiguous IDs/ranges per invocation per Huawei docs — issue the command again for more. [Source](https://support.huawei.com/enterprise/en/doc/EDOC1000178168/a197c0b1/how-do-i-create-vlans-in-a-batch) |
| Set interface to access mode | `interface GigabitEthernet0/0/1` → `port link-type access` | Analogous to Cisco `switchport mode access`. |
| Set access VLAN (PVID) | `interface GigabitEthernet0/0/1` → `port default vlan 10` | Equivalent to `port vlan 10` issued from VLAN view instead; maps conceptually to `Q-BRIDGE-MIB::dot1qPvid`. Requires `port link-type access` (or `hybrid`) first. [Source](https://support.huawei.com/enterprise/en/doc/EDOC1100333403/a0b2c054/port-vlan) |
| Set interface to trunk mode | `interface GigabitEthernet0/0/1` → `port link-type trunk` | Analogous to Cisco `switchport mode trunk`. |
| Allow VLANs on a trunk | `port trunk allow-pass vlan 10 to 20` or `port trunk allow-pass vlan all` | Note VLAN 1 is allowed by default on a new trunk port and commonly removed explicitly (`undo port trunk allow-pass vlan 1`) in hardening configs. |

## Spanning-tree configuration

| Task | Command(s) | Notes |
|---|---|---|
| Enable STP globally | `system-view` → `stp enable` | VRP defaults to MSTP mode; mode is selectable via `stp mode { stp \| rstp \| mstp }`. |
| Disable STP on an interface | `interface GigabitEthernet0/0/1` → `stp disable` | Per-port opt-out; global `stp enable` must be on first. |
| Configure a port as an edge port | `interface GigabitEthernet0/0/1` → `stp edged-port enable` | Skips STP calculation on that port for fast link-up (Cisco's `spanning-tree portfast` equivalent); also implicitly configures the port as a BPDU-filter port. `undo stp edged-port` restores automatic edge-port detection. [Source](https://support.huawei.com/enterprise/en/doc/EDOC1000015892/80fee1e/stp-edged-port) |
| Global edge-port default behavior | `stp edged-port default` (system-view) | Sets the default edge-port state applied to all ports absent a per-port override. |

## LACP / Eth-Trunk

| Task | Command(s) | Notes |
|---|---|---|
| Create an Eth-Trunk interface | `system-view` → `interface Eth-Trunk1` | Creates/enters the logical aggregation interface, VRP's port-channel equivalent. |
| Set static LACP mode | `interface Eth-Trunk1` → `mode lacp-static` | Eth-Trunk must have **no member interfaces** at the time this is run — remove members first if changing mode on an existing trunk. [Source](https://info.support.huawei.com/hedex/api/pages/EDOC1100277644/AEM10221/03/resources/vrp/dc_vrp_ethtrunk_cfg_0061.html) |
| Add a member interface to the trunk | `interface GigabitEthernet0/0/1` → `eth-trunk 1` | Run per member interface; binds that physical port into Eth-Trunk1. |
| Set LACP priority (optional, static mode) | `interface Eth-Trunk1` → `lacp priority <value>` | Lower value = higher priority for active/standby selection. |

## Port security

| Task | Command(s) | Notes |
|---|---|---|
| Enable port security on an interface | `interface GigabitEthernet0/0/1` → `port-security enable` | Must be enabled before configuring max-MAC count, sticky MAC, or protection action. Cannot coexist with `mac-limit maximum` on the same interface. |
| Set max learned secure MAC count | `port-security max-mac-num 5` | Default is 1 learned MAC per interface if port security is enabled without this. Aggregate cap across all port-security-enabled interfaces on a device is commonly 4096 (device-dependent — verify per model). [Source](https://support.huawei.com/enterprise/en/doc/EDOC1000178165/e2870031/port-security-configuration-commands) |
| Set violation/protection action | `port-security protect-action { protect \| restrict \| shutdown }` | Analogous to Cisco `switchport port-security violation`. |
| Enable sticky secure MAC | `port-security mac-address sticky` | Persists dynamically-learned secure MACs into the running config. |

## SNMP agent configuration

| Task | Command(s) | Notes |
|---|---|---|
| Enable an SNMP version | `snmp-agent sys-info version { v1 \| v2c \| v3 \| all }` | `all` enables v1/v2c/v3 concurrently. |
| Configure a v1/v2c read community | `snmp-agent community read <community-name> [ mib-view <view-name> ] [ acl <acl-number> ]` | Community strings are cleartext — flag per [connectivity-methods.md](../../00-architecture/connectivity-methods.md) security notes. |
| Configure a v1/v2c write community | `snmp-agent community write <community-name> [ mib-view <view-name> ] [ acl <acl-number> ]` | Write access still gated by the bound MIB view — see mib-view section below and [mib-reference.md](mib-reference.md). |
| Configure a trap target host | `snmp-agent target-host trap address udp-domain <ip-address> params securityname <security-string> [ v1 \| v2c \| v3 [ authentication \| privacy ] ]` | Must be paired with `snmp-agent trap enable` (globally or per feature) or no traps are actually sent. [Source](https://support.huawei.com/enterprise/en/doc/EDOC1000041693/4753c85b/snmp-agent-target-host-trap) |
| Enable trap sending | `snmp-agent trap enable` | Global switch; can be scoped to a feature, e.g. `snmp-agent trap enable feature-name lldp`. |
| Configure an SNMPv3 user | `snmp-agent usm-user v3 <user-name> <group-name> authentication-mode { md5 \| sha \| sha2-256 } <auth-password> privacy-mode { des56 \| 3des168 \| aes128 \| aes192 \| aes256 } <priv-password>` | Password: 8-255 case-sensitive characters, must include at least 2 of {upper, lower, digit, special}. Newer VRP releases add `remote-engineid` for configuring a remote-engine user. [Source](https://support.huawei.com/enterprise/en/doc/EDOC1000128405/72d3e0f1/snmp-agent-usm-user) |
| Configure an SNMPv3 group | `snmp-agent group v3 <group-name> { authentication \| noauth \| privacy } [ read-view <view> ] [ write-view <view> ] [ notify-view <view> ] [ acl <acl-number> ]` | Binds a security level and MIB views to the group; users are then assigned to the group via `snmp-agent usm-user`. |
| Create/extend a MIB view (access control) | `snmp-agent mib-view { excluded \| included } <view-name> <oid-tree>` | See [overview.md](overview.md) and [mib-reference.md](mib-reference.md) for the access-control model this enables. `oid-tree` can be numeric (`1.3.6.1.2.1`) or object-name (`system.7`) form. Default view `ViewDefault` is immutable. |
| Read back configured MIB views | `display snmp-agent mib-view [ <view-name> ]` | |

## Paging control

| Task | Command(s) | Notes |
|---|---|---|
| Disable pagination for the current session | `screen-length 0 temporary` | Scoped to the current terminal session only (`temporary` keyword) — not persisted to the saved config, so SignalScope must send this at the start of every new CLI session rather than assuming it sticks. Equivalent to Cisco `terminal length 0`. |
| Set persistent screen length (config-level) | `screen-length <0-512>` (from user-interface view, e.g. `user-interface vty 0 4`) | Persists across sessions if saved; the `temporary` per-session form above is the one SignalScope should default to using so it doesn't silently alter the device's baseline config. |

## Save/persist configuration

| Task | Command(s) | Notes |
|---|---|---|
| Save running config to next-startup config file | `save` | Interactive: if a next-startup file is already designated, prompts `Are you sure to continue?[Y/N]`; if none exists yet, prompts for a filename (Enter accepts default `vrpcfg.zip`). SignalScope's terminal automation needs to handle this Y/N (and optional filename) prompt, not just fire-and-forget the command. [Source](https://support.huawei.com/enterprise/en/doc/EDOC1000178166/4adec9f7/saving-the-configuration-file) |
| Save to an explicit named file (no prompt) | `save <configuration-file>` | Bypasses the interactive filename step; still prompts Y/N to confirm the save itself in most VRP versions. |

## Display (read-back) commands

| Task | Command(s) | Notes |
|---|---|---|
| Interface summary | `display interface brief` | Analogous to Cisco `show interfaces status`/`show ip interface brief` combined — shows admin/oper state, description snippet, speed/duplex per interface. |
| Full interface detail | `display interface GigabitEthernet0/0/1` | Counters, negotiated speed/duplex, CRC/error counters — CLI-side equivalent of walking `IF-MIB`/`ifXTable` for that ifIndex. |
| VLAN summary | `display vlan` | Lists all VLANs and their (abbreviated) port membership. |
| Single VLAN detail | `display vlan 10` | Full tagged/untagged port membership for one VLAN. |
| Spanning-tree summary | `display stp brief` | Per-port STP role/state summary, analogous to Cisco `show spanning-tree summary`. |
| Spanning-tree interface detail | `display stp interface GigabitEthernet0/0/1` | |
| Eth-Trunk summary | `display eth-trunk` or `display eth-trunk 1` | Shows member ports, LACP state (Selected/Unselected), aggregation mode. |
| Port security state | `display port-security` | |
| Running config (full or filtered) | `display current-configuration [ interface GigabitEthernet0/0/1 ]` | VRP's `show running-config` equivalent; supports the same "diff before/after a raw CLI command" use case described in [gui-cli-snmp-unification.md](../../00-architecture/gui-cli-snmp-unification.md). |
| SNMP-agent config read-back | `display snmp-agent community` / `display snmp-agent group` / `display snmp-agent usm-user` / `display snmp-agent mib-view` | Needed for the GUI to reflect SNMP config state accurately. |
