# Dell / Force10 enterprise MIBs — link-only

No raw `.mib`/`.my` files are vendored in this folder. Rationale (see [`../overview.md`](../overview.md#mib-licensingredistribution-note) for the full note):

- Dell hosts a MIB-file download for **OS9** (`Current_MIBs_pkb_en_US_1.tar`/`.zip`, covering releases 9.8.0.0–9.14.2.1) as a KB-article attachment, originally sourced from force10networks.com before the acquisition. **No license or redistribution terms are stated on that page** — the same situation this project already documented for Huawei ([`../../huawei/mibs/README.md`](../../huawei/mibs/README.md)) and Extreme ([`../../extreme/mibs/README.md`](../../extreme/mibs/README.md)).
- **OS10's own enterprise MIBs are not offered as a separate portal download** in what this session's research surfaced — Dell's own SmartFabric OS10 User Guide states they ship on-device at `/opt/dell/os10/snmp/mibs/`, which is a different distribution model from every other vendor in this docs tree (all of which offer, or a third party mirrors, a standalone downloadable file).

## Where to look instead

| MIB | Source | Notes |
|---|---|---|
| `DELLEMC-OS10-PRODUCTS-MIB` | [Observium MIB browser mirror](https://mibs.observium.org/mib/DELLEMC-OS10-PRODUCTS-MIB/) | Confirmed OID root `1.3.6.1.4.1.674.11000.5000.100.2` (`os10Products`, under Dell's own PEN 674 — see [`../mib-reference.md`](../mib-reference.md) for the PEN disambiguation). |
| `DELLEMC-OS10-SMI-MIB` | Referenced as an import by `DELLEMC-OS10-PRODUCTS-MIB` | Not independently located as a standalone downloadable file this session. |
| `DELLEMC-OS10-BGP4V2-MIB` | Referenced in Dell's own OS10 User Guide MIBs page (BGP monitoring section) | Confirms OS10 uses proprietary MIBs rather than standard `BGP4-MIB` for at least this protocol — a data point on OS10's general enterprise-MIB-over-standard-MIB posture, not independently generalized to L2/switching objects this session. |
| Legacy OS9/Force10 MIB set | [Dell Networking MIBs — KB 000181922](https://www.dell.com/support/kbdoc/en-us/000181922/dell-networking-mibs) (tar/zip attachment) | Covers OS9 releases 9.8.0.0–9.14.2.1, not OS10. |

If a future need arises to actually compile/load OS10's enterprise MIBs (e.g. for a MIB-browser integration or SNMP agent code generation), the most direct path is pulling them from a live OS10 device's own `/opt/dell/os10/snmp/mibs/` directory (per Dell's own documentation of that path) rather than a third-party mirror — do not copy Observium/LibreNMS-mirrored text into this repo without first confirming redistribution terms, per the same reasoning already applied to every other link-only vendor folder in this docs tree.
