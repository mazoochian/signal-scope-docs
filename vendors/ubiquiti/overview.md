# Ubiquiti (UniFi Switch line) — Overview

*Appendix-tier vendor doc — single file, less depth than Tier 1-3 vendors. Confirm exact syntax/OIDs against the official references linked below before relying on them for a live implementation.*

**Framing decision (per [`ROADMAP.md`](../../ROADMAP.md) Phase 0, 2026-07-16): controller-first.** Every other vendor in this docs tree is documented CLI-first, per SignalScope's core "the CLI is never abstracted away" architectural bet (see [`gui-cli-snmp-unification.md`](../../00-architecture/gui-cli-snmp-unification.md)). UniFi switches are the one vendor in this project where that framing doesn't fit the way the hardware is actually operated — see the next section for why, and read it before the CLI/API tables below, since it changes what those tables mean for SignalScope's architecture.

## Why controller-first, and a real architectural conflict this creates

UniFi switches are designed to be adopted by, and continuously managed from, a **UniFi Network Controller** (self-hosted software, a Cloud Gateway/UDM-family appliance, or Ubiquiti's hosted cloud instance) — not administered by an engineer hand-typing CLI on each switch the way every other vendor in this docs tree is. This isn't a soft preference; it's enforced by the device's own behavior:

- SSH access to a UniFi switch drops you into a **Linux shell**, not directly into a network-OS CLI — reaching the actual switch CLI requires an extra step (telnet to `127.0.0.1` from within that shell, then `enable`). The underlying CLI, once reached, is **EdgeSwitch-derived** — recognizably a Broadcom "FASTPATH" reference-CLI dialect (`vlan participation include/exclude`, `vlan pvid`, `vlan tagging`, `vlan acceptframe`), a command family also seen on other ODM/whitebox switches built on the same reference stack, not a Ubiquiti-original grammar.
- **Critically: CLI changes made this way are not durable.** Multiple independent sources (community threads, third-party how-to guides) agree that manual CLI configuration is overwritten by the controller on the switch's next reboot or re-provisioning cycle — the controller treats its own stored config as authoritative and pushes it back down, silently discarding anything typed directly on the device in the meantime.
- **This is a direct conflict with SignalScope's core architectural assumption.** [`gui-cli-snmp-unification.md`](../../00-architecture/gui-cli-snmp-unification.md) is built around a "unified, bidirectionally-visible session" where a human typing raw CLI and the GUI issuing commands are both durable, both visible, and both reconcile into the same state. For UniFi, a raw-CLI-typed change is **not durable** — it's a temporary override that the controller will erase, which breaks the "CLI/manual → GUI visibility" direction's implicit assumption that what a human types *is* the new state going forward. If SignalScope ever manages a real UniFi fleet, this needs explicit architectural treatment (e.g., surfacing CLI-typed changes on UniFi devices as "temporary, will be reverted on next provision" rather than treating them the same as a durable Cisco/Arista config change) — flagged here for a future update to the architecture doc, not resolved in this vendor-research pass.

Given this, the **UniFi Network Controller's own API** — not direct device SSH — is the practical integration point for anything SignalScope wants to be durable, and is documented here as the primary path.

## UniFi Network Controller API

Two distinct APIs exist, and a future reader should not conflate them:

- **The official UniFi Network API** (rolled out as a first-party, documented API — accessible per-controller under UniFi Network → Integrations, generating a local API key). This is Ubiquiti's now-supported integration surface. This session could not directly fetch the full reference page (`help.ui.com` blocked automated fetch), so exact endpoint shapes are **not independently confirmed** here — treat as "exists, is the recommended path, needs a follow-up read of the live reference for endpoint-level detail."
- **The older, undocumented/reverse-engineered controller REST API** (`api/s/{site}/stat/device`, `api/s/{site}/rest/...`, session-cookie auth via a login POST) — widely used by third-party tools (Home Assistant integrations, monitoring scripts, community Python clients like `unificontrol`) for years before the official API existed, and still the more thoroughly community-documented of the two. Ubiquiti's own community wiki explicitly warns this API "is undocumented, and response structures and endpoint behavior may change without notice between controller versions." **UDM/UDM-Pro-family controllers require an additional `/proxy/network` path prefix** on every endpoint relative to a standalone controller — a real, confirmed gotcha for anything targeting a UDM-hosted controller specifically.
- **Practical recommendation for SignalScope**: prefer the official API where its endpoint coverage is sufficient (VLAN/port config is confirmed to exist via the older API at minimum — `api/s/{site}/rest/...`-style endpoints); fall back to the older REST API only where the official one doesn't yet cover a needed action, and treat that fallback path as more fragile (unversioned, can change without notice) than any CLI or SNMP surface documented elsewhere in this project.

## Direct device access (secondary path — see durability caveat above)

- **SSH**: reachable using credentials configured in the controller (Settings → System → SSH Keys, or a per-site override) — **not** independent local credentials the way every other vendor's device access works. If a device is factory-adopted or the controller changes its SSH credential policy, previously-known credentials can stop working.
- **CLI shape once reached** (via the SSH → localhost-telnet → `enable` path noted above): mode-based, IOS-adjacent in verb choice (`configure`, `interface 0/1`, `vlan ...`) but the FASTPATH-derived VLAN command family (`vlan participation include/exclude <vlan-list>`, `vlan pvid <n>`, `vlan tagging <vlan-list>`, `vlan acceptframe {admituntaggedonly|all}`) is structurally distinct from every other vendor documented in this project — participation (which VLANs a port is a member of at all) and tagging (which of those are sent tagged) are configured as separate steps, rather than Cisco-style single-command access/trunk mode selection.
- **Do not build durable SignalScope automation on this path** per the persistence caveat above — appropriate uses are read-only diagnostics (`show` commands) or genuinely temporary interventions where a human understands the change won't survive the next provisioning cycle.

## SNMP

- **Enterprise PEN confirmed: 41112** (`ubnt OBJECT IDENTIFIER ::= { enterprises 41112 }`), directly from Ubiquiti's own `UBNT-MIB.txt`, hosted at `dl.ubnt.com` (Ubiquiti's own firmware/download server, no authentication wall — confirmed by direct fetch this session).
- **SNMP is controller-configured, not device-configured** — per Ubiquiti's own "SNMP Monitoring in UniFi Network" help article (title confirmed via search; full content not directly fetchable this session, blocked by the same access restriction as the official API page), SNMP is turned on and versioned from the controller's global settings and pushed to adopted devices, consistent with the controller-first management model described above.
- **The vendored `UBNT-MIB.txt` module is read-only across its entire researched contents** — a full read of the file found **zero `MAX-ACCESS read-write` objects** in 2,301 lines covering AirFiber/AirMAX/EdgeMax/UniFi/AirVision/mFi/UniTel product groups. The `ubntUniFi` sub-tree (`{ ubntMIB 6 }` = `1.3.6.1.4.1.41112.1.6`) is declared but was **not populated with any objects** in the specific firmware-companion file fetched this session (an AirFiber-firmware-bundled copy, not a UniFi-switch-specific one) — a UniFi-switch firmware's own bundled copy of this MIB likely has a richer `ubntUniFi` branch; this session's finding should be read as "the shared cross-product MIB module is read-only," not as "UniFi switches expose no enterprise MIB objects at all."
- **Practical write scope for SignalScope**: treat as **read-only/monitoring**, consistent with the confirmed absence of any read-write object in the module researched, and consistent with the general controller-first management model — there is no reason to expect a device that won't even durably keep CLI-typed changes to expose a working, independent SNMP write surface.
- MIB text not vendored in this docs tree (per project convention — link only, see below), though note Ubiquiti's own distribution (a plain file on `dl.ubnt.com`, no login, no explicit license statement found) is a **clearer first-party redistribution signal than most other vendors in this project** — worth revisiting if this vendor is ever promoted to full tier and a `mibs/` subfolder becomes worthwhile.

## Curated CLI table (direct-device EdgeSwitch-derived CLI — secondary path, see durability caveat)

| Action | Command(s) | Notes |
|---|---|---|
| Enter config mode | `configure` (from the `enable`-privileged prompt reached via the SSH→localhost-telnet path) | |
| Enter interface context | `interface 0/1` | FASTPATH-style `<unit>/<port>` naming. |
| Set VLAN participation (membership) | `vlan participation include 10` (inside interface context) / `vlan participation exclude 10` | Separate step from tagging — see overview note above. |
| Set tagged VLANs | `vlan tagging 10` (inside interface context) | |
| Set PVID (untagged/native VLAN) | `vlan pvid 10` (inside interface context) | |
| Set frame-acceptance policy | `vlan acceptframe {admituntaggedonly\|all}` | `admituntaggedonly` ≈ access-port-like ingress filtering; `all` ≈ trunk-like. |
| Read back running config | `show running-config` | |

## Illustrative GUI/CLI/SNMP mapping

| GUI Action | Primary path (Controller API) | Secondary path (device CLI) | SNMP | Notes |
|---|---|---|---|---|
| Enable/disable a port | Official or legacy REST API, device/port-config endpoint (exact payload shape not independently confirmed this session) | `interface 0/1` / `shutdown`-equivalent (exact FASTPATH keyword not independently confirmed this session) | Not confirmed writable | Unlike every other vendor in this project, the **API path is preferred over CLI by design**, not just by SignalScope convenience — see overview note above. |
| Configure VLAN membership on a port | Official or legacy REST API (confirmed to exist for VLAN config via the older API) | `vlan participation include <n>` + `vlan tagging <n>` + `vlan pvid <n>` (multi-step, see CLI table) | Not confirmed writable | |
| Persist configuration | *(nothing to do — controller-pushed config is already the durable source of truth)* | **N/A / actively discouraged** — CLI-typed changes are temporary, see durability caveat | N/A | The one vendor in this project where "Save" as a GUI concept doesn't map to a device-side action at all — the controller *is* the save target, always. |

## Official / primary sources (this session)

- [Getting Started with the Official UniFi API — Ubiquiti Help Center](https://help.ui.com/hc/en-us/articles/30076656117655-Getting-Started-with-the-Official-UniFi-API) (existence/title confirmed; full content blocked to automated fetch this session — read directly before implementation)
- [SNMP Monitoring in UniFi Network — Ubiquiti Help Center](https://help.ui.com/hc/en-us/articles/33502980942615-SNMP-Monitoring-in-UniFi-Network) (same access caveat)
- [`UBNT-MIB.txt` — dl.ubnt.com, Ubiquiti's own firmware/download server](https://dl.ubnt.com/firmwares/airfiber/v3.2/UBNT-MIB.txt) — fetched directly and read in full this session (2,301 lines).
- [products:software:unifi-controller:api — Ubiquiti Community Wiki](https://ubntwiki.com/products/software/unifi-controller/api) — the older/legacy REST API's most complete community documentation; explicitly notes it is unofficial.
- Community/third-party sources on device-CLI access and persistence behavior (search-summary sourced, not independently re-verified against a live device this session): UniFi SSH cheat-sheet writeups (UniHosted, LazyAdmin), and community threads on ui.com confirming the controller-overwrites-CLI-changes behavior.
- [Ubiquiti EdgeSwitch CLI Reference — jross.org](https://jross.org/ubiquiti-edgeswitch-cli-reference/) and [EdgeSwitch VLAN tagging/untagging — UISP Help Center](https://help.uisp.com/hc/en-us/articles/22591767474455-EdgeSwitch-EdgeSwitch-X-Tagging-and-Untagging-Port-VLANs) — FASTPATH-derived VLAN command family.

**Uncertainty flags for implementers**: the official API's exact endpoint/payload shapes, the exact device-CLI keyword for port admin-state (enable/disable), and whether UniFi-switch-specific firmware populates a richer `ubntUniFi` MIB branch than the AirFiber-bundled copy read this session are all **not independently confirmed** — read the live official-API reference and/or a UniFi-switch-specific MIB file before implementation. Given the durability caveat above, prioritize confirming the official API's coverage over deepening the CLI/SNMP research further — the CLI/SNMP paths are architecturally secondary for this vendor regardless of how well they're documented.
