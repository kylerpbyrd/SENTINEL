# ROADMAP.md — Sentinel Project Roadmap

> **Project:** Sentinel | **Phase:** Architecture Complete
> **Version:** 1.0

---

## Phase 0: Discovery & Research ✅ COMPLETE

**Status:** Complete

**Deliverables:**
- Market research and competitor analysis
- Browser extension capability assessment
- AI detection feasibility research
- Content provenance standards review (C2PA, SynthID, Content Credentials)
- Security, privacy, and legal risk assessment
- Enterprise and business model research
- Mobile platform feasibility assessment
- Product strategy and UX research
- Consolidated Discovery Report

**Exit Criteria:** Product vision coherent. Technical feasibility evaluated. Risks documented. MVP scope defined.

---

## Phase 1: Architecture ✅ COMPLETE

**Status:** Complete

**Deliverables:**
- TRUST_ENGINE_SPEC.md (product reasoning specification)
- Trust Engine architecture (subsystems, data flow, lifecycles)
- Evidence object model and pipeline
- Fusion engine and confidence model
- Scheduler and analyzer plugin architecture
- Invocation, privacy, performance, and failure architectures
- Explainability engine and browser capability layer
- Browser capability assessment (binding technical constraints)
- 16 Architecture Decision Records (ADRs)
- Validation plan (32 technical spikes, exit criteria)
- Validation harness design

**Exit Criteria:** All architecture documents approved. ADRs accepted. Validation plan defined.

---

## Phase 2: Validation & MVP Implementation

**Status:** Preparing — Pending SPIKE-C0

### Phase 2a: Validation Spikes
- Build Validation Harness
- Execute SPIKE-C0 (End-to-End Architecture Validation)
- Execute 7 Critical spikes
- Execute 7 High spikes
- Execute 7 Medium spikes
- Execute 8 Low spikes
- Architecture Readiness Review (ARR)

**Exit Criteria:** All mandatory exit criteria met. ARR passed. Implementation authorized.

### Phase 2b: MVP Implementation
- Browser extension shell (MV3, TypeScript, all browser contexts)
- Evidence pipeline implementation
- C2PA analyzer plugin (c2pa-js WASM)
- EXIF/IPTC metadata analyzer
- Text statistical analyzer
- Image forensics analyzer (ONNX: noise, frequency domain, ELA)
- Domain reputation analyzer
- Fusion engine implementation (Bayesian accumulator + conflict detection + confidence model)
- Explainability engine (3-tier output)
- Popup UI (Quick Glance + Summary View)
- Side panel UI (Detailed Report + export)
- Privacy controls (consent management, Tier 1-2-3 toggles)

**MVP Scope:** Journalists and educators as primary users. Image + text analysis. Local-first processing. Explainability as core differentiator. No cloud features (deferred to Phase 3).

**Exit Criteria:** Extension passes Chrome Web Store review. Core user journeys functional (journalist verifies image, educator assesses paper). Performance budgets met. Privacy verified (zero data exfiltration in local mode).

---

## Phase 3: Platform & Cloud

- Cloud-assisted evidence (reverse image search, fact-check, community signals, SynthID if API opens)
- User accounts and cloud sync (preferences, history)
- Team/Business tier (dashboard, API access, SSO, audit logs)
- Public API (developer access to Trust Engine)
- Per-domain automatic analysis
- Always-on passive mode (enterprise managed deployments)
- Content filtering (hide/blur/warn) for consumer use case
- Video and audio analysis (tabCapture, Chrome/Edge)
- Enterprise features (private cloud, custom models, SLAs)

**Exit Criteria:** Pro tier revenue validated. API usage growing. Enterprise design partners onboarded.

---

## Phase 4: Expansion

- iOS application (Share Extension, Camera analysis, QR/NFC provenance, Core ML)
- Android application (Share Intent, Camera analysis, TFLite/Gemini Nano)
- Desktop application (Tauri: WebView wrapper + system tray)
- SDK for third-party developers
- Trust Graph (network-effect data asset: domain reputation, content hash registry)
- Developer ecosystem (plugin marketplace, community analyzer plugins)
- Consumer-focused features (passive browsing protection)

**Exit Criteria:** Multi-platform presence. Developer ecosystem active. Trust Graph providing network-effect defensibility.

---

## Milestone Summary

| Phase | Status | Key Decision |
|-------|--------|-------------|
| Phase 0 — Discovery | ✅ Complete | Vision validated. Professional-first MVP. |
| Phase 1 — Architecture | ✅ Complete | Architecture approved. 16 ADRs accepted. |
| Phase 2 — Validation | 🔜 Preparing | Awaiting SPIKE-C0 execution. |
| Phase 3 — Platform | 📋 Planned | Contingent on Phase 2 product-market fit signals. |
| Phase 4 — Expansion | 📋 Planned | Contingent on Phase 3 platform revenue. |

---

*End of ROADMAP.md. This roadmap is reviewed and updated at the completion of each phase.*
