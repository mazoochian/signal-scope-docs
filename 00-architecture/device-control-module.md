# Device Control — Implementation Status

This is the "research → working code" handoff record for the `device-control` module in `signal-scope-be`, and the corresponding schema in `signal-scope-db`. It records what was built, why, what was verified (and how), and what's left — the counterpart, on the implementation side, to the vendor research this docs tree already contains.

**Full design decisions live in `signal-scope-be/src/device-control/README.md`** (checked into the backend repo, next to the code it documents). This file is the higher-level status record for the docs tree; read the module README for the actual architecture rationale, sanitization rules, and worker-model tradeoffs.

## What exists now

A uniform database schema (`signal-scope-db/migrations/017`–`022`) and a NestJS module (`signal-scope-be/src/device-control/`) that can:

- Dial a real device (or an emulator/simulator presenting the same interface) over SSH, Telnet, or SNMP.
- Run vendor-specific **literal** CLI command plans or SNMP SETs for a small, deliberate action set (`port.setAdminStatus`, `port.setDescription`, `vlan.setPvid`, `vlan.setTrunkAllowed`, `vlan.create`, `config.save`, `interface.setIpAddress`, `route.static.upsert`) — the same action set `comparison/snmp-write-support-matrix.md` already cross-references, so no new scope was invented.
- Refuse an SNMP SET when no documented write path exists for that vendor+action (checked against `device_capabilities`, seeded from this docs tree's own comparison tables) rather than assuming one.
- Audit every literal command/SNMP-op, with a durable session record, in `device_command_audit`/`device_sessions`.
- Cache an action into `pending_changes` when a device is unreachable (including a `planned`/ghost device that was never deployed), and drain the queue automatically on a down→up reachability transition — through the exact same execution path a live action uses.
- Treat real devices, EVE-NG-hosted devices, and Docker-simulated devices identically: same schema row shape, same code path, only the connection target's host/port/kind differs.

Vendors implemented this phase: **Cisco IOS/IOS-XE, Juniper Junos, Arista EOS, MikroTik RouterOS** — four vendors covering three structurally distinct CLI-apply models (immediate-apply/mode-based, candidate-config/commit, menu-path/no-modes) and a wide spread of SNMP-write posture (broad-for-a-few-objects, narrow-two-objects, essentially none, essentially none).

## Verification

Every claim below was actually run this session, not assumed:

- **Unit tests** (`adapters/*.adapter.spec.ts`, 101 assertions across the four vendors) — `buildCliPlan`/`buildSnmpPlan`/`buildReadbackCommand` against fixtures, no live transport. Each vendor's suite specifically asserts the SNMP-capability-gating behavior documented for that vendor (e.g. Juniper's `port.setAdminStatus` must return `null`, MikroTik's `buildSnmpPlan` must return `null` for every action, Arista's must be `null` except for the two confirmed IF-MIB objects).
- **Live-transport integration tests** (`device-control.e2e.spec.ts`, real Postgres + real Redis/BullMQ + real SSH connections to scripted CLI stub servers + a real SNMP agent) — one round trip per vendor (dial → session setup → literal plan → readback), the SNMP capability-refusal behavior against a real SNMP agent, the offline-queue-then-drain flow, and the raw-CLI-passthrough sanitizer + a real forwarded command.
- **Full existing test suite** (`npm test`): 174 passed, 1 failed — the failure is the pre-existing `app.controller.spec.ts` scaffold issue documented in `AUDIT-REPORT.md` L3, confirmed identical before and after this work (not a regression).
- `tsc --noEmit` and `nest build` clean.

## Testing infrastructure — a scope adjustment from the original plan

The plan going into this session called for a Docker-based `snmpsim` container. **This environment's Docker daemon has no container-registry egress** (`docker pull node:20-alpine` times out even though this host's own `curl` reaches Docker Hub's HTTPS endpoint fine — specific to how the daemon reaches the registry here). Rather than block on that, the simulators were built as self-contained TypeScript instead:

- `device-control/testing/snmp-agent-simulator.ts` — a real SNMP agent (`net-snmp`'s own `createAgent`/MIB-provider API), seeded with the exact OIDs the Cisco adapter's `buildSnmpPlan()` targets (`IF-MIB::ifAdminStatus`/`ifAlias`, `CISCO-VLAN-MEMBERSHIP-MIB::vmVlan`).
- `device-control/testing/cli-stub-server.ts` + `testing/scripts/<vendor>.script.ts` — a scripted SSH/Telnet responder (real `ssh2` server, real `net.Socket` telnet), one script per vendor, replaying the exact prompt/command sequences that vendor's adapter emits.

A `docker-compose.sim.yml` overlay is included at the monorepo root for a deployment environment that does have registry egress — it has **not** been run end-to-end in this session (only syntax-validated via `docker compose config`), since the actual verification above used the TypeScript simulators directly.

## Schema

New tables (`vendor_profiles`, `device_credentials`, `device_connection_targets`, `device_capabilities`, `vendor_capability_defaults`, `device_sessions`, `device_command_audit`, `pending_changes`, `device_vlans`, `interface_vlan_membership`) plus additive columns on `devices`/`interfaces`. Full rationale and the extension pattern for future vendors is in the module README; the short version: `device_capabilities`/`vendor_capability_defaults` are the structured, queryable version of `comparison/snmp-write-support-matrix.md` and `cli-syntax-matrix.md`, and `device_connection_targets.proxy_device_id`/`proxy_selector` specifically anticipate Ubiquiti's controller-first and Fortinet's FortiLink-mediated "session target differs from config target" shape (see `vendors/fortinet/fortilink-integration.md`) without needing a schema change when those vendors get adapters.

## What's explicitly not done this session

- **`signal-scope-fe` is untouched** — no inventory UI exists yet for this to plug into, per the original scoping.
- **Live WS/xterm.js terminal streaming** — not built. `CliChannel.onData()` is the tap point for a future gateway.
- **EVE-NG live verification** — EVE-NG was not reachable from this environment this session. `connection_kind='eve-ng'` is fully implemented (it's just SSH/Telnet/SNMP to a management IP — nothing EVE-NG-specific exists or is needed in the code), but no live smoke test against a real EVE-NG instance has run. **This is the concrete next step** once EVE-NG is reachable: attach a `device_connection_targets` row pointing at an EVE-NG lab device's console/management IP for one of the four implemented vendors, run one guided action through the API, and confirm the round trip — no code changes anticipated, this is a configuration step.
- **The other 9 documented vendors** (Extreme, Huawei, Aruba, Dell, D-Link, Fortinet standalone+FortiLink, Ubiquiti, Netgear, Zyxel) — schema and adapter interface accommodate them; no adapter code exists for them yet. Each one is a self-contained follow-up: implement `VendorAdapter`, register it, seed `vendor_capability_defaults` from the already-written comparison tables, write a CLI-stub script. The pattern is established and, per this session's experience, mechanical enough that a fresh subagent with the same brief shape used for Juniper/Arista/MikroTik can do most of the work per vendor.
- **Optimistic-concurrency conflict detection on drain** — `pending_changes.expected_prior_state` is recorded when supplied but not yet checked against a fresh read-back before a queued change is replayed. A drain currently applies queued changes FIFO and stops a device's whole queue on the first failure (a safe default), rather than the fuller "detect the live state diverged from what the change assumed" design.
- **Deep per-vendor `show`/`print` output parsing** — `parseReadback()` is intentionally minimal per adapter (enough to prove the round trip for the actions this phase's tests exercise), not a general parser.
- **Per-device BullMQ queue/worker cache eviction** — fine for a handful of test devices, would need an idle-eviction policy for a large fleet (see module README's worker-model section).

## A structural finding worth carrying forward

Implementing Junos surfaced a real architectural tension worth flagging for whoever works on this next: this phase's worker model runs each `DeviceAction` as its own fresh dial/session (no long-lived shared session across actions — see the module README's worker-model rationale). That's transparent for immediate-apply vendors (Cisco/Arista/MikroTik) but matters for Junos's candidate-config model: a `set`-only action and a later, separate `config.save`/`commit` action are two different sessions. The Juniper adapter deliberately uses plain shared `configure` (not `configure exclusive`, despite `cli-reference.md` recommending exclusive mode for automation) specifically because Junos's shared candidate persists uncommitted edits across separate sessions, while an exclusive-mode candidate does not survive its authoring session closing. This is a real, documented tradeoff (risk of a concurrent human's `configure exclusive` session or an uncoordinated `rollback 0` discarding a SignalScope-authored pending change), not an oversight — see `juniper-junos.adapter.ts`'s class doc comment and the module README. Any future vendor with a similar staged-apply model (candidate-config, two-phase commit, etc.) should be checked against this same tension before assuming the current worker model fits it unmodified.

## A test-hygiene quirk worth knowing about

`device-control.e2e.spec.ts` spawns child processes (`cli-stub-server.ts` × 4 vendors, `snmp-agent-simulator.ts`) in `beforeAll` and kills them in `afterAll`. If a test run is interrupted (killed mid-run rather than allowed to finish/time out normally), those children can be orphaned and keep holding their ports, which makes the *next* run fail with spurious `queued` instead of `executed` outcomes (the new run's freshly-spawned stub server fails to bind because the old one is still squatting the port, so the dial genuinely fails and the orchestrator correctly falls back to the offline queue — the observed symptom is a real consequence of the actual code path, not a flaky assertion). If this spec fails with unexpected `queued` outcomes, check for and kill any lingering `snmp-agent-simulator.ts`/`cli-stub-server.ts` processes before re-running — this bit this session's own verification pass once, confirmed as environment hygiene rather than an implementation bug.

## Environment notes (housekeeping, not architectural)

- This local Postgres instance has `devices`/`interfaces` (and other early-migration tables) owned by the `postgres` superuser rather than the `signalscope` application role — a pre-existing local-dev provisioning quirk (see project memory), not introduced by this work. New migrations that `ALTER TABLE`/add a `REFERENCES` on those tables needed to be applied via `sudo -u postgres psql` locally for this session's testing. A fresh Docker deployment (where `POSTGRES_USER=signalscope` initializes and owns everything itself) won't hit this.
- `bullmq`/`ioredis`/`ssh2`/`net-snmp` were added as new `signal-scope-be` dependencies; `npm audit` reports some transitive vulnerabilities pulled in with them — not reviewed/triaged this session, worth a pass before a production deploy.
