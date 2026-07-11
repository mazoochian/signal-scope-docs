# Huawei VRP (S-series switches) — Overview

Scope: Huawei's **VRP** (Versatile Routing Platform) CLI/SNMP stack as implemented on the enterprise campus **S-series** Ethernet switches (S1720, S2700, S3700, S5700, S6700, S6720, and their CloudEngine successors S3700/S5700/S6700 "V600" trains). VRP is also the OS for Huawei's AR routers and NetEngine/CloudEngine gear — command syntax below is generally VRP-wide, but table/OID specifics are S-series-flavored where noted.

## CLI dialect

VRP's CLI is **structurally similar to Cisco IOS** in the broad strokes — a hierarchical modal CLI with a global configuration context (`system-view`, VRP's analogue of `configure terminal`) and per-interface sub-contexts (`interface GigabitEthernet0/0/1`, analogous to `interface <if>`). Tab-completion, `?` context help, and abbreviated command matching all behave similarly to IOS.

Distinct from Cisco in ways that matter for a literal command-echo model:

| Aspect | Cisco IOS | Huawei VRP |
|---|---|---|
| Enter global config | `enable` then `configure terminal` | `system-view` (no separate unprivileged/privileged EXEC split by default — VRP has user levels 0-15 but Telnet/SSH login typically lands at a level that can go straight to `system-view`) |
| Interface naming | `GigabitEthernet0/1` (slot/port) | `GigabitEthernet0/0/1` (**stack/slot/port**, three-part on modular and stackable S-series) |
| Read-back verb | `show ...` | `display ...` |
| Negate a line | `no <command>` | `undo <command>` |
| Paging control | `terminal length 0` | `screen-length 0 temporary` (session-scoped; see below) |
| Persist config | `write` / `copy running-config startup-config` | `save` (writes to the "next-startup" config file, `vrpcfg.zip` by default — see below) |
| VLAN trunk allow-list | `switchport trunk allowed vlan add <n>` | `port trunk allow-pass vlan <n>` |
| Port on/off | `shutdown` / `no shutdown` | `shutdown` / `undo shutdown` (same verb, different negation keyword) |

The practical implication for SignalScope's prompt-detection/mode-tracking: VRP prompts are `<hostname>` (user view), `[hostname]` (system-view), `[hostname-GigabitEthernet0/0/1]` (interface view), `[hostname-vlan10]` (VLAN view) — square brackets replace the `#`/`(config)#` convention IOS uses, and the bracket content itself encodes the current context (interface name, VLAN ID, etc.), which is actually a *more* parseable signal than IOS's generic `(config-if)#`.

## SNMP version support

VRP S-series switches support **SNMPv1, SNMPv2c, and SNMPv3** concurrently (`snmp-agent sys-info version { v1 | v2c | v3 | all }`). SNMPv3 is the only version offering per-user auth (MD5/SHA, with newer software also offering SHA2-256) and privacy (DES56/3DES168/AES128/192/256).

**Practical write scope**: as with most vendors, SNMP SET support is narrower than the full config surface and should never be assumed available for a given action without checking `mib-reference.md`/the write-support matrix. VRP's SNMP write path is gated by two separate mechanisms that both must be satisfied for a SET to succeed:
1. The community (v1/v2c) or user's group (v3) must be granted **write** access at all — `snmp-agent community write <name> ...` / `snmp-agent group v3 <name> privacy write-view <view> ...`.
2. The specific OID being written must fall inside the **MIB view** bound to that community/group (see below) — a community can have "write" permission in principle yet still be unable to SET an object outside its bound view.

### `snmp-agent mib-view` — VRP's distinctive access-control mechanism

VRP implements a VACM-like (View-based Access Control Model) MIB-view mechanism directly in the CLI, independent of whether the underlying trap is v1/v2c/v3. This is worth calling out as a distinctive feature because it means **the same community string or SNMPv3 group can be scoped to only a subtree of the MIB tree** — e.g. an NMS integration granted "write" could be restricted to only `ifAdminStatus` and a VLAN table, with everything else (SNMP config itself, security objects, etc.) excluded, rather than the all-or-nothing read/write split many other vendors expose.

- `snmp-agent mib-view { excluded | included } <view-name> <oid-tree>` creates/extends a named view by including or excluding an OID subtree (either as dotted numeric OID or object name, e.g. `system.7`).
- The default view `ViewDefault` exists out of the box and **cannot be modified or deleted**.
- Views are then bound to v1/v2c communities via `snmp-agent community { read | write } <community-name> mib-view <view-name>`, or to v3 groups via `snmp-agent group v3 <group-name> { authentication | noauth | privacy } read-view <view> write-view <view> notify-view <view>`.

SignalScope should treat `snmp-agent mib-view`/community/group bindings as a first-class thing to read back and surface in the GUI when configuring SNMP access on a Huawei device — a "why did my SET fail" diagnostic needs to check view membership, not just community/group write permission. See [mib-reference.md](mib-reference.md) for full command detail.

## Config-apply model

VRP applies `system-view`/interface-view configuration lines **immediately** to the running configuration, the same immediate-apply model as Cisco/Arista/Aruba (as opposed to Juniper's candidate-config + `commit`). There is no separate "commit" step — a `port trunk allow-pass vlan 20` line takes effect on the live interface as soon as it's entered.

Persistence to the config used on next boot is a separate, explicit step: **`save`** (executed from user view or system-view). Confirmed behavior:
- `save` with no argument: if a "next-startup configuration file" is already set, prompts to confirm saving to it; if none is set yet, prompts for a filename (Enter accepts the default, `vrpcfg.zip`).
- `save <configuration-file>`: saves directly to the named file without the interactive filename prompt.
- Until `save` is run, changes survive a live session but are **lost on reboot/power-loss** — same practical caveat as Cisco's `write`/`copy run start`, and should be surfaced in the GUI's "Save" affordance identically to how the Cisco vendor doc treats it (immediate-apply, but persistence is a distinct explicit action the GUI must trigger and confirm, not assume).

## Paging control

`screen-length 0 temporary` disables `---- More ----` pagination for the **current session only** (the "temporary" keyword scopes it to the terminal session rather than persisting it as a saved config line) — confirmed against multiple Huawei command references and consistent with prior Netmiko/vendor-comparison usage. SignalScope should send this at session start over SSH/Telnet and echo it in the terminal per the "no hidden setup commands" principle (same as `connectivity-methods.md` already specifies for other vendors).

## Documentation sources used

- [SNMPv2-MIB — S-series MIB Reference](https://support.huawei.com/enterprise/en/doc/EDOC1100212421/7c6ad5b4/snmpv2-mib)
- [HUAWEI-PORT-MIB — S1720/S2700/S5700/S6720 MIB Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000178181/69628d40/huawei-port-mib)
- [Typical SNMP Configuration — S300/S500/S2700/S3700/S5700/S6700/S7700/S9700 Typical Configuration Examples](https://support.huawei.com/enterprise/en/doc/EDOC1000069520/dfaf8717/typical-snmp-configuration)
- [snmp-agent mib-view — Command Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000128405/9c1bc6fd/snmp-agent-mib-view)
- [display snmp-agent mib-view — Command Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000128405/c74a608d/display-snmp-agent-mib-view)
- [SNMP Configuration Commands — CloudEngine S3700/S5700/S6700 Command Reference](https://support.huawei.cn/enterprise/en/doc/EDOC1100368578/30eb7700/snmp-configuration-commands)
- [snmp-agent community — Command Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000128405/37b6f32c/snmp-agent-community)
- [snmp-agent target-host trap — Command Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000041693/4753c85b/snmp-agent-target-host-trap)
- [snmp-agent usm-user — Command Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000128405/72d3e0f1/snmp-agent-usm-user)
- [Saving the Configuration File — S1720/S2700/S5700/S6720 Configuration Guide](https://support.huawei.com/enterprise/en/doc/EDOC1000178166/4adec9f7/saving-the-configuration-file)
- [Typical VLAN Configuration — Typical Configuration Examples](https://support.huawei.com/enterprise/en/doc/EDOC1000069520/b699322c/typical-vlan-configuration)
- [How Do I Create VLANs in a Batch? — Configuration Guide](https://support.huawei.com/enterprise/en/doc/EDOC1000178168/a197c0b1/how-do-i-create-vlans-in-a-batch)
- [port vlan / VLAN Configuration Commands — Command Reference](https://support.huawei.com/enterprise/en/doc/EDOC1100333403/a0b2c054/port-vlan)
- [Assigning VLANs — Configuration Guide](https://support.huawei.com/enterprise/en/doc/EDOC1100277028/2580a0e4/assigning-vlans)
- [stp edged-port — Command Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000015892/80fee1e/stp-edged-port)
- [Configuring an Edge Port — Common Operation Guide](https://support.huawei.com/enterprise/en/doc/EDOC1000057410?section=j02g&topicName=configuring-an-edge-port)
- [Eth-Trunk Configuration Commands — CloudEngine S5700 Command Reference](https://support.huawei.com/enterprise/en/doc/EDOC1100291031/b7014cbe/eth-trunk-configuration-commands)
- [Example for Configuring an Eth-Trunk Interface in Static LACP Mode](https://info.support.huawei.com/hedex/api/pages/EDOC1100277644/AEM10221/03/resources/vrp/dc_vrp_ethtrunk_cfg_0061.html)
- [Link Aggregation Configuration — Typical Configuration Examples](https://support.huawei.com/enterprise/en/doc/EDOC1000069520/5d18b4c0/link-aggregation-configuration)
- [Port Security Configuration Commands — S1720/S2700/S5700/S6720 Command Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000178165/e2870031/port-security-configuration-commands)
- [Example for Configuring Port Security — Configuration Guide](https://support.huawei.com/enterprise/en/doc/EDOC1000178177/e7b747b0/example-for-configuring-port-security)
- [HUAWEI-L2VLAN-MIB — S1720/S2700/S5700/S6720 MIB Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000178181/62fde089/huawei-l2vlan-mib)
- [HUAWEI-CONFIG-MAN-MIB — S1720/S2700/S5700/S6720 MIB Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000178181/434f229f/huawei-config-man-mib)
- [MIB Overview — S1720/S2700/S5700/S6720 MIB Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000178181/2f6c0513/mib-overview)

Note on retrieval: Huawei's support portal (`support.huawei.com`) serves these pages as JavaScript-rendered SPAs — direct fetch of the raw HTML returns an empty shell, so the content above was recovered via cached search-result snippets rather than direct page fetch. Treat command syntax as high-confidence (it matches consistently across many independent command-reference pages for different device families/versions) but verify against a specific device's `V2xxRxxx` command reference before relying on exact parameter ranges or defaults, since Huawei versions its documentation per software release train and minor syntax details (e.g. available privacy algorithms, max VLAN batch count) do shift between VRP releases.

## MIB licensing/redistribution note

Huawei publishes MIB reference documentation on `support.huawei.com` (human-readable object descriptions per MIB, e.g. the `HUAWEI-PORT-MIB`/`HUAWEI-L2VLAN-MIB`/`HUAWEI-CONFIG-MAN-MIB` pages linked above) but this **does not include a stated open-redistribution license** for the underlying raw `.mib` ASN.1 files, unlike the IETF RFCs and IEEE 802.1AB LLDP-MIB vendored in `signal-scope-docs/mibs/`. Raw Huawei enterprise `.mib` files do circulate — e.g. mirrored in the [LibreNMS `mibs/huawei/`](https://github.com/librenms/librenms/tree/master/mibs/huawei) directory and browsable via [Observium's MIB browser](https://mibs.observium.org/mib/HUAWEI-L2VLAN-MIB/) — but the files themselves retain a `Copyright (C) HUAWEI TECHNOLOGIES. All rights reserved.` header with no redistribution grant; LibreNMS/Observium bundle them for their own tools' functional use, which is not the same as a public-domain or permissive-license release. Per the same caution applied project-wide, **this vendor's `mibs/` folder is link-only** — see [mibs/README.md](mibs/README.md) for the decision and links, no raw MIB text is copied into this repo.
