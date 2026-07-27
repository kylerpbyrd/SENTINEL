# Trust Engine Architecture — Invocation, Privacy, Performance & Failure Design

> **Document Type:** Architecture Specification
> **Phase:** 1 — Architecture
> **Status:** Draft
> **Owner:** Architecture Team
> **Audience:** Engineering, Product, Security
> **Scope:** Defines invocation architecture, privacy boundaries, performance execution model, and failure/degradation architecture for the Trust Engine within Manifest V3 browser extension constraints.
> **Depends On:** `TRUST_ENGINE_SPEC.md`, `docs/architecture/BROWSER_CAPABILITY_ASSESSMENT.md` (Evidence Gathering Capability Assessment)
> **Constraint Authority:** Binding constraints sourced from this document's preamble. Do not debate; design within them.

---

## Binding Constraints (Authoritative)

| Constraint | Value | Source |
|---|---|---|
| Max synchronous on page load | 50ms | MV3 content script budget |
| SW idle timeout | 30s | Chrome/Edge MV3 |
| SW max operation | 5 min | Chrome/Edge MV3 |
| Offscreen documents | 1 at a time | Chrome/Edge platform limit |
| tabCapture availability | Chrome/Edge only; user gesture required | WebExtensions API |
| Cross-origin pixel access | Blocked by canvas tainting | Browser security model |
| chrome.storage.local | Unencrypted; ~10 MB quota | Chrome extension storage |
| chrome.storage.session | In-memory; ~10 MB; cleared on SW termination | Chrome extension storage |
| IndexedDB | Available in offscreen docs, NOT service workers | Web API surface |
| Content scripts | Isolated world; DOM access; no WASM without `wasm-unsafe-eval` | MV3 CSP |
| Sandbox pages | Relaxed CSP; can run WASM/ONNX | MV3 sandbox |
| Service workers | No DOM; orchestration only | MV3 SW lifecycle |

---

# Part 1: Invocation Architecture

## 1.1 Design Principle: Unify Input, Vary Budget

Every invocation path produces an `AnalysisRequest` — a structured object consumed by the Trust Engine's orchestration layer. The invocation path determines:
- What evidence gathering is permitted (scope)
- Resource budget (CPU, memory, time)
- Privacy tier ceiling
- UI behavior

The Trust Engine itself is invocation-agnostic. It receives an `AnalysisRequest`, orchestrates evidence collection per the request's scope and budget, and returns an `Assessment`. No invocation-specific logic lives inside the Trust Engine.

## 1.2 Invocation Models

### Model 1: On-Demand ("Analyze this page")

| Dimension | Definition |
|---|---|
| **Trigger** | User clicks extension toolbar icon. Popup renders. User clicks "Analyze This Page" button. |
| **Permission** | `activeTab` (granted on click). No persistent host permissions. |
| **Evidence scope** | Full: DOM text, metadata, C2PA, pixel forensics (same-origin), video frames (tabCapture, user gesture required), cloud lookups (opt-in per analysis) |
| **Resource budget** | High. User is waiting. 2-10s acceptable for full results. Up to 5 min SW max operation. |
| **Privacy tier** | Tier 0 (local) by default. Tier 2/3 only on explicit per-analysis opt-in. |
| **UI behavior** | Popup → "Analyzing…" with progressive results. Side panel opens with full results when available. Results persist until tab closed or user dismisses. |
| **Navigation away** | Analysis aborted. Evidence gathered so far is discarded. SW receives `tabs.onRemoved` or `tabs.onUpdated` (navigating to different origin). |
| **Synchronous budget** | 0ms. No sync work. All work is async, triggered by user click. |
| **Retry behavior** | User must click again. No automatic retry. |

### Model 2: Per-Domain Automatic

| Dimension | Definition |
|---|---|
| **Trigger** | Page load on a domain in the user's allowlist. Content script detects `document_idle`, checks domain hash against pre-loaded allowlist cache, and signals SW. |
| **Permission** | Host permissions granted via `chrome.permissions.request()` at allowlist-add time. `storage` for allowlist persistence. |
| **Evidence scope** | Lightweight only: content type detection, DOM text extraction (main content region), metadata extraction (same-origin images), C2PA detection (presence check, not verification), cached domain reputation. NO pixel forensics, NO video/audio analysis, NO cloud lookups (unless opted in separately). |
| **Resource budget** | Low. Must not interfere with page experience. Sync: <50ms. Async: <2s total. |
| **Privacy tier** | Tier 0 (local) only. Domain hash may be checked against cached reputation list (local). |
| **UI behavior** | Badge on toolbar icon: grey (no signals), yellow (preliminary signals found), green (strong provenance detected), red (concerning signals). No popup. No side panel. Clicking badge opens full on-demand analysis. Inline page badge (optional, per-domain setting). |
| **Navigation away** | Lightweight analysis completes up to 2s window. If tab closes before 2s, partial results cached to `chrome.storage.session` for badge display on revisit (ephemeral). |
| **Synchronous budget** | <50ms: content type detection via DOM landmarks, page URL scheme detection, allowlist cache check, element count summary. |
| **Retry behavior** | None. Analysis runs once per page load. |

### Model 3: Always-On Passive (Phase 3+, Deferred)

| Dimension | Definition |
|---|---|
| **Trigger** | Every page load, regardless of domain. |
| **Permission** | `<all_urls>` or broad host permissions. HIGH store review scrutiny. |
| **Evidence scope** | Minimal: content type classification, domain hash, element counts. No text extraction. No image access. No C2PA. No network. |
| **Resource budget** | Very low. Sync: <50ms. Async: <300ms. |
| **Privacy tier** | Tier 0 (local). No data leaves device. |
| **UI behavior** | Toolbar icon badge only (domain classification). No popup, no side panel. |
| **Navigation away** | Complete within 300ms. No state to preserve. |
| **Design note** | Deferred to Phase 3+. Architecturally supported via the same `AnalysisRequest` structure (invocation_type: ALWAYS_ON), but NOT implemented in MVP. |

### Model 4: Context Menu

| Dimension | Definition |
|---|---|
| **Trigger** | User right-clicks an `<img>`, `<video>`, or `<canvas>` element. Chrome context menu shows "Analyze with Sentinel" (added via `chrome.contextMenus.create`). User selects it. |
| **Permission** | `activeTab` (granted on context menu click). `contextMenus`. |
| **Evidence scope** | Single element only. Image: C2PA, EXIF/IPTC, pixel forensics (if same-origin), watermark detection. Video: poster frame + tabCapture (user gesture already granted by context menu click). Canvas: pixel analysis (same-origin). |
| **Resource budget** | Medium. Single element = fast. 1-3s typical. |
| **Privacy tier** | Tier 0 (local) by default. Tier 2/3 opt-in per analysis. |
| **UI behavior** | Side panel opens with results for that specific element. Element highlighted on page. |
| **Navigation away** | Analysis aborted. |

### Model 5: Mobile Share (Future, Deferred)

| Dimension | Definition |
|---|---|
| **Trigger** | User shares content (URL, image, text, video) to Sentinel via OS Share Sheet (iOS Share Extension) or Android Share Intent. |
| **Permission** | Platform-level share permission. Extension receives shared content via registered handler. |
| **Evidence scope** | Shared content only. If URL shared: equivalent to on-demand analysis of that URL. If image shared: equivalent to context menu image analysis. If text shared: text-only analysis. |
| **Resource budget** | Varies by content size. |
| **Privacy tier** | Tier 0 (local) by default. Tier 2/3 opt-in. |
| **UI behavior** | Mobile web view or in-extension UI (platform-dependent). |
| **Design note** | Deferred to Phase 4+. Architecturally supported via `AnalysisRequest` with `invocation_type: MOBILE_SHARE` and `target_type` reflecting shared content. |

## 1.3 Unified AnalysisRequest Object

All invocation paths must produce an `AnalysisRequest` with this structure before the Trust Engine begins processing.

```
AnalysisRequest {
  // ── Identity ──
  request_id: String,                    // UUID v4
  invocation_type: Enum {                // How analysis was triggered
    ON_DEMAND,
    DOMAIN_AUTO,
    ALWAYS_ON,
    CONTEXT_MENU,
    MOBILE_SHARE
  },
  created_at: ISO8601,                   // When request was generated
  session_id: String,                    // Browser session correlation (from chrome.storage.session)

  // ── Source Context ──
  tab_id: Number,                        // chrome.tabs.Tab.id
  window_id: Number,                     // chrome.windows.Window.id
  domain_hash: String,                   // SHA-256(hostname), first 16 hex chars. For correlation only.
  url_scheme: Enum {                     // Parsed from window.location
    HTTP,
    HTTPS,
    FILE,
    CHROME,
    ABOUT,
    OTHER
  },
  page_classification: String | Null,    // "news_article" | "social_media_post" | "product_page" | ...
                                         // Populated by content script sync phase

  // ── Target ──
  target_type: Enum {                    // What to analyze
    PAGE,                                // Full page (invocation models 1, 2, 3)
    IMAGE,                               // Single image element (context menu)
    VIDEO,                               // Single video element (context menu)
    TEXT,                                // Text selection (future: context menu on text)
    URL                                  // Shared URL (mobile share)
  },
  target_selector: String | Null,        // CSS selector of specific element (context menu only)
  target_element_hash: String | Null,   // Content hash of target element for dedup

  // ── Budget & Priority ──
  max_sync_ms: Number,                   // 0 for non-page-load invocations; 50 for page-load
  max_total_ms: Number,                  // Soft deadline for complete analysis. 10s for on-demand, 2s for auto, 300ms for always-on.
  priority: Enum {
    USER_BLOCKING,                       // On-demand, context menu (user waiting)
    USER_INITIATED,                      // Mobile share (user initiated, not waiting)
    BACKGROUND                           // Per-domain auto, always-on (no user expectation)
  },

  // ── Scope ──
  evidence_scope: {
    dom_text: Boolean,                   // Extract DOM text
    metadata: Boolean,                   // Extract EXIF/IPTC/XMP
    c2pa_verification: Boolean,         // Verify C2PA manifests
    pixel_forensics: Boolean,            // ELA, noise analysis, frequency domain
    ai_fingerprint: Boolean,             // ONNX generator fingerprint models
    video_frames: Boolean,               // tabCapture + frame analysis
    audio: Boolean,                      // tabCapture + audio analysis
    watermark_detection: Boolean,        // DALL-E, Midjourney, etc.
    cross_modal: Boolean,                // CLIP-ViT, multi-modal
    cloud_lookups: Boolean,              // Tier 2/3 cloud
    reputation: Boolean                  // Domain/publisher reputation (local cache)
  },

  // ── Privacy ──
  privacy_tier_ceiling: Enum {           // Maximum privacy tier permitted
    TIER_0_LOCAL,                        // No cloud, local only
    TIER_1_DOMAIN_HASH,                  // Domain hash queries
    TIER_2_ANONYMIZED_QUERY,            // Anonymized cloud queries
    TIER_3_CONTENT_TRANSMISSION          // Content sent to cloud
  },
  cloud_features_explicitly_consented: Boolean,  // Tier 3 requires true
  consented_features: Map<Feature, Boolean>,     // Per-feature consent flags

  // ── State ──
  status: Enum {
    PENDING,                             // Created, not yet dispatched
    DISPATCHED,                          // Sent to SW for orchestration
    GATHERING,                           // Evidence gathering in progress
    ANALYZING,                           // Trust Engine fusing evidence
    COMPLETE,                            // Assessment produced
    ABORTED,                             // Tab closed / user navigated away
    ERROR                                // Fatal error
  },
  abort_controller_path: String | Null,  // Reference to AbortController in SW

  // ── Result (populated after completion) ──
  assessment_ref: String | Null,         // Reference to Assessment object
  evidence_collected: Number,            // Count of evidence signals gathered
  evidence_missed: Number,               // Count of evidence signals unavailable
  browser_capability_profile: String | Null  // "CHROME_FULL" | "FIREFOX_REDUCED" | etc.
}
```

### 1.3.1 Request Lifecycle State Machine

```
PENDING ──(dispatch)──▶ DISPATCHED ──(start gathering)──▶ GATHERING
                                                            │
                                              ┌─────────────┤
                                              ▼             ▼
                                         ANALYZING      ABORTED (tab closed/user navigated)
                                              │
                                              ▼
                                          COMPLETE
                                              │
                                              ▼
                                          (stored to timeline, 30-day retention)

Any state ──(fatal error)──▶ ERROR
```

### 1.3.2 How Each Model Populates the Request

| Field | ON_DEMAND | DOMAIN_AUTO | ALWAYS_ON | CONTEXT_MENU | MOBILE_SHARE |
|---|---|---|---|---|---|
| `invocation_type` | ON_DEMAND | DOMAIN_AUTO | ALWAYS_ON | CONTEXT_MENU | MOBILE_SHARE |
| `target_type` | PAGE | PAGE | PAGE | IMAGE \| VIDEO | URL \| IMAGE \| TEXT |
| `target_selector` | null | null | null | CSS path to element | null |
| `max_sync_ms` | 0 | 50 | 50 | 0 | 0 |
| `max_total_ms` | 30000 | 2000 | 300 | 10000 | 30000 |
| `priority` | USER_BLOCKING | BACKGROUND | BACKGROUND | USER_BLOCKING | USER_INITIATED |
| `evidence_scope.cloud_lookups` | false (opt-in) | false | false | false (opt-in) | false (opt-in) |
| `evidence_scope.pixel_forensics` | true | false | false | true | true |
| `evidence_scope.video_frames` | true (user gesture) | false | false | true (video target) | true |
| `privacy_tier_ceiling` | TIER_0 (default) | TIER_0 | TIER_0 | TIER_0 (default) | TIER_0 (default) |

## 1.4 Invocation Routing

```
                        ┌──────────────────────┐
                        │   User Interaction    │
                        │  (click, right-click, │
                        │   page load, share)   │
                        └────────┬─────────────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
              ┌──────────┐ ┌───────┐ ┌──────────────┐
              │  Popup    │ │Context│ │Content Script │
              │  Script   │ │ Menu  │ │(page load)    │
              │           │ │Script │ │               │
              └─────┬─────┘ └───┬───┘ └──────┬────────┘
                    │           │             │
                    ▼           ▼             ▼
              ┌─────────────────────────────────────┐
              │        AnalysisRequest Factory       │
              │  (populates request per model rules)  │
              └────────────────┬────────────────────┘
                               │
                               ▼
              ┌─────────────────────────────────────┐
              │         Service Worker              │
              │   (validates, enriches, dispatches)  │
              └────────────────┬────────────────────┘
                               │
                               ▼
              ┌─────────────────────────────────────┐
              │         Trust Engine               │
              │   (orchestrates evidence gathering,  │
              │    fuses signals, produces Assessment)│
              └─────────────────────────────────────┘
```

**Rule:** The `AnalysisRequest Factory` runs in the context that detects the trigger (popup script, content script, context menu handler). The Service Worker validates the request (checks consent, checks quota, checks browser capabilities), enriches with session state, and dispatches to the Trust Engine orchestrator.

---

# Part 2: Privacy Architecture

## 2.1 Privacy Boundary Map

### Boundary 1: Content Script (Isolated World)

**Data that STAYS in content script (never crosses any boundary):**
- Raw, unhashed page URL (full path, query string, fragment)
- Raw DOM content (full innerHTML, textContent of every node)
- Full page text (before extraction/minimization)
- Unprocessed image pixels (raw getImageData buffer)
- User interaction data (scroll positions, click coordinates, keystrokes)
- Form field values (passwords, credit cards, any input values)
- Cookie values, sessionStorage, localStorage contents
- WebRTC/media stream access (unless tabCapture explicitly invoked)
- Content from private/incognito sessions (flagged, never extracted)

**Data that CROSSES from content script → service worker (minimized):**
- `domain_hash`: SHA-256(hostname), truncated to first 16 hex chars (64 bits)
- `url_scheme`: "https" | "http" | "file" — scheme classification only, not full URL
- `page_classification`: Enum string ("news_article", "social_media_post", etc.) — classification label, never content
- `element_counts`: `{ images: 12, videos: 2, iframes: 3, text_blocks: 8 }` — counts only, no content
- `metadata_summary`: `{ has_og_tags: true, schema_types: ["NewsArticle", "WebPage"], c2pa_detected: 3, exif_detected: 5 }` — presence counts, never metadata values
- `page_title_hash`: SHA-256(document.title), truncated — for correlation only
- `detected_social_platform`: "twitter" | "facebook" | "reddit" | null — platform label
- `content_volume`: `{ text_chars: 4500, image_bytes_total: 1200000 }` — aggregate metrics
- `AnalysisRequest` metadata (request_id, invocation_type, target_type, timestamps)

**What MUST NOT cross content script → SW:**
- Full URLs (any component beyond hostname hash)
- Page text content
- Image data (pixels or bytes)
- Metadata values (EXIF fields, C2PA claim content)
- Any user-generated input
- Browsing history data
- Referrer information

### Boundary 2: Service Worker → Sandbox Page (Processing, Not Storage)

**Data that crosses from SW → sandbox:**
- Pre-processed text embeddings (vector of floats) — NOT raw text
- Image pixel data (only when user explicitly consented to pixel forensics, and only for same-origin images)
- C2PA manifest bytes (for cryptographic verification) — manifest, not surrounding content
- Model inputs: structured tensors, not raw content
- Content hashes (SHA-256, pHash, SimHash) for lookup matching
- Analysis session identifier (correlation only)

**Rules:**
- Sandbox receives the minimum data required for computation
- Sandbox has NO network access (relaxed CSP, no `connect-src`)
- Sandbox cannot persist data (no storage APIs)
- Sandbox output is structured Evidence objects only — no raw content in output
- SW validates sandbox output before fusion: ensures no raw content leaks through

### Boundary 3: Service Worker → Offscreen Document

**Data that crosses from SW → offscreen:**
- Image pixel data for tabCapture processing (video frames, audio samples)
- Reference data for media analysis (model weights temporarily loaded)
- Video frame buffers for face-swap/forensic analysis

**Rules:**
- Offscreen document is a transient processing context
- Data is cleared when analysis completes or offscreen document closes
- Offscreen document has access to IndexedDB for model storage but cannot persist analysis data
- Only one offscreen document exists at a time — data for one analysis at a time

### Boundary 4: Sent to Cloud (Explicit Opt-In Only)

**Tier 2 — Anonymized Query (data that MAY be sent):**
- Domain hash (SHA-256, truncated 16 chars)
- Content type classification
- Anonymized feature vector (for domain reputation correlation)
- Certificate revocation check (OCSP — hash of certificate, not URL)

**Tier 3 — Content Transmission (data that MAY be sent, per-analysis opt-in):**
- Image perceptual hash (pHash) — for reverse image search
- Text SimHash — for fact-check database lookup
- Image thumbnail (downsampled, max 256x256) — for cloud forensics
- C2PA manifest content (pub sharable by design)

**Data NEVER sent to cloud under any circumstances:**
- Full URLs (even with opt-in)
- Raw content (full-resolution images, full text)
- User identifiers (no account, no device ID, no browser fingerprint)
- IP address (transmitted via TLS but not logged server-side)
- Browsing history
- Geolocation
- Any PII

### Boundary 5: Stored Locally

**Data stored in `chrome.storage.local` (unencrypted):**
- User preferences (theme, language, analysis defaults, privacy tier defaults)
- Allow/block lists (domain hashes, not full domains)
- Cached reference data hashes (known fake image hashes — hash list, not source data)
- Consent state (per-feature, per-site flags)
- Feature flags and experiment assignments
- Extension version, last update timestamp
- Browser capability profile (detected on install)
- Model version metadata (which versions are downloaded/current)

**Data stored in `chrome.storage.session` (in-memory, ephemeral):**
- Current AnalysisRequest objects (in-flight, until analysis completes or SW dies)
- In-flight evidence cache (Evidence objects being gathered, until fusion)
- Tab-to-analysis correlation map (tab_id → analysis_id)
- AbortController references
- Recent analysis results (last 5 minutes, for popup display)
- User gesture timestamps (for tabCapture permission windows — 5s validity)
- Session correlation token (random, regenerated on SW restart)
- Stale domain reputation cache (loaded from chrome.storage.local into session on SW wake)

**Data stored in IndexedDB (offscreen document):**
- ONNX model weights (.onnx files, 5-500 MB)
- WASM binaries (c2pa-js ~3 MB, ONNX Runtime Web ~5 MB)
- Reference data bundles (C2PA trust lists, known fake image hash database, publisher credibility graph)
- Cached analysis timeline (30-day rolling window of Assessment objects)
- Telemetry batch buffers (before differential privacy application)

**Data requiring encryption at rest:**
- Analysis timeline (Assessment history): AES-256-GCM. Key derived from `chrome.storage.session` ephemeral token. If SW terminates, key is lost and timeline becomes unreadable — acceptable because timeline is for user review, not system function. Alternative: user-provided passphrase for persistent timeline across sessions.
- Telemetry batch buffer: encrypted before IndexedDB write. Decrypted only for differential privacy application and transmission. Key stored in `chrome.storage.session`.
- Exported evidence reports (PDF/JSON): encrypted with user-provided key or generated ephemeral key displayed to user at export time.
- **Rationale:** Timeline reveals browsing patterns over time — which domains user analyzed, how often, with what results. This is sensitive. Unencrypted storage of timeline would expose browsing behavior to any process with filesystem access.

## 2.2 Cloud Evidence Rules

### 2.2.1 Tier 2: Anonymized Query

**Hashing strategy:**
- Domain: SHA-256(hostname), truncated to first 16 hex characters (64 bits of hash). Collision probability with 64 bits across, say, 100M domains is negligible (~2.7×10⁻⁹). Irreversible to full domain.
- Content hash for lookup: pHash (perceptual) for images; SimHash for text. Neither is reversible to original content. Sufficient for deduplication and near-duplicate detection.
- Certificate hash: SHA-256(certificate), truncated to 16 hex chars for OCSP-style revocation check.

**What the server sees:**
- A 64-bit domain hash
- A content type label
- A sanitized feature vector (no raw features that could encode content)
- No association between domain hash and content hash (separated at request level)

### 2.2.2 Tier 3: Content Transmission

**What user MUST see before consenting (per-analysis):**

1. **Content preview.** Exactly what will be sent:
   - Image: thumbnail (256×256) with bounding box showing what's in scope
   - Text: highlighted excerpt showing what text will be transmitted
   - Video: representative keyframe thumbnail
2. **Destination.** Server hostname, jurisdiction, and privacy policy summary (max 3 bullet points).
3. **Processing guarantee.** "This content will be processed and immediately discarded. It will NOT be stored, logged, or used for training."
4. **Duration.** Estimated transmission + processing time.
5. **Action.** A "Send for Analysis" button that must be actively clicked (not a checkbox, not pre-checked). No default send.
6. **Reminder.** "This is optional. Results will still be available with local-only analysis, though confidence may be lower."

**Consent is per-analysis, not persistent.** User must consent for each transmission.

### 2.2.3 Enforcing "Never Store Content"

**Client-side enforcement:**
- Content hash computed before transmission and stored temporarily in `chrome.storage.session`
- After result received, hash is verified against server response metadata
- Hash is discarded after verification (not persisted)
- If hash verification fails (server returned result for different content), result is rejected

**Server-side contract:**
- API is implemented as serverless function with no attached persistent storage
- No database write endpoint exists for content
- Request logs exclude request bodies
- Content is processed in ephemeral container memory only
- Container is destroyed after response — no state survives

**Auditability:**
- Sentinel publishes the server-side processing code as open source
- Third-party auditors can deploy and verify the no-storage guarantee
- Users can send known test content and verify that future queries are not influenced (no caching, no training)

**Architectural invariant:**
> The cloud API accepts (content → result) and returns result. There is no persistence mechanism reachable from the processing path. The code path from `receive_request` to `send_response` contains zero write operations to any storage system.

## 2.3 Encryption & Storage Details

### 2.3.1 chrome.storage.local Allocation

| Data Category | Key Pattern | Max Size | Encryption | Retention |
|---|---|---|---|---|
| Preferences | `prefs:*` | <100 KB | None | Until changed |
| Allow/block lists | `domain_list:allow`, `domain_list:block` | <500 KB | None | Until changed |
| Consent state | `consent:*` | <10 KB | None | Permanent |
| Cached reference hashes | `ref_hashes:*` | <2 MB | None | 7-day staleness |
| Model version metadata | `model_versions` | <10 KB | None | Updated on download |
| Browser capability | `browser_capabilities` | <5 KB | None | Detected on install |
| Feature flags | `feature_flags` | <5 KB | None | Periodic refresh |
| Last update timestamp | `last_update` | <1 KB | None | Updated on update |

### 2.3.2 chrome.storage.session Allocation

| Data Category | Key Pattern | Max Size | Encryption | Lifetime |
|---|---|---|---|---|
| Active analyses | `analysis:*` | <1 MB | None (in-memory) | Until analysis complete or SW killed |
| Tab correlation | `tab_map` | <10 KB | None | Until tab closed |
| Abort controllers | `abort:*` | <1 KB | None | Until analysis complete |
| Recent results cache | `recent_assessments` | <500 KB | None | Last 5 minutes, FIFO |
| Gesture timestamps | `gestures:*` | <1 KB | None | 5s validity window |
| Session token | `session_token` | <100 bytes | None | Regenerated on SW restart |
| Encryption keys | `keys:*` | <1 KB | None (in-memory) | Until SW killed |

**Key insight:** `chrome.storage.session` is the Trust Engine's ephemeral scratch space. Nothing in it survives SW termination. The Trust Engine must be designed to lose all session state on SW kill (30s idle timeout). See Failure Architecture (Part 4, Failure 1).

### 2.3.3 IndexedDB Allocation (Offscreen Document)

| Data Category | Object Store | Approx Size | Encryption | Retention |
|---|---|---|---|---|
| ONNX models | `models` | 5-500 MB | None (public models) | Until user clears or version update |
| WASM binaries | `wasm` | 5-10 MB | None | Until user clears or update |
| C2PA trust lists | `trust_lists` | <5 MB | None (public data) | Refresh on update |
| Known fake hashes | `fake_hashes` | <50 MB | None (hash list only) | Periodic download |
| Publisher credibility | `credibility_graph` | <20 MB | None (public data) | Periodic download |
| Analysis timeline | `timeline` | <50 MB | AES-256-GCM | 30-day rolling window |
| Telemetry buffer | `telemetry_buffer` | <1 MB | AES-256-GCM | 24h batch, then transmit + clear |

### 2.3.4 Encryption Key Management

```
Timeline encryption key lifecycle:
  1. On SW startup: generate random 256-bit key → store in chrome.storage.session
  2. On timeline write: encrypt Assessment with this key → store ciphertext in IndexedDB
  3. On timeline read: decrypt with key from chrome.storage.session → return plaintext
  4. On SW termination: key lost (session storage cleared)
  5. On SW restart: new key generated → previous timeline data cannot be decrypted
  6. Recoverability: NOT designed for recovery. Timeline is ephemeral user history.
     User-exported reports use user-provided or export-time-generated keys.
```

**Design decision:** Timeline encryption keys are ephemeral by design. If a user needs persistent, encrypted analysis history across sessions, they must provide a passphrase (stretched via PBKDF2) or export reports before SW termination. This aligns with the Trust Engine's privacy principle: content analysis history is sensitive and should not persist unencrypted. Ephemeral-by-default is safer than persistent-with-weak-encryption.

## 2.4 Consent Architecture

### 2.4.1 Consent State Machine

```
                           ┌────────────────────┐
                           │      UNKNOWN        │  (first install, no consent state recorded)
                           └────────┬───────────┘
                                    │
                           (first-run UI shown)
                                    │
                                    ▼
                           ┌────────────────────┐
                           │      PENDING        │  (user has seen first-run, hasn't responded)
                           └────────┬───────────┘
                                    │
                     ┌──────────────┼──────────────┐
                     ▼                             ▼
              ┌──────────┐                  ┌──────────┐
              │ GRANTED  │                  │  DENIED  │
              └─────┬────┘                  └─────┬────┘
                    │                             │
                    │ (user revokes consent)       │ (user changes mind,
                    ▼                             │  goes through first-run again)
              ┌───────────┐                       │
              │ WITHDRAWN │                       ▼
              └─────┬─────┘                 ┌──────────┐
                    │                       │ GRANTED  │
                    │ (user re-grants)       └──────────┘
                    ▼
              ┌──────────┐
              │ GRANTED  │  (fresh start — no data recovered from prior GRANTED period)
              └──────────┘
```

**Transitions:**

| From | To | Trigger |
|---|---|---|
| UNKNOWN | PENDING | First-run privacy notice displayed on install |
| PENDING | GRANTED | User clicks "Accept & Enable" |
| PENDING | DENIED | User clicks "Decline" or dismisses permanently |
| GRANTED | WITHDRAWN | User toggles global consent off in settings |
| WITHDRAWN | GRANTED | User toggles global consent on in settings |
| DENIED | PENDING | User opens settings, initiates first-run flow again |

### 2.4.2 First-Run Consent (Blocking)

Before the Trust Engine operates, the user must explicitly agree to:

1. **Data processing scope.** What Sentinel analyzes (text, images, metadata) and what it doesn't (passwords, forms, cookies, browsing history across tabs).
2. **Local-first privacy.** All analysis runs on-device by default. No content leaves the device without explicit per-analysis consent.
3. **Permission justification.** Why each requested permission is necessary:
   - `activeTab`: "To analyze content only when you click the Sentinel icon"
   - `storage`: "To save your preferences and trusted domain list"
   - `contextMenus`: "To let you right-click an image and analyze it"
   - `offscreen` (if requested): "For video and audio analysis, on-demand only"
4. **No telemetry without separate opt-in.** Telemetry is off by default and requires a separate consent step.
5. **Right to withdraw.** How to revoke consent and what happens (see 2.4.4).

**UX law:** Consent must be obtained BEFORE any evidence gathering, even local. Extension must show "Sentinel is not active — complete setup" state.

### 2.4.3 Per-Feature Consent (Independent State Machines)

Each feature maintains its own consent state machine, operating independently:

| Feature | Consent Scope | Default | State Machine |
|---|---|---|---|
| Image analysis (pixel forensics) | All image analysis beyond metadata | GRANTED (opt-out) | Feature-level |
| Cloud features (Tier 2/3) | Any network request beyond local | DENIED (opt-in) | Feature-level, per-analysis confirmation |
| Telemetry | All telemetry collection | DENIED (opt-in) | Feature-level |
| Per-domain automatic analysis | Automatic lightweight analysis on allowlisted domains | DENIED (opt-in per domain) | Per-domain state machine |
| Audio/video analysis (tabCapture) | All tabCapture usage | DENIED (opt-in, per-use gesture) | Feature-level + per-use |

**Per-feature consent state machine** (same structure as global):
```
UNKNOWN → PENDING → GRANTED → WITHDRAWN
                 → DENIED   → PENDING (retry)
```

### 2.4.4 Consent Withdrawal

When user withdraws consent (global or per-feature):

**Immediate actions:**
1. All in-flight analyses abort via AbortController
2. `chrome.storage.session` cleared (in-flight evidence, recent results)
3. Analysis timeline in IndexedDB: user prompted to export or purge. Purge is default after 24h if no response.
4. `chrome.storage.local`: consent flags updated. Preferences, allow/block lists preserved (user effort preserved).
5. IndexedDB: model weights and reference data preserved (no privacy risk — public data). Cached timeline purged.
6. Cloud: no data to purge (content was never stored).
7. Telemetry: batch buffer purged. No queued data transmitted.
8. Extension enters "dormant" state — icon greyed out, popup shows "Sentinel is paused. Enable in settings."

**What survives withdrawal:**
- User preferences (theme, language, etc.)
- Allow/block lists (domain hashes)
- Downloaded models (neutral — user can delete separately)
- Extension installation (not uninstalled)

**What is destroyed:**
- All analysis history
- All in-flight evidence
- All ephemeral session state
- All telemetry buffers

### 2.4.5 Per-Site Allow/Block List

**Default:** Block (no automatic analysis on any domain).

**Allow list entry:**
- User adds domain via popup or settings
- Domain is hashed (SHA-256, truncated 16 chars) for storage — full domain not stored in plaintext
- Host permission requested via `chrome.permissions.request({origins: ["*://*.domain.com/*"]})`
- Entries stored in `chrome.storage.local` under `domain_list:allow`
- "Temporarily allow for this session" option: stored in `chrome.storage.session` only, cleared on SW termination

**Block list overrides allow list:**
- If a domain appears in both lists, block takes precedence
- Block list entries: same hashing strategy, stored in `domain_list:block`

**UI for allow/block:**
- Popup shows current domain status: "Analysis: Auto" / "Analysis: Manual only" / "Analysis: Blocked"
- One-click toggle: Allow / Block / Remove

## 2.5 Telemetry Boundaries

### 2.5.1 Collection Rules

| CAN Collect (opt-in only) | CANNOT Collect (ever) |
|---|---|
| Extension version, browser version, OS | URLs (full, partial, hashed beyond domain hash) |
| Feature usage counts: `{feature: "image_analysis", count: 5}` | Content of any kind (text, images, audio, video, pixels) |
| Analysis type distribution: `{text: 45%, image: 30%, c2pa: 25%}` | User identifiers (no account, no installation ID, no device ID) |
| Error report: `{error_code: "CORS_BLOCKED", count: 12, browser: "Chrome"}` | IP addresses (strip from all transmissions) |
| Aggregate confidence distributions: `{min: 0.15, max: 0.92, mean: 0.64, std: 0.18}` | Browsing patterns (sequence of domains, timing, frequency) |
| Performance: `{p50_analysis_ms: 1200, p95_analysis_ms: 4500}` | Device fingerprint (screen resolution, fonts, WebGL fingerprint) |
| Consent state counts: `{global_granted: 85%, global_denied: 15%}` | PII of any kind |
| Feature flags enabled: `{"video_analysis": true, "cloud_lookups": false}` | Geolocation |
| Model version distribution: `{"image_forensics_v3.1": 70%, "image_forensics_v3.0": 30%}` | Referrer / origin information |
|   | Extension installation timestamp |

### 2.5.2 Differential Privacy Application

**Local DP applied BEFORE transmission:**

- **Privacy budget:** ε = 2.0 per daily batch
- **Count queries:** Laplace noise with scale = sensitivity / ε. Feature usage counts receive Laplace(0, 1/ε) noise.
- **Distribution histograms:** Randomized response. Each user's reported category is randomized with probability determined by ε.
- **Aggregate statistics (mean, percentiles):** Gaussian mechanism with calibrated σ.
- **Error reports:** Sanitized before DP application. Error messages stripped of paths, URLs, filenames. Only error codes + browser/OS metadata survive.

**Batching rules:**
- Minimum batch size: 10 reports before transmission (k-anonymity threshold)
- Maximum batch delay: 24 hours (whichever comes first; if <10 reports in 24h, batch is delayed further to 48h, then transmitted regardless)
- Transmission: dedicated endpoint, no request-level logging server-side
- Transmission frequency: max once per 24 hours

**Architectural invariant:**
> Telemetry data never contains content, never contains identifiers, and is perturbed by local differential privacy (ε=2.0) before any bit leaves the device. The raw telemetry buffer in IndexedDB is encrypted at rest.

---

# Part 3: Performance Architecture

## 3.1 Synchronous Budget (<50ms)

### 3.1.1 What Executes Synchronously (Mandatory, <50ms total)

| Operation | Budget | Description |
|---|---|---|
| Content type detection | <30ms | Check `document.contentType`, scan for semantic HTML5 landmarks (`<main>`, `<article>`, `role="main"`). Count major element types. Classify page as article/social/gallery/etc. |
| Page URL scheme extraction | <1ms | `window.location.protocol` → "https", "http", "file" |
| Domain hash computation | <5ms | SHA-256(hostname) → truncated first 16 chars. Use SubtleCrypto if available; fall back to pre-computed cache for common domains. |
| Allowlist cache check | <2ms | Check pre-loaded `Set` of allowed domain hashes in content script memory (loaded from `chrome.storage.local` on script injection, cached for tab lifetime). |
| Element count summary | <5ms | Count: `<img>` elements, `<video>` elements, `<audio>` elements, `<iframe>` elements, `<article>` elements, `<section>` elements. Counts only, no attribute reading. |
| Inject analysis marker | <1ms | `document.documentElement.dataset.sentinelState = "pending"` — marker for popup to detect state. |
| Request dispatch | <2ms | If domain is allowlisted AND automatic analysis is enabled, queue lightweight `AnalysisRequest` via `chrome.runtime.sendMessage` (fire-and-forget, no await). |

**Total sync budget:** <46ms (with headroom to 50ms).

### 3.1.2 What MUST NOT Execute Synchronously

| Forbidden Operation | Reason |
|---|---|
| Any `fetch()` or `XMLHttpRequest` | Network I/O |
| Any `chrome.storage.*` get/set | Async API; forced async by browser even if data is cached in memory |
| DOM text extraction via `innerText`/`textContent` | Forces layout/reflow; O(n) in text nodes |
| `getComputedStyle()` | Forces style recalculation; expensive |
| Any canvas operations (`drawImage`, `getImageData`) | Pixel manipulation |
| Any WASM instantiation | Heavy, async, multiple MB of compilation |
| Any regex over page content | Unbounded execution time on large pages |
| Any model loading or initialization | Hundreds of milliseconds minimum |
| Any `createImageBitmap()` or image decoding | Async by API design |
| MutationObserver registration | Fine to register sync (fast), but callbacks must be async |
| Any synchronous loop over all DOM nodes | O(n) over full DOM is unbounded |

### 3.1.3 Synchronous Execution Context

The synchronous phase runs in the content script at `run_at: "document_idle"`. By this point, the DOM is parsed but images/stylesheets may still be loading. This is acceptable because the sync phase only needs structural DOM information, not rendered layout.

**If content script injects at `document_start`:**
- Sync phase is deferred to `DOMContentLoaded` listener
- This is acceptable because the 50ms budget is measured from page-ready, not script-injection

## 3.2 Asynchronous Execution Tiers

### Tier 1: Immediate Async (0-300ms post-load)

**Execution context:** Content script, non-blocking.

| Operation | Budget | Description |
|---|---|---|
| DOM text extraction | <100ms | Extract visible text from main content region only (identified in sync phase). Use TreeWalker for text nodes within content container. Cap at 500KB text. |
| Element enumeration | <50ms | Enumerate images, videos, iframes within content region. Collect `src`, `alt`, `width`, `height` attributes. Cap at 100 elements. |
| Meta tag extraction | <30ms | Extract `og:`, `twitter:`, `article:` meta tags. Extract JSON-LD schema.org data from `<script type="application/ld+json">`. |
| Content type refinement | <20ms | Refine page classification beyond sync phase. Use meta tags + element ratios to distinguish article vs. gallery vs. product page. |
| Trigger lightweight analysis | <5ms | If domain allowlisted, send `AnalysisRequest` to SW via `chrome.runtime.sendMessage`. |
| Queue C2PA detection | <5ms | If images detected with known C2PA-capable sources (`src` matches CDN patterns), queue C2PA verification for deferred async. |

**Output:** Enriched `AnalysisRequest` sent to SW with `evidence_scope` populated based on what was detected.

### Tier 2: Deferred Async (300ms-2s post-load)

**Execution context:** Service Worker orchestrating, sandbox page computing.

| Operation | Budget | Description |
|---|---|---|
| C2PA manifest verification | 500ms-2s (first load), 50-200ms (cached) | Load c2pa-js WASM if not cached. Fetch image bytes for same-origin images with C2PA indicators. Verify manifest. |
| Domain cache lookup | <5ms | Check `chrome.storage.session` for cached domain reputation. If not cached, check `chrome.storage.local` for stale reference. |
| Lightweight text analysis | <200ms | Style consistency (sentence length variance, vocabulary diversity), repetition pattern detection. Pure JS statistical analysis — no model required. |
| Update badge | <10ms | SW computes preliminary badge state from Tier 1 + early Tier 2 signals. Sets `chrome.action.setBadgeText()` and `setBadgeBackgroundColor()`. |
| Reference data hash comparison | <100ms | Compare image perceptual hashes against cached known-fake-hash database in IndexedDB (pre-loaded into session on SW wake). |

**Output:** Preliminary Assessment with Tier 1 signals + early Tier 2 signals. Badge updated. Results cached in `chrome.storage.session`.

### Tier 3: User-Invoked (>2s, no hard timeout)

**Execution context:** Service Worker orchestrating, sandbox page, offscreen document.

| Operation | Budget | Description |
|---|---|---|
| Image pixel forensics | 200ms-2s per image | ELA, noise analysis, frequency domain. Requires sandbox page. Same-origin images only. Sequential per image. |
| AI generator fingerprint | 500ms-2s per image | ONNX model inference in sandbox. Model loaded from IndexedDB → sandbox memory. |
| Video frame analysis | 2-10s | tabCapture → offscreen document → extract keyframes → sandbox for face-swap detection. |
| Audio analysis | 1-5s | tabCapture → offscreen document → AudioContext → ONNX vocoder artifact detection. |
| Cross-modal analysis | 1-3s | CLIP-ViT model in sandbox. Image + text embeddings → cosine similarity. |
| Cloud lookups | 200ms-3s (network) | Tier 2/3 requests. Only if user opted in. Parallel where independent. |
| Result compilation | <100ms | Trust Engine fuses all Evidence objects → Assessment. |

**Output:** Complete Assessment with all available signals fused. Stored to timeline. Displayed in side panel.

### Tier 4: Background (Periodic, Idle)

**Execution context:** Service Worker (during `chrome.alarms` callback only).

| Operation | Budget | Description |
|---|---|---|
| Reference data download | Minutes | Known fake image hash list, C2PA trust list, publisher credibility graph. Download in background when SW is awake for other reasons or via `chrome.alarms` (minimum 30s interval on Chrome). |
| Model update check | Seconds | Check for new ONNX model versions. Download if available and user on Wi-Fi (check `navigator.connection.effectiveType`). |
| Cache cleanup | <1s | Evict timeline entries older than 30 days. Purge stale domain reputation cache entries (>7 days). Compact IndexedDB. |
| Telemetry transmission | <5s | Apply DP to batch buffer → transmit → clear buffer. Only if opted in and batch size ≥ 10. |

## 3.3 Execution Queues

### 3.3.1 Queue Architecture

| Queue | Execution Context | Concurrency | Max Queue Depth | Backpressure Behavior |
|---|---|---|---|---|
| **Content Script Queue** | Content script (isolated world) | 1 per tab | 3 tasks per tab | Reject >3. Return "Tab busy — wait for pending analysis." |
| **Service Worker Queue** | Service Worker | 1 (SW is single-threaded) | 5 pending requests | Reject >5. Return "Sentinel is busy. Try again shortly." |
| **Sandbox Queue** | Sandbox page | 1 (WASM/ONNX inference) | 1 pending + 1 running | Reject >2. Queue one additional; reject beyond. |
| **Offscreen Document Queue** | Offscreen document | 1 (platform limit) | 1 pending + 1 running | Reject >2. Queue one additional; reject beyond. |

### 3.3.2 Queue Prioritization

Priority is determined by `AnalysisRequest.priority`:

```
USER_BLOCKING > USER_INITIATED > BACKGROUND
```

**Preemption rules:**
1. When a USER_BLOCKING request arrives and a BACKGROUND request is running:
   - Same tab: pause BACKGROUND, use partial results as starting point for USER_BLOCKING, resume BACKGROUND after
   - Different tab: BACKGROUND continues in parallel if on different queue; if same queue, BACKGROUND is paused
2. USER_BLOCKING requests are never preempted
3. USER_INITIATED requests are preempted only by USER_BLOCKING

### 3.3.3 Queue Routing

```
AnalysisRequest (priority: USER_BLOCKING)
    │
    ▼
Service Worker Queue
    │
    ├──▶ Content Script Queue (tab-specific)
    │    └── DOM text, element enumeration, metadata detection
    │
    ├──▶ Sandbox Queue (if pixel forensics or model inference needed)
    │    └── ELA, noise analysis, frequency domain, ONNX inference, C2PA
    │
    └──▶ Offscreen Document Queue (if tabCapture needed)
         └── Video frame analysis, audio analysis
```

### 3.3.4 Backpressure Protocol

When a queue rejects a task due to being full:

1. The rejecting queue emits a structured `Evidence` object with `signal_value: ERROR`, `error_class: "QUEUE_FULL"`, and `explanation` describing which queue is saturated.
2. The SW receives this Evidence and either:
   - Queues the task for retry (if retry budget remains)
   - Marks the signal source as SKIPPED in the Assessment
   - Reduces confidence ceiling to reflect missing evidence
3. The user sees: "Some analysis components are busy. Results may be incomplete." with a tooltip listing which signals were skipped.
4. No automatic retry beyond 2 attempts. After 2 rejections → SKIPPED permanently for this analysis.

## 3.4 Model Loading Strategy

### 3.4.1 Model Lifecycle State Machine

```
                    ┌──────────┐
                    │ UNLOADED │
                    └─────┬────┘
                          │ (first analysis requiring this model)
                          ▼
                    ┌──────────┐
                    │ LOADING  │
                    └─────┬────┘
                          │
                 ┌────────┼────────┐
                 ▼                 ▼
           ┌──────────┐     ┌──────────┐
           │  READY   │     │  ERROR   │  (corrupted, incompatible, network failure)
           └─────┬────┘     └──────────┘
                 │
                 │ (analysis starts)
                 ▼
           ┌──────────┐
           │  ACTIVE  │
           └─────┬────┘
                 │
                 │ (analysis completes)
                 ▼
           ┌──────────┐
           │   IDLE   │
           └─────┬────┘
                 │
        ┌────────┼────────┐
        ▼                 ▼
  (idle timeout       (SW termination,
   expires)            memory pressure)
        │                 │
        ▼                 ▼
  ┌──────────┐     ┌──────────┐
  │ UNLOADED │     │ UNLOADED │
  └──────────┘     └──────────┘
```

### 3.4.2 Model Sizing & Loading

**Small Models (WASM ~3 MB): C2PA verification**

| Dimension | Strategy |
|---|---|
| **Location** | Bundled with extension (`.wasm` in extension package) |
| **Load trigger** | First C2PA verification request |
| **Load time** | 50-100ms (WASM instantiation from bundled file) |
| **Cache** | In SW memory (module reference). Single instantiation reused. |
| **Unload** | On SW termination (30s idle). Reloaded on next SW wake + first C2PA request. |
| **Fallback** | None needed — C2PA is the only WASM for provenance verification. No alternative. |
| **IndexedDB role** | None. Bundled with extension. No download needed. |

**Medium Models (ONNX ~5-20 MB): Image forensics (generator fingerprints)**

| Dimension | Strategy |
|---|---|
| **Location** | Bundled with extension OR downloaded on first use (configurable: bundle for MVP, download for space-constrained distribution) |
| **Load trigger** | First image forensics analysis (user-invoked only) |
| **Load time** | 200-500ms (ONNX Runtime Web init + model weights into sandbox) |
| **Cache** | In sandbox memory while sandbox is alive. Model bytes in IndexedDB (permanent, until user clears). |
| **Unload** | Sandbox idle timeout: 60s after last inference → unload from memory. Model bytes persist in IndexedDB. |
| **Fallback** | If ONNX model unavailable → skip AI generator fingerprint signal → note in Assessment ("Fingerprint detection unavailable"). Other forensics (ELA, noise) still run. |

**Large Models (ONNX ~150-500 MB): CLIP-ViT (~150 MB), Text analysis models (~500 MB)**

| Dimension | Strategy |
|---|---|
| **Location** | Downloaded on demand. Too large to bundle with extension. |
| **Load trigger** | First analysis requiring this model. User prompted: "This analysis requires a 150 MB model. Download now? (Wi-Fi recommended)" |
| **Load time** | First download: 30s-2min (depending on connection). Subsequent loads from IndexedDB: 1-5s (deserialization). |
| **Cache** | In IndexedDB permanently (until user clears). Loaded into sandbox memory on demand. Model persists across sessions. |
| **Unload** | Immediately after analysis completes + sandbox idle 10s (no grace period — large memory footprint). |
| **Fallback** | If model not downloaded: prompt user to download. If user declines: skip signal, note in Assessment. If insufficient disk space: skip signal, explain. |

**Chrome AI APIs (Gemini Nano): Zero-load alternative**

| Dimension | Strategy |
|---|---|
| **Availability** | Check `chrome.aiOriginTrial.languageModel` or `window.ai` capabilities |
| **Load trigger** | Automatic — Chrome manages model lifecycle |
| **Load time** | Zero (managed by browser, pre-loaded) |
| **Cache** | N/A (browser-managed) |
| **Fallback** | If Chrome AI available → use for text analysis (zero overhead). If unavailable → fall through to ONNX text models. Priority: Chrome AI > ONNX local > Cloud (with consent). |

### 3.4.3 Model Download UX

| State | UI |
|---|---|
| **Model not downloaded** | "Text analysis requires a language model. Download (~150 MB) for more accurate results." [Download] [Skip] |
| **Downloading** | Progress bar with percentage and estimated time. "Downloading text analysis model... 45% (2 min remaining)" |
| **Downloaded, not loaded** | (No UI — loads silently when needed) |
| **Loading for analysis** | "Loading text analysis model..." (shown in analysis progress) |
| **Download failed** | "Model download failed. Check your connection and try again." [Retry] [Skip for now] |
| **Model corrupted** | "Text analysis model appears corrupted. Redownload?" [Redownload] [Skip] |
| **Insufficient space** | "Not enough storage space for this model. Free up space or skip text analysis." |

## 3.5 Memory Budget

| Component | Idle Memory | Peak Memory | Release Strategy |
|---|---|---|---|
| Content script | <5 MB | <15 MB (during DOM extraction) | Automatic — per-tab, released on navigation |
| Service Worker | <5 MB | <20 MB (during orchestration) | Automatic — SW lifecycle |
| Sandbox page (idle) | <10 MB | N/A | Sandbox terminated if idle >60s |
| Sandbox page (small model active) | ~50 MB | ~100 MB | Unload models post-analysis |
| Sandbox page (large model active) | ~200 MB | ~500 MB | Immediate unload post-analysis |
| Offscreen document | ~50 MB | ~200 MB | Terminate after analysis complete |
| **Total extension (idle)** | <20 MB | N/A | Across all contexts |
| **Total extension (peak)** | N/A | <500 MB | Large model unloaded + sandbox/offscreen terminated after analysis |

**Memory pressure response:** If `performance.memory.usedJSHeapSize > 90%` of `performance.memory.jsHeapSizeLimit` in sandbox page:
1. Unload largest inactive model immediately
2. If still >80%: abort current inference, return structured ERROR Evidence
3. Notify SW → SW reduces evidence scope for remaining analysis → produce Assessment with available signals only

---

# Part 4: Failure Architecture

## 4.0 Design Principle

> Every failure must produce structured evidence — an Evidence object with `signal_value: ERROR` and an error classification — rather than throwing exceptions that propagate to the SW or content script. The Trust Engine receives Evidence objects, not exceptions. Errors reduce confidence ceilings but do not crash the pipeline.

**Error Evidence structure:**
```
Evidence {
  signal_type: String,         // Which signal source failed
  signal_value: "ERROR",
  error_class: Enum,           // Classification from failure taxonomy below
  error_code: String,          // Machine-readable code (e.g., "SW_TERMINATED")
  explanation: String,         // Human-readable, max 280 chars
  retryable: Boolean,          // Can the Trust Engine retry?
  confidence_ceiling_impact: Number  // Fraction of confidence ceiling removed (0.0-1.0)
}
```

---

## Failure 1: Service Worker Killed (30s Idle Timeout)

| Dimension | Specification |
|---|---|
| **Detection** | (a) `chrome.runtime.sendMessage` to SW times out (no response in 2s). (b) SW receives `activate` event — on `activate`, check `chrome.storage.session` for orphaned analysis state from previous SW incarnation. (c) Content script detects SW unresponsive: message callback never fires, or `chrome.runtime.lastError` indicates "Could not establish connection." |
| **Detection latency** | 2s (message timeout) for content script. Immediate for SW `activate`. |
| **Evidence produced** | `{ signal_type: "ORCHESTRATION", signal_value: "ERROR", error_class: "INFRASTRUCTURE", error_code: "SW_TERMINATED", explanation: "Service worker terminated due to idle timeout. Analysis state was lost. Total idle before termination: ~30s.", retryable: true, confidence_ceiling_impact: 1.0 }` |
| **System behavior** | On SW restart (`activate`): (1) Scan `chrome.storage.session` for orphaned AnalysisRequest objects. (2) If found: attempt recovery — re-queue analysis with same `request_id`. Prior partial evidence discarded (can't trust partial state across SW termination). Fresh start. (3) If not found: analysis is lost. (4) Send message to originating tab's content script: "Analysis interrupted. Retry?" (5) If content script context also lost (tab navigated): analysis is permanently lost. No automatic retry. (6) Badge reset to default state. |
| **User communication** | Badge shows "!" indicator on toolbar icon. Popup: "Analysis was interrupted (idle timeout). Would you like to retry?" [Retry] [Dismiss]. If user clicks Retry, a fresh `AnalysisRequest` is created (not the recovered one — clean state). |
| **Design mitigation** | Keep SW alive with `chrome.alarms` heartbeat (periodic no-op every 25s) during active analysis. Not foolproof (browser may still kill SW), but reduces idle-timeout kills during in-progress analysis. |

---

## Failure 2: Browser Offline

| Dimension | Specification |
|---|---|
| **Detection** | (a) `navigator.onLine === false` (checked at analysis initiation and before each cloud request). (b) `fetch()` rejects with `TypeError` for network error (not HTTP error — actual network failure). (c) `fetch()` rejects with `AbortError` if timeout fires before response. |
| **Detection latency** | <1ms for `navigator.onLine`. ~5-10s for `fetch()` timeout. |
| **Evidence produced** | `{ signal_type: "NETWORK", signal_value: "ERROR", error_class: "NETWORK", error_code: "OFFLINE", explanation: "Browser is offline. Cloud-dependent evidence sources unavailable. Local analysis continues.", retryable: true, confidence_ceiling_impact: 0.3 }` (0.3 = cloud sources contribute ~30% of total possible evidence weight) |
| **System behavior** | (1) All cloud-dependent evidence sources are skipped (Tier 2, Tier 3). (2) Local-only sources proceed normally (Tier 0, most Tier 1). (3) Assessment is tagged `operational_mode: "offline"`. (4) Cached cloud data is used if available and not stale: domain reputation (cached), known fake hashes (cached). Cache staleness is noted. (5) Cloud requests are NOT queued for retry on reconnect unless analysis is still active (user still on same page, AnalysisRequest still in GATHERING state). |
| **User communication** | Side panel shows offline indicator: "Offline — cloud verification unavailable. Results based on local analysis only." Stale data warning if cached reputation >7 days: "Domain reputation data is X days old." |
| **Transition to online** | Monitor `window.addEventListener('online', ...)` in content script. On reconnect: if analysis is still active (user hasn't navigated away, request state is GATHERING), automatically retry cloud requests. Assessment is updated with new cloud evidence. Badge updates. |

---

## Failure 3: Model Unavailable

| Dimension | Specification |
|---|---|
| **Detection** | (a) WASM instantiation fails: `WebAssembly.instantiate()` rejects or throws `CompileError`. (b) ONNX `InferenceSession.create()` fails: model file missing from IndexedDB, corrupted (checksum mismatch), or incompatible version. (c) Model file checksum verification fails on load: SHA-256 of loaded bytes ≠ expected checksum from model manifest. (d) Model version < minimum required for current evidence type. (e) Sandbox page fails to load entirely (timeout on sandbox page ready signal). |
| **Detection latency** | 50-500ms (WASM compile time). 1-5s (ONNX model load from IndexedDB). |
| **Evidence produced** | `{ signal_type: "MODEL_{NAME}", signal_value: "ERROR", error_class: "MODEL", error_code: "MODEL_LOAD_FAILED" \| "MODEL_CORRUPTED" \| "MODEL_VERSION_MISMATCH", explanation: "Model {name} v{version} failed to load: {reason}. This signal source is unavailable for this analysis.", retryable: true, confidence_ceiling_impact: 0.15 }` |
| **System behavior** | (1) Skip that signal source entirely. (2) Reduce confidence ceiling by `confidence_ceiling_impact` for each missing model. (3) Log error to telemetry buffer (sanitized: model name, version, error code, browser — no content, no URL). (4) Queue model redownload attempt in background (SW, during next idle period or next alarm). Redownload does not block current analysis. (5) If multiple models fail: aggregate confidence ceiling reduction (cap at 0.4 — minimum 0.6 ceiling even if all models fail, because metadata and rule-based signals still function). |
| **User communication** | Results: "Limited analysis — {model name} unavailable. Confidence reduced." Tooltip: "The {model name} model failed to load ({reason}). Some evidence signals could not be evaluated. Sentinel will attempt to restore this model automatically." |
| **Recovery** | Background redownload triggered. On next analysis: re-attempt model load. If successful → model status updated, evidence signal available again. User not notified on recovery unless they open settings. |

---

## Failure 4: CORS Failure (Cross-Origin Pixel Access)

| Dimension | Specification |
|---|---|
| **Detection** | (a) Canvas tainting: `canvas.getImageData()` or `canvas.toDataURL()` throws `SecurityError: The canvas has been tainted by cross-origin data`. (b) `fetch(image.src)` with `{mode: 'cors'}` fails with `TypeError: Failed to fetch` or HTTP 403 due to missing `Access-Control-Allow-Origin`. (c) `createImageBitmap(fetchedBlob)` succeeds (image bytes fetched), but pixel access via canvas still tainted if image drawn without CORS handling. |
| **Detection latency** | <10ms (canvas operation throws immediately). <2s (fetch timeout if server doesn't respond). |
| **Evidence produced** | `{ signal_type: "PIXEL_ACCESS", signal_value: "ERROR", error_class: "CORS", error_code: "CANVAS_TAINTED" \| "FETCH_BLOCKED", explanation: "Cross-origin image at {domain_hash} cannot be analyzed at pixel level due to CORS policy. Image URL hostname hash: {domain_hash}.", retryable: false, confidence_ceiling_impact: 0.0 }` — Confidence ceiling is NOT reduced because CORS-blocked images are expected. The system doesn't penalize what it couldn't possibly access. |
| **System behavior** | (1) Skip pixel-level forensics for that specific image. (2) If same-origin images are also present: analyze those. (3) Per-image signal: try metadata-only path — check if C2PA manifest can be fetched separately (manifest URL may have different CORS policy). (4) Mark image as "CORS_BLOCKED" in evidence chain. No retry — CORS policy won't change during the analysis window. |
| **User communication** | Image card in results shows placeholder with indicator: "Cross-origin — pixel analysis unavailable." Tooltip: "This image is hosted on a different domain. The hosting server doesn't permit pixel-level analysis. Metadata and provenance checks may still be available." |
| **Workaround note** | Sentinel cannot bypass CORS. This is a browser security invariant. The architecture accepts this limitation. User-facing documentation explains that CORS-blocked images limit analysis depth. |

---

## Failure 5: Shadow DOM Inaccessible

| Dimension | Specification |
|---|---|
| **Detection** | (a) `element.shadowRoot` returns `null` for an element that has an attached shadow root with `mode: 'closed'`. Detection is by enumeration: for each element with a known shadow-DOM-using tag (custom elements, platform widgets), check `element.shadowRoot === null && element.attachShadow` would throw. (b) Content extraction returns zero text from a region that visually contains text (heuristic: element has large bounding rect but zero extracted textContent). |
| **Detection latency** | <1ms per element (property access). |
| **Evidence produced** | `{ signal_type: "DOM_ACCESS", signal_value: "ERROR", error_class: "DOM_ACCESS", error_code: "CLOSED_SHADOW_ROOT", explanation: "Closed shadow root detected at {selector}. Content within is inaccessible to the extension. {count} elements affected.", retryable: false, confidence_ceiling_impact: 0.0 }` |
| **System behavior** | (1) Skip content within closed shadow roots. (2) Analyze accessible content only. (3) Flag in assessment: "Some page content is encapsulated and couldn't be analyzed (closed Shadow DOM)." (4) Count inaccessible elements for impact assessment. (5) If >50% of page content (by estimated visual area) is in closed shadow roots → reduce confidence ceiling by 0.3 (assessment is based on minority of content). |
| **User communication** | Results note: "X elements with inaccessible content (closed Shadow DOM). Analysis may not reflect all content on this page." If critical: "Most of this page's content is inaccessible (closed Shadow DOM). This assessment is based on a small portion of visible content and may not be representative." |

---

## Failure 6: tabCapture Denied

| Dimension | Specification |
|---|---|
| **Detection** | (a) `chrome.tabCapture.capture({video: true, audio: true}, callback)` — callback receives `null` stream (user denied). (b) `chrome.tabCapture.getCapturedTabs()` returns empty array. (c) Browser doesn't expose `chrome.tabCapture` API at all (Firefox, Safari). (d) `chrome.runtime.lastError` set with message indicating permission denied. |
| **Detection latency** | <1s (permission prompt shown; user responds; stream or denial returned). |
| **Evidence produced** | `{ signal_type: "MEDIA_CAPTURE", signal_value: "ERROR", error_class: "PERMISSION", error_code: "TABCAPTURE_DENIED" \| "TABCAPTURE_UNAVAILABLE", explanation: "Screen/audio capture permission denied by user." or "Screen/audio capture not supported in this browser.", retryable: true (user can retry with new gesture) \| false (browser lacks API), confidence_ceiling_impact: 0.2 }` |
| **System behavior** | (1) Skip all video frame analysis and audio analysis. (2) Continue with DOM-based and image-based analysis. (3) Do NOT show the permission prompt again for the same analysis session. (4) If browser lacks tabCapture entirely (Firefox, Safari): don't attempt, skip immediately, provide INCOMPATIBLE_BROWSER classification instead. |
| **User communication** | Prompt denial: "Screen capture not allowed. Video and audio analysis skipped. You can still analyze text and images on the page." Browser incompatibility: "Video and audio analysis not available in this browser." Both show available evidence signals in results. |
| **Retry path** | For denial: user can retry by clicking "Analyze Video" again (new user gesture). For incompatibility: no retry. |

---

## Failure 7: Cloud Timeout

| Dimension | Specification |
|---|---|
| **Detection** | (a) `fetch()` with `AbortSignal.timeout(10000)` rejects with `TimeoutError` or `AbortError` (configurable timeout, default 10s for cloud requests). (b) No response received within timeout window. |
| **Detection latency** | 10s (configurable timeout). |
| **Evidence produced** | `{ signal_type: "CLOUD_{SERVICE}", signal_value: "ERROR", error_class: "NETWORK", error_code: "CLOUD_TIMEOUT", explanation: "Cloud API {service_name} timed out after {timeout_ms}ms. Local results provided instead.", retryable: false (for this analysis — don't make user wait again), confidence_ceiling_impact: 0.1 }` |
| **System behavior** | (1) Abort the specific cloud request (AbortController signal). (2) Do NOT retry automatically — user may be on metered connection. (3) Continue with local-only signals. (4) Mark assessment as "partial — cloud timed out." (5) If multiple cloud sources time out: aggregate evidence as UNAVAILABLE. |
| **User communication** | "Cloud verification timed out for {service_name}. Results shown are based on local analysis only. You can retry if you wish." [Retry Cloud Check] button (optional, generates a new lightweight AnalysisRequest for cloud-only signals). |
| **Timeout tuning** | Default 10s. Configurable per cloud service. Fact-check lookup: 5s (should be fast). Reverse image search: 10s (more data). Domain reputation: 3s (should be near-instant). |

---

## Failure 8: Memory Exhaustion

| Dimension | Specification |
|---|---|
| **Detection** | (a) Sandbox page disconnected: message port closes without graceful shutdown (browser killed sandbox to reclaim memory). (b) `performance.memory.usedJSHeapSize > 0.9 * performance.memory.jsHeapSizeLimit` (monitored in sandbox page during inference). (c) SW receives `chrome.runtime.onConnect` port `onDisconnect` from sandbox with no prior completion message. (d) ONNX inference fails with out-of-memory error. |
| **Detection latency** | <1ms (performance.memory check). <5s (sandbox killed → port disconnected). |
| **Evidence produced** | `{ signal_type: "RESOURCE", signal_value: "ERROR", error_class: "RESOURCE", error_code: "MEMORY_EXHAUSTED", explanation: "Memory limit exceeded during {operation}. Model unloaded to free resources. Analysis completed with available signals.", retryable: true (after memory frees), confidence_ceiling_impact: 0.25 }` |
| **System behavior** | (1) Pre-emptive: when heap >80%, stop loading new models, finish current inference only. (2) At >90%: unload largest inactive model. Abort current large model inference if running. (3) Sandbox killed: SW detects port disconnect → marks all sandbox-dependent evidence as UNAVAILABLE. (4) SW reduces evidence scope for remaining analysis: text-only + metadata + cached reputation. No further sandbox/offscreen work. (5) Assessment produced with available signals. Confidence ceiling reduced. |
| **User communication** | "Sentinel reduced analysis scope due to memory constraints. Basic results are available, but some detailed analysis was skipped. Try closing other tabs and retrying for complete analysis." [Retry with Full Analysis] button. |
| **Recovery** | Sandbox page recreated on next analysis request. Browser memory pressure is transient — next analysis attempt likely succeeds. |

---

## Failure 9: Browser Incompatibility

| Dimension | Specification |
|---|---|
| **Detection** | (a) On extension install/startup: feature detection matrix. Check `typeof chrome.tabCapture !== 'undefined'`, `typeof chrome.offscreen !== 'undefined'`, `typeof OffscreenCanvas !== 'undefined'`, `typeof WebAssembly !== 'undefined'`, `navigator.gpu !== undefined` (WebGPU), Chrome AI APIs. (b) Results stored in `chrome.storage.local` as `browser_capability_profile`. Checked on each analysis for evidence scope filtering. (c) Runtime check before each feature use: verify capability flag, not just assume. |
| **Detection latency** | <10ms on startup. Cached thereafter. |
| **Evidence produced** | `{ signal_type: "FEATURE_CHECK", signal_value: "ERROR", error_class: "COMPATIBILITY", error_code: "FEATURE_UNAVAILABLE", explanation: "Feature {name} not available in {browser} v{version}. Related analysis signals disabled.", retryable: false, confidence_ceiling_impact: varies_by_feature }` — One Evidence object per unavailable feature. |
| **System behavior** | (1) On install: detect capability profile → store. (2) On each analysis: filter `evidence_scope` against capability profile. Unavailable sources are NOT attempted. (3) Evidence objects for unavailable features are generated at analysis time (not just install time — to appear in Assessment evidence chain). (4) `browser_capability_profile` field populated in Assessment. (5) Settings page shows capability matrix: green (available), yellow (limited), red (unavailable) per feature. |
| **User communication** | On install: toast/popup: "Some features aren't available in your browser. Sentinel will work with reduced capabilities. [See details]". Settings page: capability matrix. Assessment result: "Note: X evidence signals were unavailable due to browser limitations. Full results are available on Chrome/Edge." |

**Feature → confidence_ceiling_impact mapping:**

| Unavailable Feature | Impact |
|---|---|
| tabCapture (no video/audio analysis) | 0.20 |
| WebGPU (slower ONNX inference) | 0.05 (can still run WASM fallback) |
| Offscreen documents (no tabCapture processing) | 0.20 |
| Chrome AI APIs (no zero-overhead text analysis) | 0.05 (ONNX fallback exists) |
| `chrome.storage.session` (Firefox) | 0.0 (can use `chrome.storage.local` as fallback) |
| sidePanel API (no side panel UI) | 0.0 (UX impact only, not evidence) |

---

## Failure 10: Reference Data Stale/Corrupted

| Dimension | Specification |
|---|---|
| **Detection** | (a) Version check: reference data manifest version < minimum required version for current analysis. (b) Checksum verification: SHA-256 of reference data file ≠ expected checksum from manifest. (c) Last update timestamp > staleness threshold (7 days for domain reputation, 30 days for known fake hashes, 90 days for publisher credibility graph). (d) Parsing failure: JSON parse error on reference data file. |
| **Detection latency** | <50ms (checksum verification). <5ms (timestamp check). |
| **Evidence produced** | `{ signal_type: "REFERENCE_DATA", signal_value: "ERROR", error_class: "DATA", error_code: "REFERENCE_DATA_STALE" \| "REFERENCE_DATA_CORRUPTED", explanation: "Reference data {name} v{current} is outdated (minimum v{required}) or corrupted. Analysis accuracy reduced.", retryable: true, confidence_ceiling_impact: 0.1 }` |
| **System behavior** | (1) Skip reference data comparison entirely (known fake hash match, publisher credibility, domain reputation if cached version is stale). (2) Mark assessment as using reduced reference set. (3) Queue background download of fresh data (SW alarm, next idle period). (4) If cached version exists and is within staleness tolerance: use with warning. If no version or corrupted: skip. (5) Multiple corrupted reference sets: aggregate impact. |
| **User communication** | Assessment: "Reference data needs updating. Some verification checks were skipped. Accuracy may be reduced until update completes." Tooltip shows which reference sets are affected. Settings page shows reference data status: up-to-date green / updating yellow / stale red. |
| **Background refresh** | Periodic alarm fires. SW wakes → checks reference data version against server manifest → downloads if newer → verifies checksum → writes to IndexedDB → marks as current. Stale assessment cache invalidated. |

---

## Failure 11: Analyzer Plugin Crash

| Dimension | Specification |
|---|---|
| **Detection** | (a) Sandbox message port disconnects unexpectedly during inference (no completion message, no error message — port just closes). (b) Sandbox sends malformed Evidence object (missing required fields, wrong types, impossible values). (c) Inference exceeds per-plugin timeout (configurable: 5s for image forensics, 30s for text models). (d) Sandbox sends ERROR Evidence for internal crash. |
| **Detection latency** | 5-30s (timeout). <1s (port disconnect). |
| **Evidence produced** | `{ signal_type: "ANALYZER_{NAME}", signal_value: "ERROR", error_class: "PROCESSING", error_code: "ANALYZER_CRASH" \| "ANALYZER_TIMEOUT" \| "ANALYZER_MALFORMED_OUTPUT", explanation: "Analyzer plugin {name} crashed, timed out, or returned invalid result. Signal skipped.", retryable: true (next analysis), confidence_ceiling_impact: 0.05 }` |
| **System behavior** | (1) Isolate the crashed plugin: mark as DISABLED_FOR_SESSION in SW state. Do NOT attempt to use for remainder of this analysis or any concurrent analyses. (2) Re-create sandbox page (terminate corrupted one). (3) Continue with remaining signal sources. (4) Log crash to telemetry: plugin name, version, error code, browser — sanitized. (5) Plugin is re-enabled on next analysis (fresh sandbox). If plugin crashes 3 times within 10 analyses → mark as SUSPECT in persistent state (chrome.storage.local). SUSPECT plugins require user action to re-enable (settings page). |
| **User communication** | "An analysis component encountered an error ({name}). Remaining checks completed successfully. Confidence may be slightly reduced." If plugin is SUSPECT: "The {name} analyzer has been temporarily disabled due to repeated errors. You can re-enable it in settings." |

---

## Failure 12: Content Too Large

| Dimension | Specification |
|---|---|
| **Detection** | (a) Image: `img.naturalWidth * img.naturalHeight > MAX_PIXELS` (configurable, default 64 megapixels = 8000×8000). Or image file byte size > MAX_BYTES (configurable, default 50 MB). (b) Text: extracted text length > MAX_CHARS (configurable, default 500,000 characters). (c) Video: `video.duration > MAX_DURATION` (configurable, default 600 seconds = 10 minutes). Or total frame count for analysis > MAX_FRAMES (configurable, default 100 frames). (d) Page: total DOM nodes in content region > MAX_NODES (configurable, default 10,000). |
| **Detection latency** | <1ms (property access). <50ms (text length check). |
| **Evidence produced** | `{ signal_type: "INPUT_SIZE", signal_value: "ERROR", error_class: "INPUT", error_code: "CONTENT_TOO_LARGE", explanation: "{content_type} exceeds maximum size: {actual} > {max}. Analysis performed on truncated/subsampled content.", retryable: false, confidence_ceiling_impact: 0.1 }` |
| **System behavior** | (1) Image: downsample to max dimensions (bicubic scaling in canvas). Analyze downsampled version at reduced confidence. Note original dimensions and that analysis is on subsample. (2) Text: truncate to max chars with truncation indicator. Analyze truncated text. Note truncation in Assessment. (3) Video: extract keyframes only (uniform sampling to max frames). Analyze frame subset. (4) DOM: analyze content within first N nodes only (depth-first order). Truncation noted. (5) Confidence ceiling reduced proportionally: if 50% of content analyzed → ceiling reduced by 0.1 per content type. |
| **User communication** | "Content was truncated for analysis ({content_type}: {actual} exceeds {max} limit). Results are based on a representative sample. Full analysis of this content size is not supported." |

---

## Failure 13: User Revokes Permission Mid-Analysis

| Dimension | Specification |
|---|---|
| **Detection** | (a) `chrome.permissions.contains({origins: [...]})` returns `false` for a required origin. Checked at each evidence-gathering step boundary. (b) SW receives `chrome.permissions.onRemoved` event during active analysis. (c) `chrome.tabs.query` for originating tab returns empty (tab closed). This is a sub-case of permission revocation — tab-level permission ended. |
| **Detection latency** | <10ms (permissions API check at step boundary). Near-instant for `onRemoved` event. |
| **Evidence produced** | `{ signal_type: "PERMISSION", signal_value: "ERROR", error_class: "PERMISSION", error_code: "PERMISSION_REVOKED", explanation: "Permission {permission_name} was revoked during analysis. Analysis aborted.", retryable: false, confidence_ceiling_impact: 1.0 }` |
| **System behavior** | (1) Abort all in-flight analysis for the affected tab/domain immediately via AbortController. (2) Discard all in-flight evidence. Do NOT save partial results — user clearly does not want analysis. (3) For activeTab revocation (tab closed): equivalent to navigation-away abort. (4) For host permission revocation: all automatic analyses for that domain cease. On-demand still works for other tabs. (5) For global permission revocation: all analyses cease. Extension enters dormant state. |
| **User communication** | Tab closed: no communication (tab is gone). Permission revoked in settings: popup/side panel shows "Permission revoked. Analysis cannot continue. Re-enable in settings." Settings page shows permission status. |
| **Design invariant** | The Trust Engine checks consent at every evidence-gathering boundary: before DOM text extraction, before image fetch, before pixel analysis, before cloud requests. A permission check failure at any boundary immediately aborts the pipeline. This prevents "mid-analysis" permission gaps from producing partial data. |

---

## Failure 14: Content Behind Paywall

| Dimension | Specification |
|---|---|
| **Detection** | (a) DOM analysis: paywall indicators — common selectors (`.paywall`, `.metered`, `#paywall`, `[data-paywall]`, `.subscription-required`), text patterns ("subscribe to read", "premium content", "log in to continue", "you've reached your monthly limit"). (b) Content length anomaly: article type page classification but text extracted <200 chars (implies teaser-only). (c) Visual occlusion: elements with `position: fixed`, high z-index, covering >50% of viewport, containing paywall-like text. |
| **Detection latency** | <50ms (DOM query + regex on extracted text). |
| **Evidence produced** | `{ signal_type: "CONTENT_ACCESS", signal_value: "ERROR", error_class: "CONTENT_ACCESS", error_code: "PAYWALL_DETECTED", explanation: "Content appears to be behind a paywall. Analysis is based on visible preview only ({chars} characters of preview text). Full article text was not analyzed.", retryable: false, confidence_ceiling_impact: 0.4 }` |
| **System behavior** | (1) Analyze visible content only (preview/teaser). (2) Set confidence ceiling LOW — 0.4 reduction reflects that the majority of content is unseen. (3) Mark analysis as partial: "Paywall detected — limited to visible preview." (4) Do NOT attempt to bypass paywall (no browser automation, no cookie manipulation). This is an ethical and legal constraint. (5) If user has subscription (paywall not triggered for them): no paywall detected → full analysis proceeds normally. |
| **User communication** | Results: "This content appears to be behind a paywall. Analysis is based on the visible preview only (X% of estimated full content) and may not reflect the full article." Tooltip: "Paywalled content analysis is limited. The preview may omit key claims, context, or corrections present in the full article." |
| **Edge case: Soft paywall** | If content is partially visible (e.g., first 3 paragraphs of an article, then paywall), the Trust Engine analyzes the visible portion. Confidence ceiling is reduced proportionally to content visibility. Assessment notes: "X% of content analyzed before paywall. Remaining Y% unavailable." |

---

## 4.1 Failure Interaction Matrix

Failures can co-occur. The Trust Engine handles multiple failures by aggregating Evidence objects and applying combined confidence ceiling reduction:

| Failure Combination | Behavior |
|---|---|
| **Offline + Cloud timeout** | Redundant. Both produce NETWORK ERROR. Only one evidence object recorded. |
| **Memory exhausted + Model unavailable** | Cascading — memory pressure may cause model unavailability. Both recorded; ceiling reduction capped at 0.4 total. |
| **CORS + Paywall** | Independent. CORS affects image analysis; paywall affects text analysis. Both recorded; ceiling reductions are additive. |
| **SW killed + Any in-progress** | SW killed is the dominant failure. All other failures become irrelevant (analysis lost). Recovery: full restart. |
| **Browser incompatible + Any feature** | Incompatibility is pre-filtered. Features that can't run aren't attempted, so their failures can't co-occur with other runtime errors. |

## 4.2 Confidence Ceiling Impact Summary

Cumulative confidence ceiling reduction across all failures is capped at 0.5 — meaning the lowest possible confidence ceiling is 0.5 (50%). This prevents catastrophic confidence collapse from cascading failures.

| Failure | Individual Impact | Cap |
|---|---|---|
| Service worker killed | 1.0 (analysis lost, retry) | N/A (not aggregated) |
| Browser offline | 0.30 |   |
| Model unavailable (per model) | 0.15 |   |
| CORS failure | 0.0 |   |
| Shadow DOM inaccessible | 0.0-0.30 |   |
| tabCapture denied | 0.20 |   |
| Cloud timeout | 0.10 |   |
| Memory exhaustion | 0.25 |   |
| Browser incompatibility (aggregate) | 0.20-0.30 |   |
| Reference data stale/corrupted | 0.10 |   |
| Analyzer plugin crash (per plugin) | 0.05 |   |
| Content too large | 0.10 |   |
| Permission revoked | 1.0 (analysis aborted) | N/A (not aggregated) |
| Paywall | 0.40 |   |
| **Cumulative cap** | **0.50** |   |

**Formula:** `effective_ceiling = 1.0 - min(0.50, sum of individual impacts for non-fatal failures)`

If any fatal failure occurs (SW killed → lost, permission revoked → aborted), the analysis is restarted, not degraded.

---

## 4.3 Error Communication Design

### User-Facing Error Tiers

| Tier | User Sees | Examples |
|---|---|---|
| **Silent degradation** | Normal results, possibly slightly lower confidence. No error message. | CORS on some images; shadow DOM blocking minor content; single model unavailable but alternatives ran. |
| **Informational notice** | Small notice in results: "Some analysis skipped." Expandable to see details. | Offline mode; reference data stale; cloud timeout. |
| **Warning** | Yellow warning banner in results: "Limited analysis." Details visible. | Paywall detected; multiple models unavailable; memory exhaustion during analysis. |
| **Error** | Red error state: "Analysis could not be completed." Reasons listed. Retry button. | SW killed; permission revoked; all sandbox operations failed. |
| **Catastrophic** | Extension shows dormant state. "Sentinel is not operational." Link to settings. | Global consent denied; browser completely incompatible; extension corrupted. |

### Error in Assessment Output

Every Assessment includes an `errors` array:

```
Assessment {
  // ... standard fields ...
  errors: [
    {
      error_code: "PAYWALL_DETECTED",
      severity: "WARNING",
      impact: "Confidence ceiling reduced by 0.40",
      user_message: "Content behind paywall — analysis limited to visible preview."
    }
  ],
  confidence_ceiling: 0.60,  // 1.0 - sum of non-fatal impacts, capped
  evidence_sources: {
    attempted: 12,
    succeeded: 8,
    failed: 4,
    skipped: 2  // Pre-filtered by scope/browser capabilities
  }
}
```

### The "No Surprises" Principle

The Trust Engine must never silently drop evidence. Every signal that was attempted and failed, or was skipped due to constraints, must appear in the Assessment's evidence chain. The user can expand "Why is confidence reduced?" to see every failure and its impact.

---

## Appendix A: Key Design Invariants

These invariants must survive implementation. Any code change that violates an invariant is a regression.

| # | Invariant | Source Section |
|---|---|---|
| 1 | All invocation paths produce the same `AnalysisRequest` structure | Part 1 |
| 2 | The Trust Engine receives `AnalysisRequest`, not invocation-specific parameters | Part 1 |
| 3 | Content scripts never send full URLs across any boundary | Part 2 |
| 4 | Cloud transmission requires explicit per-analysis opt-in with content preview | Part 2 |
| 5 | `chrome.storage.session` data is treated as ephemeral — survives SW kill never assumed | Part 2, Part 4 |
| 6 | Synchronous page-load execution never exceeds 50ms | Part 3 |
| 7 | User-invoked analysis always preempts automatic analysis | Part 3 |
| 8 | Every failure produces structured Evidence (ERROR), never an exception | Part 4 |
| 9 | Confidence ceiling is reduced by failures but never below 0.50 | Part 4 |
| 10 | Every attempted signal appears in Assessment output, whether succeeded, failed, or skipped | Part 4 |

---

## Appendix B: State Machine Summary

| State Machine | States | Documented In |
|---|---|---|
| AnalysisRequest lifecycle | PENDING → DISPATCHED → GATHERING → ANALYZING → COMPLETE / ABORTED / ERROR | Part 1 |
| Consent (global) | UNKNOWN → PENDING → GRANTED / DENIED → WITHDRAWN | Part 2 |
| Consent (per-feature) | Same as global, independent per feature | Part 2 |
| Model lifecycle | UNLOADED → LOADING → READY → ACTIVE → IDLE → UNLOADED / ERROR | Part 3 |
| SW recovery | ACTIVE → TERMINATED → ACTIVATING → RECOVERING → ACTIVE | Part 4 |
| Analyzer plugin | ACTIVE → DISABLED_FOR_SESSION → SUSPECT → (user) → ACTIVE | Part 4 |
