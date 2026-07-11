# Huawei enterprise MIBs — link-only

No raw `.mib`/ASN.1 files are vendored in this folder. Rationale (see [../overview.md](../overview.md#mib-licensingredistribution-note) for the full note):

- Huawei's `support.huawei.com` MIB reference pages document objects in human-readable form but do not state an open-redistribution license for the underlying MIB module text, unlike the IETF RFCs and the IEEE 802.1AB LLDP-MIB vendored under [../../../mibs/](../../../mibs/) (see that folder's `README.md` for the license basis used there).
- Raw Huawei `.mib` files circulate via third-party monitoring-tool mirrors, but each carries a `Copyright (C) HUAWEI TECHNOLOGIES. All rights reserved.` header with no explicit redistribution grant — those projects bundle the files for their own tool's functional use, which isn't the same as a permissive release this project can safely re-vendor.

## Where to look instead

| MIB | Huawei's own reference page | Third-party mirror (unlicensed for redistribution — reference only) |
|---|---|---|
| `HUAWEI-PORT-MIB` | [support.huawei.com](https://support.huawei.com/enterprise/en/doc/EDOC1000178181/69628d40/huawei-port-mib) | [LibreNMS mirror](https://github.com/librenms/librenms/blob/master/mibs/huawei/HUAWEI-PORT-MIB) |
| `HUAWEI-L2VLAN-MIB` | [support.huawei.com](https://support.huawei.com/enterprise/en/doc/EDOC1000178181/62fde089/huawei-l2vlan-mib) | [Observium MIB browser](https://mibs.observium.org/mib/HUAWEI-L2VLAN-MIB/) |
| `HUAWEI-CONFIG-MAN-MIB` | [support.huawei.com](https://support.huawei.com/enterprise/en/doc/EDOC1000178181/434f229f/huawei-config-man-mib) | — |
| `HUAWEI-MIB` (top-level enterprise tree) | — | [LibreNMS mirror](https://github.com/librenms/librenms/blob/master/mibs/huawei/HUAWEI-MIB) |
| Broader Huawei MIB set | [MIB Overview — S1720/S2700/S5700/S6720 MIB Reference](https://support.huawei.com/enterprise/en/doc/EDOC1000178181/2f6c0513/mib-overview) | [LibreNMS `mibs/huawei/` directory](https://github.com/librenms/librenms/tree/master/mibs/huawei) |

If a future need arises to actually compile/load these MIBs (e.g. for a MIB-browser integration or SNMP agent code generation), get them directly from a device via `tftp`/`display` on Huawei's documentation site, or contact Huawei for licensing terms — do not copy the LibreNMS/Observium-mirrored text into this repo without first confirming redistribution terms with Huawei, since their presence in another GPL/AGPL-licensed tool's repo does not itself grant a redistribution license for the vendor-copyrighted MIB content.
