# ExtremeXOS (EXOS) — CLI Reference

Curated command tables. Port naming below uses EXOS's native `<slot>:<port>` form on modular/stackable chassis (e.g. `1:3` = slot/unit 1, port 3) or bare port numbers on fixed 1-slot switches; substitute the platform's actual numbering.

## No persistent interface/config sub-mode — read this before the tables below

Unlike Cisco/Arista/Huawei/Aruba, **EXOS has no `interface X` (or `configure vlan X` as an entered-and-stayed-in) sub-mode that subsequent bare commands implicitly target.** Every row below is a **complete, self-contained command line** — the port list, VLAN name, or STPD name is always an explicit argument, not implied by a prior "enter context" command. There is also no separate unprivileged/privileged EXEC split to model. Practically, this means SignalScope's session-state tracker does not need a "current mode" concept for EXOS the way it does for every other vendor documented so far — each emitted command can be sent independently without first checking/establishing context. This is worth carrying into the vendor comparison matrix as a structural outlier.

## Interface admin state & description

| Task | Command(s) | Notes |
|---|---|---|
| Enable (bring up) a port | `enable port 1:3` | Sets `IF-MIB::ifAdminStatus` to `up(1)`. Accepts a port list/range: `enable port 1:1-1:4`. |
| Disable (bring down) a port | `disable port 1:3` | Sets `ifAdminStatus` to `down(2)`. Link is physically brought down. |
| Set a short display label | `configure ports 1:3 display-string uplink-1` | Limited to a short length (documentation states values around 15-20 characters depending on release); shown in compact `show ports` table output. |
| Set a longer free-text description | `configure ports 1:3 description-string "uplink to core-sw1 gi0/24"` | Up to 255 characters in modern releases; quote if it contains spaces/punctuation. Maps to `IF-MIB::ifAlias` (`ifXTable`). This is the closer analog to Cisco's `description` command; `display-string` is EXOS-specific and shorter. |
| Set speed/duplex | `configure ports 1:3 auto off speed 1000 duplex full` (example) | Syntax varies by port/media type (copper vs. SFP); `auto on` re-enables autonegotiation. |

Sources: [Enabling and Disabling Switch Ports](https://documentation.extremenetworks.com/exos_32.1/GUID-41B51D50-7B27-432A-A7B7-D9DB47447A48.shtml), [`enable port`](https://documentation.extremenetworks.com/exos_commands_31.5/GUID-998EFFCC-E9AD-40B0-95D4-2A7A191E1783.shtml), [`disable port`](https://documentation.extremenetworks.com/exos_commands_31.7/GUID-D3367FFD-C9B1-4F0E-A360-26D2C8136E92.shtml), [`configure ports display-string`](https://documentation.extremenetworks.com/exos_32.6.1/GUID-F0B1A25D-FDC5-461F-A1B8-CD49E82ABCEF.shtml).

## VLAN configuration

| Task | Command(s) | Notes |
|---|---|---|
| Create a VLAN | `create vlan accounting` | Newly created VLAN has no member ports, is untagged by default, and gets an auto-assigned tag counting down from 4094 unless a tag is given. |
| Create a VLAN with an explicit 802.1Q tag | `create vlan accounting tag 100` | Combines create + tag assignment in one line. |
| Assign/change a VLAN's 802.1Q tag on an existing VLAN | `configure vlan accounting tag 100` | Separate command if the VLAN was created without a tag or the tag needs changing. |
| Add tagged ports to a VLAN | `configure vlan accounting add ports 1:1, 1:2, 1:3 tagged` | The `tagged`/`untagged` keyword is mandatory-in-practice per port-list — omitting it makes EXOS default to attempting untagged, which can silently displace an existing untagged VLAN on that port depending on the `auto-move` setting. Always specify explicitly. |
| Add untagged (access) ports to a VLAN | `configure vlan accounting add ports 1:4, 1:5 untagged` | An untagged port can only belong to **one** untagged VLAN at a time; if it's currently untagged in the default VLAN, it must be removed from that VLAN first (untagged membership is exclusive, unlike Cisco's single-PVID-plus-optional-trunk model — EXOS ties it directly to VLAN egress-port membership). |
| Remove ports from a VLAN | `configure vlan accounting delete ports 1:1, 1:2` | |
| Delete a VLAN | `delete vlan accounting` | |
| Read back VLAN-to-port mapping | `show vlan` / `show vlan accounting` / `show ports 1:3 vlan` | See "show commands" section. |

Sources: [`create vlan`](https://documentation.extremenetworks.com/switchengine_commands_32.5/GUID-A21F22AC-0A99-4D60-AB09-D7246A3B3FD8.shtml), [`configure vlan tag`](https://documentation.extremenetworks.com/exos_commands_32.1/GUID-95823723-90A6-4D81-8EBE-E7A4764D2A4A.shtml), [Adding and Removing Ports from a VLAN](https://documentation.extremenetworks.com/exos_22.5/GUID-B9870B2A-731B-4D0C-A22C-CCD3E774F1B1.shtml), [Understanding EXOS VLANs and tagged/untagged ports](https://gtacknowledge.extremenetworks.com/articles/How_To/Understanding-EXOS-VLANS-and-tagged-and-untagged-ports).

## Spanning tree — STPD (Spanning Tree Domain)

EXOS's spanning-tree model is organized around the **STPD (Spanning Tree Domain)**, a named spanning-tree instance that one or more VLANs are associated with — conceptually closer to Cisco MST's "instance groups multiple VLANs" than to per-VLAN PVST, though EXOS also supports per-VLAN (`dot1d`) and per-VLAN-with-shared-topology (`emistp`) encapsulation modes on a per-STPD-membership basis. A switch can run multiple STPDs simultaneously, each independently electing its own root and covering a distinct set of VLANs/ports — this is the unit SignalScope should treat as "one spanning-tree instance" for EXOS, analogous to a Cisco MST instance or a PVST per-VLAN tree, rather than assuming one global STP process per switch.

| Task | Command(s) | Notes |
|---|---|---|
| Create an STPD | `create stpd stpd1` | Named domain; the default switch config typically ships with one STPD (commonly named `s0`) already associated with the default VLAN. |
| Add a VLAN (and its ports) to an STPD | `configure stpd stpd1 add vlan marketing ports all` | Adds all of `marketing`'s member ports into `stpd1`'s topology. `{dot1d \| emistp \| pvst-plus}` mode keyword optional at the end to select encapsulation. |
| Add specific ports of a VLAN to an STPD, tagged, with encapsulation mode | `configure vlan marketing add ports 1:2, 2:3 tagged stpd stpd1 emistp` | This is the combined "add VLAN ports and also wire them into an STPD" form issued via the `configure vlan ... add ports` command rather than the `configure stpd` command — both command families can express STPD port membership. |
| Enable an STPD | `enable stpd stpd1` | STPDs are created disabled; must be explicitly enabled to run. |
| Disable an STPD | `disable stpd stpd1` | |
| Set STPD to a specific protocol version | `configure stpd stpd1 mode {dot1d \| dot1w \| mstp}` | `dot1d` = classic STP, `dot1w` = RSTP, `mstp` = Multiple Spanning Tree — analogous to Cisco's `spanning-tree mode` global switch, but scoped per-STPD rather than global-per-switch. |
| Set per-port STPD path cost / priority | `configure stpd stpd1 ports cost 1:3 4` / `configure stpd stpd1 ports priority 1:3 16` | |
| Read back STPD state | `show stpd` / `show stpd stpd1` / `show stpd stpd1 ports` | See "show commands" section. |

Sources: [Configuring STP on the Switch](https://documentation.extremenetworks.com/exos_30.1/GUID-2CF3B061-9821-45C8-92A7-9CA73BB929A3.shtml), [`configure stpd add vlan`](https://documentation.extremenetworks.com/switchengine_commands_32.4/GUID-3251352C-B502-42E1-9AAB-D09329F7FD95.shtml), [`configure vlan add ports stpd`](https://documentation.extremenetworks.com/exos_commands_32.2/GUID-4F0B8874-CE40-4A4E-80AE-7457FB66E69A.shtml).

## LACP / sharing (link aggregation)

EXOS calls link aggregation **"sharing"** — there is no separate `Port-channel`/`channel-group`-style logical-interface-creation step the way Cisco/Arista have; the aggregation group *is* one of its own member ports, designated as the group's "logical"/master port.

| Task | Command(s) | Notes |
|---|---|---|
| Create a LACP-negotiated sharing group | `enable sharing 1:1 grouping 1:1-1:4 algorithm address-based L2 lacp` | Port `1:1` becomes the master/logical port representing the whole group (and the LAG's group ID for LACP purposes); `grouping` lists all member ports including the master. `algorithm` selects the load-distribution hash (`address-based {L2 \| L3 \| L3_L4 \| custom}` or `port-based`). |
| Create a static (non-LACP) sharing group | `enable sharing 1:1 grouping 1:1-1:4 algorithm address-based L2` | Omit the trailing `lacp` keyword for a statically-configured aggregate with no LACP negotiation — both ends must agree out-of-band, same caveat as Cisco `channel-group ... mode on`. |
| Disable/remove a sharing group | `disable sharing 1:1` | Removes the aggregation; member ports (other than the still-existing master port `1:1`) revert to independent ports. |
| Read back sharing/LACP state | `show ports sharing` / `show lacp` | See "show commands" section. |

Sources: [Configuring LACP — EXOS User Guide](https://documentation.extremenetworks.com/exos_22.5/GUID-F5744EA0-C4D9-4DE8-ACCE-96D3D1EB33A6.shtml), [How To: Configure a Sharing Group (LAG) with LACP](https://extreme-networks.my.site.com/ExtrArticleDetail?an=000082730), [`enable sharing grouping`](https://documentation.extremenetworks.com/exos_22.3/GUID-C2819994-B8D2-4E9F-99C7-3471777DC258.shtml).

## Port security (MAC locking / per-port learn limits)

EXOS's nearest equivalent to Cisco `switchport port-security` is called **MAC locking**, plus a separate simpler per-port-per-VLAN learn-limit mechanism.

| Task | Command(s) | Notes |
|---|---|---|
| Enable MAC locking globally | `enable mac-locking` | Prerequisite before per-port locking configuration takes effect. |
| Lock a port to a fixed, manually-specified MAC set | `configure mac-locking ports 1:3 static limit-learning 4` | "Static" locking: administrator supplies the allowed MAC list; max locked MACs per port is documented as 64. |
| Lock a port to whichever MACs it sees first (dynamic/"first-arrival") | `configure mac-locking ports 1:3 first-arrival limit-learning 4` | Switch learns and locks the first N MACs seen, then blocks new ones — closer in spirit to Cisco's "sticky" port-security learning. |
| Enable locking enforcement on specific ports | `enable mac-locking port 1:3` | Locking must be enabled both globally and per-port. |
| Simpler per-VLAN learn-limit (no full lock semantics) | `configure ports 1:3 vlan accounting limit-learning 10 action blackhole` | A lighter-weight alternative: caps MAC-table entries learned on a given port within a given VLAN; `action {blackhole \| stop-learning}` controls what happens once the limit is hit. `lock-learning`/`unlock-learning`/`unlimited-learning` are related sub-keywords. |
| Read back MAC-locking state | `show mac-locking` / `show mac-locking ports 1:3` | See "show commands" section. |

Sources: [Configuring MAC Locking — EXOS User Guide](https://documentation.extremenetworks.com/exos_22.4/GUID-AB4DE02D-F3EB-44BA-B4DC-54039202C351.shtml), [MAC Locking Configuration Example](https://documentation.extremenetworks.com/exos_31.5/GUID-BF138684-592E-46B3-8F3D-D18442527AA7.shtml), [`configure mac-locking ports ... first-arrival limit-learning`](https://documentation.extremenetworks.com/switchengine_commands_32.3/GUID-C4F42D08-2F0D-488C-B990-1273DD53B25B.shtml), [How to limit the number of MAC addresses learned on a port](https://extreme-networks.my.site.com/ExtrArticleDetail?an=000082688).

## SNMP agent configuration (on the device itself)

| Task | Command(s) | Notes |
|---|---|---|
| Add a read-only v1/v2c community | `configure snmp add community readonly public` | Cleartext, applies to both v1 and v2c queries using this string. Up to 16 read-only + 16 read/write community strings can exist including defaults. |
| Add a read-write v1/v2c community | `configure snmp add community readwrite private` | Enables SNMP SET for whatever is actually supported (see `overview.md`/`mib-reference.md`). |
| Add a trap receiver (v1/v2c) | `configure snmp add trapreceiver 10.0.0.5 community public` | Optional `port`, `from <src-ip>`, `vr <vr-name>`, `mode <trap-mode>` qualifiers. Default trap version is v2c; receivers default to "enhanced" mode. Community must already exist via the command above. |
| Add an SNMPv3 user | `configure snmpv3 add user v3admin authentication sha AuthPass123 privacy aes PrivPass123` | `authentication {md5 \| sha}`, `privacy {des \| 3des \| aes {128\|192\|256}}` — SHA + AES128/192/256 is the modern-recommended pairing. `{volatile}` keyword available for non-persistent (test) users. |
| Add an SNMPv3 group (ties users to an access policy) | `configure snmpv3 add group v3group user v3admin sec-model usm` | |
| Add SNMPv3 access (security level + MIB views) | `configure snmpv3 add access v3group sec-model usm sec-level priv read-view defaultAdminView write-view defaultAdminView notify-view defaultAdminView` | `sec-level {noauth \| auth \| priv}`; ties the group to specific read/write/notify **MIB views** (see next row and `mib-reference.md`). |
| Define/restrict a custom MIB view (access control) | `configure snmpv3 add mib-view myView subtree 1.3.6.1.2.1.2 type included` | The distinctive EXOS SNMPv3 access-control primitive — restricts a view to a specific OID subtree (with optional mask), `type included`/`excluded`, repeatable to build up an arbitrarily precise view. See `mib-reference.md` for the fuller group/view/access relationship. |
| Add an SNMPv3 notification target | `configure snmpv3 add target-addr ...` / `configure snmpv3 add notify ...` / `configure snmpv3 add target-params ...` | Multi-command chain analogous to `snmp-server host` + `snmp-server enable traps` on Cisco, but SNMPv3's target/notify/params model is split into three separate objects (target address, notify filter, and target params referencing a user/security level) per the standard USM notification architecture — not collapsed into one line the way v1/v2c trap receivers are. |
| Enable/disable SNMP access entirely, or per-version | `enable snmp access` / `disable snmp access` (and documented version-scoped variants) | Lets v1/v2c be disabled while v3 remains active, or vice versa — verify exact per-version sub-keywords against the target release's command reference before relying on this for a specific device. |

Sources: [`configure snmp add community`](https://documentation.extremenetworks.com/4000%20Series%20v33.2.1%20User%20Guide/Switch_Operating_Systems/Switch_Engine/Command_References/configure_snmp_add_community.shtml), [`configure snmp add trapreceiver`](https://documentation.extremenetworks.com/exos_commands_31.3/GUID-13C1CA93-00A6-4E98-8381-529CDDC1E246.shtml), [`configure snmpv3 add user`](https://documentation.extremenetworks.com/exos_commands_31.6/GUID-0596F408-87A2-4C03-A6BE-421FD660E38E.shtml), [Setting SNMPv3 MIB Access Control](https://documentation.extremenetworks.com/exos_30.2/GUID-D57AEEBA-8942-4460-9704-01413FD52EB0.shtml), [SNMPv3 — EXOS User Guide](https://documentation.extremenetworks.com/exos_31.1/GUID-79586155-3EA1-4AED-AE1D-1E41879504E5.shtml), [How to configure SNMPv3 securely on Extreme XOS (independent walkthrough, cross-referenced for the group/access/mib-view chain)](https://robert.penz.name/877/how-to-configure-snmpv3-securely-on-extreme-networks-xos/), [Enabling/Disabling SNMPv1/v2/v3](https://www.plixer.com/blog/extreme-networks-enabling-and-disabling-snmpv1-snmpv2-and-snmpv3/).

## Paging control (automation-session setup)

| Task | Command(s) | Notes |
|---|---|---|
| Disable `--More--`-style pausing for the current session | `disable clipaging` | No arguments. Per Extreme's own documentation this is **session-scoped only** — it cannot be saved into the persistent config and must be re-sent on every new CLI session, same operational shape as Cisco's `terminal length 0`. Per `connectivity-methods.md`, SignalScope should send this automatically at session start and echo it in the terminal like any other command. |
| Re-enable paging | `enable clipaging` | |

Sources: [Disable Clipaging — Command Reference](https://documentation.extremenetworks.com/exos_commands_22.6/GUID-6C44D895-A635-4A8F-93C9-B5CC382BC655.shtml).

## Save / persist configuration

| Task | Command(s) | Notes |
|---|---|---|
| Save running config to the currently-selected config slot | `save configuration` | Saves to whichever of `primary`/`secondary` (or a named `.cfg` file) is currently active/selected-for-boot. |
| Save to a specific named slot | `save configuration primary` / `save configuration secondary` | EXOS keeps two named configs by default (`primary.cfg`, `secondary.cfg`); a switch boots from whichever is currently selected (`use configuration primary`/`secondary`, a separate command, controls that selection). |
| Save to a new/existing arbitrary filename | `save configuration new-config` / `save configuration existing-config` | For custom-named `.cfg` files beyond the two defaults. |

Sources: [`save configuration` — Command Reference](https://documentation.extremenetworks.com/exos_commands_32.1/GUID-BB723584-7FCF-4B29-90B3-E22C4290D4F1.shtml), [Save the Configuration — User Guide](https://documentation.extremenetworks.com/exos_31.4/GUID-96FE907C-FC5A-4E75-B632-002C6965139F.shtml), [Managing the Configuration File](https://documentation.extremenetworks.com/exos_32.6.1/GUID-948439E8-14A8-4194-9A66-DD8D7D1C927A.shtml).

## Show commands for read-back / polling

| Task | Command(s) | Notes |
|---|---|---|
| Per-port summary (admin/oper/speed/VLAN) | `show ports` / `show ports 1:3 information` | Compact table; good source for GUI port-grid read-back. |
| Full port detail | `show ports 1:3 information detail` | |
| VLAN-to-port membership | `show vlan` / `show vlan accounting` | `show vlan` alone lists all VLANs with tag, protocol, STPD association, and port counts; add a VLAN name for that VLAN's full port list. |
| Per-port VLAN/tag membership (port-centric view) | `show ports 1:3 vlan` / `show port vid` | Lists untagged/tagged VID membership per port rather than per-VLAN. |
| STPD state | `show stpd` / `show stpd stpd1 detail` / `show stpd stpd1 ports` | |
| Sharing/LACP state | `show ports sharing` / `show lacp` | |
| MAC-locking state | `show mac-locking` | |
| SNMP config as currently configured | `show snmp community` / `show snmpv3 user` / `show snmpv3 mib-view` / `show snmpv3 access` | Useful for verifying SNMP config changes landed as expected — mirrors the "read own config back" pattern from other vendors' `show running-config | section snmp`. |
| MAC address table | `show fdb` | EXOS's forwarding-database show command; backs the same data as `BRIDGE-MIB::dot1dTpFdbTable`. |

Sources: [`show vlan`](https://documentation.extremenetworks.com/exos_32.7.1/GUID-7B100813-EABB-4A9E-BAAC-D6ADB8FAA36D.shtml), [What commands show port vlan and tagging information](https://extreme-networks.my.site.com/ExtrArticleDetail?an=000089317).
