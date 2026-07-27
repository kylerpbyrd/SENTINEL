# PROMPT.md — Canonical Project Bootstrap

> **Project:** Sentinel | **Owner:** Byrd Ventures LLC
> **Purpose:** This file enables any new AI agent session to understand the project immediately.

---

## Project Context

Sentinel is a **Trust Layer for the Internet** — a privacy-first browser extension platform that analyzes digital content authenticity using explainable, evidence-based confidence scoring. It is NOT an AI detector. It does NOT output binary verdicts. It combines multiple independent evidence signals (provenance, forensics, reputation, cross-modal consistency) into calibrated confidence assessments.

**Current Phase:** Architecture Complete. Validation Phase Preparing. Implementation Pending.

**Organization:** Byrd Ventures LLC
**Repository:** `https://github.com/kylerpbyrd/SENTINEL.git`

---

## Architecture State (As of Last Commit)

All architecture documents are approved and frozen. No architecture changes without an ADR.

**Authoritative documents (read these first):**

| Priority | Document | What It Answers |
|----------|----------|-----------------|
| 1 | `VISION.md` | What are we building and why? |
| 2 | `docs/discovery/DISCOVERY_REPORT.md` | What did Phase 0 research find? |
| 3 | `ARCHITECTURE.md` | Architecture overview and index |
| 4 | `docs/TRUST_ENGINE_SPEC.md` | How does the Trust Engine reason? |
| 5 | `REQUIREMENTS.md` | What must the system do? |
| 6 | `AGENTS.md` | What rules bind all AI agents? |

**Detailed architecture documents:**

| Document | Domain |
|----------|--------|
| `docs/architecture/TRUST_ENGINE_ARCHITECTURE_MASTER.md` | Master integration — subsystem diagram, data flow, lifecycles, messaging, state, contexts |
| `docs/architecture/EVIDENCE_PIPELINE.md` | Evidence object model, pipeline stages, interfaces |
| `docs/architecture/FUSION_ENGINE_ARCHITECTURE.md` | Evidence graph, Bayesian fusion, conflict detection, confidence model |
| `docs/architecture/SCHEDULER_AND_PLUGIN_ARCHITECTURE.md` | Scheduler, priority system, analyzer plugin architecture |
| `docs/architecture/TRUST_ENGINE_ARCHITECTURE.md` | Invocation models, privacy boundaries, performance tiers, failure modes |
| `docs/architecture/BROWSER_CAPABILITY_ASSESSMENT.md` | **BINDING** — Browser capability assessment (CORS, MV3, WASM, storage limits) |
| `docs/architecture/VALIDATION_PLAN.md` | 32 technical spikes, exit criteria, risk matrix |
| `docs/architecture/VALIDATION_HARNESS.md` | Validation platform design (harness + spike registration) |

---

## Tech Stack Assumptions

- **Platform:** Manifest V3 browser extension (Chrome, Firefox, Edge primary; Safari secondary)
- **Language:** TypeScript (strict mode)
- **ML Runtime:** ONNX Runtime Web (WASM backend, WebGPU acceleration where available)
- **Provenance:** c2pa-js (WASM) for C2PA/Content Credentials verification
- **Compute:** Sandbox pages (WASM/ONNX), Offscreen documents (Chrome/Edge only)
- **Storage:** chrome.storage.session (ephemeral), chrome.storage.local (preferences), IndexedDB via offscreen proxy (models, reference data)
- **Messaging:** chrome.runtime.sendMessage (CS↔SW), MessagePort (SW↔Sandbox)

---

## Development Rules

1. **No implementation without validated architecture.** All critical assumptions must pass technical spikes before code begins.
2. **Architecture documents are authoritative.** If code contradicts architecture, architecture wins until an ADR amends it.
3. **TypeScript strict mode.** No `any` without justification.
4. **Immutability by default.** Evidence objects are frozen. Fusion results are append-only.
5. **Every failure produces structured Evidence, not exceptions.**
6. **Minimal dependencies.** Every dependency must be justified.
7. **Privacy-first:** local processing default, cloud opt-in only, content never stored.

---

## How to Continue Work

**If you are starting a new session:**
1. Read `AGENTS.md` for binding rules
2. Read `VISION.md` for product context
3. Read `ARCHITECTURE.md` for system overview
4. Read the domain-specific architecture document for your task
5. Check `docs/architecture/VALIDATION_PLAN.md` for current spike status
6. Check git log for recent activity

**If you are implementing a technical spike:**
1. Read `docs/architecture/VALIDATION_PLAN.md` for the spike definition
2. Read `docs/architecture/VALIDATION_HARNESS.md` for harness architecture
3. Read the spike's README in `validation-harness/spikes/spike-{id}/`
4. Implement within the harness — do not create a separate extension

**If you are implementing production code (Phase 2+):**
1. Ensure all mandatory exit criteria in VALIDATION_PLAN.md are met
2. Ensure the Architecture Readiness Review (ARR) has passed
3. Read the relevant ADRs for the subsystem you're implementing
4. Follow the `CONTRIBUTING.md` workflow

---

## Key Terminology

| Term | Definition |
|------|-----------|
| **Trust Engine** | Core reasoning system — combines evidence signals into confidence assessments |
| **Evidence** | Immutable object produced by an analyzer plugin, containing signal + confidence + metadata |
| **Fusion Engine** | Bayesian evidence accumulation + conflict detection + safety overrides |
| **Confidence Assessment** | Discrete-level output (6 levels) with explainability. Never a binary verdict. |
| **Analyzer Plugin** | Independently loadable evidence source (C2PA, EXIF, ONNX forensics, etc.) |
| **Tier 1-4** | Evidence hierarchy: Ground Truth → Strong → Moderate → Weak |
| **CapabilityProvider** | Browser abstraction layer — queries capabilities, not browser names |
| **C2PA** | Coalition for Content Provenance and Authenticity — cryptographic provenance standard |

---

*End of PROMPT.md. This file enables rapid context acquisition for new AI agent sessions.*
