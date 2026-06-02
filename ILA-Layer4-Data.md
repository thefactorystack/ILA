# ILA Layer 4 - Data

**Historians, databases, MQTT brokers, OPC UA aggregators, edge data services, APIs, and the read-only operational data layer.**

---

## Purpose

The Data Layer collects, stores, structures, enriches, and publishes operational data. It exists to answer: **what happened, when, where, under which recipe, and why?**

Layer 4 is powerful because it makes factory data useful outside the PLC and HMI. It is dangerous when it becomes an unofficial control layer. ILA keeps that boundary explicit.

This layer is governed most directly by **ILA Rule 4: Data flows up, commands flow down.**

## What Belongs Here

- Process historians (time-series databases)
- Relational databases (SQL) for batch records, recipe storage, quality data
- OPC UA aggregation servers
- MQTT brokers (Mosquitto, HiveMQ, etc.)
- Data pipeline services (Sparkplug B encoders, protocol translators)
- Edge computing and data preprocessing (aggregation, downsampling)
- API endpoints that expose OT data to authorized consumers
- Data contracts for MES, ERP, QMS, CMMS, and cloud consumers
- Data quality, retention, and lineage rules

## What Does Not Belong Here

- Process logic or control decisions (that is Layer 2)
- Operator interfaces or visualization (that is Layer 3)
- Network infrastructure or firewalls (that is Layer 5)
- Direct writes to PLC tags or field devices
- Undocumented data transformations that change the meaning of a process value

## Data Layer Rules

These rules keep operational data trustworthy, contextual, and safe to consume.

| Rule | Principle | Meaning |
|------|-----------|---------|
| R1 | **Define retention before go-live** | Retention policy is not an afterthought. If the first retention decision happens when the disk is full, the data architecture has already failed. |
| R2 | **Historian data is evidence, not backup** | Historian data must be trusted, timestamped, and traceable. It is evidence of what happened in production; it is not a replacement for PLC, SCADA, VM, database, or configuration backups. |
| R3 | **Data collection must never impact production** | The data layer is a passive observer. If data collection affects machine performance, scan time, network stability, or operator response, it was designed incorrectly. |
| R4 | **Data without context is worthless** | Timestamp, unit, source tag, asset relation, and quality/status are the minimum. A number without context cannot be trusted. |
| R5 | **Version everything** | Recipes, schemas, tag structures, calculations, dashboards, and data contracts must be versioned. Without version history, you cannot trust what you are looking at. |

## Key Standards

### OPC UA / IEC 62541

OPC UA (Open Platform Communications Unified Architecture) is the primary standard for structured, secure data exchange in industrial automation.

**Where OPC UA fits in ILA:**

| Connection | Direction | Purpose |
|-----------|-----------|---------|
| Layer 2 → Layer 3 | Up | PLC exposes tags to SCADA via OPC UA server |
| Layer 2 → Layer 4 | Up | Historian subscribes to PLC tags via OPC UA |
| Layer 3 → Layer 2 | Down | SCADA writes operator commands via OPC UA |
| Layer 4 -> Layer 2 | Read only | Data layer reads; it does not command the PLC |

**ILA principle:** The PLC or approved control gateway exposes the authoritative process model. Layer 4 reads that model. It may aggregate, buffer, publish, or store data, but it does not issue commands to the control system.

**OPC UA node design (aligns with Rule 5):**

The OPC UA address space should mirror the PLC program structure:

```
Root
├── {Area}
│   ├── PackML
│   │   ├── StateCurrent     (read)
│   │   ├── CmdStart         (write - Layer 3 only)
│   │   └── CmdStop          (write - Layer 3 only)
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

**ILA principle:** Sparkplug B topic structure should align with ILA naming. The topic and payload should make the site, area, unit, and device identity traceable without relying on a separate spreadsheet.

**When to use MQTT vs. OPC UA:**

| Scenario | Recommended | Rationale |
|----------|-------------|-----------|
| PLC to SCADA (Layer 2 → 3) | OPC UA | Structured, bidirectional, security built in |
| PLC to Historian (Layer 2 → 4) | OPC UA | Historian vendors have native OPC UA drivers |
| Edge to cloud or multi-site (Layer 4 -> external) | MQTT/Sparkplug B | Lightweight, firewall-friendly, pub/sub model |
| High-volume sensor streaming | MQTT | Lower overhead per message |
| Mixed vendor, many consumers | MQTT + Sparkplug B | Decoupled architecture, standardized payloads |

### UNS - Unified Namespace

A Unified Namespace (UNS) is an architectural pattern for organizing industrial data into a shared, discoverable namespace. It is not a formal ISA or IEC standard, but it is a useful way to make Layer 4 data understandable across SCADA, historians, MES, analytics, and cloud systems.

**Where UNS fits in ILA:**

- UNS belongs in Layer 4 as a data organization and publication pattern.
- UNS should preserve ILA naming and asset hierarchy instead of inventing a parallel model.
- UNS consumers should read and subscribe to data; they should not gain command authority over Layer 2.
- UNS topics or paths should carry enough context to identify site, area, unit, asset, signal, timestamp, unit, and quality/status.

**ILA principle:** A UNS is valuable only if it improves traceability. If the namespace hides the source tag, strips units, loses asset context, or becomes a backdoor command bus, it violates ILA.

**Example UNS path:**

```text
{Site}/{Area}/{Unit}/{Asset}/{Signal}

S01/IC01/Inspection/Camera01/InspResult
S01/IC01/Production/PackML/StateCurrent
```

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

**ILA principle:** The historian's tag structure should preserve the PLC tag identity. Friendly aliases are acceptable for reports and dashboards, but the source tag must remain traceable. One signal should not become five unrelated names.

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

Rule 4 - Data flows up, commands flow down - is the most critical principle at Layer 4.

**What "up" means:** Data moves from the physical process through the control system and into storage, analytics, reporting, or integration systems. At each step, the source and meaning of the data remain traceable.

**What "down" means:** Commands originate from an operator or approved supervisory workflow, pass through Layer 3, and are validated by Layer 2. Layer 4 does not initiate commands.

**Violations of Rule 4 (and why they are dangerous):**

| Violation | Risk |
|-----------|------|
| Historian writes a setpoint to the PLC | Unsupervised control action with no operator in the loop |
| Dashboard sends a start command to a machine | Bypasses HMI safety interlocks and operator awareness |
| Cloud system modifies a recipe in the PLC directly | Unaudited change with no local validation |
| Layer 4 service calculates and writes a derived tag back to PLC | Control logic now depends on an external system's availability |

## Edge Computing

Edge computing in ILA sits in Layer 4 when it processes data. It may run physically close to the machine, but physical location does not change architectural responsibility.

**Acceptable edge computing tasks:**

- Data downsampling (for example, 100 ms scan to 1 s average for historian)
- Protocol translation (OPC UA to MQTT/Sparkplug B)
- Local buffering (store-and-forward when connectivity is lost)
- Data enrichment (adding metadata: area, unit, shift context)

**Not acceptable at the edge (still Layer 2):**

- Closed-loop control decisions
- Process interlocks or safety calculations
- Real-time setpoint adjustments based on analytics

## Cloud and Hybrid Historian Architecture

Many organizations want (or are required by corporate IT) to send OT data to cloud platforms — Azure IoT Hub, AWS IoT SiteWise, Google Cloud IoT, or cloud-hosted historians. ILA takes a clear position on this:

**ILA principle: the local OT historian is the operational source of truth.** Cloud platforms may receive replicated or filtered data via the DMZ, but the plant must continue to operate and record locally when the cloud connection is unavailable. Cloud data is useful for fleet analytics; it is not authoritative for local control decisions.

**Architecture pattern:**

```
Layer 2 (PLC)
    │ OPC UA
    ▼
Layer 4 (Local Historian) <- operational source of truth
    │ Filtered replication via MQTT or API
    ▼
Layer 5 (DMZ)
    │ Outbound only, firewall-controlled
    ▼
Cloud Historian / Data Lake
```

**What to replicate to cloud:** Aggregated KPIs, batch summaries, quality summaries, equipment health metrics, and energy consumption. Be cautious with raw high-frequency control signals, safety data, credentials, and detailed PLC diagnostics. Replication scope should be intentional, documented, and approved.

**Multi-site considerations:** For organizations with multiple plants, cloud becomes the aggregation point for cross-site comparison. Each site maintains its own local historian (source of truth for that site), and a standardized data pipeline pushes agreed-upon KPIs to the cloud. ILA tag naming consistency across sites (using a site prefix: `S01_IC01_CAM01_TriggerReq`) is what makes cross-site comparison possible. Without consistent naming, cloud aggregation is an ETL nightmare.

## ISA-95 Activity Models at the Data Boundary

ISA-95 (IEC 62264) defines standardized activity models for data exchange between manufacturing operations and business systems. At Layer 4, the relevant models are:

**Production Operations:** Scheduling, dispatching, execution tracking, data collection, and performance analysis. When a work order flows down from ERP to MES to Layer 3, and production data flows back up through Layer 4, ISA-95 provides the data model to structure the exchange.

**Quality Operations:** Test execution, quality data collection, SPC (Statistical Process Control). Quality data generated at Layer 2 (inspection results, measurements) flows up to Layer 4 for storage and analysis, and may be forwarded to quality management systems using ISA-95 structures.

**Maintenance Operations:** Equipment monitoring, maintenance scheduling, failure tracking. Diagnostic data from Layer 1 (IO-Link device health) and Layer 2 (PLC fault logs) flows to Layer 4 where it supports maintenance planning.

**ILA principle:** Use ISA-95 activity models to define data contracts between OT and business systems. Ad-hoc exports may be acceptable as temporary interfaces, but they should not become undocumented production architecture.

## Data-Layer Decisions

Make these decisions explicit:

| Decision | ILA default |
|----------|-------------|
| PLC access | Layer 4 is read-only |
| Historian naming | Preserve source tag identity |
| Cloud role | Replica and analytics consumer, not control authority |
| Edge role | Buffer, transform, enrich, publish - not control |
| Data contracts | Use ISA-95 concepts where integrating with business systems |
| Unified Namespace | Organize published data using ILA hierarchy and preserve source traceability |
| Data retention | Define by use case, compliance, and storage cost |
| Data quality | Track source, timestamp, unit, and transformation history |

## Practical Checklist

- [ ] Data flows up only — Layer 4 never writes commands to Layer 2 (Rule 4)
- [ ] Historian tag names match PLC tag names exactly (Rule 5)
- [ ] OPC UA connections use certificate-based authentication
- [ ] MQTT/Sparkplug B topics align with ILA tag naming
- [ ] UNS paths preserve ILA hierarchy, source tag identity, units, and quality/status
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
- [ ] Data transformations preserve lineage and units
- [ ] Exceptions to read-only Layer 4 access are reviewed and documented

---

*Back to [ILA Overview](ILA-Overview.md) | Previous: [Layer 3 - Supervisory](ILA-Layer3-Supervisory.md) | Next: [Layer 5 - OT Platform](ILA-Layer5-OTPlatform.md)*
