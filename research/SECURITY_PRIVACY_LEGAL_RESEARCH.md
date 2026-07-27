# Sentinel — Security, Privacy & Legal Research (Phase 0)

> **Document Type:** Discovery Research
> **Phase:** 0 — Pre-Architecture
> **Status:** Draft
> **Owner:** Byrd Ventures LLC / Security & Privacy Architect
> **Audience:** Architect Agent, Product Manager, Legal Counsel

---

## Executive Summary

Sentinel faces a unique risk profile: it sits at the intersection of **browser extension security**, **AI regulation**, **content moderation liability**, **GDPR/CCPA compliance**, and **platform store acceptance risk**. This document catalogs 41 discrete risks, identifies 4 potential showstoppers, and provides a compliance checklist across 7 jurisdictions.

**Bottom line:** The project is feasible but must be architected with privacy-by-default, local-first processing, and rigorous legal disclaimers from day one. Cloud processing of user content creates the most significant regulatory and platform-acceptance risk. A fully local inference model eliminates most privacy/legal risk but limits model capability.

---

## 1. Browser Extension Security

### 1.1 Threat Surface

A browser extension that reads page content for authenticity analysis has an enormous attack surface. The extension sits in the rendering pipeline of every page the user visits.

| Risk | Severity | Likelihood | Description |
|------|----------|------------|-------------|
| **R-01: Content Script Injection** | Critical | Medium | If Sentinel's content script is compromised (XSS in extension UI, eval of untrusted content), attacker gains read access to every page the user visits. |
| **R-02: Supply Chain via Dependencies** | High | Medium | Extensions frequently bundle npm packages. A compromised dependency loads arbitrary code in the extension's privileged context. |
| **R-03: Permission Escalation** | High | Low | An attacker who compromises the extension could request additional permissions via `optional_permissions` in manifest. |
| **R-04: Message Spoofing** | High | Low | If content-script-to-background message validation is weak, a malicious page could inject fake analysis results or manipulate the trust score display. |
| **R-05: Storage Exfiltration** | Medium | Medium | `chrome.storage.local` is unencrypted on disk. User preferences, trust decisions, and any cached analysis results could be exfiltrated by malware already on the device. |
| **R-06: Update Channel Hijack** | High | Very Low | If the Chrome Web Store developer account is compromised, an attacker could push a malicious update to all users simultaneously. |
| **R-07: Native Messaging Abuse** | Medium | Low | If Sentinel uses Native Messaging for local ML inference, a malicious native host could read all extension data or inject false results. |

### 1.2 Manifest V3 Security Model

Key constraints and implications:

| MV3 Feature | Security Implication for Sentinel |
|-------------|----------------------------------|
| **Service Workers (no persistent background)** | ML model must be reloaded on each service worker wake-up or stored in `chrome.storage.session`. Large models become impractical. |
| **`declarativeNetRequest` only** | Cannot intercept/modify network requests programmatically. Sentinel cannot block content at the network level using MV3. |
| **No remotely hosted code** | All ML inference code must be bundled in the extension package. No CDN-loaded models at runtime. *This is enforced by the Chrome Web Store.* |
| **CSP enforcement** | Extensions have a stricter CSP by default. `unsafe-eval` is blocked, which may affect ONNX Runtime Web or TensorFlow.js WASM backends. |
| **`sandbox` pages** | MV3 allows sandboxed pages with relaxed CSP for compute-heavy ML work (e.g., ONNX inference). This is the recommended pattern for local model execution. |
| **`externally_connectable`** | If Sentinel exposes APIs to websites, this must be tightly scoped to prevent arbitrary websites from probing the extension. |

### 1.3 Required Security Architecture Decisions

1. **All content analysis must happen in sandboxed pages**, not in content scripts or service workers. This limits blast radius.
2. **Strict input validation** on every message boundary: content script → service worker, service worker → sandbox, sandbox → storage.
3. **No `eval()` or `new Function()`** anywhere in extension code. Chrome Web Store rejects extensions with these patterns.
4. **Subresource Integrity (SRI)** for any bundled third-party code. Pin all dependency hashes.
5. **Minimum viable permissions.** Start with `activeTab` (only inject on pages the user explicitly activates) rather than `<all_urls>`.
6. **Code signing** via the Chrome Web Store and AMO signing processes is mandatory and automatic — but only for the store channel. Self-distributed builds require separate signing.

### 1.4 Secure Storage Strategy

`chrome.storage.local` is NOT encrypted at rest. Any sensitive data stored there is accessible to:
- Other extensions with `storage` permission (Chrome does NOT isolate extension storage from other extensions)
- Malware with filesystem access

**Recommendation:**
- Never store API keys, user identifiers, or authentication tokens in `chrome.storage.local` without encryption
- Use `chrome.storage.session` (memory-only, cleared on browser close) for ephemeral tokens
- For persistent secrets, encrypt with a key derived from the extension's unique ID + user-provided passphrase
- For enterprise deployments, keys should be provisioned via managed storage (`chrome.storage.managed`)

---

## 2. Threat Modeling

### 2.1 Attacker Profiles

| Attacker | Motivation | Capability |
|----------|------------|------------|
| **Disinformation Actor** | Evade detection so AI-generated propaganda appears trustworthy | Medium — can test against the extension, craft adversarial content |
| **Competing Platform** | Discredit Sentinel's accuracy to undermine trust in the product | Medium-High — has resources for adversarial ML research |
| **Nation-State Actor** | Compromise the trust layer to manipulate what content users see as trustworthy | Very High — could compromise supply chain, models, or API endpoints |
| **Malicious Website Operator** | Trick the extension into mislabeling their content as trustworthy | Low-Medium — can craft adversarial page content |
| **Privacy Abuser** | Exfiltrate user browsing history via the extension's data flows | Medium-High — can monitor network traffic if extension sends data to cloud |

### 2.2 Critical Attack Scenarios

**S-01: Adversarial Content Bypass**
An attacker crafts content specifically designed to receive a high trust score from Sentinel. Since the trust engine uses multiple signals (metadata, provenance, ML models), an attacker must defeat ALL signals simultaneously. This is harder than defeating a single model but still possible.

*Risk:* If Sentinel becomes widely adopted, adversaries will invest heavily in bypass research.
*Mitigation:* Ensemble diversity is critical. Models must use fundamentally different architectures and feature sets. Regular adversarial retraining. "Ground truth" provenance signals (C2PA, cryptographic signatures) are substantially harder to forge than ML signals.

**S-02: Trust Layer Poisoning**
An attacker creates a website that appears trustworthy to Sentinel, then later pivots to serving malicious content. If Sentinel caches trust scores by domain, the cached score persists.

*Mitigation:* Trust scores must be time-bound. Domain reputation must decay. No permanent whitelisting.

**S-03: Man-in-the-Middle on Provenance**
If Sentinel verifies C2PA or Content Credentials signatures, a MITM could strip provenance metadata from content in transit.

*Mitigation:* Sentinel should flag content with missing provenance where provenance is expected, rather than treating absence as neutral.

**S-04: Model Supply Chain Attack**
If Sentinel downloads ML models from a cloud endpoint, a compromised endpoint could serve a poisoned model that selectively marks certain content as trustworthy.

*Mitigation:* Models must be signed. The extension must verify model signatures before loading. Model hashes should be pinned in the extension manifest or verified against a transparency log.

**S-05: API Abuse / Credential Harvesting**
If Sentinel offers a cloud API, attackers could:
- Reverse-engineer the API to build competitors
- Flood the API with requests (DDoS)
- Submit adversarial content to probe detection thresholds

*Mitigation:* Rate limiting, API key rotation, request signing, anomaly detection. Consider requiring authentication for cloud features.

### 2.3 Trust Boundary Diagram (Logical)

```
┌─────────────────────────────────────────────────────┐
│                  USER'S DEVICE                       │
│  ┌──────────┐   ┌──────────┐   ┌────────────────┐   │
│  │ Content   │──▶│ Service   │──▶│ Sandboxed      │   │
│  │ Script    │   │ Worker    │   │ ML Runtime     │   │
│  │ (untrusted│   │ (trusted) │   │ (trusted,      │   │
│  │  page ctx)│   │           │   │  isolated)     │   │
│  └──────────┘   └──────────┘   └────────────────┘   │
│        │              │                │             │
│        ▼              ▼                ▼             │
│  ┌──────────────────────────────────────────────┐    │
│  │         chrome.storage (local/session)        │    │
│  │         (NOT encrypted at rest)               │    │
│  └──────────────────────────────────────────────┘    │
│                         │                            │
└─────────────────────────┼────────────────────────────┘
                          │ HTTPS
                          ▼
┌─────────────────────────────────────────────────────┐
│                  SENTINEL CLOUD (if used)             │
│  ┌──────────┐   ┌──────────┐   ┌────────────────┐   │
│  │ API       │──▶│ Trust     │──▶│ Model          │   │
│  │ Gateway   │   │ Engine    │   │ Registry       │   │
│  │ (auth,    │   │           │   │ (signed models)│   │
│  │  rate lim)│   │           │   │                │   │
│  └──────────┘   └──────────┘   └────────────────┘   │
└─────────────────────────────────────────────────────┘
```

Key trust boundaries:
1. **Page ↔ Content Script:** The page is actively hostile. Content script must validate everything.
2. **Extension ↔ Cloud:** If sent, user content is in transit. Must be encrypted. Must not be retained.
3. **Model Registry:** Models must be signed and verified before loading.

---

## 3. Cloud Security (If Applicable)

The project philosophy says "local-first" and "privacy-first." But Phase 3 envisions cloud sync, accounts, and API. If cloud infrastructure is built:

### 3.1 API Security Requirements

- **Authentication:** OAuth 2.0 / OIDC. Never accept API keys in URL parameters.
- **Authorization:** Principle of least privilege. Separate API keys for extension vs. enterprise SDK.
- **Rate Limiting:** Token bucket per API key. Aggressive limits for free tier.
- **Input Validation:** Strict schema validation on all API inputs. Reject oversized payloads. Sanitize all content before processing.
- **CORS:** Restrict to known extension origins. No `Access-Control-Allow-Origin: *`.
- **Audit Logging:** Log all API access. Retain logs for security investigation. Do NOT log user content.

### 3.2 Data Protection

| Data Type | In Transit | At Rest | Retention |
|-----------|-----------|---------|-----------|
| User authentication tokens | TLS 1.3 | Encrypted (AES-256) | Until revoked |
| Analysis request payloads (content) | TLS 1.3 | NEVER STORED | Process and discard |
| Analysis results / trust scores | TLS 1.3 | Encrypted, per-user | 30 days max (user-configurable) |
| Telemetry / metrics | TLS 1.3 | Encrypted, aggregated | 90 days raw, indefinite aggregated |
| User account data (email, etc.) | TLS 1.3 | Encrypted, hashed where possible | Until account deletion |

### 3.3 Multi-Tenant Isolation

If enterprise customers have separate tenants:
- Logical separation at minimum (tenant ID on every record, enforced at query layer)
- Separate encryption keys per tenant
- No shared databases for enterprise-tier customers

---

## 4. Secrets Management

### 4.1 API Key Handling

**Problem:** Browser extensions are effectively public code. Any API key embedded in the extension package is accessible to any user who inspects the extension source.

**DO NOT** embed API keys directly in extension code. This is the #1 mistake in extension development.

**Recommended approach for public API keys:**
- Use a backend proxy. The extension authenticates the user, the backend holds the actual API key.
- If Sentinel provides a public API for third-party developers, those keys are managed by the developers — not embedded in Sentinel's extension.

**For Sentinel's own cloud API:**
- OAuth 2.0 PKCE flow from the extension
- Access tokens stored in `chrome.storage.session` (memory-only)
- Refresh token rotation
- No long-lived API keys in extension storage

### 4.2 Model Signing

All ML models shipped with the extension or downloaded at runtime must be:
1. **Signed** with an offline private key (held by Byrd Ventures)
2. **Verified** by the extension before loading using the corresponding public key
3. **Hashed** — model hash pinned in the extension manifest or verified against a transparency log

This prevents:
- Supply chain attacks on model delivery
- Man-in-the-middle model substitution
- Compromised CDN serving poisoned models

### 4.3 Code Signing

- **Chrome Web Store:** Automatic signing upon publication. CRX files are signed by Google.
- **Mozilla Add-ons (AMO):** Automatic signing upon approval. XPI files are signed by Mozilla.
- **Safari:** Uses Apple's notarization process for Safari Web Extensions.
- **Self-distribution:** Requires separate code signing infrastructure.

---

## 5. Privacy: Data Minimization

### 5.1 Data Classification

| Data Category | Must Process | Should Process | Must NOT Process |
|---------------|-------------|----------------|-------------------|
| Page text content | Analysis | — | — |
| Page images/video | Analysis (if enabled) | — | — |
| Page metadata (C2PA, EXIF) | Verification | — | — |
| Page URL | Signal (domain reputation) | — | — |
| User's full browsing history | — | — | MUST NOT collect |
| User identity / PII | — | — | MUST NOT collect (unless account) |
| Form data / passwords | — | — | MUST NOT access |
| Cookies / session data | — | — | MUST NOT access |
| Geolocation | — | — | MUST NOT access |
| WebRTC / media streams | — | — | MUST NOT access |

### 5.2 Can It Work With Zero Data Leaving the Device?

**For MVP (text analysis, basic image analysis, metadata verification): YES.**

Small BERT-based models, ONNX Runtime Web, and in-browser transformers make on-device inference viable for:
- Text authenticity analysis (perplexity, burstiness, style consistency)
- Metadata extraction and C2PA verification
- Basic image forensics (noise patterns, compression artifacts)

**For advanced capabilities (will require cloud):**
- Large vision-language models for deepfake detection
- Cross-referencing against known AI-generated image databases
- Real-time video analysis
- Complex audio forensics (voice cloning detection)

**The architecture should tier capabilities:**
- **Tier 1 (Local):** Text + metadata + basic image → zero data leaves device
- **Tier 2 (Optional Cloud):** Advanced deepfake detection → user explicitly opts in, content sent to cloud, NEVER retained
- **Tier 3 (Enterprise):** Custom models, private cloud deployment → enterprise controls data

Any cloud feature MUST:
1. Be opt-in, not default
2. Explicitly inform the user what content is being sent
3. Process and discard content immediately — never store it
4. Use TLS 1.3 in transit
5. Be clearly disclosed in the privacy policy and store listing

### 5.3 Browsing Data Exposure Risk

The core risk: **Even if Sentinel doesn't send data to the cloud, the extension has access to all page content on activated tabs.** This creates:

1. **Perception risk:** Users and privacy advocates will question why an extension needs access to page content at all. The Chrome Web Store disclosure will say "This extension can read and change all your data on websites you visit."
2. **Compromise risk:** If the extension is compromised (R-01 through R-07 above), the attacker gets everything the extension can read.
3. **Accidental exfiltration risk:** Bugs in the extension could inadvertently send content to a logging service, analytics, or error reporting endpoint.

**Mitigations:**
- Use `activeTab` permission: only read content on tabs where the user explicitly invokes the extension
- Consider a toggle model: "Analyze this page" button rather than always-on scanning
- If always-on is requested, scope to specific domains or content types
- Never use `<all_urls>` if avoidable; prefer `activeTab` + host permission requests at runtime
- Implement data loss prevention: ensure no page content reaches telemetry/logging systems

---

## 6. Regulatory Compliance

### 6.1 GDPR Analysis

**Key question: Is Byrd Ventures a data controller, processor, or neither?**

| Scenario | Classification | GDPR Impact |
|----------|---------------|-------------|
| All processing on-device, no data leaves device | **Neither** (no personal data processing by Byrd Ventures) | GDPR does not apply to the company. The user processes their own data. |
| Cloud analysis, content sent to Sentinel servers | **Data Controller** (determines purpose and means) | Full GDPR obligations: lawful basis, DPA, data subject rights, breach notification, DPO potentially required |
| Enterprise deployment on customer infrastructure | **Data Processor** (processes on behalf of enterprise controller) | Processor obligations only. DPA with enterprise customer required. |
| Telemetry (anonymous, opt-in) | **Controller** (for telemetry data) | GDPR applies, but anonymized data has reduced obligations |

**Recommendation:** Architect the MVP so that GDPR does not apply to Byrd Ventures. All processing on-device. If cloud features are added later, they must be architected for GDPR compliance from the start.

**If cloud becomes necessary, GDPR requirements include:**
- **Lawful basis:** Consent (most appropriate for optional cloud features) or legitimate interest (harder to justify)
- **Data Processing Agreement (DPA):** Required with any cloud provider
- **Data Protection Impact Assessment (DPIA):** Required before processing. Content authenticity analysis is novel and likely requires a DPIA.
- **Right to Erasure:** Users must be able to delete all their data
- **Right to Access:** Users must be able to export all their data
- **Right to Object:** Users must be able to opt out
- **Data Residency:** EU user data must be processable within the EU or adequate jurisdictions
- **Breach Notification:** 72-hour notification requirement
- **DPO:** May be required depending on scale and nature of processing

### 6.2 CCPA/CPRA (California)

Similar to GDPR but with opt-out model rather than opt-in. If Sentinel collects any personal information from California residents:
- **Notice at Collection:** Disclose what is collected and why
- **Right to Know:** Users can request all collected data
- **Right to Delete:** Users can request deletion
- **Right to Opt-Out:** Of sale/sharing of personal information
- **Sensitive Personal Information:** Browsing history could be classified as sensitive — requires explicit opt-in

**Recommendation:** Same as GDPR — avoid collecting personal information entirely for MVP.

### 6.3 EU AI Act

The EU AI Act (Regulation 2024/1689) entered into force August 2024, with phased applicability through 2027.

**Classification analysis for Sentinel:**

| Use Case | AI Act Classification | Requirements |
|----------|----------------------|-------------|
| AI-generated content detection | **Likely NOT high-risk** — not listed in Annex III high-risk categories | Transparency obligations (Article 50) may apply |
| Trust scoring for general web content | Not high-risk unless used for critical infrastructure, law enforcement, or consequential decisions | Limited requirements |
| Enterprise API for content verification | Depends on use case — if used by regulated entities, they absorb the compliance burden | Provider transparency obligations |
| User-facing content labeling | **Article 50(4):** AI systems that generate or manipulate text/image/audio must disclose. Sentinel is a *detector*, not a *generator* — likely exempt from Article 50(4) | — |
| General Purpose AI (GPAI) if Sentinel deploys large foundation models | If using a GPAI model (e.g., LLaMA, GPT for analysis), the model provider bears Chapter V obligations | Sentinel would have downstream deployer obligations |

**Key takeaway:** Sentinel itself is likely **not classified as high-risk** under the EU AI Act because content authenticity verification does not fall under Annex III categories (critical infrastructure, education, employment, law enforcement, migration, democratic processes). However, Sentinel should:
1. Document the non-high-risk classification rationale
2. Provide transparency about AI use (even if not strictly required)
3. Monitor EU AI Act implementation — the list of high-risk categories can be amended
4. If Sentinel's trust scores are used by third parties for consequential decisions (hiring, lending, law enforcement), those third parties bear the compliance burden — but Sentinel should consider contractual protections

### 6.4 ePrivacy Directive / Cookie Consent

The ePrivacy Directive (EU) and UK PECR govern tracking and cookies. Key implication:

- **If Sentinel sets any cookies or uses local storage for tracking purposes**, consent is required
- **If Sentinel uses `chrome.storage` solely for functionality** (user preferences, cached analysis results), this falls under the "strictly necessary" exemption
- **If Sentinel uses any form of browser fingerprinting** (even for fraud detection), this requires consent under ePrivacy

**Recommendation:**
- Never use cookies or tracking technologies
- `chrome.storage` use should be limited to user preferences and analysis configuration
- Document the "strictly necessary" rationale for any local storage

### 6.5 Children's Privacy (COPPA / UK Age Appropriate Design Code)

- **COPPA (US):** Applies if the extension is directed at children under 13 or knowingly collects data from them
- **UK Age Appropriate Design Code:** Applies if the extension is "likely to be accessed" by children under 18

**Risk:** If Sentinel is available on the Chrome Web Store and Mozilla Add-ons without age restrictions, it is "likely to be accessed" by children. However, if Sentinel collects no personal data, COPPA and the Age Appropriate Design Code have limited application.

**Recommendation:**
- Do not collect personal data → minimizes COPPA exposure
- Do not market toward children
- Include age-appropriate language in privacy policy if cloud features are added
- Consider whether the extension stores could classify the content analysis tool as requiring age restrictions

### 6.6 Emerging Regulations (2025-2026)

| Regulation | Jurisdiction | Status | Relevance |
|------------|-------------|--------|-----------|
| **EU Data Act** | EU | In force Jan 2025 | Governs data sharing from connected products. Low relevance for browser extension. |
| **EU Digital Services Act (DSA)** | EU | In force, enforcement ramping | If Sentinel operates a "platform" (user accounts, content sharing), DSA may apply. Affects transparency reporting, risk assessments. |
| **UK Online Safety Act** | UK | Received Royal Assent 2023 | If Sentinel is a "user-to-user service" or "search service," it has duties regarding illegal content and children's safety. Low risk if Sentinel is purely analytical/on-device. |
| **US Federal Privacy Law (APRA)** | US | Proposed, uncertain | Would create GDPR-like framework in US. Monitor. |
| **US State Privacy Laws** | US | Multiple states active (CT, CO, VA, UT, etc.) | Patchwork compliance burden if collecting personal data. |
| **Canada Bill C-27 / Quebec Law 25** | Canada | Various stages | GDPR-like requirements. Quebec Law 25 active. |
| **Brazil LGPD** | Brazil | Active | GDPR-like. Applies if offering services to Brazilian users. |

### 6.7 Compliance Requirements Checklist

| Requirement | MVP (Local-Only) | With Cloud | Enterprise |
|-------------|------------------|------------|------------|
| GDPR compliance (full) | Not required | Required | Required (processor) |
| CCPA compliance | Not required | Required | Required |
| EU AI Act transparency | Minimal | Required (Article 50) | Required |
| DPIA | Not required | Required | Required |
| DPA with cloud provider | Not required | Required | Required |
| Privacy Policy (CWS/AMO) | Required (store requirement) | Required | Required |
| Cookie/ePrivacy consent | Not required | May be required | May be required |
| COPPA compliance | Not required (no data) | May be required | May be required |
| Data Processing Agreement | Not required | Required | Required (with customers) |
| SOC 2 / ISO 27001 | Not required | Recommended | Expected |
| Data residency controls | Not required | Required for EU | Required |
| Breach notification process | Not required | Required | Required |

---

## 7. Consent & Transparency

### 7.1 Required Consent Mechanisms

**Onboarding consent (pre-first use):**
1. **What Sentinel does:** Clear, non-technical explanation that Sentinel analyzes content to assess authenticity
2. **What Sentinel does NOT do:** Explicitly state that Sentinel does not track browsing history, does not collect personal data, does not modify content
3. **Processing mode selection:**
   - "Analyze when I click" (activeTab only) ← **recommended default**
   - "Analyze automatically" (requires broader permissions)
4. **Cloud features (if offered):** Separate explicit opt-in with clear disclosure of what data leaves the device

**Per-feature consent:**
- Image analysis: opt-in (computationally expensive, higher privacy sensitivity)
- Audio analysis: opt-in
- Cloud deepfake detection: explicit opt-in with content preview
- Telemetry: opt-in (not opt-out)

### 7.2 "Analyze This Page" vs "Always Analyze"

| Mode | Permission Level | Privacy Impact | UX Impact |
|------|-----------------|---------------|-----------|
| **On-demand ("Analyze this page")** | `activeTab` | Minimal — only reads pages user explicitly activates | Extra click per page. Slower. |
| **Per-domain allowlist** | Host permissions at runtime | Medium — reads all pages on allowed domains | Good UX. User controls scope. |
| **Always on** | `<all_urls>` or broad host permissions | High — reads every page | Seamless UX. Privacy risk. Hard to justify to stores. |

**Recommendation:** Start with on-demand + per-domain allowlist. "Always on" should require the user to pass through multiple consent screens with clear warnings. The Chrome Web Store and Mozilla both scrutinize extensions that read all page content.

### 7.3 Privacy Nutrition Labels

**Apple App Store Privacy Labels (for Safari extension):**
If a Safari extension is built:
- Data Used to Track You: Should be "None"
- Data Linked to You: Should be minimal (user ID if account exists)
- Data Not Linked to You: Usage data, diagnostics (if collected)
- Categories: Browsing History → "Not Collected" (or "Collected" with explanation if always-on)

**Mozilla Add-ons Data Disclosure:**
Mozilla requires add-ons to disclose in the privacy policy:
- What data is collected
- How it's used
- Who it's shared with
- How long it's retained
- How users can request deletion

### 7.4 Required Privacy Policy Disclosures

At minimum, the privacy policy must address:

1. **Data collection:** What data is collected (content, URLs, usage stats, etc.)
2. **Processing location:** On-device vs. cloud — clearly differentiated
3. **Data retention:** How long, if at all, data is retained
4. **Third-party sharing:** Who, if anyone, receives data
5. **User rights:** GDPR/CCPA rights if applicable
6. **Children:** Whether the service is intended for children
7. **Contact:** How to reach the data protection officer / privacy contact
8. **Changes:** How users will be notified of policy changes

**The Chrome Web Store and Mozilla AMO will reject extensions without a privacy policy if they collect any user data.** Even a local-only extension should have a privacy policy stating that no data is collected — this builds trust.

---

## 8. Anonymous Telemetry

### 8.1 What Telemetry Is Acceptable?

**Acceptable without violating privacy:**
- Extension version and browser version
- Feature usage counts (e.g., "analyze text clicked 50 times")
- Analysis mode distribution (text vs. image vs. both)
- Error/crash reports (sanitized — no URLs, no content)
- Trust score distribution (aggregated, anonymized)
- Performance metrics (analysis latency, model load time)

**NOT acceptable:**
- URLs of pages analyzed
- Content analyzed (even temporarily)
- User identifiers (even hashed)
- IP addresses (beyond what's technically required for transmission)
- Unique device identifiers
- Browsing patterns or timing data

### 8.2 Differential Privacy Approaches

If aggregate statistics about trust scores are valuable:
- Apply **local differential privacy (ε = 1-5)** before data leaves the device
- Use the **RAPPOR** (Randomized Aggregatable Privacy-Preserving Ordinal Responses) model: noise is added on-device before transmission
- Maintain a **privacy budget** — users with high activity should not be individually distinguishable in aggregates

### 8.3 Opt-in vs Opt-out

| Model | When Appropriate |
|-------|-----------------|
| **Opt-in** | Telemetry, crash reports, cloud features, any data collection |
| **Opt-out** | Nothing — Sentinel should not have opt-out data collection |
| **No choice** | Technical necessities (TLS for HTTPS, etc.) — must be documented |

**Recommendation:** All telemetry should be opt-in. On first install, present a clear choice: "Help improve Sentinel with anonymous usage data" (Yes/No). Respect the choice. Never change it silently.

### 8.4 Store Policy Requirements for Data Collection

Both Chrome Web Store and Mozilla AMO require:
- Disclosure of all data collection in the privacy policy
- Disclosure in the store listing (Chrome: "Privacy practices" section; Mozilla: data disclosure fields)
- Certification of compliance with the store's data use policies
- For Chrome: compliance with the "Limited Use" policy — user data may not be sold or used for unrelated purposes

---

## 9. Liability & Legal Risk

### 9.1 False Positive / False Negative Liability

This is potentially the **highest-severity legal risk** for the project.

**Scenario A: False Positive (content incorrectly labeled as AI-generated / untrustworthy)**
- A journalist's original photography is labeled as "possibly AI-generated"
- A political campaign's authentic video is labeled as "low trust"
- An artist's original work is flagged as "likely synthetic"

**Legal theories for liability:**
- **Defamation per se:** If the labeling implies fakery, fraud, or dishonesty
- **Tortious interference:** If the labeling causes economic harm (lost contracts, de-platforming)
- **Negligence:** If the system was designed without reasonable accuracy safeguards
- **Consumer protection statutes:** Deceptive trade practices claims

**Scenario B: False Negative (AI-generated content incorrectly labeled as trustworthy)**
- User relies on Sentinel's "high trust" score for deepfake video
- User makes financial, political, or personal decision based on Sentinel's assessment
- Sentinel fails to detect sophisticated AI-generated content

**Legal theories:**
- **Negligence / product liability:** If Sentinel was negligent in its detection design
- **Misrepresentation:** If Sentinel's marketing claimed capabilities it couldn't deliver
- **Consumer protection:** If "trust" branding creates unreasonable reliance

### 9.2 Key Defenses

1. **Confidence, not certainty.** Sentinel must NEVER claim to "detect AI" or "verify truth." Every output must be framed as a confidence assessment with clear uncertainty indicators. This is already the project philosophy — it must be rigorously implemented in UX.

2. **Explainability as a legal shield.** If Sentinel says "We're 67% confident this text shows AI patterns because: [perplexity analysis, repetition patterns, style consistency]," that is substantially less legally dangerous than a binary "AI GENERATED" label.

3. **User control as liability limitation.** Users decide what action to take. Sentinel provides information, not decisions. The user remains the decision-maker.

4. **Disclaimers.** Every interface showing trust scores should include or link to disclaimers:
   - "Sentinel provides confidence assessments, not definitive determinations."
   - "AI detection is probabilistic and may produce incorrect results."
   - "Users should not rely solely on Sentinel for important decisions."

5. **No "certification" language.** Words like "verified," "certified," "authenticated," "guaranteed," "confirmed" create legal liability. Avoid them entirely in product and marketing.

### 9.3 Defamation Risk Analysis

| What Sentinel Says | Defamation Risk | Safer Alternative |
|--------------------|----------------|-------------------|
| "This image is AI-generated" | **HIGH** — factual statement that may be false | "Sentinel confidence: 78% that this image shows AI generation patterns" |
| "This content is untrustworthy" | **HIGH** — attacks reputation/character | "Sentinel did not find provenance or authenticity signals for this content" |
| "This website is a fake news site" | **CRITICAL** — direct defamation | "This domain lacks established trust signals. 12% of analyzed content here contained verifiable falsehoods." |
| "This video is a deepfake" | **HIGH** | "Sentinel's analysis found: face manipulation indicators (moderate), voice synthesis indicators (low), metadata anomalies (high). Overall confidence: 82%." |

**Defamation requires:**
1. A false statement of fact
2. Publication to a third party
3. Harm to reputation
4. Fault (negligence or actual malice depending on plaintiff)

Sentinel can avoid defamation by:
- **Never making factual statements** — only providing confidence assessments
- **Never publishing trust scores publicly** — only showing them to the user who installed the extension
- **Providing evidence** — showing WHY a confidence score exists, not just the score
- **Allowing feedback/correction** — a mechanism for content creators to challenge assessments

**Critical protection:** If trust scores are only shown to the individual user and never published publicly, defamation risk is substantially reduced (though not eliminated — the user could screenshot and share).

### 9.4 Section 230 (US)

**Section 230 of the Communications Decency Act** provides immunity for "interactive computer services" against liability for content provided by third parties.

**Does Section 230 apply to Sentinel?**

| Factor | Analysis |
|--------|----------|
| Is Sentinel an "interactive computer service"? | Unclear. A browser extension that analyzes content on the user's device differs from a website hosting third-party content. |
| Does Sentinel "publish" third-party content? | No — Sentinel analyzes and labels, it does not host or publish |
| Is Sentinel's trust score "information provided by another information content provider"? | Sentinel creates the trust score — it is Sentinel's own content, not third-party content |

**Likely conclusion:** Section 230 provides **weak or no protection** for Sentinel's trust assessments. Sentinel creates the trust score; it is not merely hosting someone else's content. This means Sentinel is exposed to defamation/product liability claims in a way that platforms (Twitter, Facebook) are not.

**This is a significant legal risk that requires expert legal counsel.**

### 9.5 Platform Terms of Service

Sentinel analyzes content on third-party platforms. Could those platforms take legal action?

**Platforms' potential claims:**
- **Terms of Service violation:** Most platforms prohibit automated access, scraping, or analysis. However, a browser extension that processes content the user already has lawful access to operates in a legal gray area.
- **Computer Fraud and Abuse Act (CFAA):** If Sentinel accesses a platform beyond what the user is authorized to see, CFAA liability arises. If Sentinel only analyzes content the user is already viewing, this risk is lower.
- **Copyright:** Analyzing content for authenticity does not copy or distribute it. Fair use / implied license arguments apply.
- **Trademark dilution:** Using platform logos in the extension UI could be trademark infringement.

**Recommendation:**
- Analyze content on-device — the user's browser, the user's access rights
- Do not scrape, crawl, or systematically access platforms
- Do not use platform branding or imply platform endorsement
- Consider caching only analysis results, not content itself

### 9.6 Could Chrome Web Store / Mozilla Block the Extension?

**Yes, and this is a showstopper risk.**

| Blocking Ground | Likelihood | Analysis |
|-----------------|------------|----------|
| Privacy violation / excessive data collection | Medium | Depends on permission scope and data handling. `activeTab` + local-only processing minimizes risk. |
| Misleading claims about AI detection | Medium | If Sentinel's marketing or functionality claims are unsubstantiated, stores may remove it. |
| Content censorship / manipulation claims | Medium | If Sentinel is perceived as censoring or manipulating web content, stores may act. |
| Trademark/copyright complaints | Low-Medium | If platforms complain about Sentinel analyzing their content |
| Defamation complaints from content creators | Medium | If influential parties complain to Google/Mozilla about being mislabeled |
| "Be Useful" principle (Chrome) | Low | If Sentinel provides genuine utility, this is not a blocking concern |

**Critical considerations:**
- **Google's Chrome Web Store has notoriously opaque enforcement.** Extensions can be removed with limited explanation and limited appeal rights.
- **Mozilla's review process is more transparent** but also more rigorous — they review source code.
- **Both stores reserve the right to remove extensions at their sole discretion.**
- **Business-critical mitigation:** If Sentinel is removed from stores, users must still be able to use it. Self-distribution (side-loading) must be possible, though it drastically reduces user reach.

---

## 10. Intellectual Property

### 10.1 Training Data Copyright

If Sentinel develops custom ML models for content detection:
- **Training data sources must be legally obtained.** Cannot train on copyrighted content without license or fair use basis.
- **"Fair use" for detection training** is an unsettled area of law. The *Warhol Foundation v. Goldsmith* (2023) decision narrowed transformative use. Training detection models on copyrighted works may not qualify as fair use.
- **Open datasets with known licensing issues** (e.g., LAION, which faced copyright challenges in EU) must be avoided or carefully vetted.
- **Models trained on infringing data** may be "tainted" — EU AI Act requires disclosure of training data. The emerging legal doctrine of "unlawful training" could force model deletion.

**Recommendation:**
- Use only permissively licensed or public domain training data for custom models
- For commercially licensed models (e.g., through Hugging Face, OpenAI, etc.), verify training data compliance
- Document all training data sources and licenses
- Monitor legal developments in training data copyright (multiple cases pending in US and EU courts)

### 10.2 Open Source License Considerations

Sentinel is "open-core." License choice matters critically:

| License | Pros | Cons |
|---------|------|------|
| **AGPL v3** | Strong copyleft. Forces cloud competitors to open-source modifications. | Toxic for enterprise adoption. Many companies have blanket AGPL bans. |
| **GPL v3** | Strong copyleft. Does not trigger on network use. | Still restrictive for proprietary integrations. |
| **Apache 2.0** | Permissive. Enterprise-friendly. Includes patent grant. | Competitors can build proprietary forks. Provides no competitive moat. |
| **MIT** | Simplest. Widest adoption. | No patent protection. No competitive protection. |
| **BSL (Business Source License)** | Eventually becomes open-source. Protects commercial use for a period. | Complex. Requires legal review. Not OSI-approved. |
| **Dual License (AGPL + Commercial)** | Open-core model. AGPL for community, commercial for enterprise. | Requires contributor license agreement (CLA). Administrative overhead. |

**Recommendation:** Dual-license strategy:
- **Core extension code:** Apache 2.0 (maximizes adoption, enterprise-friendly)
- **Detection models:** Proprietary or custom license (core IP)
- **Cloud/API code:** AGPL v3 (prevents competitors from offering Sentinel-as-a-Service without contributing back)
- **Enterprise SDK:** Commercial license

Require a **Contributor License Agreement (CLA)** from all external contributors to enable dual-licensing.

### 10.3 Patent Landscape

The AI detection space has active patenting. Key areas to monitor:

| Patent Area | Risk | Action |
|-------------|------|--------|
| Methods for detecting AI-generated text | Medium | Freedom-to-operate search recommended before commercial launch |
| Deepfake detection algorithms | Medium | Multiple patents exist. Some held by large companies (Microsoft, Google, Adobe) |
| Content provenance verification (C2PA-related) | Low | C2PA is an open standard. Implementation likely non-infringing |
| Trust scoring / reputation systems | Low-Medium | Broader patents exist but likely not specific to AI detection |

**Recommendation:**
- Commission a freedom-to-operate patent search before Phase 2
- File provisional patents on Sentinel's novel ensemble/explainability methods
- Monitor C2PA/CAI patent commitments (Adobe has made IP pledges around Content Authenticity Initiative)

---

## 11. Platform Policies

### 11.1 Chrome Web Store

**Key applicable policies:**

| Policy | Relevance | Compliance Requirements |
|--------|-----------|------------------------|
| **Privacy Policy** | Required if extension handles user data | Public privacy policy linked in store listing |
| **Limited Use** | Data collected must only be used for extension's core functionality | No selling data. No unrelated marketing use. |
| **Use of Permissions** | Extensions must request minimum necessary permissions | Justify each permission in store listing |
| **Disclosure Requirements** | All functionality must be disclosed | No hidden features. No surprise behavior. |
| **Handling Requirements** | Secure handling of user data | Encryption in transit. Data minimization. |
| **Misleading Behavior** | No deceptive claims about capabilities | Claims must be accurate. No "guaranteed detection." |
| **API Use** | Must use APIs as documented | No circumvention of Chrome security features |
| **MV3 Requirements** | Extensions must use Manifest V3 | No remotely hosted code. Service worker background. |
| **Minimum Functionality** | Extension must provide value | Must work as advertised |
| **Spam and Abuse** | No keyword stuffing, fake reviews, etc. | Organic store presence only |

**Chrome Web Store enforcement risk:** Google's review process is partially automated and can be inconsistent. An extension analyzing all page content will face heightened scrutiny. Every permission must be clearly justified in the listing.

### 11.2 Mozilla Add-ons

**Key applicable policies:**

| Policy | Relevance | Compliance Requirements |
|--------|-----------|------------------------|
| **No Surprises** | Functionality must be transparent | Full disclosure of what the extension does and how |
| **Data Collection & Transmission Disclosure** | Must disclose all data collection, including in privacy policy | Even telemetry must be disclosed. Data collection categories must be specified. |
| **Content** | No misleading or deceptive content | Accurate claims. No "guaranteed detection." |
| **Development Practices** | Secure development required | Source code review. No obfuscation without justification. |
| **Security, Compliance & Blocking** | Mozilla can block extensions | Must comply with Mozilla's security requirements |

**Mozilla review advantage:** Human reviewers examine source code. If Sentinel's code is clean, well-documented, and clearly privacy-preserving, Mozilla review is an asset, not a liability.

**Mozilla data collection specification:** Extensions must declare data collection categories using Mozilla's taxonomy:
- `technical` (extension operation data)
- `interaction` (feature usage)
- `web_activity` (URLs, page content)
- `supplemental` (user-provided information)

Sentinel should declare only `technical` and `interaction` at most, with any `web_activity` collection being strictly opt-in.

### 11.3 Safari / Apple App Store

Safari Web Extensions are distributed through the Mac App Store and iOS App Store. Additional requirements:
- **App Tracking Transparency (ATT):** If tracking across apps/websites, must use ATT framework
- **Privacy Nutrition Labels:** Mandatory
- **Sandboxing:** Safari extensions run in a strict sandbox
- **Native code:** Any native code must be bundled. WebExtensions API only in Safari extension JS context.
- **Review:** Manual review by Apple. Can be stricter than Google/Mozilla on privacy.

### 11.4 Microsoft Edge Add-ons

Edge uses the Chromium extension platform. Most Chrome extensions work with minimal modification. Edge-specific policies largely mirror Chrome's, with additional emphasis on:
- Microsoft Store policies (similar to Chrome Web Store)
- Integration with Microsoft account services (optional)

---

## 12. Risk Register (Prioritized)

### CRITICAL (Potential Showstoppers)

| ID | Risk | Impact | Mitigation |
|----|------|--------|------------|
| **R-C1** | **Chrome Web Store rejection for "excessive permissions" or "privacy concerns"** | Project cannot reach Chrome users (75%+ market share). Business non-viable. | Use `activeTab`. Justify every permission. Local-only processing. Clear privacy policy. Build relationship with Chrome developer relations. |
| **R-C2** | **Defamation liability from incorrect trust labeling** | Lawsuits. Bankruptcy. Legal fees kill the project. | Never make factual claims. Always show confidence + evidence + disclaimers. Consult defamation counsel. Consider liability insurance. Form LLC if not already. |
| **R-C3** | **GDPR non-compliance if cloud features launched without privacy architecture** | Fines up to €20M or 4% of global revenue. Reputational destruction. | Architect cloud features with GDPR from day one. Local-only MVP. DPO consultation before any cloud feature. |
| **R-C4** | **Platform store removal for "misleading claims" about AI detection capabilities** | Distribution channel eliminated. User trust destroyed. | Rigorously accurate marketing. No "AI detector" positioning. "Trust assessment" framing with clear disclaimers. Evidence-based outputs. |

### HIGH

| ID | Risk | Impact | Mitigation |
|----|------|--------|------------|
| R-H1 | Model supply chain compromise | Poisoned models mark adversarial content as trustworthy | Model signing + hash verification |
| R-H2 | Adversarial ML bypass becomes widespread | Sentinel becomes unreliable, users abandon | Ensemble of diverse architectures. Continuous adversarial testing. |
| R-H3 | API key exposure in extension code | Unauthorized API usage, cost explosion, data breach | OAuth PKCE, proxy architecture, never embed keys |
| R-H4 | Extension compromise via dependency supply chain | All user data on every page exposed to attacker | Minimal dependencies. SRI. Audit. Lockfile. |
| R-H5 | Content scraping accusations by major platforms | Legal threats, platform blocking, reputational harm | On-device only. No scraping. User's own access. |
| R-H6 | EU AI Act classification change (reclassified as high-risk) | Retroactive compliance burden | Monitor regulatory developments. Document non-high-risk rationale. |

### MEDIUM

| ID | Risk | Impact | Mitigation |
|----|------|--------|------------|
| R-M1 | Telemetry accidentally captures user content | Privacy violation. Regulatory exposure. Trust loss. | Data loss prevention. Audit telemetry pipeline. Never log content. |
| R-M2 | MV3 service worker limitations prevent complex local ML | Product quality reduces. Users churn. | Sandbox pages. ONNX optimized models. Tier-2 cloud option. |
| R-M3 | C2PA/provenance adoption slow → Sentinel has less signal to work with | Product less useful. Higher reliance on weaker ML signals. | Use multiple evidence sources. Design for missing provenance. |
| R-M4 | Competitor patents block specific detection methods | Need to redesign detection pipeline. | Freedom-to-operate search. File defensive patents. |
| R-M5 | Accessibility of trust scores for visually impaired users | Regulatory risk (ADA, EAA). Excluded users. | ARIA labels, screen reader support, confidence as text not just color. |
| R-M6 | Credential harvesting via phishing email pretending to be Sentinel | User accounts compromised if cloud features exist | Anti-phishing guidance. 2FA for accounts. |

### LOW

| ID | Risk | Impact | Mitigation |
|----|------|--------|------------|
| R-L1 | Update channel compromise via developer account takeover | Mass deployment of malicious update | Strong 2FA. Account security monitoring. |
| R-L2 | Copyright claim on training data | Model must be retrained or removed | Due diligence on all training data. Licenses documented. |
| R-L3 | Safari/iOS WebExtension limitations prevent feature parity | Cannot serve Apple users fully | Accept reduced functionality. Communicate honestly. |

---

## 13. Recommended Privacy Architecture Principles

These principles must be embedded in every architectural decision:

1. **Data Minimization is the Default** — Process only what is necessary. Every data point collected must be justified.
2. **Local-First by Design** — The extension must function fully without sending data to any server. Cloud is for enhanced capabilities only, and always opt-in.
3. **On-Device Processing Priority** — ML inference should run on-device whenever technically feasible. Cloud inference is a fallback, not a primary path.
4. **Transparency by Architecture** — The user must be able to see what data is being accessed and why. No hidden data flows.
5. **User-Controlled Data** — Users control what is analyzed, when, and how. Users can delete all local data with one click.
6. **No PII Collection** — The extension must not collect or transmit personally identifiable information. Even telemetry must be anonymous.
7. **Evidence-Based, Not Conclusive** — All outputs are confidence assessments with explainability, not binary verdicts.
8. **Auditable by Default** — Extension source code is open. Data flows are documented. Trust but verify.
9. **Platform Independence** — If a platform blocks Sentinel, users must be able to continue using it via alternative channels (self-distribution, side-loading).
10. **Security Depth** — Defense in depth. Assume compromise at every layer. Limit blast radius of each component.

---

## 14. Key Unanswered Questions for Legal Counsel

These questions require licensed legal advice before proceeding to Phase 1:

1. **What is the defamation liability exposure** for a browser extension that provides confidence scores about content authenticity? Does the "opinion defense" apply?
2. **Does Sentinel qualify as an "interactive computer service"** under Section 230 for its content analysis functionality? If not, what are the implications?
3. **What disclaimers are legally sufficient** to shield against false positive/false negative claims?
4. **What is Byrd Ventures' liability structure?** Is the current LLC sufficient or should Sentinel be a separate entity?
5. **Are any patents likely to block** Sentinel's core detection approaches? Should a freedom-to-operate search be commissioned?
6. **What GDPR registration requirements** apply if Sentinel launches in the EU with cloud features? Is a Data Protection Officer required?
7. **Do the Chrome Web Store and Mozilla Developer Agreement conflict** with any of Sentinel's planned features?
8. **What is the liability for enterprise customers** who use Sentinel's API for consequential decisions? What contractual protections are needed?
9. **Can training on publicly available content for detection purposes** be considered fair use? What are the boundaries?
10. **What additional regulations apply if Sentinel is used by minors** (COPPA, UK Age Appropriate Design Code, etc.)?

---

## 15. References

### Regulations
- **GDPR:** Regulation (EU) 2016/679 — [eur-lex.europa.eu](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
- **CCPA/CPRA:** California Civil Code § 1798.100 et seq.
- **EU AI Act:** Regulation (EU) 2024/1689 — [eur-lex.europa.eu](https://eur-lex.europa.eu/eli/reg/2024/1689/oj)
- **ePrivacy Directive:** Directive 2002/58/EC (under revision as ePrivacy Regulation)
- **COPPA:** 15 U.S.C. § 6501-6506
- **Section 230 CDA:** 47 U.S.C. § 230

### Platform Policies
- **Chrome Web Store Program Policies:** [developer.chrome.com/docs/webstore/program-policies](https://developer.chrome.com/docs/webstore/program-policies)
- **Mozilla Add-on Policies:** [extensionworkshop.com/documentation/publish/add-on-policies](https://extensionworkshop.com/documentation/publish/add-on-policies/)
- **Apple App Store Review Guidelines:** [developer.apple.com/app-store/review/guidelines](https://developer.apple.com/app-store/review/guidelines)
- **Microsoft Edge Add-ons Policies:** [docs.microsoft.com/en-us/microsoft-edge/extensions-chromium/store-policies](https://docs.microsoft.com/en-us/microsoft-edge/extensions-chromium/store-policies)

### Security
- **Manifest V3:** [developer.chrome.com/docs/extensions/mv3](https://developer.chrome.com/docs/extensions/mv3)
- **OWASP Top 10:** [owasp.org/www-project-top-ten](https://owasp.org/www-project-top-ten)
- **NIST SP 800-53:** Security and Privacy Controls for Information Systems
- **C2PA Specification:** [c2pa.org/specifications](https://c2pa.org/specifications)

### Intellectual Property
- **Adobe CAI IP Pledge:** [contentauthenticity.org/adobe-ip-pledge](https://contentauthenticity.org/adobe-ip-pledge)
- **Warhol Foundation v. Goldsmith**, 598 U.S. 508 (2023) — Fair use / transformative use

---

## 16. Next Actions

1. **[LEGAL] Engage legal counsel** to answer the 10 questions in Section 14, with priority on defamation liability (Q1) and Section 230 applicability (Q2).
2. **[SECURITY] Define CSP and sandbox architecture** for the browser extension before any code is written.
3. **[PRIVACY] Draft initial privacy policy** — even for a local-only MVP, this is required for store submission.
4. **[COMPLIANCE] Document GDPR non-applicability rationale** for the local-only MVP. This must be defensible.
5. **[PLATFORM] Initiate contact with Chrome Web Store developer relations** to discuss the extension's permission model before submission.
6. **[IP] Commission a preliminary patent landscape search** for AI content detection methods.
7. **[ARCHITECTURE] Make the final decision on local-only vs. hybrid cloud** before Phase 1 architecture begins — this decision fundamentally shapes the privacy, legal, and security posture.
