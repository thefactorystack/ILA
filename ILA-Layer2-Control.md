# ILA Layer 2 — Control

**PLCs, safety controllers, motion controllers — the layer that owns the process logic and dictates the structure of everything above it.**

---

## Purpose

The Control Layer is the heart of ILA. It is where process logic lives, where machine states are managed, and where the structure of the entire automation system is defined. Every other layer aligns to the decisions made here.

This is the layer governed most directly by **ILA Rule 5: Structure is dictated by the Control Layer.**

## What Belongs in Layer 2

- PLC programs (main process logic)
- Safety PLC programs (safety logic on safety-rated hardware)
- Motion controllers and motion profiles
- PackML state machines
- ISA-88 batch control logic
- Recipe validation logic
- OPC UA server configuration (exposing tags to higher layers)
- All process interlocks, sequencing, and timing

## What Does NOT Belong in Layer 2

- HMI/SCADA scripting (that is Layer 3)
- Database queries or historian writes (that is Layer 4)
- Network configuration (that is Layer 5)

## Key Standards

### IEC 61131-3 — PLC Programming

IEC 61131-3 defines the five PLC programming languages: Structured Text (ST), Ladder Diagram (LD), Function Block Diagram (FBD), Sequential Function Chart (SFC), and Instruction List (IL, deprecated).

**ILA recommendation:** Use Structured Text (ST) as the primary language for new projects. It is version-control friendly, readable, and maps well to modern software practices. Use Ladder Diagram where mandated by local standards or maintenance team capabilities.

**Program organization:**

```
Project
├── Application
│   ├── TaskConfig
│   │   ├── MainTask (cyclic, e.g. 10 ms)
│   │   └── SlowTask (cyclic, e.g. 100 ms)
│   ├── POU (Program Organization Units)
│   │   ├── PRG_Main           — Main program entry point
│   │   ├── PRG_PackML         — PackML state machine
│   │   ├── PRG_Station01      — Station 01 sequence
│   │   ├── PRG_Station02      — Station 02 sequence
│   │   ├── FB_Cylinder        — Reusable cylinder function block
│   │   ├── FB_Motor           — Reusable motor function block
│   │   └── FC_RecipeValidate  — Recipe validation function
│   └── GVL (Global Variable Lists)
│       ├── GVL_IO             — Physical IO mapping
│       ├── GVL_PackML         — PackML state and command variables
│       ├── GVL_HMI            — Variables exposed to Layer 3
│       └── GVL_Recipe         — Recipe data structures
```

**ILA principle:** The POU structure mirrors the physical machine. If the machine has three stations, the PLC has three station programs. The SCADA screen hierarchy, historian tag groups, and alarm groups all derive from this structure (Rule 5).

### PackML / ISA-TR88.00.02 — Machine State Model

PackML defines a standardized state model with 17 states that describe the lifecycle of a machine from idle to producing to faulted.

**Why PackML matters for ILA:** It provides a universal language for machine states. Whether you build packaging lines, inspection cells, or process plants, the state names and transitions are the same. This means Layer 3 (SCADA) can use consistent HMI faceplates, and Layer 4 (Data) can compare OEE across machines.

**The 17 PackML states:**

| State | Type | Description |
|-------|------|-------------|
| Idle | Acting | Machine is powered, ready for a command |
| Starting | Acting | Ramp-up sequence before production |
| Execute | Acting | Normal production operation |
| Completing | Acting | Finishing current work, no new work accepted |
| Complete | Wait | Work is done, awaiting reset |
| Resetting | Acting | Returning to Idle |
| Holding | Acting | Controlled stop (internal condition, e.g. upstream starved) |
| Held | Wait | Paused, awaiting un-hold |
| Unholding | Acting | Resuming from Held |
| Suspending | Acting | Controlled stop (external condition, e.g. downstream full) |
| Suspended | Wait | Paused by external condition |
| Unsuspending | Acting | Resuming from Suspended |
| Stopping | Acting | Controlled shutdown |
| Stopped | Wait | Machine stopped, requires reset to return to Idle |
| Aborting | Acting | Emergency or fault shutdown |
| Aborted | Wait | Machine aborted, requires clear to reach Stopped |
| Clearing | Acting | Clearing faults, moving to Stopped |

**Core vs. Optional States — Where to Start**

Not every machine needs all 17 states. ILA divides them into core states (implement always) and optional states (implement when your process requires them):

*Core states (always implement):*  Idle, Starting, Execute, Stopping, Stopped, Aborting, Aborted, Clearing, Resetting. These nine states cover the full lifecycle of any machine — startup, production, controlled shutdown, and fault handling.

*Optional states — Hold/Unhold:*  Add these when the machine has *internal* pause conditions — for example, a feeder that runs empty while the machine can resume automatically once refilled. Hold means "I paused myself."

*Optional states — Suspend/Unsuspend:*  Add these when the machine has *external* pause conditions — for example, a downstream conveyor that is full and the machine must wait. Suspend means "something outside me caused me to pause."

*Optional states — Completing/Complete:*  Add these when the machine processes finite batches or work orders and needs to distinguish between "still running" and "finished the current job, awaiting new work."

**Rule of thumb:** Start with the 9 core states. Run the machine in simulation. If you find yourself inventing ad-hoc workarounds for pause/resume behavior, that is your signal to add Hold or Suspend. Do not add states preemptively — each state adds testing surface and HMI complexity.

**Implementation pattern:**

```iec-st
CASE iState OF
    STATE_IDLE:
        IF bCmdStart AND bRecipeValid THEN   // Rule 5: validate before starting
            iState := STATE_STARTING;
        END_IF

    STATE_STARTING:
        // Ramp-up sequence
        // Validate recipe data (Rule 5 enforcement)
        IF bStartSequenceDone THEN
            iState := STATE_EXECUTE;
        END_IF

    STATE_EXECUTE:
        // Normal production
        IF bCmdStop THEN
            iState := STATE_STOPPING;
        ELSIF bFaultDetected THEN
            iState := STATE_ABORTING;
        END_IF

    // ... remaining states
END_CASE
```

**ILA Rule 5 enforcement:** Recipe validation happens in the `Starting` state. The Control Layer validates that the Supervisory Layer has provided a valid, complete recipe before the machine enters `Execute`. The SCADA never decides if a recipe is valid — it sends the recipe, and the PLC confirms or rejects it.

### ISA-88 / IEC 61512 — Batch Control

ISA-88 provides the framework for batch manufacturing with three core models:

**Physical Model** — describes the physical equipment hierarchy:

```
Enterprise → Site → Area → Process Cell → Unit → Equipment Module → Control Module
```

**Procedural Model** — describes the recipe execution hierarchy:

```
Procedure → Unit Procedure → Operation → Phase
```

**Process Model** — describes the process itself (stages and transitions).

**Where ISA-88 meets ILA:**

- The Physical Model maps directly to ILA tag naming at Layer 1 and Layer 2
- The Procedural Model is implemented as PLC logic in Layer 2
- Recipe parameters are entered at Layer 3 (Supervisory) and validated at Layer 2 (Control)
- Batch records and process data flow to Layer 4 (Data)

**Example: Brewery mashing process**

```
Physical:     BR01 → MASH01 → HLT01 (Hot Liquor Tank)
                              MLT01 (Mash/Lauter Tun)
                              BK01  (Boil Kettle)

Procedural:   MashProcedure
              ├── FillUnitProc
              │   ├── FillOp
              │   │   ├── OpenValvePhase
              │   │   └── MonitorLevelPhase
              ├── HeatUnitProc
              │   ├── HeatOp
              │   │   ├── RampTempPhase
              │   │   └── HoldTempPhase
              └── TransferUnitProc
```

### PackML and ISA-88 — How They Work Together

A common source of confusion: PackML and ISA-88 are not alternatives. They solve different problems and run in parallel.

**PackML** manages *machine state* — is the unit idle, starting, executing, or faulted? It answers: "What is the machine doing right now?"

**ISA-88** manages *recipe execution* — which procedure is running, which phase is active, what parameters apply? It answers: "What is the machine making right now, and how far along is it?"

In a batch machine (e.g., a brewery mash tun), the PackML state machine governs the unit's lifecycle: Idle → Starting → Execute → Completing → Complete. *Within* the Execute state, ISA-88's procedural model runs the recipe: FillPhase → HeatPhase → HoldPhase → TransferPhase. The PackML state machine provides the container; ISA-88 fills it with process content.

**ILA principle:** Every machine has a PackML state machine. Machines that execute recipes or batch processes *also* implement ISA-88 procedural control inside the Execute state. PackML is mandatory; ISA-88 is applied where the process demands it.

**Modes:** PackML defines three modes (Automatic, Semi-Automatic, Manual). ISA-88 adds recipe modes (e.g., recipe-controlled vs. operator-controlled phases). Both coexist — the PackML mode governs the machine, while ISA-88 mode governs the procedure. Document both in the PLC and expose both to Layer 3.

### ISA 5.1 — Tag Naming at the Control Layer

While ISA 5.1 tag naming is introduced at Layer 1 (Field), the Control Layer is where tag names are defined in the PLC program and exposed to every layer above.

**ILA principle:** The GVL (Global Variable List) in the PLC is the master tag registry. Every tag that appears on an HMI screen, in a historian, or in an alarm list must trace back to a defined variable in the Control Layer.

**Naming discipline at Layer 2:**

| Variable | Layer 1 device | Layer 3 use | Layer 4 use |
|----------|---------------|-------------|-------------|
| `IC01_CAM01_TriggerReq` | Camera trigger signal | Shown on operator HMI | Logged in historian |
| `IC01_CAM01_InspResult` | Camera inspection result | Pass/Fail display | Quality trending |
| `BR01_TT01_Value` | Temperature transmitter | Live temp display | Batch record |

## OPC UA Server Configuration

The Control Layer exposes its tags to higher layers via OPC UA (or equivalent). This is the boundary between Layer 2 and Layers 3/4.

**ILA principle:** The PLC's OPC UA server defines exactly which tags are visible to SCADA and historians. Not every internal PLC variable should be exposed — only those with a clear consumer in Layer 3 or Layer 4.

**Recommended OPC UA node structure:**

```
Root
├── IC01 (Inspection Cell 01)
│   ├── PackML
│   │   ├── StateCurrent
│   │   ├── CmdStart
│   │   ├── CmdStop
│   │   └── CmdReset
│   ├── CAM01
│   │   ├── TriggerReq
│   │   ├── InspResult
│   │   └── PartCount
│   └── Recipe
│       ├── ActiveRecipeID
│       ├── RecipeValid
│       └── BatchID
```

## Practical Checklist

- [ ] PLC program structure mirrors the physical machine (Rule 5)
- [ ] PackML state machine is implemented with all relevant states
- [ ] Recipe validation happens in the Control Layer, not SCADA (Rule 5)
- [ ] ISA 5.1 tag naming is enforced in all GVLs
- [ ] OPC UA server exposes only the tags consumed by Layer 3 and Layer 4
- [ ] Safety logic runs on safety-rated hardware, not standard PLC
- [ ] Function blocks (FB_Cylinder, FB_Motor, etc.) are reusable and version-controlled
- [ ] Batch control follows ISA-88 models where applicable
- [ ] Every PLC variable that appears in SCADA or historian has a defined, documented purpose

---

*Back to [ILA Overview](ILA-Overview.md) | Previous: [Layer 1 — Field](ILA-Layer1-Field.md) | Next: [Layer 3 — Supervisory](ILA-Layer3-Supervisory.md)*
