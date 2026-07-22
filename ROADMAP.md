# signal-scope-docs — State & Roadmap

Self-assessment file, written for whoever (human or Claude) picks this docs effort up next. Purpose: know exactly what's done, what's *deliberately* scoped lighter (not a gap), and what to tackle next — without re-reading every vendor file to reconstruct that picture from scratch. Update this file whenever the shape of the work changes materially; don't let it go stale the way a status doc usually does.

## What this docs tree is for

Research groundwork for SignalScope's real-device connectivity (CLI/SSH/Telnet + SNMP), living entirely separate from the product code (`signal-scope-fe`/`signal-scope-be`/`signal-scope-db`, each their own repo). Nothing here is wired into the app yet — see [Relationship to the actual product](#relationship-to-the-actual-product-important-context) below. The `00-architecture/` docs set the design (GUI/CLI/SNMP unification, connectivity methods, data-model gaps); the `vendors/` tree is per-vendor research; `comparison/` rolls the per-vendor findings up into cross-vendor tables; `mibs/` vendors the shared standard MIBs once.

## Tiering system (established convention, now formalized here)

Three vendor doc tiers exist, established implicitly by the previous work and made explicit here:

- **Tier 1-3 ("full")**: `overview.md`, `cli-reference.md`, `mib-reference.md`, `gui-cli-snmp-mapping.md`, plus a `mibs/` subfolder (vendored `.my`/`.txt` files where redistribution terms allow it, otherwise a `mibs/README.md` explaining why not and linking official + third-party sources instead).
- **Appendix tier**: a single `overview.md` folding in curated CLI tables, MIB support, and a GUI/CLI/SNMP mapping table, explicitly marked "*Appendix-tier vendor doc*" at the top. Reserved for vendors that are lower-priority for SignalScope (SMB/prosumer-oriented, or split across incompatible internal dialects that make deep treatment less valuable until one dialect is chosen as the actual target).
- **Unstarted**: nothing beyond an empty scaffold directory.

This tiering is a **deliberate scope decision**, not laziness — do not "complete" an appendix-tier vendor into a full Tier 1-3 set without checking with the user first, since the previous session chose appendix tier for D-Link/Fortinet specifically to spend depth where SignalScope's actual target market (enterprise switch fleets) benefits most.

## Current state by vendor

| Vendor | Tier | Status | Notes |
|---|---|---|---|
| Cisco IOS/IOS-XE | Full | **Done** | Only vendor with actual `.my` MIB files vendored (Cisco's own copyright terms permit it) — `CISCO-CONFIG-COPY-MIB`, `CISCO-PORT-SECURITY-MIB`, `CISCO-STACK-MIB`, `CISCO-VLAN-MEMBERSHIP-MIB`. |
| Arista EOS | Full | **Done** | Narrowest confirmed SNMP write surface of the full-tier vendors (2 of 9 actions). |
| Extreme EXOS | Full | **Done** | `mibs/README.md` link-only (HPE/Extreme MIB licensing unclear) — see licensing note. |
| Huawei VRP | Full | **Done**; SNMP-write partially re-verified 2026-07-16 (Phase 2) | `mibs/README.md` link-only (same licensing situation). Access-VLAN/PVID write now confirmed (`Q-BRIDGE-MIB::dot1qPvid`); trunk/STP/LACP/port-security/save remain unconfirmed — still this project's second-weakest-evidenced SNMP-write vendor (after Dell). |
| Juniper Junos | Full | **Done** | Functionally CLI/NETCONF-only for config; candidate-config/commit model fully documented. |
| HPE Aruba (AOS-CX) | Full | **Done**; SNMP-write substantially re-verified 2026-07-16 (Phase 2) | **By a clear margin the broadest confirmed SNMP write surface of any vendor in this project** — port-security, config-copy/checkpoint, VLAN (creation/PVID/trunk-allowed-list via `Q-BRIDGE-MIB`), and STP port-hardening (edge-port/BPDU-Guard/Root-Guard/Loop-Guard via `ARUBAWIRED-MSTP-MIB`, 11 objects fully enumerated from first-hand MIB source) are all confirmed read-write. The edge-port finding is unique across every vendor in this project. Legacy dialects (ArubaOS-Switch/ProCurve, Comware-based HPE) explicitly out of scope — flagged as a separate future research pass if SignalScope needs to support that older hardware. |
| MikroTik RouterOS | Full | **Done this session** — was missing `mib-reference.md` and `gui-cli-snmp-mapping.md`; also fixed a real bug where `overview.md` claimed a MIB file was vendored that didn't actually exist on disk. | One of only two vendors (with Juniper) with essentially no SNMP config-write surface — confirmed by grepping the actual vendored MIB file rather than inferring from prose. |
| D-Link | Appendix | Done, as scoped | Genuinely two incompatible CLI dialects on the same product line (classic xStack vs. newer Cisco-like) — this is *why* it's appendix tier, not a symptom of incompleteness. |
| Fortinet (FortiSwitch) | Appendix | Done, as scoped | Distinctive `config`/`edit`/`next`/`end` block grammar with auto-save-on-`end` — structurally the third pattern (alongside immediate-apply and candidate-commit) worth remembering when reasoning about session-state tracking generically. |
| Dell PowerSwitch (SmartFabric OS10) | Full | **Done 2026-07-16** | Genuinely novel finding: opt-in `start transaction` candidate-configuration mode layered on an immediate-apply default — a third config-apply pattern distinct from both Cisco's persist-only and Junos's mandatory candidate/commit. SNMP-write research is thinner than the original seven vendors (search-summary sourced, not full primary-document reads) — flagged as a verification-backlog priority. PEN ambiguity documented but not resolved (674 vs. legacy Force10 6027). |
| Ubiquiti (UniFi switches) | Appendix (controller-first) | **Done 2026-07-16** | Major finding: CLI-typed config on UniFi switches is **not durable** — the controller overwrites it on next reboot/reprovision, a direct conflict with the architecture doc's CLI/GUI-durability assumption (flagged for a future `gui-cli-snmp-unification.md` update, not resolved in this pass). Controller API (official + legacy REST) documented as the primary integration path. |
| Netgear (ProSAFE/M-series) | Appendix | **Done 2026-07-16** | Confirmed real product-tier fragmentation (not just dialect drift like D-Link): "Smart Managed"/GS-series is web-GUI-only by design (unofficial telnet at best), full CLI only exists on the M-series/"Intelligent Edge" line — SignalScope's Netgear capability detection must check product line before assuming CLI reachability at all. |
| Zyxel | Appendix | **Done 2026-07-16** | Same product-tier fragmentation pattern as Netgear, one notch more extreme: GS1900 confirmed to have **no config-write CLI path at all** (only `configure terminal`/`Unknown command`) — the only device class in this docs tree where CLI-only is architecturally not an option and SNMP-write status (unresolved) becomes the highest-priority open question. |

## Cross-vendor synthesis (`comparison/`)

Both files were **referenced by name from multiple existing docs before they existed** — `gui-cli-snmp-unification.md`, `connectivity-methods.md`, and several vendors' `gui-cli-snmp-mapping.md` files all linked to `comparison/cli-syntax-matrix.md` and `comparison/snmp-write-support-matrix.md` as if they were already written. This was the single highest-priority gap going into this session (broken internal links in an otherwise rigorously cross-referenced docs tree) and is now closed:

- **`cli-syntax-matrix.md`**: mode/context model, negation keyword, interface naming, paging control, save/persist semantics, and port enable/disable syntax across all seven full-tier vendors.
- **`snmp-write-support-matrix.md`**: a per-action × per-vendor grid (port state, description, VLAN, trunk, STP-edge, LACP, port-security, config-save, SNMP self-config) with ✅/⚠️/❌ confidence levels, so SignalScope's capability-detection layer has one place to check "does an SNMP fallback exist here" instead of re-deriving it from seven mapping files.

## Next vendor research (not yet started)

Four vendors have no content. Recommended tier and rationale for each, for whoever picks this up — **confirm the tier choice with the user before investing full-tier effort**, since that's a real time/depth tradeoff decision, not a technical one:

- **Dell (PowerSwitch / OS10)**: OS10 is a genuine enterprise NOS (Linux-based, Broadcom SAI, used in real datacenter fleets) — plausibly deserves **full tier**, not appendix. Has its own quirks worth documencing: OS10 CLI is deliberately IOS-like at the surface but has a very different underlying (declarative/YANG-driven) config model, closer in spirit to Junos's candidate-config than to Cisco's immediate-apply — verify this before assuming either pattern from `cli-syntax-matrix.md` applies directly.
- **Ubiquiti (UniFi)**: almost entirely **GUI/controller-managed** in real deployments (UniFi Network Controller/UniFi OS) — direct CLI access exists (SSH into `EdgeSwitch`-derived or UniFi switch firmware) but is not how these devices are normally administered, which cuts against SignalScope's core "CLI is never abstracted away" bet (see `00-architecture/gui-cli-snmp-unification.md`). Worth a scoping conversation with the user about whether SignalScope should target the UniFi Controller's own API instead of/alongside direct device CLI before writing this vendor's docs — the standard per-vendor template may not fit well until that's decided.
- **Netgear (ProSAFE/M-series)**: recommend **appendix tier** — SMB-oriented, and (like D-Link) plausibly spans more than one CLI generation across its product range. Verify this assumption before writing rather than assuming it mirrors D-Link's exact two-family split.
- **Zyxel**: recommend **appendix tier** for the same SMB-market reasoning as Netgear, pending verification.

For all four: follow the research discipline already established (real primary sources cited inline, explicit "not independently confirmed this session" flags rather than invented syntax, and a MIB-licensing check before vendoring any `.mib`/`.my` text — see `vendors/huawei/mibs/README.md` and `vendors/mikrotik/mibs/README.md` for the two different resolutions of that check).

## Other flagged-but-deferred scope items

Called out inline in existing docs, collected here so they don't get lost:

- **Ruckus ICX/FastIron** — flagged in `vendors/extreme/mibs/README.md` as a distinct product line (legacy Foundry PEN 1991, not Extreme's 1916) that would warrant its own vendor folder rather than being folded into Extreme's, if SignalScope adds ICX support.
- **Legacy Aruba dialects** — `ArubaOS-Switch`/ProCurve (2530/2620/2930/3810/5400zl families) and Comware-based HPE switches (ex-3Com/H3C, FlexNetwork/FlexFabric) are explicitly out of scope in `vendors/aruba/overview.md`, deliberately kept separate from the ArubaOS-CX docs. Only worth a research pass if SignalScope needs to support that older/different hardware.
- **Aruba's `no page` persistence claim** — community-sourced only, not confirmed against an official AOS-CX page (several official CLI-bank pages 403'd to automated fetch during that research session). If SignalScope ends up targeting Aruba devices, this is worth a live-device check before relying on it.

## Live-device verification backlog

A recurring, honest pattern across the full-tier vendor docs: many SNMP write claims are marked "not independently confirmed this session" rather than asserted outright — this is the docs set's core discipline (never invent a working SNMP path), but it leaves a real backlog of items that only a live device (or a very thorough primary-source re-read) can resolve.

**Phase 2 update (2026-07-16)** — worked as a deeper primary-source research pass (no lab access, per the Phase 0 decision). Three of the original five items resolved, with substantial findings:

1. ~~Huawei's entire SNMP-write column is unconfirmed~~ — **partially resolved**: `Q-BRIDGE-MIB::dot1qPvid` (access-VLAN assignment) confirmed writable, directly from Huawei's own MIB reference page. Trunk-allowed-list/STP/LACP/port-security/save remain unconfirmed and are still Huawei's weakest-evidenced areas.
2. ~~Whether `Q-BRIDGE-MIB::dot1qPvid` is writable on Aruba AOS-CX~~ — **resolved: yes**, plus trunk allowed-list (`dot1qVlanStaticEgressPorts`) and VLAN creation (`dot1qVlanStaticRowStatus`) — all confirmed via an official HPE page dedicated specifically to this topic. Search-summary-sourced (page itself 403'd to fetch), one tier below first-hand confirmation.
3. Whether `EXTREME-VLAN-MIB` or `Q-BRIDGE-MIB` is the real SNMP-write target for VLAN membership on EXOS — **still open**, and a near-miss worth flagging explicitly: a search surfaced an official-looking Extreme page confirming exactly this `Q-BRIDGE-MIB` pattern, but it turned out to document the **ExtremeSwitching 200 Series, a different, non-EXOS product line** — a real example of the cross-product-line conflation risk this docs tree's discipline is designed to catch. Not applied to the EXOS vendor folder. A follow-up search scoped specifically to `documentation.extremenetworks.com/exos_*` paths is needed.
4. ~~Full enumeration of `ARUBAWIRED-MSTP-MIB`'s 5 confirmed-but-unmapped read-write objects~~ — **resolved, and the single biggest finding of this pass**: **11** objects (not 5 — the original count undercounted), first-hand-read directly from vendor MIB source. Includes `arubaWiredMstpPortAdminEdge` — **the STP edge-port/PortFast-equivalent object, and the only confirmed SNMP write path for that concept across every vendor in this project.** Also confirmed: BPDU Guard + auto-recovery timeout, Root Guard, Loop Guard, all read-write. This single finding moved Aruba from "unusually SNMP-write-capable" to "by a clear margin the broadest confirmed SNMP write surface in this project" — see `comparison/snmp-write-support-matrix.md`.
5. Aruba's `no page` paging-persistence claim — **not revisited this pass, still open.**

**New item surfaced by this pass**: Dell OS10's SNMP-write column (added in Phase 1a, after the original backlog was written) is now the weakest-evidenced of all eight researched vendors — no object was confirmed writable in either the original Dell research or this Phase 2 pass, and it wasn't targeted this time since Phase 2 focused on the original five-item list. Should be the top priority for the *next* Phase 2 continuation.

## Relationship to the actual product (important context)

The signal-scope-be/fe app (separate repos, see the monorepo `README.md`) currently has **no real device connectivity at all** — `devices.service.ts`'s vendor/model/role fields are free text with no link to this docs tree's vendor profiles, and (per a separate, unrelated security/quality audit of the app done 2026-07-12, `AUDIT-REPORT.md` in this repo's root) several read endpoints return **fabricated telemetry** (hardcoded utilization/error arrays, padded vendor counts, a static root-cause-chain array) rather than real polled data. That audit is a different workstream (app security/correctness, not vendor documentation) but the finding matters here: **this entire docs tree is pre-implementation research**, not yet reflected in a single line of product code. `00-architecture/data-model-notes.md` already flags the concrete schema gaps (no credential storage, no vendor normalization, no session/audit tables) that stand between "this documentation exists" and "SignalScope can actually open a session to a real switch." Don't assume any CLI/SNMP behavior documented here has been validated against the running app — it hasn't; it's validated (to the level flagged per-claim) against vendor documentation and, for MikroTik/Cisco/Aruba's MIB claims, actual vendored MIB source text.

## Phases

Formal phase breakdown of the "next steps" above, so progress is trackable across sessions instead of re-derived each time. Update each phase's status inline as work happens; don't let this drift out of sync with reality.

### Phase 0 — Scope confirmation

**Goal:** lock tier + approach decisions for the four unstarted vendors before spending research effort on any of them, since tier/approach is a product-priority call, not a research question.
**Deliverable:** decisions recorded in this file (see status table above).
**Exit criteria:** every one of Dell/Ubiquiti/Netgear/Zyxel has an assigned tier (or an explicit "skip for now") and, for Ubiquiti specifically, a decision on controller-API-vs-direct-CLI framing.
**Status:** ✅ done. Decisions (2026-07-16): Dell → full tier. Ubiquiti → appendix tier, controller-first framing (UniFi Network Controller/UniFi OS API as the primary integration path, direct device SSH/SNMP as secondary — matches how these devices are actually run in practice, even though it's a different shape from every other vendor in this docs tree). Netgear → appendix tier. Zyxel → appendix tier. Phase 2 → redefined as "deeper primary-source research pass," no lab access available; still flag results as unconfirmed rather than silently upgrading confidence.

### Phase 1 — New vendor documentation

**Goal:** produce doc sets for the four previously-unstarted vendors, one sub-phase per vendor, each following the established per-vendor template and research discipline (real primary sources, explicit "not independently confirmed" flags, MIB-licensing check before vendoring any text).

- **1a. Dell (PowerSwitch / OS10)** — full tier. Status: ✅ done 2026-07-16.
- **1b. Ubiquiti (UniFi)** — appendix tier, controller-first framing (see Phase 0 decision above). Status: ✅ done 2026-07-16.
- **1c. Netgear (ProSAFE/M-series)** — appendix tier. Status: ✅ done 2026-07-16.
- **1d. Zyxel** — appendix tier. Status: ✅ done 2026-07-16.

**Phase 1 status: ✅ all four sub-phases done 2026-07-16.** `comparison/` tables were **not** updated to add Dell/Ubiquiti/Netgear/Zyxel as new columns/rows — those two files were scoped to the seven original Tier-1 vendors and restructuring them for four more (three appendix-tier, with much thinner per-action confirmation) is a real editorial judgment call, not a mechanical addition. Treat that as a residual Phase 1 follow-up rather than done-by-default: revisit whether `cli-syntax-matrix.md`/`snmp-write-support-matrix.md` should gain a second table/section for the appendix-tier vendors, or stay scoped to Tier 1-3 as originally written.

**Cross-vendor pattern that emerged during Phase 1, worth carrying into future architecture discussion**: three of the four new vendors (Ubiquiti, Netgear, Zyxel) turned out to have a **controller/GUI-primacy vs. direct-CLI** tension that doesn't exist for any of the original seven — Ubiquiti's controller actively overwrites CLI changes, Netgear/Zyxel's entry-tier hardware has partial-to-zero CLI config capability by design. This is a real, recurring pattern (not three unrelated vendor quirks) that `00-architecture/gui-cli-snmp-unification.md` doesn't yet account for — its "Handling vendor asymmetry" section currently only covers immediate-apply-vs-candidate-config and inconsistent-SNMP-write-scope; a third asymmetry axis ("does this device even have a durable, independent CLI-config surface, or is GUI/controller the only durable path") is now evidenced by three separate vendors and worth a dedicated architecture-doc update in a future session.

### Phase 2 — Live-device verification backlog

**Goal:** resolve the "not independently confirmed this session" flags listed above.
**Constraint (important):** this docs effort has no physical or virtual lab device access. "Live-device verification" as originally written assumed lab access that doesn't actually exist yet.
**Status:** in progress. First pass completed 2026-07-16 — worked the original five-item backlog (see [Live-device verification backlog](#live-device-verification-backlog) below): 3 of 5 resolved (Huawei PVID, Aruba VLAN write, Aruba MSTP full enumeration — the last being this pass's major finding, see below), 1 still open (Extreme VLAN-MIB, with a near-miss cross-product-line conflation caught and avoided), 1 not revisited (Aruba `no page`). Dell's SNMP-write column emerged as a new top-priority item for the next continuation. Most official vendor doc pages (Huawei, Aruba HTML/PDF guides) are JS-rendered or return 403 to automated fetch — this phase's findings are consistently one confidence tier below a live device: either "first-hand MIB source read" (the Aruba MSTP enumeration, done via direct GitHub-mirrored MIB fetch) or "search-engine-summary of an official page" (Huawei PVID, Aruba VLAN-write) — both explicitly flagged per-claim rather than silently upgraded to "live-device confirmed."

### Phase 3 — Deferred scope items

**Goal:** revisit the items flagged but not acted on: whether D-Link/Fortinet should be promoted from appendix to full tier, whether Ruckus ICX/FastIron gets its own vendor folder, whether legacy Aruba dialects (ArubaOS-Switch/ProCurve, Comware) warrant a research pass.
**Status:** not started — lowest priority of the three phases, deliberately sequenced last.
