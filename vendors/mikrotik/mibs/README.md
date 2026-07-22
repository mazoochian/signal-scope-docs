# MikroTik enterprise MIB — vendored

Unlike most other vendor `mibs/` folders in this docs tree (Huawei, Extreme, Aruba — see their own `mibs/README.md` files), this one **does** vendor a raw MIB file, because a clearer redistribution basis exists here than for those vendors. See [`../overview.md`](../overview.md#mib-licensingredistribution-note) for the full reasoning.

| File | MIB | Source | License/redistribution basis |
|---|---|---|---|
| `MIKROTIK-MIB.txt` | `MIKROTIK-MIB` (enterprise PEN 14988) | Mirrored from [LibreNMS](https://github.com/librenms/librenms/blob/master/mibs/mikrotik/MIKROTIK-MIB), `master` branch, module revision `202505190000Z` (2025-05-19) | LibreNMS is GPLv3-licensed (explicit, permits redistribution with attribution) — a clearer basis than MikroTik's own MIB download page, which links only to its general software EULA with no separate MIB redistribution grant. |

**Caveat, repeated from `overview.md`**: this is a third-party mirror snapshot, not vendor-original text, and may lag or lead the exact OID set on any given RouterOS version. Cross-check any write-path-critical OID against a live device's `print oid` output (see `overview.md`'s "Distinctive introspection feature" section) before treating a number from this file as authoritative.
