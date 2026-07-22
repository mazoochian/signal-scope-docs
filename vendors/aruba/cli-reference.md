# ArubaOS-CX CLI Reference

Curated command tables for SignalScope's terminal/GUI unification. All syntax below is ArubaOS-CX (6300/6400/8320/8325/8400 and related current-generation series) — do **not** apply this to legacy ArubaOS-Switch/ProCurve or Comware-based HPE gear (see [overview.md](overview.md)). Where a command's exact syntax was sourced only via search-engine summary rather than a directly-fetched official page (several official CLI-bank pages returned HTTP 403 to automated fetch this session), that is flagged inline — treat those as high-confidence but not first-hand-verified.

## Entering config mode and interface context

| Task | Command(s) | Notes |
|---|---|---|
| Enter global config mode | `configure terminal` (short form `conf t` generally accepted) | Standard privileged-EXEC → config-mode transition, IOS-like in shape. |
| Enter interface context | `interface 1/1/1` | Interface names are slot/module-agnostic `<member>/<slot>/<port>` on chassis/stacked systems (e.g. `1/1/1`), not `GigabitEthernet0/1`-style names. On fixed 1U switches this is typically `1/1/<port>`. |
| Enter LAG interface context | `interface lag 1` | See LACP/LAG section below. |
| Enter VLAN interface (SVI) context | `interface vlan 10` | Creates/enters the routed VLAN interface, separate from `vlan 10` (the L2 VLAN definition itself). |
| Exit current context | `exit` | Standard. |
| Return to privileged EXEC from any depth | `end` | IOS-like convenience, not independently re-verified this session but consistent with all observed AOS-CX transcripts. |

## Interface admin state & description

| Task | Command(s) | Notes |
|---|---|---|
| Disable a port | `shutdown` (inside `interface 1/1/1` context) | Maps to `IF-MIB::ifAdminStatus` = `down(2)`. |
| Enable a port | `no shutdown` | Maps to `ifAdminStatus` = `up(1)`. |
| Set interface description | `description <TEXT>` | Maps to `IF-MIB::ifAlias` (`ifXTable`). |
| Remove description | `no description` | |

## VLAN configuration

| Task | Command(s) | Notes |
|---|---|---|
| Create/enter a VLAN | `vlan 10` (global config context, not interface context) | Creates VLAN 10 if it doesn't exist. |
| Name a VLAN | `name <TEXT>` (inside `vlan 10` context) | |
| Set access port + access VLAN | `vlan access 10` (inside `interface 1/1/1` context) | Sets the interface to access mode and assigns PVID 10 in one command — distinct two-step Cisco-style `switchport mode access` + `switchport access vlan 10` is collapsed into one line on AOS-CX. Maps conceptually to `Q-BRIDGE-MIB::dot1qPvid`. |
| Configure trunk allowed VLANs | `vlan trunk allowed 10,20` (inside interface context) | Sourced via search summary of the official CLI-bank page (direct fetch 403'd this session) — full syntax is `vlan trunk allowed {<VLAN-LIST> | all}`, supports ranges (e.g. `11-15`) and a documented limit of up to 256 VLAN IDs / 50 VLAN names combined. `no vlan trunk allowed [<VLAN-LIST>]` removes specific IDs or the whole trunk-allowed set. |
| Set trunk native VLAN | `vlan trunk native 1` (inside interface context) | Sets the untagged/native VLAN on a trunk port. |
| Set trunk native VLAN tag behavior | `vlan trunk native <vlan-id> tag` | Optional `tag` keyword forces the native VLAN to also be tagged (no truly-untagged VLAN on that trunk) — confirm exact keyword against firmware version in use. |
| Remove VLAN | `no vlan 10` (global config) | |
| Show VLAN detail | see Show commands section | |

## Spanning tree

| Task | Command(s) | Notes |
|---|---|---|
| Enable/select STP mode globally | `spanning-tree` / `spanning-tree mode {mstp\|rpvst\|pvst}` | AOS-CX supports MSTP (default/802.1s) and RPVST+ (Rapid-PVST, Cisco-interop mode) as alternate global modes — confirm exact mode keyword set per firmware/model, not independently re-verified this session beyond MSTP being confirmed present. |
| Set port to edge/admin-edge (portfast-equivalent) | `spanning-tree port-type admin-edge` (inside interface context) | Immediately transitions the port to forwarding on link-up, skipping listening/learning — the AOS-CX analogue of Cisco's `spanning-tree portfast`. Confirmed via official AOS-CX 10.16 L2 Bridging Guide. |
| Enable BPDU Guard on a port | `spanning-tree bpdu-guard` (inside interface context) | Disables (error-disables) the port on receipt of any BPDU — pairs with admin-edge ports. Confirmed via official AOS-CX 10.15 Hardening Guide. |
| Configure BPDU Guard auto-recovery timeout | `spanning-tree bpdu-guard timeout <seconds>` | Auto re-enables a BPDU-Guard-disabled port after the timeout. |
| Set port STP priority | `spanning-tree <PORT-LIST> priority <priority-multiplier>` | Sourced via search summary — verify exact multiplier range (typically 0-15 in steps of 16, i.e. 0-240, standard 802.1D/w convention) against a live device. |
| Disable STP on a port | `spanning-tree bpdu-filter` or `no spanning-tree` (context-dependent) | Not independently confirmed this session — verify exact per-port-disable keyword before relying on it. |

## LACP / LAG

| Task | Command(s) | Notes |
|---|---|---|
| Create a LAG interface | `interface lag 1` (global config) | Creates/enters LAG (link aggregation group) 1. |
| Add a member port to a LAG | `lag 1` (inside the member's `interface 1/1/1` context) | Assigns physical interface 1/1/1 as a member of LAG 1. |
| Set LACP mode active | `lacp mode active` (inside `interface lag 1` context) | Confirmed via official AOS-CX Link Aggregation Guide + CLI-bank. Active mode initiates LACP negotiation; the alternative is `lacp mode passive`. |
| Set LACP mode passive | `lacp mode passive` | Only responds to LACP, doesn't initiate — both sides passive means the LAG never forms. |
| Show LACP status | see Show commands section | |

## Port security

| Task | Command(s) | Notes |
|---|---|---|
| Enable port security on a port | `port-access port-security enable` (inside interface context, entering a `config-if-port-security` sub-context) | Confirmed structurally via official AOS-CX Security Guide (8360 series) and CLI-bank; exact availability/depth varies by switch series/firmware (introduced later than base L2 features). |
| Set max MAC/client limit | `client-limit <1-64>` (inside `config-if-port-security` context) | Confirmed range 1-64 directly from `ARUBAWIRED-PORTSECURITY-MIB` source (`arubaWiredClientLimit Unsigned32 (1..64)`) — see [mib-reference.md](mib-reference.md). |
| Statically permit a MAC | `mac-address <MAC>` (inside `config-if-port-security` context) | |
| Set violation action | (mib-confirmed values: notify / shutdown — exact CLI keyword not independently re-verified this session) | `ARUBAWIRED-PORTSECURITY-MIB::arubaWiredViolationAction` confirms the two possible states (`notify(1)`, `shutdown(2)`); the literal CLI keyword pairing needs confirmation against the Security Guide for the exact firmware/model before relying on it verbatim. |
| Enable sticky MAC learning | (mib-confirmed feature exists: `arubaWiredStickyEnable`) | CLI keyword not independently re-verified this session. |

## SNMP agent configuration

| Task | Command(s) | Notes |
|---|---|---|
| Configure a read-only v1/v2c community | `snmp-server community <STRING>` | Optional `acl <ACL-NAME>` to restrict source IPs — sourced via search summary of the official Configuring SNMP guide. |
| Configure a trap host (SNMPv3) | `snmp-server host <IP> trap version v3 user <USER> vrf mgmt` | `vrf mgmt` scopes the trap send to the out-of-band management VRF where used — confirm VRF name/necessity against the target deployment. |
| Configure a trap host (v1/v2c) | `snmp-server host <IP> trap version {1\|2c} community <STRING>` | Inferred structurally from the v3 form and general AOS-CX SNMP guide conventions — not independently re-verified this session, confirm exact keyword order against the Configuring SNMP guide. |
| Create an SNMPv3 user | `snmpv3 user <USER> auth {md5\|sha} auth-pass plaintext <PASS> [priv {des\|aes} priv-pass plaintext <PASS>] access-level {ro\|rw}` | Confirmed via official Configuring SNMPv3 guide content (auth/priv protocol keywords, `access-level` parameter). Updating auth/priv on an existing user requires also re-specifying `access-level`. |
| Create an SNMPv3 context (VRF-scoped) | `snmpv3 context <NAME> vrf <VRF-NAME> [community <STRING>]` | Confirmed via official CLI-bank `snmpv3 context` page (search-summary sourced). |
| Enable/disable the SNMP agent | (not independently confirmed this session) | Verify whether AOS-CX has a global `snmp-server enable`-equivalent toggle vs. agent-always-on-if-community-configured. |

## Paging control

| Task | Command(s) | Notes |
|---|---|---|
| Disable output paging for the session | `no page` | **Updated (this pass) — the earlier "persists across reboot" claim does not hold up.** The official AOS-CX CLI-Bank page for this exact command was located (`arubanetworking.hpe.com/techdocs/AOS-CX/AOSCX-CLI-Bank/cli_8400/Content/cli_ses_cmds/page.htm`, matched by title/path, not a guess) but 403'd to direct fetch (same Akamai bot-protection behavior hit by prior Aruba research passes). A search-engine summary of that specific official page states plainly: the `page` command "specifies the number of output lines the CLI displays before pausing... **this setting is not persistent and applies to the current session only**." This is consistent with the equivalent AOS8 (ArubaOS, the wireless-controller product line, not AOS-CX) documentation, which explicitly describes `no paging` as per-user-session. **Confidence tier: search-engine summary of the specific, correctly-identified official page — one tier below a first-hand fetch, but a real update from the previous community-forum-only sourcing.** Treat `no page` as a normal per-session toggle (like every other vendor in this docs tree), not a persistent global setting — reverse the previous guidance. SignalScope should still echo this command explicitly per the "no hidden setup commands" principle (see [connectivity-methods.md](../../00-architecture/connectivity-methods.md)), and re-issue it per new session rather than assuming a prior session's `no page` carries forward. |

## Save / persist configuration and checkpoints

| Task | Command(s) | Notes |
|---|---|---|
| Save running-config to startup-config | `copy running-config startup-config` | Confirmed via official CLI-bank. Overwrites existing startup-config. |
| Save running-config to startup-config (alias) | `write memory` | Confirmed as a documented alias of `copy running-config startup-config` via official CLI-bank `write memory` page. |
| Create a named checkpoint from running-config | `copy running-config checkpoint <CHECKPOINT-NAME>` | Confirmed via official CLI-bank. Does **not** touch startup-config. |
| Restore running-config from a checkpoint | `copy checkpoint <CHECKPOINT-NAME> running-config` | Confirmed via official CLI-bank (`copy checkpoint <CHECKPOINT-NAME> {running-config \| startup-config}`). |
| Save a checkpoint to startup-config | `copy checkpoint <CHECKPOINT-NAME> startup-config` | Same command family as above, alternate destination. |
| Diff two checkpoints | `checkpoint diff <NAME1> <NAME2>` | Referenced in HPE Airheads community discussion of checkpoint tooling; exact output symbol legend (+/-/@) not independently confirmed this session — community members note the diff-symbol documentation itself is thin, so treat output parsing as needing live-device verification. |
| Guarded apply with auto-rollback | `checkpoint auto <MINUTES>` | Community-sourced (Airheads) — arms an auto-revert timer; CLI prompts for confirmation near the end of the window, and un-confirmed changes revert automatically. See [overview.md](overview.md) config-apply model discussion. |
| List/show checkpoints | see Show commands section | |

## Show commands (read-back)

| Task | Command(s) | Notes |
|---|---|---|
| Interface status/detail | `show interface 1/1/1` | Standard per-interface detail (admin/oper state, counters, description). |
| Interface brief summary | `show interface brief` | Table view across all interfaces — likely name/status/VLAN/duplex/speed columns, not independently re-verified this session for exact column set. |
| VLAN table | `show vlan` | Lists VLAN ID/name/status/port-membership summary. |
| VLAN detail for one VLAN | `show vlan 10` | |
| Spanning-tree status | `show spanning-tree` | Global + per-port STP state. |
| Spanning-tree detail per interface | `show spanning-tree interface 1/1/1` | Inferred from standard AOS-CX `show ... interface <X>` pattern used elsewhere — not independently re-verified this session. |
| LACP/LAG status | `show lacp interfaces` | Confirmed via official CLI-bank page title/existence (`show lacp interfaces`). |
| LAG summary | `show interface lag 1` | Inferred from standard interface-show pattern. |
| Running configuration | `show running-config` | Standard, used for CLI→GUI diff-based state sync per [gui-cli-snmp-unification.md](../../00-architecture/gui-cli-snmp-unification.md). |
| Startup configuration | `show startup-config` | |
| List checkpoints | `show checkpoint` (exact form not independently confirmed) | Verify against live device — some AOS-CX documentation implies checkpoint listing may live under `show running-config files` or a dedicated `show checkpoint` command; not resolved with high confidence this session. |
| SNMP configuration | `show snmp-server` (exact form not independently confirmed) | Verify exact command name against the Configuring SNMP guide for the target firmware. |
