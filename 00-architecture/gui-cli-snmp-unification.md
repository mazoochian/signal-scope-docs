# Unifying GUI, CLI, and SNMP

This is the core architectural bet of SignalScope: **the CLI is never abstracted away**. Every other tool in this space picks one of two lanes — full abstraction (NAPALM, Ansible network modules, Terraform providers: call `set_interface_state(up)` and never see what actually got typed) or monitoring-only (Zabbix, LibreNMS: read-only, no control plane at all). SignalScope's bet is that network engineers don't trust black-box abstraction for changes that can take down a switch, but also don't want to hand-type every routine change. The resolution is a **unified, bidirectionally-visible session** rather than an abstraction layer.

## The two directions

**GUI → CLI/SNMP visibility.** When a user clicks a GUI control (e.g. "Enable port"), SignalScope must:
1. Determine, per the device's vendor, exactly what a human would type to achieve this (which may be multiple lines — entering a config context, issuing the command, possibly exiting the context).
2. Actually execute those lines over the device's live CLI session (SSH/Telnet) — **not** a side-channel SNMP SET that bypasses the terminal — whenever a CLI path exists and the user's connectivity mode is CLI-based. If the device is only reachable via SNMP for this action (no live CLI session open, or the action is genuinely SNMP-only), fall back to a documented SNMP SET and render that operation in the terminal pane as a synthetic "command" (e.g. `# SNMP SET IF-MIB::ifAdminStatus.24 = up(1)`) so it's still visible in the same stream, clearly marked as SNMP rather than CLI.
3. Echo the literal command text into the same terminal buffer the user would see if they'd typed it themselves, tagged as GUI-originated (e.g. a distinct color/prefix) but otherwise indistinguishable in content from manual entry.

**CLI/manual → GUI visibility.** When a user (or an external engineer on a concurrent session) types a raw command directly in the embedded terminal, SignalScope must detect the resulting state change and reflect it in the GUI without requiring the user to separately "sync":
1. Parse the command as it's echoed back by the device (or diff `show running-config`/relevant `show` output before and after) to determine what changed.
2. Update the GUI's model of device state from that parse, rather than waiting for the next scheduled poll — the whole point is the GUI should never look stale relative to something a user just typed.
3. Where CLI-detected parsing is ambiguous or incomplete (e.g. a command whose effect isn't obvious from output alone), fall back to an immediate targeted SNMP GET or `show` re-query of just the affected object(s), not a full re-poll of the device.

## Why this rules out a traditional device-driver abstraction

A NAPALM/Ansible-style abstraction (`configure_interface(name, vlan=10, enabled=True)`) is attractive for the GUI→CLI direction alone, but it breaks the CLI→GUI direction: if a driver's internal template doesn't match what a human actually typed (because the human used a different but equivalent syntax, or because the driver batches/reorders commands for efficiency), the "any GUI action is deducible from the terminal and vice versa" guarantee is gone. SignalScope's per-vendor `gui-cli-snmp-mapping.md` files are deliberately literal — they specify the exact command text a human would type for a given action, not a templating DSL, precisely so the terminal and the GUI can be proven equivalent.

## Session model

A single **device session** object should own:
- The live CLI transport (SSH or Telnet channel, PTY-backed) if one is open, including current mode/context (user-exec, privileged, config, config-if, etc.) so both a human and the GUI-issued commands operate against the same tracked state (e.g. GUI must know it needs to emit `interface GigabitEthernet0/1` before `no shutdown` if the session isn't already in that interface's config context).
- An event stream that both the terminal UI (raw bytes/lines, for xterm.js-style rendering) and the structured GUI subscribe to. GUI actions and agentic-poller actions both publish into this same stream as the single source of truth — there's no back door that writes to the device without appearing here.
- An audit log persisted independently of the live stream (surviving session close), recording: actor (human via GUI, human via raw terminal, or agent), literal command/SNMP-operation text, timestamp, and resulting diff — satisfying compliance/traceability needs beyond what a scrollback buffer provides.

This matches the shape recommended in [connectivity-methods.md](connectivity-methods.md)'s "supervised remediation" tier and in [prior-art.md](prior-art.md)'s discussion of WebSSH2 (terminal transport) plus Oxidized (audit/diff) — SignalScope's novelty is fusing these into one session object instead of keeping them as separate, disconnected tools.

## Handling vendor asymmetry

Two structural asymmetries surface repeatedly across vendors (documented per-vendor, rolled up in [comparison/](../comparison/)):

- **Immediate-apply vs. candidate-config vendors.** Cisco/Arista/Huawei/Aruba-family CLIs apply each line as typed; Juniper requires an explicit `commit` (and supports `commit confirmed` rollback timers). The GUI's "Save"/"Apply" affordance must map to different underlying semantics per vendor — for Juniper, a GUI action is not "done" until commit succeeds, and the terminal must show the `commit` line, not just the `set`/`edit` lines.
- **SNMP write coverage is inconsistent and often narrow.** Where a vendor's SNMP agent is read-only by policy or default, GUI actions for that device must always route through the CLI session — there's no fallback. Where SNMP write does exist, it's typically limited to a handful of well-known objects (VLAN membership, port admin status, config-copy triggers) rather than general configuration. Never assume SNMP SET as a universal fallback; check the specific vendor's `mib-reference.md` and [comparison/snmp-write-support-matrix.md](../comparison/snmp-write-support-matrix.md) first.

## Open questions to resolve during implementation (not this research phase)

- How to represent "no live session open" GUI actions — does clicking a GUI control open a session on demand, queue the action until a session exists, or require an explicit device connection step first?
- Multi-user concurrency: if two users (or a user and an agent) have sessions open on the same device simultaneously, how are their command streams reconciled/labeled in a shared terminal view, if at all?
- How much CLI output parsing is generic (regex/table-parsing shared across vendors) vs. vendor-specific — likely needs a small per-vendor parser registry, not unlike how Netmiko/NAPALM structure their vendor plugins internally, while keeping the *emitted commands* fully literal per the point above.
