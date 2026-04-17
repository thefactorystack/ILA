# ILA Layer 3 — Supervisory

**SCADA, HMI, MES integration points — visualization and operator interaction, not logic.**

---

## Purpose

The Supervisory Layer is the operator's window into the process. It displays state, accepts commands, manages alarms, and provides the interface between human decisions and machine execution. It is governed most directly by **ILA Rule 2: No programming in SCADA.**

This is the layer where the temptation to add "just a little logic" is greatest — and where the cost of doing so is highest.

## What Belongs in Layer 3

- HMI / SCADA runtime screens
- Alarm management and notification
- Operator command inputs (start, stop, reset, recipe selection)
- Recipe entry and parameter display
- Production dashboards and status overviews
- MES integration points (work order receipt, production reporting)
- Audit trails and electronic signatures (where required)

## What Does NOT Belong in Layer 3

- Process logic, conditional branching, or calculations that determine machine behavior
- Timer-based sequencing or interlock logic
- Direct field device communication bypassing the Control Layer
- Data aggregation, trending calculations, or report generation (that is Layer 4)

## ILA Rule 2 — No Programming in SCADA

This rule is the defining principle of Layer 3. It exists because logic hidden in SCADA scripts is:

- **Invisible** — PLC programmers don't see it during code reviews
- **Fragile** — SCADA runtime updates can break or change script behavior silently
- **Unmaintainable** — the next engineer doesn't know to look in SCADA for logic
- **Untestable** — SCADA scripts rarely have structured testing or simulation

**The test is simple:** If you removed all SCADA scripting and replaced the HMI with a completely different platform, would the machine still function correctly? If the answer is no, you have logic in the wrong layer.

**What is allowed in SCADA:**

- Display formatting (unit conversion for display purposes, color changes based on state)
- Navigation logic (screen changes, popup triggers)
- User input validation at the UI level (numeric range checks before sending to PLC)
- Alarm acknowledgment and shelving

**What is NOT allowed in SCADA:**

- Calculating setpoints or derived values used by the process
- Conditional logic that determines equipment state or transitions
- Timer-based operations that affect production
- Writing directly to field devices (bypassing the PLC)
- Storing process-critical data only in SCADA (no PLC variable backing it)

## Key Standards

### ISA-101 — HMI Design

ISA-101 (Human Machine Interfaces for Process Automation Systems) provides guidelines for designing effective, safe, and consistent operator interfaces.

**Core principles relevant to ILA:**

**High-performance HMI design:**
- Use grayscale backgrounds — color is reserved for abnormal conditions and alarms
- Avoid decorative 3D graphics (realistic tank drawings, photographic backgrounds)
- Display process values with context: current value, setpoint, operating range, alarm limits
- Use analog indicators (bar graphs, trend sparklines) rather than just numeric displays
- Navigation should follow the physical plant hierarchy (which aligns with Layer 2 structure — Rule 5)

**A note on high-performance HMI:** ILA recommends high-performance principles as the default starting point. They are well-supported by research and reduce operator error in alarm-heavy environments. However, operator teams in some facilities have strong preferences shaped by years of experience with specific visual styles. If you deviate from high-performance principles, document the rationale and ensure the deviation is a conscious team decision — not an accident of legacy screen design.

**Screen hierarchy (aligned with Rule 5):**

```
Level 1: Plant Overview     — All areas, key KPIs, alarm summary
Level 2: Area Overview      — All units in an area, state and status
Level 3: Unit Detail        — Single unit, all instruments and controls
Level 4: Device/Loop Detail — Single control loop, tuning, diagnostics
```

**ILA principle:** This screen hierarchy maps directly to the PLC program structure defined in Layer 2. If the PLC has `PRG_Station01`, `PRG_Station02`, and `PRG_Station03`, Layer 3 has corresponding detail screens for each station. No orphan screens. No hidden logic screens.

### PackML HMI State Visualization

PackML provides a standardized way to display machine state on HMI screens. Every ILA-compliant machine should have a PackML faceplate showing:

- Current state (text and/or color-coded indicator)
- Available commands (only commands valid for the current state are active)
- Mode (Automatic / Semi-Automatic / Manual)
- Key counters (produced, rejected, cycles)

**Command button behavior:**

| Current State | Available Commands |
|---------------|-------------------|
| Idle | Start |
| Execute | Hold, Suspend, Stop |
| Held | Unhold, Stop |
| Suspended | Unsuspend, Stop |
| Complete | Reset |
| Stopped | Reset |
| Aborted | Clear |

**ILA principle:** The HMI reads `StateCurrent` from the PLC via OPC UA and enables/disables command buttons accordingly. The button logic is a simple state-to-button map — not a script that evaluates process conditions.

### ISA-88 — Recipe Management at the Supervisory Layer

In ISA-88 batch systems, the Supervisory Layer is responsible for:

- Presenting available recipes to the operator
- Accepting recipe parameter input
- Sending recipe parameters to the Control Layer
- Displaying batch progress and phase status
- Collecting batch records for Layer 4

**ILA principle (Rule 5 enforcement):** The operator selects and parameterizes a recipe at Layer 3. The recipe is sent to Layer 2, where the PLC validates it before accepting. If the PLC rejects the recipe (missing parameters, out-of-range values), Layer 3 displays the rejection reason. The SCADA never decides if a recipe is valid.

**Workflow:**

```
Operator selects recipe          → Layer 3
Layer 3 sends recipe to PLC     → Layer 2 (via OPC UA)
PLC validates recipe             → Layer 2 (Rule 5)
PLC accepts/rejects              → Layer 2
Layer 3 displays result          → Layer 3
PLC enters Starting state        → Layer 2 (only if accepted)
```

## Alarm Management

Alarm management is a Layer 3 responsibility, but alarm *detection* happens in Layer 2.

**ILA alarm flow:**

1. PLC detects alarm condition (Layer 2)
2. PLC sets alarm tag to TRUE with priority and description
3. SCADA reads alarm tag via OPC UA (Layer 3)
4. SCADA presents alarm to operator with ISA-18.2 attributes
5. Operator acknowledges alarm at Layer 3
6. Alarm data is logged in historian (Layer 4)

**ISA-18.2 alarm attributes (managed at Layer 3):**

- Priority (critical, high, medium, low, diagnostic)
- State (active/cleared, acknowledged/unacknowledged)
- Shelving (temporary suppression with automatic return)
- Alarm rationalization documentation

**ILA principle:** Alarm *conditions* are defined in PLC logic (Layer 2). Alarm *presentation*, priority classification, and operator response workflows are managed at Layer 3. Alarm *history* and trending are stored in Layer 4.

## MES Integration

Manufacturing Execution Systems (MES) interface with ILA at Layer 3 as a gateway between shop floor operations and business systems.

**Typical MES data flows:**

| Direction | Data | Example |
|-----------|------|---------|
| MES → Layer 3 | Work orders, production schedules | "Produce 1000 units of Product A" |
| Layer 3 → MES | Production counts, quality data, batch records | "Batch BR001 complete, 998 units produced" |

**ILA principle:** MES integration happens at Layer 3 or Layer 4, never directly to the PLC. The SCADA acts as the translation layer between MES work orders and PLC recipe parameters.

**Work order rejection flow:** When the PLC rejects a work order (invalid product code, missing recipe, equipment not available), the rejection must flow back through Layer 3 to MES with a reason code. The MES must not retry indefinitely — the operator and the MES both need to see *why* the order was rejected so the root cause is resolved, not masked by retries.

**ISA-95 activity models:** ISA-95 defines standard activity models for production, quality, maintenance, and inventory operations. When integrating MES at Layer 3, use ISA-95 activity models to structure the data exchange — this makes MES integration vendor-independent and prevents building point-to-point interfaces that break when either system is upgraded.

## Regulated Environments

In regulated industries (pharmaceutical, food safety, medical devices), Layer 3 carries additional responsibilities defined by regulatory frameworks.

**FDA 21 CFR Part 11 — Electronic Records and Signatures:**

When your SCADA system generates electronic records (batch records, quality logs, production reports) or accepts electronic signatures (operator confirmations, recipe approvals), it must comply with 21 CFR Part 11. Key requirements at Layer 3:

- **Audit trails:** Every operator action — recipe change, alarm acknowledgment, manual override — must be logged with timestamp, user identity, old value, new value, and reason for change. The audit trail must be tamper-evident and retained for the required period.
- **Electronic signatures:** When an operator signs off on a batch or approves a deviation, the signature must be linked to the specific record, include the signer's identity, and record the date/time and meaning of the signature (e.g., "reviewed," "approved," "released").
- **Access control:** Role-based access must prevent unauthorized changes. An operator who can run the process should not be able to modify recipes. A supervisor who can approve deviations should not be able to delete audit trail entries.
- **System validation:** The SCADA system must be validated (IQ/OQ/PQ) to demonstrate that it functions as intended and that electronic records are trustworthy.

**ILA principle:** 21 CFR Part 11 compliance is a Layer 3 responsibility — the SCADA platform provides the audit trails, signatures, and access control. The PLC (Layer 2) enforces process logic regardless of regulatory context. Do not implement compliance logic in the PLC; it does not belong there.

**Other regulatory frameworks:** GMP (Good Manufacturing Practice), IEC 62304 (medical device software), and EU Annex 11 (computerized systems in pharmaceutical) impose similar requirements. The pattern is the same: audit, authenticate, validate — and keep all of it at Layer 3, not buried in PLC logic.

## Practical Checklist

- [ ] Zero process logic exists in SCADA scripts (Rule 2 — verified by review)
- [ ] HMI screen hierarchy mirrors the PLC program structure (Rule 5)
- [ ] PackML faceplate shows current state, available commands, and mode
- [ ] Command buttons are state-dependent (not always-enabled)
- [ ] High-performance HMI principles are followed (grayscale, no decorative 3D)
- [ ] Recipe entry sends parameters to PLC — PLC validates and accepts/rejects (Rule 5)
- [ ] Alarm detection is in the PLC; alarm presentation and management is in SCADA
- [ ] All SCADA tags trace back to defined PLC variables (no orphan tags)
- [ ] If the SCADA platform were replaced, the machine would continue to function
- [ ] MES work order rejection flows back with reason codes (no silent retries)
- [ ] *Regulated:* Audit trails capture every operator action with timestamp, user, old/new values
- [ ] *Regulated:* Electronic signatures are linked to specific records with signer identity
- [ ] *Regulated:* Role-based access control prevents unauthorized changes
- [ ] *Regulated:* SCADA system is validated (IQ/OQ/PQ) if generating electronic records

---

*Back to [ILA Overview](01-overview.md) | Previous: [Layer 2 — Control](04-layer2-control.md) | Next: [Layer 4 — Data](06-layer4-data.md)*
