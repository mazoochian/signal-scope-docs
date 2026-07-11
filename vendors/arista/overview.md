# Arista EOS — Overview

## CLI dialect

EOS's CLI is **deliberately modeled on Cisco IOS syntax** — Arista's founders (ex-Cisco) built EOS this way specifically to lower switching costs for network engineers already fluent in IOS. This is not superficial: mode structure (user EXEC → privileged EXEC → global config → interface config), prompt shape (`hostname>`, `hostname#`, `hostname(config)#`, `hostname(config-if)#`), and a large fraction of command verbs (`interface`, `shutdown`, `description`, `switchport`, `spanning-tree`, `channel-group`, `snmp-server`) are near-identical to IOS. This matters directly for SignalScope's [cli-syntax-matrix](../../comparison/cli-syntax-matrix.md): Arista and Cisco should be treated as the closest pair in that matrix, with divergences called out explicitly per-command rather than assumed. See [cli-reference.md](cli-reference.md) for a per-command IOS-parity/divergence note.

Known divergence points worth flagging up front (detailed further in cli-reference.md):
- Interface naming has no slot/module dash-slash convention on most fixed-config EOS switches — commonly `Ethernet1`, `Ethernet1/1` (modular) rather than IOS's `GigabitEthernet0/1`.
- Port-channel interface creation and `channel-group` syntax are close to IOS but see notes on `mode on` (static, LACP-disabled) vs `active`/`passive`.
- EOS adds config-session / "config sessions" (candidate-config-like commit workflow, `configure session <name>`) as an *optional* feature layered on top of the otherwise IOS-style immediate-apply model — SignalScope should treat the default (no session) EOS behavior as immediate-apply like IOS, per the architecture docs, and treat config sessions as an opt-in feature to model separately if ever surfaced in the GUI.

## SNMP version support and write scope

EOS supports all three SNMP versions:
- **SNMPv1** (RFC 1157) — community-string based.
- **SNMPv2c** (RFC 1901/1905/1906) — community-string based, adds GetBulk/64-bit counters.
- **SNMPv3** (RFC 2273–2275) — USM security, configurable auth/priv.

Community strings can be configured `ro` (read-only) or `rw` (read-write) via `snmp-server community`, and SNMPv3 groups can be given `read`/`write` view names via `snmp-server group` — so the *agent* configuration surface nominally supports SET. However, per Arista's own SNMP MIBs documentation page, **"all MIB support is read-only unless otherwise noted"** across both standard and Arista enterprise MIBs — the exceptions called out are narrow (notably `IF-MIB::ifAdminStatus` and `IF-MIB::ifAlias` are documented as writable). SignalScope should treat Arista as a **read-only-leaning SNMP posture by default**: configuring `rw` community strings/views is possible, but there are very few actual writable objects behind that access for switch-config purposes beyond interface admin-status/description. See [mib-reference.md](mib-reference.md) and [gui-cli-snmp-mapping.md](gui-cli-snmp-mapping.md) for the practical consequence — most GUI actions on Arista devices have no realistic SNMP SET path and must go through the CLI session.

## Config-apply model

Like Cisco IOS, EOS applies each config-mode line **immediately** to the running-config as it's typed/sent — there is no separate "candidate config" staging step for the default CLI mode. Persistence to non-volatile storage requires an explicit step:
- `write` or `write memory` — EOS shorthand, copies running-config to startup-config.
- `copy running-config startup-config` — the fuller/IOS-familiar form of the same action.

Both are equivalent; SignalScope should treat either as the literal "Save" command to echo into the terminal (see [gui-cli-snmp-mapping.md](gui-cli-snmp-mapping.md)).

EOS also supports an optional **configuration session** mechanism (`configure session <name>` ... `commit`) that layers a Juniper-like candidate/commit workflow on top of the CLI, but this is opt-in — not the default interactive mode. SignalScope should assume immediate-apply unless a device session is explicitly using `configure session`.

### eAPI (Command API) — noted, not primary

EOS additionally exposes **eAPI**, a JSON-RPC-over-HTTP(S) management channel (`POST /command-api`) that accepts a list of literal CLI command strings (e.g. `['enable', 'configure', 'interface Ethernet1', 'shutdown']`) and returns structured JSON results per command. This is architecturally interesting because it's *also* CLI-command-literal (the wire format is the same command text a human would type, just batched and returned as JSON rather than screen-scraped) — but per SignalScope's CLI-literal philosophy, the primary automation path for GUI actions should still be the live interactive SSH/Telnet CLI session (so the terminal pane shows a real, live, human-equivalent session), not a side-channel HTTP API, even though eAPI would be a legitimate lower-friction alternative for a future non-terminal-coupled integration. Noted here for completeness; not treated as a first-class SignalScope channel for Arista in the current architecture.

## Official documentation sources

- [EOS SNMP user manual](https://www.arista.com/en/um-eos/eos-snmp) — SNMP versions, community/user/host/view configuration, verified this session.
- [Arista SNMP MIBs page](https://www.arista.com/en/support/product-documentation/arista-snmp-mibs) — MIB downloads (41 Arista enterprise MIBs + standard IETF/IEEE MIB support list), "read-only unless otherwise noted" statement, verified this session.
- [EOS SNMP commands section (43.4)](https://www.arista.com/en/um-eos/eos-section-43-4-snmp-commands) — command syntax reference; page returned "not found" on direct fetch this session but is indexed and referenced from the SNMP chapter above; command syntax cross-checked via the main SNMP page and search-indexed excerpts instead.
- [EOS Interface Configuration chapter](https://www.arista.com/en/um-eos/eos-interface-configuration) — chapter index for Ethernet ports / port-channel / MLAG sections.
- [EOS Port Channels and LACP](https://www.arista.com/en/um-eos/eos-port-channels-and-lacp) — `channel-group`/LACP mode syntax, verified this session.
- [EOS Spanning Tree Protocol](https://www.arista.com/en/um-eos/eos-spanning-tree-protocol) — mode/portfast/cost/priority syntax, verified this session.
- [EOS Virtual LANs (VLANs)](https://www.arista.com/en/um-eos/eos-virtual-lans-vlans) — VLAN creation, access/trunk switchport syntax, verified this session.
- [EOS Configuration Files](https://www.arista.com/en/um-eos/eos-configuration-files) — running/startup-config, `write`/`copy running-config startup-config`, verified this session.
- [EOS Command-Line Interface (CLI) chapter](https://www.arista.com/en/um-eos/eos-command-line-interface-cli) — general CLI behavior including paging (`terminal length 0`).
- [Arista eAPI whitepaper](https://www.arista.com/assets/data/pdf/Whitepapers/Arista_eAPI_FINAL.pdf) and [Arista Community: How to use the JSON-RPC API](https://eos.arista.com/how-to-use-the-json-rpc-api/) — eAPI/Command API protocol description.
- Port security: referenced from Arista Community/EOS data-transfer documentation search results (`switchport port-security`, `switchport port-security mac-address maximum`); exact chapter URL not confirmed this session — treat command names as directionally correct but verify exact syntax/defaults against a live device or the current release's user manual before relying on flags beyond what's listed in [cli-reference.md](cli-reference.md).

## MIB licensing / redistribution note

Arista's MIB downloads are served directly from `arista.com` (e.g. `https://www.arista.com/assets/data/docs/MIBS/ARISTA-SMI-MIB.txt`). The file itself carries only a bare notice:

> Copyright (c) 2008 Arista Networks, Inc. All rights reserved.

No explicit redistribution grant, open-source license (e.g. BSD/MIT-style permission text), or terms-of-use statement was found accompanying the MIB text files or the MIB download listing page. A bare "all rights reserved" copyright notice without an accompanying redistribution grant should be read as **no redistribution permission implied** — vendoring these files into this repo would be legally ambiguous at best.

**Decision: link-only, no vendoring.** [mibs/](mibs/) is left empty (or contains only a pointer file) rather than raw `.mib`/`.txt` copies. If this needs revisiting, the accurate path is to request explicit redistribution permission from Arista or confirm a clearer license statement in a later EOS release's MIB package before vendoring.
