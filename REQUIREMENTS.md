# REQUIREMENTS.md — Sentinel Functional & Non-Functional Requirements

> **Project:** Sentinel | **Phase:** Architecture Complete
> **Version:** 1.0

---

## Functional Requirements

### FR-1: Browser Extension

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1.1 | Extension shall load on Chrome, Firefox, and Edge (Manifest V3) | P0 |
| FR-1.2 | Extension shall load on Safari (reduced capability tier) | P2 |
| FR-1.3 | Extension shall support on-demand analysis ("Analyze this page") via popup trigger | P0 |
| FR-1.4 | Extension shall support per-domain automatic analysis (user-configured allowlist) | P1 |
| FR-1.5 | Extension shall support context menu analysis (right-click image → "Analyze with Sentinel") | P1 |
| FR-1.6 | Extension shall use `activeTab` permission for on-demand mode | P0 |
| FR-1.7 | Extension shall not require `<all_urls>` permission for core functionality | P0 |

### FR-2: Trust Engine

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-2.1 | Trust Engine shall produce confidence assessments in 6 discrete levels | P0 |
| FR-2.2 | Trust Engine shall produce explainable output citing specific evidence sources | P0 |
| FR-2.3 | Trust Engine shall never output binary "AI" or "Human" verdicts | P0 |
| FR-2.4 | Trust Engine shall never output 0% or 100% confidence | P0 |
| FR-2.5 | Trust Engine shall combine multiple independent evidence signals using Bayesian fusion | P0 |
| FR-2.6 | Trust Engine shall detect and surface conflicting evidence explicitly | P0 |
| FR-2.7 | Trust Engine shall apply safety overrides (C2PA floor, known synthetic hash floor) | P0 |
| FR-2.8 | Trust Engine shall enforce evidence diversity requirements (minimum signal count, class diversity) | P0 |
| FR-2.9 | Trust Engine shall cap single-signal weight at 40% and single-class weight at 60% | P0 |
| FR-2.10 | Trust Engine shall support progressive result delivery (fast signals first, refined later) | P0 |

### FR-3: Evidence Gathering

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-3.1 | System shall verify C2PA/Content Credentials manifests on same-origin and cooperative cross-origin images | P0 |
| FR-3.2 | System shall extract EXIF, IPTC, and XMP metadata from accessible images | P0 |
| FR-3.3 | System shall extract DOM text from main content regions | P0 |
| FR-3.4 | System shall run image forensics (ELA, noise analysis, frequency domain) on same-origin images | P1 |
| FR-3.5 | System shall run AI generator fingerprint detection on same-origin images | P1 |
| FR-3.6 | System shall analyze text for statistical patterns (burstiness, perplexity) as weak corroborating signal | P1 |
| FR-3.7 | System shall check domain reputation from local cached database | P1 |
| FR-3.8 | System shall support cloud-assisted evidence (reverse image search, fact-check, community signals) with explicit opt-in | P2 |
| FR-3.9 | System shall detect and classify page content types (article, social media, gallery) | P0 |
| FR-3.10 | System shall exclude navigation, ads, comments, and UI chrome from analysis | P0 |

### FR-4: Analysis Capabilities

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-4.1 | System shall analyze text content | P0 |
| FR-4.2 | System shall analyze image content | P0 |
| FR-4.3 | System shall analyze video content (Chrome/Edge only, user-invoked) | P2 |
| FR-4.4 | System shall analyze audio content (Chrome/Edge, user-invoked) | P2 |
| FR-4.5 | System shall perform cross-modal consistency checks when multiple modalities are present | P2 |
| FR-4.6 | System shall support re-analysis of previously assessed content | P1 |

### FR-5: User Interface

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-5.1 | UI shall display confidence level badge (6 discrete levels) | P0 |
| FR-5.2 | UI shall display Quick Glance view (1-3 second scanning target) | P0 |
| FR-5.3 | UI shall display Summary View with per-signal evidence breakdown | P0 |
| FR-5.4 | UI shall support Detailed Report view with full evidence chain | P1 |
| FR-5.5 | UI shall display conflict warnings when evidence signals disagree | P0 |
| FR-5.6 | UI shall display missing evidence with reasons for absence | P0 |
| FR-5.7 | UI shall display actionable recommendations based on confidence level | P0 |
| FR-5.8 | UI shall support export of signed assessment reports (PDF, JSON) | P1 |
| FR-5.9 | UI shall support user feedback (override, report incorrect assessment) | P1 |
| FR-5.10 | UI shall display privacy mode indicator (local-only vs cloud-assisted) | P0 |

### FR-6: Privacy Controls

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-6.1 | System shall process all Tier 1 evidence locally with zero data exfiltration | P0 |
| FR-6.2 | System shall require explicit opt-in for any cloud-assisted evidence (Tier 3) | P0 |
| FR-6.3 | System shall provide single-toggle to disable all cloud features | P0 |
| FR-6.4 | System shall allow per-domain allow/block list for analysis | P1 |
| FR-6.5 | System shall allow per-feature consent (image analysis, cloud features, telemetry) | P1 |
| FR-6.6 | System shall never store analyzed content | P0 |
| FR-6.7 | System shall never store full URLs | P0 |
| FR-6.8 | System shall never access passwords, form data, cookies, or browsing history | P0 |

---

## Non-Functional Requirements

### NFR-1: Performance

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-1.1 | Synchronous page load impact | < 50ms |
| NFR-1.2 | First evidence delivered to UI | < 500ms |
| NFR-1.3 | C2PA verification latency (warm) | < 500ms |
| NFR-1.4 | ONNX inference latency (512×512 image, WASM) | < 2s |
| NFR-1.5 | Progressive assessment refinement | < 10s to comprehensive |
| NFR-1.6 | Peak memory during analysis | < 300MB |
| NFR-1.7 | Extension install size (without models) | < 5MB |
| NFR-1.8 | Model download size (post-install) | < 200MB total |

### NFR-2: Reliability

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-2.1 | SW survival during 60s analysis | 100% of trials |
| NFR-2.2 | Cross-context message integrity | 100% (no corruption) |
| NFR-2.3 | Graceful degradation on failure | Every failure produces structured Evidence, not exception |
| NFR-2.4 | Offline operation | Fully functional with reduced confidence (local evidence only) |

### NFR-3: Security

| ID | Requirement |
|----|-------------|
| NFR-3.1 | All provenance evidence must be cryptographically verified before acceptance |
| NFR-3.2 | Model weights must be signed and verified before loading |
| NFR-3.3 | Reference data bundles must be signed and verified |
| NFR-3.4 | No secrets, API keys, or credentials in source code |
| NFR-3.5 | Extension must not trust content from the page being analyzed |
| NFR-3.6 | Confidence assessments must be cryptographically signed when exported |

### NFR-4: Browser Compatibility

| ID | Requirement |
|----|-------------|
| NFR-4.1 | Full capability on Chrome (all evidence types) |
| NFR-4.2 | Full capability on Edge (all evidence types) |
| NFR-4.3 | Reduced capability on Firefox (no tabCapture, no offscreen doc) |
| NFR-4.4 | Degraded capability on Safari (no tabCapture, no offscreen, slower WASM) |
| NFR-4.5 | Capability degradation must be communicated to user |

### NFR-5: Offline Operation

| ID | Requirement |
|----|-------------|
| NFR-5.1 | All local analyzers must function without network access |
| NFR-5.2 | Cached reference data must be used when online refresh unavailable |
| NFR-5.3 | Offline confidence ceiling must be communicated to user |
| NFR-5.4 | Re-analysis prompt must be offered when connectivity restored |

### NFR-6: Accessibility

| ID | Requirement |
|----|-------------|
| NFR-6.1 | Confidence levels must be communicated with both color AND symbol/text (colorblind accessible) |
| NFR-6.2 | All UI elements must have ARIA labels |
| NFR-6.3 | Keyboard navigation must be supported in popup and side panel |
| NFR-6.4 | Evidence text must be screen-reader accessible |

---

## Out of Scope (Phase 2 MVP)

- Content filtering (hide/blur/warn) — deferred to Phase 3+
- Consumer passive monitoring — deferred to Phase 3+
- User accounts and cloud sync — deferred to Phase 3
- Mobile applications — deferred to Phase 4
- Enterprise dashboard — deferred to Phase 3
- Public API — deferred to Phase 3
- Community trust graph — deferred to Phase 4

---

*End of REQUIREMENTS.md. These requirements are derived from the TRUST_ENGINE_SPEC.md product reasoning specification and the Phase 0 Discovery Report.*
