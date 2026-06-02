# ILA Standards Bridge

**What the standard says vs. what ILA decides.**

---

## Why This Document Exists

Industrial standards are written to be broadly applicable. They define terminology, models, requirements, and recommended practices, but they often avoid prescribing one implementation pattern because every plant, process, risk profile, and organization is different. That flexibility is necessary.

The problem is that practitioners do not work in abstract plants. They work in specific machines, lines, brownfield networks, vendor ecosystems, maintenance teams, and budget constraints. "The recipe management architecture should be appropriate to the application" may be correct, but it does not tell a project team where recipe validation belongs.

ILA fills that gap. It takes the standards seriously, then turns their open areas into practical defaults.

This document maps key standards to the decisions ILA makes. These are defaults, not a substitute for risk assessment, regulatory review, or site engineering judgment.

## Decision Map

| Standard | Standard focus | ILA decision |
|----------|----------------|--------------|
| ISA 5.1 | Instrument identification | Tag names are hierarchical coordinates across all layers |
| PackML / ISA-TR88.00.02 | Machine state model | Start with core states; add optional states when behavior requires them |
| ISA-88 / IEC 61512 | Batch and procedural control | PLC validates recipes; ISA-88 runs inside the machine state model where needed |
| ISA-101 | HMI lifecycle and design | Use high-performance HMI principles by default; keep process logic out of SCADA |
| ISA-18.2 | Alarm management | PLC detects alarm conditions; Layer 3 manages presentation and response |
| IEC 62443 | Industrial cybersecurity | Use ILA layers as a default zone model, refined by site risk assessment |
| ISA-95 / IEC 62264 | Enterprise-control integration | Keep MES/ERP from directly commanding PLCs |
| OPC UA / IEC 62541 | Secure data exchange | Mirror the PLC structure and restrict Layer 4 to read-only access |
| UNS | Shared industrial namespace pattern | Use ILA hierarchy for data discovery without creating command authority |

---

## ISA 5.1 — Instrumentation Symbols and Identification

**What the standard says:**

ISA 5.1 provides a system for identifying instruments and their functions. It defines letter codes for measured variables and functional identifiers, allows user-defined extensions, and supports consistent identification across engineering documents.

**What ILA decides:**

ILA enforces a specific hierarchical naming pattern: `{Area}_{Unit}_{DeviceType}{Seq}_{Attribute}`. This goes beyond ISA 5.1's scope — the standard does not require an Area-Unit hierarchy in the tag name itself. ILA does, because the tag name must serve as a physical address that is readable by a maintenance technician, traceable from the PLC through the historian, and consistent across all five layers.

ILA also decides that tag names use English technical terms regardless of plant location. Operator-facing text can and should be localized. Tags are a technical coordinate system, not prose.

ILA further decides (Rule 1) that tag naming *is* architecture — meaning that a poorly named tag is an architectural defect, not a cosmetic issue. ISA 5.1 treats naming as documentation. ILA treats it as a structural decision with consequences in every layer.

---

## PackML / ISA-TR88.00.02 — Machine State Model

**What the standard says:**

ISA-TR88.00.02 defines a 17-state model for packaging machines. It specifies state names, transitions, and modes. It provides the model, but implementation details still belong to the project.

**What ILA decides:**

ILA divides the 17 states into a practical starting set and optional states. Start with Idle, Starting, Execute, Stopping, Stopped, Aborting, Aborted, Clearing, and Resetting. Add Hold/Unhold, Suspend/Unsuspend, and Completing/Complete when the process behavior requires them.

ILA decides that recipe validation happens in the Starting state — the PLC confirms that all recipe parameters are valid before transitioning to Execute. The standard does not prescribe this; it merely defines Starting as "the machine is preparing to execute." ILA fills in the specifics: Starting is where the Control Layer exercises Rule 5 by validating input from the Supervisory Layer.

ILA applies the PackML state model beyond packaging when it fits: inspection cells, assembly stations, utility skids, and batch units. The lifecycle pattern is broadly useful even when the equipment is not a packaging machine.

---

## ISA-88 / IEC 61512 — Batch Control

**What the standard says:**

ISA-88 defines physical, procedural, and process models for batch control. It separates recipe types and defines phases as a core unit of procedural control. It gives a strong model, but a project still has to decide how that model is enforced in a specific automation stack.

**What ILA decides:**

ILA decides that the PLC (Layer 2) is the final authority on recipe acceptance. The Supervisory Layer (Layer 3) presents recipes and accepts operator input, but it cannot force a recipe onto the Control Layer. If the PLC rejects a recipe (missing parameters, out-of-range values, wrong equipment mode), that rejection is final until the operator corrects the issue. This is Rule 5 enforcement: the Control Layer dictates structure.

ILA also decides the relationship between ISA-88 and PackML: they are complementary, not competing. PackML manages the machine state lifecycle; ISA-88 manages the recipe execution *within* the Execute state. ISA-88 is silent on machine state management; PackML is silent on recipe execution. ILA connects them explicitly.

ILA maps the ISA-88 Physical Model to ILA naming. The physical hierarchy informs the `{Area}_{Unit}_{DeviceType}` structure so batch logic, PLC structure, HMI screens, and data records use the same mental model.

---

## ISA-101 — Human Machine Interfaces

**What the standard says:**

ISA-101 establishes a framework for managing HMIs throughout their lifecycle: design, implementation, operation, and maintenance. It covers design philosophy, style guides, alarm integration, and user interface standards. It recommends a structured design process but does not mandate a specific visual style — it explicitly states that the style guide is site-specific.

**What ILA decides:**

ILA recommends high-performance HMI principles as the default. This means grayscale backgrounds, color reserved for abnormal states, analog indicators alongside numeric values, and minimal decorative elements. ISA-101 allows this as one valid approach among many. ILA makes it the starting point, with documented deviations where operator teams have strong, justified preferences for alternative styles.

ILA decides (Rule 2) that HMI screens contain no process logic. ISA-101 focuses on effective HMI lifecycle and design. ILA adds an architectural boundary: if replacing the SCADA platform changes machine behavior, the behavior was in the wrong layer.

ILA decides that the HMI screen hierarchy mirrors the PLC program structure (Rule 5). ISA-101 recommends a logical screen hierarchy but does not tie it to the control system structure. ILA does — because a screen that has no corresponding PLC program is either orphaned or hiding logic that belongs elsewhere.

---

## IEC 62443 — Industrial Cybersecurity

**What the standard says:**

IEC 62443 is a comprehensive, multi-part standard covering organizational security (policies, procedures, training), system security (zones, conduits, security levels), and component security (device hardening, secure development). It defines the concept of zones (groups of assets with the same security level) and conduits (controlled communication paths between zones). It defines four Security Levels (SL 1–4). It requires a risk assessment to determine target security levels but does not prescribe specific network topologies or firewall rules.

**What ILA decides:**

ILA maps its layers to a default IEC 62443 zone model, with Layer 1 and Layer 2 often sharing a control zone because of real-time communication requirements. IEC 62443 does not dictate those exact boundaries; it provides a method for defining them through risk assessment. ILA gives teams a starting point.

ILA decides that firewall rules physically enforce Rule 4 (data flows up, commands flow down). Specifically: Layer 4 (Data) is blocked from writing to Layer 2 (Control) at the network level. IEC 62443 would arrive at a similar conclusion through risk assessment, but ILA makes it an upfront architectural rule — not a finding from an assessment that may or may not be performed.

ILA decides (Rule 3) that OT owns the production-impacting stack. IT and cybersecurity are required partners, but OT must have authority over decisions that can affect physical process state, safety, or recovery.

ILA also decides that the OT team has final authority over incident response decisions that affect production systems. IEC 62443 requires an incident response capability but does not resolve the IT/OT authority boundary. ILA does: IT advises, OT decides, because only OT understands whether shutting down a system mid-process creates a safety hazard.

---

## ISA-95 / IEC 62264 — Enterprise-Control Integration

**What the standard says:**

ISA-95 defines enterprise-control integration levels, activity models for manufacturing operations, and data structures such as B2MML for business-to-manufacturing exchange.

**What ILA decides:**

ILA maps its five layers to Purdue levels but is not a 1:1 copy. ILA separates the Data Layer (Layer 4) from the Supervisory Layer (Layer 3), which ISA-95 groups together at Level 3. This separation matters because the data infrastructure has fundamentally different security requirements, operational patterns, and ownership than the SCADA systems. ISA-95 treats Level 3 as a single tier; ILA splits it to enforce clearer boundaries.

ILA decides that MES integration happens through Layer 3 or Layer 4, not directly to the PLC. Direct business-system-to-PLC connections create unaudited pathways that bypass operator awareness and Control Layer validation.

ILA decides that the local historian is the operational source of truth, even when data is replicated to cloud platforms. Cloud systems can aggregate and analyze; they do not become the authority for local process decisions.

---

## OPC UA / IEC 62541 — Data Exchange

**What the standard says:**

OPC UA defines a platform-independent, service-oriented architecture for secure, reliable data exchange. It supports information modeling (address space with nodes, references, and data types), security (certificate-based authentication, encryption), subscriptions (efficient data change notification), and historical data access. It does not prescribe how the address space should be organized — that is application-specific.

**What ILA decides:**

ILA decides that the OPC UA address space mirrors the PLC program structure and ILA naming hierarchy. This makes the server navigable by anyone who understands the machine structure.

ILA treats the PLC or approved control gateway as the source of the authoritative process model. Aggregation servers are acceptable in Layer 4, but they aggregate and publish; they do not create control authority.

ILA restricts OPC UA write access to approved Layer 3 command paths. Layer 4 has read-only access. OPC UA supports this through roles and access control; ILA makes it an architectural default.

---

## UNS — Unified Namespace

**What the pattern says:**

A Unified Namespace (UNS) organizes industrial data into a shared, discoverable namespace. It is commonly implemented with MQTT, Sparkplug B, brokers, historians, and data platforms, but UNS itself is an architectural pattern rather than a formal ISA or IEC standard.

**What ILA decides:**

ILA treats UNS as a Layer 4 pattern for organizing and publishing operational data. The namespace should follow the same structure as the machine and site hierarchy: site, area, unit, asset, signal, and context.

ILA decides that a UNS must preserve source traceability. A consumer should be able to trace a UNS value back to the PLC tag, historian tag, device, unit, timestamp, engineering unit, and quality/status. A nice topic tree that hides the source is not a good namespace.

ILA also decides that UNS is not a command bus. Subscribers may consume data for dashboards, analytics, MES, quality, and maintenance workflows. Commands still flow through approved Layer 3 paths and are validated in Layer 2.

---

## Summary: Where ILA Adds Value

| Standard | Leaves Open | ILA Decides |
|----------|-------------|-------------|
| ISA 5.1 | Tag structure and hierarchy | `{Area}_{Unit}_{Device}{Seq}_{Attr}`, English, hierarchical |
| PackML | Which states to implement, what happens in each | 9 core + 8 optional, recipe validation in Starting |
| ISA-88 | Where recipe validation occurs, relationship to state model | PLC validates (Rule 5), ISA-88 runs inside PackML Execute |
| ISA-101 | Visual style, logic boundary | High-performance default, no process logic in SCADA (Rule 2) |
| ISA-18.2 | Alarm lifecycle and rationalization | PLC detects conditions; Layer 3 manages presentation and response |
| IEC 62443 | Zone boundaries, IT/OT ownership | 1:1 layer-to-zone mapping, OT owns the stack (Rule 3) |
| ISA-95 | Cloud architecture, direct PLC access | Local historian is truth, no direct MES-to-PLC (Rule 4) |
| OPC UA | Address space structure, access control | Mirrors PLC structure, Layer 4 is read-only (Rule 4) |
| UNS | Namespace structure, source traceability, command boundary | ILA hierarchy structures the namespace; UNS is read/subscribe, not command authority |

The pattern is consistent: **standards define the playing field, ILA makes the practical calls.** ILA's five rules are the decisions practitioners otherwise make inconsistently, implicitly, or not at all.

When a site needs to deviate, document the exception. Good architecture is not the absence of exceptions; it is the absence of invisible exceptions.

---

*Back to [ILA Overview](ILA-Overview.md)*
