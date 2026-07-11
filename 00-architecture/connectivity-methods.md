# Connectivity Methods

SignalScope needs to reach real switches through four distinct channels. This document covers how each one actually works at the protocol/session level, and what a SignalScope backend implementation needs to handle. It intentionally does not propose abstracting these into one "device driver" interface — see [gui-cli-snmp-unification.md](gui-cli-snmp-unification.md) for why.

## 1. Telnet

- Cleartext, unauthenticated-transport TCP session (typically port 23). Still ships enabled by default on many SMB switches (Netgear, D-Link, some MikroTik configs) and is common on older Cisco/HP gear reachable only via out-of-band management VLANs.
- No native framing — it's a raw byte stream with optional Telnet protocol negotiation (IAC sequences for terminal type, window size, echo). Most network-gear telnet servers barely negotiate anything; a client can often get away with disabling option negotiation entirely and treating it as a dumb pipe once login prompts are handled.
- Session shape a client must handle: `Username:` / `Password:` prompt scraping (regex-based, vendor prompt text varies), then landing in either user-exec or already-privileged mode depending on vendor (e.g. Huawei VRP often drops straight into a mode requiring `system-view`; Cisco IOS lands in user EXEC requiring `enable` + a second password).
- Because Telnet has no transport security, SignalScope should flag it explicitly in the UI per-device ("this session is unencrypted") and never store Telnet passwords with weaker protection than SSH credentials.

## 2. SSH

- Same interactive-session shape as Telnet but over an encrypted, server-authenticated (host key) transport, client-authenticated via password or public key.
- Implementation-wise this is a PTY-backed shell channel, not "exec a command and get a return code" — network OS CLIs are interactive REPLs with stateful context (config mode, interface sub-mode) and often no usable exit codes at all. This is the same reason tools like Netmiko and Ansible's `network_cli` connection plugin exist — they manage a persistent shell channel and read until a **prompt regex** matches, not until EOF.
- Practical session-handling concerns (all vendor-specific, see each vendor's `cli-reference.md`):
  - **Prompt detection**: read output until a line matches the vendor's prompt pattern for the current mode (e.g. `hostname>`, `hostname#`, `hostname(config)#`, `hostname(config-if)#`). Mode transitions must be tracked so the terminal UI knows what "context" a raw command will execute in.
  - **Paging**: nearly every vendor CLI paginates `show`/`display` output by default (`--More--`, `---- More ----`) and needs a one-time command to disable it per session (`terminal length 0` on Cisco, `set cli screen-length 0` on Juniper, `terminal length 0` on Arista, `screen-length 0 temporary` on Huawei, `no page` differs per vendor — tracked in [comparison/cli-syntax-matrix.md](../comparison/cli-syntax-matrix.md)). SignalScope should send this automatically at session start, but **echo it in the terminal like any other command**, since the "no hidden commands" principle applies to setup commands too.
  - **Enable/privileged mode**: Cisco-family CLIs separate unprivileged and privileged EXEC; entering config mode and making changes requires the privileged transition first. This needs to be modeled as explicit state, not hidden.
  - **Config commit semantics differ fundamentally by vendor**: Cisco/Arista/Huawei/Aruba apply CLI config lines immediately to the running config (a separate `write`/`copy running-config startup-config` persists to NVRAM); Juniper uses a candidate-config model requiring an explicit `commit` before changes take effect at all. This distinction must be visible in the GUI (see unification doc) — a Juniper "Save" button means something structurally different from a Cisco "Save" button.

## 3. SNMP

- UDP-based (port 161 for agent queries, 162 for traps/informs), operating on a MIB (Management Information Base) tree of OIDs (Object Identifiers).
- **Versions**: SNMPv1 (community string, no auth beyond that, 32-bit counters only), SNMPv2c (community string, adds GetBulk and 64-bit counters), SNMPv3 (user-based security model — USM — with separate authentication, usually SHA/MD5, and optional privacy/encryption, usually AES/DES). SignalScope should default to v3 where a device supports it and clearly surface in the UI when a device is only reachable via v1/v2c community strings (a security-relevant fact, not just a config detail).
- **Operations relevant to SignalScope**:
  - `GET` / `GETNEXT` / `GETBULK` — read one object, walk sequentially, or bulk-walk a table efficiently (v2c/v3 only). This is what read-only polling uses (see Agentic Polling below).
  - `SET` — write a scalar or table-row object. This is how SNMP-driven config changes happen (e.g. setting `IF-MIB::ifAdminStatus` to bring a port up/down). **Critically, SNMP SET support for anything beyond a handful of standard objects is inconsistent and often disabled by default** across vendors — many switches ship SNMP read-only, or only expose a small vendor MIB subset as writable (commonly VLAN membership and a config-save trigger, since those are common NMS integration points). This inconsistency is tracked per-vendor in each `mib-reference.md` and rolled up in [comparison/snmp-write-support-matrix.md](../comparison/snmp-write-support-matrix.md). SignalScope's GUI must never assume an SNMP SET path exists for an action without checking the specific vendor's documented support.
  - `TRAP` / `INFORM` — asynchronous device-initiated notifications (link up/down, config change, environmental alarms). Relevant to the polling architecture below as a push-based complement to pull-based polling.
- MIBs are organized as a tree under `iso.org.dod.internet` (`1.3.6.1`), with `mib-2` (`1.3.6.1.2.1`) holding standard objects and `private.enterprises` (`1.3.6.1.4.1.<vendor-PEN>`) holding vendor-specific objects. See [standard-mibs.md](standard-mibs.md) for the cross-vendor standard objects SignalScope will use most, and each vendor folder for enterprise-MIB specifics.

## 4. Agentic Polling

Zabbix/LibreNMS/PRTG-style tools poll SNMP (and sometimes SSH-executed scripts) purely to **observe**: GETBULK-walk `ifTable`/`ifXTable` for interface counters, discover table rows via low-level-discovery-style walks, store time series, and fire alerts on thresholds. This is read-only by design — the monitoring system is never supposed to be a control plane.

SignalScope's polling should do everything the above does (it already has the telemetry storage half-built — see [data-model-notes.md](data-model-notes.md) on `device_metrics`/`interface_metrics` hypertables) but the brief specifically raises whether polling can also be **agentic** in the sense of taking action, not just monitoring. Concretely, three tiers, in increasing order of autonomy:

1. **Passive telemetry polling** (Zabbix-equivalent): scheduled SNMP GETBULK against `IF-MIB`/`BRIDGE-MIB`/vendor health MIBs, or scheduled `show` command scraping over SSH where SNMP doesn't expose something (e.g. per-vendor CPU/memory objects are notoriously inconsistent — some vendors only expose these via CLI, not SNMP, at all).
2. **Config-drift detection** (Oxidized/RANCID-equivalent, and already schema-supported by `device_configs`): periodically pull the running config (`show running-config` over SSH, or trigger a TFTP/SCP export via a vendor's config-copy MIB over SNMP where supported — e.g. Cisco's `CISCO-CONFIG-COPY-MIB`), diff against the last stored snapshot, and surface drift as an alert.
3. **Supervised remediation** (the actual differentiator): an agent loop that observes telemetry/drift and is authorized — per-device, per-action-class, with explicit approval gates configurable by the user — to *act*. The critical design constraint, carried over from the unification principle: **an agent-initiated change must be issued through the same session/command path a human GUI action would use**, so it shows up in the terminal exactly as if a human typed it, with the same audit trail. This avoids the common automation-tool failure mode of a hidden API-only control plane that the CLI/GUI can't account for. Full design in [gui-cli-snmp-unification.md](gui-cli-snmp-unification.md).

## Security/credential handling notes (applies to all four)

- Telnet and SNMPv1/v2c both transmit credentials in the clear (SNMP community strings are effectively passwords). Any credential store SignalScope builds needs to support "this device is only reachable insecurely" as a first-class, visibly-flagged state rather than pretending all devices are equally secured.
- SSH host keys and SNMPv3 engine IDs should be pinned/tracked per device to detect device replacement or MITM, similar to how `known_hosts` works for SSH generally.
- Enable/privileged-mode secrets, SNMPv3 auth/priv passphrases, and SSH keys are distinct credential types per device and need independent storage/rotation — they should not collapse into a single "device password" field the way the current `devices` table implies (see [data-model-notes.md](data-model-notes.md)).
