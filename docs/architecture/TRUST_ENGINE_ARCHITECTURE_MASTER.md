# Trust Engine Architecture — Master Integration Document

> **Document Type:** Architecture Specification
> **Phase:** 1 — Architecture
> **Status:** Draft — Awaiting Stakeholder Review
> **Owner:** Lead Architect
> **Audience:** Architecture Team, Engineering, Product, Security
> **Depends On:**
> - `TRUST_ENGINE_SPEC.md` (Product Reasoning Specification — authoritative)
> - `docs/architecture/BROWSER_CAPABILITY_ASSESSMENT.md` (Evidence Gathering Capability Assessment — binding constraints)
> - `docs/architecture/EVIDENCE_PIPELINE.md` (Evidence Object Model & Pipeline)
> - `docs/architecture/FUSION_ENGINE_ARCHITECTURE.md` (Fusion Engine & Confidence Model)
> - `docs/architecture/SCHEDULER_AND_PLUGIN_ARCHITECTURE.md` (Scheduler & Plugin Architecture)
> - `docs/architecture/TRUST_ENGINE_ARCHITECTURE.md` (Invocation, Privacy, Performance, Failure)

**Scope:** This document integrates all subsystem architectures into a complete Trust Engine architecture. It provides the subsystem diagram, data flow, browser messaging, state transitions, Explainability Engine, Browser Capability Layer, Extension Architecture Diagram, and ADR recommendations.

---

## Table of Contents

1. [Subsystem Architecture](#1-subsystem-architecture)
2. [Data Flow](#2-data-flow)
3. [Evidence Lifecycle](#3-evidence-lifecycle)
4. [Analysis Lifecycle](#4-analysis-lifecycle)
5. [Browser Messaging Architecture](#5-browser-messaging-architecture)
6. [State Transitions](#6-state-transitions)
7. [Storage Architecture](#7-storage-architecture)
8. [Context Responsibilities](#8-context-responsibilities)
9. [Explainability Engine Architecture](#9-explainability-engine-architecture)
10. [Browser Capability Layer](#10-browser-capability-layer)
11. [Extension Architecture Diagram](#11-extension-architecture-diagram)
12. [ADR Recommendations](#12-adr-recommendations)

---

## 1. Subsystem Architecture

### 1.1 Subsystem Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           BROWSER EXTENSION                              │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    CONTENT SCRIPT (per tab)                        │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐     │   │
│  │  │ Content   │  │ Metadata │  │ DOM       │  │ Invocation   │     │   │
│  │  │ Discovery │  │ Extract  │  │ Observer  │  │ Handler      │     │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘     │   │
│  │       └──────────────┴────────────┴───────────────┘              │   │
│  │                          │ Content Message Bus                     │   │
│  └──────────────────────────┼────────────────────────────────────────┘   │
│                             │ chrome.runtime.sendMessage                  │
│  ┌──────────────────────────┼────────────────────────────────────────┐   │
│  │                    SERVICE WORKER                                   │   │
│  │                          ▼                                          │   │
│  │  ┌────────────────────────────────────────────────────────────┐    │   │
│  │  │                    MESSAGE ROUTER                            │    │   │
│  │  │  Routes messages to: Scheduler, CapabilityProvider, Cache,  │    │   │
│  │  │  AnalysisSessionManager                                     │    │   │
│  │  └────────┬──────────┬──────────┬──────────┬──────────────────┘    │   │
│  │           │          │          │          │                        │   │
│  │           ▼          ▼          ▼          ▼                        │   │
│  │  ┌────────┐  ┌──────────┐  ┌────────┐  ┌──────────────────┐       │   │
│  │  │Schedule│  │Capability│  │Analysis│  │Storage/Cache     │       │   │
│  │  │r       │  │Provider  │  │Session │  │Manager           │       │   │
│  │  │        │  │          │  │Manager │  │                  │       │   │
│  │  └───┬────┘  └──────────┘  └───┬────┘  └──────────────────┘       │   │
│  │      │                         │                                    │   │
│  │      │    ┌────────────────────┘                                    │   │
│  │      │    │                                                         │   │
│  │      ▼    ▼                                                         │   │
│  │  ┌──────────────────────────────────────────────┐                  │   │
│  │  │           ANALYSIS SESSION                    │                  │   │
│  │  │  ┌──────────┐  ┌──────────┐  ┌────────────┐ │                  │   │
│  │  │  │ Evidence │  │ Evidence │  │  Fusion     │ │                  │   │
│  │  │  │ Pipeline │  │ Graph    │  │  Engine     │ │                  │   │
│  │  │  │(orchest.)│  │          │  │             │ │                  │   │
│  │  │  └────┬─────┘  └────┬─────┘  └──────┬──────┘ │                  │   │
│  │  │       └──────────────┴──────────────┘         │                  │   │
│  │  │                          │                     │                  │   │
│  │  │                          ▼                     │                  │   │
│  │  │  ┌──────────────────────────────────────┐     │                  │   │
│  │  │  │        TRUST ENGINE (core)            │     │                  │   │
│  │  │  │  Confidence Model + Explainability    │     │                  │   │
│  │  │  └──────────────────┬───────────────────┘     │                  │   │
│  │  └─────────────────────┼─────────────────────────┘                  │   │
│  │                        │                                            │   │
│  │                        ▼                                            │   │
│  │  ┌──────────────────────────────────────────────────────────┐      │   │
│  │  │              PROGRESS & RESULT BUS                        │      │   │
│  │  │  Publishes: EvidenceReceived, GraphUpdated,              │      │   │
│  │  │  ConfidenceChanged, AssessmentCompleted                  │      │   │
│  │  └──────────────────────┬───────────────────────────────────┘      │   │
│  └─────────────────────────┼──────────────────────────────────────────┘   │
│                            │ chrome.runtime.sendMessage / port.postMessage  │
│  ┌─────────────────────────┼──────────────────────────────────────────┐   │
│  │                    UI CONTEXTS                                       │   │
│  │                         │                                            │   │
│  │  ┌──────────────────────┼──────────────────────────────────────┐   │   │
│  │  │  POPUP / SIDE PANEL  │                                        │   │   │
│  │  │  ┌────────────┐  ┌───┴──────────┐  ┌───────────────────┐    │   │   │
│  │  │  │ Explanation│  │ Trust         │  │ User Controls     │    │   │   │
│  │  │  │ Generator  │  │ Assessment    │  │ (override, report, │    │   │   │
│  │  │  │            │  │ Display       │  │  export, settings) │    │   │   │
│  │  │  └────────────┘  └──────────────┘  └───────────────────┘    │   │   │
│  │  └──────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    SANDBOXED COMPUTE CONTEXT                       │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐     │   │
│  │  │ C2PA     │  │ Image    │  │ Audio    │  │ Text         │     │   │
│  │  │ Verifier │  │ Forensic │  │ Forensic │  │ Analyzer     │     │   │
│  │  │ (WASM)   │  │ (ONNX)   │  │ (ONNX)   │  │ (ONNX / JS)  │     │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘     │   │
│  │  Communication: MessagePort ONLY. No storage access.              │   │
│  │  One sandbox page OR one offscreen doc for heavy compute.         │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    PERIODIC BACKGROUND TASKS                       │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐    │   │
│  │  │ Reference    │  │ Model        │  │ Calibration          │    │   │
│  │  │ Data Sync    │  │ Update Check │  │ Telemetry Upload     │    │   │
│  │  │ (chrome.alarm)│  │ (chrome.alarm)│  │ (opt-in only)       │    │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Subsystem Responsibilities

| Subsystem | Location | Responsibility |
|-----------|----------|---------------|
| **Content Discovery** | Content Script | Identify content types on page, extract DOM text, enumerate images/video/audio, detect social media containers |
| **Metadata Extractor** | Content Script | Read EXIF/IPTC/XMP from same-origin images, extract C2PA manifest location, read IPTC Digital Source Type |
| **DOM Observer** | Content Script | Monitor SPA navigation, infinite scroll, dynamic content injection via MutationObserver |
| **Invocation Handler** | Content Script | Respond to context menu clicks, receive popup→tab analysis requests, build ContentSnapshot |
| **Message Router** | Service Worker | Central message dispatch. Routes to correct AnalysisSession, Scheduler, or CapabilityProvider |
| **Scheduler** | Service Worker | Orchestrate all evidence gathering: priority, parallelism, resource budgets, timeouts, retry, cancellation |
| **CapabilityProvider** | Service Worker | Detect browser capabilities (WebGPU, tabCapture, offscreen, WASM threading, Chrome AI APIs) |
| **AnalysisSessionManager** | Service Worker | Create/destroy analysis sessions per tab. Manage session lifecycle. Handle navigation cancellation |
| **Storage/Cache Manager** | Service Worker | Read/write chrome.storage (.local, .session, .managed). Manage IndexedDB via offscreen doc proxy |
| **Evidence Pipeline** | Service Worker (orchestration) + Sandbox (compute) | 11-stage pipeline from discovery to UI. Orchestration in SW; compute delegated to sandboxes |
| **Evidence Graph** | Service Worker (in-memory) | Append-only DAG of immutable Evidence nodes. Single source of truth per analysis session |
| **Fusion Engine** | Service Worker | Bayesian evidence accumulation, conflict detection, safety overrides, confidence classification |
| **Trust Engine (core)** | Service Worker | Confidence Model (8 dimensions), mapping to 6 discrete levels, assessment finalization |
| **Progress & Result Bus** | Service Worker | Publish events: EvidenceReceived, GraphUpdated, ConfidenceChanged, AssessmentCompleted |
| **Sandbox Compute** | Sandbox Page / Offscreen Document | Heavy computation: WASM C2PA verification, ONNX model inference, image processing |
| **Explanation Generator** | Popup / Side Panel | Transform AssessmentResult into 3-tier explainability output (Quick Glance / Summary / Detailed) |
| **Trust Assessment Display** | Popup / Side Panel | Render confidence level, evidence breakdown, conflicts, missing signals, actionable recommendations |
| **User Controls** | Popup / Side Panel | Override, report, export (signed PDF/JSON), settings, consent management |
| **Background Tasks** | Service Worker (chrome.alarms) | Reference data sync, model update checks, calibration telemetry upload (opt-in) |

---

## 2. Data Flow

### 2.1 End-to-End Analysis Flow

```
USER ACTION           CONTENT SCRIPT        SERVICE WORKER          SANDBOX/OFFSCREEN       UI
───────────           ──────────────        ──────────────          ────────────────        ──

Click extension    ──►
icon / right-click
                        ┌────────────────┐
                        │ Invocation     │
                        │ Handler builds │
                        │ ContentSnapshot│
                        └───────┬────────┘
                                │ chrome.runtime.sendMessage({type:'ANALYZE', snapshot})
                                ▼
                    ┌───────────────────────────────────────┐
                    │ Message Router → AnalysisSessionManager│
                    │ Creates or retrieves session for tabId │
                    └───────────────────┬───────────────────┘
                                        │
                                        ▼
                    ┌───────────────────────────────────────┐
                    │ Scheduler receives AnalysisRequest    │
                    │ 1. Queries CapabilityProvider         │
                    │ 2. Resolves eligible analyzers        │
                    │ 3. Builds dependency DAG              │
                    │ 4. Creates execution groups           │
                    │ 5. Begins dispatching                 │
                    └───────────────────┬───────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                    │
                    ▼                   ▼                    ▼
            ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
            │ TIER 1:      │   │ TIER 2:      │   │ TIER 3:      │
            │ IMMEDIATE    │   │ FAST         │   │ STANDARD     │
            │ Metadata ext.│   │ C2PA verify  │   │ ONNX models  │
            │ Hash lookup  │   │ Text analysis│   │ Face-swap    │
            │ Domain cache │   │ EXIF parse   │   │ Audio detect │
            └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
                   │                  │                    │
                   │ Evidence objects │                    │
                   ▼                  ▼                    ▼
            ┌──────────────────────────────────────────────────┐
            │              EVIDENCE GRAPH (DAG)                 │
            │  Nodes: Immutable Evidence objects                │
            │  Edges: supports, contradicts, derived_from,      │
            │         correlates_with, supersedes               │
            └──────────────────────┬───────────────────────────┘
                                   │
                                   ▼
            ┌──────────────────────────────────────────────────┐
            │              FUSION ENGINE                        │
            │  1. Dependency Resolver (cap weights)             │
            │  2. Bayesian Accumulator (log-odds)               │
            │  3. Conflict Detector                             │
            │  4. Safety Override                               │
            │  5. Confidence Classifier                         │
            │  6. Dimension Computer (8 dimensions)             │
            └──────────────────────┬───────────────────────────┘
                                   │
                                   ▼
            ┌──────────────────────────────────────────────────┐
            │              TRUST ENGINE                         │
            │  Maps 8 dimensions → 6 discrete confidence levels │
            │  Produces AssessmentResult                        │
            └──────────────────────┬───────────────────────────┘
                                   │
                                   │ Progress & Result Bus
                                   │ Port messages to UI
                                   ▼
            ┌──────────────────────────────────────────────────┐
            │              EXPLANATION GENERATOR                │
            │  Quick Glance (1-3s) → Summary (10-30s) →        │
            │  Detailed Report (2-5min, exportable)             │
            └──────────────────────┬───────────────────────────┘
                                   │
                                   ▼
            ┌──────────────────────────────────────────────────┐
            │              UI DISPLAY                           │
            │  Confidence level badge + evidence cards +        │
            │  conflicts + actionable recommendations           │
            └──────────────────────────────────────────────────┘
```

### 2.2 Progressive Analysis Flow

Analysis results are delivered progressively as evidence tiers complete:

```
t=0ms       AnalysisRequest received
t=50ms      Capability detection complete
t=80ms      Analyzer dependency DAG built
t=100ms     Tier 1 (IMMEDIATE) dispatched
t=200ms     First Evidence nodes added to Graph
t=250ms     Fusion Engine runs first computation
t=300ms     PROGRESSIVE RESULT #1 sent to UI: "Preliminary assessment"
            (Confidence level may be LOW due to insufficient evidence)
t=500ms     Tier 1 complete. Tier 2 (FAST) dispatched.
t=1.5s      C2PA verification completes. Evidence nodes added.
t=1.6s      Fusion Engine recomputes.
t=1.7s      PROGRESSIVE RESULT #2: "Refined assessment"
            (Confidence may have changed — animation transition)
t=2s        Tier 2 complete. Tier 3 (STANDARD) dispatched.
t=5s        ONNX models complete. Evidence nodes added.
t=5.1s      Fusion Engine recomputes.
t=5.2s      PROGRESSIVE RESULT #3: "Comprehensive assessment"
t=10s       Tier 3 complete. Tier 4 (CLOUD) dispatched (if opted in).
t=15s       Cloud results return. Final Evidence nodes added.
t=15.1s     Fusion Engine recomputes.
t=15.2s     FINAL RESULT. AssessmentCompleted event fired.
```

---

## 3. Evidence Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│                        EVIDENCE LIFECYCLE                            │
│                                                                      │
│  DISCOVERY                                                           │
│  ├─ Content Script identifies content types on page                  │
│  ├─ Enumerates images, text blocks, video, audio elements            │
│  └─ Produces: ContentSnapshot {text[], imageRefs[], videoRefs[]}     │
│           │                                                          │
│           ▼                                                          │
│  EXTRACTION                                                          │
│  ├─ Per-content-type extraction: DOM text, image bytes, audio stream │
│  ├─ Respects CORS: same-origin = full; cross-origin = metadata only  │
│  ├─ Uses fetch() for image bytes; tabCapture for video/audio         │
│  └─ Produces: RawContent {type, bytes?, metadata?, domText?}         │
│           │                                                          │
│           ▼                                                          │
│  NORMALIZATION                                                       │
│  ├─ Standardize to canonical formats: JPEG→pixel buffer, text→UTF-8  │
│  ├─ Truncate oversized content (max image: 4096px, max text: 50K ch)│
│  ├─ Hash content for dedup (SHA-256)                                 │
│  └─ Produces: NormalizedContent {type, data, hash, dimensions}       │
│           │                                                          │
│           ▼                                                          │
│  ANALYSIS (per analyzer plugin)                                      │
│  ├─ Scheduler dispatches NormalizedContent to eligible analyzers     │
│  ├─ Analyzers run in assigned context (content script, sandbox, etc) │
│  ├─ Each analyzer produces independently                             │
│  └─ Produces: Evidence (immutable, 19 fields)                        │
│           │                                                          │
│           ▼                                                          │
│  VALIDATION                                                          │
│  ├─ Evidence object validated against schema                         │
│  ├─ Confidence must be 0.0-1.0 with calibration metadata             │
│  ├─ Timestamps must be ISO 8601 and logically consistent             │
│  ├─ Rejected if: missing required fields, impossible confidence,     │
│  │   circular dependency, content hash mismatch                      │
│  └─ Produces: Validated Evidence (same object, validated flag set)   │
│           │                                                          │
│           ▼                                                          │
│  EVIDENCE GRAPH INSERTION                                            │
│  ├─ Node added to Evidence Graph (append-only DAG)                   │
│  ├─ Edges created based on dependency, correlation, contradiction    │
│  ├─ Graph checked for acyclicity                                     │
│  └─ Triggers: GraphUpdated event → Fusion Engine recomputation       │
│           │                                                          │
│           ▼                                                          │
│  FUSION                                                              │
│  ├─ Dependency Resolver caps weights, applies correlation penalty    │
│  ├─ Bayesian Accumulator computes posterior + credible interval      │
│  ├─ Conflict Detector classifies disagreements                      │
│  ├─ Safety Override applies floor constraints                       │
│  ├─ Confidence Classifier maps to discrete level                    │
│  └─ Produces: FusionResult {posterior, conflicts, dimensionScores}   │
│           │                                                          │
│           ▼                                                          │
│  EXPLANATION                                                         │
│  ├─ 3-tier generation: Quick Glance → Summary → Detailed             │
│  ├─ Natural language from structured evidence                        │
│  ├─ Missing signals surfaced with reasons                            │
│  ├─ Actionable recommendations per confidence level                  │
│  └─ Produces: AssessmentResult {level, summary, detail, exportData}  │
│           │                                                          │
│           ▼                                                          │
│  UI DISPLAY                                                          │
│  ├─ Confidence level badge + visual indicator                        │
│  ├─ Evidence cards (per-signal breakdown)                            │
│  ├─ Conflict warnings (if any)                                       │
│  ├─ Missing/inconclusive signal list                                 │
│  ├─ Actionable recommendations                                      │
│  └─ Export button (PDF/JSON, signed)                                 │
│                                                                      │
│  PERSISTENCE                                                         │
│  ├─ AssessmentResult cached in chrome.storage.session (30 days)      │
│  ├─ Evidence Graph discarded on tab close (re-analysis possible)     │
│  ├─ Exported reports signed and self-contained                       │
│  └─ User override/feedback stored locally                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Analysis Lifecycle

### 4.1 Analysis Session State Machine

```
                    ┌──────────┐
                    │  IDLE    │
                    └────┬─────┘
                         │ AnalysisRequest received
                         ▼
                    ┌──────────┐
                    │INITIALIZE│──► Content type detection, capability check
                    └────┬─────┘
                         │ Capabilities resolved
                         ▼
                    ┌──────────┐
                    │SCHEDULING│──► Build dependency DAG, create execution groups
                    └────┬─────┘
                         │ First analyzers dispatched
                         ▼
                    ┌──────────┐
                    │RUNNING   │◄──► Evidence arrives asynchronously
                    │          │──► Fusion recomputes after each wave
                    │          │──► Progressive results sent to UI
                    └────┬─────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
    ┌──────────┐  ┌──────────┐  ┌──────────┐
    │ COMPLETE │  │CANCELLED │  │  ERROR   │
    │ All tiers│  │ User nav │  │ Critical │
    │ finished │  │ away or  │  │ failure  │
    │          │  │ explicit │  │ (all     │
    │          │  │ cancel   │  │ tiers    │
    │          │  │          │  │ failed)  │
    └────┬─────┘  └────┬─────┘  └────┬─────┘
         │             │             │
         │  Assessment │  Partial    │  Structured
         │  finalized  │  result     │  error with
         │  + saved    │  saved      │  diagnostics
         │             │             │
         ▼             ▼             ▼
    ┌──────────────────────────────────────┐
    │            COMPLETED                  │
    │  Final AssessmentResult persisted    │
    │  Session eligible for garbage collect │
    └──────────────────────────────────────┘
```

### 4.2 Session Lifecycle Events

| Event | Trigger | Action |
|-------|---------|--------|
| `session:created` | First AnalysisRequest for tabId | Create Session object, initialize Evidence Graph, start session timer |
| `session:evidence_added` | Scheduler dispatches evidence | Append Evidence node to Graph, update edges, trigger Fusion |
| `session:fusion_updated` | Fusion Engine recomputes | Update AssessmentResult, publish Progress event to UI |
| `session:tier_complete` | All analyzers in a tier finished | Dispatch next tier, send progressive result to UI |
| `session:all_tiers_complete` | All tiers dispatched and finished | Finalize AssessmentResult, fire AssessmentCompleted |
| `session:cancelled` | User navigates away, closes tab, or explicit cancel | AbortController.abort() on all in-flight analyzers. Save partial result |
| `session:errored` | Critical unrecoverable failure | Save error diagnostics. Notify user |
| `session:expired` | Session timer exceeded max duration (30 min) | Force-complete with available evidence. Mark as timed-out |
| `session:garbage_collected` | Tab closed + retention period elapsed | Remove session from AnalysisSessionManager. Clear Evidence Graph from memory |

---

## 5. Browser Messaging Architecture

### 5.1 Message Types

Communication between browser contexts uses structured messages. All messages are typed, versioned, and validated.

```
┌─────────────────────────────────────────────────────────────────┐
│                     MESSAGE TYPE HIERARCHY                       │
│                                                                  │
│  ANALYSIS MESSAGES                                                │
│  ├─ AnalyzeRequest {tabId, snapshot, invocation, privacyTier}    │
│  ├─ AnalyzeCancel {sessionId, reason}                            │
│  └─ AnalyzeProgress {sessionId, stage, completed, total}         │
│                                                                  │
│  EVIDENCE MESSAGES                                               │
│  ├─ EvidenceProduced {sessionId, evidence: Evidence}             │
│  ├─ EvidenceFailed {sessionId, analyzerId, error: EvidenceError} │
│  └─ EvidenceRequest {sessionId, contentKey}                      │
│                                                                  │
│  COMPUTE MESSAGES (SW ↔ Sandbox)                                 │
│  ├─ ComputeRequest {analyzerId, content, parameters}             │
│  ├─ ComputeResult {evidence: Evidence}                           │
│  ├─ ComputeError {error: EvidenceError}                          │
│  └─ ComputeCancel {analyzerId}                                   │
│                                                                  │
│  UI MESSAGES (SW → Popup/SidePanel)                              │
│  ├─ AssessmentUpdate {sessionId, result: AssessmentResult}       │
│  ├─ AssessmentCompleted {sessionId, final: AssessmentResult}     │
│  ├─ ProgressUpdate {sessionId, completed, total, statusText}     │
│  └─ ErrorNotification {sessionId, message, severity}             │
│                                                                  │
│  USER ACTION MESSAGES (Popup → SW)                               │
│  ├─ UserOverride {sessionId, overriddenLevel, reason}            │
│  ├─ UserReport {sessionId, isCorrect, detail}                    │
│  ├─ ExportRequest {sessionId, format: PDF|JSON}                  │
│  └─ SettingsUpdate {setting, value}                              │
│                                                                  │
│  LIFECYCLE MESSAGES                                              │
│  ├─ TabActivated {tabId}                                         │
│  ├─ TabClosed {tabId}                                            │
│  ├─ TabNavigated {tabId, newUrl}                                 │
│  └─ ExtensionUnloaded {}                                         │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Communication Channels

| Channel | Technology | Between | Characteristics |
|---------|-----------|---------|----------------|
| **CS → SW** | `chrome.runtime.sendMessage` | Content Script → Service Worker | One-shot request/response. ContentSnapshot can be large (serialize carefully). |
| **SW → CS** | `chrome.tabs.sendMessage` | Service Worker → Content Script | One-shot. Used for re-extraction requests, context menu triggers. |
| **SW → Popup** | `chrome.runtime.sendMessage` | Service Worker → Popup | Popup must be open. Assessment updates are batched. |
| **SW → SidePanel** | `chrome.runtime.sendMessage` | Service Worker → SidePanel | SidePanel persists while open. Progressive updates streamed. |
| **SW ↔ Sandbox** | `MessagePort` via `chrome.runtime.connect` | Service Worker ↔ Sandbox Page | Persistent channel for multi-request compute sessions. Sandbox cannot use chrome APIs — only MessagePort. |
| **SW ↔ Offscreen** | `chrome.runtime.sendMessage` | Service Worker ↔ Offscreen Document | Offscreen does not persist across SW restarts. Re-establish channel on SW wake. |
| **Popup → SW** | `chrome.runtime.sendMessage` | Popup → Service Worker | User actions: override, report, export, settings. |
| **Periodic** | `chrome.alarms` | Service Worker → SW self-wake | Reference data sync, model update checks. Min 30s interval (Chrome), 1s (Firefox). |

### 5.3 SW Keep-Alive Strategy

Service workers terminate after 30 seconds of inactivity. To keep the SW alive during long analysis chains:

1. **Long-lived port:** Open a `chrome.runtime.connect` channel from popup/sidePanel to SW. This keeps the SW alive as long as the port is open.
2. **Periodic API calls:** If no UI is connected, the SW calls `chrome.runtime.getPlatformInfo()` every 25 seconds during active analysis. This resets the 30s idle timer.
3. **chrome.alarms fallback:** If the alarm fires and the SW was terminated, the SW restarts and checks for interrupted sessions. It may resume from checkpoint or mark sessions as timed-out.
4. **Checkpoint & Resume:** For analysis exceeding 5 minutes (SW hard limit), the scheduler checkpoints state to `chrome.storage.session` before the 5-minute mark, allows SW to restart, and resumes from checkpoint.

---

## 6. State Transitions

### 6.1 AnalysisSession State

```typescript
type SessionState =
  | 'IDLE'           // No active analysis for this tab
  | 'INITIALIZING'   // Content type detection, capability check
  | 'SCHEDULING'     // Building dependency DAG, creating execution groups
  | 'RUNNING'        // Analyzers executing, evidence arriving
  | 'FUSION_PENDING' // New evidence arrived, fusion queued (debounced)
  | 'FUSION_ACTIVE'  // Fusion Engine computing
  | 'COMPLETED'      // All tiers finished, assessment finalized
  | 'CANCELLED'      // User cancelled or navigated away
  | 'ERROR'          // Critical unrecoverable failure
  | 'TIMED_OUT'      // Session exceeded max duration
```

### 6.2 Analyzer Execution State

```typescript
type AnalyzerState =
  | 'PENDING'    // In queue, not yet dispatched
  | 'SCHEDULED'  // Assigned to execution slot, awaiting context availability
  | 'RUNNING'    // Actively executing
  | 'COMPLETED'  // Produced valid Evidence
  | 'FAILED'     // Produced EvidenceError
  | 'TIMED_OUT'  // Exceeded latency budget
  | 'CANCELLED'  // Cancelled before completion
  | 'SKIPPED'    // Not eligible (browser capability, content type, privacy tier)
```

### 6.3 Evidence Verification State

```typescript
type VerificationState =
  | 'UNVERIFIED'         // Raw evidence, not yet validated against schema
  | 'VALIDATED'          // Schema validation passed
  | 'REJECTED'           // Schema validation failed; reason recorded
  | 'CRYPTO_VERIFIED'    // Cryptographic signature/chains verified (for provenance)
  | 'CRYPTO_FAILED'      // Cryptographic verification failed
  | 'SELF_CERTIFIED'     // Source self-reports (reputation, community) — unverifiable
  | 'EXPIRED'            // Evidence exceeded its TTL without re-validation
```

### 6.4 Consent State Machine

```
     ┌─────────┐
     │ UNKNOWN │──► Initial state, no consent decision made
     └────┬────┘
          │ User shown consent prompt
          ▼
     ┌─────────┐
     │ PENDING │──► User has been prompted but hasn't responded
     └────┬────┘
          │
    ┌─────┼─────┐
    │     │     │
    ▼     ▼     ▼
┌───────┐ ┌───────┐ ┌──────────┐
│GRANTED│ │DENIED │ │WITHDRAWN │──► User previously granted, now revoked
└───┬───┘ └───┬───┘ └─────┬────┘
    │         │            │
    │         │            │ Purge all evidence requiring this consent
    │         │            │ Remove from cloud processing eligibility
    │         │            │
    └────┬────┘            │
         │                 │
         ▼                 ▼
    ┌────────────────────────────┐
    │ Consent decision recorded  │
    │ in chrome.storage.local    │
    │ Consent state persists     │
    │ across sessions            │
    └────────────────────────────┘
```

---

## 7. Storage Architecture

### 7.1 Storage Backend Allocation

| Storage Backend | Capacity | Persistence | Content |
|-----------------|----------|-------------|---------|
| **chrome.storage.session** | ~10 MB, in-memory | Cleared on SW termination, extension reload, browser restart | Active analysis session state, in-flight Evidence objects, FusionEngine intermediate results, current AssessmentResult, loaded model references |
| **chrome.storage.local** | ~10 MB (expandable with `unlimitedStorage`) | Until extension uninstall or user clear | User preferences, consent decisions, domain allow/block lists, user override history, cached domain reputation (hashed), extension settings, persona profile selection |
| **IndexedDB** (via offscreen doc proxy) | % of available disk | Survives cache clear. Lost on extension uninstall | Large ONNX model weights (5-150 MB), reference data bundles (signed, verified), C2PA WASM compiled module, calibration datasets, periodic hash lists |
| **chrome.storage.managed** | Policy-defined | Enterprise admin controlled | Enterprise fleet settings, mandatory policies, domain analysis requirements |

### 7.2 Encryption Strategy

`chrome.storage.local` is NOT encrypted at rest. Sensitive data requires application-layer encryption:

| Data Type | Encrypted? | Key Source |
|-----------|-----------|------------|
| User preferences | No | N/A — non-sensitive |
| Domain allow/block lists | No | N/A — user-managed, not sensitive |
| User override history | **Yes** | Key derived from extension ID + installation timestamp |
| Analysis session state | **No** (session storage, memory-only) | Ephemeral by nature |
| Cached evidence | **Yes** (if persisted beyond session) | Ephemeral session key + content hash |
| Reference data bundles | No (but verified via signature) | Bundle signature verified on load |

### 7.3 Cache Strategy

| Cache Type | Location | TTL | Invalidation |
|-----------|----------|-----|-------------|
| Evidence objects (in-session) | chrome.storage.session | Duration of analysis session | Discarded on session garbage collect |
| Assessment results | chrome.storage.session | 30 days or user clear | User-triggered clear, age-based expiry |
| Domain reputation | chrome.storage.local | 7 days | Stale data deprecated; refreshed on next sync |
| C2PA certificate cache | chrome.storage.session | Browser session | Re-verified on each analysis |
| ONNX model weights | IndexedDB | Until model version update | Version hash comparison triggers re-download |
| Reference data bundles | IndexedDB | Per-data-type freshness SLA | Signed bundle atomic swap; corrupt → rollback |
| User preferences | chrome.storage.local | Indefinite | User modification |

---

## 8. Context Responsibilities

### 8.1 Content Script

**Runs in:** Every tab matching host permissions (isolated world)
**Access:** DOM (read-only, with host permission), `chrome.runtime.sendMessage`, limited Web APIs
**Cannot:** Use `chrome.storage`, access other tabs, run WASM (without `wasm-unsafe-eval`)

**Responsibilities:**
- **Synchronous (<50ms on page load):**
  - Detect main content region using HTML5 landmarks
  - Extract page URL and domain
  - Classify page type (article, social media, gallery, video)
  - Count eligible content elements (images, text blocks, videos)
  - Check against user's domain allow/block list
- **Asynchronous (post-load):**
  - Extract DOM text from main content region
  - Enumerate `<img>`, `<video>`, `<audio>` elements with src attributes
  - Extract same-origin image metadata (EXIF/IPTC/XMP) via `fetch()` + binary parsing
  - Detect social media platform containers
  - Observe dynamic content via MutationObserver (debounced)
- **Never:**
  - Access password/credit card fields
  - Read content from other tabs
  - Access incognito mode content (unless explicitly enabled)
  - Send content directly to cloud (all cloud requests go through SW with consent check)
  - Load or execute models (delegated to sandbox)

### 8.2 Service Worker

**Runs in:** Extension background (no DOM)
**Access:** All `chrome.*` APIs (except those requiring active tab or DOM), `chrome.storage`, `chrome.alarms`, IndexedDB (limited), `fetch`
**Cannot:** Access DOM, use `window`, persist indefinitely (30s idle timeout)

**Responsibilities:**
- **Message routing:** Central dispatch for all inter-context communication
- **Session management:** Create/destroy AnalysisSessions per tab
- **Scheduling:** Orchestrate all evidence gathering
- **Capability detection:** Determine browser capabilities at startup
- **Storage management:** Read/write chrome.storage, proxy IndexedDB through offscreen doc
- **Fusion Engine:** Run Bayesian accumulation, conflict detection, confidence classification
- **Progress reporting:** Publish assessment updates to UI
- **Background tasks:** Reference data sync, model update checks (via chrome.alarms)
- **Keep-alive:** Maintain long-lived port to popup/sidePanel during analysis

### 8.3 Sandbox Page

**Runs in:** Isolated page with relaxed CSP (`sandbox` manifest key)
**Access:** WASM, WebAssembly, Web Workers, WebGPU, limited `chrome.runtime` (MessagePort only), `postMessage`
**Cannot:** Access `chrome.*` APIs (no storage, no tabs, no network), DOM access only to own sandbox page

**Responsibilities:**
- **Heavy computation:** Image forensics (ELA, noise analysis, frequency domain), text perplexity via ONNX, audio analysis, face-swap detection
- **WASM execution:** C2PA verification (`c2pa-js`), ONNX Runtime Web inference
- **Model hosting:** Load ONNX models from IndexedDB (via transfer from offscreen doc)
- **Communication:** Receive ComputeRequest via MessagePort, return ComputeResult/ComputeError

### 8.4 Offscreen Document

**Runs in:** Hidden page with full DOM access (Chrome/Edge only, one at a time)
**Access:** `chrome.runtime` APIs (except a few), DOM APIs, IndexedDB, `fetch`, `AudioContext`, Web Workers
**Cannot:** Access tab content directly, use `chrome.tabs`, persist across SW restarts

**Responsibilities:**
- **Model storage proxy:** IndexedDB access for large model weights and reference data (service workers cannot access IndexedDB directly)
- **tabCapture consumer:** Receive MediaStream from tabCapture, extract frames/audio for analysis
- **Multi-modal computation:** CLIP-ViT for image-text consistency, video frame batching
- **Fallback for Firefox:** On Firefox (no offscreen docs), sandbox page handles what it can; remaining work goes to cloud (opt-in) or is skipped with explanation

### 8.5 Popup

**Runs in:** Extension popup (short-lived, closes on blur)
**Access:** Full `chrome.runtime` APIs, DOM for popup UI
**Lifetime:** Destroyed when popup loses focus

**Responsibilities:**
- **On-demand analysis trigger:** "Analyze this page" button
- **Quick Glance display:** Most recent assessment for current tab
- **Summary View:** Expandable evidence breakdown
- **User actions:** Override, report, export, adjust settings
- **Connection maintenance:** Open long-lived port to SW to keep it alive during analysis

### 8.6 Side Panel

**Runs in:** Persistent side panel (Chrome/Edge only, stays open)
**Access:** Same as popup but persists across tab switches
**Lifetime:** Until user closes side panel

**Responsibilities:**
- **Persistent analysis display:** Shows and updates as user browses (per-domain auto mode)
- **Detailed Report:** Full evidence chain, methodology, export options
- **Batch view:** Multiple assessments for power users
- **Settings:** Advanced configuration, persona profile selection

---

## 9. Explainability Engine Architecture

### 9.1 Three-Tier Generation Pipeline

```
┌────────────────────┐
│  AssessmentResult  │  ← From Trust Engine (8 dimensions, confidence level,
│                    │      evidence graph, conflicts, missing signals)
└────────┬───────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│                 EXPLAINABILITY GENERATOR                     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ TIER 1: QUICK GLANCE (1-3 second scanning target)     │   │
│  │ ┌─────────────────────────────────────────────────┐  │   │
│  │ │ • Confidence level badge (discrete, 6 levels)   │  │   │
│  │ │ • Direction indicator (◄──→)                    │  │   │
│  │ │ • Top 1-2 evidence points (natural language)    │  │   │
│  │ │ • Highest-severity conflict (if any)            │  │   │
│  │ │ • Privacy mode indicator                        │  │   │
│  │ │ • Timestamp ("2 min ago")                       │  │   │
│  │ └─────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                   │
│                          ▼                                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ TIER 2: SUMMARY VIEW (10-30 second reading target)    │   │
│  │ ┌─────────────────────────────────────────────────┐  │   │
│  │ │ • Confidence level with credible interval       │  │   │
│  │ │ • Per-signal breakdown (every evidence source)  │  │   │
│  │ │ • Signal agreement summary                      │  │   │
│  │ │ • Missing signals with reasons                  │  │   │
│  │ │ • Conflict details + possible explanations      │  │   │
│  │ │ • Key findings (natural language paragraphs)    │  │   │
│  │ │ • Actionable recommendations                    │  │   │
│  │ │ • Browser capability impact summary             │  │   │
│  │ │ • "Expand for detailed report" link             │  │   │
│  │ └─────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                   │
│                          ▼                                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ TIER 3: DETAILED REPORT (2-5 minute reading target)   │   │
│  │ ┌─────────────────────────────────────────────────┐  │   │
│  │ │ • Full evidence chain (every signal, source,    │  │   │
│  │ │   version, confidence, contribution weight)     │  │   │
│  │ │ • Evidence Graph visualization (DAG)            │  │   │
│  │ │ • Fusion methodology notes                      │  │   │
│  │ │ • Signal independence assessment                │  │   │
│  │ │ • Calibration history (model versions used)     │  │   │
│  │ │ • 8-dimension confidence breakdown              │  │   │
│  │ │ • Browser capability profile at time of analysis│  │   │
│  │ │ • Chain of custody data                         │  │   │
│  │ │ • Export as signed PDF / JSON                   │  │   │
│  │ │ • Cryptographic verification signature          │  │   │
│  │ └─────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 Natural Language Generation Rules

The Explanation Generator produces natural language from structured evidence using templates, not free-form LLM output (deterministic, reproducible):

| Evidence Pattern | Template |
|-----------------|----------|
| Valid C2PA + recognized CA | "Valid content credentials from [issuer] confirm this [content_type] was [created/captured] by [creator] on [date] using [tool]." |
| Valid C2PA + unknown CA | "Content credentials are present but from an unrecognized certificate authority. The signer claims to be [issuer]. Treat provenance as unconfirmed." |
| Failed C2PA verification | "Content credentials were found but failed cryptographic verification. This may indicate tampered or corrupted provenance data." |
| No provenance found | "No content credentials or provenance metadata were found for this [content_type]. Most web content lacks provenance data — this does not indicate inauthenticity." |
| Frequency artifacts detected | "Pixel-level analysis detected patterns consistent with [AI generator name] generation. These patterns are statistical, not definitive." |
| Cross-modal inconsistency | "The [modality_A] and [modality_B] in this content are inconsistent: [specific inconsistency]. This may indicate AI synthesis or post-generation editing." |
| Conflicting evidence | "Evidence is conflicting: [signal_A] suggests [conclusion_A], while [signal_B] suggests [conclusion_B]. Possible explanations: [list]. Additional verification is recommended." |
| Insufficient evidence | "Sentinel could not gather enough evidence to assess this content. [X] signals were unavailable because [reasons]. Try enabling [features] or re-analyzing." |

### 9.3 Per-Persona Output Adaptation

| Persona | Default Tier | Evidence Format | Additional |
|---------|-------------|-----------------|------------|
| Journalist | Summary View | Citable evidence with source attribution | Export-ready for editorial notes |
| Educator | Summary View | Natural language, suitable for academic committees | Student communication templates |
| Content Marketer | Quick Glance (batch) | Per-article flag with AI % threshold | Integration with CMS workflow |
| Consumer (future) | Quick Glance | Simplest format: badge + 1-line summary | Optional expand to Summary |

---

## 10. Browser Capability Layer

### 10.1 Capability Provider Architecture

The Trust Engine must never contain browser-specific logic. Instead, a `CapabilityProvider` abstracts all browser differences behind a capability query interface.

```
┌──────────────────────────────────────────────────────────────────┐
│                     CAPABILITY PROVIDER                           │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                 Capability Detection (startup)                │ │
│  │  Phase 1: API Presence (<1ms)                                │ │
│  │  • Check chrome.tabs, chrome.offscreen, chrome.tabCapture,  │ │
│  │    chrome.sidePanel, chrome.storage.session,                │ │
│  │    chrome.userScripts (Firefox), navigator.gpu (WebGPU)     │ │
│  │  • Static feature detection: typeof chrome.tabCapture !==   │ │
│  │    'undefined'                                              │ │
│  │                                                              │ │
│  │  Phase 2: Runtime Probing (deferred)                         │ │
│  │  • WebGPU: navigator.gpu.requestAdapter() → test            │ │
│  │  • WASM threading: attempt SharedArrayBuffer allocation     │ │
│  │  • AudioContext: attempt construction with sample rate      │ │
│  │  • Chrome AI APIs: check ai.languageModel availability       │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              BrowserCapabilityProfile (immutable)             │ │
│  │                                                               │ │
│  │  browser:           'chrome' | 'firefox' | 'safari' | 'edge' │ │
│  │  browserVersion:    '140.0.0.0'                              │ │
│  │                                                               │ │
│  │  // Compute capabilities                                      │ │
│  │  supportsWASM:              true (all browsers)               │ │
│  │  supportsWebGPU:            true | false                      │ │
│  │  supportsONNX:              true (WASM backend)               │ │
│  │  supportsONNXWebGPU:        depends on WebGPU                 │ │
│  │  supportsTabCapture:        true | false (Chrome/Edge only)   │ │
│  │  supportsOffscreen:         true | false (Chrome/Edge only)   │ │
│  │  supportsSidePanel:         true | false (Chrome/Edge only)   │ │
│  │  supportsSessionStorage:    true | false (Chrome/Edge only)   │ │
│  │  supportsChromeAI:          true | false (Chrome exp. only)   │ │
│  │  supportsWASMThreading:     true | false                      │ │
│  │  supportsSharedArrayBuffer: true | false                      │ │
│  │                                                               │ │
│  │  // Context capabilities                                       │ │
│  │  maxOffscreenDocuments:     1 (Chrome/Edge) | 0 (others)     │ │
│  │  serviceWorkerLifetime:     30s idle | EventPage (Firefox)   │ │
│  │  swKeepAliveStrategy:       'port' | 'alarm' | 'eventPage'   │ │
│  │  alarmsIntervalMinimum:     30s (Chrome) | 1s (Firefox)      │ │
│  │                                                               │ │
│  │  // Storage capabilities                                      │ │
│  │  storageLocalQuota:         10MB | limitedUnlimitedStorage    │ │
│  │  storageSessionAvailable:   true | false                      │ │
│  │  indexedDBAvailableInSW:    false (Chrome) | limited (FF)    │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                   Capability Query API                        │ │
│  │                                                               │ │
│  │  canRunAnalyzer(analyzerId): boolean                          │ │
│  │  getEligibleAnalyzers(): AnalyzerId[]                          │ │
│  │  getConfidenceCeiling(browser): ConfidenceLevel               │ │
│  │  getMissingCapabilities(analyzerId): Capability[]              │ │
│  │  getRecommendedTier(browser): number (1-5)                    │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### 10.2 Capability → Evidence Mapping

The Scheduler uses the CapabilityProvider to determine which analyzers are eligible:

| Capability | Required By | Missed Evidence When Unavailable |
|-----------|-------------|--------------------------------|
| supportsTabCapture | Video Analyzer, Audio Analyzer (full) | Video frame analysis, audio tab capture |
| supportsWebGPU | All ONNX analyzers (GPU acceleration) | Slower inference (WASM fallback), >2x slower |
| supportsOffscreen | Heavy compute orchestration | No offscreen doc; sandbox page used instead |
| supportsChromeAI | Text Analyzer (Gemini Nano path) | Fallback to WASM ONNX text analysis |
| supportsSessionStorage | Session state persistence | Cannot persist analysis across SW restarts; must checkpoint to local storage |
| supportsSidePanel | Persistent assessment display | Popup-only UI; loses context on blur |
| supportsWASMThreading | Multi-threaded analysis | Single-threaded only; slower |
| supportsSharedArrayBuffer | Real-time audio/video processing | Slower postMessage-based transfer |

---

## 11. Extension Architecture Diagram

### 11.1 Complete Context Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                               BROWSER PROCESS                                │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                        WEB PAGE (TAB)                                   │ │
│  │  ┌────────────────────────────────────────────────────────────────────┐│ │
│  │  │                   CONTENT SCRIPT (Isolated World)                    ││ │
│  │  │                                                                     ││ │
│  │  │  ┌───────────────┐  ┌──────────────┐  ┌──────────────────────────┐ ││ │
│  │  │  │ Content        │  │ Metadata      │  │ Invocation               │ ││ │
│  │  │  │ Discovery      │  │ Extractor     │  │ Handler                  │ ││ │
│  │  │  │ • Landmark     │  │ • EXIF        │  │ • Right-click context     │ ││ │
│  │  │  │ • Social media │  │ • IPTC        │  │ • Popup-triggered         │ ││ │
│  │  │  │ • Text blocks  │  │ • XMP         │  │ • Per-domain auto         │ ││ │
│  │  │  │ • Image enum   │  │ • ICC profile │  │ • ContentSnapshot builder │ ││ │
│  │  │  │ • Video/audio  │  │ • C2PA locate │  │                           │ ││ │
│  │  │  └───────┬───────┘  └──────┬───────┘  └────────────┬──────────────┘ ││ │
│  │  │          └─────────────────┼────────────────────────┘                ││ │
│  │  │                            │ ContentSnapshot                          ││ │
│  │  └────────────────────────────┼─────────────────────────────────────────┘│ │
│  └───────────────────────────────┼──────────────────────────────────────────┘ │
│                                  │ chrome.runtime.sendMessage                │
│                                  ▼                                           │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         SERVICE WORKER                                   │ │
│  │                                                                         │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │                     MESSAGE ROUTER                                │   │ │
│  │  │  Routes by message.type to correct handler                        │   │ │
│  │  └───┬──────────┬──────────┬──────────┬──────────┬─────────────────┘   │ │
│  │      │          │          │          │          │                      │ │
│  │      ▼          ▼          ▼          ▼          ▼                      │ │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────────────┐     │ │
│  │  │Schedule│ │Capabil │ │Session │ │Storage │ │ Progress/Result  │     │ │
│  │  │r       │ │Provider│ │Manager │ │Manager │ │ Bus              │     │ │
│  │  └───┬────┘ └────────┘ └───┬────┘ └───┬────┘ └────────┬─────────┘     │ │
│  │      │                     │          │              │                │ │
│  │      │    ┌────────────────┘          │              │                │ │
│  │      │    │                           │              │                │ │
│  │      ▼    ▼                           │              │                │ │
│  │  ┌──────────────────────────┐        │              │                │ │
│  │  │   ANALYSIS SESSION        │        │              │                │ │
│  │  │  ┌────────────────────┐  │        │              │                │ │
│  │  │  │ Pipeline Orchestr. │  │        │              │                │ │
│  │  │  └────────┬───────────┘  │        │              │                │ │
│  │  │           │              │        │              │                │ │
│  │  │           ▼              │        │              │                │ │
│  │  │  ┌────────────────────┐  │        │              │                │ │
│  │  │  │ Evidence Graph     │  │        │              │                │ │
│  │  │  │ (append-only DAG)  │◄─┼────────┼──────────────┼── Evidence     │ │
│  │  │  └────────┬───────────┘  │        │              │   objects      │ │
│  │  │           │              │        │              │   arrive       │ │
│  │  │           ▼              │        │              │   asynchronously│ │
│  │  │  ┌────────────────────┐  │        │              │                │ │
│  │  │  │ Fusion Engine      │  │        │              │                │ │
│  │  │  │ • Dependency Res.  │  │        │              │                │ │
│  │  │  │ • Bayesian Accum.  │  │        │              │                │ │
│  │  │  │ • Conflict Detect  │  │        │              │                │ │
│  │  │  │ • Safety Override  │  │        │              │                │ │
│  │  │  │ • Confidence Class │  │        │              │                │ │
│  │  │  └────────┬───────────┘  │        │              │                │ │
│  │  │           │              │        │              │                │ │
│  │  │           ▼              │        │              │                │ │
│  │  │  ┌────────────────────┐  │        │              │                │ │
│  │  │  │ Trust Engine (core)│  │        │              │                │ │
│  │  │  │ • Dimension Comp.  │  │        │              │                │ │
│  │  │  │ • Level Mapper     │  │        │              │                │ │
│  │  │  │ • AssessmentResult │──┼────────┼──────────────┼──► UI          │ │
│  │  │  └────────────────────┘  │        │              │                │ │
│  │  └──────────────────────────┘        │              │                │ │
│  │                                      │              │                │ │
│  │  ┌──────────────────────────────────┐│              │                │ │
│  │  │ Background Tasks (chrome.alarms)  ││              │                │ │
│  │  │ • Reference data sync            ││              │                │ │
│  │  │ • Model update check             ││              │                │ │
│  │  │ • Calibration telemetry          ││              │                │ │
│  │  └──────────────────────────────────┘│              │                │ │
│  └──────────────────────────────────────┼──────────────┼────────────────┘ │
│                                         │              │                   │
│  ┌──────────────────────────────────────┼──────────────┼────────────────┐ │
│  │                    UI CONTEXTS       │              │                │ │
│  │  ┌─────────────────┐  ┌─────────────┼──────────────┼──────────────┐ │ │
│  │  │ POPUP           │  │ SIDE PANEL  │              │              │ │ │
│  │  │ • Analysis trig │  │ • Persist   │              │              │ │ │
│  │  │ • Quick Glance  │  │ • Summary   │              │              │ │ │
│  │  │ • User actions  │  │ • Detailed  │              │              │ │ │
│  │  │ • Export        │  │ • Batch     │              │              │ │ │
│  │  │ • Settings      │  │ • Settings  │              │              │ │ │
│  │  └────────┬────────┘  └──────┬──────┘              │              │ │ │
│  │           └──────────────────┼─────────────────────┘              │ │ │
│  │                              │ AssessmentUpdate messages           │ │ │
│  └──────────────────────────────┼────────────────────────────────────┘ │ │
│                                 │                                        │ │
│  ┌──────────────────────────────┼────────────────────────────────────┐ │ │
│  │                    COMPUTE CONTEXTS                                │ │ │
│  │                              │                                     │ │ │
│  │  ┌───────────────────────────┼──────────────────────────────┐     │ │ │
│  │  │ SANDBOX PAGE              │                               │     │ │ │
│  │  │ • C2PA verification (c2pa-js WASM)                        │     │ │ │
│  │  │ • Image forensics (ONNX: DIRE, NPR, noise, ELA)           │     │ │ │
│  │  │ • Text analysis (ONNX: perplexity)                        │     │ │ │
│  │  │ • Watermark detection (DALL-E, Midjourney)                │     │ │ │
│  │  │ • Audio analysis (ONNX: anti-spoofing)                    │     │ │ │
│  │  │ Communication: MessagePort only                           │     │ │ │
│  │  └───────────────────────────────────────────────────────────┘     │ │ │
│  │                                                                     │ │ │
│  │  ┌───────────────────────────────────────────────────────────┐     │ │ │
│  │  │ OFFSCREEN DOCUMENT (Chrome/Edge only, one at a time)       │     │ │ │
│  │  │ • tabCapture consumer (video frame extraction, audio)      │     │ │ │
│  │  │ • IndexedDB proxy (model weights, reference data)          │     │ │ │
│  │  │ • Multi-modal models (CLIP-ViT, cross-modal)               │     │ │ │
│  │  │ • Large batch processing                                   │     │ │ │
│  │  │ Communication: chrome.runtime.sendMessage                  │     │ │ │
│  │  └───────────────────────────────────────────────────────────┘     │ │ │
│  └────────────────────────────────────────────────────────────────────┘ │ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 11.2 Cross-Browser Context Strategy

| Context | Chrome | Edge | Firefox | Safari |
|---------|--------|------|---------|--------|
| **Content Script** | Isolated world | Isolated world | Xray Vision | Isolated world |
| **Service Worker** | 30s idle, 5min max | Same as Chrome | Event Pages (persist-ish) | Background script (native wrapper) |
| **Sandbox Page** | `sandbox` manifest | Same as Chrome | `sandbox` manifest | `sandbox` manifest |
| **Offscreen Document** | `chrome.offscreen` | Same as Chrome | **N/A** (use sandbox) | **N/A** (use sandbox) |
| **Popup** | `action.onClicked` | Same | `browserAction.onClicked` | `browserAction.onClicked` |
| **Side Panel** | `chrome.sidePanel` | Same | `sidebarAction` | **N/A** |
| **Periodic tasks** | `chrome.alarms` (30s min) | Same | `browser.alarms` (1s min) | **N/A** (limited) |

---

## 12. ADR Recommendations

The following Architecture Decision Records should be created during Phase 1 architecture. Each ADR resolves a specific architectural decision with context, options considered, decision, tradeoffs, and consequences.

### Priority 1 (Must resolve before any implementation)

| ADR # | Title | Key Decision |
|-------|-------|-------------|
| **ADR-001** | Extension Platform & Browser Context Strategy | Manifest V3 target. Service worker for orchestration, sandbox pages for ML compute, offscreen docs for Chrome/Edge video/audio. Content scripts for DOM extraction only. |
| **ADR-002** | Evidence Object Model | 19-field immutable Evidence interface. All analyzers emit this format. Structural typing. Serialization via structuredClone. |
| **ADR-003** | Analyzer Plugin API | Capability-based plugin registration. Static manifest (built-in) + dynamic registration (future). Sandboxed execution tiers. Model signing and verification. |
| **ADR-004** | Fusion Engine Architecture | Bayesian evidence accumulation as primary. Evidence Graph (append-only DAG). Conflict detection as separate layer. Safety overrides as config data, not hardcoded rules. |
| **ADR-005** | Confidence Model | 8 independent dimensions (Confidence, Trust, Certainty, Coverage, Quality, Browser Completeness, Offline Completeness, Evidence Completeness). Mapped to 6 discrete levels. |
| **ADR-006** | Scheduler Design | Tiered execution (5 latency classes). Dependency-based DAG scheduling. Resource budgets per invocation model. SW keep-alive via long-lived port + alarms. |

### Priority 2 (Must resolve before Phase 2 MVP)

| ADR # | Title | Key Decision |
|-------|-------|-------------|
| **ADR-007** | Storage Strategy | chrome.storage.session for ephemeral analysis state. chrome.storage.local for preferences. IndexedDB (via offscreen proxy) for models and reference data. Application-layer encryption for sensitive cached evidence. |
| **ADR-008** | Browser Capability Layer | CapabilityProvider abstraction. Two-phase detection (static API presence + runtime probing). Analyzer eligibility determined by capability query, not browser name. |
| **ADR-009** | Cloud Privacy Architecture | Three-tier evidence classification (Local-Only / Anonymized Query / Content Transmission). Explicit opt-in per tier. Content processed and immediately discarded. Domain hashing for Tier 2 queries (SHA-256 truncated). |
| **ADR-010** | Model Loading & Caching | Lazy loading post-install. Version-hash verification before load. LRU eviction weighted by signal tier. Chrome AI APIs as zero-overhead alternative when available. |
| **ADR-011** | Invocation Model | Unified AnalysisRequest object. Five invocation paths sharing same pipeline. activeTab for on-demand. Per-domain allowlist for automatic. Context menu for single-element. |
| **ADR-012** | Explainability Engine | Three-tier generation (Quick Glance / Summary / Detailed). Template-based natural language (deterministic). Per-persona output adaptation. Exportable signed reports. |

### Priority 3 (Must resolve before Phase 3 Platform)

| ADR # | Title | Key Decision |
|-------|-------|-------------|
| **ADR-013** | Dynamic Plugin Sandboxing | Capability Broker for user-installed plugins. Trust tier system (Built-in → Verified Dynamic → User-installed). Explcit capability grants. |
| **ADR-014** | Enterprise Deployment Model | Fleet management via chrome.storage.managed. Admin policy controls. Tenant isolation. Audit log architecture. |
| **ADR-015** | Cross-Device Sync | If user accounts added (Phase 3): sync scope (preferences only? history?), encryption model, conflict resolution. |
| **ADR-016** | API Architecture | Public Trust Engine API design. Rate limiting, authentication, conformal prediction wrapper. SDK versioning strategy. |

---

## Appendix A: Architecture Decision Summary

| Decision | Rationale | Tradeoff |
|----------|-----------|----------|
| Bayesian fusion over linear weighting | Mathematically honest, handles missing signals naturally, explainable decomposition | Requires signal calibration; independence assumption must be enforced |
| Evidence Graph (DAG) over simple list | Encodes relationships, enables conflict detection, supports dependency reasoning | More complex than flat list; graph recomputation cost |
| Discrete confidence levels over continuous percentages | Anti-inversion, anti-liability, user-comprehensible | Less granular; may frustrate power users who want precision |
| Template-based explanations over LLM-generated | Deterministic, reproducible, auditable | Less flexible; may sound mechanical |
| Capability query over browser-name branching | Extensible to new browsers, no hardcoded lists | Requires comprehensive capability detection at startup |
| chrome.storage.session for analysis state | Ephemeral by design — data naturally expires. No encryption needed. | Lost on SW termination; requires checkpoint for long analyses |
| Offscreen doc for IndexedDB proxy | SW cannot access IndexedDB directly in Chrome | One-at-a-time constraint; adds complexity for model loading |

---

## Appendix B: Architecture Principles Satisfied

| Principle | How Satisfied |
|-----------|--------------|
| **SOLID** | Single Responsibility: each subsystem has one clear role. Open/Closed: plugin architecture enables extension without modification. Interface Segregation: narrow interfaces (Analyzer, EvidenceSource, PipelineStage). Dependency Inversion: high-level modules depend on abstractions (Evidence, FusionResult). |
| **Clean Architecture** | Domain logic (Fusion Engine, Confidence Model) depends on nothing. Infrastructure (Storage, Messaging) depends on domain. UI depends on domain via AssessmentResult. |
| **Ports & Adapters** | Evidence Pipeline, CapabilityProvider, Storage Manager are ports. Specific browser APIs are adapters. |
| **Dependency Injection** | Scheduler receives analyzer list and capability profile. Fusion Engine receives configurable safety overrides. Trust Engine receives dimension weights. |
| **Event-driven** | Progress & Result Bus decouples computation from display. Evidence arrival triggers fusion recomputation. Session events trigger lifecycle management. |
| **Immutable Evidence** | Evidence objects frozen after construction. Evidence Graph is append-only. AssessmentResult is versioned, not mutated. |
| **Plugin-first** | Every analyzer is a plugin. Built-in analyzers use the same interface as future user-installed plugins. |
| **Browser abstraction** | CapabilityProvider shields all other subsystems from browser-specific logic. |
| **Testability** | Fusion Engine is pure function (EvidenceGraph → FusionResult). Confidence Model is pure mapping. Scheduler can be tested with mock analyzers. |
| **Deterministic** | Same evidence graph → same confidence assessment. Template-based explanations. Capability detection is reproducible. |
| **Explainable** | Every confidence level traces to specific evidence signals. Three-tier explanation system. User-visible signal contributions. |
| **Privacy by design** | Three-tier evidence classification. Local-first default. Explicit opt-in for cloud. Content never stored. |

---

*End of Trust Engine Architecture — Master Integration Document. All subsystem architectures are cross-referenced. Proceed to ADR drafting upon stakeholder review.*
