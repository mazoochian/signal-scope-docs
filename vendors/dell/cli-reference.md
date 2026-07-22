# Dell SmartFabric OS10 — CLI Reference

Curated command tables. All syntax is **OS10** (not OS9/FTOS, not N-Series DNOS 6.x — see [overview.md](overview.md)). Interface names use OS10's `ethernet <node>/<slot>/<port>` scheme (e.g. `ethernet 1/1/2`); substitute the platform's actual numbering.

## Mode entry

| Task | Command(s) | Notes |
|---|---|---|
| Enter privileged/config mode | `configure terminal` | No separate `enable` step confirmed this session — OS10 examples consistently show `configure terminal` reachable directly, unlike Cisco's `enable` → `configure terminal` two-step. Verify per-deployment (AAA/privilege-level config could change this). |
| Enter interface context | `interface ethernet 1/1/2` | Prompt becomes `(config-if-eth1/1/2)`. |
| Enter port-channel interface context | `interface port-channel 100` | Prompt becomes `(conf-if-po-100)`. |
| Enter VLAN interface context | `interface vlan 10` | Creates/enters the VLAN's routed interface context. |
| Exit current context | `exit` | Standard. |
| Enter candidate-configuration (transaction) mode | `start transaction` (issued **before** `configure terminal`) | See [overview.md](overview.md)'s config-apply model section — opt-in, not default. Every command entered after this until `commit`/`discard` is staged, not applied. |
| Commit a transaction | `commit` | Applies the entire staged candidate set. |
| Discard a transaction | `discard` | Abandons staged changes without applying. |
| Preview staged changes | `show candidate-configuration` | |
| Diff staged vs. live config | `show diff candidate-configuration running-configuration` | |

## Interface admin state & description

| Task | Command(s) | Notes |
|---|---|---|
| Enable a port | `interface ethernet 1/1/2` → `no shutdown` | Maps to `IF-MIB::ifAdminStatus` = `up(1)`. **Caveat specific to OS10**: per Dell's own SmartFabric OS10 User Guide, `shutdown` on a VLAN (SVI) interface "stops L3-routed traffic only" — L2 traffic continues to pass through the VLAN if it has no IP address configured. This is a real behavioral divergence from every other vendor's `shutdown`, which is unconditional — do not assume OS10 `shutdown` on a VLAN interface has the same effect as on a physical Ethernet interface. |
| Disable a port | `interface ethernet 1/1/2` → `shutdown` | See caveat above for VLAN-interface scope. |
| Set interface description | `description <text>` (inside interface context) — not independently re-confirmed this session, inferred from IOS-like pattern; verify exact keyword | Maps to `IF-MIB::ifAlias`. |

## VLAN configuration

| Task | Command(s) | Notes |
|---|---|---|
| Create/enter a VLAN | `interface vlan 10` (global config) | Confirmed from Dell's "Create or Remove VLANs" documentation page. |
| Set trunk mode | `switchport mode trunk` (inside physical/port-channel interface context) | IOS-identical verb. |
| Allow tagged VLANs on a trunk | `switchport trunk allowed vlan 10` (comma lists and hyphenated ranges supported) | Confirmed. VLAN 1 is untagged-by-default on a trunk-mode interface. |
| Set the untagged/native VLAN on a trunk | `switchport access vlan 20` | **Distinctive naming**: OS10 uses the `switchport access vlan` keyword to set a trunk port's *untagged* VLAN, not to put the port into access mode the way Cisco's identically-named command does — confirm this dual-purpose usage against the full command reference before assuming Cisco-equivalent semantics; Dell's own trunk-mode configuration example issues `switchport mode trunk` + `switchport access vlan 20` + `switchport trunk allowed vlan 10` together, i.e. `switchport access vlan` is being used *within* trunk mode for the untagged VLAN, not as an access-mode toggle. |
| Read back VLAN/port membership | see Show commands section | |

## Spanning tree

| Task | Command(s) | Notes |
|---|---|---|
| Enable EdgePort (PortFast-equivalent) | `spanning-tree port type edge` (inside interface context) | Confirmed — this is OS10's renamed equivalent of OS9's `spanning-tree portfast`; the rename itself is confirmed via Dell's own OS9-to-OS10 command-mapping tech sheet. Skips the Blocking/Learning states, forwarding ~30 seconds sooner. Configure only on end-station-facing links, per Dell's own guidance — enabling on a network-facing link risks loops. |
| Other STP mode/priority commands | Not independently confirmed this session | Verify against the full command reference before generating for a GUI action. |

## LACP / Port-Channel

| Task | Command(s) | Notes |
|---|---|---|
| Create a port-channel interface | `interface port-channel 100` | |
| Add a member, LACP active | `interface ethernet 1/1/2` → `channel-group 100 mode active` | Confirmed sequence: create the port-channel interface first, `exit`, then enter each member's interface context and issue `channel-group <n> mode active` — the member interface references the port-channel ID, rather than the port-channel interface listing its members. |
| Add a member, LACP passive | `channel-group 100 mode passive` | |
| STP interaction caveat | — | Dell's own documentation warns that globally disabling spanning-tree can cause LACP-enabled port-channel member interfaces to flap due to packet loops — worth surfacing as a GUI warning if SignalScope ever exposes a global STP-disable action on OS10. |

## Port security

| Task | Command(s) | Notes |
|---|---|---|
| Enable port security on an interface | `interface ethernet 1/1/1` → `switchport port-security` → `no disable` (enters `config-if-port-sec` sub-context) | Confirmed sequence from Dell's own port-security documentation. The `no disable` step is distinctive — port-security is entered as a sub-context that itself needs explicit enabling, unlike most vendors' single-command enable. |
| Set max learned-MAC limit | `mac-learn limit 100` (inside `config-if-port-sec` context) | Confirmed keyword: `mac-learn limit`, not `port-security maximum` (Cisco) or `client-limit` (Aruba) — a genuine per-vendor naming divergence worth adding to [`comparison/cli-syntax-matrix.md`](../../comparison/cli-syntax-matrix.md) if this vendor's actions get added there later. |
| Set violation action | `mac-learn limit violation {shutdown\|log\|drop\|flood}` | Confirmed four-way enum: `shutdown` (interface goes to **error-disable**, not plain `DOWN` — distinct status worth GUI-surfacing), `log` (record which MAC caused the violation), `drop` (drop offending packets), `flood` (forward anyway) — a richer violation-action set than most other vendors documented in this project (Cisco/Huawei/Aruba use a 2-3-way `protect`/`restrict`/`shutdown`-style enum). |

## SNMP agent configuration

| Task | Command(s) | Notes |
|---|---|---|
| Configure an SNMPv3 user | `snmp-server user <user-name> <group-name> <security-model> [[noauth \| auth {md5\|sha} <auth-password>] [priv {des\|aes} <priv-password>]] [localized] [access <acl-name>] [remote <ip-address> <udp-port>]` | Confirmed full syntax from Dell's own command reference page. |
| Configure a remote engine ID (prerequisite for a remote SNMPv3 user) | `snmp-server engineID <remote-engine-id>` | Must be configured **before** `snmp-server user ... remote ...`, per Dell's own documentation — sequencing matters, same as Huawei's `snmp-agent trap enable` + `target-host` pairing. |
| Configure a trap/inform destination | `snmp-server host {<ipv4>\|<ipv6>} {informs version <n> \| traps version <n> \| version <n>} [security-level] [community-name] [udp-port <port>] [dom\|entity\|envmon\|lldp\|snmp]` | Confirmed structure; the trailing `[dom\|entity\|envmon\|lldp\|snmp]` looks like a per-feature trap-category filter similar in spirit to Huawei's `snmp-agent trap enable feature-name lldp` scoping — not independently confirmed in full this session. |
| Restrict a MIB view | `snmp-server view <view-name> <oid-tree> {included\|excluded}` | Confirmed — same `included`/`excluded` access-control shape as Huawei's `snmp-agent mib-view` and EXOS's `configure snmpv3 add mib-view`, a recurring cross-vendor pattern worth noting generically. |
| Read back configured users | `show snmp user` | Confirmed. |
| Read back configured communities | `show snmp community` (page title confirmed to exist) | |

## Paging control

| Task | Command(s) | Notes |
|---|---|---|
| Suppress pagination for one command | `show running-configuration \| no-more` | Confirmed — a **per-command pipe filter**, not a session-wide setting, structurally different from every other vendor's session-start toggle (`terminal length 0`, `disable clipaging`, etc.) documented in [`comparison/cli-syntax-matrix.md`](../../comparison/cli-syntax-matrix.md). |
| Session-wide paging control | A `terminal` command family exists (confirmed to appear in OS10's "common commands" list) but the exact session-scoped pagination-disable syntax (Cisco's `terminal length 0` equivalent) was **not independently confirmed this session** | If SignalScope needs a one-time session-start paging-disable command for OS10 (per the [`connectivity-methods.md`](../../00-architecture/connectivity-methods.md) pattern used for every other vendor), verify the exact `terminal` sub-command against the full CLI reference before relying on `| no-more` alone — piping every individual `show` command is a materially different automation shape than the other six vendors' one-time session setup. |

## Save / persist configuration

| Task | Command(s) | Notes |
|---|---|---|
| Save running config to startup config | `write memory` | Confirmed, IOS/EOS-shorthand style. |
| Copy running config (general form) | Not independently confirmed this session whether a longer `copy running-configuration startup-configuration` form also exists alongside `write memory`, the way Cisco/Arista offer both | Verify before assuming only the short form exists. |
| Restore startup configuration | Referenced by page title ("Restore startup configuration") in Dell's own docs but exact command not independently confirmed this session | |

## Show commands for read-back

| Task | Command(s) | Notes |
|---|---|---|
| Running configuration | `show running-configuration` | Confirmed page exists in the official user guide. |
| Candidate configuration (staged, unapplied) | `show candidate-configuration` | Only meaningful inside a `start transaction` session — see mode-entry section above. |
| LACP / port-channel state | `show lacp port-channel` (confirmed page title) | Exact output columns not independently confirmed this session. |
| Port-security state | `show switchport port-security` (confirmed page title) | |
| SNMP config read-back | `show snmp user` / `show snmp community` | |

**Uncertainty flags carried over from [overview.md](overview.md)**: this vendor's syntax was researched via targeted search-summary and page fetches rather than a full primary-document read throughout, unlike the original seven Tier-1 vendors. Rows without an explicit "confirmed" note above should be treated as plausible-but-unverified — re-check against the full SmartFabric OS10 User Guide PDF (linked in overview.md) before generating any of this verbatim in production GUI-to-CLI mapping logic.
