# Prior Art

What existing tools in this space do, what SignalScope should borrow, and — just as importantly — what to deliberately not copy, since SignalScope's core positioning (CLI never abstracted, GUI/CLI/SNMP unified in one visible stream) is a direct reaction against how most of these tools are built.

## Netmiko

A Python library for interactive CLI sessions to network devices over SSH/Telnet/serial. **Borrow**: its session-handling primitives are exactly right for SignalScope's transport layer —
- A `BaseConnection`/vendor-subclass architecture where each vendor class only overrides prompt regex, paging-disable command, and mode-transition sequences (`enable`, config entry/exit) — the mechanics (read-until-prompt-match, send-and-wait) are generic.
- Explicit `disable_paging()` step run once per session, vendor command varies (`terminal length 0` vs `set cli screen-length 0` vs others — see [comparison/cli-syntax-matrix.md](../comparison/cli-syntax-matrix.md)).
- Prompt-based mode detection (unprivileged vs. `enable` vs. config vs. config-subinterface) as explicit state, not inferred after the fact.

**Don't copy**: Netmiko's `send_command()`/`send_config_set()` API is a call-and-response abstraction — the caller gets a return string back, and nothing about "what got sent" is inherently visible to a UI unless the caller wires that up separately. SignalScope needs the wiring-up (terminal-visible echo of every sent line) to be the default, not a bolt-on.

## NAPALM (Network Automation and Programmability Abstraction Layer with Multivendor support)

A Python library exposing one API (`get_facts()`, `get_interfaces()`, `load_merge_candidate()`, `commit_config()`, etc.) across Cisco IOS/IOS-XR/NX-OS, Juniper Junos, Arista EOS, and community drivers — using each platform's best available underlying transport (NX-API for Nexus, eAPI for Arista, NETCONF for Junos, SSH for classic IOS). This is the most mature existing "hide the differences" layer in networking automation.

**Borrow**: the idea that different vendors are best reached over different underlying transports where available (eAPI/NX-API/NETCONF are more efficient and less fragile than screen-scraping CLI output), and that a *read* path (facts-gathering) benefits from normalization even in a CLI-first tool — SignalScope's telemetry-polling layer (see [connectivity-methods.md](connectivity-methods.md)) can normalize *readings* internally without that implying the *write* path should be abstracted too.

**Don't copy** — this is the central lesson for SignalScope's positioning: NAPALM's `load_merge_candidate()` takes a config template and pushes it, but a human reading the GUI afterward has no direct line of sight to "what CLI would I have typed for this." That's precisely the trust gap SignalScope exists to close. Any templating/batching for efficiency must still resolve to the literal per-vendor command text specified in each vendor's `gui-cli-snmp-mapping.md`, echoed as such.

## Oxidized (RANCID's spiritual successor)

A configuration-backup tool: for each device in inventory, connects via SSH (falling back to Telnet), runs the vendor's "dump full config" command, diffs against the last stored version, and commits changes to a Git repository on a schedule (typically every few hours).

**Borrow directly** — this is close to a checklist for SignalScope's existing `device_configs` table and the "config-drift detection" tier of agentic polling: SSH-first-Telnet-fallback connection strategy, per-vendor "show config" command knowledge (already partially encoded as mock output in `config-generator.ts`), diff-on-change rather than storing every poll, and separate output backends (SignalScope already has Postgres for this instead of Git, which is a reasonable substitution given it also needs to serve the live GUI, not just archival).

**Extend, don't stop there**: Oxidized is intentionally read-only/archival — it never writes to a device. SignalScope's differentiator is that the same connection machinery Oxidized uses for backup should also serve the live, bidirectional GUI/CLI session — treat Oxidized's polling model as the read/audit half of a system that also has a write half.

## Zabbix (and LibreNMS/Observium/PRTG as similar agentless-SNMP monitoring tools)

Monitoring platforms that treat SNMP as a purely agentless read source: bulk `GETBULK`-based table walks (e.g. walking `ifTable` via `ifIndex` as a low-level-discovery key to auto-generate one item/trigger per discovered interface), stored as time series, alerted on thresholds. No write path exists at all — these tools are architecturally incapable of changing a device, by design, since their job is trusted-observer, not control-plane.

**Borrow**: the discovery-via-table-walk pattern (walk `ifTable`, auto-create per-interface monitoring instead of hand-configuring each one) is directly applicable to SignalScope's interface/telemetry pages. Bulk `GETBULK` over sequential `GETNEXT` matters for polling performance at scale.

**The explicit gap SignalScope fills**: the user's framing — "monitoring in Zabbix's sense, but perhaps ours can do both" — is exactly right. Zabbix-style tools stop at observation because that's their entire value proposition (safety through incapability). SignalScope's agentic-polling tier (see [connectivity-methods.md](connectivity-methods.md)) extends the same observation machinery into supervised action, which is a materially different trust model and needs the audit/visibility guarantees from [gui-cli-snmp-unification.md](gui-cli-snmp-unification.md) to be safe — Zabbix doesn't need those guarantees because it never acts.

## WebSSH2 (and the general xterm.js + node-pty/ssh2 + WebSocket pattern)

A reference architecture for embedding a real terminal in a browser: `xterm.js` renders in the browser, a server-side SSH client (`ssh2` in Node, or `node-pty` for local shells) holds the actual session, and a WebSocket carries raw bytes both directions — terminal keystrokes flow to the SSH channel, SSH channel output flows back to the renderer.

**Borrow directly** as the transport shape for SignalScope's embedded web terminal, given the backend is already NestJS/Node: `ssh2` for the SSH channel, `xterm.js` on the Next.js frontend, WebSocket (NestJS has native WS gateway support) for the duplex byte stream. The one addition SignalScope needs beyond stock WebSSH2: a **tap on the byte/line stream** that also feeds the structured event bus described in [gui-cli-snmp-unification.md](gui-cli-snmp-unification.md), so GUI state and the audit log derive from the exact same stream the terminal renders, rather than WebSSH2's model of "the terminal is the only consumer."

## Summary table

| Tool | What it's good at | What SignalScope borrows | What SignalScope rejects |
|---|---|---|---|
| Netmiko | Interactive CLI session mechanics | Prompt-regex mode tracking, per-vendor paging/enable handling | Response-string-only API with no inherent UI visibility |
| NAPALM | Multi-vendor read/write abstraction | Best-available-transport selection for reads | Templated/abstracted writes that hide literal CLI |
| Oxidized | Scheduled config backup & diff | SSH-first/Telnet-fallback, diff-on-change, per-vendor dump commands | Read-only-only scope (SignalScope extends to writes) |
| Zabbix/LibreNMS | Agentless SNMP monitoring at scale | GETBULK table-walk discovery pattern | No write path at all (by design — SignalScope adds supervised writes) |
| WebSSH2 | Browser-embedded terminal | xterm.js + ssh2 + WebSocket duplex transport | Terminal-only consumer (SignalScope taps the stream for GUI+audit too) |
