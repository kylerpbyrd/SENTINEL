# Sentinel Phase 0 — Discovery Report

> **Project:** Sentinel | **Phase:** 0 — Discovery & Research
> **Date:** July 2026
> **Owner:** Byrd Ventures LLC
> **Author:** Lead Architect (consolidated from 8 specialized research agents)
> **Status:** Draft — Awaiting Stakeholder Review

---

## Executive Summary

Phase 0 research confirms that Sentinel's core thesis is **directionally correct but overly broad for Phase 2 execution.** The market for content authenticity verification is real, growing, and underserved at the professional level. However, the consumer market is aspirational and unvalidated. The recommended path: **narrow the MVP to serve journalists and educators first, validate with revenue, then expand toward the infrastructure vision.**

**Key findings:**

| Dimension | Verdict |
|-----------|---------|
| **Product viability** | Viable, but narrower than project plan assumes |
| **Technical feasibility** | Feasible with caveats — text detection is fundamentally unreliable; provenance verification is solid |
| **Commercial viability** | Strong for B2B; unvalidated for consumer |
| **Legal viability** | Viable IF local-only, explainability-focused, and legally counseled |
| **Competitive position** | "Trust Layer" is unclaimed territory; explainability is a genuine moat |
| **Biggest risk** | Defamation liability + Chrome Web Store rejection risk |

**Bottom line:** GO — with constraints. Build for professionals first. Defer consumer features. Engage legal counsel before public launch.

---

## 1. Research Consolidation

### 1.1 Market & Competitor Landscape

**Source:** Market Research Analyst

- **Competitive landscape is a red ocean** for binary AI detection (GPTZero: 17M users, Originality.ai: enterprise partnerships, Turnitin: 16K+ institutions, Copyleaks: 30M+ monthly scans). All compete on accuracy claims. All output binary/percentage scores without explainability.
- **No competitor combines** AI detection + provenance verification + source credibility + fact corroboration into a single browser experience. This is Sentinel's gap.
- **"Trust Layer" is unclaimed territory.** Every competitor positions as an "AI detector" — a commoditizing category.
- **Consumer provenance verification is an open field.** C2PA Content Credentials infrastructure exists (500+ member companies) but no consumer browser extension surfaces them during browsing.
- **Pricing benchmarks:** AI detectors charge $13-180/mo. Sentinel's "trust infrastructure" positioning supports higher enterprise pricing ($50K-500K+/yr).
- **Unanswered questions:** C2PA adoption rate on web content (determines provenance verification usefulness at launch); whether Google is building native Chrome trust features (existential risk).

### 1.2 Browser Extension Technical Capabilities

**Source:** Browser Extension Engineer

- **Manifest V3 is mandatory.** MV2 fully deprecated. Service workers replace persistent background pages — 30-second idle termination, 5-minute max per operation.
- **Local ML inference is feasible** via WebAssembly (ONNX Runtime Web) + WebGPU. Chrome's built-in AI APIs (Gemini Nano on-device) are emerging for text analysis.
- **Hard constraints that cannot be overcome:**
  - Cross-origin image pixel data blocked by CORS (canvas tainting)
  - Closed Shadow DOM inaccessible
  - Cannot read network response bodies in MV3 (declarativeNetRequest is header-only)
  - No Chrome Android extension support
  - No persistent background processing
  - Cannot run on chrome://, about:, or store pages
- **Recommended architecture:** Content scripts extract DOM/metadata locally; sandboxed pages run WASM models; cloud fallback for heavyweight analysis (opt-in only).
- **Cross-browser strategy:** Chrome/Edge (Chromium, near-identical), Firefox (Event Pages instead of service workers, better for background work), Safari (severely constrained, requires native app wrapper).
- **Key risk:** Chrome Web Store rejection for broad host permissions is a potential showstopper.

### 1.3 AI Detection Technical Feasibility

**Source:** ML Researcher

- **Text detection is fundamentally unreliable.** Sadasivan et al. (2023) proved mathematically that as LLMs improve, AUROC of even the best detector approaches 0.5 (random). Real-world accuracy: 39.5% baseline, drops to 17.4% with adversarial paraphrasing. False positives disproportionately affect non-native English speakers (up to 70% FPR).
- **Image detection is the most viable modality** (75-90% accuracy on known generators, drops on unseen/post-processed images). DIRE-based artifact detection + C2PA verification is the strongest combination.
- **Video detection:** Deepfake face-swaps detectable at 80-90% (temporal consistency + physiological signals). Fully AI-generated video (Sora, Runway) is essentially undetectable with useful accuracy.
- **Audio detection:** Viable for raw source audio (80-90%), unreliable after telephony compression.
- **The evidence pyramid (strongest to weakest):**
  - **Tier 1 (primary):** Provenance metadata (C2PA), cross-modal consistency violations, known synthetic signatures
  - **Tier 2 (corroborating):** Image frequency analysis, audio vocoder artifacts, text burstiness analysis, invisible watermark scanning
  - **Tier 3 (weak, aggregate only):** Text perplexity classification, video AI-generation detection, telephony audio detection
  - **Tier 4 (do not use):** Binary AI/human text classifiers as primary signal, short text detection, non-native English classification
- **Critical finding:** Sentinel's multi-signal evidence fusion approach is technically correct. The explainability differentiator is backed by research. No single signal is reliable; combining weak signals is the only honest path.

### 1.4 Content Provenance Standards

**Source:** Content Provenance Researcher

- **C2PA v2.4** is the leading open provenance standard. Industry backing: Adobe, Microsoft, Google, OpenAI, Meta, Sony, BBC, Intel. Verifiable entirely locally via `c2pa-js` WASM library (MIT license) — aligns perfectly with Sentinel's privacy-first architecture.
- **Adoption is growing but still early.** Adobe Photoshop/Lightroom, Leica/Sony cameras, DALL-E, Gemini all implement C2PA. But most social platforms strip metadata on upload, consumer awareness is near-zero, and no browser natively displays Content Credentials.
- **SynthID (Google)** covers all modalities but is proprietary — not independently verifiable without Google's cooperation. Creates a walled garden. Sentinel cannot fully integrate it.
- **Major gaps:** No good text provenance standard (C2PA crJSON is new, minimal adoption). Social media strips metadata. Screenshots destroy provenance chains. "Air gap" (analog hole) is unsolved.
- **Sentinel's opportunity:** Multi-standard aggregation (C2PA + IPTC + EXIF + detection signals in one view). The "no provenance" signal — flagging absence as meaningful context. First browser extension to surface Content Credentials during browsing.

### 1.5 Security, Privacy & Legal

**Source:** Security & Privacy Architect

- **4 potential showstoppers identified:**
  - **R-C1:** Chrome Web Store rejection for excessive permissions/privacy concerns — cannot reach 75%+ users
  - **R-C2:** Defamation liability from incorrect trust labeling — Section 230 likely provides no protection. This could bankrupt Byrd Ventures.
  - **R-C3:** GDPR non-compliance from premature cloud features — fines up to €20M or 4% global revenue
  - **R-C4:** Platform removal for "misleading claims" about AI detection capabilities
- **Local-only MVP eliminates most privacy and regulatory risk.** GDPR, CCPA, and COPPA essentially don't apply if no data leaves the device. Chrome Web Store and Mozilla AMO are far more likely to approve a local-only extension.
- **Critical legal recommendation:** Never make factual statements ("This is AI-generated"). Always provide confidence assessments with explainability. "Confidence scores, not verdicts." Words like "verified," "certified," "authenticated" create legal liability.
- **Top priority action:** Engage legal counsel for defamation liability analysis and terms of service before public launch.
- **EU AI Act:** Sentinel likely NOT classified as high-risk. Content authenticity verification not in Annex III categories. But transparency obligations apply.

### 1.6 Enterprise & Business Model

**Source:** Business Strategy Analyst

- **Recommended model:** Open-core + usage-based SaaS + enterprise platform. (Sentry/PostHog playbook.)
- **Pricing:** Free browser extension (local analysis) → Pro $9-15/mo (cloud-assisted, all modalities) → Team $25-50/user/mo → Enterprise custom $50K-500K+/yr.
- **Open-core boundary:** Open-source: detection orchestration framework, browser extension core, C2PA verification library, signal format, SDKs, CLI tools (MIT/Apache 2.0). Proprietary: detection models, trust graph, multi-modal fusion engine, cloud infrastructure, enterprise dashboard.
- **Primary moat:** Independence from any AI model provider or platform. Google/Meta/OpenAI cannot credibly be a neutral trust layer. Trust graph network effects + C2PA standards leadership create additional defensibility.
- **GTM:** Developers first (they influence org tooling), then enterprise via the Grammarly playbook (extension → Pro → Team → Enterprise).
- **Biggest monetization risk:** If AI detection becomes unsolvable, pivot to cryptographic provenance (C2PA). Detection is one signal, not the product.

### 1.7 Mobile & Platform Constraints

**Source:** Mobile & Platform Architect

- **iOS:** Safari Web Extensions work but are limited to Safari. Share Extensions enable cross-app content analysis (user-initiated). Core ML + ANE enable on-device ML. App Store guidelines ban "virus/malware scanner" positioning — Sentinel must position as "authenticity assessment."
- **Android:** No extension support on Chrome Android. AccessibilityService is restricted — cannot use for content analysis. On-device ML via TFLite + NNAPI + Gemini Nano is strong. Share Intent pattern works for cross-app analysis.
- **Hard constraints on both platforms:** No cross-app content inspection (sandboxing). No background web monitoring on iOS. Cannot analyze DRM/encrypted/E2EE content. App bundle size limits constrain model size.
- **Recommended sequence:** iOS MVP (Share Extension + Camera + QR/NFC) → Android → Desktop companion (Tauri).
- **Mobile-specific opportunities:** QR code/NFC provenance (bridges digital-physical trust), camera-based analysis of real-world content, offline-first operation, Share Sheet integration across apps.

### 1.8 Product Strategy & UX

**Source:** Product Strategist & UX Researcher

- **"This is a B2B company wearing consumer clothing."** The paying customers in Phase 2-3 will be institutions (newsrooms, universities, content agencies), not consumers.
- **Demand is real but concentrated.** Journalists and educators have urgent, recurring needs. Consumer demand is aspirational and unvalidated.
- **The "so what?" gap is the biggest product risk.** Telling users "78% likely AI" without telling them what to DO with that information makes Sentinel a curiosity, not a product. Each persona needs a concrete action path.
- **MVP should be narrow:** Build for one persona (journalist verifying one image) and prove the value chain end-to-end. Defer consumer content filtering (hide/blur/warn), video/audio analysis, and enterprise features.
- **Performance is existential.** > 500ms page load impact or any data exfiltration = instant uninstall. Privacy must be technically verifiable, not just claimed.
- **False positive asymmetry is a design constraint.** For educators, a false positive ruins a student's academic career. For journalists, a false negative ruins publication credibility. Per-persona sensitivity profiles are required.
- **"Boy Who Cried Wolf" problem:** Every false positive erodes trust. Calibrated confidence thresholds, error transparency, and user override mechanisms are critical UX requirements.

---

## 2. Conflict Resolution

### Conflict 1: GTM Strategy — Developers First vs. Professionals First

| Position | Source |
|----------|--------|
| Target developers first (PostHog/Sentry model) | Business Strategy Analyst |
| Target journalists and educators first (B2B content verification) | Product Strategist |

**Resolution:** Both strategies are compatible but the **professional-first path is better validated.** Journalists and educators have urgent, recurring, unfilled needs and existing budgets. Developers influence tooling decisions but don't feel acute pain from content authenticity problems. The browser extension CAN serve both simultaneously — it's the same product. But marketing and onboarding should center on professional use cases (journalist verifying a source image, educator assessing a student paper).

### Conflict 2: Product Scope — Infrastructure vs. Point Tool

| Position | Source |
|----------|--------|
| Sentinel is the "Trust Layer for the Internet" — infrastructure from day one | Project Plan |
| Sentinel should be a tool for professionals first, infrastructure later | Product Strategist |

**Resolution:** The vision is the long-term destination; execution must be pragmatic. **For Phase 0-2, position and build as "a tool that helps professionals verify content authenticity with explainable evidence."** Phase 3+ expands to platform/infrastructure. The architectural decisions should support the long-term vision (APIs, modularity, open-core), but the go-to-market and MVP scope should be narrow.

### Conflict 3: Cloud Features — Monetization vs. Privacy

| Position | Source |
|----------|--------|
| Cloud-assisted analysis needed for Pro tier monetization | Business Strategy Analyst |
| Local-only MVP eliminates most regulatory risk; defer cloud | Security & Privacy Architect |

**Resolution:** **Security/Privacy concern wins for MVP.** Cloud features for monetization should be deferred to Phase 3. The MVP must prove value with local-only processing. This has secondary benefits: faster store approval, stronger privacy positioning, and reduced legal surface area. When cloud features are added, they must be strictly opt-in with clear disclosure. Pro tier can initially monetize on advanced local features (explainability reports, history, custom thresholds, export) rather than cloud processing.

### Conflict 4: Text Detection — Include or Exclude?

| Position | Source |
|----------|--------|
| Text detection is fundamental — users need it most | Product Strategist (persona research) |
| Text detection is mathematically proven to converge to random | ML Researcher |

**Resolution:** **Include text analysis ONLY as a weak corroborating signal in multi-signal fusion. Never ship it as a standalone feature.** The mathematical proof (Sadasivan et al., 2023) is definitive: as models improve, AUROC → 0.5. Text analysis should be clearly labeled in the UI as "least reliable signal" with appropriate caveats. The product should heavily emphasize provenance (C2PA) and image forensics as primary signals. Text classification should never produce a binary verdict or be the sole basis for any confidence score.

### Conflict 5: "AI Detector" vs. "Trust Layer" Marketing

| Position | Source |
|----------|--------|
| Users search for "AI detector" — must use that language for discovery | Market Research Analyst |
| "AI detector" is a shrinking, commoditizing category; avoid at all costs | Product Strategist, Project Philosophy |

**Resolution:** **Market as "Trust Layer" but ensure discoverability for "AI detection" queries.** The explainability differentiator IS the competitive moat. SEO can target "AI detection" keywords while brand positioning uses "trust assessment." The messaging should be: "Not an AI detector. A trust layer that helps you understand what's real." This acknowledges the search behavior while differentiating the product.

---

## 3. Assumption Challenges

### Assumption: "Users want transparency into content authenticity"

**Challenge:** Most users do not actively seek to verify content authenticity. The consumer use case is aspirational, not validated. Professional users (journalists, educators) have this need; consumers have expressed concern but not behavior change.

**Impact:** MVP should target professionals, not consumers. Consumer adoption is a Phase 3+ growth bet, not a Phase 2 assumption.

### Assumption: "The browser extension will become infrastructure"

**Challenge:** Infrastructure companies (Stripe, Twilio, AWS) started with products that solved immediate problems for specific users, then became platforms. Sentinel cannot sell infrastructure on day one.

**Impact:** Phase 2 must prove a clear, narrow value proposition. Infrastructure positioning is for Phase 3+ after product-market fit.

### Assumption: "Users will decide to hide/blur/warn/ignore"

**Challenge:** The people who will actually pay for Sentinel (journalists, educators) don't want to hide content — they want to make decisions with evidence. Content filtering is a consumer feature, and consumer demand is unvalidated.

**Impact:** Defer content filtering to Phase 3+. MVP should focus on evaluation controls (evidence detail, sensitivity thresholds, exportable reports).

### Assumption: "Local processing is sufficient for useful analysis"

**Challenge:** Local-only processing limits model capability. Heavyweight deepfake detection, reverse image search, and real-time threat intelligence require cloud. However, C2PA verification, metadata analysis, lightweight text/image models, and basic forensics all work locally.

**Impact:** The MVP CAN deliver value with local-only processing, but must be transparent about limitations. Advanced analysis requires cloud (Phase 3, opt-in).

### Assumption: "C2PA adoption will provide sufficient signal"

**Challenge:** C2PA adoption is growing but still early. Most web content lacks C2PA manifests. Social platforms strip metadata. Consumer awareness is near-zero.

**Impact:** Sentinel must handle the "no provenance" case gracefully. The absence of provenance IS a signal. Multiple signal sources reduce dependency on any single standard.

---

## 4. Largest Risks (Prioritized)

### Tier 1: Existential Risks (Could Kill the Project)

| # | Risk | Severity | Mitigation |
|---|------|----------|------------|
| **R1** | **Defamation liability from incorrect trust labeling** — false positive could lead to lawsuits bankrupting Byrd Ventures. Section 230 provides weak or no protection since Sentinel creates the trust score. | CRITICAL | Never make factual claims. Always show confidence + evidence + disclaimers. Consult defamation counsel before public launch. Form LLC if not already. |
| **R2** | **Chrome Web Store rejection** for excessive permissions or privacy concerns — cannot reach Chrome users (75%+ market share). | CRITICAL | Use `activeTab` permission. Local-only processing. Clear privacy policy. Justify every permission. Build Firefox/Edge support in parallel. |
| **R3** | **AI detection becomes solvable only by provenance** — Sadasivan et al. proved text detection AUROC → 0.5 as models improve. Image/video detection degrades 10-20% per new model generation. | CRITICAL | Multi-signal architecture is essential. Provenance (C2PA) is the most durable signal. Detection is a supporting signal, not the product. |
| **R4** | **Platform competition** — Google/Apple/Microsoft building trust features natively. | HIGH | Multi-platform, API-first, independence positioning. Google cannot be a neutral trust layer. |

### Tier 2: Major Risks (Could Severely Damage the Project)

| # | Risk | Severity | Mitigation |
|---|------|----------|------------|
| **R5** | **The "so what?" problem** — confidence scores without actionable outcomes make Sentinel a curiosity, not a product. | HIGH | MVP must provide concrete action paths per persona. "Based on this assessment, you can: publish with evidence, open an integrity case, reject content, etc." |
| **R6** | **Consumer demand unproven** — the project plan assumes consumer adoption. Research suggests consumer willingness to pay is near-zero. | HIGH | Build for professionals first. Consumer is a growth bet, not a core assumption. |
| **R7** | **False positive reputational damage** — "Boy Who Cried Wolf" erosion of trust if Sentinel flags too many false positives. | HIGH | Calibrated confidence, user override, error transparency, per-persona sensitivity profiles. |
| **R8** | **GDPR non-compliance from cloud features** — fines and reputational destruction. | HIGH | Local-only MVP. Cloud features deferred to Phase 3 with full GDPR architecture. |
| **R9** | **Browser performance degradation** — > 500ms page load impact causes uninstalls. | HIGH | Progressive result loading. Process in sandboxed pages. Yield to browser rendering. |

### Tier 3: Significant Risks (Require Active Management)

| # | Risk | Mitigation |
|---|------|------------|
| **R10** | Model supply chain compromise | Model signing + hash verification |
| **R11** | Adversarial ML bypass becomes widespread | Ensemble of diverse architectures, continuous adversarial testing |
| **R12** | C2PA adoption slower than projected | Multiple evidence sources; design for missing provenance |
| **R13** | Extension dependency supply chain attack | Minimal dependencies, SRI, audit, lockfile |
| **R14** | Competitor patents block specific methods | Freedom-to-operate search before Phase 2 |
| **R15** | Enterprise sales cycles too long for startup | Start with self-serve Pro/Team revenue; enterprise later |

---

## 5. Missed Opportunities (Not in Original Vision)

1. **C2PA consumer verification as first-mover wedge.** No browser extension currently surfaces Content Credentials during browsing. The `c2pa-js` library (MIT, WASM-based) makes this technically straightforward. This is a low-risk, high-signal entry point that no competitor has claimed.

2. **The "no provenance" signal.** Simply flagging content that lacks provenance data — and explaining what that means — is uniquely valuable context. Competitors treat absence as neutral; Sentinel can treat it as informative.

3. **QR code/NFC provenance for mobile.** Scanning physical provenance credentials (printed C2PA QR codes, NFC-tagged documents) bridges digital and physical trust. No existing tool does this. Mobile MVP differentiator.

4. **Trust graph as data moat.** Domain reputation, content hash registry, source reliability scores — this is defensible network-effect IP that grows more valuable with usage. Plans should start in Phase 2, not Phase 4.

5. **Regulatory compliance verification.** EU AI Act mandates AI content labeling. US executive orders require watermarking. Sentinel could be the verification tool for regulatory compliance — a clear enterprise value proposition with regulatory tailwinds.

6. **Writing/creation process replay.** Originality.ai introduced "writing replay" for text. Sentinel could extend this concept to images (edit history visualization, capture metadata timeline). This provides evidence for false positive defense.

---

## 6. Vision Amendments

### What Should Change

| Aspect | Current Vision | Recommended Amendment | Rationale |
|--------|---------------|----------------------|-----------|
| **MVP target user** | General consumers ("Trust Layer for the Internet") | Journalists and educators (professional content evaluators) | Consumer demand unvalidated; professional demand proven and budgeted |
| **MVP scope** | Multi-modal analysis + content filtering (hide/blur/warn) | Image + text analysis + explainability panel. DEFER filtering. | Narrow scope proves value chain; filtering is consumer-only |
| **Phase 2 positioning** | "Trust Layer for the Internet" (infrastructure) | "Content authenticity verification for professionals" (tool) | Infrastructure is Phase 3+ positioning; tool is Phase 2 reality |
| **Cloud timeline** | Phase 3 (platform with sync, accounts, API) | Phase 3 (opt-in cloud features for Pro tier only) | Local-only MVP eliminates regulatory risk and speeds store approval |
| **Mobile timeline** | Phase 4 | Phase 4 (after extension PMF validated) | Extension must prove concept first; mobile adds complexity without validated demand |
| **Text detection role** | Primary signal alongside image/video | Weak corroborating signal only; never standalone | Mathematically proven to converge to random as models improve |
| **Language about output** | "Trust score" | "Confidence assessment" / "Authenticity analysis" | "Trust" implies guarantee; "confidence" is defensible and accurate |

### What Should Stay

| Aspect | Why |
|--------|-----|
| **Privacy-first, local-first architecture** | Core differentiator. Eliminates regulatory risk. Aligns with target personas (journalists need source protection). |
| **Explainability as core differentiator** | The market is moving toward this (Originality.ai's "AI Allowance" validates). Binary detectors are commoditizing. |
| **Multi-signal evidence fusion** | Backed by research. No single signal is reliable. Combination of weak signals is the only honest approach. |
| **Open-core business model** | Alignment with trust/transparency positioning. Developer adoption pathway. Sentry/PostHog playbook validated. |
| **Platform independence** | Avoid vendor lock-in. Google/Meta/OpenAI cannot be neutral trust providers. Independence is the moat. |
| **Evidence over certainty** | Core philosophy validated by research. Matches legal risk mitigation strategy. |
| **C2PA/Content Credentials integration** | Standards alignment provides ecosystem positioning and future-proofing. Low technical risk, high signal value. |

---

## 7. Recommended Phase 1 Plan

### Phase 1: Architecture & Prototype Validation
**Duration:** Estimated 4-8 weeks
**Goal:** Design the complete system architecture and validate core assumptions with a minimal prototype.

### Phase 1 Deliverables

| # | Deliverable | Priority | Owner |
|---|-------------|----------|-------|
| 1 | **ARCHITECTURE.md** — System architecture, service boundaries, data flow | CRITICAL | Architect |
| 2 | **ADR-001 through ADR-005** — Core architectural decisions (language, framework, ML runtime, extension structure, API design) | CRITICAL | Architect |
| 3 | **VISION.md** — Revised vision with professional-first positioning | HIGH | Product Manager |
| 4 | **REQUIREMENTS.md** — Functional and non-functional requirements for MVP | HIGH | Product Manager |
| 5 | **C2PA Integration Spike** — Prove `c2pa-js` works in a browser extension context with real-world images | HIGH | Browser Extension Engineer |
| 6 | **Image Analysis Spike** — Prove DIRE-based or lightweight image forensics model runs in WASM/ONNX in a sandboxed extension page | HIGH | ML Researcher |
| 7 | **ROADMAP.md** — Prioritized milestones with dependencies and success criteria | HIGH | Product Manager |
| 8 | **BUSINESS_PLAN.md** — Revised with professional-first GTM and phased enterprise strategy | MEDIUM | Business Analyst |
| 9 | **AGENTS.md** — Agent definitions, responsibilities, and coordination rules | MEDIUM | Architect |
| 10 | **PROMPT.md** — Global engineering rules and standards | MEDIUM | Architect |
| 11 | **README.md** — Project overview, mission, getting started | MEDIUM | Documentation Engineer |

### Phase 1 NOT In Scope
- Implementation code (beyond technical spikes)
- UI/UX mockups (beyond wireframes for architecture validation)
- Cloud infrastructure
- Mobile architecture (deferred to post-MVP)
- Video/audio analysis architecture

### Phase 1 Success Criteria
- Architecture reviewed and approved against all research findings
- C2PA verification proven in extension context
- Image forensics model proven runnable in extension sandbox
- All critical ADRs drafted and reviewed
- Documentation set complete and cross-referenced

---

## 8. Documentation Roadmap (Priority Order)

These documents should be created during Phase 1, in this order. Each document should answer a specific question and be cross-referenced rather than duplicating information.

| # | Document | Question It Answers | Depends On |
|---|----------|---------------------|------------|
| 1 | **ARCHITECTURE.md** | How does the system work end-to-end? | All Phase 0 research |
| 2 | **ADR-001: Extension Platform & Architecture** | Manifest V3, service workers, sandbox pages, or offscreen docs? | Browser Extension Research |
| 3 | **ADR-002: ML Runtime & Model Strategy** | ONNX Runtime Web, Chrome AI APIs, or custom WASM? | ML Research |
| 4 | **ADR-003: Trust Engine Architecture** | How do signals combine into confidence scores? | ML + Provenance Research |
| 5 | **ADR-004: Privacy Architecture** | What data stays local, what goes to cloud, when, and how? | Security/Privacy Research |
| 6 | **ADR-005: Open-Core Boundary** | What's open source vs. proprietary? | Business Strategy Research |
| 7 | **VISION.md** | What are we building and why? | All Phase 0 research |
| 8 | **REQUIREMENTS.md** | What must the MVP do? | VISION, ARCHITECTURE |
| 9 | **ROADMAP.md** | When do we build what? | REQUIREMENTS |
| 10 | **BUSINESS_PLAN.md** | How does this become a sustainable company? | Market + Enterprise Research |
| 11 | **AGENTS.md** | Who does what in the AI development team? | Architect |
| 12 | **PROMPT.md** | What standards govern all development? | Architect |
| 13 | **README.md** | What is Sentinel? (For newcomers) | VISION, ARCHITECTURE |
| 14 | **LICENSE** | What are the usage rights? | Open-Core ADR |

---

## 9. Immediate Next Actions

1. **Stakeholder review of this Discovery Report** — Accept, reject, or modify the recommended vision amendments before proceeding.

2. **Engage legal counsel** — Priority questions:
   - Defamation liability exposure for confidence assessments vs. binary labeling
   - Terms of service and disclaimer language
   - Section 230 applicability analysis
   - EU AI Act classification confirmation
   - C2PA integration IP considerations

3. **Validate go-to-market**, specifically: target professional users (journalists/educators) as primary initial market, defer consumer features.

4. **Begin Phase 1 architecture** — Start with C2PA integration spike and image forensics spike to validate the two highest-signal technical assumptions before designing the full architecture.

5. **Set up project infrastructure** — ADR directory (`docs/adr/`), Graphify knowledge graph (if using), CI/CD pipeline planning.

---

## Appendix A: Research Sources

All research consolidated from 8 specialized agent reports, available at:
- `research/SECURITY_PRIVACY_LEGAL_RESEARCH.md`
- `research/product-strategy-and-ux-research.md`
- `research/content-provenance-standards.md`
- `BUSINESS_PLAN.md`

Plus agent responses (Market Research, Browser Extension, ML Research, Mobile Platforms) returned directly to the Architect.

Key external references cited throughout the research include:
- Sadasivan et al. (2023), "Can AI-Generated Text be Reliably Detected?" — TMLR
- Perkins et al. (2024), "GenAI Detection Tools" — IJETHE
- Liang et al. (2024), "GPT detectors are biased against non-native English writers" — Patterns
- C2PA Specification v2.4 — c2pa.org
- Chrome Extensions MV3 Documentation — developer.chrome.com
- EU AI Act (Regulation 2024/1689)
- Originality.ai, GPTZero, Copyleaks, Turnitin — public product pages and studies

---

## Appendix B: Key Terms

| Term | Sentinel Definition |
|------|-------------------|
| **Confidence assessment** | A probabilistic estimate of content authenticity, with explicit evidence and uncertainty quantification. Never a binary verdict. |
| **Trust score** | DEPRECATED. Use "confidence assessment" to avoid implying guarantee. |
| **Evidence signal** | An independent source of information about content authenticity (metadata, provenance, forensic analysis, reputation, cross-modal consistency). |
| **Multi-signal fusion** | The process of combining multiple weak evidence signals into a calibrated confidence assessment. |
| **Explainability** | The presentation of evidence and reasoning behind a confidence assessment, enabling users to understand WHY Sentinel reached its conclusion. |
| **Provenance** | Cryptographic or metadata-based evidence of content origin, creation process, and modification history (C2PA, Content Credentials). |
| **Local-first** | Processing that occurs on the user's device without sending content to external servers. The default operating mode of Sentinel. |

---

*End of Discovery Report. Phase 1 architecture planning begins upon stakeholder approval of vision amendments.*
