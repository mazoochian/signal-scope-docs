# CLI Syntax Matrix — Cross-Vendor Comparison

Referenced from [`cli-reference.md`](../vendors/arista/cli-reference.md) footnotes (Arista's "IOS parity" column) and from [`00-architecture/prior-art.md`](../00-architecture/prior-art.md)'s Netmiko discussion. This file rolls up the structural CLI differences documented per-vendor into one table, so SignalScope's session-state tracker and command-emission logic can be designed against the *shape* of the differences rather than re-deriving them from each vendor folder individually.

Scope: the seven original full-tier vendors — Cisco IOS/IOS-XE, Arista EOS, Extreme EXOS, Huawei VRP, Juniper Junos, HPE Aruba AOS-CX, MikroTik RouterOS — **plus Dell SmartFabric OS10** (added 2026-07-26; OS10 has full-tier-depth docs, `cli-reference.md`/`mib-reference.md`/`mibs/`, matching the original seven's research standard, so it joins the main tables below rather than a separate section). D-Link and Fortinet are still noted in a separate short section further down rather than given full matrix rows, since each spans multiple internal dialects that don't collapse into one row (see their `overview.md` files). Ubiquiti, Netgear, and Zyxel — appendix-tier vendors researched at a lighter confirmation standard — get their own [extended-appendix section](#appendix-tier-vendors-extended-comparison-added-2026-07-26) below the main tables rather than being merged into them, so the confidence-tier difference stays visible rather than implied by column position.

## Mode/context model

This is the single most consequential structural difference for SignalScope's session object (see [`00-architecture/gui-cli-snmp-unification.md`](../00-architecture/gui-cli-snmp-unification.md)'s "Session model" section) — it determines whether the session tracker needs a "current context" concept at all, and if so, whether that context is a linear stack or something else.

| Vendor | Model | Unprivileged/privileged split? | Config buffer vs. immediate-apply | Context re-entry needed per command? |
|---|---|---|---|---|
| Cisco IOS/IOS-XE | Mode-based: user EXEC → privileged EXEC → global config → sub-mode (config-if, etc.) | Yes (`enable`) | Immediate-apply; separate persist step (`copy run start`) | Yes — must `interface X` before an interface-scoped command if not already in that context |
| Arista EOS | Same shape as Cisco, deliberately IOS-compatible | Yes (`enable`) | Immediate-apply; separate persist (`write`/`copy running-config startup-config`) | Yes, identical pattern to Cisco |
| Extreme EXOS | **No persistent sub-mode at all** — every command is a complete, self-contained line naming its own scope (port list, VLAN name, STPD name) as an explicit argument | No | Immediate-apply; separate persist (`save configuration [primary\|secondary\|<file>]`) | **No** — this is the structural outlier; SignalScope's session tracker does not need a "current mode" concept for EXOS |
| Huawei VRP | Mode-based, similar shape to Cisco (system-view → interface/VLAN view) | Not by default (no separate `enable`; VRP uses privilege levels 0-15 instead) | Immediate-apply; separate persist (`save`, interactive Y/N prompt) | Yes, same pattern as Cisco |
| Juniper Junos | **Candidate-configuration model** — `configure` opens an edit buffer; `set`/`delete` mutate the *candidate*, not the running device; nothing takes effect until `commit` | No separate enable step (`configure` from operational mode) | **Candidate/commit** — categorically different from every other vendor here; `commit confirmed <n>` provides a Junos-native auto-rollback safety net with no analogue elsewhere except Aruba's checkpoint system (see below) | Yes, via `edit <hierarchy>` / `top` / `up` navigation, but changes aren't live until `commit` regardless of navigation |
| HPE Aruba AOS-CX | Mode-based, IOS-like shape (`configure terminal` → `interface`/`vlan` sub-modes) | Not by default at this layer (privileged access is separate from config-mode entry) | Immediate-apply, **plus** an optional layered `checkpoint auto <n>` guarded-apply/auto-rollback mechanism — conceptually similar in *purpose* to Junos `commit confirmed` but opt-in per-change rather than the default apply path | Yes, same pattern as Cisco |
| MikroTik RouterOS | **Menu-path hierarchy**, not mode-based — `/interface bridge port set ...` is one self-contained command whether issued from `/` or after `cd`-ing into `/interface bridge`; no "enter a context, then bare sub-commands" *requirement* (optional convenience, not structural) | No | Immediate-apply; **no running/startup-config split and no save step at all** — the only categorical "no persistence step exists" vendor in this table | No — fully-qualified path always produces identical result regardless of current menu position |
| Dell SmartFabric OS10 | Mode-based, IOS-adjacent shape (`configure terminal` → `interface`/`vlan` sub-modes) | No separate enable step confirmed this session — `configure terminal` reached directly, unlike Cisco's two-step (verify per-deployment, AAA could change this) | Immediate-apply by default, **plus** an opt-in `start transaction` (issued before `configure terminal`) that layers a Junos-like candidate/`commit`/`discard` model on top — a third config-apply pattern distinct from both Cisco's persist-only and Junos's mandatory candidate/commit, since it's opt-in rather than the default path | Yes, same pattern as Cisco, when not in transaction mode |

## Negation / removal keyword

| Vendor | Keyword | Example |
|---|---|---|
| Cisco / Arista | `no` | `no shutdown` |
| Extreme EXOS | `disable`/`delete` (verb-specific, no single negation particle) | `disable port 1:3`, `delete vlan accounting` |
| Huawei VRP | `undo` | `undo shutdown` |
| Juniper Junos | `delete` (removes a candidate statement) vs. `set` (adds/changes one) — not a per-line negation particle | `delete interfaces ge-0/0/1 disable` |
| Aruba AOS-CX | `no` (IOS-like) | `no shutdown` |
| MikroTik RouterOS | N/A — `disable`/`enable`/`remove` are the verbs themselves, not a particle prepended to another verb | `/interface disable ether1` |
| Dell SmartFabric OS10 | `no` (IOS-like) | `no shutdown` |

## Interface naming convention

| Vendor | Convention | Example |
|---|---|---|
| Cisco IOS/IOS-XE | `<Type><slot>/<port>` | `GigabitEthernet0/1` |
| Arista EOS | `Ethernet<n>` or `Ethernet<slot>/<port>`, no speed-encoded type prefix | `Ethernet1` |
| Extreme EXOS | `<slot>:<port>` | `1:3` |
| Huawei VRP | `<Type><stack-id>/<slot>/<port>` (three-part, not two-part like Cisco) | `GigabitEthernet0/0/1` |
| Juniper Junos | `<media>-<fpc>/<pic>/<port>`, plus a separate `unit` (logical interface) layer below it | `ge-0/0/1`, config under `ge-0/0/1 unit 0` |
| Aruba AOS-CX | `<member>/<slot>/<port>` | `1/1/1` |
| MikroTik RouterOS | Free-text name, either factory-default (`ether1`) or user-renamed; not a fixed positional scheme | `ether1`, or renamed via `/interface set ether1 name=uplink-core1` |
| Dell SmartFabric OS10 | `ethernet <node>/<slot>/<port>` (type-prefixed, three-part like Huawei/Aruba rather than Cisco's two-part) | `ethernet 1/1/2` |

## Paging control (session-setup command SignalScope must send and echo)

Per [`connectivity-methods.md`](../00-architecture/connectivity-methods.md), every vendor below requires an explicit one-time paging-disable command per session (except where noted), and SignalScope must **echo it in the terminal like any other command** — never a silent setup step.

| Vendor | Command | Persists across sessions? |
|---|---|---|
| Cisco IOS/IOS-XE | `terminal length 0` | No — per-session |
| Arista EOS | `terminal length 0` (identical text to IOS) | No — per-session |
| Extreme EXOS | `disable clipaging` | No — per-session (cannot be saved to persistent config) |
| Huawei VRP | `screen-length 0 temporary` | No — the `temporary` keyword is what makes it session-scoped; omitting it changes a persistent VTY setting instead, which SignalScope should avoid |
| Juniper Junos | `set cli screen-length 0` (operational mode, not config mode) | No — per-session |
| Aruba AOS-CX | `no page` | **Reportedly yes** (community-sourced, not independently confirmed against an official page this session — see `vendors/aruba/cli-reference.md`) — flagged as an open verification item, since if true it's the one vendor where this command has a side effect beyond the current session |
| MikroTik RouterOS | Not applicable — RouterOS console output is not paginated the way EXEC-style CLIs are | N/A |
| Dell SmartFabric OS10 | **No confirmed session-wide toggle** — the only confirmed mechanism is `show running-configuration \| no-more`, a **per-command pipe filter**, not a one-time session-start command like every other vendor here. A `terminal` command family exists (per OS10's "common commands" list) but the exact session-scoped pagination-disable syntax was not independently confirmed this session. | N/A (per-command, not session-scoped) |

## Save/persist command (the GUI "Save" affordance's literal CLI text)

This is the other structural fork documented in [`gui-cli-snmp-unification.md`](../00-architecture/gui-cli-snmp-unification.md) — three genuinely different semantics exist, not one:

| Vendor | Save semantics | Command |
|---|---|---|
| Cisco IOS/IOS-XE | Persist-only (running-config already live; this writes it to NVRAM) | `copy running-config startup-config` / `write memory` |
| Arista EOS | Same persist-only semantics as Cisco | `write` / `write memory` / `copy running-config startup-config` |
| Extreme EXOS | Persist-only, but **targets one of two named config slots** (`primary`/`secondary`) or a custom filename — not a single implicit target like Cisco's `startup-config` | `save configuration [primary\|secondary\|<name>]` |
| Huawei VRP | Persist-only, but **interactive** (Y/N confirmation, optional filename prompt) — SignalScope's terminal automation must handle this prompt, not fire-and-forget | `save` |
| Juniper Junos | **Commit, not persist** — this is the step that makes candidate-config changes take effect at all, categorically different from the other vendors' "already live, just persisting" model | `commit` (or `commit confirmed <n>` / `commit check`) |
| Aruba AOS-CX | Persist-only for the base case (`copy running-config startup-config` / `write memory`, IOS-identical), **plus** a distinct, separate "checkpoint" concept (`copy running-config checkpoint <name>`, `checkpoint auto <n>`) that has no equivalent on Cisco/Arista/Huawei and only a loose conceptual parallel to Junos's `commit confirmed` | `write memory` (persist) or `checkpoint auto <n>` (guarded apply) |
| MikroTik RouterOS | **No save concept at all** — every command is durably persisted the instant it runs; `/system backup save`/`/export` are backup/portability mechanisms, not a prerequisite for a change to survive reboot | *(nothing — no command needed)* |
| Dell SmartFabric OS10 | Persist-only for the immediate-apply default (IOS/EOS-shorthand style); **`commit` instead, when in opt-in transaction mode** (see mode/context row above) | `write memory` (default mode) / `commit` (transaction mode) — not independently confirmed this session whether a longer `copy running-configuration startup-configuration` form also exists alongside `write memory` |

## Port enable/disable (the most universal single action, for calibration)

| Vendor | Disable | Enable |
|---|---|---|
| Cisco / Arista | `interface <name>` → `shutdown` | `interface <name>` → `no shutdown` |
| Extreme EXOS | `disable port <list>` (single line, no context entry) | `enable port <list>` |
| Huawei VRP | `interface <name>` → `shutdown` | `interface <name>` → `undo shutdown` |
| Juniper Junos | `set interfaces <name> disable` then `commit` | `delete interfaces <name> disable` then `commit` — note this is a *deletion*, not a positive enable statement |
| Aruba AOS-CX | `interface <name>` → `shutdown` | `interface <name>` → `no shutdown` |
| MikroTik RouterOS | `/interface disable <name>` (single line) | `/interface enable <name>` |
| Dell SmartFabric OS10 | `interface ethernet 1/1/2` → `shutdown` | `interface ethernet 1/1/2` → `no shutdown` |

## D-Link / Fortinet standalone (not given full matrix rows — see their own `overview.md`)

- **D-Link**: spans *two* internal CLI families that don't collapse into one row — classic xStack (EXOS-like, no persistent context, `save` bare command) vs. newer "Cisco-like" (`interface range`, `switchport`, `copy running-config startup-config`) depending on product line. See `vendors/dlink/overview.md`'s dialect table.
- **Fortinet (FortiSwitch), standalone CLI**: a third structural pattern not represented above — nested `config`/`edit`/`next`/`end` block syntax (stack-based context, explicit close per level) with **auto-save on `end`** (no separate persist command exists at all, the inverse of MikroTik's "no save needed" for a different reason — Fortinet auto-persists on block-close, MikroTik never needs persisting because there's no buffer distinction to begin with). See `vendors/fortinet/overview.md`. **Fortinet also has a second, structurally distinct CLI path** — FortiGate-mediated (`config switch-controller managed-switch`), the dominant real-world deployment shape — documented separately in [`vendors/fortinet/fortilink-integration.md`](../vendors/fortinet/fortilink-integration.md) rather than folded into this table, since the session target there is a different device (the FortiGate) than the one being configured (the FortiSwitch), a shape none of the tables above accommodate.

## Appendix-tier vendors — extended comparison (added 2026-07-26)

Ubiquiti, Netgear, and Zyxel, researched to a lighter confirmation standard than the main tables above (search-summary-sourced in large part, explicitly flagged per-vendor in each `overview.md`). Kept as a separate section rather than merged into the main tables so the confidence-tier difference stays visible. Cells read "not confirmed" where the source vendor doc says so explicitly — this section does not invent or infer syntax to fill gaps.

### Mode/context model

| Vendor | Model | Config buffer vs. immediate-apply | Notes |
|---|---|---|---|
| Ubiquiti (UniFi, direct-device CLI) | Mode-based, IOS-adjacent verb choice (`configure`, `interface 0/1` sub-mode) once reached — but reaching it requires SSH into a Linux shell, then `telnet 127.0.0.1`, then `enable`, a materially different session-establishment shape from every vendor in the main tables | Immediate-apply (assumed from IOS-adjacent shape; not separately re-confirmed) | **Not durable regardless of syntax** — the UniFi Network Controller overwrites CLI-typed changes on next reboot/reprovision. This is a durability problem, not a syntax one — see `vendors/ubiquiti/overview.md`. |
| Netgear (M-series / Intelligent Edge scope) | Mode-based, IOS-adjacent (`configure`, interface sub-modes) | Immediate-apply (presumed by convention; not independently re-confirmed this session) | GS-series/"Smart Managed" tier has no officially-supported CLI at all — this row applies only to the M-series/Intelligent Edge line; see product-tier table in `vendors/netgear/overview.md`. |
| Zyxel (standalone-managed line scope) | Mode-based, IOS-adjacent: `configure` (not `configure terminal` — shorter form confirmed for this vendor) | Immediate-apply (presumed; not independently re-confirmed) | GS1900 tier has **no config-mode CLI path at all** (`configure terminal` returns `Unknown command`, confirmed directly, not inferred) — this row applies only to the standalone-managed line (XGS4600/GS19x0/GS2210/GS3700); see `vendors/zyxel/overview.md`. |

### Negation keyword, interface naming, save command

| Vendor | Negation keyword | Interface naming | Save/persist command |
|---|---|---|---|
| Ubiquiti (UniFi) | Not confirmed as a single particle — VLAN config uses distinct verbs (`vlan participation include/exclude`, separate from `vlan tagging`) rather than a negation particle prepended to one verb | `<unit>/<port>` (FASTPATH-style), e.g. `0/1` | **N/A / actively discouraged** — no durable save exists on this path; the controller is the only durable target. |
| Netgear (M-series) | `no` (IOS-like, presumed by convention; not independently re-confirmed) | `<unit>/<slot>/<port>`, confirmed for M4300, e.g. `1/0/1` | `copy system:running-config nvram:startup-config` — **confirmed**, distinctive `system:`/`nvram:` filesystem-prefix notation not seen on any main-table vendor. |
| Zyxel (standalone-managed line) | Not independently confirmed this session | Not independently confirmed this session (bare port number vs. `port <n>` vs. slot/port scheme all unresolved) | `write memory` — **confirmed**, same shorthand-save convention as Cisco/Arista/Dell OS10. |

### Port enable/disable

| Vendor | Disable | Enable | Confidence |
|---|---|---|---|
| Ubiquiti (UniFi) | Not independently confirmed this session (exact FASTPATH keyword unresolved) | Not independently confirmed this session | Low — flagged as an open item in `vendors/ubiquiti/overview.md`. |
| Netgear (M-series) | `interface 1/0/1` → `shutdown` (assumed from IOS-adjacent shape) | `interface 1/0/1` → `no shutdown` (assumed) | Medium — inferred from CLI family, not independently re-confirmed this session. |
| Zyxel (standalone-managed line) | Not independently confirmed this session | Not independently confirmed this session | Low — exact keyword/interface-naming pattern unresolved, per `vendors/zyxel/overview.md`. |

**Cross-vendor pattern worth noting**: Ubiquiti's direct-device CLI and Netgear's interface-naming/VLAN-command-family shape both trace to the same underlying **Broadcom "FASTPATH" reference-CLI lineage** — flagged as an inference (structural similarity), not a confirmed shared codebase, in both vendors' `overview.md` files. This is a real cross-vendor pattern (several SMB/mid-market switch vendors license the same reference stack) worth keeping in mind when reasoning about any *other* unresearched SMB vendor's CLI shape generically, though it should not be treated as confirmation of exact syntax for either vendor specifically.

## Open verification items

- Aruba's `no page` persistence-beyond-session claim (community-sourced only).
- Whether `show running-config | section X` (Cisco/Arista pipe-to-section) vs. EOS's `show running-config section X` (no pipe) is a real syntax divergence or documentation inconsistency — flagged in `vendors/arista/cli-reference.md`, not resolved here.
- Dell OS10's session-wide paging-disable syntax (only the per-command `| no-more` filter is confirmed).
- Ubiquiti/Netgear/Zyxel's port enable/disable keywords and Zyxel's interface-naming scheme, per the extended-appendix section above — all flagged "not independently confirmed" in their source `overview.md` files.
