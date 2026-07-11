# Cisco IOS / IOS-XE — CLI Reference

Curated command tables for the Catalyst 9000-family IOS-XE dialect (applies near-identically to older IOS-based Catalyst platforms). Interface names below use `GigabitEthernet0/1` as the example — substitute the platform's actual interface naming (`TwoGigabitEthernet1/0/1`, `TenGigabitEthernet1/1/1`, `Gi1/0/1` shorthand accepted interactively, etc.) and `Port-channel<N>` for EtherChannel logical interfaces.

## Session setup: privileged / config mode

| Task | Command(s) | Notes |
|---|---|---|
| Enter privileged EXEC | `enable` | Prompts for the enable secret/password if configured (`hostname>` → `hostname#`). Required before `configure terminal`. |
| Enter global config mode | `configure terminal` (or `conf t`) | `hostname#` → `hostname(config)#`. |
| Enter interface sub-mode | `interface GigabitEthernet0/1` | `hostname(config)#` → `hostname(config-if)#`. Also accepts a range: `interface range GigabitEthernet0/1 - 4`. |
| Exit one mode level | `exit` | Steps back one context level. |
| Return directly to privileged EXEC from any config depth | `end` (or Ctrl-Z) | Common terminator after a batch of config-mode lines. |

## Interface admin state & description

| Task | Command(s) | Notes |
|---|---|---|
| Disable a port | `interface GigabitEthernet0/1` then `shutdown` | Sets `IF-MIB::ifAdminStatus` to `down(2)`. |
| Enable a port | `interface GigabitEthernet0/1` then `no shutdown` | Sets `ifAdminStatus` to `up(1)`. |
| Set interface description | `interface GigabitEthernet0/1` then `description <text>` | Maps to `IF-MIB::ifAlias` (`ifXTable`). Max length is platform/IOS-version dependent (commonly 240 chars on IOS-XE). |
| Set speed | `interface GigabitEthernet0/1` then `speed {10 \| 100 \| 1000 \| 2500 \| 5000 \| 10000 \| auto}` | Valid values depend on the physical port's supported speeds. |
| Set duplex | `interface GigabitEthernet0/1` then `duplex {auto \| full \| half}` | `half` is not valid/meaningful above 100 Mbps on modern hardware. |

## VLAN configuration

| Task | Command(s) | Notes |
|---|---|---|
| Create a VLAN | `configure terminal` → `vlan 10` → `name SALES` → `end` | `vlan 10` enters VLAN config sub-mode (`(config-vlan)#`) and implicitly creates VLAN 10 if it doesn't exist; `name` is optional. |
| Delete a VLAN | `configure terminal` → `no vlan 10` | |
| Configure access port | `interface GigabitEthernet0/1` → `switchport mode access` → `switchport access vlan 10` | Sets the port's PVID; corresponds to `Q-BRIDGE-MIB::dot1qPvid`. |
| Configure trunk port | `interface GigabitEthernet0/1` → `switchport trunk encapsulation dot1q` (only needed on platforms supporting ISL, mostly legacy) → `switchport mode trunk` → `switchport trunk allowed vlan 10,20,30` | `switchport trunk allowed vlan` replaces the allowed list; use `add`/`remove`/`except` sub-forms (e.g. `switchport trunk allowed vlan add 40`) to modify incrementally rather than replace. |
| Set native VLAN on a trunk | `interface GigabitEthernet0/1` then `switchport trunk native vlan 99` | |
| Read back VLAN-to-port mapping | `show vlan brief` / `show interfaces trunk` | See "show commands" section. |

## Spanning tree

| Task | Command(s) | Notes |
|---|---|---|
| Set global STP mode | `configure terminal` → `spanning-tree mode {pvst \| rapid-pvst \| mst}` | `pvst` = classic Per-VLAN Spanning Tree (802.1D+Cisco PVST extensions), `rapid-pvst` = Rapid PVST+ (802.1w per VLAN, Cisco's common modern default), `mst` = Multiple Spanning Tree (802.1s). |
| Enable PortFast on one port | `interface GigabitEthernet0/1` then `spanning-tree portfast` | Only appropriate on access ports connecting to end hosts, not to other switches — IOS-XE prints a warning if issued on a port that could form a loop. |
| Enable PortFast on all eligible ports globally | `configure terminal` then `spanning-tree portfast default` | Applies to all non-trunking ports; per-port `spanning-tree portfast disable` opts a specific port out. |
| Set per-port path cost | `interface GigabitEthernet0/1` then `spanning-tree cost <1-200000000>` | Maps toward `BRIDGE-MIB::dot1dStpPortPathCost` (read side); no reliable SNMP-write path (see `mib-reference.md`). |
| Set per-port priority | `interface GigabitEthernet0/1` then `spanning-tree port-priority <0-240, increments of 16>` | |
| Set per-VLAN bridge priority | `configure terminal` then `spanning-tree vlan 10 priority 4096` | Value must be a multiple of 4096. |
| Read back STP state | `show spanning-tree` / `show spanning-tree interface GigabitEthernet0/1` | See "show commands" section. |

## LACP / EtherChannel

| Task | Command(s) | Notes |
|---|---|---|
| Add a port to a LACP channel (active) | `interface GigabitEthernet0/1` then `channel-group 1 mode active` | Creates `Port-channel1` automatically if it doesn't exist. `active` = this side initiates LACP negotiation. |
| Add a port to a LACP channel (passive) | `interface GigabitEthernet0/1` then `channel-group 1 mode passive` | Responds to LACP but doesn't initiate; at least one side of a channel must be `active`. |
| Static/no-protocol EtherChannel | `interface GigabitEthernet0/1` then `channel-group 1 mode on` | No LACP/PAgP negotiation at all — both ends must be manually configured identically; not interoperable with `active`/`passive` on the same group. |
| Configure the Port-channel logical interface | `interface Port-channel1` then apply `switchport`/`spanning-tree`/etc. commands as normal | Member ports typically inherit L2 config from the port-channel once bundled, depending on platform/config; still expected to configure the logical interface directly for VLAN/trunk settings. |
| Read back EtherChannel/LACP state | `show etherchannel summary` / `show lacp neighbor` | See "show commands" section. |

## Port security

| Task | Command(s) | Notes |
|---|---|---|
| Enable port security on a port | `interface GigabitEthernet0/1` → `switchport mode access` (required first — port security needs a fixed access or trunk mode, not dynamic) → `switchport port-security` | Enabling with no other options applies IOS-XE defaults (max 1 MAC, violation action `shutdown`). |
| Set max secure MAC addresses | `interface GigabitEthernet0/1` then `switchport port-security maximum 3` | |
| Set violation action | `interface GigabitEthernet0/1` then `switchport port-security violation {protect \| restrict \| shutdown}` | `protect`: drop over-limit traffic silently. `restrict`: drop + log/increment counter + optional SNMP trap. `shutdown` (default): err-disable the port entirely on violation — requires manual `shutdown`/`no shutdown` or `errdisable recovery` to bring back up. |
| Configure sticky learning | `interface GigabitEthernet0/1` then `switchport port-security mac-address sticky` | Learned MACs get written into the running-config as static entries. |
| Read back port-security state | `show port-security interface GigabitEthernet0/1` / `show port-security address` | See "show commands" section. |

## SNMP agent configuration (on the device itself)

| Task | Command(s) | Notes |
|---|---|---|
| Configure a read-only community | `configure terminal` then `snmp-server community public RO` | Cleartext, v1/v2c only. Avoid `public`/`private` defaults in real deployments; SignalScope should flag this as insecure per `connectivity-methods.md`. |
| Configure a read-write community | `snmp-server community private RW` | Enables SNMP SET for whatever's supported (see `overview.md`/`mib-reference.md` for what that actually covers). Optionally scope with an ACL: `snmp-server community private RW 10` (numbered ACL 10). |
| Configure a trap/inform destination | `snmp-server host 10.0.0.5 traps public` (or `informs` instead of `traps`, and a v3 user instead of a community for SNMPv3 notifications) | Requires trap types to actually be enabled (next row) to fire. |
| Enable specific trap categories | `snmp-server enable traps snmp linkdown linkup` / `snmp-server enable traps config` / etc. | IOS-XE has dozens of trap categories (`snmp-server enable traps ?` lists what a given image/platform supports); nothing is sent unless both enabled here *and* a `snmp-server host` destination exists. |
| Configure SNMPv3 group (security model + view) | `snmp-server group ADMINGROUP v3 priv read ADMINVIEW write ADMINVIEW` | Ties a group name to an auth level (`noauth`/`auth`/`priv`) and read/write MIB views. |
| Configure SNMPv3 user | `snmp-server user admin ADMINGROUP v3 auth sha AUTHPASSWORD priv aes 128 PRIVPASSWORD` | Must reference an existing group (previous row). SHA auth + AES priv is the modern-recommended combination; MD5/DES are legacy/weaker options still accepted for compatibility. |
| Restrict SNMP access by ACL | `snmp-server community private RW 10` (paired with `access-list 10 permit 10.0.0.0 0.0.0.255`) | Applies to v1/v2c community-based access; v3 access control is via the group/view model above instead. |

## Paging control (automation-session setup)

| Task | Command(s) | Notes |
|---|---|---|
| Disable `--More--` pagination for the current session | `terminal length 0` | Must be sent once per session (it's a per-line/per-VTY setting, not persistent across reconnects) before running any `show` command whose output could exceed a screen, or a scripted session will hang waiting for a keypress. Per the architecture doc, SignalScope should send this automatically at session start but **echo it in the terminal** like any other command. |
| Disable it persistently for a given line/vty (not per-session) | `configure terminal` → `line vty 0 15` → `length 0` | Device-side persistent alternative if SignalScope prefers not to send it every session — changes the actual line config rather than a per-session default. |

## Save / persist configuration

| Task | Command(s) | Notes |
|---|---|---|
| Persist running-config to NVRAM (interactive) | `copy running-config startup-config` | Prompts `Destination filename [startup-config]?` — press Enter to accept default. |
| Persist running-config to NVRAM (non-interactive shorthand) | `write memory` (or `wr`) | Functionally equivalent to the above with no prompt; still the most common form in scripts/automation. |
| View pending (unsaved) differences | `show archive config differences running-config startup-config` (requires the `archive` feature enabled) | Not available by default on all images; if absent, there is no built-in "diff before save" — `show running-config` vs. the last-known `startup-config` snapshot is the fallback (this is effectively what SignalScope's own config-drift/backup tooling does per `connectivity-methods.md`). |

## Useful `show` commands for read-back / polling

| Task | Command(s) | Notes |
|---|---|---|
| Per-interface admin/oper state + speed/duplex/VLAN summary | `show interfaces status` | Compact one-line-per-port table; good for GUI port-grid read-back. |
| Full interface detail (counters, description, etc.) | `show interfaces GigabitEthernet0/1` | |
| VLAN-to-port membership | `show vlan brief` | |
| Trunk state and allowed-VLAN lists | `show interfaces trunk` | |
| Spanning-tree state (all VLANs/instances) | `show spanning-tree` | |
| Spanning-tree state for one interface | `show spanning-tree interface GigabitEthernet0/1` | |
| EtherChannel/LACP summary | `show etherchannel summary` | |
| Port-security state | `show port-security interface GigabitEthernet0/1` | |
| Effective running config for one interface | `show running-config interface GigabitEthernet0/1` | The most reliable single source for "what did the GUI/CLI actually apply" read-back on a specific interface — this is what SignalScope's CLI→GUI diff-detection (per `gui-cli-snmp-unification.md`) should target after any interface-scoped change, rather than parsing raw command echo alone. |
| SNMP agent's own config | `show running-config \| section snmp-server` | Useful for verifying SNMP config changes landed as expected. |
