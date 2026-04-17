# ILA Getting Started — Your First Machine

**A step-by-step walkthrough of ILA applied to a single machine, from field device to network segment.**

---

## Who This Document Is For

You have read the [ILA Overview](01-overview.md). You understand the five layers and the five rules. Now you want to see what it looks like in practice — on a real machine, with real tags, real PLC code, and real network decisions.

This document walks through a single machine: an inspection cell called **IC01**. It has a camera, a cylinder, a reject chute, and a conveyor. Simple enough to understand in one sitting. Complex enough to exercise all five ILA layers.

## The Machine: Inspection Cell 01 (IC01)

**What it does:** Parts arrive on a conveyor. A sensor detects a part. A cylinder stops the part. A camera inspects the part. If the part passes, the cylinder retracts and the part continues. If the part fails, a diverter sends it to the reject bin.

**Physical equipment:**

- 1 conveyor motor (VFD-driven)
- 1 part-present sensor (photoelectric)
- 1 stopper cylinder (pneumatic, double-acting)
- 1 inspection camera (smart camera with Ethernet interface)
- 1 reject diverter cylinder (pneumatic)
- 1 reject bin full sensor
- 1 safety e-stop

## Layer 1 — Name Every Device

Before writing a single line of PLC code, name every field device using the ILA tag naming pattern: `{Area}_{Unit}_{DeviceType}{Seq}_{Attribute}`.

The area is `IC01` (Inspection Cell 01). Each device gets a unit or device code:

| Physical Device | Tag Base | Attributes |
|----------------|----------|------------|
| Conveyor motor | `IC01_CONV01` | `CmdRun`, `FbkRunning`, `SpeedRef`, `SpeedAct` |
| Part-present sensor | `IC01_PE01` | `Status` |
| Stopper cylinder | `IC01_CYL01` | `CmdExtend`, `CmdRetract`, `FbkExtended`, `FbkRetracted` |
| Inspection camera | `IC01_CAM01` | `TriggerReq`, `TriggerAck`, `InspResult`, `InspComplete` |
| Reject diverter | `IC01_CYL02` | `CmdExtend`, `CmdRetract`, `FbkExtended`, `FbkRetracted` |
| Reject bin sensor | `IC01_PE02` | `BinFull` |
| E-stop | `IC01_ES01` | `Status` |

**What you have now:** Every device has a name that tells you where it is (IC01), what it is (CYL01), and what the signal means (FbkExtended). A maintenance technician can read `IC01_CYL02_FbkRetracted` and walk directly to the reject diverter in Inspection Cell 01 to check the retracted position sensor. No drawings needed at 3 AM.

**Test yourself:** Can you look at any tag in the list above and know where to physically walk to find the device? If yes, your naming works. If no, revise until it does.

## Layer 2 — Structure the PLC Program

The PLC program structure mirrors the physical machine. IC01 is one machine, so it gets one application with a clear structure:

```
Project: TheFactoryStack_IC01
├── Application
│   ├── TaskConfig
│   │   └── MainTask (10 ms cycle)
│   ├── Programs
│   │   ├── PRG_Main            — Calls everything, handles mode selection
│   │   ├── PRG_PackML          — State machine (core 9 states to start)
│   │   └── PRG_InspSequence    — Inspection cycle sequence
│   ├── Function Blocks
│   │   ├── FB_Cylinder         — Reusable: cmd, fbk, timeout, fault
│   │   └── FB_Motor            — Reusable: cmd, fbk, speed, fault
│   └── GVL
│       ├── GVL_IO              — Physical IO mapping (ISA 5.1 names)
│       ├── GVL_PackML          — StateCurrent, CmdStart, CmdStop, etc.
│       ├── GVL_HMI             — Variables exposed to Layer 3
│       └── GVL_Recipe          — RecipeID, PartType, ToleranceLimits
```

**PackML state machine (start with core states):**

For this simple machine, implement the 9 core states. You do not need Hold/Suspend yet — add them later if the machine needs to pause for upstream/downstream conditions.

```iec-st
CASE StateCurrent OF
    STATE_IDLE:
        // Conveyor stopped, cylinder retracted, camera idle
        IF CmdStart AND RecipeValid THEN
            StateCurrent := STATE_STARTING;
        END_IF

    STATE_STARTING:
        // Validate recipe (Rule 5): does the recipe ID exist?
        // Does the camera have the correct inspection program loaded?
        IF bRecipeLoaded AND bCameraReady THEN
            StateCurrent := STATE_EXECUTE;
        ELSIF bStartTimeout THEN
            StartFaultReason := 'Recipe or camera not ready';
            StateCurrent := STATE_ABORTING;
        END_IF

    STATE_EXECUTE:
        // Call PRG_InspSequence — the actual inspection cycle
        PRG_InspSequence();
        IF CmdStop THEN
            StateCurrent := STATE_STOPPING;
        ELSIF bFault THEN
            StateCurrent := STATE_ABORTING;
        END_IF

    STATE_STOPPING:
        // Complete current inspection cycle, then stop conveyor
        IF bCycleDone AND NOT CONV01_FbkRunning THEN
            StateCurrent := STATE_STOPPED;
        END_IF

    STATE_STOPPED:
        IF CmdReset THEN
            StateCurrent := STATE_RESETTING;
        END_IF

    STATE_RESETTING:
        // Retract all cylinders, clear fault flags, reset counters
        IF bResetDone THEN
            StateCurrent := STATE_IDLE;
        END_IF

    STATE_ABORTING:
        // Immediate safe stop: stop conveyor, retract cylinders
        IF bAbortDone THEN
            StateCurrent := STATE_ABORTED;
        END_IF

    STATE_ABORTED:
        IF CmdClear THEN
            StateCurrent := STATE_CLEARING;
        END_IF

    STATE_CLEARING:
        // Clear fault registers, prepare for reset
        IF bClearDone THEN
            StateCurrent := STATE_STOPPED;
        END_IF
END_CASE
```

**Inspection sequence (called from Execute):**

```iec-st
CASE iSeqStep OF
    0: // Wait for part
        IF IC01_PE01_Status THEN
            iSeqStep := 10;
        END_IF

    10: // Stop the part
        IC01_CYL01_CmdExtend := TRUE;
        IF IC01_CYL01_FbkExtended THEN
            iSeqStep := 20;
        END_IF

    20: // Trigger camera
        IC01_CAM01_TriggerReq := TRUE;
        IF IC01_CAM01_InspComplete THEN
            IC01_CAM01_TriggerReq := FALSE;
            IF IC01_CAM01_InspResult THEN  // TRUE = pass
                iSeqStep := 30;
            ELSE
                iSeqStep := 40;
            END_IF
        END_IF

    30: // Pass — release part
        IC01_CYL01_CmdExtend := FALSE;
        IC01_CYL01_CmdRetract := TRUE;
        iPartsPassed := iPartsPassed + 1;
        IF IC01_CYL01_FbkRetracted THEN
            iSeqStep := 0;
        END_IF

    40: // Fail — divert to reject
        IC01_CYL02_CmdExtend := TRUE;  // Open diverter
        IC01_CYL01_CmdExtend := FALSE; // Release stopper
        iPartsRejected := iPartsRejected + 1;
        IF IC01_CYL01_FbkRetracted THEN
            iSeqStep := 50;
        END_IF

    50: // Reset diverter
        IC01_CYL02_CmdRetract := TRUE;
        IF IC01_CYL02_FbkRetracted THEN
            IC01_CYL02_CmdExtend := FALSE;
            IC01_CYL02_CmdRetract := FALSE;
            iSeqStep := 0;
        END_IF
END_CASE
```

**Rule 5 in action:** The recipe is validated in the Starting state. The camera program is confirmed loaded before production begins. The SCADA can send any recipe it wants — the PLC decides if it is valid.

**OPC UA exposure:** Define which tags Layer 3 and Layer 4 need:

```
Root/IC01/
├── PackML/
│   ├── StateCurrent          → Layer 3 (HMI state display)
│   ├── CmdStart              → Layer 3 (operator button)
│   ├── CmdStop               → Layer 3 (operator button)
│   ├── CmdReset              → Layer 3 (operator button)
│   └── CmdClear              → Layer 3 (operator button)
├── Production/
│   ├── PartsPassed           → Layer 3 + Layer 4
│   ├── PartsRejected         → Layer 3 + Layer 4
│   └── CycleTime             → Layer 4 (historian trending)
├── Recipe/
│   ├── ActiveRecipeID        → Layer 3 + Layer 4
│   ├── RecipeValid           → Layer 3 (display)
│   └── PartType              → Layer 4 (batch record)
└── Faults/
    ├── FaultActive           → Layer 3 (alarm)
    └── FaultCode             → Layer 3 + Layer 4
```

## Layer 3 — Build the Operator Interface

The HMI for IC01 follows the screen hierarchy dictated by the PLC structure (Rule 5):

**Screen 1: Area Overview** — Shows IC01 as a single machine tile with state indicator (color-coded), parts passed/rejected counters, and active alarm count. If you had multiple cells (IC01, IC02, IC03), each would be a tile on this screen.

**Screen 2: IC01 Detail** — The main operator screen. Shows the PackML faceplate (current state, available command buttons), live status of each device (CYL01 extended/retracted, camera last result), production counters, and active recipe.

**Screen 3: IC01 Recipe Entry** — Allows operator to select and parameterize a recipe (part type, tolerance limits, camera program). When the operator hits "Send to PLC," the recipe parameters are written to the PLC's recipe GVL via OPC UA. The screen then shows whether the PLC accepted or rejected the recipe — and if rejected, *why*.

**Rule 2 verification:** Remove all SCADA scripting from IC01's screens. Does the machine still run? If you replaced the entire SCADA platform with a different vendor's product, would IC01 still inspect parts? If yes, Rule 2 is satisfied.

**Alarm list for IC01:**

| Alarm Tag | Priority | Description |
|-----------|----------|-------------|
| `IC01_CYL01_Timeout` | High | Stopper cylinder did not reach position in time |
| `IC01_CYL02_Timeout` | High | Reject diverter did not reach position in time |
| `IC01_CAM01_Fault` | High | Camera communication fault |
| `IC01_PE02_BinFull` | Medium | Reject bin full — operator must empty |
| `IC01_ES01_Tripped` | Critical | E-stop activated |
| `IC01_PackML_StartFault` | Medium | Recipe validation failed during Starting |

Every alarm is *detected* in the PLC (Layer 2) and *presented* at Layer 3. The SCADA does not decide what constitutes a fault — it only displays and manages what the PLC reports.

## Layer 4 — Store and Trend the Data

**Historian configuration:**

Connect the historian to IC01's OPC UA server. Subscribe to the following tags with the exact same names as the PLC:

| Historian Tag | Source | Storage Rate | Purpose |
|---------------|--------|-------------|---------|
| `IC01_PackML_StateCurrent` | OPC UA | On change | OEE state tracking |
| `IC01_Production_PartsPassed` | OPC UA | On change | Production counting |
| `IC01_Production_PartsRejected` | OPC UA | On change | Quality tracking |
| `IC01_Production_CycleTime` | OPC UA | Every cycle | Performance trending |
| `IC01_Recipe_ActiveRecipeID` | OPC UA | On change | Batch record |
| `IC01_Faults_FaultCode` | OPC UA | On change | Downtime analysis |

**Rule 5 in action:** The historian folder structure is `IC01/PackML/`, `IC01/Production/`, `IC01/Recipe/`, `IC01/Faults/` — identical to the OPC UA node structure, which is identical to the PLC GVL structure. One name, one structure, from PLC to historian.

**Rule 4 in action:** The historian reads these tags. It never writes to them. No historian-driven setpoint changes. No dashboard-triggered commands.

**SQL batch record (one row per inspected part):**

```sql
CREATE TABLE IC01_InspectionLog (
    LogID           INT IDENTITY PRIMARY KEY,
    Timestamp       DATETIME DEFAULT GETDATE(),
    RecipeID        VARCHAR(50),
    PartType        VARCHAR(50),
    InspResult      BIT,            -- 1 = pass, 0 = fail
    CycleTimeMs     INT,
    BatchID         VARCHAR(50)
);
```

## Layer 5 — Give It a Network Home

**VLAN assignment:**

| Device | IP Address | VLAN | Layer |
|--------|-----------|------|-------|
| IC01 PLC | 10.1.10.101 | VLAN 10 (Control) | 2 |
| IC01 Camera | 10.1.10.201 | VLAN 10 (Control) | 1 |
| SCADA Server | 10.1.20.10 | VLAN 20 (Supervisory) | 3 |
| Historian | 10.1.30.10 | VLAN 30 (Data) | 4 |
| AD Server | 10.1.40.10 | VLAN 40 (Infrastructure) | 5 |
| OT Firewall | 10.1.40.1 | All VLANs | 5 |

**Firewall rules for IC01:**

| Source | Destination | Port | Protocol | Allow/Deny |
|--------|-------------|------|----------|------------|
| SCADA (10.1.20.10) | PLC (10.1.10.101) | 4840 | OPC UA | Allow |
| Historian (10.1.30.10) | PLC (10.1.10.101) | 4840 | OPC UA (read only) | Allow |
| PLC (10.1.10.101) | Camera (10.1.10.201) | 8080 | HTTP/API | Allow |
| Historian (10.1.30.10) | PLC (10.1.10.101) | Any | Write | **Deny** |
| Enterprise IT | PLC (10.1.10.101) | Any | Any | **Deny** |

**Rule 4 enforced by firewall:** Even if someone misconfigures the historian to write a tag to the PLC, the firewall blocks it. The architecture prevents the violation at the network level — not just the policy level.

## What You Have Now

After completing this walkthrough, IC01 is an ILA-compliant machine:

- **Layer 1:** Every device has an ISA 5.1-compliant tag name that doubles as a physical address
- **Layer 2:** PLC program structure mirrors the physical machine. PackML state machine manages lifecycle. Recipe validation happens in the PLC, not SCADA (Rule 5)
- **Layer 3:** HMI displays state, accepts commands, shows alarms — contains zero logic (Rule 2)
- **Layer 4:** Historian stores data with identical tag names and folder structure. Data flows up only (Rule 4)
- **Layer 5:** Network segmentation places each layer in its own VLAN. Firewall rules enforce data flow direction

**The one-machine test:** If your organization decides to adopt ILA, start with one machine. Apply the five rules. See if the next engineer who touches it can understand the structure without a walkthrough. If they can, ILA is working. Scale from there.

## Next Steps

- Add a second machine (IC02) and see how the naming, SCADA hierarchy, and historian structure scale
- Implement Hold/Suspend states when IC01 integrates with upstream/downstream equipment
- Add ISA-88 recipe management if IC01 needs to run multiple product variants with different inspection parameters
- Set up OEE calculation in the historian using PackML state data
- Conduct a Rule 2 audit: remove all SCADA scripting and verify the machine still runs

---

*Back to [ILA Overview](01-overview.md)*
