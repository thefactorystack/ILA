# Industrial Layered Architecture (ILA)

**A practitioner-written, brand-agnostic governance framework for industrial automation.**

---

## What Is ILA?

Industrial Layered Architecture (ILA) is a five-layer framework that gives OT teams a shared language and a clear set of rules for building, structuring, and maintaining factory automation systems — regardless of vendor, PLC brand, or SCADA platform.

ILA is not a product. It is a way of thinking about where things belong, who owns what, and how data and commands move through a plant.

## Why ILA Exists

Most factory floors grow organically. Tag names are invented on the spot. Business logic creeps into HMI screens. Nobody can explain where the "source of truth" lives for a given recipe or batch ID. When something breaks at 2 AM, the on-call engineer spends more time figuring out *where* the logic lives than *what* went wrong.

ILA exists to fix that. It gives every signal, every screen, and every data flow a home — so the system is understandable by the next person who touches it.

## The Five Layers

| # | Layer | Scope | Key Standards |
|---|-------|-------|---------------|
| 1 | **Field** | Sensors, actuators, drives, robots, vision systems — anything with a wire or a signal | ISA 5.1, IO-Link, industrial Ethernet protocols |
| 2 | **Control** | PLCs, safety controllers, motion controllers — the layer that owns the process logic | IEC 61131-3, PackML / ISA-88, ISA 5.1 tag naming |
| 3 | **Supervisory** | SCADA, HMI, MES integration points — visualization and operator interaction, **not** logic | ISA-101, PackML HMI state models |
| 4 | **Data** | Historians, databases, MQTT brokers, OPC UA aggregation — moving and storing process data | OPC UA, MQTT/Sparkplug B, SQL, time-series DBs |
| 5 | **OT Platform** | Network infrastructure, firewalls, AD/LDAP, virtualization, backup — the foundation everything runs on | IEC 62443, Purdue Model / ISA-95 zones, network segmentation |

Each layer has its own dedicated document with practical guidance, standard references, and implementation patterns.

## The Five Rules

ILA is governed by five rules. These are non-negotiable principles that keep the architecture clean and maintainable.

### Rule 1 — Tag Naming Is Architecture

> If your tags don't tell you where a device is and what it does, your architecture is invisible.

A tag name is not a label — it is a coordinate in your system. ILA enforces ISA 5.1-based hierarchical naming so that every signal can be traced from the field device to the historian without ambiguity.

**Pattern:** `{Area}_{Unit}_{DeviceType}{Sequence}_{Attribute}`

Example: `IC01_CAM01_TriggerReq` — Inspection Cell 01, Camera 01, Trigger Request.

See: [Layer 1 — Field](ILA-Layer1-Field.md) and [Layer 2 — Control](ILA-Layer2-Control.md) for naming conventions.

### Rule 2 — No Programming in SCADA

> SCADA displays state. SCADA does not create state.

The Supervisory Layer (Layer 3) is for visualization, alarming, and operator commands. It must never contain process logic, calculations, or conditional branching that determines machine behavior. All logic belongs in the Control Layer (Layer 2).

If your SCADA script decides whether a valve opens — you have a Layer 2 problem living in Layer 3.

See: [Layer 3 — Supervisory](ILA-Layer3-Supervisory.md)

### Rule 3 — OT Owns the Stack / Security Is Everyone's Job

> OT infrastructure is not an IT afterthought. And security is not someone else's problem.

The OT team owns and operates every layer of the stack — from field wiring to network segmentation. But security awareness must exist at every layer and in every role. IEC 62443 provides the structure; ILA makes it practical.

See: [Layer 5 — OT Platform](ILA-Layer5-OTPlatform.md)

### Rule 4 — Data Flows Up, Commands Flow Down

> Sensors report upward. Setpoints and commands travel downward. Never sideways, never skipping layers.

This is the fundamental data flow discipline. A historian reads from OPC UA tags exposed by the Control Layer — it never writes directly to a field device. An operator sends a command through SCADA, which passes it to the PLC, which acts on the field device. No shortcuts.

See: [Layer 4 — Data](ILA-Layer4-Data.md)

### Rule 5 — Structure Is Dictated by the Control Layer

> The PLC program structure defines the machine. Everything else aligns to it.

The Control Layer is the single source of truth for machine structure. Tag naming, HMI screen hierarchy, historian folder structure, and alarm grouping all derive from how the PLC program is organized. If the PLC says there are three stations, the entire stack reflects three stations.

See: [Layer 2 — Control](ILA-Layer2-Control.md)

## Standards Reference

ILA does not replace standards — it maps them to practical decisions. Here is how the key standards relate to ILA:

| Standard | What It Covers | ILA Layers |
|----------|---------------|------------|
| **ISA 5.1** | Instrument and tag identification | 1, 2 |
| **IEC 61131-3** | PLC programming languages and structure | 2 |
| **PackML / ISA-TR88.00.02** | Machine state model (17 states) | 2, 3 |
| **ISA-88 / IEC 61512** | Batch control: physical, procedural, and process models | 2, 3 |
| **ISA-101** | HMI design and alarm management | 3 |
| **OPC UA / IEC 62541** | Secure, structured data exchange | 2, 3, 4 |
| **MQTT / Sparkplug B** | Lightweight pub/sub messaging for industrial data | 4 |
| **IEC 62443** | Industrial cybersecurity | 5 (applies to all) |
| **ISA-95 / IEC 62264** | Enterprise-to-control integration levels (Purdue Model) | 3, 4, 5 |

For a detailed comparison of how ILA's decisions relate to the standards, see [ILA Standards Bridge](ILA-Standards-Bridge.md).

## How to Use This Framework

1. **New to ILA?** Start with the [Getting Started](ILA-GettingStarted.md) guide — a hands-on walkthrough applying all five layers to a single machine.
2. **Read the overview** (this document) to understand the layers and rules.
3. **Read each layer document** for practical guidance on implementation.
4. **Read the [Standards Bridge](ILA-Standards-Bridge.md)** to understand where ILA makes decisions that the standards leave open.
5. **Use the rules as a checklist** during design reviews, code reviews, and commissioning.
6. **Adopt incrementally** — ILA works even if you start with just Rule 1 (tag naming) on a single machine.

## Contributing

ILA is a living document maintained by practitioners. Contributions, feedback, and real-world implementation stories are welcome.

- GitHub: [TheFactoryStack/ILA](https://github.com/TheFactoryStack/ILA)
- Hashtag: #TheFactoryStack
- Community: Discord (coming soon)

---

*ILA is a project by [TheFactoryStack](https://thefactorystack.com). The framework is free and open. Commercial offerings (courses, certification, consulting) are separate from the framework itself.*
