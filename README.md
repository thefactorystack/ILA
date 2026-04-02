# Industrial Layered Architecture (ILA)

> A practical, open framework for structuring the industrial automation stack — across vendors, industries, and roles.

---

## What is ILA?

Industrial automation is full of standards. What's missing is the layer between the standard and the actual implementation — the opinionated, practical decisions that tell you *what to do on Monday morning*.

ILA fills that gap.

It organises the automation stack into **five layers** and **five rules** that give every technician, engineer, and manager a shared language for building factories that are understandable, scalable, and secure.

ILA is not a vendor product. It is not tied to a specific PLC brand, SCADA platform, or cloud provider. It is a framework written by practitioners, for practitioners.

---

## The Five Layers

| Layer | Name | What lives here |
|-------|------|-----------------|
| L1 | Field | Sensors, actuators, robots, cameras |
| L2 | Control | PLCs, motion controllers, state machines |
| L3 | Supervisory | SCADA, HMI, operator interfaces |
| L4 | Data | Historians, databases, analytics |
| L5 | OT Platform | Network, security, infrastructure |

---

## The Five Rules

1. **Tag naming is architecture**
2. **No programming in SCADA**
3. **OT owns the stack — security is everyone's job**
4. **Data flows up, commands flow down**
5. **Structure is dictated by the Control Layer**

---

## Documentation

| Document | Description |
|----------|-------------|
| [ILA-Overview.md](ILA-Overview.md) | Framework introduction and core concepts |
| [ILA-GettingStarted.md](ILA-GettingStarted.md) | Start here — practical onboarding guide |
| [ILA-Layer1-Field.md](ILA-Layer1-Field.md) | Field Layer specification |
| [ILA-Layer2-Control.md](ILA-Layer2-Control.md) | Control Layer specification |
| [ILA-Layer3-Supervisory.md](ILA-Layer3-Supervisory.md) | Supervisory Layer specification |
| [ILA-Layer4-Data.md](ILA-Layer4-Data.md) | Data Layer specification |
| [ILA-Layer5-OTPlatform.md](ILA-Layer5-OTPlatform.md) | OT Platform Layer specification |
| [ILA-Standards-Bridge.md](ILA-Standards-Bridge.md) | How ILA relates to ISA, IEC, and OPC standards |
| [GOVERNANCE.md](GOVERNANCE.md) | Community governance and RFC process |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to contribute |

---

## Join the Community

ILA is built in the open. Questions, challenges, and real-world implementations are all welcome.

👉 [Start a discussion](../../discussions)

---

## License

ILA is published under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/). You are free to use, adapt, and build on it — with attribution.

---

Built by Simon Christiansen.
