# ILA Getting Started - Your First Machine.

**A practical walkthrough of ILA applied to one inspection cell, from field device naming to firewall rules.**

---

## Who This Document Is For

Start here if you understand the five ILA layers but want to see what they look like on a real machine.

The example is intentionally small: one inspection cell called **IC01**. It has a conveyor, a stopper cylinder, a camera, a reject diverter, sensors, and an e-stop. That is enough to exercise all five layers without turning the document into a full project manual.

This is not the only valid implementation. It is a reference pattern: copy the structure, then adapt the details to your PLC platform, SCADA system, network standards, and safety requirements.

## The Machine: Inspection Cell 01

Parts arrive on a conveyor. A photoelectric sensor detects a part. A stopper cylinder holds the part in place. A camera inspects it. Passing parts continue downstream. Failed parts are diverted to a reject bin.

**Physical equipment:**

| Quantity | Device | Notes |
|----------|--------|-------|
| 1 | Conveyor motor | VFD-driven |
| 1 | Part-present sensor | Photoelectric |
| 1 | Stopper cylinder | Pneumatic, double-acting |
| 1 | Inspection camera | Smart camera, Ethernet interface |
| 1 | Reject diverter cylinder | Pneumatic |
| 1 | Reject bin full sensor | Photoelectric or limit switch |
| 1 | E-stop | Safety-rated input |

## Layer 1 - Name Every Device

Before writing PLC code, name the field devices. The goal is not pretty tags. The goal is traceability.

**Pattern:** `{Area}_{Unit}_{DeviceType}{Seq}_{Attribute}`

For this example, `IC01` is the area/unit identifier for Inspection Cell 01.

| Physical device | Tag base | Example attributes |
|-----------------|----------|--------------------|
| Conveyor motor | `IC01_CONV01` | `CmdRun`, `FbkRunning`, `SpeedRef`, `SpeedAct`, `Fault` |
| Part-present sensor | `IC01_PE01` | `Status` |
| Stopper cylinder | `IC01_CYL01` | `CmdExtend`, `CmdRetract`, `FbkExtended`, `FbkRetracted`, `Timeout` |
| Inspection camera | `IC01_CAM01` | `ProgramID`, `ProgramLoaded`, `TriggerReq`, `TriggerAck`, `InspResult`, `InspComplete`, `Fault` |
| Reject diverter | `IC01_CYL02` | `CmdExtend`, `CmdRetract`, `FbkExtended`, `FbkRetracted`, `Timeout` |
| Reject bin sensor | `IC01_PE02` | `BinFull` |
| E-stop | `IC01_ES01` | `Status`, `Tripped` |

**Field-layer test:** Can a maintenance technician read `IC01_CYL02_FbkRetracted` and know where to walk and what to inspect? If not, the tag name is not doing its job.

## Layer 2 - Structure the PLC Program

The PLC structure mirrors the machine structure. For IC01, keep the program small but explicit:

```text
Project: TheFactoryStack_IC01
├── Application
│   ├── TaskConfig
│   │   └── MainTask (10 ms cycle)
│   ├── Programs
│   │   ├── PRG_Main             - Calls machine logic and handles mode selection
│   │   ├── PRG_PackML           - Machine state model
│   │   └── PRG_InspSequence     - Inspection cycle sequence
│   ├── Function Blocks
│   │   ├── FB_Cylinder          - Command, feedback, timeout, fault handling
│   │   ├── FB_Motor             - Command, feedback, speed, fault handling
│   │   └── FB_CameraHandshake   - Program select, trigger, complete, fault, timeout
│   └── GVL
│       ├── GVL_IO               - Physical I/O mapping
│       ├── GVL_PackML           - State, mode, commands, status
│       ├── GVL_HMI              - Variables exposed to Layer 3
│       ├── GVL_Recipe           - Recipe and inspection parameters
│       └── GVL_Faults           - Fault bits, codes, and descriptions
```

### PackML State Model

Start with the nine core states: `Idle`, `Starting`, `Execute`, `Stopping`, `Stopped`, `Aborting`, `Aborted`, `Clearing`, and `Resetting`.

Add `Hold` or `Suspend` only when the process needs controlled pause/resume behavior. Do not implement all 17 states just to look complete; every state adds HMI, test, and maintenance surface.

```iec-st
CASE StateCurrent OF
    STATE_IDLE:
        IF CmdStart AND RecipeValid THEN
            StateCurrent := STATE_STARTING;
        END_IF

    STATE_STARTING:
        // Rule 5: the PLC validates that the recipe and camera program are usable.
        IF bRecipeLoaded AND bCameraReady AND (CameraProgramLoaded = RecipeCameraProgramID) THEN
            StateCurrent := STATE_EXECUTE;
        ELSIF bStartTimeout THEN
            FaultCode := FAULT_START_CONDITION_NOT_MET;
            StateCurrent := STATE_ABORTING;
        END_IF

    STATE_EXECUTE:
        PRG_InspSequence();
        IF CmdStop THEN
            StateCurrent := STATE_STOPPING;
        ELSIF FaultActive THEN
            StateCurrent := STATE_ABORTING;
        END_IF

    STATE_STOPPING:
        IC01_CONV01_CmdRun := FALSE;
        IF bCycleDone AND NOT IC01_CONV01_FbkRunning THEN
            StateCurrent := STATE_STOPPED;
        END_IF

    STATE_STOPPED:
        IF CmdReset THEN
            StateCurrent := STATE_RESETTING;
        END_IF

    STATE_RESETTING:
        // Return actuators to a known state before going back to Idle.
        IF bResetDone THEN
            StateCurrent := STATE_IDLE;
        END_IF

    STATE_ABORTING:
        IC01_CONV01_CmdRun := FALSE;
        IC01_CYL01_CmdExtend := FALSE;
        IC01_CYL02_CmdExtend := FALSE;
        IF bAbortDone THEN
            StateCurrent := STATE_ABORTED;
        END_IF

    STATE_ABORTED:
        IF CmdClear THEN
            StateCurrent := STATE_CLEARING;
        END_IF

    STATE_CLEARING:
        IF bClearDone THEN
            StateCurrent := STATE_STOPPED;
        END_IF
END_CASE
```

### Inspection Sequence

The sequence runs only inside `Execute`. Real projects should implement command latching, timeout handling, and fault reset in reusable function blocks; the sample below shows the structure.

```iec-st
CASE iSeqStep OF
    0: // Wait for part
        IC01_CONV01_CmdRun := TRUE;
        IF IC01_PE01_Status THEN
            iSeqStep := 10;
        END_IF

    10: // Stop the part
        IC01_CYL01_CmdExtend := TRUE;
        IF IC01_CYL01_FbkExtended THEN
            iSeqStep := 20;
        ELSIF IC01_CYL01_Timeout THEN
            FaultCode := FAULT_STOPPER_EXTEND_TIMEOUT;
            FaultActive := TRUE;
        END_IF

    20: // Trigger camera
        IC01_CAM01_TriggerReq := TRUE;
        IF IC01_CAM01_InspComplete THEN
            IC01_CAM01_TriggerReq := FALSE;
            IF IC01_CAM01_InspResult THEN
                iSeqStep := 30;
            ELSE
                iSeqStep := 40;
            END_IF
        ELSIF IC01_CAM01_Fault THEN
            FaultCode := FAULT_CAMERA_INSPECTION;
            FaultActive := TRUE;
        END_IF

    30: // Pass - release part
        IC01_CYL01_CmdExtend := FALSE;
        IC01_CYL01_CmdRetract := TRUE;
        IF IC01_CYL01_FbkRetracted THEN
            iPartsPassed := iPartsPassed + 1;
            iSeqStep := 0;
        END_IF

    40: // Fail - divert to reject
        IC01_CYL02_CmdExtend := TRUE;
        IC01_CYL01_CmdExtend := FALSE;
        IC01_CYL01_CmdRetract := TRUE;
        IF IC01_CYL01_FbkRetracted THEN
            iPartsRejected := iPartsRejected + 1;
            iSeqStep := 50;
        END_IF

    50: // Reset diverter
        IC01_CYL02_CmdExtend := FALSE;
        IC01_CYL02_CmdRetract := TRUE;
        IF IC01_CYL02_FbkRetracted THEN
            IC01_CYL02_CmdRetract := FALSE;
            iSeqStep := 0;
        END_IF
END_CASE
```

**Rule 5 in action:** SCADA can send a requested recipe. The PLC decides whether it is valid for the machine state, camera program, and equipment readiness.

### OPC UA Exposure

Expose only the variables that Layer 3 and Layer 4 need. Internal sequence variables, temporary calculations, and implementation details should stay internal unless there is a clear operational reason to publish them.

```text
Root/IC01/
├── PackML/
│   ├── StateCurrent          -> Layer 3 (state display), Layer 4 (OEE)
│   ├── ModeCurrent           -> Layer 3
│   ├── CmdStart              -> Layer 3 write
│   ├── CmdStop               -> Layer 3 write
│   ├── CmdReset              -> Layer 3 write
│   └── CmdClear              -> Layer 3 write
├── Production/
│   ├── PartsPassed           -> Layer 3 + Layer 4
│   ├── PartsRejected         -> Layer 3 + Layer 4
│   └── CycleTimeMs           -> Layer 4
├── Recipe/
│   ├── RequestedRecipeID     -> Layer 3 write
│   ├── ActiveRecipeID        -> Layer 3 + Layer 4
│   ├── RecipeValid           -> Layer 3
│   └── RejectReason          -> Layer 3 + Layer 4
└── Faults/
    ├── FaultActive           -> Layer 3
    ├── FaultCode             -> Layer 3 + Layer 4
    └── FaultText             -> Layer 3
```

## Layer 3 - Build the Operator Interface

The HMI follows the PLC structure. It does not invent its own model of the machine.

| Screen | Purpose |
|--------|---------|
| Area Overview | Shows IC01 as one machine tile with state, production counts, and active alarms |
| IC01 Detail | Shows PackML faceplate, device status, camera result, counters, and active recipe |
| IC01 Recipe | Lets an operator select a recipe and send it to the PLC for validation |
| IC01 Faults | Shows active and recent PLC-detected faults with operator guidance |

**Rule 2 verification:** Remove all SCADA scripts that affect machine behavior. If IC01 can still inspect parts under PLC control, Layer 3 is doing the right job.

**Alarm list for IC01:**

| Alarm tag | Priority | Detected in | Description |
|-----------|----------|-------------|-------------|
| `IC01_CYL01_Timeout` | High | PLC | Stopper cylinder did not reach position in time |
| `IC01_CYL02_Timeout` | High | PLC | Reject diverter did not reach position in time |
| `IC01_CAM01_Fault` | High | PLC | Camera communication or inspection fault |
| `IC01_PE02_BinFull` | Medium | PLC | Reject bin full |
| `IC01_ES01_Tripped` | Critical | Safety system / PLC | E-stop activated |
| `IC01_PackML_StartFault` | Medium | PLC | Recipe, camera, or start condition failed |

SCADA presents and manages alarms. It does not decide what a process fault is.

## Layer 4 - Store and Trend the Data

Connect the historian to the IC01 OPC UA server or to an approved aggregation service. The historian is a read-only consumer of control data.

| Historian tag | Source | Storage trigger | Purpose |
|---------------|--------|-----------------|---------|
| `IC01_PackML_StateCurrent` | OPC UA | On change | OEE and state analysis |
| `IC01_Production_PartsPassed` | OPC UA | On change | Production count |
| `IC01_Production_PartsRejected` | OPC UA | On change | Quality count |
| `IC01_Production_CycleTimeMs` | OPC UA | Every cycle | Performance trend |
| `IC01_Recipe_ActiveRecipeID` | OPC UA | On change | Batch and quality context |
| `IC01_Faults_FaultCode` | OPC UA | On change | Downtime analysis |

**Rule 4 in action:** Layer 4 reads. It does not write setpoints, commands, recipe changes, or reset bits to the PLC.

**Example inspection log:**

```sql
CREATE TABLE IC01_InspectionLog (
    LogID           INT IDENTITY PRIMARY KEY,
    TimestampUtc    DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    RecipeID        VARCHAR(50) NOT NULL,
    PartType        VARCHAR(50) NULL,
    InspResult      BIT NOT NULL,
    CycleTimeMs     INT NULL,
    FaultCode       VARCHAR(50) NULL,
    BatchID         VARCHAR(50) NULL
);
```

## Layer 5 - Give It a Network Home

For a small cell, Layer 1 and Layer 2 often share a control VLAN because the PLC, drives, camera, and IO need low-latency communication. Supervisory, data, and infrastructure services should be separated and firewalled where the site architecture supports it.

| Device | IP address | VLAN | ILA layer |
|--------|------------|------|-----------|
| IC01 PLC | `10.1.10.101` | VLAN 10 Control | 2 |
| IC01 Camera | `10.1.10.201` | VLAN 10 Control | 1 |
| SCADA Server | `10.1.20.10` | VLAN 20 Supervisory | 3 |
| Historian | `10.1.30.10` | VLAN 30 Data | 4 |
| OT AD / LDAP | `10.1.40.10` | VLAN 40 Infrastructure | 5 |
| OT Firewall | `10.1.40.1` | Routed boundary | 5 |

**Example firewall rules:**

| Source | Destination | Port | Protocol | Decision |
|--------|-------------|------|----------|----------|
| SCADA | IC01 PLC | 4840 | OPC UA | Allow read and approved command writes |
| Historian | IC01 PLC | 4840 | OPC UA | Allow read only |
| IC01 PLC | IC01 Camera | Vendor-specific | Camera API / industrial protocol | Allow required control traffic |
| Historian | IC01 PLC | Any | Write commands | Deny |
| Enterprise IT | IC01 PLC | Any | Any | Deny direct access |

The firewall should enforce the same architecture the documents describe. Policy without enforcement becomes tribal knowledge.

## What You Have Now

After this walkthrough, IC01 has an ILA-shaped architecture:

- **Layer 1:** Devices have names that work as physical coordinates.
- **Layer 2:** PLC structure mirrors the machine and owns state, sequence, interlocks, and recipe validation.
- **Layer 3:** HMI displays state, captures operator intent, and presents alarms without owning process behavior.
- **Layer 4:** Historian and database store process data as read-only consumers.
- **Layer 5:** Network and access rules support the intended data and command flows.

## The One-Machine Test

Apply ILA to one machine. Then ask a technician, a PLC engineer, a SCADA engineer, and an OT infrastructure person to explain how the machine works using the same tag names and structure. If they can do it without a whiteboard archaeology session, ILA is working.

## Next Steps

- Add a second machine, `IC02`, and test whether the naming and HMI hierarchy scale.
- Add `Hold` or `Suspend` only when upstream/downstream behavior requires it.
- Add ISA-88 recipe structure if product variants or batch records require it.
- Use PackML state history to calculate OEE in Layer 4.
- Run a Rule 2 audit: find every SCADA script and classify it as display, navigation, validation, or process logic.
- Document any exception to the five rules with its reason, risk, and owner.

---

*Back to [ILA Overview](ILA-Overview.md)*
