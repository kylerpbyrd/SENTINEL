# ARCHITECTURE.md — Sentinel System Architecture

> **Project:** Sentinel | **Phase:** Architecture Complete
> **Status:** Approved — Implementation Pending Validation

---

## Purpose

This document provides the high-level architecture overview for Sentinel. It indexes all authoritative architecture documents. It does NOT duplicate detailed design — it references the authoritative sources.

---

## System Overview

Sentinel is a Manifest V3 browser extension that analyzes web page content for authenticity. The Trust Engine — the core reasoning system — combines evidence from multiple independent analyzer plugins into calibrated confidence assessments.

```
Browser Tab → Content Script → Service Worker → Sandbox/Offscreen → Fusion Engine → UI
                      ↑                                                              │
                      └────────── Messaging Bus (all contexts) ──────────────────────┘
```

---

## Major Subsystems

| Subsystem | Purpose | Authoritative Doc |
|-----------|---------|-------------------|
| **Trust Engine (Core)** | Combine evidence signals into confidence assessments. Bayesian fusion + conflict detection + explainability. | `docs/TRUST_ENGINE_SPEC.md` (product reasoning), `docs/architecture/FUSION_ENGINE_ARCHITECTURE.md` (algorithm) |
| **Evidence Pipeline** | 11-stage pipeline: Discovery → Extraction → Normalization → Validation → Evidence Objects → Confidence → Graph → Fusion → Trust → Explanation → UI | `docs/architecture/EVIDENCE_PIPELINE.md` |
| **Analyzer Plugins** | Independently loadable evidence sources. C2PA verifier, EXIF parser, ONNX image forensics, text analyzer, domain reputation, etc. | `docs/architecture/SCHEDULER_AND_PLUGIN_ARCHITECTURE.md` |
| **Scheduler** | Orchestrate evidence gathering: priority, parallelism, resource budgets, timeouts, retry, cancellation, SW keep-alive | `docs/architecture/SCHEDULER_AND_PLUGIN_ARCHITECTURE.md` |
| **Fusion Engine** | Bayesian evidence accumulation on DAG. Conflict detection. Safety overrides. 8-dimension confidence model. | `docs/architecture/FUSION_ENGINE_ARCHITECTURE.md` |
| **Explainability Engine** | 3-tier output: Quick Glance → Summary View → Detailed Report. Template-based natural language. | `docs/architecture/TRUST_ENGINE_ARCHITECTURE_MASTER.md` §9 |
| **Browser Capability Layer** | Abstracts browser differences. Analyzers query capabilities, not browser names. | `docs/architecture/TRUST_ENGINE_ARCHITECTURE_MASTER.md` §10 |
| **Invocation Architecture** | 5 invocation paths (on-demand, per-domain, always-on, context menu, mobile share) via unified AnalysisRequest | `docs/architecture/TRUST_ENGINE_ARCHITECTURE.md` §1 |
| **Privacy Architecture** | 3-tier evidence classification. Local-first default. Explicit cloud opt-in. | `docs/architecture/TRUST_ENGINE_ARCHITECTURE.md` §2 |
| **Performance Architecture** | 4 async execution tiers with strict timing budgets. Progressive result delivery. | `docs/architecture/TRUST_ENGINE_ARCHITECTURE.md` §3 |
| **Failure Architecture** | 14 failure modes with structured degradation. Every failure produces Evidence, not exceptions. | `docs/architecture/TRUST_ENGINE_ARCHITECTURE.md` §4 |

---

## Data Flow

```
1. User invokes analysis (click icon, right-click image, per-domain auto)
2. Content Script: extracts DOM text, enumerates images/video/audio, reads metadata
3. ContentSnapshot sent to Service Worker
4. Scheduler: resolves eligible analyzers, builds dependency DAG, dispatches
5. Analyzers execute in assigned context (content script, sandbox, offscreen, cloud)
6. Evidence objects (immutable) emitted and added to Evidence Graph (append-only DAG)
7. Fusion Engine: Bayesian accumulation → conflict detection → safety overrides → confidence level
8. Trust Engine: 8-dimension model → discrete confidence level → AssessmentResult
9. Explanation Generator: 3-tier output → UI display
10. UI: confidence badge, evidence cards, conflicts, actionable recommendations, export
```

---

## Extension Architecture

Six browser contexts in a single MV3 extension:

| Context | Role | Model |
|---------|------|-------|
| **Content Script** | DOM extraction, metadata reading | Isolated world per tab |
| **Service Worker** | Orchestration, scheduling, fusion, storage | 30s idle timeout, 5min max operation |
| **Sandbox Page** | WASM/ONNX compute (C2PA, forensics) | Relaxed CSP, MessagePort-only |
| **Offscreen Document** | IndexedDB proxy, tabCapture consumer | Chrome/Edge only, one at a time |
| **Popup** | Quick Glance, trigger, user controls | Short-lived (closes on blur) |
| **Side Panel** | Persistent results, detailed reports | Chrome/Edge only |

Full diagram: `docs/architecture/TRUST_ENGINE_ARCHITECTURE_MASTER.md` §11

---

## Evidence Lifecycle

```
DISCOVERY → EXTRACTION → NORMALIZATION → ANALYSIS → VALIDATION → GRAPH → FUSION → EXPLANATION → UI
```

Detailed lifecycle: `docs/architecture/TRUST_ENGINE_ARCHITECTURE_MASTER.md` §3

---

## Validation Strategy

Before any production implementation, 32 technical spikes validate every architectural assumption:

- **7 Critical spikes** (must pass — architecture redesign if any fail)
- **7 High spikes** (must pass — architecture can absorb with modification)
- **7 Medium spikes** (validate before Phase 2 implementation)
- **8 Low spikes** (can be deferred)

All spikes execute within a reusable Validation Harness (single MV3 extension).

**Exit criteria:** All Critical + High spikes pass → Architecture Readiness Review → Implementation authorized.

Full plan: `docs/architecture/VALIDATION_PLAN.md`
Harness design: `docs/architecture/VALIDATION_HARNESS.md`

---

## Architecture Decision Records (ADRs)

Architectural decisions are documented as ADRs. Current status:

| ADR | Title | Status |
|-----|-------|--------|
| ADR-001 | Extension Platform & Browser Context Strategy | Accepted |
| ADR-002 | Evidence Object Model | Accepted |
| ADR-003 | Analyzer Plugin API | Accepted |
| ADR-004 | Fusion Engine Architecture | Accepted |
| ADR-005 | Confidence Model | Accepted |
| ADR-006 | Scheduler Design | Accepted |
| ADR-007 | Storage Strategy | Accepted |
| ADR-008 | Browser Capability Layer | Accepted |
| ADR-009 | Cloud Privacy Architecture | Accepted |
| ADR-010 | Model Loading & Caching | Accepted |
| ADR-011 | Invocation Model | Accepted |
| ADR-012 | Explainability Engine | Accepted |
| ADR-013-016 | Phase 3 ADRs | Proposed |

ADRs are maintained in `docs/adr/`. Each follows the template: Problem → Context → Options Considered → Decision → Tradeoffs → Consequences.

---

## Binding Constraints

The architecture operates within these non-negotiable constraints (validated by the Capability Assessment):

- MV3 service worker: 30s idle timeout, 5min max operation
- Cross-origin pixel analysis: blocked by canvas tainting
- No reading network response bodies in MV3
- Offscreen documents: Chrome/Edge only, one at a time
- tabCapture: Chrome/Edge only, requires user gesture
- 50ms synchronous budget on page load
- chrome.storage.local: unencrypted, ~10MB
- IndexedDB: available in offscreen docs, not service workers

Full constraints: `docs/architecture/BROWSER_CAPABILITY_ASSESSMENT.md`

---

*End of ARCHITECTURE.md. For detailed design, reference the authoritative documents indexed above.*
