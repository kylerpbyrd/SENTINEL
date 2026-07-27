# PRIVACY.md — Sentinel Privacy Policy

> **Project:** Sentinel | **Owner:** Byrd Ventures LLC
> **Version:** 1.0
> **Principle:** Zero data exfiltration by default.

---

## Privacy Philosophy

Sentinel is designed so that the user's content never needs to leave their device. Privacy is not a feature checkbox — it is a core architectural constraint.

Most AI detection tools send user content to cloud servers for analysis. Sentinel inverts this: local processing is the default. Cloud features exist only as explicit, per-analysis opt-ins. Users are shown exactly what data would leave their device before they consent.

This is not just philosophy. It is also legal risk mitigation. A local-only extension eliminates most GDPR, CCPA, and COPPA obligations because Byrd Ventures processes no personal data.

---

## Data Processing Model

### Tier 1: Local-Only (Default — Always On)

Content never leaves the device. No network requests. No telemetry required.

- C2PA verification (local WASM)
- EXIF/IPTC/XMP metadata reading
- DOM text extraction
- Image forensics (noise analysis, ELA, frequency domain)
- AI generator fingerprint models (local ONNX inference)
- Text statistical analysis
- Known AI content hash matching (local cache)
- Watermark detection (DALL-E, Midjourney)

### Tier 2: Anonymized Query (Implicit Consent — Togglable)

Network requests that do NOT send user content. Domain-level data only.

- Domain reputation lookup (SHA-256 hashed domain, truncated to 16 hex chars)
- Certificate revocation check
- Known AI content hash lookup (hash transmitted, not content)

### Tier 3: Content Transmission (Explicit Opt-In — Off by Default)

Content sent to external service. User must explicitly consent per analysis. Content preview shown before consent.

- Reverse image search
- Cloud-based deepfake detection
- Threat intelligence lookups
- Community signal aggregation
- Fact-check database queries
- Full-model LLM analysis

**All Tier 3 requests:** content processed and immediately discarded server-side. Never stored. Never logged. TLS 1.3 in transit.

---

## What Sentinel NEVER Accesses

Even when technically possible:

- User identity / PII
- Browsing history beyond the current page
- Passwords, form data, payment information
- Cookies, session tokens
- Geolocation
- WebRTC/media streams (except when user explicitly invokes tabCapture)
- Content from other browser tabs
- Content from private/incognito browsing (unless explicitly enabled by user)

---

## Storage Policy

| Storage | Contents | Encryption |
|---------|----------|------------|
| chrome.storage.session | Active analysis state, in-flight evidence | N/A (memory-only, cleared on SW termination) |
| chrome.storage.local | User preferences, consent decisions, domain allow/block lists, cached domain reputation (hashed only) | Application-layer encryption for sensitive data (user overrides) |
| IndexedDB | Model weights, reference data bundles | Signature-verified (integrity), not encrypted (non-sensitive) |
| Server-side | Content analyzed via Tier 3: **NEVER stored.** Logs: request metadata only (no content). | In transit: TLS 1.3. At rest: N/A (nothing stored). |

---

## Telemetry Policy

**Telemetry is opt-in only. Off by default.** Users must explicitly enable telemetry during onboarding or in settings.

| Allowed (with opt-in) | Never Collected |
|-----------------------|-----------------|
| Extension version and browser version | URLs analyzed |
| Feature usage counts (aggregate) | Content analyzed |
| Error reports (sanitized — no URLs, no content) | User identifiers |
| Performance metrics (latency, memory) | IP addresses (beyond transmission necessity) |
| Confidence level distribution (aggregated) | Browsing patterns |
| | Device fingerprints |

Local differential privacy (ε=2.0) applied to all aggregate statistics before transmission.

---

## User Controls

| Control | Default | Description |
|---------|---------|-------------|
| Cloud analysis toggle | OFF | Single switch disables all Tier 3 features |
| Per-feature consent | Separate | Image analysis, cloud features, telemetry each have independent consent |
| Per-domain allow/block list | Empty | Users control which domains are analyzed automatically |
| Consent withdrawal | Supported | Withdrawing consent purges all data requiring that consent |
| Clear all data | Available | Removes all locally stored analysis history, preferences, cache |

---

## Enterprise Privacy (Phase 3+)

- Enterprise admins see aggregate usage, not individual user analysis history
- Mandatory policies may require Tier 1-2 for compliance (Tier 3 always user-consented)
- EU data residency: data processed within EU for EU users
- Audit logs: who analyzed what, when — but never the content itself

---

## Compliance

By processing all Tier 1 evidence locally, Sentinel operates as a **local software tool** rather than a data processor:

- **GDPR:** Does not apply to Byrd Ventures (no personal data processed by the company)
- **CCPA:** Does not apply (no personal information collected)
- **COPPA:** Does not apply (no data collected from children)

When Tier 3 cloud features are used (Phase 3+), full GDPR/CCPA compliance architecture is required. This is deferred to Phase 3 with proper legal counsel.

---

*End of PRIVACY.md. Full privacy architecture specification available in `docs/architecture/TRUST_ENGINE_ARCHITECTURE.md` §2.*
