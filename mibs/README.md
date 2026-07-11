# Vendored Standard MIBs

Raw text for the cross-vendor standard MIBs discussed in [../00-architecture/standard-mibs.md](../00-architecture/standard-mibs.md). These are shared by every vendor and are therefore vendored once here rather than duplicated under each `vendors/<name>/mibs/` folder.

| File | MIB | Source | License/redistribution basis |
|---|---|---|---|
| `rfc3418-SNMPv2-MIB.txt` | SNMPv2-MIB | [RFC 3418](https://www.rfc-editor.org/rfc/rfc3418.txt) | IETF RFC — freely redistributable per IETF Trust rules |
| `rfc2863-IF-MIB.txt` | IF-MIB | [RFC 2863](https://www.rfc-editor.org/rfc/rfc2863.txt) | IETF RFC |
| `rfc4188-BRIDGE-MIB.txt` | BRIDGE-MIB | [RFC 4188](https://www.rfc-editor.org/rfc/rfc4188.txt) | IETF RFC |
| `rfc4363-Q-BRIDGE-MIB.txt` | Q-BRIDGE-MIB | [RFC 4363](https://www.rfc-editor.org/rfc/rfc4363.txt) | IETF RFC |
| `rfc6933-ENTITY-MIB.txt` | ENTITY-MIB | [RFC 6933](https://www.rfc-editor.org/rfc/rfc6933.txt) | IETF RFC |
| `rfc2819-RMON-MIB.txt` | RMON-MIB | [RFC 2819](https://www.rfc-editor.org/rfc/rfc2819.txt) | IETF RFC |
| `ieee802.1AB-LLDP-MIB.txt` | LLDP-MIB | [IEEE 802.1 public MIB repository](http://www.ieee802.org/1/files/public/MIBs/LLDP-MIB-200505060000Z.txt) (mirrored via NVIDIA's public docs) | IEEE-published standard MIB, publicly distributed for third-party tooling use |

Each file is the full RFC (for IETF ones) or the MIB module text (for the IEEE one) as published at the source — the ASN.1 MIB module definition is embedded within the RFC text, not extracted separately, so the RFC's own descriptive text/context is preserved alongside the object definitions.

Vendor-specific enterprise MIBs are **not** vendored here — see each `vendors/<name>/mibs/` folder and that vendor's `overview.md` for the redistribution basis used (or not used) for that vendor's proprietary MIB files.
