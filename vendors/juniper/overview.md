# Juniper (Junos OS) — EX/QFX Series Switches — Overview

## CLI dialect

Junos CLI is **not** a Cisco-IOS-style dialect. It's structurally its own thing, loosely modeled on Unix shell conventions (pipe `|` for output filtering, e.g. `show interfaces terse | match ge-0`), with two top-level modes:

- **Operational mode** (`>` prompt): `show`, `ping`, `traceroute`, `monitor`, `request` (reboot, software install), `file`. Read-only against the device; no config changes happen here.
- **Configuration mode** (`#` prompt, entered via `configure`): a **hierarchical, tree-structured** configuration edited with `set` / `delete` / `edit` / `up` / `top` / `show` (config-mode `show` previews the candidate tree, distinct from operational-mode `show`). There is no Cisco-style `interface X` / `no shutdown` per-line-apply model — see "Config-apply model" below, which is the single most important structural difference to internalize.

Interface names are physical-slot-encoded: `ge-0/0/1` (`type-fpc/pic/port`), `xe-0/0/0` (10GbE), `ae0` (aggregated Ethernet), `irb.0` (routed VLAN interface). Logical units (`unit 0`) sit under a physical interface, which is where Layer 2/3 family config actually attaches (`ge-0/0/1 unit 0 family ethernet-switching ...`) — a switch port's Layer 2 config is one level deeper than the physical interface itself.

## SNMP version support

Junos supports **SNMPv1, SNMPv2c, and SNMPv3**, confirmed directly in Juniper's own FAQ:

> "Junos OS supports SNMP version 1 (SNMPv1), version 2 (SNMPv2c), and version 3 (SNMPv3)."
> — [Junos OS SNMP FAQs](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/junos-os-snmp-faqs.html)

### What's actually writable via SNMP SET — read this before assuming parity with Cisco

This is the second load-bearing structural fact for SignalScope, after the commit model below:

- SNMP `SET` requires a community (or SNMPv3 user) explicitly configured with `authorization read-write`, plus a `view` statement enumerating which OID subtrees are writable — **the default view is read-only for everything**; nothing is writable by default even once a community exists. ([SNMP Communities](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/snmp-communities.html), [community (SNMP) statement](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/ref/statement/community-edit-snmp.html))
- Even with a read-write community configured, Juniper's own FAQ explicitly rules out the single most common cross-vendor SNMP write action:
  > "SNMP is not allowed to set the ifAdminStatus."
  > — [Junos OS SNMP FAQs](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/junos-os-snmp-faqs.html)
  This means the `IF-MIB::ifAdminStatus` SET that SignalScope's cross-vendor "enable/disable port" GUI action relies on as a fallback (per [gui-cli-snmp-unification.md](../../00-architecture/gui-cli-snmp-unification.md)) **does not work on Junos at all** — there is no SNMP path for this action; it must always go through a live CLI session for Juniper devices.
  - Confirmed by third-party field reports too, e.g. a Juniper community thread on the exact symptom: [Port ifAdminStatus Down via SNMP](https://community.juniper.net/t5/Ethernet-Switching/Port-ifAdminStatus-Down-via-SNMP/td-p/276511).
- The FAQ page does document a small set of objects that **are** SET-able where a read-write view is configured — things like `snmpCommunityTable`, RMON `eventTable`/`alarmTable` rows, and the classic `sysContact.0`/`sysName.0`/`sysLocation.0` scalars. This is narrow, and none of it is switch-port-configuration surface (VLANs, trunking, STP, LACP).
- **Practical conclusion for SignalScope**: treat Junos as an SNMP agent that is effectively **read-only for switch configuration purposes**. See [mib-reference.md](mib-reference.md) for the full breakdown and why real Junos automation for config changes uses **NETCONF**, not SNMP SET.

## Config-apply model — candidate configuration + explicit commit

**This is structurally different from every immediate-apply vendor (Cisco/Arista/Huawei/Aruba) and must be reflected explicitly in SignalScope's GUI semantics.**

- Every `set` / `delete` / `edit ... set` command issued in configuration mode modifies only the **candidate configuration** — an in-memory/staged copy. It has **zero effect on the device's running behavior** until a `commit` is issued.
- `commit` — validates and activates the candidate configuration as the new active (and default-boot) configuration. Prior configs are retained in a numbered rollback history (up to 50 by default) via `show system commit` / `rollback <n>`.
- `commit confirmed <minutes>` — activates the candidate config **temporarily** (default window 10 minutes if no value given); if a second `commit` (or `commit check`) isn't issued within the window, Junos automatically reverts to the prior configuration. This is the safety mechanism for risky changes (e.g. anything that could cut off the management session) and has no Cisco equivalent.
- `commit at "<time>"` — schedules activation for a future time or upon next reboot, without activating now.
- `commit check` — validates candidate config syntax/semantics without activating it; does not start a confirm timer and does not affect rollback history.
- `rollback <n>` — loads a previous committed configuration into the candidate buffer (still requires a subsequent `commit` to take effect); `rollback 0` discards uncommitted candidate changes entirely.
- ([`commit` command reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/commit.html), [Commit the Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html), [`rollback` command reference](https://www.juniper.net/documentation/en_US/junos/topics/reference/command-summary/rollback.html))

**Why this matters for SignalScope's GUI**: on every other vendor in this documentation set, a GUI "Apply" action corresponds to CLI lines that take effect immediately (persistence to NVRAM is a separate, optional `write`/`copy run start` step, and even without it the change is live). On Junos, a GUI "Apply"/"Save" action **is not done, and has had no effect on the device at all**, until the terminal shows a `commit` line succeed. SignalScope must:
1. Never claim a Junos change "applied" based on the `set` lines alone.
2. Always show the `commit` (or `commit confirmed N`) line as a distinct, visible step in the terminal — not folded silently into the GUI action — per [gui-cli-snmp-mapping.md](gui-cli-snmp-mapping.md).
3. Strongly prefer `commit confirmed` as the default for any change touching interface/management reachability, surfacing the confirm window and auto-rollback risk in the UI.

## Official documentation sources used

- [SNMP MIB Explorer](https://apps.juniper.net/mib-explorer/) — Juniper's tool for browsing/searching supported MIB objects per platform/release.
- [SNMP MIBs Supported by Junos OS and Junos OS Evolved](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/snmp-mibs-supported-by-junos-os-and-junos-os-evolved.html) — canonical list of standard MIBs Junos implements.
- [Enterprise-Specific MIBs Overview](https://www.juniper.net/documentation/en_US/junos/topics/concept/enterprise-specific-mibs-overview.html) — Juniper's own enterprise MIB catalog (JUNIPER-MIB family).
- [CLI Explorer](https://apps.juniper.net/cli-explorer/) — Juniper's searchable CLI command/statement reference across releases and platforms.
- [Junos OS SNMP FAQs](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/junos-os-snmp-faqs.html) — source for the SNMP version support and `ifAdminStatus`/write-scope findings above.
- [SNMP Communities](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/snmp-communities.html) and [`community` (SNMP) statement](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/ref/statement/community-edit-snmp.html) — read-write view/authorization scoping.
- [`commit` command reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/commit.html) and [Commit the Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html) — candidate-config/commit model.
- [Interfaces User Guide for Switches](https://www.juniper.net/documentation/us/en/software/junos/interfaces-ethernet-switches/index.html) and [Configuring Gigabit Ethernet Interfaces (CLI Procedure)](https://www.juniper.net/documentation/en_US/junos/topics/task/configuration/ex-series-gigabit-interfaces-cli.html) — EX-series interface configuration.
- [Ethernet Switching User Guide (Bridging and VLANs)](https://www.juniper.net/documentation/us/en/software/junos/multicast-l2/topics/topic-map/bridging-and-vlans.html) and [Configuring VLANs for EX Series Switches (CLI Procedure)](https://www.juniper.net/documentation/en_US/junos12.3/topics/task/configuration/bridging-vlans-ex-series-cli.html) — VLAN/trunk configuration.
- [Overview of Port Security](https://www.juniper.net/documentation/us/en/software/junos/security-services/topics/topic-map/overview-port-security.html) and [`secure-access-port` statement](https://www.juniper.net/documentation/us/en/software/junos/fcoe-ex/security-services/topics/ref/statement/secure-access-port-port-security.html) — EX port-security feature set.
- [Configuring RSTP](https://www.juniper.net/documentation/us/en/software/junos/stp-l2/topics/topic-map/spanning-tree-configuring-rstp.html), [Configuring MSTP](https://www.juniper.net/documentation/us/en/software/junos/stp-l2/topics/topic-map/spanning-tree-configuring-mstp.html), [`edge` (Spanning Trees) statement](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/edge-spanning-trees-ex-series.html).
- [Aggregated Ethernet Interfaces Overview](https://www.juniper.net/documentation/us/en/software/junos/interfaces-ethernet/topics/topic-map/aggregated-ethernet-interfaces-lacp-configure.html) — LACP/`ae` interface configuration.
- [Configure SNMPv3](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/configure-snmpv3.html) and [Configure the SNMPv3 Authentication Type and Encryption Type](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/configure-the-snmpv3-authentication-type-and-encryption-type.html).

## MIB licensing/redistribution note — why `mibs/` is link-only for Juniper

Juniper does not publish its MIB `.mib`/`.txt` files under a clear open redistribution license from a canonical public location; they're distributed through the [MIB Explorer](https://apps.juniper.net/mib-explorer/) tool and the support portal (`supportportal.juniper.net`), both of which require a Juniper account/session and don't carry an explicit "redistribution permitted" grant.

As instructed, we checked whether an open-source NMS project mirrors Juniper's MIBs under clearer terms:

- **LibreNMS** does vendor a set of Juniper MIB files directly in its repo (e.g. [`librenms-mibs/JUNIPER-MIB`](https://github.com/librenms/librenms-mibs/blob/master/JUNIPER-MIB), and `mibs/junos/JUNIPER-VLAN-MIB`, `JUNIPER-SMI`, etc. in the main [librenms/librenms](https://github.com/librenms/librenms) repo). However, inspecting the actual file content shows each carries Juniper's own header:
  ```
  -- Juniper Enterprise Specific MIB: Chassis MIB
  --
  -- Copyright (c) 1998-2008, Juniper Networks, Inc.
  -- All rights reserved.
  --
  -- The contents of this document are subject to change without notice.
  ```
  This is a Juniper copyright notice with **no explicit redistribution grant** (contrast with MIBs that carry an IETF/RFC boilerplate permission notice, or a vendor notice like "permission is granted to reproduce this document"). LibreNMS's own repo license (GPLv3-family) governs LibreNMS's *code*, not the legal status of this bundled third-party text, and doesn't resolve the ambiguity — it's the same "all rights reserved, no visible grant" situation as pulling straight from Juniper's own portal, just already downloaded.
- **Observium** also has JUNIPER-MIB/JUNIPER-VLAN-MIB pages in its public [MIB browser](https://mibs.observium.org/mib/JUNIPER-MIB/), same provenance.

**Decision: do not vendor raw Juniper MIB files under `mibs/`.** Neither source resolves the licensing ambiguity the assignment flagged — they're re-hosting the same Juniper-copyrighted text, not offering a cleaner grant. This matches the assignment's expectation that link-only is a fine, expected outcome for this vendor. See [mibs/README.md](mibs/README.md) for pointers instead of vendored files, and [mib-reference.md](mib-reference.md) for the OID-level documentation (sourced from Juniper's own doc pages and cross-checked against the LibreNMS/Observium mirrors for accuracy, without redistributing their file contents).
