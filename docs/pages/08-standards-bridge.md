# ILA Standards Bridge

**What the standard says vs. what ILA decides.**

---

## Why This Document Exists

Industrial standards are written to be universally applicable. They define *what* should be achieved but deliberately avoid prescribing *how* — because every plant, every process, and every organization is different. This is a feature, not a bug. Standards need to be broad enough to apply to a refinery and a cookie factory.

The problem: practitioners don't work in abstract, universally applicable environments. They work in specific plants with specific constraints, and they need concrete answers. "The recipe management architecture should be appropriate to the application" is technically correct and practically useless at 2 PM on a Tuesday when you need to decide where to put the recipe validation logic.

ILA exists to fill that gap. It takes the standards seriously — and then makes the decisions that the standards intentionally leave open.

This document maps each key standard to the specific, opinionated positions ILA takes.

---

## ISA 5.1 — Instrumentation Symbols and Identification

**What the standard says:**

ISA 5.1 provides a system for identifying instruments and their functions. It defines letter codes for measured variables (T = Temperature, P = Pressure, F = Flow) and functional identifiers (I = Indicator, C = Controller, T = Transmitter). It allows for user-defined extensions and does not mandate a specific hierarchical structure beyond the tag number itself.

**What ILA decides:**

ILA enforces a specific hierarchical naming pattern: `{Area}_{Unit}_{DeviceType}{Seq}_{Attribute}`. This goes beyond ISA 5.1's scope — the standard does not require an Area-Unit hierarchy in the tag name itself. ILA does, because the tag name must serve as a physical address that is readable by a maintenance technician, traceable from the PLC through the historian, and consistent across all five layers.

ILA also decides that tag names are in English — regardless of the plant's location. ISA 5.1 is silent on language. ILA takes the position that tags are a technical coordinate system, not prose, and a universal language prevents confusion in multi-national teams, vendor support interactions, and cross-site standardization.

ILA further decides (Rule 1) that tag naming *is* architecture — meaning that a poorly named tag is an architectural defect, not a cosmetic issue. ISA 5.1 treats naming as documentation. ILA treats it as a structural decision with consequences in every layer.

---

## PackML / ISA-TR88.00.02 — Machine State Model

**What the standard says:**

ISA-TR88.00.02 defines a 17-state model for packaging machines. It specifies the state names, the allowed transitions, and the general purpose of each state. It defines three modes (Automatic, Semi-Automatic, Manual). It does not prescribe which states are mandatory vs. optional, nor does it specify what logic should execute in each state — that is left to the implementer.

**What ILA decides:**

ILA divides the 17 states into 9 core states (Idle, Starting, Execute, Stopping, Stopped, Aborting, Aborted, Clearing, Resetting) and 8 optional states (Hold/Unhold, Suspend/Unsuspend, Completing/Complete). The standard makes no such distinction. ILA does, because implementing all 17 states on a simple pick-and-place machine adds testing surface, HMI complexity, and maintenance burden without process benefit.

ILA decides that recipe validation happens in the Starting state — the PLC confirms that all recipe parameters are valid before transitioning to Execute. The standard does not prescribe this; it merely defines Starting as "the machine is preparing to execute." ILA fills in the specifics: Starting is where the Control Layer exercises Rule 5 by validating input from the Supervisory Layer.

ILA also extends PackML beyond packaging. The standard was written for packaging machines, but ILA applies the state model to any machine — inspection cells, batch reactors, assembly stations. The state model works because the lifecycle (idle → prepare → run → stop → fault) is universal.

---

## ISA-88 / IEC 61512 — Batch Control

**What the standard says:**

ISA-88 defines three models: the Physical Model (equipment hierarchy), the Procedural Model (recipe execution hierarchy), and the Process Model (process stages). It defines the separation between master recipes, control recipes, and equipment recipes. It specifies the concept of phases as the smallest unit of procedural control. It does not prescribe where recipe validation occurs, how recipes are transmitted to controllers, or which system has final authority over recipe acceptance.

**What ILA decides:**

ILA decides that the PLC (Layer 2) is the final authority on recipe acceptance. The Supervisory Layer (Layer 3) presents recipes and accepts operator input, but it cannot force a recipe onto the Control Layer. If the PLC rejects a recipe (missing parameters, out-of-range values, wrong equipment mode), that rejection is final until the operator corrects the issue. This is Rule 5 enforcement: the Control Layer dictates structure.

ILA also decides the relationship between ISA-88 and PackML: they are complementary, not competing. PackML manages the machine state lifecycle; ISA-88 manages the recipe execution *within* the Execute state. ISA-88 is silent on machine state management; PackML is silent on recipe execution. ILA connects them explicitly.

ILA maps the ISA-88 Physical Model to ILA tag naming. The Physical Model hierarchy (Area → Process Cell → Unit → Equipment Module → Control Module) directly informs the `{Area}_{Unit}_{DeviceType}` structure. The standard does not make this connection to tag naming — ILA does.

---

## ISA-101 — Human Machine Interfaces

**What the standard says:**

ISA-101 establishes a framework for managing HMIs throughout their lifecycle: design, implementation, operation, and maintenance. It covers design philosophy, style guides, alarm integration, and user interface standards. It recommends a structured design process but does not mandate a specific visual style — it explicitly states that the style guide is site-specific.

**What ILA decides:**

ILA recommends high-performance HMI principles as the default. This means grayscale backgrounds, color reserved for abnormal states, analog indicators alongside numeric values, and minimal decorative elements. ISA-101 allows this as one valid approach among many. ILA makes it the starting point, with documented deviations where operator teams have strong, justified preferences for alternative styles.

ILA decides (Rule 2) that HMI screens contain zero process logic. ISA-101 is primarily a design standard and does not explicitly address the boundary between display logic and control logic. ILA draws a hard line: if removing all SCADA scripting would change the machine's behavior, the architecture is wrong. ISA-101 concerns itself with how information is *presented*. ILA concerns itself with ensuring that nothing beyond presentation exists at Layer 3.

ILA decides that the HMI screen hierarchy mirrors the PLC program structure (Rule 5). ISA-101 recommends a logical screen hierarchy but does not tie it to the control system structure. ILA does — because a screen that has no corresponding PLC program is either orphaned or hiding logic that belongs elsewhere.

---

## IEC 62443 — Industrial Cybersecurity

**What the standard says:**

IEC 62443 is a comprehensive, multi-part standard covering organizational security (policies, procedures, training), system security (zones, conduits, security levels), and component security (device hardening, secure development). It defines the concept of zones (groups of assets with the same security level) and conduits (controlled communication paths between zones). It defines four Security Levels (SL 1–4). It requires a risk assessment to determine target security levels but does not prescribe specific network topologies or firewall rules.

**What ILA decides:**

ILA maps each of its five layers to an IEC 62443 zone, with Layer 1 and Layer 2 sharing a zone (because they require real-time, low-latency communication). This is a specific architectural decision — IEC 62443 does not dictate zone boundaries; it provides a methodology for determining them through risk assessment. ILA provides a default zone model that works for most industrial environments, with the expectation that high-risk sites may refine the boundaries through formal risk assessment.

ILA decides that firewall rules physically enforce Rule 4 (data flows up, commands flow down). Specifically: Layer 4 (Data) is blocked from writing to Layer 2 (Control) at the network level. IEC 62443 would arrive at a similar conclusion through risk assessment, but ILA makes it an upfront architectural rule — not a finding from an assessment that may or may not be performed.

ILA decides (Rule 3) that the OT team owns and operates the infrastructure. IEC 62443-2-1 addresses organizational responsibilities but does not prescribe whether IT or OT owns the OT network. ILA takes the position that OT owns it — because the people who understand the physical consequences of network decisions must be the people who make those decisions.

ILA also decides that the OT team has final authority over incident response decisions that affect production systems. IEC 62443 requires an incident response capability but does not resolve the IT/OT authority boundary. ILA does: IT advises, OT decides, because only OT understands whether shutting down a system mid-process creates a safety hazard.

---

## ISA-95 / IEC 62264 — Enterprise-Control Integration

**What the standard says:**

ISA-95 defines hierarchical levels for enterprise-to-control integration (the Purdue Model: Levels 0–5). It defines activity models for production, quality, maintenance, and inventory operations. It provides the B2MML (Business to Manufacturing Markup Language) data schema for structured data exchange between manufacturing operations and business systems. It defines the boundary between enterprise systems and manufacturing operations at Level 3/4.

**What ILA decides:**

ILA maps its five layers to Purdue levels but is not a 1:1 copy. ILA separates the Data Layer (Layer 4) from the Supervisory Layer (Layer 3), which ISA-95 groups together at Level 3. This separation matters because the data infrastructure has fundamentally different security requirements, operational patterns, and ownership than the SCADA systems. ISA-95 treats Level 3 as a single tier; ILA splits it to enforce clearer boundaries.

ILA decides that MES integration happens at Layer 3 or Layer 4 — never directly to the PLC. ISA-95 defines the data exchange models but does not prohibit direct Level 4-to-Level 2 communication if the risk assessment allows it. ILA prohibits it as an architectural rule, because direct business-system-to-PLC connections create unaudited pathways that bypass operator awareness and Layer 3 validation.

ILA decides that the local historian is always the source of truth, even when data is replicated to cloud platforms. ISA-95 does not address cloud architecture (it predates widespread cloud adoption in manufacturing). ILA fills this gap: cloud is a replica consumer, not an authority. If the cloud says one thing and the local historian says another, the local historian wins.

---

## OPC UA / IEC 62541 — Data Exchange

**What the standard says:**

OPC UA defines a platform-independent, service-oriented architecture for secure, reliable data exchange. It supports information modeling (address space with nodes, references, and data types), security (certificate-based authentication, encryption), subscriptions (efficient data change notification), and historical data access. It does not prescribe how the address space should be organized — that is application-specific.

**What ILA decides:**

ILA decides that the OPC UA address space mirrors the PLC program structure and the ILA tag naming hierarchy. The standard allows any node structure; ILA prescribes one: `Root/{Area}/{Unit}/{Attribute}`. This means the OPC UA server is navigable by anyone who understands the ILA naming convention — no separate documentation needed to find a tag.

ILA decides that the PLC is the OPC UA server, and Layers 3 and 4 are clients. The standard supports any topology (server, client, pub/sub, aggregation). ILA constrains it: the source of truth is the PLC, and it serves data upward. Aggregation servers are acceptable at Layer 4, but they aggregate — they do not modify or create new data that is written back down.

ILA decides that OPC UA write access is restricted to Layer 3 (Supervisory) for operator commands only. Layer 4 (Data) has read-only access. The standard's security model supports this through user roles and access control, but it does not mandate it. ILA does — as a direct enforcement of Rule 4.

---

## Summary: Where ILA Adds Value

| Standard | Leaves Open | ILA Decides |
|----------|-------------|-------------|
| ISA 5.1 | Tag structure and hierarchy | `{Area}_{Unit}_{Device}{Seq}_{Attr}`, English, hierarchical |
| PackML | Which states to implement, what happens in each | 9 core + 8 optional, recipe validation in Starting |
| ISA-88 | Where recipe validation occurs, relationship to state model | PLC validates (Rule 5), ISA-88 runs inside PackML Execute |
| ISA-101 | Visual style, logic boundary | High-performance default, zero logic in SCADA (Rule 2) |
| IEC 62443 | Zone boundaries, IT/OT ownership | 1:1 layer-to-zone mapping, OT owns the stack (Rule 3) |
| ISA-95 | Cloud architecture, direct PLC access | Local historian is truth, no direct MES-to-PLC (Rule 4) |
| OPC UA | Address space structure, access control | Mirrors PLC structure, Layer 4 is read-only (Rule 4) |

The pattern is consistent: **standards define the playing field, ILA makes the calls.** ILA's five rules are the decisions that practitioners would otherwise make inconsistently, implicitly, or not at all — and the cost of not making them is an architecture that nobody can explain, debug, or maintain.

---

*Back to [ILA Overview](01-overview.md)*
