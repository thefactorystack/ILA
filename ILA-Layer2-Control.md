# ILA Layer 2 - Control

**PLCs, safety controllers, motion controllers, state machines, interlocks, recipes, and the executable structure of the machine.**

---

## Purpose

The Control Layer is where machine behavior is decided. It owns process logic, state transitions, interlocks, sequencing, timing, recipe validation, and the executable structure that the rest of the stack should follow.

This is the layer governed most directly by **ILA Rule 5: Structure is dictated by the Control Layer.**

## What Belongs Here

- PLC programs (main process logic)
- Safety PLC programs (safety logic on safety-rated hardware)
- Motion controllers and motion profiles
- PackML state machines
- ISA-88 batch control logic
- Recipe validation logic
- OPC UA server configuration (exposing tags to higher layers)
- Equipment modules and reusable function blocks
- All process interlocks, sequencing, permissives, and timing

## What Does Not Belong Here

- HMI/SCADA scripting (that is Layer 3)
- Database queries or historian writes (that is Layer 4)
- Network configuration (that is Layer 5)
- Business rules that belong in MES or ERP, unless they must be enforced for safe machine operation

## Control Layer Rules

These rules keep the PLC and related controllers readable, deterministic, and safe to operate for the life of the machine.

| Rule | Principle | Meaning |
|------|-----------|---------|
| R1 | **Safety is always the first priority** | Fault handling is not a feature; it is architecture. Safety logic belongs on safety-rated hardware and must not be mixed casually with standard process logic. |
| R2 | **Use standards** | Clear sequence, clear state, clear transition. Use PackML, ISA-88, IEC 61131-3, and site standards where they fit. Unnamed custom patterns become chaos over time. |
| R3 | **PLC code must be readable in 10 years** | Use named constants, clear variables, reusable function blocks, and explicit state transitions. No cryptic names, hidden quick fixes, or commissioning hacks left behind as permanent architecture. |
| R4 | **Recipes and parameters come from above; validation happens here** | Layer 3 or MES may provide recipe intent and parameters. The PLC validates them and executes safely. The PLC should not become the master recipe authoring system. |
| R5 | **If SCADA goes down, the machine still runs safely** | Design from day one so the machine can continue, stop, hold, or recover safely without relying on SCADA scripts or HMI availability. |

## Key Standards

### IEC 61131-3 - PLC Programming

IEC 61131-3 defines the five PLC programming languages: Structured Text (ST), Ladder Diagram (LD), Function Block Diagram (FBD), Sequential Function Chart (SFC), and Instruction List (IL, deprecated).

**ILA recommendation:** Use Structured Text (ST) as the primary language for new projects when the team can maintain it. It is version-control friendly and works well for state machines, data structures, and reusable libraries. Use Ladder Diagram where local standards, technician capability, or safety review practices require it. Maintainability by the plant team matters more than language fashion.

**Program organization:**

```
Project
├── Application
│   ├── TaskConfig
│   │   ├── MainTask (cyclic, e.g. 10 ms)
│   │   └── SlowTask (cyclic, e.g. 100 ms)
│   ├── POU (Program Organization Units)
│   │   ├── PRG_Main           - Main program entry point
│   │   ├── PRG_PackML         - Machine state model
│   │   ├── PRG_Station01      - Station 01 sequence
│   │   ├── PRG_Station02      - Station 02 sequence
│   │   ├── FB_Cylinder        - Reusable cylinder function block
│   │   ├── FB_Motor           - Reusable motor function block
│   │   └── FC_RecipeValidate  - Recipe validation function
│   └── GVL (Global Variable Lists)
│       ├── GVL_IO             — Physical IO mapping
│       ├── GVL_PackML         — PackML state and command variables
│       ├── GVL_HMI            — Variables exposed to Layer 3
│       └── GVL_Recipe         — Recipe data structures
```

**ILA principle:** The POU structure mirrors the physical machine. If the machine has three stations, the PLC has three station programs. The SCADA screen hierarchy, historian tag groups, and alarm groups all derive from this structure (Rule 5).

### PackML / ISA-TR88.00.02 - Machine State Model

PackML defines a standardized state model with 17 states that describe the lifecycle of a machine from idle to producing to faulted.

**Why PackML matters for ILA:** It provides a shared language for machine states. Whether you build packaging lines, inspection cells, assembly stations, or process units, the lifecycle is familiar: idle, start, execute, stop, fault, clear, reset. This lets Layer 3 use consistent state displays and lets Layer 4 compare state history across machines.

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

*Core states (default starting set):* Idle, Starting, Execute, Stopping, Stopped, Aborting, Aborted, Clearing, Resetting. These nine states cover startup, production, controlled shutdown, and fault handling.

*Optional states — Hold/Unhold:*  Add these when the machine has *internal* pause conditions — for example, a feeder that runs empty while the machine can resume automatically once refilled. Hold means "I paused myself."

*Optional states — Suspend/Unsuspend:*  Add these when the machine has *external* pause conditions — for example, a downstream conveyor that is full and the machine must wait. Suspend means "something outside me caused me to pause."

*Optional states — Completing/Complete:*  Add these when the machine processes finite batches or work orders and needs to distinguish between "still running" and "finished the current job, awaiting new work."

**Rule of thumb:** Start with the 9 core states. Run the machine in simulation. If you find yourself inventing ad-hoc pause/resume behavior, add the relevant PackML states. Do not add states preemptively; every state adds test cases, HMI behavior, alarm behavior, and operator training.

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

### ISA-88 / IEC 61512 - Batch Control

ISA-88 provides the framework for batch manufacturing with three core models:

**Physical Model** - describes the physical equipment hierarchy:

```
Enterprise -> Site -> Area -> Process Cell -> Unit -> Equipment Module -> Control Module
```

**Procedural Model** - describes the recipe execution hierarchy:

```
Procedure -> Unit Procedure -> Operation -> Phase
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

### PackML and ISA-88 - How They Work Together

A common source of confusion: PackML and ISA-88 are not alternatives. They solve different problems and run in parallel.

**PackML** manages *machine state* - is the unit idle, starting, executing, or faulted? It answers: "What is the machine doing right now?"

**ISA-88** manages *recipe execution* - which procedure is running, which phase is active, what parameters apply? It answers: "What is the machine making right now, and how far along is it?"

In a batch machine (e.g., a brewery mash tun), the PackML state machine governs the unit's lifecycle: Idle → Starting → Execute → Completing → Complete. *Within* the Execute state, ISA-88's procedural model runs the recipe: FillPhase → HeatPhase → HoldPhase → TransferPhase. The PackML state machine provides the container; ISA-88 fills it with process content.

**ILA principle:** Every machine should have a defined state model. PackML is the default state model unless there is a documented reason to use a site-specific equivalent. Machines that execute recipes or batch processes also implement ISA-88 procedural control inside the executing state.

**Modes:** PackML defines three modes (Automatic, Semi-Automatic, Manual). ISA-88 adds recipe modes (e.g., recipe-controlled vs. operator-controlled phases). Both coexist — the PackML mode governs the machine, while ISA-88 mode governs the procedure. Document both in the PLC and expose both to Layer 3.

### ISA 5.1 - Tag Naming at the Control Layer

While ISA 5.1 tag naming is introduced at Layer 1 (Field), the Control Layer is where tag names are defined in the PLC program and exposed to every layer above.

**ILA principle:** The PLC variable model is the master tag registry for machine behavior. Every HMI tag, alarm tag, historian tag, and recipe status that affects or describes the machine must trace back to a defined Control Layer variable.

**Naming discipline at Layer 2:**

| Variable | Layer 1 device | Layer 3 use | Layer 4 use |
|----------|---------------|-------------|-------------|
| `IC01_CAM01_TriggerReq` | Camera trigger signal | Shown on operator HMI | Logged in historian |
| `IC01_CAM01_InspResult` | Camera inspection result | Pass/Fail display | Quality trending |
| `BR01_TT01_Value` | Temperature transmitter | Live temp display | Batch record |

## OPC UA Server Configuration

The Control Layer exposes its tags to higher layers via OPC UA (or equivalent). This is the boundary between Layer 2 and Layers 3/4.

**ILA principle:** The PLC's OPC UA server defines exactly which tags are visible to Layer 3 and Layer 4. Not every internal PLC variable should be exposed. Publish only tags with a clear consumer, purpose, and access mode.

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

## Control-Layer Decisions

Make these decisions explicitly during design review:

| Decision | ILA default |
|----------|-------------|
| Machine state model | PackML core states first, optional states when needed |
| Recipe validation | PLC validates before execution |
| Process interlocks | PLC or safety PLC, not SCADA |
| HMI command handling | HMI writes intent; PLC validates and acts |
| Published data model | OPC UA structure mirrors PLC structure |
| Safety logic | Safety-rated hardware and reviewed safety design |
| Reusable device logic | Function blocks with consistent command, feedback, timeout, and fault behavior |

## Practical Checklist

- [ ] PLC program structure mirrors the physical machine (Rule 5)
- [ ] Machine state model is implemented and documented
- [ ] Recipe validation happens in the Control Layer, not SCADA (Rule 5)
- [ ] ISA 5.1 tag naming is enforced in all GVLs
- [ ] OPC UA server exposes only the tags consumed by Layer 3 and Layer 4
- [ ] Safety logic runs on safety-rated hardware, not standard PLC
- [ ] Function blocks (FB_Cylinder, FB_Motor, etc.) are reusable and version-controlled
- [ ] Batch control follows ISA-88 models where applicable
- [ ] Every PLC variable that appears in SCADA or historian has a defined, documented purpose
- [ ] Exceptions to PackML, naming, or exposure rules are documented with rationale

---

*Back to [ILA Overview](ILA-Overview.md) | Previous: [Layer 1 - Field](ILA-Layer1-Field.md) | Next: [Layer 3 - Supervisory](ILA-Layer3-Supervisory.md)*
