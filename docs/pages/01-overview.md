# Industrial Layered Architecture (ILA)

**A practical, vendor-neutral framework for structuring industrial automation systems from field device to OT platform.**

---

## What Is ILA?

Industrial Layered Architecture (ILA) gives factories a shared way to decide where things belong: tags, PLC logic, HMI screens, recipes, historians, networks, security controls, and ownership.

ILA is not a product, platform, or replacement for ISA/IEC standards. It is the missing practical layer between standards and implementation. Standards tell you what good looks like. ILA helps a project team decide what to do on Monday morning.

Use ILA when you need an automation stack that the next technician, engineer, integrator, or OT architect can understand without detective work.

## Why ILA Exists

Most factory systems grow in layers over many years. A PLC program is extended during commissioning. A SCADA script gets added because it was faster than changing the PLC. A historian tag is renamed because a reporting tool needed a nicer label. A firewall rule is opened because production was waiting. None of these decisions are malicious. Together, they create systems where nobody can explain where the truth lives.

ILA exists to make those decisions explicit. Every signal has a name. Every piece of logic has a home. Every data flow has a direction. Every layer has an owner.

## The Five Layers

| # | Layer | What belongs here | Primary question |
|---|-------|-------------------|------------------|
| 1 | **Field** | Sensors, actuators, drives, robots, cameras, safety devices, IO-Link devices, wiring | What physically interacts with the process? |
| 2 | **Control** | PLCs, safety PLCs, motion controllers, PackML states, ISA-88 phases, interlocks, sequences | Where is machine behavior decided? |
| 3 | **Supervisory** | HMI, SCADA, operator commands, alarms, recipe entry, MES integration points | How do people and operations interact with the machine? |
| 4 | **Data** | Historians, databases, MQTT brokers, OPC UA aggregators, APIs, edge data services | What happened, when, and why? |
| 5 | **OT Platform** | Networks, firewalls, VLANs, identity, virtualization, backups, certificates, monitoring | What makes the stack secure, recoverable, and operable? |

The layers are not a maturity ladder. Layer 5 is not "above" Layer 4 in importance, and Layer 1 is not "less advanced" than Layer 4. The layers separate responsibilities so that each decision has a clear home.

## The Five Rules

ILA is built around five rules. Treat them as architectural guardrails: strong defaults that keep systems understandable. If a real plant must break a rule, document the reason, the risk, and the compensating control.

### Rule 1 - Tag Naming Is Architecture

> If your tags do not tell you where a device is and what it does, your architecture is invisible.

A tag is not just a label. It is a coordinate in the factory. ILA uses a hierarchical naming pattern so the same identity can travel from the field device, through the PLC and HMI, into the historian and documentation.

**Pattern:** `{Area}_{Unit}_{DeviceType}{Sequence}_{Attribute}`

**Example:** `IC01_CAM01_TriggerReq` means Inspection Cell 01, Camera 01, Trigger Request.

See [Layer 1 - Field](ILA-Layer1-Field.md) and [Layer 2 - Control](ILA-Layer2-Control.md).

### Rule 2 - No Process Logic in SCADA

> SCADA displays state and captures operator intent. It does not decide machine behavior.

Layer 3 may format values, navigate screens, validate user input, acknowledge alarms, and send operator commands. It must not contain process logic, interlocks, timers, sequencing, or calculations that determine equipment behavior. Those decisions belong in Layer 2.

The test is simple: if the SCADA platform were replaced, would the machine still run correctly under PLC control? If not, process logic has leaked into Layer 3.

See [Layer 3 - Supervisory](ILA-Layer3-Supervisory.md).

### Rule 3 - OT Owns the Stack, Security Is Everyone's Job

> OT infrastructure is not an IT afterthought, and security is not someone else's problem.

OT must own the architecture of the systems that affect production, safety, maintainability, and recovery. IT and security teams are essential partners, but decisions that can stop a line, corrupt a batch, or affect a safety state must include OT authority.

See [Layer 5 - OT Platform](ILA-Layer5-OTPlatform.md).

### Rule 4 - Data Flows Up, Commands Flow Down

> Process data moves upward. Operator and system commands move downward through controlled paths.

Layer 4 reads from the control system and stores, aggregates, or publishes data. It does not command equipment. Commands originate from authorized operators or approved supervisory workflows, pass through Layer 3, and are validated by Layer 2 before affecting Layer 1.

See [Layer 4 - Data](ILA-Layer4-Data.md).

### Rule 5 - The Control Layer Defines the Machine Structure

> The PLC structure defines the machine. The HMI, historian, alarms, and documentation align to it.

The PLC program is the clearest executable model of the machine. If the PLC has three stations, the HMI, alarms, historian folders, and troubleshooting documentation should reflect the same three stations. This reduces translation work across roles and tools.

See [Layer 2 - Control](ILA-Layer2-Control.md).

## Layer-Specific Rules

The five ILA rules define the architecture across the whole stack. Each layer also has its own five practical rules that translate the architecture into day-to-day engineering decisions:

| Layer | Rule theme |
|-------|------------|
| **Field** | Naming starts at installation, signals are standardized, devices fail safely, measurements are calibrated, and documentation follows the physical asset. |
| **Control** | Safety comes first, standards guide structure, PLC code stays readable, recipes are validated in control, and machines remain safe without SCADA. |
| **Supervisory** | Screens are designed for stress, SCADA owns no process logic, UI serves operators, alarms are actionable, and consistency beats creativity. |
| **Data** | Retention is planned, historian data is evidence, collection is passive, context is mandatory, and structures are versioned. |
| **OT Platform** | Segmentation is foundational, default deny is the access model, backups are proven, monitoring is mandatory, and infrastructure is reproducible. |

## Adopt ILA Incrementally

ILA is useful even when the existing factory is messy. Do not wait for a perfect greenfield project.

| Adoption level | Goal | Typical first step |
|----------------|------|--------------------|
| **ILA Basic** | Establish shared language and naming discipline | Apply Rule 1 to one machine or line |
| **ILA Standard** | Align PLC, HMI, alarms, and historian structure | Use Rules 1, 2, 4, and 5 in a machine design review |
| **ILA Advanced** | Standardize OT platform, security, data contracts, and governance | Apply all five layers across a line, site, or multi-site program |

For legacy systems, start where change is cheapest and value is visible: naming, documentation, alarm ownership, historian structure, and clear data-flow rules. Refactor PLC and SCADA behavior when a machine is already being modified, upgraded, or recommissioned.

## Standards Reference

ILA does not replace industrial standards. It maps them to practical decisions.

| Standard | What it covers | Where ILA uses it |
|----------|----------------|-------------------|
| **ISA 5.1** | Instrument and tag identification | Field and Control naming |
| **IEC 61131-3** | PLC languages and program structure | Control implementation |
| **PackML / ISA-TR88.00.02** | Machine state model | Control state model and HMI state display |
| **ISA-88 / IEC 61512** | Batch control models | Recipes, phases, procedures, batch records |
| **ISA-101** | HMI lifecycle and design | Supervisory interface design |
| **ISA-18.2** | Alarm management | Alarm presentation and rationalization |
| **OPC UA / IEC 62541** | Secure industrial data exchange | Layer 2/3/4 communication |
| **MQTT / Sparkplug B** | Industrial pub/sub data movement | Data layer and cloud/edge replication |
| **UNS (Unified Namespace)** | Shared industrial data namespace pattern | Layer 4 data organization, context, and cross-system discovery |
| **IEC 62443** | Industrial cybersecurity | OT zones, conduits, hardening, monitoring |
| **ISA-95 / IEC 62264** | Enterprise-control integration | MES, ERP, and cross-system data contracts |

For the detailed comparison, see [Standards Bridge](ILA-Standards-Bridge.md).

## How to Use This Framework

1. Start with [Getting Started](ILA-GettingStarted.md) to see ILA applied to one inspection cell.
2. Read each layer document when designing, reviewing, or troubleshooting that part of the stack.
3. Use the five rules as a checklist during design reviews, FAT/SAT, commissioning, and change control.
4. Record exceptions. A documented exception is manageable; an invisible exception becomes tribal knowledge.
5. Adopt incrementally. One well-structured machine is better than a site-wide standard nobody follows.

## Contributing

ILA is a living framework maintained by practitioners. Questions, corrections, challenges, implementation stories, and industry-specific examples are welcome.

- GitHub: [TheFactoryStack/ILA](https://github.com/TheFactoryStack/ILA)
- Contribution guide: [CONTRIBUTING.md](CONTRIBUTING.md)
- Governance: [GOVERNANCE.md](GOVERNANCE.md)

---

*ILA is a project by [TheFactoryStack](https://thefactorystack.com). The framework is free and open. Commercial offerings such as courses, certification, consulting, and starter kits are separate from the open framework.*
