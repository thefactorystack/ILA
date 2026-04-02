# ILA Layer 4 — Data

**Historians, databases, MQTT brokers, OPC UA aggregation — moving and storing process data.**

---

## Purpose

The Data Layer is responsible for collecting, storing, and making process data available — without altering it and without sending commands downward. It exists to answer the question: "What happened, when, and why?"

This layer is governed most directly by **ILA Rule 4: Data flows up, commands flow down.**

## What Belongs in Layer 4

- Process historians (time-series databases)
- Relational databases (SQL) for batch records, recipe storage, quality data
- OPC UA aggregation servers
- MQTT brokers (Mosquitto, HiveMQ, etc.)
- Data pipeline services (Sparkplug B encoders, protocol translators)
- Edge computing and data preprocessing (aggregation, downsampling)
- API endpoints that expose OT data to authorized consumers

## What Does NOT Belong in Layer 4

- Process logic or control decisions (that is Layer 2)
- Operator interfaces or visualization (that is Layer 3)
- Network infrastructure or firewalls (that is Layer 5)
- Direct writes to PLC tags or field devices

## Key Standards

### OPC UA / IEC 62541

OPC UA (Open Platform Communications Unified Architecture) is the primary standard for structured, secure data exchange in industrial automation.

**Where OPC UA fits in ILA:**

| Connection | Direction | Purpose |
|-----------|-----------|---------|
| Layer 2 → Layer 3 | Up | PLC exposes tags to SCADA via OPC UA server |
| Layer 2 → Layer 4 | Up | Historian subscribes to PLC tags via OPC UA |
| Layer 3 → Layer 2 | Down | SCADA writes operator commands via OPC UA |
| Layer 4 → Layer 2 | Read only | Data layer reads — never writes commands to PLC |

**ILA principle:** The PLC (Layer 2) is the OPC UA server. Layers 3 and 4 are OPC UA clients. Commands flow down through Layer 3 (operator-initiated) — Layer 4 never issues commands to the control system.

**OPC UA node design (aligns with Rule 5):**

The OPC UA address space should mirror the PLC program structure:

```
Root
├── {Area}
│   ├── PackML
│   │   ├── StateCurrent     (read)
│   │   ├── CmdStart         (write — Layer 3 only)
│   │   └── CmdStop          (write — Layer 3 only)
│   ├── {Unit}
│   │   ├── {Device}_Value   (read)
│   │   ├── {Device}_Status  (read)
│   │   └── {Device}_Alarm   (read)
│   └── Recipe
│       ├── ActiveRecipeID   (read)
│       └── BatchID          (read)
```

**Security at the OPC UA boundary:**

- Use certificate-based authentication (not anonymous)
- Define user roles: SCADA operator (read + write commands), Historian (read only), Engineer (read + browse)
- Encrypt traffic (OPC UA Security Mode: SignAndEncrypt)
- This aligns with IEC 62443 zone and conduit design (Layer 5)

### MQTT and Sparkplug B

MQTT is a lightweight publish/subscribe protocol. Sparkplug B adds an industrial standardization layer on top of MQTT, providing structured topic namespaces, birth/death certificates, and defined payload encoding.

**Where MQTT/Sparkplug B fits in ILA:**

- MQTT brokers are Layer 4 infrastructure
- Edge nodes (MQTT publishers) sit at the boundary between Layer 2/3 and Layer 4
- MQTT subscribers consume data for historians, dashboards, or cloud forwarding

**Sparkplug B topic structure:**

```
spBv1.0/{GroupID}/{MessageType}/{EdgeNodeID}/{DeviceID}

Example:
spBv1.0/PlantFloor/DDATA/InspectionCell01/Camera01
```

**ILA principle:** Sparkplug B topic structure should align with ILA tag naming. The `EdgeNodeID` maps to the ILA Area, and the `DeviceID` maps to the ILA Unit. This ensures that data arriving at Layer 4 is self-describing and traceable back to its physical source.

**When to use MQTT vs. OPC UA:**

| Scenario | Recommended | Rationale |
|----------|-------------|-----------|
| PLC to SCADA (Layer 2 → 3) | OPC UA | Structured, bidirectional, security built in |
| PLC to Historian (Layer 2 → 4) | OPC UA | Historian vendors have native OPC UA drivers |
| Edge to cloud or multi-site (Layer 4 → external) | MQTT/Sparkplug B | Lightweight, firewall-friendly, pub/sub model |
| High-volume sensor streaming | MQTT | Lower overhead per message |
| Mixed vendor, many consumers | MQTT + Sparkplug B | Decoupled architecture, standardized payloads |

### Time-Series Databases and Historians

Process historians are the primary data store in Layer 4. They optimize for high-volume, time-stamped data with efficient compression.

**Common options:**

| Historian | Notes |
|-----------|-------|
| InfluxDB | Open source, widely used, good for smaller deployments |
| TimescaleDB | PostgreSQL extension, SQL-compatible |
| Ignition Historian | Integrated with Ignition SCADA, stores in SQL |
| OSIsoft PI (AVEVA) | Industry standard in process industries |
| FactoryTalk Historian | Rockwell ecosystem |

**ILA principle:** The historian's tag structure must mirror the PLC tag structure (Rule 5). If the PLC has `BR01_TT01_Value`, the historian stores it under the same name — not renamed, not restructured. One name, one signal, from field device to historian.

### Relational Databases (SQL)

Relational databases complement historians for structured data that is not purely time-series:

- Batch records (ISA-88 batch reports)
- Recipe storage (master recipe library)
- Quality records (inspection results, pass/fail counts)
- Equipment configuration and maintenance logs
- Alarm history (denormalized for reporting)

**Example schema pattern for batch records:**

```sql
-- Batch header
CREATE TABLE Batch (
    BatchID         VARCHAR(50) PRIMARY KEY,
    RecipeID        VARCHAR(50),
    Area            VARCHAR(20),
    Unit            VARCHAR(20),
    StartTime       DATETIME,
    EndTime         DATETIME,
    Status          VARCHAR(20)  -- Complete, Aborted, etc.
);

-- Batch phase log
CREATE TABLE BatchPhase (
    PhaseID         INT PRIMARY KEY,
    BatchID         VARCHAR(50) REFERENCES Batch(BatchID),
    PhaseName       VARCHAR(50),
    StartTime       DATETIME,
    EndTime         DATETIME,
    Result          VARCHAR(20)
);

-- Batch parameter log
CREATE TABLE BatchParameter (
    ParameterID     INT PRIMARY KEY,
    BatchID         VARCHAR(50) REFERENCES Batch(BatchID),
    PhaseName       VARCHAR(50),
    ParameterName   VARCHAR(50),
    Setpoint        FLOAT,
    ActualValue     FLOAT,
    Unit            VARCHAR(10)
);
```

## Data Flow Discipline (Rule 4)

Rule 4 — Data flows up, commands flow down — is the most critical principle at Layer 4.

**What "up" means:** Data moves from the physical process (Layer 1) through the PLC (Layer 2), optionally through SCADA (Layer 3), and into the Data Layer (Layer 4). At each step, the data is read — never modified in transit.

**What "down" means:** Commands originate from an operator or system at Layer 3, pass through OPC UA to Layer 2, and are executed at Layer 1. Layer 4 never initiates commands.

**Violations of Rule 4 (and why they are dangerous):**

| Violation | Risk |
|-----------|------|
| Historian writes a setpoint to the PLC | Unsupervised control action — no operator in the loop |
| Dashboard sends a start command to a machine | Bypasses HMI safety interlocks and operator awareness |
| Cloud system modifies a recipe in the PLC directly | Unaudited change, no local validation |
| Layer 4 service calculates and writes a derived tag back to PLC | Control logic now depends on an external system's availability |

## Edge Computing

Edge computing in ILA sits within Layer 4 — it processes and transforms data close to the source before forwarding it to historians, cloud, or analytics platforms.

**Acceptable edge computing tasks:**

- Data downsampling (e.g., 100ms scan → 1s average for historian)
- Protocol translation (OPC UA → MQTT/Sparkplug B)
- Local buffering (store-and-forward when connectivity is lost)
- Data enrichment (adding metadata: area, unit, shift context)

**Not acceptable at the edge (still Layer 2):**

- Closed-loop control decisions
- Process interlocks or safety calculations
- Real-time setpoint adjustments based on analytics

## Cloud and Hybrid Historian Architecture

Many organizations want (or are required by corporate IT) to send OT data to cloud platforms — Azure IoT Hub, AWS IoT SiteWise, Google Cloud IoT, or cloud-hosted historians. ILA takes a clear position on this:

**ILA principle: The local OT historian is always the source of truth.** Cloud receives a replicated, filtered subset of data via the DMZ. If the cloud connection goes down, the plant continues to operate and record data locally. If the local historian goes down, the cloud copy is not authoritative for process decisions.

**Architecture pattern:**

```
Layer 2 (PLC)
    │ OPC UA
    ▼
Layer 4 (Local Historian) ← Source of truth
    │ Filtered replication via MQTT or API
    ▼
Layer 5 (DMZ)
    │ Outbound only, firewall-controlled
    ▼
Cloud Historian / Data Lake
```

**What to replicate to cloud:** Aggregated KPIs (OEE, throughput, quality rates), batch summary records, equipment health metrics, energy consumption. What *not* to replicate: raw high-frequency control signals, safety system states, detailed PLC diagnostics — these stay on-premises.

**Multi-site considerations:** For organizations with multiple plants, cloud becomes the aggregation point for cross-site comparison. Each site maintains its own local historian (source of truth for that site), and a standardized data pipeline pushes agreed-upon KPIs to the cloud. ILA tag naming consistency across sites (using a site prefix: `S01_IC01_CAM01_TriggerReq`) is what makes cross-site comparison possible. Without consistent naming, cloud aggregation is an ETL nightmare.

## ISA-95 Activity Models at the Data Boundary

ISA-95 (IEC 62264) defines standardized activity models for data exchange between manufacturing operations and business systems. At Layer 4, the relevant models are:

**Production Operations:** Scheduling, dispatching, execution tracking, data collection, performance analysis. When a work order flows down from ERP → MES → Layer 3, and production data flows back up through Layer 4, ISA-95 provides the data model (B2MML — Business to Manufacturing Markup Language) to structure this exchange.

**Quality Operations:** Test execution, quality data collection, SPC (Statistical Process Control). Quality data generated at Layer 2 (inspection results, measurements) flows up to Layer 4 for storage and analysis, and may be forwarded to quality management systems using ISA-95 structures.

**Maintenance Operations:** Equipment monitoring, maintenance scheduling, failure tracking. Diagnostic data from Layer 1 (IO-Link device health) and Layer 2 (PLC fault logs) flows to Layer 4 where it supports maintenance planning.

**ILA principle:** Use ISA-95 activity models to define the data contracts between OT (Layers 1-4) and business systems (ERP, QMS, CMMS). This makes the integration structured and vendor-independent. Do not build ad-hoc CSV exports or custom API integrations without referencing ISA-95 — you will rebuild them every time a system is upgraded.

## Practical Checklist

- [ ] Data flows up only — Layer 4 never writes commands to Layer 2 (Rule 4)
- [ ] Historian tag names match PLC tag names exactly (Rule 5)
- [ ] OPC UA connections use certificate-based authentication
- [ ] MQTT/Sparkplug B topics align with ILA tag naming
- [ ] Batch records follow ISA-88 structure
- [ ] Edge computing is limited to data processing — no control logic
- [ ] Data retention policies are defined and documented
- [ ] SQL database schemas reference ILA areas and units consistently
- [ ] All data consumers are read-only clients of Layer 2 data
- [ ] Local historian is the source of truth — cloud receives filtered replicas only
- [ ] Cloud replication uses outbound-only connections through the DMZ
- [ ] Multi-site tag naming uses site prefix for cross-site consistency
- [ ] ISA-95 activity models structure the data exchange with business systems
- [ ] MES/ERP integrations use standardized data contracts, not ad-hoc exports

---

*Back to [ILA Overview](ILA-Overview.md) | Previous: [Layer 3 — Supervisory](ILA-Layer3-Supervisory.md) | Next: [Layer 5 — OT Platform](ILA-Layer5-OTPlatform.md)*
