# Data Model Notes

Notes on how SignalScope's existing schema would need to evolve to support real connectivity. This is a research note, not a migration plan — no schema changes are made as part of this documentation effort.

## What exists today

From `signal-scope-db/migrations/002_schema.sql` and `006_device_configs.sql`, and `signal-scope-be/src/devices/devices.service.ts`:

- `devices` table: `id, name, ip, vendor, model, role, site_id, status, icon, up_since, created_at, updated_at`. `vendor`/`model`/`role` are free-text strings, not normalized/foreign-keyed — there's no `device_types` or `vendors` table, and nothing constrains `vendor` to a known value.
- `device_configs`: config-version snapshots (currently populated with mock text from `config-generator.ts`), keyed to a device, already shaped like what a real config-drift-detection poller (see [connectivity-methods.md](connectivity-methods.md)) would populate for real.
- `interfaces`, `device_metrics`/`interface_metrics` (TimescaleDB hypertables), `alerts`, `inventory_assets`, `discovered_devices` — the telemetry/inventory half of the schema is already reasonably close to what real SNMP/CLI polling would populate.
- No credential storage, no session/connection tracking, no command-audit table, no capability registry of what a given vendor/model supports.

## Gaps that matter for this feature set

1. **Vendor normalization.** `vendor: string` free text can't drive "which CLI dialect / MIB set / prompt regex applies to this device" — that needs to resolve to one of the documented vendor profiles (`vendors/cisco`, `vendors/juniper`, etc.). At minimum this wants an enum or lookup table mapping recognized vendor strings to a `vendor_profile` identifier, since the exact same device row today (`vendor: "Cisco"`) currently has no structured link to anything.
2. **Per-device capability/support flags.** Not every device of a given vendor supports the same things (SNMP version, SNMP write scope, Telnet vs SSH availability, NETCONF/RESTCONF). This is closer to a per-device (or per-model) capability record than a static vendor-level fact, since it can depend on firmware version and licensing (e.g. some Cisco licensing tiers gate NETCONF/RESTCONF).
3. **Credential storage**, distinctly per connectivity method (see [connectivity-methods.md](connectivity-methods.md) security notes): SSH key/password, Telnet password (flagged insecure), SNMP community (v1/v2c, flagged insecure) or SNMPv3 USM credentials (auth protocol/passphrase, priv protocol/passphrase), and separately an enable/privileged-mode secret where the CLI dialect requires one. None of this exists yet; it needs encryption-at-rest and must not collapse into a single password field.
5. **Session/connection state.** The "unified session" object described in [gui-cli-snmp-unification.md](gui-cli-snmp-unification.md) needs a live-state representation (current CLI mode/context, open transport type) that's ephemeral (in-memory/Redis-like), plus a durable **command audit log** table (actor, literal command or SNMP operation, timestamp, device, resulting diff) that is distinct from — and much more granular than — the existing `device_configs` snapshot table. `device_configs` records config *state at a point in time*; the new audit log records *actions taken*, which is what makes GUI actions traceable in the terminal and vice versa.
4. **OID/MIB registry.** If SignalScope ever wants to browse or validate MIBs interactively (not just hardcode a handful of OIDs per feature), it would need some representation of the OID trees documented in `vendors/*/mib-reference.md` and `mibs/` — likely generated/loaded from the vendored `.mib` files rather than hand-modeled in SQL.

## Non-goals for this note

This document intentionally does not propose exact column/table DDL — that's implementation work for later, informed by whichever connectivity method gets built first. The purpose here is to flag that `vendor`/`model` as free text, and the complete absence of credential/session/audit modeling, are the concrete gaps between today's simulated schema and what real connectivity requires.
