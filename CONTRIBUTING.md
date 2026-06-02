# Contributing to ILA

Thank you for your interest in contributing to Industrial Layered Architecture.

ILA is built by practitioners, for practitioners. Every contribution helps: a question from a technician, a correction from a PLC engineer, an example from a system integrator, or a full RFC from an OT architect.

---

## How to Contribute

### Ask a Question or Start a Discussion
Open a thread in [GitHub Discussions](https://github.com/thefactorystack/ILA/discussions). This is the best place for questions, ideas, and feedback that don't require a code or documentation change.

### Report an Error or Suggest an Improvement
Open an [Issue](https://github.com/thefactorystack/ILA/issues) with a clear description of what's wrong or what could be better. Include the file name and section if relevant.

Good issues usually include:

- What you expected the document to say
- What it currently says
- Why the difference matters in a real plant
- Any relevant standard, implementation example, or field experience

### Contribute a Change
1. Fork the repository
2. Create a branch: `git checkout -b your-topic`
3. Make your changes
4. Submit a pull request with a clear description of what you changed and why

All pull requests require review from at least one Core Contributor before merging.

---

## What We Are Looking For

- Corrections to documentation (factual errors, unclear language, broken links)
- Real-world examples of ILA applied in practice
- Industry-specific adaptations (food & beverage, pharma, automotive, etc.)
- Diagrams, checklists, templates, and implementation patterns
- Clarifications that make ILA easier to adopt in brownfield factories
- Translations (future)

## Contribution Quality Bar

ILA is intentionally practical. Contributions should:

- Be grounded in real industrial experience or a clearly cited standard
- Preserve the five-layer model unless proposing an RFC
- Make the framework easier to understand or apply
- Avoid vendor marketing language
- Distinguish between a rule, a recommendation, an example, and an exception
- Include enough context that a reviewer can understand the operational impact

---

## What Requires an RFC

Changes to the ILA Core Specification require an RFC. See [Governance](GOVERNANCE.md) for the full RFC process.

Examples that require an RFC:

- Changing the definition of a layer
- Adding, removing, or changing one of the five rules
- Changing the default naming pattern
- Introducing a new required standard
- Changing governance or contributor roles

Examples that usually do not require an RFC:

- Fixing unclear wording
- Adding a practical example
- Adding a checklist item that supports an existing rule
- Correcting a broken link or typo
- Adding an industry-specific note that does not change the core model

For everything else, a pull request is sufficient.

---

## Language

English is the working language for documentation, examples, tags, and code comments. Operator-facing HMI text may be localized in real implementations, but the framework documentation uses English so it can be shared across plants, vendors, and countries.

---

## Code of Conduct

Be professional. Assume good faith. Keep discussions focused on the work.

Full details are in [Governance](GOVERNANCE.md).
