# Sentinel — Trust Layer for the Internet

**A privacy-first platform for digital content authenticity assessment.**

[![Status](https://img.shields.io/badge/status-architecture%20complete-blue)]()
[![Phase](https://img.shields.io/badge/phase-validation%20preparing-orange)]()
[![License](https://img.shields.io/badge/license-Apache%202.0-green)]()

---

## What Is Sentinel?

Sentinel is **not** an AI detector. Sentinel is a **Trust Layer for the Internet.**

It analyzes digital content (images, text, video, audio) and produces **explainable confidence assessments** — showing users what evidence exists, how strong it is, and why the system reached its conclusion. It never outputs binary "AI" or "human" verdicts.

Sentinel combines multiple independent evidence signals:
- **Provenance:** C2PA/Content Credentials verification, hardware attestation
- **Forensics:** AI generator fingerprinting, noise analysis, artifact detection
- **Reputation:** Domain history, publisher credibility
- **Cross-modal:** Physics violations, lip-sync mismatches, semantic gaps

---

## Current Status

```
Phase 0: Discovery & Research        ✅ COMPLETE
Phase 1: Architecture                 ✅ COMPLETE
Phase 2: Validation & MVP             🔜 PREPARING
Phase 3: Platform & Cloud             📋 PLANNED
Phase 4: Expansion                    📋 PLANNED
```

**Architecture is complete and approved. Implementation is pending technical spike validation.**

---

## Architecture

Sentinel is a **Manifest V3 browser extension** (Chrome, Firefox, Edge, Safari) with a modular, plugin-based Trust Engine.

| Document | Description |
|----------|-------------|
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | High-level architecture overview and index |
| [`docs/TRUST_ENGINE_SPEC.md`](docs/TRUST_ENGINE_SPEC.md) | Product reasoning specification |
| [`docs/architecture/TRUST_ENGINE_ARCHITECTURE_MASTER.md`](docs/architecture/TRUST_ENGINE_ARCHITECTURE_MASTER.md) | System architecture (subsystems, data flow, lifecycles) |
| [`docs/architecture/EVIDENCE_PIPELINE.md`](docs/architecture/EVIDENCE_PIPELINE.md) | Evidence object model and pipeline |
| [`docs/architecture/FUSION_ENGINE_ARCHITECTURE.md`](docs/architecture/FUSION_ENGINE_ARCHITECTURE.md) | Fusion engine and confidence model |
| [`docs/architecture/SCHEDULER_AND_PLUGIN_ARCHITECTURE.md`](docs/architecture/SCHEDULER_AND_PLUGIN_ARCHITECTURE.md) | Scheduler and analyzer plugin architecture |
| [`docs/architecture/VALIDATION_PLAN.md`](docs/architecture/VALIDATION_PLAN.md) | Technical spike plan (32 spikes, exit criteria) |
| [`docs/architecture/VALIDATION_HARNESS.md`](docs/architecture/VALIDATION_HARNESS.md) | Validation platform design |

---

## Quick Links

- [Vision](VISION.md) — Why Sentinel exists
- [Requirements](REQUIREMENTS.md) — What Sentinel must do
- [Roadmap](ROADMAP.md) — Development phases and milestones
- [Security](SECURITY.md) — Threat model and security rules
- [Privacy](PRIVACY.md) — Privacy philosophy and data handling
- [Contributing](CONTRIBUTING.md) — How to contribute
- [Business Plan](BUSINESS_PLAN.md) — Commercial strategy
- [Market Research](MARKET_RESEARCH.md) — Competitive landscape

---

## For AI Agents

If you are an AI agent working on this project, start here:
1. [`AGENTS.md`](AGENTS.md) — Binding rules for all AI agents
2. [`PROMPT.md`](PROMPT.md) — Project bootstrap and context

---

## Tech Stack

- **Platform:** Manifest V3 browser extension
- **Language:** TypeScript (strict mode)
- **ML Runtime:** ONNX Runtime Web (WASM + WebGPU)
- **Provenance:** c2pa-js (WASM) for C2PA verification
- **Fusion:** Bayesian evidence accumulation with conflict detection

---

## License

Open-core model:

- **Open Source (Apache 2.0):** Browser extension core, evidence pipeline framework, C2PA verification library, signal definition format, SDKs
- **Proprietary:** Detection models, trust graph, multi-modal fusion engine, cloud infrastructure

See [`LICENSE`](LICENSE) for details.

---

## Organization

**Byrd Ventures LLC** — Project owner and maintainer.

---

*Sentinel: Because you deserve to know what's real.*
