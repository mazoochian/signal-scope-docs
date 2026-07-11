# MikroTik RouterOS — CLI Reference

All commands below are literal RouterOS console menu-path syntax. Per `overview.md`, there is no config-mode to enter/exit — each line is a complete, immediately-applied command whether typed from `/` (fully-qualified path) or from inside the relevant menu level after navigating there (e.g. `/interface bridge` then bare `port set ...`). Both forms produce identical device state and identical text for SignalScope's terminal-echo/audit purposes.

## Interface admin state and description

| Task | Command(s) | Notes |
|---|---|---|
| Disable a port | `/interface disable ether1` | Takes effect immediately — no `shutdown`/`no shutdown` verb pair, no context switch. Accepts a comma list: `/interface disable ether1,ether2`. |
| Enable a port | `/interface enable ether1` | |
| Set interface comment/description | `/interface set ether1 comment="uplink to core-sw1"` | `comment` is RouterOS's equivalent of Cisco `description` / maps to `IF-MIB::ifAlias` for read-back. |
| Rename an interface | `/interface set ether1 name=uplink-core1` | Renaming changes the string used in all subsequent CLI references to this port — SignalScope must track this, not assume `ether1` is a stable handle. |

## Bridge / VLAN configuration — modern bridge-VLAN-filtering approach (RouterOS v7, current best practice)

RouterOS v7's recommended model: a single bridge interface per switch (or per isolated switching domain) with `vlan-filtering=yes`, port VLAN membership expressed via `/interface bridge vlan`, and access-port PVID via `/interface bridge port`. This is a software-defined 802.1Q model layered on the bridge, distinct from the older switch-chip-native approach below.

| Task | Command(s) | Notes |
|---|---|---|
| Create a bridge | `/interface bridge add name=bridge1` | |
| Add ports to the bridge | `/interface bridge port add bridge=bridge1 interface=ether1`<br>`/interface bridge port add bridge=bridge1 interface=ether2` | Do this **before** enabling `vlan-filtering` to avoid losing access mid-configuration if management is via one of these ports. |
| Enable VLAN filtering on the bridge | `/interface bridge set bridge1 vlan-filtering=yes` | Globally turns on 802.1Q VLAN enforcement for the bridge. MikroTik's own guidance: configure ports and VLAN table entries first, enable this last, from a session that won't be cut off (console/out-of-band) in case of lockout. |
| Set an access port's PVID | `/interface bridge port set [find interface=ether2] pvid=20` | PVID tags untagged ingress traffic on that port with VLAN 20. You do not need to separately add the port as an "untagged" bridge-vlan member — see next row. |
| Add a bridge-VLAN entry (trunk + access ports) | `/interface bridge vlan add bridge=bridge1 tagged=ether1 vlan-ids=20`<br>`/interface bridge vlan add bridge=bridge1 tagged=ether1 untagged=ether2 vlan-ids=30` | `tagged=` lists trunk-style member ports (VLAN tag preserved on egress); `untagged=` lists access-style member ports (tag stripped on egress). Per MikroTik's docs, ports whose `pvid` already matches a VLAN are auto-added as untagged members of that VLAN's entry — explicit `untagged=` entries are mainly needed when you want to be explicit or add ports whose PVID differs from what's implied. `vlan-ids=` accepts single IDs, comma lists, or ranges (`100-115,120`). |
| Set VLAN ID range/list on one entry | `/interface bridge vlan set [find vlan-ids=20] tagged=ether1,ether5` | Multiple VLAN IDs in one bridge-vlan entry should only be used when all member ports are `tagged` — mixing multiple VLAN IDs with `untagged` ports in the same entry is documented as unsafe (ambiguous which VLAN an access port's untagged traffic belongs to). |
| Restrict ingress frame types on a port | `/interface bridge port set [find interface=ether6] frame-types=admit-only-vlan-tagged` | Options: `admit-all` (default) / `admit-only-untagged-and-priority-tagged` / `admit-only-vlan-tagged`. Equivalent in spirit to Cisco's implicit trunk/access enforcement. |

### Older switch-chip-based VLAN approach (hardware-offloaded, pre-v7-style — still present on many CRS1xx/2xx deployments)

| Task | Command(s) | Notes |
|---|---|---|
| Add VLAN entry to the switch chip's own table | `/interface ethernet switch vlan add switch=switch1 vlan-id=20 ports=ether1,ether2` | Operates on the switch chip's native VLAN table (`/interface ethernet switch`), bypassing the Linux-side bridge entirely — fully hardware-offloaded on all supported chips, unlike bridge VLAN-filtering which disables HW offload on some chips (RTL8367 / 88E6393X / 88E6191X / 88E6190 / MT7621 / MT7531 / EN7523 support HW-offloaded bridge VLAN-filtering as of RouterOS v7; older/other chips do not). |
| Tag egress on a port | `/interface ethernet switch egress-vlan-tag add tagged-ports=ether1 vlan-id=20 switch=switch1` | Marks which ports should egress VLAN 20 traffic tagged (trunk-style). |
| Set default ingress VLAN (access port) | `/interface ethernet switch port set ether2 default-vlan-id=20 vlan-mode=secure` | Chip-native analogue of PVID. |
| **Recommendation** | — | MikroTik's current documentation treats bridge VLAN-filtering as the modern default; the switch-chip approach remains documented for CRS1xx/2xx-family hardware and for cases needing full HW offload on chips the bridge approach can't offload. New SignalScope-managed configs should default to the bridge approach; detect and preserve the switch-chip approach read-only if found already configured on a device. |

## Spanning tree

| Task | Command(s) | Notes |
|---|---|---|
| Set STP/RSTP/MSTP mode on a bridge | `/interface bridge set bridge1 protocol-mode=rstp` | Options: `none` \| `stp` \| `rstp` (default) \| `mstp`. This is a bridge-wide setting, not per-port. |
| Set per-port STP priority/path cost | `/interface bridge port set [find interface=ether1] priority=0x80 path-cost=10` | |
| Read spanning-tree state | `/interface bridge monitor bridge1` | Live/streaming status view (see Read-back section). |

## LACP / bonding

| Task | Command(s) | Notes |
|---|---|---|
| Create an LACP (802.3ad) bond | `/interface bonding add name=bond1 slaves=ether1,ether2 mode=802.3ad lacp-rate=1sec` | `mode=802.3ad` is IEEE-standard LACP. Other modes exist (`active-backup`, `balance-xor`, etc.) but only `802.3ad`, `balance-xor`, and `active-backup` are hardware-offloaded on supporting chips — other modes consume CPU. |
| Add a slave after creation | `/interface bonding set bond1 slaves=ether1,ether2,ether3` | |
| Monitor LACP state | `/interface bonding monitor bond1` | On an 802.3ad bond, output includes additional LACP-specific fields (`lacp-system-id`, `lacp-system-priority`, per-slave LACP state). |

## Port isolation

RouterOS has no single "port-security" feature bundle equivalent to Cisco's (MAC-limiting + violation actions on one config surface); port isolation on a MikroTik bridge is achieved through one of two independent mechanisms:

| Task | Command(s) | Notes |
|---|---|---|
| Software split-horizon isolation (bridge-level) | `/interface bridge port set [find interface=ether2] horizon=1`<br>`/interface bridge port set [find interface=ether3] horizon=1` | Ports sharing the same non-zero `horizon` value cannot forward/flood traffic to each other (any other horizon value, including a different one, can still communicate). **Disables hardware offload** for the bridge — a real throughput cost, called out explicitly in MikroTik's docs. |
| Hardware switch-chip port isolation | `/interface ethernet switch port set ether2 isolate=yes` (chip/model-dependent property name/availability) | Uses the switch chip's native isolation feature — keeps HW offload intact. Available only where the underlying switch chip supports it; verify per-model. |
| **Recommendation** | — | Prefer switch-chip isolation when full HW-offloaded throughput matters and the chip supports it; use bridge `horizon` when it doesn't or when isolation groups need to span switch-chip boundaries. |

## SNMP agent configuration

| Task | Command(s) | Notes |
|---|---|---|
| Enable the SNMP agent | `/snmp set enabled=yes` | Disabled by default. Verify with `/snmp print`. |
| Add a v1/v2c-style community | `/snmp community add name=ro-community address=10.0.0.0/24 read-access=yes write-access=no security=none` | `address` restricts source IPs/subnets allowed to query using this community (default `0.0.0.0/0` if unset). `security=none` = no SNMPv3-style auth/priv (i.e., this entry behaves as a plain v1/v2c community). |
| Add an SNMPv3 community (auth+priv) | `/snmp community add name=v3-user security=private authentication-protocol=SHA1 authentication-password=<8+ chars> encryption-protocol=AES encryption-password=<8+ chars>` | RouterOS folds SNMPv3 user/security-model config into the same `/snmp community` table rather than a separate USM user table. `security` levels: `none` (noAuthNoPriv) \| `authorized` (authNoPriv) \| `private` (authPriv). Auth protocol: `MD5`/`SHA1`; encryption: `DES`/`AES`. Both passwords must be ≥ 8 characters. |
| Enable write access on a community | `/snmp community set ro-community write-access=yes` | Per `overview.md`, only a narrow, documented object set is actually writable regardless of this flag — enabling it does not open general config-by-SNMP. |
| Set trap version/destination | `/snmp set trap-version=2 trap-target=10.0.0.5 trap-community=ro-community` | `trap-version` accepts `1`\|`2`\|`3`, default `1`. |

## Save/persist — there is no "save" step

Per `overview.md`, every command above takes effect and is durably persisted the instant it runs — there is no running/startup-config split and no commit step. The two commands below are **backup/export mechanisms only**, not prerequisites for a change to "stick":

| Task | Command(s) | Notes |
|---|---|---|
| Save a full binary config snapshot | `/system backup save name=pre-change-2026-07-11` | Produces a `.backup` file on device flash. Restorable with `/system backup load name=pre-change-2026-07-11`. Binary, not human-diffable. |
| Export current config as a replayable script | `/export file=running-config-2026-07-11` | Produces a `.rsc` text file of CLI commands that reconstruct current state — the practical equivalent of "show running-config" for diffing/version-control purposes (relevant to SignalScope's config-drift-detection tier). `/export compact` omits default-valued properties; `/export terse` produces single-line-per-item output. |
| Retrieve backup/export files off-device | `/tool fetch` (upload) or SCP/SFTP from a management host | Not covered further here — file-transfer mechanics, not CLI config syntax. |

## Read-back / `print` commands

| Task | Command(s) | Notes |
|---|---|---|
| List interfaces + admin/oper state | `/interface print` | Columns include `R`/`X` flags (running/disabled) per row. |
| Print with numeric OIDs for every visible field | `/interface print oid` | See `overview.md` / `mib-reference.md` — RouterOS's live OID-introspection feature, usable at nearly any menu level (`/system health print oid`, `/system resource print oid`, etc.). |
| List bridge VLAN table | `/interface bridge vlan print` | Shows bridge, VLAN IDs, tagged/untagged/current port sets. |
| List bridge ports (PVID, frame-types, horizon) | `/interface bridge port print detail` | |
| Live/streaming bridge status | `/interface bridge monitor bridge1` | Streaming view (STP role/state, etc.) — analogous to Cisco `show spanning-tree interface ... detail` but push-updating in the terminal until interrupted. |
| List bonding interfaces | `/interface bonding print detail` | |
| Print SNMP config | `/snmp print`<br>`/snmp community print detail` | |
| Full config dump (human-readable) | `/export` (see above) | |
