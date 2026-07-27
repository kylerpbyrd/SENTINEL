# SECURITY.md — Sentinel Security Policy

> **Project:** Sentinel | **Owner:** Byrd Ventures LLC
> **Version:** 1.0

---

## Security Philosophy

Sentinel operates in a hostile environment. The web pages it analyzes may be actively adversarial. The evidence sources it relies on may be compromised. The models it loads may be poisoned. Every security decision starts from the assumption of an untrusted environment.

---

## Threat Model

| Attacker | Motivation | Capability |
|----------|------------|------------|
| **Disinformation actor** | Make synthetic content appear trustworthy; evade detection | Can craft adversarial content, strip metadata |
| **Platform manipulator** | Poison reputation signals; create fake consensus | Can generate large volumes of content |
| **Nation-state actor** | Compromise trust layer to manipulate what populations perceive as trustworthy | Can compromise supply chains, CAs, infrastructure |
| **Malicious website** | Trick extension into mislabeling content as trustworthy | Controls page DOM |
| **Competing platform** | Discredit Sentinel's accuracy | Has ML expertise for adversarial research |

Full threat model: `docs/TRUST_ENGINE_SPEC.md` (Security section) and Phase 0 research.

---

## Extension Security Model

### Permissions

- **`activeTab`** is the default permission model. Extension accesses page content ONLY when user explicitly invokes analysis.
- **`<all_urls>`** is avoided unless per-domain automatic analysis is explicitly enabled by user.
- **`offscreen`** permission required for IndexedDB proxy (Chrome/Edge only).
- **`storage`** for preferences and session state.
- **`alarms`** for periodic background tasks.

### Mandatory Rules

1. **No unnecessary permissions.** Every permission must be justified in the extension manifest and store listing.
2. **No credential access.** Extension must not access password fields, form data, payment information, or cookies.
3. **No input inspection.** Extension must not read values from `<input>`, `<textarea>`, or `contenteditable` regions where user is actively typing.
4. **No silent cloud uploads.** Any data leaving the device must be preceded by explicit user consent with content preview.
5. **No page-controlled trust.** Extension must not read trust signals from the page being analyzed (meta tags, JS APIs). The analyzed page is untrusted.
6. **UI isolation.** Extension UI must be rendered in isolated contexts that the page cannot access or modify.

---

## Data Handling

| Data Type | Storage | Encryption | Retention |
|-----------|---------|------------|-----------|
| Analysis results (active) | chrome.storage.session | N/A (memory-only) | Session duration |
| User preferences | chrome.storage.local | No | Indefinite |
| User override history | chrome.storage.local | Yes (extension-derived key) | 90 days |
| Cached domain reputation | chrome.storage.local | No (public data) | 7 days |
| Model weights | IndexedDB (via offscreen proxy) | No (but signature-verified) | Until version update |
| Reference data bundles | IndexedDB | Signature-verified | Per-data-type SLA |
| Content analyzed | NEVER STORED | N/A | Process and discard |
| Full URLs | NEVER STORED | N/A | N/A |

**Critical:** `chrome.storage.local` is unencrypted at rest. Any data accessible to the extension is also accessible to any malware with filesystem access. Sensitive data (user override history) must be application-layer encrypted.

---

## Supply Chain Security

| Asset | Protection |
|-------|-----------|
| **Model weights** | Cryptographically signed by Byrd Ventures. Verified before loading. Hashes published in transparency log. |
| **Reference data bundles** | Signed and verified. Atomic swap (verify → replace). Rollback to previous valid bundle on corruption. |
| **Extension code** | Signed by Chrome Web Store / Mozilla AMO. No remotely hosted code (MV3 requirement). |
| **Dependencies** | Minimized. Every dependency reviewed for supply chain risk. Lockfile committed. SRI hashes where applicable. |
| **Dynamic plugins (Phase 3+)** | Sandboxed. Capability Broker enforces access limits. Code signed or install-time hashed. |

---

## Model Integrity

- All model weights are cryptographically signed before distribution.
- Public key for verification is embedded in the extension (not fetched at runtime).
- Model version hashes are published in a transparency log for post-hoc audit.
- Downgrade attacks prevented: model version must be newer than currently loaded version.
- Compromised model revocation: signed revocation notice triggers rollback to known-good version.

---

## Provenance Verification

- C2PA manifests must undergo FULL cryptographic verification: signature validity, certificate chain to trusted root, revocation check, content hash binding.
- Presence alone is not sufficient. A manifest that fails verification is treated as actively suspicious.
- Hardware-attested provenance (camera secure element) carries higher weight than software-based signatures.
- Certificate trust store for content provenance is independent of browser TLS trust store.

---

## Output Integrity

- Confidence assessments expressed in discrete levels, never continuous percentages (anti-inversion, anti-liability).
- Exported assessments cryptographically signed — third parties can verify the report was genuinely produced by Sentinel.
- UI isolated from page context — malicious pages cannot inject fake Sentinel badges or read assessment results.
- Fake badge detection: extension detects and warns when pages display visual elements mimicking Sentinel indicators.

---

## Vulnerability Reporting

Security vulnerabilities should be reported to Byrd Ventures LLC via email (contact information to be published with first release). Do NOT open public issues for security vulnerabilities.

---

*End of SECURITY.md. Full security requirements specification available in the Trust Engine security architecture documents.*
