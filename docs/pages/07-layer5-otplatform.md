# ILA Layer 5 — OT Platform

**Network infrastructure, firewalls, AD/LDAP, virtualization, backup — the foundation everything runs on.**

---

## Purpose

The OT Platform Layer is the infrastructure that enables every other layer to function. It provides the network, the servers, the authentication, and the security boundaries. When this layer is neglected, the entire stack is fragile, insecure, and unmaintainable.

This layer is governed most directly by **ILA Rule 3: OT owns the stack / Security is everyone's job.**

## What Belongs in Layer 5

- Network switches, routers, and firewalls (OT-specific)
- Network segmentation and VLAN design
- OT firewalls and DMZ architecture
- Active Directory / LDAP for centralized authentication
- Virtualization hosts and hypervisors
- Server infrastructure (physical and virtual)
- Backup and disaster recovery systems
- Time synchronization (NTP/PTP)
- Patch management (OT-appropriate cadence)
- Certificate management (for OPC UA, HTTPS, VPN)

## What Does NOT Belong in Layer 5

- PLC programs or process logic (that is Layer 2)
- SCADA applications (that is Layer 3)
- Historian or database applications (that is Layer 4)
- Business IT infrastructure (ERP, email, corporate network)

## Key Standards

### IEC 62443 — Industrial Cybersecurity

IEC 62443 is the defining standard for OT cybersecurity. It provides a comprehensive framework covering organization, system, and component security across the entire lifecycle.

**IEC 62443 structure:**

| Part | Scope | ILA Relevance |
|------|-------|---------------|
| 62443-1-x | General concepts, terminology | Foundation for security discussions |
| 62443-2-x | Policies and procedures | OT security governance, patch management |
| 62443-3-x | System security | Zone/conduit design, security levels |
| 62443-4-x | Component security | PLC, SCADA, device hardening |

**Zones and Conduits:**

IEC 62443 defines the concept of *zones* (groups of assets with the same security level) and *conduits* (communication paths between zones with defined security controls).

**Mapping ILA layers to IEC 62443 zones:**

```
┌─────────────────────────────────────────────┐
│  Enterprise Zone (IT)                       │
│  ERP, email, business systems               │
│  ── NOT part of ILA ──                      │
├─────────── DMZ / Firewall ──────────────────┤
│  OT DMZ                                     │
│  Historian replica, remote access gateway    │
├─────────── OT Firewall ────────────────────┤
│  ILA Layer 5 — OT Platform Zone             │
│  AD/LDAP, virtualization, NTP, backup       │
├─────────────────────────────────────────────┤
│  ILA Layer 4 — Data Zone                    │
│  Historian, MQTT broker, SQL databases      │
├─────────────────────────────────────────────┤
│  ILA Layer 3 — Supervisory Zone             │
│  SCADA servers, HMI clients                 │
├─────────────────────────────────────────────┤
│  ILA Layer 2 — Control Zone                 │
│  PLCs, safety controllers, motion           │
├─────────────────────────────────────────────┤
│  ILA Layer 1 — Field Zone                   │
│  Sensors, actuators, IO-Link, robots        │
└─────────────────────────────────────────────┘
```

**Security Levels (SL):**

IEC 62443 defines four security levels:

| SL | Protection Against | Typical ILA Application |
|----|-------------------|------------------------|
| SL 1 | Casual or coincidental violation | Development/lab environments |
| SL 2 | Intentional attack with low resources | Standard production environments |
| SL 3 | Sophisticated attack with moderate resources | Critical infrastructure, regulated plants |
| SL 4 | State-sponsored attack | Rarely applied in full; aspirational |

**ILA principle:** Every zone must have a defined target security level. The security level drives decisions about authentication, network controls, monitoring, and hardening within that zone.

### ISA-95 / IEC 62264 — The Purdue Model

The Purdue Model (from ISA-95) defines hierarchical levels for enterprise-to-control integration. ILA layers align with but are not identical to Purdue levels.

**Purdue-to-ILA mapping:**

| Purdue Level | Description | ILA Layer |
|-------------|-------------|-----------|
| Level 0 | Physical process | Layer 1 (Field) |
| Level 1 | Basic control (sensors, actuators, drives) | Layer 1 / Layer 2 boundary |
| Level 2 | Area supervisory control (PLCs, DCS) | Layer 2 (Control) |
| Level 3 | Site manufacturing operations (MES, historian) | Layer 3 + Layer 4 |
| Level 3.5 | DMZ | Layer 5 (OT Platform) |
| Level 4-5 | Enterprise (IT) | Outside ILA scope |

**ILA principle:** The Purdue Model provides the network segmentation rationale. ILA adds practical guidance for what goes in each zone and how layers communicate across boundaries.

## Network Design

### VLAN Design

Every ILA layer should be on its own VLAN (or set of VLANs for larger plants). Inter-VLAN traffic is controlled by a firewall — not just a router.

**Recommended VLAN structure:**

| VLAN | ILA Layer | Purpose | Example Subnet |
|------|-----------|---------|----------------|
| 10 | Layer 1 + 2 | Field/Control network | 10.10.10.0/24 |
| 20 | Layer 3 | SCADA/HMI network | 10.10.20.0/24 |
| 30 | Layer 4 | Data/Historian network | 10.10.30.0/24 |
| 40 | Layer 5 | Infrastructure services | 10.10.40.0/24 |
| 99 | DMZ | IT/OT boundary | 10.10.99.0/24 |

**ILA principle:** Layer 1 (field devices) and Layer 2 (PLCs) share a network segment because they need real-time, low-latency communication. All other layers are separated by firewall rules.

**Firewall rules follow ILA data flow (Rule 4):**

| Source | Destination | Direction | Allowed Traffic |
|--------|-------------|-----------|-----------------|
| Layer 3 (SCADA) | Layer 2 (PLC) | Down | OPC UA (write commands) |
| Layer 2 (PLC) | Layer 3 (SCADA) | Up | OPC UA (tag reads, subscriptions) |
| Layer 2 (PLC) | Layer 4 (Historian) | Up | OPC UA (tag reads) |
| Layer 4 (Historian) | Layer 2 (PLC) | **Blocked** | No writes from Layer 4 to Layer 2 |
| Layer 4 (Data) | DMZ | Up | Controlled data replication |
| Enterprise IT | Layer 2 | **Blocked** | Never direct IT-to-PLC access |

### IP Address Scheme

**ILA principle:** Use a consistent IP addressing scheme that makes any device's layer and purpose identifiable from its IP address.

**Pattern:** `10.{site}.{layer*10}.{device}`

| Device | IP | Meaning |
|--------|-----|---------|
| `10.1.10.101` | PLC 01, Site 1, Layer 1+2 | Control zone |
| `10.1.20.10` | SCADA Server, Site 1, Layer 3 | Supervisory zone |
| `10.1.30.10` | Historian, Site 1, Layer 4 | Data zone |
| `10.1.40.10` | AD Server, Site 1, Layer 5 | Infrastructure zone |

## Centralized Authentication (AD/LDAP)

**ILA principle:** A single identity source (Active Directory or LDAP) provides authentication for all systems that support it.

**Systems that should use centralized auth:**

| System | Layer | Auth Method |
|--------|-------|-------------|
| SCADA / Ignition | 3 | LDAP/AD integration |
| Historian | 4 | AD service accounts |
| SQL databases | 4 | AD-integrated authentication |
| Firewalls | 5 | RADIUS / LDAP |
| Hypervisor management | 5 | AD integration |
| Windows VMs | All | Domain-joined |

**Systems that typically cannot use centralized auth:**

| System | Layer | Reason |
|--------|-------|--------|
| PLCs | 2 | Most PLCs have local user management only |
| Robots | 1 | Proprietary auth, no LDAP support |
| Field devices | 1 | No auth capability or local only |

**ILA principle:** Where centralized auth is not possible, document the local credentials securely, enforce password policies, and include these devices in security audits. This is an honest reflection of real OT environments — not every device supports AD integration, and pretending otherwise is dishonest.

**Practical solutions for non-AD devices:**

For the PLCs, robots, and field devices that cannot join Active Directory, use a Privileged Access Management (PAM) solution or a credential vault (e.g., CyberArk, HashiCorp Vault, or even a well-secured KeePass database as a minimum). The key requirements: credentials are stored encrypted, access is logged, and passwords are rotated on a defined schedule. No shared sticky notes on the HMI cabinet. No default passwords left from commissioning.

For larger installations with hundreds of non-AD devices, consider a dedicated OT credential management workflow: a spreadsheet is not sufficient at scale. The PAM solution should support scheduled password rotation with change logging, role-based access (maintenance sees different credentials than engineering), break-glass procedures for emergency access during outages, and integration with your change management process.

## Virtualization

Virtualization is standard practice in modern OT. SCADA servers, historians, databases, and engineering workstations run as VMs on a shared hypervisor.

**What should be virtualized:**

- SCADA servers (Layer 3)
- Historian and SQL servers (Layer 4)
- AD/LDAP servers (Layer 5)
- Engineering workstations

**What should NOT be virtualized:**

- PLCs and safety controllers (Layer 2 — dedicated hardware)
- Field devices (Layer 1 — physical)
- Firewalls (dedicated appliance recommended; virtual firewall acceptable for lab/dev)

**Backup strategy:**

- Full VM snapshots on a defined schedule (daily or before changes)
- Backup storage on a separate physical device or NAS
- Retain at least 3 rotation generations
- Test restores periodically — a backup you have never restored is not a backup

## Time Synchronization

Accurate time is critical for alarm sequencing, batch records, historian data correlation, and forensic analysis.

**ILA time hierarchy:**

```
GPS / NTP external source
    └── OT NTP server (Layer 5)
        ├── SCADA servers (Layer 3)
        ├── Historian (Layer 4)
        ├── PLCs (Layer 2) — if NTP-capable
        └── Domain controllers (Layer 5)
```

**ILA principle:** One authoritative time source for the entire OT environment. All systems synchronize to the OT NTP server, not directly to the internet. PLCs that support NTP should use it; those that don't should be documented as potential drift risks.

## Patch Management

OT patching follows a different cadence than IT. Unplanned downtime from a bad patch is as damaging as a security breach.

**ILA patching principles:**

- Patch OT systems on a quarterly or semi-annual cycle, not monthly
- Test all patches in a staging/lab environment before production deployment
- Coordinate patches with planned maintenance windows
- Document all patches applied, including rollback procedures
- Prioritize patches for internet-facing or DMZ systems
- Legacy systems that cannot be patched must have compensating controls (network isolation, enhanced monitoring)

## OT Monitoring and Anomaly Detection

Prevention alone is not sufficient. IEC 62443 assumes breach will happen — detection and response are equally important.

**What to monitor in an OT environment:**

- **Network traffic:** Baseline normal traffic patterns (which devices talk to which, on which ports, at what volume). Anomalies — a PLC suddenly communicating with an unknown IP, or a historian initiating outbound connections it has never made before — are high-priority alerts.
- **OPC UA sessions:** Monitor for unexpected client connections to PLC OPC UA servers. An unknown client connecting to your PLC is a potential compromise.
- **Authentication events:** Failed login attempts on SCADA, AD, firewalls, and VPN. Brute-force patterns are early indicators.
- **Configuration changes:** PLC program downloads, firmware updates, firewall rule changes. All must be logged and correlated with authorized change windows.
- **USB and removable media:** Detect USB device insertions on engineering workstations and HMI terminals.

**Tools:** OT-specific monitoring platforms (Claroty, Nozomi Networks, Dragos) provide passive network monitoring purpose-built for industrial protocols. For smaller environments, a combination of firewall logging, Syslog aggregation, and Wireshark-based audits provides baseline visibility. The key is that *something* is watching — not that it is expensive.

**ILA principle:** OT monitoring is a Layer 5 responsibility. Monitoring agents must never impact real-time control performance. Use passive/span-port monitoring for control network traffic — never inline inspection that could add latency to PLC communication.

## Incident Response

When a security incident occurs — a detected breach, a ransomware infection, an unauthorized PLC program change — the OT team must have a pre-defined response plan.

**ILA incident response principles:**

**Who decides:** The OT team has final authority over the decision to take production systems offline. IT security may advise, but the OT team understands the physical consequences of shutting down a process mid-cycle (e.g., a furnace that cannot be safely stopped, a chemical batch that will be ruined). This decision authority must be agreed with IT *before* an incident, not negotiated during one.

**Isolation, not shutdown:** The first response to a suspected network breach is to isolate the affected zone at the firewall — not to power off PLCs or abort running processes. Layer 2 devices should continue to operate safely in isolation while the incident is investigated.

**Evidence preservation:** Do not reimage or reboot affected systems before capturing forensic evidence. Take VM snapshots, export firewall logs, and preserve network captures. Coordinate with your incident response team or external forensic specialists.

**Communication plan:** Define who is notified (OT lead, IT security, plant management, and if required: regulatory bodies) and through which channels (not through the potentially compromised OT network).

**Post-incident:** Conduct a root cause analysis. Update firewall rules, credentials, and monitoring baselines. Document lessons learned and update the incident response plan.

## Multi-Site Considerations

For organizations operating multiple plants, Layer 5 must address standardization across sites.

**Tag naming across sites:** Add a site prefix to the ILA tag naming convention: `{Site}_{Area}_{Unit}_{DeviceType}{Seq}_{Attribute}`. Example: `S01_IC01_CAM01_TriggerReq` (Site 01, Inspection Cell 01, Camera 01). This enables cross-site data aggregation at Layer 4 without naming collisions.

**Network design across sites:** Each site maintains its own OT network with its own firewall and DMZ. Inter-site communication for data replication (historian to cloud, cross-site dashboards) flows through site DMZs — never direct site-to-site OT network connections. Use site-to-site VPN or SD-WAN through the DMZ tier only.

**Centralized vs. distributed AD:** For multi-site environments, decide whether to run a single AD forest with site-specific domain controllers, or independent AD domains per site. ILA recommends site-local domain controllers at minimum — if the WAN link goes down, the site must continue to authenticate users locally.

## Practical Checklist

- [ ] OT network is owned and operated by the OT team (Rule 3)
- [ ] IEC 62443 zones and conduits are defined and documented
- [ ] Each ILA layer is on a separate VLAN with firewall rules
- [ ] Firewall rules enforce Rule 4 (data up, commands down — Layer 4 cannot write to Layer 2)
- [ ] IP addressing scheme is consistent and self-documenting
- [ ] Centralized authentication (AD/LDAP) is used where possible
- [ ] Local credentials on PLCs and field devices are managed via PAM or credential vault
- [ ] VM backups run on a defined schedule with tested restores
- [ ] NTP is configured with a single authoritative OT time source
- [ ] Patch management follows an OT-appropriate cadence with testing
- [ ] DMZ exists between IT and OT networks
- [ ] Remote access is through a controlled gateway (VPN, jump host) — never direct
- [ ] OT network monitoring is deployed (passive, not inline on control network)
- [ ] OPC UA sessions, auth events, and config changes are logged and reviewed
- [ ] Incident response plan exists with clear OT team decision authority
- [ ] IT and OT have pre-agreed escalation and isolation procedures
- [ ] *Multi-site:* Tag naming includes site prefix for cross-site consistency
- [ ] *Multi-site:* Each site has local AD domain controllers for WAN-independent auth
- [ ] *Multi-site:* Inter-site data flows through DMZ only — no direct OT-to-OT links

---

*Back to [ILA Overview](01-overview.md) | Previous: [Layer 4 — Data](06-layer4-data.md)*
