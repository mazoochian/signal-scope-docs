# Junos CLI Reference — EX/QFX Series Switches

Curated, not exhaustive. All `set`/`delete` commands below operate on the **candidate configuration only** — see [overview.md](overview.md) for the commit model. Every config-mode task table ends with an explicit commit step; do not treat the `set` lines alone as "applied." Sourced against [CLI Explorer](https://apps.juniper.net/cli-explorer/) and the specific doc pages cited per section.

## Entering configuration mode, hierarchy navigation, committing

| Task | Command(s) | Notes |
|---|---|---|
| Enter config mode | `configure` | Shared/non-exclusive — other users can also edit candidate config concurrently (merged at commit, with warnings on conflict). |
| Enter config mode exclusively | `configure exclusive` | Locks the candidate config against other editors until you exit; recommended for SignalScope-driven sessions to avoid merge surprises with a concurrent human session. |
| Enter config mode, private copy | `configure private` | Each user gets an isolated candidate copy; changes merge into the shared candidate only at `commit`. |
| Navigate into a hierarchy | `edit interfaces ge-0/0/1` | Changes the config-mode prompt context, e.g. `[edit interfaces ge-0/0/1]`; subsequent `set`/`delete` are relative to this level. |
| Move up one level | `up` | |
| Return to top of config tree | `top` | |
| Preview candidate changes | `show \| compare` or `show` (in current context) | `show \| compare` (a.k.a. `show \| compare rollback 0`) diffs candidate vs. active config — the Junos equivalent of a pre-commit dry-run diff. |
| Validate without activating | `commit check` | Syntax/semantic validation only; does **not** start a confirm timer, does **not** touch rollback history. |
| Commit (activate immediately, permanent) | `commit` | Takes effect now; becomes the new rollback-0 baseline. |
| Commit with auto-rollback safety | `commit confirmed 5` | Activates now, but auto-reverts to the prior config if a follow-up `commit`/`commit check` isn't issued within 5 minutes (default window is 10 minutes if no value given). **Recommended default for any change touching interface/management reachability.** |
| Confirm a pending `commit confirmed` | `commit` | A plain `commit` inside the confirm window cancels the pending auto-rollback and finalizes the change. |
| Schedule a future/deferred commit | `commit at "2026-07-12 02:00:00"` or `commit at reboot` | Stages activation without committing now. |
| View commit/rollback history | `show system commit` | Lists rollback slots 0–49 with timestamp, user, and log message. |
| Roll back to a previous commit | `rollback 1` then `commit` | `rollback <n>` only loads history slot `n` into the candidate buffer — it still requires a subsequent `commit` to take effect. |
| Discard uncommitted candidate edits | `rollback 0` | Discards edits not yet committed; does not affect device state (nothing was live anyway). |
| Exit config mode | `exit` (warns on uncommitted changes) or `exit configuration-mode` | |

Sources: [`commit` command reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/commit.html), [Commit the Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html), [`rollback` command reference](https://www.juniper.net/documentation/en_US/junos/topics/reference/command-summary/rollback.html).

## Interface admin state & description

| Task | Command(s) | Notes |
|---|---|---|
| Disable a port | `set interfaces ge-0/0/1 disable` then `commit` | Candidate-only until commit — see above. |
| Re-enable a port | `delete interfaces ge-0/0/1 disable` then `commit` | Junos has no positive "enable" statement; re-enabling means *deleting* the `disable` statement. |
| Set port description | `set interfaces ge-0/0/1 description "Uplink to core-sw1 Gi1/0/24"` then `commit` | Maps to `IF-MIB::ifAlias` on read-back. |
| Set interface speed/duplex | `set interfaces ge-0/0/1 ether-options speed 1g` / `set interfaces ge-0/0/1 ether-options link-mode full-duplex` | Under `ether-options` (copper GbE) or `gigether-options`/`fastether-options` depending on media type. |

Source: [Configuring Gigabit Ethernet Interfaces (CLI Procedure)](https://www.juniper.net/documentation/en_US/junos/topics/task/configuration/ex-series-gigabit-interfaces-cli.html).

## VLAN configuration

| Task | Command(s) | Notes |
|---|---|---|
| Create a VLAN | `set vlans staff vlan-id 10` then `commit` | `staff` is the VLAN name (Junos VLANs are named, not just numbered — the name is the primary config-tree key). |
| Assign an access port to a VLAN | `set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members staff` then `commit` | Config lives under the logical unit (`unit 0`), one level below the physical interface — distinct from Cisco's flat `switchport access vlan 10`. |
| Set interface mode to access (explicit) | `set interfaces ge-0/0/1 unit 0 family ethernet-switching interface-mode access` | Often implicit with a single VLAN member + no trunk statement, but recommended explicitly for clarity/idempotency. |
| Set interface mode to trunk | `set interfaces ge-0/0/1 unit 0 family ethernet-switching interface-mode trunk` then `commit` | |
| Assign trunk VLAN members | `set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members [ staff guests ]` | Bracketed list syntax for multiple values at one statement; can also use a VLAN range, e.g. `vlan members 10-20`. |
| Set native/untagged VLAN on a trunk | `set interfaces ge-0/0/1 unit 0 family ethernet-switching native-vlan-id 1` | |
| Read back VLAN membership | `show vlans` / `show vlans staff detail` | |
| Read back interface-level config | `show configuration interfaces ge-0/0/1` | Shows exactly the committed (active) config subtree — the literal source of truth for GUI read-back. |

Sources: [Bridging and VLANs](https://www.juniper.net/documentation/us/en/software/junos/multicast-l2/topics/topic-map/bridging-and-vlans.html), [Configuring VLANs for EX Series Switches (CLI Procedure)](https://www.juniper.net/documentation/en_US/junos12.3/topics/task/configuration/bridging-vlans-ex-series-cli.html).

## Spanning tree (RSTP/MSTP)

| Task | Command(s) | Notes |
|---|---|---|
| Enable RSTP globally (default protocol) | `set protocols rstp` | RSTP is Junos's default STP mode on EX switches if `protocols (rstp\|mstp\|vstp)` is configured at all; there is no separate "enable" toggle beyond configuring the protocol block. |
| Mark a port as an edge port (PortFast equivalent) | `set protocols rstp interface ge-0/0/1 edge` then `commit` | Same `edge` statement exists under `protocols mstp interface <name> edge`. |
| Disable STP on an interface | `set protocols rstp interface ge-0/0/1 disable` | |
| Configure MSTP region/instance | `set protocols mstp configuration-name REGION1`, `set protocols mstp revision-level 1`, `set protocols mstp msti 1 vlan 10` | MSTP requires explicit region name + revision to match across switches, unlike Cisco's simpler `spanning-tree mst configuration`. |
| Block BPDUs on edge ports (safety) | `set protocols rstp bpdu-block-on-edge` | Converts an edge port to blocking (not just non-edge) if a BPDU arrives — a safety net against accidental loops from an edge port. |
| Read back STP state | `show spanning-tree interface` | Per-port role/state (root/designated/blocking/forwarding). |

Sources: [Configuring RSTP](https://www.juniper.net/documentation/us/en/software/junos/stp-l2/topics/topic-map/spanning-tree-configuring-rstp.html), [Configuring MSTP](https://www.juniper.net/documentation/us/en/software/junos/stp-l2/topics/topic-map/spanning-tree-configuring-mstp.html), [`edge` (Spanning Trees) statement](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/edge-spanning-trees-ex-series.html), [`bpdu-block-on-edge` statement](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/bpdu-block-on-edge-edit-protocols-stp.html).

## LACP / Aggregated Ethernet

| Task | Command(s) | Notes |
|---|---|---|
| Reserve `ae` interfaces (required first step) | `set chassis aggregated-devices ethernet device-count 5` | Must be set **before** member interfaces can reference `ae0`..`ae4`; this is a chassis-wide resource allocation, not part of the `ae` interface config itself. |
| Add a member link to an aggregate | `set interfaces ge-0/0/1 ether-options 802.3ad ae0` | Statement name is `ether-options` for GbE copper, `gigether-options` for some Gigabit fiber SKUs — confirm against the specific PIC/interface type; both accept the `802.3ad ae<N>` sub-statement. |
| Enable LACP, active mode | `set interfaces ae0 aggregated-ether-options lacp active` then `commit` | `active` initiates LACP; `passive` only responds — use `active` unless the peer requires passive. |
| Set LACP periodic interval | `set interfaces ae0 aggregated-ether-options lacp periodic fast` | `fast` (1s) vs `slow` (30s, default) PDU interval. |
| Assign the `ae` interface to a VLAN | `set interfaces ae0 unit 0 family ethernet-switching vlan members staff` | Same Layer 2 config surface as a physical port, applied to the aggregate instead. |
| Read back aggregate/member state | `show interfaces ae0 terse`, `show lacp interfaces ae0` | `show lacp interfaces` shows per-member actor/partner LACP state (detached/attached/collecting/distributing). |

Sources: [Aggregated Ethernet Interfaces Overview](https://www.juniper.net/documentation/us/en/software/junos/interfaces-ethernet/topics/topic-map/aggregated-ethernet-interfaces-lacp-configure.html), [Junos Aggregated Ethernet Example](https://www.networkcuriosity.com/junos-aggregated-ethernet-example/).

## Port security

Junos's EX-series port-security feature set lives under a dedicated hierarchy, not folded into interface config directly — introduced in Junos 9.0 for EX switches:

| Task | Command(s) | Notes |
|---|---|---|
| Enable secure access port features on an interface | `set ethernet-switching-options secure-access-port interface ge-0/0/1 mac-limit 4 action drop` then `commit` | `mac-limit` caps learned MAC addresses on the port; `action` is `drop`, `log`, `shutdown`, or `none`. |
| Enable DHCP snooping on a VLAN | `set ethernet-switching-options secure-access-port vlan staff dhcp-snooping` | Configured per-VLAN, not per-port. |
| Mark an uplink as DHCP-trusted | `set ethernet-switching-options secure-access-port interface ge-0/0/24 dhcp-trusted` | Untrusted (access/edge) ports are the default; trust must be explicitly granted to uplinks toward the real DHCP server. |
| Enable dynamic ARP inspection on a VLAN | `set ethernet-switching-options secure-access-port vlan staff arp-inspection` | Relies on the DHCP snooping binding table. |
| Enable IP source guard on a VLAN | `set ethernet-switching-options secure-access-port vlan staff ip-source-guard` | |
| Statically allow a MAC on a port | `set ethernet-switching-options secure-access-port interface ge-0/0/1 allowed-mac 00:11:22:33:44:55` | |
| Read back port-security config | `show configuration ethernet-switching-options secure-access-port` | |
| Read back MAC-limit/violation state | `show ethernet-switching interfaces ge-0/0/1 detail` | |

Note: newer Junos Evolved / QFX ELS-style platforms are migrating some of this toward `set protocols` equivalents in places — verify the exact hierarchy against [CLI Explorer](https://apps.juniper.net/cli-explorer/) for the specific EX/QFX model and Junos release in use before assuming `ethernet-switching-options` is universal across every current SKU.

Sources: [Overview of Port Security](https://www.juniper.net/documentation/us/en/software/junos/security-services/topics/topic-map/overview-port-security.html), [`secure-access-port` statement](https://www.juniper.net/documentation/us/en/software/junos/fcoe-ex/security-services/topics/ref/statement/secure-access-port-port-security.html), [Example: Configuring MAC Limiting](https://www.juniper.net/documentation/us/en/software/junos/security-services/topics/topic-map/example-configuring-port-limiting.html).

## SNMP agent configuration

| Task | Command(s) | Notes |
|---|---|---|
| Configure a read-only v1/v2c community | `set snmp community public authorization read-only` then `commit` | Default view (all supported MIB objects) at read-only scope. |
| Restrict a community to specific client IPs | `set snmp community public clients 10.0.0.0/24` | |
| Configure a read-write community | `set snmp community <name> authorization read-write` plus a `view` statement enumerating writable OIDs | **Per Junos's own SNMP FAQ, `ifAdminStatus` cannot be SET even with read-write configured** — see [overview.md](overview.md) and [mib-reference.md](mib-reference.md). Do not assume this grants general config-write capability. |
| Configure a trap target group (v1/v2c) | `set snmp trap-group ops-traps version v2 targets 10.0.0.50` then `commit` | `destination-port` optional (defaults to UDP 162). |
| Configure SNMPv3 USM user (SHA auth, AES128 priv) | `set snmp v3 usm local-engine user monitor authentication-sha authentication-password "<passphrase>"` then `set snmp v3 usm local-engine user monitor privacy-aes128 privacy-password "<passphrase>"` then `commit` | Passwords are converted to localized keys against the engine ID at commit time. |
| Configure SNMPv3 trap notification target | `set snmp v3 vacm security-to-group security-model usm security-name monitor group ops-group`, `set snmp v3 target-address ...`, `set snmp v3 notify ...` | SNMPv3 trap targets require the fuller `vacm`/`target-address`/`target-parameters`/`notify` chain, not a single `trap-group` line — more verbose than v1/v2c. |
| Read back full SNMP config | `show configuration snmp` | |

Sources: [SNMP Communities](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/snmp-communities.html), [SNMP Traps](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/snmp-traps.html), [Configure SNMPv3](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/configure-snmpv3.html), [Configure the SNMPv3 Authentication Type and Encryption Type](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/configure-the-snmpv3-authentication-type-and-encryption-type.html).

## Session/paging control

| Task | Command(s) | Notes |
|---|---|---|
| Disable output paging for the session | `set cli screen-length 0` | Operational-mode command (not config mode); SignalScope should send this at session start and **echo it in the terminal** like any other command, per [connectivity-methods.md](../../00-architecture/connectivity-methods.md). |
| Disable screen width wrapping | `set cli screen-width 0` | Useful alongside screen-length for clean scripted parsing of wide `show` output. |
| Disable confirmation prompts for destructive ops | `set cli restart-on-upgrade` (unrelated) — no direct Junos equivalent of Cisco's `terminal no confirm` | Junos generally prompts less aggressively than IOS for destructive operational commands; most risk is scoped through the commit model instead. |

## Show commands for read-back

| Task | Command(s) | Notes |
|---|---|---|
| Interface summary (admin/oper/description) | `show interfaces terse` | Compact table: interface, admin, link, proto, local, remote — closest Junos equivalent to `show ip interface brief`. |
| Full interface detail | `show interfaces ge-0/0/1 extensive` | Includes counters, errors, optics diagnostics. |
| VLAN table | `show vlans` / `show vlans staff detail` | |
| Spanning-tree per-interface state | `show spanning-tree interface` | |
| Bridge MAC table | `show ethernet-switching table` | Junos-specific equivalent of `show mac address-table`. |
| LLDP neighbors | `show lldp neighbors` | |
| Committed interface config (source of truth for GUI read-back) | `show configuration interfaces ge-0/0/1` | Reflects **active/committed** config only — will not show uncommitted candidate edits, which is itself an important GUI-state distinction (see [gui-cli-snmp-mapping.md](gui-cli-snmp-mapping.md)). |
| Commit/rollback history | `show system commit` | |
