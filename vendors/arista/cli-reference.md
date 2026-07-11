# Arista EOS — CLI Reference (curated)

Not exhaustive — curated to the command set SignalScope's GUI actions and terminal-echo model need. "IOS parity" column calls out whether the exact text is identical to Cisco IOS or diverges, feeding [comparison/cli-syntax-matrix.md](../../comparison/cli-syntax-matrix.md).

Sources: [EOS SNMP](https://www.arista.com/en/um-eos/eos-snmp), [EOS Port Channels and LACP](https://www.arista.com/en/um-eos/eos-port-channels-and-lacp), [EOS Spanning Tree Protocol](https://www.arista.com/en/um-eos/eos-spanning-tree-protocol), [EOS Virtual LANs (VLANs)](https://www.arista.com/en/um-eos/eos-virtual-lans-vlans), [EOS Configuration Files](https://www.arista.com/en/um-eos/eos-configuration-files), [EOS Command-Line Interface (CLI)](https://www.arista.com/en/um-eos/eos-command-line-interface-cli).

## Mode entry

| Task | Command(s) | Notes |
|---|---|---|
| Enter privileged EXEC | `enable` | Identical to IOS. |
| Enter global config mode | `configure` (or `configure terminal`) | Identical to IOS; EOS accepts the short form `configure`. |
| Enter interface config mode | `interface Ethernet1` | IOS-parity in verb/structure. **Naming diverges**: EOS typically uses `Ethernet1`, `Ethernet1/1` (no `GigabitEthernet0/1`-style speed-encoded name) — actual token depends on platform/fixed vs modular chassis. |
| Enter VLAN config mode | `vlan 10` then `name <text>` | Similar to IOS `vlan database`/`vlan <id>` context; confirmed syntax: `switch(config)# vlan 25` → `switch(config-vlan-25)# name corporate_100`. |
| Exit a config sub-mode | `exit` | Identical to IOS. |

## Interface admin state & description

| Task | Command(s) | Notes |
|---|---|---|
| Disable a port | `shutdown` (in interface config mode) | Identical to IOS. |
| Enable a port | `no shutdown` | Identical to IOS. |
| Set interface description | `description <text>` | Identical to IOS. Maps to `IF-MIB::ifAlias` for SNMP read-back (writable per Arista's stated MIB exceptions). |
| Set interface speed | `speed <speed>` (e.g. `speed 10full`, `speed forced 10000full`, `speed auto`) | Verb identical to IOS; exact value tokens are EOS-specific (auto-negotiation and forced-speed keyword forms) — confirm exact accepted values per platform against `show interfaces <if> capabilities` before hard-coding a GUI dropdown. |

## VLAN configuration

| Task | Command(s) | Notes |
|---|---|---|
| Create a VLAN | `vlan <id>` then `name <text>` | Identical structure to IOS `vlan <id>` / `name` in vlan-config submode. |
| Set access mode | `switchport mode access` | Identical to IOS. |
| Assign access VLAN | `switchport access vlan <id>` | Identical to IOS. Maps to `Q-BRIDGE-MIB::dot1qPvid` conceptually (see [standard-mibs.md](../../00-architecture/standard-mibs.md)); no confirmed EOS SNMP write path for this object — see [gui-cli-snmp-mapping.md](gui-cli-snmp-mapping.md). |
| Set trunk mode | `switchport mode trunk` | Identical to IOS. |
| Restrict trunk VLANs | `switchport trunk allowed vlan <list>` (also `add`/`remove`/`except` sub-forms) | Identical verb/structure to IOS. Default (no command issued) is **all VLANs permitted** on a trunk port — same IOS default behavior. |

## Spanning tree

| Task | Command(s) | Notes |
|---|---|---|
| Set global STP mode | `spanning-tree mode mstp` \| `rstp` \| `rapid-pvst` | Verb identical to IOS; EOS's mode keyword set (`mstp`/`rstp`/`rapid-pvst`) matches IOS naming. |
| Set bridge priority | `spanning-tree priority <0-61440, mult. of 4096>` (global); `spanning-tree vlan-id <vlan> priority <n>` (per-VLAN, rapid-pvst); `spanning-tree mst <instance> priority <n>` (per-MST-instance) | Identical structure to IOS's per-VLAN/per-instance priority commands. |
| Force root bridge | `spanning-tree root primary` \| `spanning-tree root secondary` | Identical to IOS. |
| Enable PortFast on a port | `spanning-tree portfast` (interface config mode) | Identical to IOS `spanning-tree portfast`. EOS additionally supports `spanning-tree portfast auto`, `spanning-tree portfast network`, `spanning-tree portfast edge` (edge/network port-type framing closer to 802.1w/RSTP terminology) — these extra forms are EOS-specific refinements, not present verbatim in classic IOS. |
| Per-port path cost | `spanning-tree cost <1-200000000>`; `spanning-tree vlan-id <vlan> cost <n>`; `spanning-tree mst <instance> cost <n>` | Identical structure to IOS. |
| Per-port priority | `spanning-tree port-priority <0-240, mult. of 16>`; `spanning-tree mst <instance> port-priority <n>` | Identical to IOS. |
| Read back STP state | `show spanning-tree` | Identical to IOS. |

## LACP / Port-Channel

| Task | Command(s) | Notes |
|---|---|---|
| Create port-channel interface | `interface port-channel <n>` | Identical structure to IOS; creating a channel-group with a matching ID also implicitly creates the port-channel interface. |
| Add a physical interface to a channel, static (no LACP) | `channel-group <n> mode on` | Identical to IOS's `channel-group <n> mode on` (static EtherChannel, no LACP negotiation). |
| Add a physical interface to a channel, LACP active | `channel-group <n> mode active` | Identical to IOS. Actor sends LACPDUs; will form with a partner in `active` or `passive`. |
| Add a physical interface to a channel, LACP passive | `channel-group <n> mode passive` | Identical to IOS. Actor only responds to LACPDUs, does not initiate. |
| Read back port-channel/LACP state | `show port-channel`, `show port-channel dense`, `show lacp aggregates`, `show lacp interface` | `show port-channel`/`show lacp *` verb families match IOS's `show etherchannel`/`show lacp` conceptually, but exact command names diverge from IOS (`show etherchannel summary` on IOS vs `show port-channel` on EOS) — treat this as a **divergence point** for the cli-syntax-matrix, not identical text. |

## Port security

| Task | Command(s) | Notes |
|---|---|---|
| Enable MAC-based port security | `switchport port-security` (interface config mode) | Verb identical to IOS. Default max secure MAC addresses per port is 1 (matches IOS default). Violation action: port goes to `errdisabled` state — same terminology as IOS. |
| Set max secure MAC addresses | `switchport port-security mac-address maximum <n>` | Identical structure to IOS's `switchport port-security maximum <n>`, though the exact EOS token order (`mac-address maximum`) was found via secondary/community sources this session, not a directly fetched primary-doc excerpt — **verify exact syntax against a live device or the current release's user manual before relying on it in generated GUI-to-CLI mappings.** |
| Persistent secure-MAC across reboot/flap | Enabled by default (documented as "Persistent PortSec-Protect"); can be disabled | Exact disable command syntax not confirmed this session. |

## SNMP agent configuration

| Task | Command(s) | Notes |
|---|---|---|
| Configure a community string | `snmp-server community <string> [view <view-name>] [ro\|rw]` | Verb identical to IOS's `snmp-server community`. |
| Configure a trap/inform destination | `snmp-server host <ip-address> [informs\|traps] version {1\|2c\|3} <community-or-user> ...` | Structure matches IOS's `snmp-server host` family. |
| Configure an SNMPv3 user | `snmp-server user <user-name> <group-name> [remote <addr>] {v1\|v2c\|v3} [auth {md5\|sha} <passphrase> [priv {des\|aes} <passphrase>]]` | Structure matches IOS's `snmp-server user` command; exact keyword ordering should be confirmed against the current release's command reference before generating this line verbatim for a GUI action, since SNMPv3 auth/priv keyword ordering is a common point of inter-vendor divergence even where the verb is shared. |
| Configure an SNMPv3 group (view mapping) | `snmp-server group <group-name> {v1\|v2c\|v3 {auth\|noauth\|priv}} [context <name>] [read <view-name>] [write <view-name>]` | Matches IOS structure. |
| Define/restrict a MIB view | `snmp-server view <view-name> <mib-family> {include\|exclude}` | **EOS-distinctive access-control mechanism** — see [mib-reference.md](mib-reference.md). Not a command SignalScope will typically need to *generate*, but relevant to interpreting why a given community/user may have narrower object visibility than expected. |
| Read back configured views | `show snmp view` | Output columns: view name, MIB object/family, inclusion level (`included`/`excluded`). |

## Paging control

| Task | Command(s) | Notes |
|---|---|---|
| Disable output paging for the session | `terminal length 0` | **Identical text to IOS.** Standard first command SignalScope's SSH session manager should send at session start (see [connectivity-methods.md](../../00-architecture/connectivity-methods.md)), echoed into the terminal per the "no hidden setup commands" principle. |

## Save / persist configuration

| Task | Command(s) | Notes |
|---|---|---|
| Persist running-config to startup-config | `write` or `write memory` | EOS shorthand form; functionally identical outcome to the IOS form below. `write` alone is accepted on EOS (IOS historically required `write memory` or `copy run start`, though modern IOS also accepts bare `write`). |
| Persist running-config to startup-config (explicit form) | `copy running-config startup-config` | Identical text to IOS. Prefer this as the canonical "Save" command SignalScope echoes, since it's unambiguous across both vendors' terminal transcripts. |

## Show commands for read-back

| Task | Command(s) | Notes |
|---|---|---|
| Interface summary/status | `show interfaces status` | Identical to IOS. |
| VLAN table | `show vlan` | Identical to IOS. |
| Spanning-tree state | `show spanning-tree` | Identical to IOS. |
| Running-config, interface section only | `show running-config section interface` | EOS supports the `section <regex>` filter form; IOS traditionally uses `show running-config | section interface` (pipe-to-section) rather than the same `section` sub-keyword directly after `show running-config` — treat as a **likely divergence** worth confirming per exact IOS/EOS release rather than assuming identical text. |
| Port-channel/LACP state | `show port-channel`, `show lacp interface` | See LACP section above — command family name diverges from IOS's `show etherchannel`. |
