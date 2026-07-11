# Cisco IOS / IOS-XE (Catalyst) — Overview

## CLI dialect / platform scope

- **Dialect family**: "Cisco IOS" CLI — the classic EXEC-mode / global-config / sub-mode grammar (`enable`, `configure terminal`, `interface X`, `end`) that Cisco has kept essentially stable since the 1990s. On current hardware this CLI is delivered by **IOS-XE** (a Linux-hosted, modular successor to monolithic IOS), not classic IOS, but the command syntax at the switching layer is deliberately near-identical so this doc treats "IOS" and "IOS-XE" CLI as one dialect for SignalScope's purposes. Legacy **CatOS** (`set` command syntax, e.g. `set vlan 10 2/1`) is a *different* dialect from a much older Catalyst generation (4000/5000/6000 with Supervisor running CatOS) — out of scope here; assume every device SignalScope talks to over this dialect is running IOS/IOS-XE.
- **Platforms covered**: Catalyst 9000 series (9200/9300/9400/9500/9600 — the current fixed/modular access-to-core lineup, all IOS-XE) is the primary target. Older but still commonly deployed IOS-XE and classic-IOS platforms (Catalyst 3750-X/3850/2960-X, IOS 15.x) use the same CLI dialect documented here with minor per-platform command-availability differences (e.g. StackWise support, some `show` output formatting) rather than syntax differences.
- Cisco's own switching-config documentation is organized per-platform-family under `docs.cisco.com` (see Sources below) rather than one unified reference; command syntax has been cross-checked against the Catalyst 9300 IOS-XE 17.x guides as the canonical modern reference.

## SNMP version support

- **v1**: supported for backward compatibility, community-string only, 32-bit counters. Rarely the right choice for new deployments.
- **v2c**: supported, adds `GETBULK` and 64-bit counters (`ifXTable`'s `ifHCInOctets` etc.) — still community-string auth only, cleartext.
- **v3**: supported on IOS-XE via `snmp-server user` / `snmp-server group`, with USM auth (MD5/SHA, SHA preferred/required on newer releases) and optional privacy (DES/3DES/AES; AES128/192/256 on current IOS-XE). This is the version SignalScope should default to per the architecture doc's general posture.
- All three can be enabled simultaneously on the same device (separate `snmp-server community` vs `snmp-server group`/`user` configuration); the agent doesn't have a single global "version" switch.

### Practical scope of SNMP SET on a real Catalyst switch

This is the load-bearing fact for SignalScope's write-path design: **out of the box, Cisco's SNMP agent on a Catalyst switch is read-mostly**. What's realistically SET-able in practice:

| Realistically writable via SNMP SET | Notes |
|---|---|
| `IF-MIB::ifAdminStatus` | Bring an interface up/down. Works on essentially all Cisco SNMP-enabled devices; this is the one standard-MIB write path that's reliably present. |
| `IF-MIB::ifAlias` | Interface description string — writable in practice on IOS-XE. |
| `CISCO-VLAN-MEMBERSHIP-MIB::vmVlan` / `vmVlanType` | Per-port VLAN assignment — documented and supported by Cisco specifically for this purpose (see `mib-reference.md`), but requires the SNMP read-write community/user to have access and (historically) VTP considerations. |
| `CISCO-CONFIG-COPY-MIB` (`ccCopyTable`) | Trigger a config copy — including `running-config` → `startup-config` (the SNMP equivalent of `write memory`) or an export to a TFTP/SCP server. This is the standard NMS-integration path for "save config" and for pulling backups, and is explicitly documented by Cisco (see Sources). |
| `Q-BRIDGE-MIB::dot1qVlanStaticRowStatus` / static VLAN table | Creating/destroying VLANs via SNMP is documented by Cisco as supported, though it's less commonly exercised in the field than the CLI equivalent. |

| Realistically read-only / not exposed for SET in practice | Notes |
|---|---|
| Spanning-tree parameters (`BRIDGE-MIB`, `dot1dStp*`) | Cisco's SNMP agent exposes STP state for polling; per-port cost/priority/portfast are not a reliable SNMP-write path on modern IOS-XE — these go through CLI. |
| LACP/EtherChannel membership | No standard or commonly-enabled Cisco MIB exposes SNMP-settable channel-group membership; this is CLI-only. |
| Port-security enable/max/violation-action | `CISCO-PORT-SECURITY-MIB` objects are marked read-write in the MIB definition (see `mib-reference.md`), but Cisco does not document or generally support driving port-security configuration via SNMP SET in production guidance — treat as CLI-only in practice, MIB notwithstanding. |
| SNMP agent's own config (community strings, trap hosts, users) | Cannot be configured via SNMP itself (chicken-and-egg / by design) — CLI or another management channel only. |
| CPU/memory (`CISCO-PROCESS-MIB`, `CISCO-MEMORY-POOL-MIB`) | Read-only by nature (telemetry, not config). |

The general rule SignalScope should encode: **assume CLI-only unless a specific object is called out above (or in `mib-reference.md`) as documented-writable.** Never assume a SET path exists just because a MIB defines an object as `read-write` — Cisco's own field guidance and community MIB browsers frequently show `MAX-ACCESS read-write` on objects (e.g. in `CISCO-PORT-SECURITY-MIB`) that aren't part of any Cisco-published SNMP-driven configuration workflow.

## Config-apply model

IOS/IOS-XE uses an **immediate-apply, separately-persisted** model — this is the same family behavior as Arista/Huawei/Aruba, and the opposite of Juniper's candidate-config/`commit` model:

1. Every line entered in a config sub-mode (`configure terminal`, `interface GigabitEthernet0/1`, `no shutdown`, etc.) takes effect on the **running-config immediately**, line by line, as it's typed/received — there is no staging step and no separate "commit" verb for ordinary configuration.
2. The running-config lives in RAM and is lost on reload/power-loss unless explicitly persisted.
3. Persisting to NVRAM (`startup-config`) requires an explicit separate step: `copy running-config startup-config` (interactive, prompts for destination filename, defaults to `startup-config`) or the older shorthand `write memory` / `wr` (non-interactive, does the same thing without prompting).
4. This has a direct GUI consequence per SignalScope's unification model: a GUI "Apply" action for a Cisco device is **already** effectively live the moment the CLI lines are sent — there's no pending/uncommitted state to show, unlike Juniper. A GUI "Save" button for a Cisco device therefore doesn't mean "make my pending edits take effect" (they already have) — it means **"persist the already-live running-config to NVRAM so it survives a reload,"** i.e. it should map to `write memory` / `copy running-config startup-config`, not to some notion of committing edits. The GUI should probably surface this as a distinct affordance from "Apply" (e.g. "Save to startup-config") so users don't confuse Cisco's semantics with Juniper's when working across a multi-vendor topology.
5. Corollary risk SignalScope's terminal/GUI should surface: since config takes effect immediately, an interface accidentally shut down or a bad ACL applied mid-session is live *now*, before any "Save" — the safety net Cisco/IOS-XE offers here is `reload in <minutes>` + `configure confirm`/`reload cancel` for planned changes, and simply **not** issuing `write memory` (a `reload` without saving reverts to the last-persisted startup-config). SignalScope's supervised-remediation tier (see `connectivity-methods.md`) should be aware that "undo" for Cisco devices generally means "don't save, then reload" rather than any transactional rollback.

## Official documentation sources used

- [Cisco SNMP Object Navigator](https://www.cisco.com/c/en/us/support/web/tools/snmp/help/index.html) — used to cross-check MIB/object availability per platform.
- [`cisco/cisco-mibs` GitHub mirror](https://github.com/cisco/cisco-mibs) — Cisco's own public MIB repository (see licensing note below); its GitHub Pages index is at [cisco.github.io/cisco-mibs](https://cisco.github.io/cisco-mibs/).
- [Cisco SNMP Object Navigator / MIB FAQ](https://www.cisco.com/c/en/us/support/docs/ip/simple-network-management-protocol-snmp/9226-mibs-9226.html)
- [How To Copy Configurations To and From Cisco Devices Using SNMP](https://www.cisco.com/c/en/us/support/docs/ip/simple-network-management-protocol-snmp/15217-copy-configs-snmp.html) — `CISCO-CONFIG-COPY-MIB` usage, including `write memory`-equivalent SNMP flow.
- [How To Add, Modify, and Remove VLANs on a Catalyst Using SNMP](https://www.cisco.com/c/en/us/support/docs/ip/simple-network-management-protocol-snmp/45080-vlans.html) — `CISCO-VLAN-MEMBERSHIP-MIB` / VTP MIB usage.
- [Collect CPU Utilization on Cisco IOS Devices with SNMP](https://www.cisco.com/c/en/us/support/docs/ip/simple-network-management-protocol-snmp/15215-collect-cpu-util-snmp.html) — `CISCO-PROCESS-MIB` guidance.
- [Cisco IOS Interface and Hardware Component Command Reference](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/interface/command/ir-cr-book.html)
- Catalyst 9300 **Layer 2 Configuration Guide** family, e.g. [Configuring Spanning Tree Protocol / Optional Spanning-Tree Features, IOS-XE 17.x](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-15/configuration_guide/lyr2/b_1715_lyr2_9300_cg/configuring_spanning_tree_protocol.html)
- Catalyst 9300 **Security Configuration Guide** — [Port Security, IOS-XE 17.x](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-16/configuration_guide/sec/b_1716_sec_9300_cg/port_security.html)
- Catalyst 9300/9500 **Network Management / System Management Configuration Guide** — [Configuring Simple Network Management Protocol, IOS-XE 16.11.x](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9500/software/release/16-11/configuration_guide/nmgmt/b_1611_nmgmt_9500_cg/configuring_simple_network_management_protocol.html), and [Administering the Device (System Mgmt CG), IOS-XE 17.9.x](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-9/configuration_guide/sys_mgmt/b_179_sys_mgmt_9300_cg/administering_the_device.html)
- [Command Reference — Layer 2/3 Commands (EtherChannel/`channel-group`), IOS-XE 16.11.x, Catalyst 9300](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/16-11/command_reference/b_1611_9300_cr/layer_2_3_commands.html)
- Product landing pages for configuration-guide indexes: [Catalyst 9300](https://www.cisco.com/c/en/us/support/switches/catalyst-9300-series-switches/products-installation-and-configuration-guides-list.html), [Catalyst 9500](https://www.cisco.com/c/en/us/support/switches/catalyst-9500-series-switches/products-installation-and-configuration-guides-list.html)

Note: `cisco.com` pages return HTTP 403 to non-browser fetch tools in this environment, so specific page content above was cross-checked via search-result summaries and independent MIB-browser mirrors (`oidref.com`, `mibs.observium.org`) rather than direct page fetch; URLs are recorded as-found and should still be treated as the canonical Cisco sources.

## MIB licensing / redistribution note

The `cisco/cisco-mibs` GitHub repository ([cisco/cisco-mibs](https://github.com/cisco/cisco-mibs)) is **Cisco's own public mirror** of the MIB files formerly served from `ftp://ftp.cisco.com/pub/mibs/` (that FTP service was decommissioned October 15, 2022; GitHub is now the canonical public source). It carries Cisco's standard copyright headers per-file but **no repository-level open-source license file** (no `LICENSE`). SignalScope treats this as "vendor-published for third-party interoperability tooling use" — the whole point of Cisco publishing MIBs is so NMS/tooling vendors can consume them — but out of caution:

- We mirror a **small, individually-selected set of `.my` files** (not a bulk clone of the repo) into `vendors/cisco/mibs/` in this docs tree, each retaining its original Cisco copyright header.
- We attribute the source per-file below rather than presenting them as SignalScope-authored.
- If SignalScope ever needs the full MIB set at runtime (e.g. for a general-purpose MIB-browser feature), the correct approach is to fetch/cache from `cisco/cisco-mibs` at build/deploy time rather than vendoring the whole repo into this docs tree.

### Files vendored under `mibs/`

| File | Source | Why vendored |
|---|---|---|
| `CISCO-CONFIG-COPY-MIB.my` | [`v2/CISCO-CONFIG-COPY-MIB.my`](https://raw.githubusercontent.com/cisco/cisco-mibs/main/v2/CISCO-CONFIG-COPY-MIB.my) | Backs the SNMP "save config" / config-backup GUI action (see `gui-cli-snmp-mapping.md`). |
| `CISCO-VLAN-MEMBERSHIP-MIB.my` | [`v2/CISCO-VLAN-MEMBERSHIP-MIB.my`](https://raw.githubusercontent.com/cisco/cisco-mibs/main/v2/CISCO-VLAN-MEMBERSHIP-MIB.my) | Backs the SNMP-writable per-port VLAN assignment GUI action. |
| `CISCO-STACK-MIB.my` | [`v2/CISCO-STACK-MIB.my`](https://raw.githubusercontent.com/cisco/cisco-mibs/main/v2/CISCO-STACK-MIB.my) | Referenced by the assignment brief; note this is a **large legacy MIB** (originates 1995, CatOS-era Catalyst 4000/5000/6000 chassis/module/port management) — largely superseded on modern IOS-XE Catalyst 9000 stacks by `CISCO-STACKWISE-MIB` (not vendored — no exact-OID confirmation obtained this session beyond its root, see `mib-reference.md`) plus `ENTITY-MIB`. Vendored for reference/attribution completeness, not as a recommended primary source for new IOS-XE integration work. |
| `CISCO-PORT-SECURITY-MIB.my` | [`v2/CISCO-PORT-SECURITY-MIB.my`](https://raw.githubusercontent.com/cisco/cisco-mibs/main/v2/CISCO-PORT-SECURITY-MIB.my) | Backs the port-security GUI-mapping row; also the concrete example of a MIB whose objects are `read-write` in definition but not a documented Cisco SNMP-configuration workflow in practice (see overview table above). |

Not vendored (found to exist at the same `v2/` path, 200 OK, but out of scope for this curated set): `CISCO-VTP-MIB.my`, `CISCO-PROCESS-MIB.my`, `CISCO-MEMORY-POOL-MIB.my`. OIDs for these are cited numerically in `mib-reference.md` from search-confirmed sources without vendoring the files themselves.
