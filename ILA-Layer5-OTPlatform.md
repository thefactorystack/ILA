# ILA Layer 5 - OT Platform

**Networks, firewalls, identity, virtualization, backup, certificates, time synchronization, patching, monitoring, and incident response.**

---

## Purpose

The OT Platform Layer is the foundation that lets the other layers run securely and recoverably. It provides network segmentation, identity, servers, virtualization, backups, certificates, time, patching, monitoring, and incident response boundaries.

When Layer 5 is neglected, the factory may still run, but it becomes fragile: unknown firewall rules, unpatched servers, shared passwords, no restore tests, no time sync, and no clear authority during incidents.

This layer is governed most directly by **ILA Rule 3: OT owns the stack / Security is everyone's job.**

## What Belongs Here

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
- Remote access architecture
- Logging, monitoring, and incident response tooling

## What Does Not Belong Here

- PLC programs or process logic (that is Layer 2)
- SCADA applications (that is Layer 3)
- Historian or database applications (that is Layer 4)
- Business IT infrastructure (ERP, email, corporate network)
- Process ownership decisions that belong to operations and engineering

## OT Platform Rules

These rules keep the infrastructure controlled, observable, recoverable, and rebuildable.

| Rule | Principle | Meaning |
|------|-----------|---------|
| R1 | **Network segmentation is foundational** | Segmentation is not a finishing touch. It is the prerequisite for controlled traffic, security monitoring, incident containment, and safe integration between layers. |
| R2 | **Default deny. Always.** | If traffic, access, or remote connectivity is not explicitly required, approved, and documented, it is blocked. Allow rules must be intentional, reviewed, and owned. |
| R3 | **Backups must be proven, not assumed** | If you have not performed a restore test, you do not have a backup; you have a hope. Restore tests must cover PLC projects, SCADA, historians, VMs, databases, firewall configs, and certificates where applicable. |
| R4 | **Monitoring is mandatory** | If you do not know what is happening, you have already lost control of the platform. Monitor traffic, authentication, configuration changes, remote access, backups, and critical service health. |
| R5 | **Everything must be documented and reproducible** | No hidden configurations. No cowboy fixes. Infrastructure must be rebuildable from documented configuration, backups, versioned exports, and known credentials. |

## Key Standards

### IEC 62443 - Industrial Cybersecurity

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
│  -- NOT part of ILA --                      │
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

**ILA principle:** Every zone should have a target security level based on risk. ILA provides a default zone model; IEC 62443 risk assessment refines it for the site.

### ISA-95 / IEC 62264 - The Purdue Model

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

**ILA principle:** The Purdue Model provides segmentation rationale. ILA adds practical guidance for what belongs in each zone and how layers communicate across boundaries.

## Network Design

### VLAN Design

Every ILA layer should have an intentional network boundary. In mature environments this usually means separate VLANs or zones with firewall control. In smaller environments, the first step may be documenting existing traffic and separating the highest-risk paths before redesigning the whole network.

**Recommended VLAN structure:**

| VLAN | ILA Layer | Purpose | Example Subnet |
|------|-----------|---------|----------------|
| 10 | Layer 1 + 2 | Field/Control network | 10.10.10.0/24 |
| 20 | Layer 3 | SCADA/HMI network | 10.10.20.0/24 |
| 30 | Layer 4 | Data/Historian network | 10.10.30.0/24 |
| 40 | Layer 5 | Infrastructure services | 10.10.40.0/24 |
| 99 | DMZ | IT/OT boundary | 10.10.99.0/24 |

**ILA principle:** Layer 1 and Layer 2 often share a control zone because field devices and PLCs need deterministic, low-latency communication. Layer 3, Layer 4, infrastructure services, and external access should be separated and controlled.

**Firewall rules follow ILA data flow (Rule 4):**

| Source | Destination | Direction | Allowed Traffic |
|--------|-------------|-----------|-----------------|
| Layer 3 (SCADA) | Layer 2 (PLC) | Down | OPC UA (write commands) |
| Layer 2 (PLC) | Layer 3 (SCADA) | Up | OPC UA (tag reads, subscriptions) |
| Layer 2 (PLC) | Layer 4 (Historian) | Up | OPC UA (tag reads) |
| Layer 4 (Historian) | Layer 2 (PLC) | **Blocked** | No command writes from Layer 4 to Layer 2 |
| Layer 4 (Data) | DMZ | Up | Controlled data replication |
| Enterprise IT | Layer 2 | **Blocked** | Never direct IT-to-PLC access |

### IP Address Scheme

**ILA principle:** Use a consistent IP addressing scheme that makes a device's site, zone, and purpose easier to identify. IP addresses support the asset inventory; they do not replace it.

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

**ILA principle:** Where centralized authentication is not possible, manage local credentials deliberately. This is normal in OT. Pretending every PLC, robot, drive, or field device supports enterprise identity creates blind spots.

**Practical solutions for non-AD devices:**

For PLCs, robots, and field devices that cannot join Active Directory, use a PAM solution or credential vault where possible. For smaller sites, a well-controlled encrypted vault is better than a spreadsheet or shared commissioning password. The key requirements are encrypted storage, access control, change logging, and a defined rotation process.

For larger installations with hundreds of non-AD devices, consider a dedicated OT credential management workflow: a spreadsheet is not sufficient at scale. The PAM solution should support scheduled password rotation with change logging, role-based access (maintenance sees different credentials than engineering), break-glass procedures for emergency access during outages, and integration with your change management process.

## Virtualization

Virtualization is standard practice in modern OT, but it must be operated like production infrastructure. SCADA servers, historians, databases, domain controllers, and engineering workstations can run as VMs when latency, supportability, backup, and recovery requirements are understood.

**What should be virtualized:**

- SCADA servers (Layer 3)
- Historian and SQL servers (Layer 4)
- AD/LDAP servers (Layer 5)
- Engineering workstations

**What should NOT be virtualized:**

- PLCs and safety controllers (Layer 2 — dedicated hardware)
- Field devices (Layer 1 — physical)
- Firewalls in production, unless the site has a validated virtual firewall architecture

**Backup strategy:**

- Full VM backups or snapshots on a defined schedule
- Backup storage on a separate physical device or NAS
- Retain enough restore points to cover operational and ransomware scenarios
- Test restores periodically — a backup that has never been restored is only a hope

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

OT patching follows a different risk profile than IT patching. A bad patch that stops production can be as damaging as an unpatched vulnerability. That does not mean "never patch"; it means patch deliberately.

**ILA patching principles:**

- Patch OT systems on a risk-based cadence, commonly quarterly or semi-annually
- Test all patches in a staging/lab environment before production deployment
- Coordinate patches with planned maintenance windows
- Document all patches applied, including rollback procedures
- Prioritize internet-facing, remote-access, and DMZ systems
- Legacy systems that cannot be patched must have compensating controls (network isolation, enhanced monitoring)

## OT Monitoring and Anomaly Detection

Prevention alone is not sufficient. IEC 62443 assumes breach will happen — detection and response are equally important.

**What to monitor in an OT environment:**

- **Network traffic:** Baseline normal traffic patterns (which devices talk to which, on which ports, at what volume). Anomalies — a PLC suddenly communicating with an unknown IP, or a historian initiating outbound connections it has never made before — are high-priority alerts.
- **OPC UA sessions:** Monitor for unexpected client connections to PLC OPC UA servers. An unknown client connecting to your PLC is a potential compromise.
- **Authentication events:** Failed login attempts on SCADA, AD, firewalls, and VPN. Brute-force patterns are early indicators.
- **Configuration changes:** PLC program downloads, firmware updates, firewall rule changes. All must be logged and correlated with authorized change windows.
- **USB and removable media:** Detect USB device insertions on engineering workstations and HMI terminals.

**Tools:** OT-specific monitoring platforms provide passive visibility for industrial protocols. Smaller environments can start with firewall logging, Syslog, asset inventory, switch monitoring, and periodic packet captures. The key is that someone can answer: what normally talks to what, and what changed?

**ILA principle:** OT monitoring is a Layer 5 responsibility. Monitoring must not endanger real-time control performance. Prefer passive or span-port monitoring for control networks. Be careful with inline inspection where latency or failure modes could affect production.

## Incident Response

When a security incident occurs — a detected breach, a ransomware infection, an unauthorized PLC program change — the OT team must have a pre-defined response plan.

**ILA incident response principles:**

**Who decides:** The OT team has final authority over decisions that affect production state. IT security may advise, but OT understands the physical consequences of stopping a process mid-cycle. This authority must be agreed before an incident, not negotiated during one.

**Isolation, not panic shutdown:** The first response to a suspected network breach is usually to isolate affected zones, not power off PLCs or abort running processes without understanding the physical consequence. Layer 2 should continue operating safely where possible.

**Evidence preservation:** Do not reimage or reboot affected systems before capturing forensic evidence. Take VM snapshots, export firewall logs, and preserve network captures. Coordinate with your incident response team or external forensic specialists.

**Communication plan:** Define who is notified (OT lead, IT security, plant management, and if required: regulatory bodies) and through which channels (not through the potentially compromised OT network).

**Post-incident:** Conduct a root cause analysis. Update firewall rules, credentials, and monitoring baselines. Document lessons learned and update the incident response plan.

## Multi-Site Considerations

For organizations operating multiple plants, Layer 5 must address standardization across sites.

**Tag naming across sites:** Add a site prefix to the ILA tag naming convention: `{Site}_{Area}_{Unit}_{DeviceType}{Seq}_{Attribute}`. Example: `S01_IC01_CAM01_TriggerReq` (Site 01, Inspection Cell 01, Camera 01). This enables cross-site data aggregation at Layer 4 without naming collisions.

**Network design across sites:** Each site maintains its own OT network with its own firewall and DMZ. Inter-site communication for data replication (historian to cloud, cross-site dashboards) flows through site DMZs — never direct site-to-site OT network connections. Use site-to-site VPN or SD-WAN through the DMZ tier only.

**Centralized vs. distributed AD:** For multi-site environments, decide whether to run a single AD forest with site-specific domain controllers, or independent AD domains per site. ILA recommends site-local domain controllers at minimum — if the WAN link goes down, the site must continue to authenticate users locally.

## Minimum Viable Layer 5

Not every site can implement a mature IEC 62443 program on day one. Start with controls that reduce the most operational risk:

| Level | Minimum outcome |
|-------|-----------------|
| Basic | Asset inventory, network diagram, unique accounts where possible, known backups, documented remote access |
| Standard | Layered VLANs/zones, firewall rules, centralized identity, tested restores, NTP, patch windows |
| Advanced | IEC 62443 zone/conduit model, OT monitoring, PAM, certificate lifecycle, incident exercises, multi-site standards |

The goal is not to buy tools. The goal is to make the OT environment understandable, recoverable, and governable.

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
- [ ] Minimum viable controls are defined for smaller or legacy sites
- [ ] Exceptions are risk-reviewed and time-bound where possible

---

*Back to [ILA Overview](ILA-Overview.md) | Previous: [Layer 4 - Data](ILA-Layer4-Data.md)*
