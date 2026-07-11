# Extreme Networks (EXOS) enterprise MIBs — link-only

No raw `.mib`/ASN.1 files are vendored in this folder. Rationale (see [../overview.md](../overview.md#mib-licensing--redistribution-note) for the full note):

- Extreme's own MIB distribution (`extremenetworks.com/support/policies/mibs/` and the Extreme customer support portal) requires an authenticated account for several product lines and, for some, a serial/agreement number — not an unambiguous open-redistribution posture like the IETF RFCs and IEEE 802.1AB LLDP-MIB vendored under [../../../mibs/](../../../mibs/) (see that folder's own licensing basis).
- Third-party monitoring-tool mirrors (LibreNMS, Observium, mibbrowser.online, oidview.com) do host individual Extreme `.mib`/`.my` files, apparently sourced from Extreme's own historical public releases, but no explicit redistribution grant for those copies was found this session — same situation as the Huawei MIBs (see [`../../huawei/mibs/README.md`](../../huawei/mibs/README.md) for the identical reasoning applied there).

## Where to look instead

| MIB | Extreme's own reference page | Third-party mirror (unlicensed for redistribution — reference only) |
|---|---|---|
| `EXTREME-SYSTEM-MIB` | [documentation.extremenetworks.com](https://documentation.extremenetworks.com/exos_30.5/GUID-7B2EA11E-6CE5-4822-8F22-511B7D4F8D70.shtml) | [Observium MIB browser](https://mibs.observium.org/mib/EXTREME-SYSTEM-MIB/) |
| `EXTREME-VLAN-MIB` | — (existence confirmed only via third-party mirror this session) | [Observium MIB browser](https://mibs.observium.org/mib/EXTREME-VLAN-MIB/) |
| `EXTREME-SNMPv3-MIB` | [documentation.extremenetworks.com](https://documentation.extremenetworks.com/exos_22.5/GUID-70F85543-2561-48B6-BA62-AE43D120CD1E.shtml) | [Observium MIB browser](https://mibs.observium.org/mib/EXTREME-SNMPV3-MIB/) |
| `EXTREME-STP-EXTENSIONS-MIB` | — (existence confirmed only via third-party mirror search this session) | [oidview.com Extreme MIB index (PEN 1916)](http://www.oidview.com/mibs/1916/md-1916-1.html) |
| `EXTREME-BASE-MIB` (top-level enterprise tree) | — | [LibreNMS mirror](https://github.com/librenms/librenms/blob/master/mibs/extreme/EXTREME-BASE-MIB) |
| Broader Extreme MIB set | [Extreme MIB download policy page](https://www.extremenetworks.com/support/policies) (requires account for several products) | [LibreNMS `mibs/extreme/` directory](https://github.com/librenms/librenms/tree/master/mibs/extreme) |

## Ruckus ICX / FastIron (separate product line, separate PEN — noted for completeness, not covered by this vendor folder)

Ruckus ICX runs FastIron, not EXOS, and its enterprise objects live under the legacy Foundry PEN **1991** (`FOUNDRY-SN-ROOT-MIB`), not Extreme's own `1916`. If SignalScope later adds ICX support, its MIBs should get their own folder rather than being merged into this one — see [`../overview.md`](../overview.md) for why the two dialects/MIB trees are kept separate.

| MIB | Ruckus's own reference | Third-party mirror |
|---|---|---|
| FastIron SNMP MIB Reference (full set, per-release PDF) | [Ruckus support portal, e.g. FastIron 08.0.95](https://support.ruckuswireless.com/documents/3463-fastiron-08-0-95-ga-snmp-mib-reference) | — |
| `FOUNDRY-SN-ROOT-MIB` | — | [LibreNMS mirror](https://github.com/librenms/librenms/blob/master/mibs/brocade/FOUNDRY-SN-ROOT-MIB) |

If a future need arises to actually compile/load these MIBs (e.g. for a MIB-browser integration or SNMP agent code generation), get them directly from an EXOS device's own shipped MIB bundle (EXOS ships its MIBs alongside the firmware image) or via an authenticated Extreme support-portal download — do not copy the LibreNMS/Observium-mirrored text into this repo without first confirming redistribution terms with Extreme, since presence in another project's repo does not itself grant a redistribution license for the vendor-copyrighted MIB content.
