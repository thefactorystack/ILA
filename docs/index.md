# Industrial Layered Architecture (ILA)

> A practical, open framework for structuring the industrial automation stack — across vendors, industries, and roles.

---

## What is ILA?

Industrial automation is full of standards. What's missing is the layer between the standard and the actual implementation — the opinionated, practical decisions that tell you *what to do on Monday morning*.

**ILA fills that gap.**

It organises the automation stack into **five layers** and **five rules** that give every technician, engineer, and manager a shared language for building factories that are understandable, scalable, and secure.

ILA is not a vendor product. It is not tied to a specific PLC brand, SCADA platform, or cloud provider. It is a framework written by practitioners, for practitioners.

---

## The Five Layers

| Layer | Name | What lives here |
|-------|------|-----------------|
| **L1** | **Field** | Sensors, actuators, robots, cameras |
| **L2** | **Control** | PLCs, motion controllers, state machines |
| **L3** | **Supervisory** | SCADA, HMI, operator interfaces |
| **L4** | **Data** | Historians, databases, analytics |
| **L5** | **OT Platform** | Network, security, infrastructure |

---

## The Five Rules

1. **Tag naming is architecture** — If your tags don't tell you where a device is and what it does, your architecture is invisible.

2. **No programming in SCADA** — SCADA displays state. SCADA does not create state.

3. **OT owns the stack — security is everyone's job** — OT infrastructure is not an IT afterthought.

4. **Data flows up, commands flow down** — Sensors report upward. Setpoints and commands travel downward. Never sideways, never skipping layers.

5. **Structure is dictated by the Control Layer** — The PLC program structure defines the machine. Everything else aligns to it.

---

## Where to Start

**New to ILA?** Start with the **[Getting Started](pages/02-getting-started.md)** guide. It walks through all five layers applied to a single inspection cell.

**Already familiar?** Jump straight to the [Overview](pages/01-overview.md) or pick a specific layer:

- [Layer 1 — Field](pages/03-layer1-field.md)
- [Layer 2 — Control](pages/04-layer2-control.md)
- [Layer 3 — Supervisory](pages/05-layer3-supervisory.md)
- [Layer 4 — Data](pages/06-layer4-data.md)
- [Layer 5 — OT Platform](pages/07-layer5-otplatform.md)

**Want to understand the standards?** Read the **[Standards Bridge](pages/08-standards-bridge.md)** to see where ILA makes decisions that ISA, IEC, and OPC standards leave open.

---

## Join the Community

ILA is built in the open. Questions, challenges, and real-world implementations are all welcome.

- **Contribute:** See [Contributing](pages/contributing.md)
- **Governance:** Learn how decisions are made in [GOVERNANCE](pages/governance.md)
- **GitHub:** [thefactorystack/ILA](https://github.com/thefactorystack/ILA)

---

## License

ILA is published under the **[Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/)**. You are free to use, adapt, and build on it — with attribution.

---

Built by [Simon Christiansen](https://github.com/simonchr).
