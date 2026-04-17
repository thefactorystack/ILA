# ILA Layer 1 — Field

**Sensors, actuators, drives, robots, vision systems — everything with a wire or a signal.**

---

## Purpose

The Field Layer is the physical interface between the automation system and the real world. Every measurement, every motion, and every detection starts here. If this layer is poorly named, poorly wired, or poorly documented, every layer above it inherits the confusion.

## What Belongs in Layer 1

- Sensors (proximity, photoelectric, temperature, pressure, flow, level, vision)
- Actuators (solenoid valves, pneumatic cylinders, motors, drives)
- Industrial robots and cobots
- Vision systems and smart cameras
- Barcode and RFID readers
- IO-Link masters and devices
- Field wiring, junction boxes, terminal strips
- Safety devices (e-stops, light curtains, safety interlocks)

## What Does NOT Belong in Layer 1

- PLC logic (that is Layer 2)
- Network switches or firewalls (that is Layer 5)
- Any device that makes decisions about process flow

## Key Standards

### ISA 5.1 — Tag Naming

ISA 5.1 (Instrumentation Symbols and Identification) provides the naming conventions that make field devices identifiable across the entire stack.

**Why it matters:** A tag name assigned at Layer 1 is carried through PLC programs (Layer 2), SCADA screens (Layer 3), historians (Layer 4), and network documentation (Layer 5). Get it wrong here and you pay for it everywhere.

**ILA tag naming pattern:**

```
{Area}_{Unit}_{DeviceType}{Sequence}_{Attribute}
```

| Segment | Description | Example |
|---------|-------------|---------|
| Area | Functional area or cell | `IC01` (Inspection Cell 01) |
| Unit | Specific equipment or station | `CAM01` (Camera 01) |
| DeviceType + Seq | ISA 5.1 device type code | `PS01` (Pressure Switch 01) |
| Attribute | Signal type or function | `TriggerReq`, `Value`, `Alarm` |

**Examples:**

| Tag | Meaning |
|-----|---------|
| `IC01_CAM01_TriggerReq` | Inspection Cell 01 → Camera 01 → Trigger Request |
| `BR01_FV01_CmdOpen` | Brewery 01 → Fill Valve 01 → Command Open |
| `BR01_TT01_Value` | Brewery 01 → Temperature Transmitter 01 → Value |
| `PK01_CYL03_FbkExtended` | Packaging 01 → Cylinder 03 → Feedback Extended |

**Rules for tag names:**

- No spaces, no special characters beyond underscores
- English language (even in non-English-speaking plants — the tags are a technical coordinate system)
- Consistent left-to-right hierarchy: Area → Unit → Device → Attribute
- The tag name alone must tell you: where the device is, what it is, and what the signal represents

### IO-Link (IEC 61131-9)

IO-Link provides point-to-point, parameterized communication between smart field devices and the control layer.

**Where IO-Link fits in ILA:**

- IO-Link devices are Layer 1 devices
- IO-Link masters sit at the boundary between Layer 1 and Layer 2
- IO-Link device parameters should be stored and version-controlled (this supports ILA Rule 1 — the device identity is part of the architecture)

**Practical benefits:**

- Automatic device replacement (a new sensor loads its predecessor's parameters)
- Diagnostic data (signal quality, temperature, operating hours) flows up to Layer 4
- Reduced wiring complexity compared to traditional analog/digital IO

### Industrial Ethernet Protocols

Field devices increasingly communicate over Ethernet-based protocols rather than traditional 4–20 mA or 24V digital signals.

| Protocol | Typical Use | Notes |
|----------|-------------|-------|
| PROFINET | Siemens-dominated environments | Real-time, IRT for motion |
| EtherNet/IP | Rockwell / Allen-Bradley environments | CIP-based, uses standard Ethernet |
| EtherCAT | High-speed motion, packaging | Distributed clocks, very fast cycle times |
| Modbus TCP | Legacy and simple devices | No built-in security — isolate on network |

**ILA principle:** The choice of field protocol is a Layer 1 / Layer 5 decision. The Control Layer (Layer 2) should abstract field protocol specifics so that PLC logic is not dependent on a specific communication technology.

## Safety Devices

Safety-related field devices (e-stops, light curtains, safety interlocks, safe torque-off drives) are Layer 1 devices but connect to safety controllers or safety PLCs that execute safety logic according to IEC 62061 / ISO 13849.

**ILA principle:** Safety logic is Control Layer logic (Layer 2) executed on safety-rated hardware. Safety field devices are Layer 1 devices with dedicated, traceable tag names following ISA 5.1 conventions.

**Example safety tags:**

| Tag | Meaning |
|-----|---------|
| `IC01_ES01_Status` | Inspection Cell 01 → E-Stop 01 → Status |
| `IC01_LC01_Muted` | Inspection Cell 01 → Light Curtain 01 → Muted |
| `PK01_SG01_DoorClosed` | Packaging 01 → Safety Gate 01 → Door Closed |

## Robots and Vision Systems

Industrial robots and vision systems are Layer 1 devices — they are physical actuators/sensors that interact with the process. However, they often contain embedded controllers with their own programs.

**ILA principle:** The robot's internal program handles motion paths and sequences. The PLC (Layer 2) owns the process flow and tells the robot *when* to act via a handshake protocol. The robot does not decide process flow — it executes motion on command.

**Basic handshake pattern (OPC UA or discrete IO):**

```
PLC → Robot:  StartCmd = TRUE
Robot → PLC:  Busy = TRUE
Robot → PLC:  Done = TRUE
PLC → Robot:  StartCmd = FALSE (acknowledge)
Robot → PLC:  Busy = FALSE, Done = FALSE (reset)
```

This basic pattern applies equally to vision systems, label printers, or any smart field device with its own controller. Real-world implementations extend it:

**Multi-program selection:** When a robot executes different programs depending on product variant or recipe, the PLC sends a program ID before issuing StartCmd. The robot confirms the loaded program before the PLC asserts Start.

```
PLC → Robot:  ProgramID = 3
Robot → PLC:  ProgramLoaded = 3 (confirm)
PLC → Robot:  StartCmd = TRUE
```

**Fault handling:** When a robot enters a fault state while the PLC is waiting for Done, the handshake must not deadlock. The robot signals Fault, and the PLC responds by transitioning to its own fault or abort state — it does not wait indefinitely.

```
PLC → Robot:  StartCmd = TRUE
Robot → PLC:  Busy = TRUE
Robot → PLC:  Fault = TRUE, Busy = FALSE     ← fault during execution
PLC:          Detect Fault, transition to Aborting state
PLC → Robot:  StartCmd = FALSE
```

**ILA principle:** Always implement a PLC-side timeout on robot handshakes. If Done and Fault are both FALSE for longer than a defined maximum cycle time, the PLC should raise an alarm and transition to a safe state. Never assume the handshake will complete.

**Safe home position:** Define a "home" position for every robot. The PLC must be able to verify that the robot is at home (via a HomePos feedback signal) before starting a new cycle or after a fault recovery. The robot's internal program must include a "return to home" routine that can be triggered independently of the production cycle.

## Maintenance and Troubleshooting at Layer 1

Field devices are where maintenance technicians spend their time — often at 3 AM with minimal documentation at hand.

**How ILA tag naming supports troubleshooting:** A tag name like `PK01_CYL03_FbkExtended` tells the technician three things without opening a single drawing: go to Packaging area 01, find Cylinder 03, check the extended position feedback sensor. The tag name is a physical address — if it doesn't lead you to the device, the tag name is wrong.

**Diagnostic data:** Modern field devices (especially IO-Link devices) provide diagnostic information beyond their primary signal — operating hours, signal quality, internal temperature, supply voltage. ILA recommends exposing diagnostic tags alongside process tags so that Layer 4 (Data) can trend device health over time. This enables predictive maintenance rather than reactive replacement.

**Device replacement workflow:** When a field device is replaced, the new device must inherit the same tag name, the same IO-Link parameters (if applicable), and the same wiring terminal. The tag name does not change because the physical device changed — the tag identifies the *function and location*, not the specific hardware instance.

**Example diagnostic tags:**

| Tag | Purpose |
|-----|---------|
| `PK01_CYL03_FbkExtended` | Process signal — cylinder position |
| `PK01_CYL03_CycleCount` | Diagnostic — total actuations |
| `PK01_CYL03_SwitchQuality` | Diagnostic — sensor signal quality (IO-Link) |

## Practical Checklist

- [ ] Every field device has an ISA 5.1-compliant tag name
- [ ] Tag names follow the `{Area}_{Unit}_{DeviceType}{Seq}_{Attribute}` pattern
- [ ] IO-Link device parameters are backed up and version-controlled
- [ ] Safety devices are tagged and documented separately
- [ ] Robot/vision handshakes include fault handling and PLC-side timeouts
- [ ] Robot programs support multi-program selection via PLC program ID
- [ ] Safe home position is defined and verifiable by PLC for every robot
- [ ] Diagnostic tags (cycle count, signal quality) are exposed for critical devices
- [ ] Device replacement workflow preserves tag name and IO-Link parameters
- [ ] Field device documentation (datasheets, wiring diagrams) references the ILA tag name
- [ ] No field device makes autonomous process decisions — all process logic is in Layer 2

---

*Back to [ILA Overview](01-overview.md) | Next: [Layer 2 — Control](04-layer2-control.md)*
