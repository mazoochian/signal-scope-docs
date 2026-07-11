# Juniper MIBs — link-only (no vendored files)

Unlike the cross-vendor standard MIBs vendored under [`signal-scope-docs/mibs/`](../../../mibs/), no raw Juniper `.mib`/`.txt` files are vendored in this directory. Rationale (full detail in [../overview.md](../overview.md#mib-licensingredistribution-note--why-mibs-is-link-only-for-juniper)):

- Juniper distributes its MIB files through [MIB Explorer](https://apps.juniper.net/mib-explorer/) and the support portal, neither of which carries a clear redistribution license.
- We checked whether open-source NMS projects mirror these under clearer terms. Both do mirror them, but neither resolves the licensing question:
  - [LibreNMS `librenms-mibs/JUNIPER-MIB`](https://github.com/librenms/librenms-mibs/blob/master/JUNIPER-MIB) and `librenms/librenms`'s `mibs/junos/` directory (e.g. `JUNIPER-VLAN-MIB`, `JUNIPER-SMI`) — each file's own header reads:
    ```
    -- Juniper Enterprise Specific MIB: Chassis MIB
    --
    -- Copyright (c) 1998-2008, Juniper Networks, Inc.
    -- All rights reserved.
    --
    -- The contents of this document are subject to change without notice.
    ```
    No explicit permission-to-redistribute grant accompanies this notice. LibreNMS's own project license covers LibreNMS's code, not this bundled third-party text.
  - [Observium's MIB browser](https://mibs.observium.org/mib/JUNIPER-MIB/) hosts the same files, same provenance/ambiguity.
- Given the ambiguity is unresolved either way, we chose **not** to vendor copies here, per the assignment's explicit guidance that link-only is an acceptable, expected outcome for this vendor.

## Where to actually get the MIB text

- [MIB Explorer](https://apps.juniper.net/mib-explorer/) — official Juniper tool, browse/search by platform and release.
- [SNMP MIBs Supported by Junos OS and Junos OS Evolved](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/snmp-mibs-supported-by-junos-os-and-junos-os-evolved.html) — canonical standard-MIB support list.
- [Enterprise-Specific MIBs Overview](https://www.juniper.net/documentation/en_US/junos/topics/concept/enterprise-specific-mibs-overview.html) — canonical enterprise-MIB (JUNIPER-MIB family) catalog.

See [../mib-reference.md](../mib-reference.md) for the OID-level documentation of what SignalScope actually needs from these MIBs.
