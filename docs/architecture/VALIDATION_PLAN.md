# Sentinel — Implementation Validation Plan

> **Document Type:** Engineering Validation Plan
> **Phase:** 1.5 — Pre-Implementation Validation
> **Status:** Draft — Awaiting Approval
> **Owner:** Lead Architect
> **Audience:** Engineering Team, QA, Product
> **Depends On:** All Phase 1 Architecture Documents
> **Output:** Go/No-Go decision for Phase 2 implementation

---

## Purpose

The architecture documents define WHAT to build. This document defines what must be PROVEN before building begins.

Every architectural assumption that could be wrong must be validated through a targeted technical spike. A spike answers exactly one question. If the spike fails, the architecture must be revised before implementation proceeds.

**This is not a feature roadmap. This is a risk reduction plan.**

---

## Section 1: Assumption Register

### 1.1 Critical Assumptions (Must Validate — Architecture Rests on These)

Failure of any critical assumption requires an architecture redesign before implementation.

| ID | Assumption | Source Document | Implementation Risk | Failure Impact |
|----|-----------|-----------------|---------------------|----------------|
| **A-C1** | c2pa-js WASM module loads, initializes, and verifies C2PA manifests within an MV3 sandbox page context | Master §8.3, Scheduler §2.4 | **Medium** — library is mature but MV3 sandbox constraints (MessagePort-only communication, no chrome.* APIs) may interfere with WASM initialization. The library was not designed for extension sandbox contexts. | **ARCHITECTURE REDESIGN** — C2PA is the highest-weight Tier 1 signal. Without it, the Trust Engine loses its strongest evidence type. Provenance-first architecture collapses. Must redesign to cloud-only C2PA or alternative provenance path. |
| **A-C2** | ONNX Runtime Web loads and executes inference in an MV3 sandbox page with WASM backend (WebGPU optional acceleration) | Master §8.3, Scheduler §2.4 | **Medium** — ONNX Runtime Web is known to work in browsers but MV3 sandbox pages have restricted CSP and no network access. Model weights must be pre-loaded via MessagePort transfer. | **ARCHITECTURE REDESIGN** — All forensic analyzers (image, audio, text) depend on ONNX inference. Without local ONNX, must move all ML to cloud (privacy regression) or abandon forensic detection (product regression). |
| **A-C3** | Service Worker survives 30+ second analysis chains using keep-alive strategy (long-lived port + periodic chrome.runtime.getPlatformInfo calls) | Master §5.3, Scheduler §1.4 | **HIGH** — MV3 SW lifecycle is aggressive. Community reports of keep-alive strategies being unreliable or patched by Chrome. 30s idle timeout is enforced at the browser level and may not be circumventable. | **ARCHITECTURE REDESIGN** — If SW cannot stay alive for analysis >30s, the progressive analysis model fails. Must redesign to: (a) complete all analysis in <30s (impossible for Tier 3+), (b) move all analysis to offscreen doc (one-at-a-time bottleneck), or (c) use Firefox Event Pages as primary and accept Chrome degradation. |
| **A-C4** | Cross-context message passing (content script → SW → sandbox → SW → popup) reliably transmits ContentSnapshot and Evidence objects without data loss, corruption, or size limit violations | Master §5, Evidence Pipeline §1.2 | **MEDIUM** — chrome.runtime.sendMessage has undocumented size limits (~64MB for most payloads, but varies). structuredClone() has limits. ContentSnapshots with DOM text from large articles may approach limits. Evidence object array may grow large with many analyzers. | **ARCHITECTURE REDESIGN** — If message passing is unreliable for large payloads, must redesign to: (a) chunked messaging protocol, (b) shared memory via MessagePort transferables, or (c) IndexedDB as intermediary (adds latency). |
| **A-C5** | Sandbox page can communicate bidirectionally with Service Worker via MessagePort for multi-request compute sessions (model stays loaded across requests) | Master §8.3, Evidence Pipeline §3 | **MEDIUM** — MV3 sandbox pages have no chrome.* API access. MessagePort is the only channel. The port must survive model loading time (seconds) without timing out. Port lifecycle across SW restarts is unknown. | **ARCHITECTURE REDESIGN** — If MessagePort is unreliable for persistent compute sessions, must reload models per-request (adds 2-5s per analysis) or move compute to offscreen document (Chrome-only, one-at-a-time). |
| **A-C6** | Offscreen document can serve as IndexedDB proxy for service worker (SW cannot access IndexedDB directly in Chrome) | Master §7.1, §8.4 | **HIGH** — SW→offscreen message passing for IndexedDB operations has not been validated at scale. Chrome may enforce restrictions on offscreen document lifetime or API access. One-at-a-time constraint may bottleneck. | **ARCHITECTURE REDESIGN** — Without IndexedDB access from SW, model weights (~5-150MB) cannot be stored persistently. Must: (a) bundle all models in extension package (bloats install size, store rejection risk), (b) download models to chrome.storage.local (10MB quota insufficient), or (c) download on every session (bandwidth + latency). |
| **A-C7** | Fetching image bytes from cross-origin URLs via fetch() in content script (with extension origin privileges, not page origin) bypasses CORS for metadata extraction (not pixel analysis) | Capability Assessment §1.1 | **MEDIUM** — Extension content scripts have elevated fetch privileges compared to page scripts, but server CORS headers still apply. Some CDNs and platforms explicitly block non-browser User-Agent or extension-origin requests. | **ARCHITECTURE REDESIGN** — If cross-origin image byte fetching is blocked, metadata extraction (EXIF/IPTC/C2PA manifest location) fails for most web images. Must redesign to: (a) cloud proxy for image fetching (privacy regression), (b) accept that metadata analysis only works on same-origin images, or (c) negotiate CORS access with major platforms (partnership requirement). |

### 1.2 High Assumptions (Significant Impact If Wrong)

| ID | Assumption | Source Document | Implementation Risk | Failure Impact |
|----|-----------|-----------------|---------------------|----------------|
| **A-H1** | ONNX models (5-150MB) can be loaded into sandbox page memory without exceeding browser memory limits or triggering OOM termination | Scheduler §1.3, §2.3 | **MEDIUM** — Browser memory limits for extensions are undocumented. Large models may be terminated by browser memory pressure. Peak memory with model + inference buffers may exceed 300MB. | **SIGNIFICANT** — Individual analyzers fail or crash. Must reduce model size (quantization, smaller architectures) or increase memory budget. May limit which analyzers can run simultaneously. |
| **A-H2** | WebGPU acceleration provides meaningful speedup (>2x) over WASM-only for ONNX inference in sandbox page | Scheduler §1.5, Master §10 | **LOW** — WebGPU is stable in Chrome/Edge. ONNX Runtime Web WebGPU backend is known to work. The question is whether the speedup is realized in a sandboxed extension context. | **MODERATE** — Without WebGPU, all inference is WASM-only (2-5x slower). Tier 2 analyzers may exceed their 2s budget. Tier 3 may exceed 10s. Progressive result delivery timeline shifts right. Firefox/Safari already planned for WASM-only, so architecture survives — but Chrome/Edge user experience degrades. |
| **A-H3** | chrome.storage.session quota (~10MB) sufficient for active analysis session state (Evidence Graph + intermediate fusion results + in-flight evidence) | Master §7.1 | **MEDIUM** — 10MB is tight. An Evidence object may be 2-5KB serialized. 15 analyzers = ~75KB. Evidence Graph with edges = ~200KB. Intermediate fusion results = ~500KB. AssessmentResult = ~100KB. Total per session: ~1-2MB — within quota for single session. Multiple concurrent sessions (multiple tabs) may exhaust quota. | **SIGNIFICANT** — If session storage overflows, must redesign to: (a) store only current-tab session in session storage, others in local storage (encrypted, slower), (b) reduce per-session state, or (c) cap concurrent analysis sessions. |
| **A-H4** | IndexedDB quota sufficient for model storage (target: 150MB models + 50MB reference data = 200MB) | Master §7.1 | **LOW** — IndexedDB quota is typically a percentage of available disk (often GBs). 200MB should be well within limits. Risk is on low-storage devices (Chromebooks with 16GB total). | **SIGNIFICANT** — On low-storage devices, model download may fail or be evicted. Must: (a) detect quota and degrade to smaller models, (b) offer cloud-only mode as fallback, or (c) accept that low-storage devices get reduced analysis. |
| **A-H5** | CapabilityProvider correctly detects all browser capabilities via two-phase detection (static API presence + runtime probing) within <50ms | Master §10 | **LOW** — Static API detection is trivial (typeof checks). Runtime probing (WebGPU adapter request, WASM threading, AudioContext) may take 50-200ms. Exceeding 50ms delays scheduler startup. | **MODERATE** — If capability detection is slower than expected, scheduler startup is delayed. First evidence delivery timeline shifts right. Mitigation: cache capability profile and re-probe only on extension update. |
| **A-H6** | SW checkpoint-and-resume across 5min hard limit works reliably (serialize state → chrome.storage.session → SW restart → deserialize → resume) | Master §5.3 | **HIGH** — SW restart is not guaranteed to occur immediately. chrome.storage.session is cleared on SW termination in some Chrome versions (Chrome bug). State serialization must be complete and consistent. In-flight analyzer results may be lost. | **SIGNIFICANT** — If checkpoint-and-resume is unreliable, analyses exceeding 5 minutes (deep forensic, batch) are impossible. Must: (a) redesign all analysis to complete in <5min (constrains Tier 5 BACKGROUND analyzers), or (b) move long-running analysis entirely to offscreen document. |
| **A-H7** | Template-based explainability generates coherent, useful natural language from structured evidence for all evidence patterns | Master §9 | **LOW** — Template-based generation is deterministic and well-understood. Risk is in edge cases: unusual combinations of evidence, all signals missing, contradictory signals with no clear resolution. | **MODERATE** — If templates produce confusing or misleading output for edge cases, users lose trust. Must: (a) expand template coverage, (b) add LLM-based fallback (privacy concern), or (c) show structured evidence directly without natural language for edge cases. |

### 1.3 Medium Assumptions (Manageable Impact If Wrong)

| ID | Assumption | Source Document | Implementation Risk | Failure Impact |
|----|-----------|-----------------|---------------------|----------------|
| **A-M1** | Firefox Event Pages provide measurably longer background persistence than Chrome SW (enabling longer analysis chains without keep-alive hacks) | Master §11.2, Capability Assessment §5 | **LOW** — Firefox documentation confirms Event Pages persist longer. The question is empirical: how much longer, and under what conditions do they terminate? | **MANAGEABLE** — Architecture already accounts for browser differences via CapabilityProvider. If Firefox Event Pages don't provide significantly better persistence, Firefox analysis simply uses the same keep-alive strategy as Chrome. |
| **A-M2** | Safari WASM performance adequate for basic analysis (C2PA verification, lightweight text analysis) within extended time budgets | Master §11.2 | **MEDIUM** — Safari WASM performance is known to be slower than Chrome. C2PA verification may take 3-5s instead of <2s. Text analysis may exceed 500ms. | **MANAGEABLE** — Safari already planned for reduced capability tier. If performance is unacceptable, Safari tier may need further reduction (metadata-only, no forensic analysis). |
| **A-M3** | Social media container detection heuristics work on current platform DOMs (Twitter/X, Facebook, Instagram, Reddit, LinkedIn, YouTube) | Capability Assessment §2.2 | **HIGH** — Platform DOMs change frequently without notice. Heuristics break silently. Detecting that a heuristic has broken is as hard as fixing it. | **MANAGEABLE** — Social media content analysis is a Tier 2 feature. If heuristics break frequently, fallback to generic web content analysis (no platform-specific extraction). Content is still analyzed, just less precisely scoped. |
| **A-M4** | Domain hashing (SHA-256 truncated to 16 hex chars) provides meaningful anonymity for Tier 2 anonymized queries | Invocation/Privacy §2.1 | **LOW** — 16 hex chars = 64 bits of hash. Preimage resistance is not required (domain is public). The question is whether 64 bits is sufficient to prevent trivial hash reversal for most domains. | **MANAGEABLE** — If 64-bit hash is deemed insufficient, increase to 128-bit (32 hex chars). No architecture change required. |
| **A-M5** | Application-layer encryption of chrome.storage.local works without key management issues (key derived from extension ID + installation timestamp) | Invocation/Privacy §2.2 | **MEDIUM** — Key derivation from extension ID (public, in manifest) and installation timestamp (discoverable) provides weak security. An attacker with filesystem access can derive the key. | **MANAGEABLE** — The threat model already assumes filesystem access = compromised. The encryption is a defense-in-depth measure against casual inspection, not determined attackers. If stronger encryption needed, add user-provided passphrase (UX friction). |
| **A-M6** | Evidence objects survive structuredClone() across all context boundaries in all browsers | Evidence Pipeline §1.1 | **LOW** — structuredClone() is well-standardized. Edge cases: large objects, objects with Transferable references (ArrayBuffer), objects with circular references (should not exist in Evidence). | **MANAGEABLE** — If structuredClone() fails for some browser/context combinations, fallback to JSON serialization (loses typed arrays, requires explicit Buffer handling). |
| **A-M7** | Per-analyzer timeouts enforced correctly (self-terminate + force-terminate + downgrade) across all execution contexts | Scheduler §1.4 | **MEDIUM** — AbortController works in all contexts. BUT: WASM execution cannot be preempted mid-computation. A runaway WASM function may continue executing beyond its timeout. | **MANAGEABLE** — WASM functions cannot be forcibly terminated. Scheduler wraps in a higher-level timeout that marks the analyzer as TIMED_OUT even if computation continues (resource waste). Mitigation: analyzers must be designed to check AbortSignal between computation chunks. |

### 1.4 Low Assumptions (Minor Impact If Wrong)

| ID | Assumption | Source Document | Implementation Risk | Failure Impact |
|----|-----------|-----------------|---------------------|----------------|
| **A-L1** | 19-field Evidence object size within message passing limits (~2-5KB serialized) | Evidence Pipeline §1.2 | **LOW** | **MINOR** — If Evidence objects are larger, optimize serialization or increase batch size. |
| **A-L2** | chrome.alarms minimum interval (30s Chrome, 1s Firefox) suitable for periodic reference data sync | Master §5.2 | **LOW** | **MINOR** — Reference data sync is not time-critical. 30s granularity is acceptable. |
| **A-L3** | Reference data bundles (domain lists, known hashes, publisher DBs) stay within chrome.storage.local quota (~10MB) | Scheduler §1.3 | **LOW** | **MINOR** — If bundles exceed quota, compress or reduce update frequency. |
| **A-L4** | MutationObserver on content region (not entire document) has acceptable performance overhead for SPA/infinite scroll monitoring | Capability Assessment §3.1 | **LOW** | **MINOR** — If MutationObserver overhead is higher than expected, increase debounce window or limit observation to specific element types. |
| **A-L5** | User consent state machine persists correctly across browser sessions via chrome.storage.local | Invocation/Privacy §2.3 | **LOW** | **MINOR** — chrome.storage.local is reliable for small key-value data. |
| **A-L6** | Differential privacy (ε=2.0) applied to telemetry produces useful aggregate statistics without excessive noise | Invocation/Privacy §2.4 | **LOW** | **MINOR** — If aggregates are too noisy at ε=2.0, increase ε (reduced privacy) or accept noisier data. |
| **A-L7** | Per-persona output adaptation (Journalist vs Educator vs Consumer) changes explainability display correctly without separate code paths | Master §9.3 | **LOW** | **MINOR** — Template selection logic is straightforward. Fallback to default persona if persona detection fails. |
| **A-L8** | Content script CSP `wasm-unsafe-eval` directive permits WASM execution in content scripts (for analyzers that must run in-page) | Capability Assessment §1.2 | **LOW** | **MINOR** — Architecture prefers sandbox for WASM. Content script WASM is a fallback. If CSP blocks it, move those analyzers to sandbox. |

---

## Section 2: Technical Spike Backlog

### 2.1 Critical Spikes (Must Pass — Stop If Any Fails)

---

#### SPIKE-C1: C2PA WASM Validation in MV3 Sandbox

**Objective:** Can the `c2pa-js` library load, initialize, and verify a C2PA manifest within an MV3 sandbox page context?

**Assumption validated:** A-C1

**Success criteria:**
1. `c2pa-js` WASM module loads from extension package assets via sandbox page `<script>` tag
2. Module initializes within 2 seconds on first load, 500ms on subsequent loads (warm cache)
3. A known-valid C2PA-signed JPEG (e.g., Adobe Firefly output) is verified: signature valid, manifest extracted, creator/date/tool assertions readable
4. A known-invalid (tampered) C2PA JPEG is correctly rejected with verification failure
5. A JPEG without C2PA manifest is correctly identified as having no manifest
6. Model files (WASM binary + assets) fit within sandbox page resource constraints

**Failure criteria:**
- `c2pa-js` fails to initialize due to missing browser APIs in sandbox context
- WASM compilation fails due to sandbox CSP restrictions
- Verification takes >5 seconds (unacceptable for Tier 2 FAST analyzer)
- Cannot access locally bundled manifest files from sandbox page
- MessagePort communication back to SW cannot transmit verification results

**Estimated duration:** 2 days

**Dependencies:** None (standalone)

**Browser coverage:** Chrome (primary), Firefox, Edge, Safari

**Required permissions:** `"sandbox"` manifest key, `"content_security_policy"` with `"wasm-unsafe-eval"` or sandbox page with relaxed CSP

**Expected deliverables:**
- Working sandbox page that loads c2pa-js and verifies manifests
- SW↔Sandbox MessagePort communication harness
- Pass/fail report per browser
- Latency measurements (first load, warm load, per verification)
- Bundle size report (WASM binary + JS wrapper)
- List of any sandbox API limitations discovered

---

#### SPIKE-C2: ONNX Runtime Web in MV3 Sandbox

**Objective:** Can ONNX Runtime Web load a quantized image forensics model (~10MB ONNX) and execute inference on an image within an MV3 sandbox page?

**Assumption validated:** A-C2

**Success criteria:**
1. ONNX Runtime Web initializes from bundled assets in sandbox page (WASM backend)
2. A quantized INT8 ONNX model (~10-15MB) loads from IndexedDB (loaded via offscreen doc proxy or MessagePort transfer from SW)
3. Model inference on a 512×512 image completes within 2 seconds (WASM) or 500ms (WebGPU)
4. Inference produces a valid tensor output that matches a pre-computed reference output (deterministic)
5. Model stays loaded in memory for subsequent inference requests (no reload per image)
6. Multiple models can be loaded sequentially (one at a time, evict and reload)

**Failure criteria:**
- ONNX Runtime Web fails to initialize in sandbox context
- Model loading exceeds 10 seconds or fails entirely
- Inference produces incorrect results (WASM numerical issues, quantization errors)
- Memory usage exceeds 300MB during inference
- Sandbox page is terminated by browser during inference
- WebGPU backend not available or produces different results than WASM backend

**Estimated duration:** 3 days

**Dependencies:** SPIKE-C6 (offscreen doc IndexedDB proxy) or accept bundled model for initial spike

**Browser coverage:** Chrome (WASM + WebGPU), Firefox (WASM), Edge (WASM + WebGPU), Safari (WASM)

**Required permissions:** `"sandbox"` manifest key, sandbox page with `"wasm-unsafe-eval"` and WebGPU origin trial (Chrome)

**Expected deliverables:**
- Working sandbox page that loads ONNX Runtime Web, loads a model, and runs inference on a test image
- Reference inference results for validation
- Per-browser latency report (first load, warm load, per-inference)
- Per-browser memory report (peak RSS during model load + inference)
- WebGPU vs WASM speedup measurements
- Model loading strategy validation (IndexedDB vs bundled)

---

#### SPIKE-C3: Service Worker Keep-Alive for Long Analysis

**Objective:** Can a Manifest V3 service worker be kept alive for 60+ seconds using a long-lived port + periodic API calls, sustaining an active analysis session?

**Assumption validated:** A-C3

**Success criteria:**
1. SW survives 60 seconds of continuous activity with a long-lived `chrome.runtime.connect()` port open from popup/side panel
2. SW survives 60 seconds with NO UI port open, using periodic `chrome.runtime.getPlatformInfo()` calls every 25 seconds
3. SW state (a simple counter or session ID) persists through the keep-alive period without reset
4. SW does NOT receive the `"idle"` termination event during the keep-alive period
5. An active `fetch()` or `MessagePort` operation is not interrupted by SW lifecycle management
6. Keep-alive works in Chrome, Edge, and Firefox (Event Pages)

**Failure criteria:**
- SW is terminated before 30 seconds despite keep-alive strategy
- Chrome/Edge behavior differs from documented 30s idle timeout
- Periodic API calls are throttled or suppressed by browser
- SW state is lost between periodic wake events
- Firefox Event Pages terminate unexpectedly

**Estimated duration:** 2 days

**Dependencies:** None (standalone — no Trust Engine logic needed)

**Browser coverage:** Chrome (primary risk), Edge, Firefox (Event Page comparison), Safari (background script)

**Required permissions:** `"background"` (service_worker), `"alarms"` (for alarm-based keep-alive fallback)

**Expected deliverables:**
- Instrumented SW that logs lifecycle events (install, activate, terminate, fetch, message)
- Keep-alive test harness (popup opens port, measures SW lifetime)
- Per-browser SW lifetime report (with port, without port, with alarms)
- Documentation of any keep-alive strategy failures or browser-specific behaviors
- Verdict: is the architecture's keep-alive assumption valid?

---

#### SPIKE-C4: Cross-Context Message Passing Reliability

**Objective:** Can ContentSnapshot and Evidence objects be reliably transmitted across all context boundaries (content script → SW → sandbox → SW → popup/side panel) without data loss, corruption, or size limit violations?

**Assumption validated:** A-C4

**Success criteria:**
1. A ContentSnapshot with 50KB of DOM text, 20 image references, and metadata is transmitted CS→SW without truncation
2. An array of 15 Evidence objects (each ~3KB) is transmitted SW→Popup without truncation
3. Message round-trip time CS→SW→Popup < 10ms for small payloads, < 50ms for large payloads
4. structuredClone() successfully clones an Evidence object with ArrayBuffer fields (image hash, C2PA manifest bytes)
5. Message size limits are empirically determined per context boundary
6. No silent data corruption detected (hash verification on received objects)

**Failure criteria:**
- Messages exceeding an undocumented size limit are silently truncated
- structuredClone() fails for objects with Transferable references
- Message latency exceeds 100ms for any context boundary
- JSON serialization (fallback) loses typed array data
- Chrome/Firefox/Safari have different and incompatible size limits

**Estimated duration:** 2 days

**Dependencies:** SPIKE-C1 or SPIKE-C2 (need real Evidence objects to test with)

**Browser coverage:** Chrome, Firefox, Edge, Safari

**Required permissions:** `"activeTab"` (for content script injection)

**Expected deliverables:**
- Message stress test harness (increasing payload sizes until failure)
- Per-boundary size limit report (CS→SW, SW→Sandbox, SW→Popup, SW→SidePanel)
- Per-boundary latency report (P50, P95, P99)
- structuredClone() compatibility matrix per browser
- Serialization format validation (JSON vs structuredClone vs MessagePort transferables)
- Verdict: is message passing reliable for the architecture's data volumes?

---

#### SPIKE-C5: Sandbox Page MessagePort for Persistent Compute

**Objective:** Can a MessagePort channel between SW and sandbox page sustain a persistent compute session where an ONNX model stays loaded across multiple inference requests?

**Assumption validated:** A-C5

**Success criteria:**
1. MessagePort is established SW→Sandbox and remains open for >60 seconds
2. Sandbox page loads an ONNX model (or simulated 10MB payload) once
3. SW sends 10 sequential ComputeRequest messages over the same port
4. Sandbox responds to each with a ComputeResult (inference output or simulated result)
5. Model stays loaded in sandbox memory across all 10 requests (no reload)
6. MessagePort survives SW idle periods (if SW was briefly idle between requests)
7. Port error/close events are handled gracefully and trigger reconnection

**Failure criteria:**
- MessagePort closes unexpectedly after SW idle period
- Sandbox page is terminated by browser during idle
- Messages are dropped or arrive out of order
- Sandbox page cannot be recreated if terminated (lost model state)
- Port establishment latency > 500ms (delays first analysis)

**Estimated duration:** 2 days

**Dependencies:** SPIKE-C2 (ONNX model loading) for realistic test. Can use simulated payload for initial spike.

**Browser coverage:** Chrome (primary), Firefox, Edge, Safari

**Required permissions:** `"sandbox"` manifest key

**Expected deliverables:**
- Working SW↔Sandbox MessagePort communication harness
- Port lifetime report (survival through SW idle, SW restart, sandbox reload)
- Message throughput report (messages/second, latency per message)
- Port error handling validation
- Verdict: is persistent sandbox compute via MessagePort viable?

---

#### SPIKE-C6: Offscreen Document as IndexedDB Proxy

**Objective:** Can an offscreen document serve as a reliable IndexedDB proxy for the service worker (which cannot access IndexedDB directly in Chrome)?

**Assumption validated:** A-C6

**Success criteria:**
1. Offscreen document is created by SW via `chrome.offscreen.createDocument()`
2. Offscreen document opens an IndexedDB database and stores a 10MB test blob
3. SW sends a message to offscreen doc: "retrieve blob with key X"
4. Offscreen doc reads from IndexedDB and returns blob to SW via `chrome.runtime.sendMessage()`
5. Blob integrity verified (SHA-256 hash matches) after round-trip
6. Offscreen document survives for duration of an analysis session (>60 seconds)
7. Offscreen document is successfully closed and recreated (simulating SW restart)
8. IndexedDB data persists across offscreen document recreation

**Failure criteria:**
- `chrome.offscreen.createDocument()` fails or is not available (Firefox, Safari — expected)
- Offscreen document is terminated before analysis completes
- IndexedDB operations fail with quota errors for 200MB database
- Message passing for large blobs (>10MB) fails or times out
- Only one offscreen document can exist — blocks other use cases

**Estimated duration:** 2 days

**Dependencies:** None (standalone)

**Browser coverage:** Chrome (primary — only browser with offscreen API), Edge

**Required permissions:** `"offscreen"` manifest permission

**Expected deliverables:**
- Working SW→Offscreen→IndexedDB round-trip harness
- Blob throughput and latency report (1MB, 10MB, 50MB blobs)
- Offscreen document lifetime report
- IndexedDB quota report (actual available quota, not theoretical)
- Firefox/Safari fallback validation (identify alternative strategy)
- Verdict: can offscreen doc serve as reliable IndexedDB proxy?

---

#### SPIKE-C7: Cross-Origin Image Byte Fetching

**Objective:** Can image bytes be fetched from cross-origin URLs using content script `fetch()` with extension origin privileges for metadata extraction (C2PA manifest location, EXIF/IPTC parsing)?

**Assumption validated:** A-C7

**Success criteria:**
1. `fetch()` from content script to a cross-origin image URL on a CDN (e.g., `cdn.example.com/image.jpg`) succeeds and returns image bytes
2. Image bytes are complete and uncorrupted (verified by file signature / magic bytes)
3. Fetch works for at least 3 of 5 tested platforms: Cloudinary, Imgix, WordPress CDN, Amazon S3 (public), social media image CDN
4. C2PA manifest location can be identified in fetched bytes (JUMBF box detection)
5. EXIF data can be parsed from fetched bytes for a known-valid JPEG
6. Fetch latency is measured and acceptable (<500ms for typical image sizes)

**Failure criteria:**
- `fetch()` is blocked by CORS headers from the server (even from extension origin)
- `fetch()` succeeds but returns opaque response (no body access)
- Server blocks requests based on User-Agent or Origin header from extension
- Fetch works on some platforms but not others (fragmentation)
- Large images (>10MB) timeout or fail
- fetch() in content script behaves differently than in service worker

**Estimated duration:** 2 days

**Dependencies:** None (standalone)

**Browser coverage:** Chrome (primary), Firefox, Edge, Safari

**Required permissions:** `"activeTab"` or host permissions for test URLs

**Expected deliverables:**
- Test harness that fetches images from 10+ real-world URLs across platforms
- Per-platform success/failure report
- C2PA manifest detection success rate
- EXIF parsing success rate
- Latency report per image size
- Content script vs service worker fetch behavior comparison
- Verdict: how much of the web can Sentinel actually access for metadata extraction?

---

### 2.2 High-Priority Spikes (Validate After Criticals Pass)

---

#### SPIKE-H1: ONNX Model Memory Profiling

**Objective:** What is the actual peak memory usage when loading and running ONNX inference models of varying sizes (5MB, 10MB, 20MB, 50MB, 150MB) in a sandbox page? At what size does the browser terminate the sandbox?

**Assumption validated:** A-H1

**Success criteria:**
1. Peak memory measured for model load + inference at each size tier
2. Browser termination threshold empirically determined (at what memory does Chrome kill the sandbox page?)
3. Memory scales predictably with model size
4. Multiple sequential model loads (load A → evict → load B) don't leak memory
5. Inference on subsequent images doesn't significantly increase memory (model is the dominant consumer)

**Failure criteria:**
- Browser terminates sandbox at memory below expected 300MB limit
- Memory usage is unpredictable (varies by >50% between runs)
- Model loading leaks memory (peak grows with each load/unload cycle)
- ONNX Runtime Web has significant per-session overhead beyond model size

**Estimated duration:** 2 days

**Dependencies:** SPIKE-C2 (ONNX Runtime Web loading)

**Expected deliverables:** Memory profiling report per model size per browser. Peak memory thresholds. Memory leak detection results.

---

#### SPIKE-H2: WebGPU Acceleration Speedup Measurement

**Objective:** What is the actual inference speedup using WebGPU backend vs WASM backend for ONNX Runtime Web in a sandbox page?

**Assumption validated:** A-H2

**Success criteria:**
1. WebGPU backend initializes successfully in sandbox page (Chrome/Edge)
2. Inference speedup measured for image forensics model (10MB): target >2x faster
3. Inference speedup measured for text perplexity model (50MB): target >2x faster
4. WebGPU produces identical numerical results to WASM (within floating-point tolerance)
5. WebGPU memory usage is acceptable (GPU memory is separate from CPU memory budget)

**Failure criteria:**
- WebGPU not available in sandbox page (Origin Trial required?)
- Speedup is marginal (<1.5x) — not worth the complexity
- WebGPU produces different results than WASM (numerical precision issues)
- WebGPU backend is unstable (crashes, hangs, produces NaN outputs)

**Estimated duration:** 1 day

**Dependencies:** SPIKE-C2

**Expected deliverables:** WebGPU vs WASM speedup report. Numerical accuracy comparison. Stability report.

---

#### SPIKE-H3: Session Storage Quota Exhaustion

**Objective:** How many concurrent analysis sessions can fit within chrome.storage.session (~10MB) before quota exhaustion?

**Assumption validated:** A-H3

**Success criteria:**
1. Per-session storage usage empirically measured (Evidence Graph + fusion state + in-flight evidence)
2. Max concurrent sessions determined before quota exhaustion
3. Graceful degradation verified: when quota is near limit, oldest sessions are evicted
4. chrome.storage.session quota accurately reported by browser API (not just theoretical)

**Failure criteria:**
- Single session exceeds 5MB (allowing only 2 concurrent sessions)
- chrome.storage.session quota is smaller than documented 10MB
- Quota exhaustion causes write failures without clear error reporting
- Session eviction corrupts active analysis state

**Estimated duration:** 1 day

**Dependencies:** SPIKE-C4 (Evidence object sizes known)

**Expected deliverables:** Per-session storage report. Concurrent session limit. Eviction behavior validation.

---

#### SPIKE-H4: IndexedDB Quota on Low-Storage Devices

**Objective:** What is the actual IndexedDB quota on a low-storage device (16GB Chromebook, Android tablet)? Can 200MB of models + reference data be stored?

**Assumption validated:** A-H4

**Success criteria:**
1. IndexedDB quota queried programmatically via `navigator.storage.estimate()`
2. 200MB write test: sequentially write 200MB of test data, verify all bytes readable
3. Browser eviction behavior tested: does the browser evict extension IndexedDB under storage pressure?
4. Graceful degradation: if quota < 200MB, which models are prioritized?

**Failure criteria:**
- IndexedDB quota on low-storage device < 100MB
- Browser silently evicts IndexedDB data under storage pressure
- `navigator.storage.estimate()` returns inaccurate quota
- Persistent storage permission (`navigator.storage.persist()`) not available for extensions

**Estimated duration:** 1 day

**Dependencies:** SPIKE-C6 (offscreen IndexedDB proxy)

**Expected deliverables:** Quota report per device class. Eviction behavior. Storage strategy recommendation.

---

#### SPIKE-H5: CapabilityProvider Detection Performance

**Objective:** Can CapabilityProvider detect all required browser capabilities within 50ms?

**Assumption validated:** A-H5

**Success criteria:**
1. Static API detection completes in <1ms
2. Runtime probing (WebGPU, WASM threading, AudioContext) completes in <50ms total
3. WebGPU adapter request doesn't block the thread or cause UI jank
4. Detection results are deterministic (same browser version → same results)
5. Firefox and Safari capability detection works without errors (some APIs undefined)

**Failure criteria:**
- WebGPU adapter request takes >100ms
- WASM threading probe causes crash on unsupported browsers
- Detection results vary between runs on same browser

**Estimated duration:** 1 day

**Dependencies:** None

**Expected deliverables:** Capability detection latency report per browser. Detection accuracy report. Cross-browser compatibility matrix.

---

#### SPIKE-H6: SW Checkpoint-and-Resume Reliability

**Objective:** Can the service worker serialize analysis state, survive intentional termination (5min limit simulation), and resume analysis with in-flight evidence intact?

**Assumption validated:** A-H6

**Success criteria:**
1. Analysis state (Evidence Graph, fusion results, in-flight analyzer list) serialized to chrome.storage.session
2. SW intentionally terminated (close all ports, wait for idle timeout, or use chrome://serviceworker-internals)
3. SW restarts (triggered by alarm or new message)
4. On restart, SW detects interrupted session, deserializes state, resumes analysis
5. Evidence that was in-flight during termination is either recovered or re-requested from analyzers
6. No duplicate Evidence nodes created (dedup by content hash)

**Failure criteria:**
- chrome.storage.session data is lost during SW termination (cleared in some Chrome versions)
- Serialized state is too large for chrome.storage.session (>5MB per session)
- SW restart doesn't occur promptly (minutes of delay)
- Resumed analysis produces different result than uninterrupted analysis (non-determinism)

**Estimated duration:** 2 days

**Dependencies:** SPIKE-C3 (SW keep-alive), SPIKE-C4 (message passing)

**Expected deliverables:** Checkpoint-and-resume reliability report. State serialization size. Resume time measurement. Determinism verification.

---

#### SPIKE-H7: Explainability Template Edge Case Coverage

**Objective:** Do the template-based explanation patterns produce coherent, non-misleading output for unusual evidence combinations (all signals missing, all signals conflicting, single signal only, all Tier 4 signals)?

**Assumption validated:** A-H7

**Success criteria:**
1. Template output for "all signals missing" produces a clear "insufficient evidence" explanation (not confusing default text)
2. Template output for "critical conflict" (Tier 1 vs Tier 1) appropriately urgent but not alarmist
3. Template output for "all Tier 4 signals" correctly communicates low confidence and signal weakness
4. Template output for "single signal" does not overstate confidence
5. Template output for "rapidly changing confidence" (progressive results) doesn't contradict itself
6. All templates pass a manual review: would a non-technical user understand this?

**Failure criteria:**
- Templates produce grammatically incorrect or nonsensical output for some combinations
- Templates overstate confidence when evidence is weak
- Templates understate confidence when evidence is strong
- Progressive result updates contradict earlier output (e.g., "High confidence" → "Uncertain" without explaining why)

**Estimated duration:** 1 day

**Dependencies:** None (can test with mocked Evidence objects)

**Expected deliverables:** Template coverage matrix. Edge case output examples. Manual review pass/fail report.

---

### 2.3 Medium-Priority Spikes (Validate Before End of Phase 1.5)

---

#### SPIKE-M1: Firefox Event Page Persistence

**Objective:** How much longer do Firefox Event Pages survive compared to Chrome service workers? Is the difference architecturally meaningful?

**Assumption validated:** A-M1

**Estimated duration:** 1 day | **Dependencies:** None | **Expected deliverables:** FF vs Chrome SW lifetime comparison. Idle termination timing per browser. Recommendation on whether to exploit FF advantage.

---

#### SPIKE-M2: Safari WASM Performance Baseline

**Objective:** What is the actual WASM performance in Safari for C2PA verification and lightweight ONNX inference? Does it meet extended time budgets?

**Assumption validated:** A-M2

**Estimated duration:** 1 day | **Dependencies:** SPIKE-C1, SPIKE-C2 | **Expected deliverables:** Safari WASM latency report. Pass/fail against extended time budgets. Safari capability tier recommendation.

---

#### SPIKE-M3: Social Media Container Detection Accuracy

**Objective:** How accurately do content container detection heuristics identify content regions on Twitter/X, Facebook, Instagram, Reddit, LinkedIn, and YouTube (as of July 2026 DOM structures)?

**Assumption validated:** A-M3

**Estimated duration:** 2 days | **Dependencies:** None | **Expected deliverables:** Per-platform detection accuracy. False positive rate (non-content identified as content). Heuristic brittleness assessment.

---

#### SPIKE-M4: Domain Hash Anonymity Validation

**Objective:** For 100 popular domains, what percentage can be re-identified from SHA-256(domain) truncated to 16 hex chars via dictionary attack?

**Assumption validated:** A-M4

**Estimated duration:** 0.5 days | **Dependencies:** None | **Expected deliverables:** Re-identification rate. Recommendation on hash length (64-bit vs 128-bit).

---

#### SPIKE-M5: Storage Encryption Key Management

**Objective:** Can application-layer encryption of chrome.storage.local be implemented without key management failures (key derivation from extension ID + install timestamp, key rotation on extension update)?

**Assumption validated:** A-M5

**Estimated duration:** 1 day | **Dependencies:** None | **Expected deliverables:** Encryption harness validation. Key derivation determinism verification. Key survival across extension updates.

---

#### SPIKE-M6: structuredClone() Cross-Browser Compatibility

**Objective:** Does structuredClone() work identically for Evidence objects (with ArrayBuffer fields) in all 4 target browsers?

**Assumption validated:** A-M6

**Estimated duration:** 0.5 days | **Dependencies:** SPIKE-C4 | **Expected deliverables:** structuredClone() compatibility matrix. Per-browser known issues. Fallback strategy validation.

---

#### SPIKE-M7: AbortController Effectiveness for WASM

**Objective:** Can WASM computation be effectively aborted via AbortController? Does the WASM function check the signal, or does it run to completion regardless?

**Assumption validated:** A-M7

**Estimated duration:** 1 day | **Dependencies:** SPIKE-C2 | **Expected deliverables:** AbortController effectiveness report. Time-to-termination after abort. Recommendation on analyzer design for abortability.

---

### 2.4 Low-Priority Spikes (Validate If Time Permits Before Phase 2)

Eight low-priority spikes (SPIKE-L1 through SPIKE-L8) for assumptions A-L1 through A-L8. Each ~0.5 days. Not individually critical — can be validated during Phase 2 implementation with minimal risk.

---

## Section 3: Risk Matrix

### 3.1 Risk Ranking (Likelihood × Impact)

| Rank | Risk | Likelihood | Impact | Score | Spike |
|------|------|-----------|--------|-------|-------|
| **1** | SW cannot stay alive >30s despite keep-alive | Medium | Critical (architecture redesign) | **CRITICAL** | SPIKE-C3 |
| **2** | Offscreen doc fails as IndexedDB proxy | Medium | Critical (model storage) | **CRITICAL** | SPIKE-C6 |
| **3** | Cross-origin image fetch blocked by most platforms | Medium | Critical (metadata extraction) | **CRITICAL** | SPIKE-C7 |
| **4** | ONNX Runtime Web fails in sandbox | Low-Medium | Critical (all forensic ML) | **HIGH-CRITICAL** | SPIKE-C2 |
| **5** | c2pa-js fails in sandbox | Low | Critical (Tier 1 signal) | **HIGH** | SPIKE-C1 |
| **6** | MessagePort unreliable for persistent compute | Medium | High (model reload per analysis) | **HIGH** | SPIKE-C5 |
| **7** | Cross-context messaging unreliable | Medium | High (data loss/corruption) | **HIGH** | SPIKE-C4 |
| **8** | ONNX model memory exceeds browser limits | Medium | High (analyzer crashes) | **HIGH** | SPIKE-H1 |
| **9** | SW checkpoint-and-resume unreliable | High | Significant (long analysis impossible) | **HIGH** | SPIKE-H6 |
| **10** | Session storage quota insufficient | Medium | Significant (concurrent session limit) | **MEDIUM-HIGH** | SPIKE-H3 |
| **11** | WebGPU speedup marginal | Low | Moderate (slower inference) | **MEDIUM** | SPIKE-H2 |
| **12** | IndexedDB quota insufficient on low-storage | Low | Significant (reduced analysis for some users) | **MEDIUM** | SPIKE-H4 |
| **13** | Capability detection exceeds 50ms | Low | Moderate (delayed scheduler start) | **MEDIUM** | SPIKE-H5 |
| **14** | Explainability templates fail edge cases | Low | Moderate (confusing output) | **MEDIUM-LOW** | SPIKE-H7 |
| **15** | Social media heuristics break frequently | High | Manageable (fallback to generic) | **MEDIUM-LOW** | SPIKE-M3 |
| **16** | Firefox Event Pages not significantly better | Low | Manageable (same keep-alive strategy) | **LOW** | SPIKE-M1 |
| **17** | Safari WASM too slow | Medium | Manageable (reduced capability tier) | **LOW** | SPIKE-M2 |
| **18-24** | Medium-priority assumptions M4-M7 | Low | Manageable | **LOW** | SPIKE-M4-M7 |
| **25-32** | Low-priority assumptions L1-L8 | Low | Minor | **VERY LOW** | SPIKE-L1-L8 |

### 3.2 Risk Heat Map

```
IMPACT
  CRITICAL  │  SPIKE-C3   SPIKE-C6              │
            │  SPIKE-C7                         │
            │                                    │
  HIGH      │  SPIKE-C2   SPIKE-C4   SPIKE-H6   │
            │  SPIKE-C1   SPIKE-C5   SPIKE-H1   │
            │                                    │
  MODERATE  │             SPIKE-H3   SPIKE-M3   │
            │             SPIKE-H2              │
            │             SPIKE-H4              │
            │             SPIKE-H5              │
            │                                    │
  LOW       │                        SPIKE-M1   │
            │                        SPIKE-M2   │
            │                        SPIKE-M4-7 │
            │                                    │
  MINOR     │                                    │
            └────────────────────────────────────┘
              LOW         MEDIUM       HIGH
                        LIKELIHOOD
```

---

## Section 4: Proof-of-Concept Plan

### 4.1 Validation Order (Risk-Prioritized)

Spikes are ordered by risk score. Within the same risk tier, dependencies determine order. The plan is structured in waves.

**Wave 0: Foundation (Days 1-2) — Parallel**
These spikes have NO dependencies and validate the most fundamental assumptions:

| Spike | Days | Rationale |
|-------|------|-----------|
| SPIKE-C3 (SW keep-alive) | 2 | **Highest risk overall.** If this fails, the entire progressive analysis model collapses. Must know immediately. |
| SPIKE-C6 (Offscreen IndexedDB proxy) | 2 | **Second highest risk.** Model storage strategy depends on this. Affects SPIKE-C2. |

**Wave 1: Compute Foundation (Days 3-5) — After Wave 0**
These depend on Wave 0 results or are independent compute validation:

| Spike | Days | Depends On |
|-------|------|-----------|
| SPIKE-C2 (ONNX Runtime Web) | 3 | SPIKE-C6 (IndexedDB proxy for model loading) |
| SPIKE-C1 (c2pa-js WASM) | 2 | None (standalone) |
| SPIKE-C7 (Cross-origin fetch) | 2 | None (standalone) |

**Wave 2: Integration (Days 6-8) — After Waves 0+1**
These validate cross-context integration:

| Spike | Days | Depends On |
|-------|------|-----------|
| SPIKE-C5 (MessagePort persistent compute) | 2 | SPIKE-C2 (model loading for realistic test) |
| SPIKE-C4 (Cross-context messaging) | 2 | SPIKE-C1 or C2 (real Evidence objects) |

**Wave 3: Resource Profiling (Days 9-11) — After Wave 1**
These require working models to profile:

| Spike | Days | Depends On |
|-------|------|-----------|
| SPIKE-H1 (ONNX memory profiling) | 2 | SPIKE-C2 |
| SPIKE-H2 (WebGPU speedup) | 1 | SPIKE-C2 |
| SPIKE-H3 (Session storage quota) | 1 | SPIKE-C4 (Evidence sizes known) |
| SPIKE-H4 (IndexedDB quota) | 1 | SPIKE-C6 |
| SPIKE-H5 (Capability detection perf) | 1 | None |

**Wave 4: Robustness (Days 12-14) — After Wave 2**
These validate edge cases and reliability:

| Spike | Days | Depends On |
|-------|------|-----------|
| SPIKE-H6 (SW checkpoint-and-resume) | 2 | SPIKE-C3, SPIKE-C4 |
| SPIKE-H7 (Explainability templates) | 1 | None (mocked Evidence) |

**Wave 5: Cross-Browser & Edge Cases (Days 15-18) — After Wave 1**
These validate browser-specific assumptions:

| Spike | Days | Depends On |
|-------|------|-----------|
| SPIKE-M3 (Social media heuristics) | 2 | None |
| SPIKE-M1 (Firefox Event Pages) | 1 | None |
| SPIKE-M2 (Safari WASM) | 1 | SPIKE-C1, SPIKE-C2 |
| SPIKE-M4-M7 (Medium-priority) | 2 | Various |

**Wave 6: Low Priority (Days 19-20) — Optional**
SPIKE-L1 through SPIKE-L8. Can be deferred to Phase 2 if timeline is tight.

### 4.2 Timeline (Aggressive — 4 Weeks for Critical + High)

```
Week 1: Waves 0-1 (Critical foundation)
  Mon-Tue: SPIKE-C3 (SW keep-alive) || SPIKE-C6 (IndexedDB proxy)
  Wed-Fri: SPIKE-C2 (ONNX Runtime) || SPIKE-C1 (c2pa-js) || SPIKE-C7 (Cross-origin fetch)

  GO/NO-GO GATE #1: All Critical spikes must show feasibility.
  If any Critical spike FAILS → STOP, redesign architecture.

Week 2: Waves 2-3 (Integration + Profiling)
  Mon-Tue: SPIKE-C5 (MessagePort) || SPIKE-C4 (Messaging)
  Wed-Fri: SPIKE-H1-H5 (Memory, WebGPU, Storage, Quota, Capability)

Week 3: Waves 4-5 (Robustness + Cross-Browser)
  Mon-Tue: SPIKE-H6 (Checkpoint-resume) || SPIKE-H7 (Templates)
  Wed-Fri: SPIKE-M1-M7 (Firefox, Safari, Social media, etc.)

  GO/NO-GO GATE #2: All Critical AND High spikes must pass.
  If any High spike fails → assess whether architecture can absorb without redesign.

Week 4: Wave 6 (Low priority) + Buffer + Report Writing
  Mon-Wed: SPIKE-L1-L8 (low-priority if time permits)
  Thu-Fri: Validation Report compilation, ADR review, Go/No-Go recommendation
```

---

## Section 5: Prototype Repository Structure

### 5.1 Repository Layout

```
sentinel-validation/
├── README.md                          # How to run spikes, prerequisites
├── package.json                       # Shared dev dependencies only (TypeScript, test runner, linter)
├── tsconfig.base.json                 # Shared TS config
│
├── harness/                           # Shared spike infrastructure (MINIMAL — no production code)
│   ├── messaging/                     # Cross-context message helpers (connect, send, receive)
│   ├── storage/                       # Storage wrapper (session, local, IndexedDB via offscreen)
│   ├── capabilities/                  # Capability detection probes (static + runtime)
│   └── reporting/                     # Test result logger, pass/fail assertions
│
├── spikes/
│   ├── c1-c2pa-wasm/                  # SPIKE-C1
│   │   ├── README.md                  # Objective, success criteria, how to run
│   │   ├── manifest.json              # MV3 manifest (minimal permissions)
│   │   ├── service-worker.ts          # SW: spawns sandbox, sends test images
│   │   ├── sandbox/
│   │   │   ├── sandbox.html           # Sandbox page
│   │   │   └── c2pa-verifier.ts       # Loads c2pa-js, verifies manifests
│   │   ├── test-assets/               # Test C2PA-signed images, tampered images
│   │   └── results.md                 # Filled after spike: pass/fail, measurements
│   │
│   ├── c2-onnx-sandbox/               # SPIKE-C2
│   │   ├── README.md
│   │   ├── manifest.json
│   │   ├── service-worker.ts
│   │   ├── sandbox/
│   │   │   ├── sandbox.html
│   │   │   └── onnx-inference.ts
│   │   ├── models/                    # Small test ONNX models (committed or .gitignored if large)
│   │   └── results.md
│   │
│   ├── c3-sw-keepalive/               # SPIKE-C3
│   │   ├── README.md
│   │   ├── manifest.json
│   │   ├── service-worker.ts          # SW with keep-alive instrumentation
│   │   ├── popup/
│   │   │   ├── popup.html
│   │   │   └── popup.ts               # Opens long-lived port, measures SW lifetime
│   │   └── results.md
│   │
│   ├── c4-messaging/                  # SPIKE-C4
│   │   ├── README.md
│   │   ├── manifest.json
│   │   ├── content-script.ts
│   │   ├── service-worker.ts
│   │   ├── sandbox/
│   │   ├── popup/
│   │   └── results.md
│   │
│   ├── c5-messageport-compute/        # SPIKE-C5
│   │   └── ...
│   │
│   ├── c6-offscreen-indexeddb/        # SPIKE-C6
│   │   └── ...
│   │
│   ├── c7-cross-origin-fetch/         # SPIKE-C7
│   │   └── ...
│   │
│   ├── h1-onnx-memory/               # SPIKE-H1
│   ├── h2-webgpu-speedup/            # SPIKE-H2
│   ├── h3-session-storage/           # SPIKE-H3
│   ├── h4-indexeddb-quota/           # SPIKE-H4
│   ├── h5-capability-detection/      # SPIKE-H5
│   ├── h6-checkpoint-resume/         # SPIKE-H6
│   ├── h7-explainability-templates/  # SPIKE-H7
│   │
│   ├── m1-firefox-eventpages/        # SPIKE-M1
│   ├── m2-safari-wasm/               # SPIKE-M2
│   ├── m3-social-media-heuristics/   # SPIKE-M3
│   ├── m4-domain-hash/               # SPIKE-M4
│   ├── m5-storage-encryption/        # SPIKE-M5
│   ├── m6-structured-clone/          # SPIKE-M6
│   ├── m7-abort-controller/          # SPIKE-M7
│   │
│   └── low/                           # SPIKE-L1 through L8 (optional)
│       ├── l1-evidence-size/
│       ├── l2-alarms-interval/
│       └── ...
│
└── reports/
    ├── wave-0-report.md
    ├── wave-1-report.md
    ├── go-nogo-recommendation.md      # Final recommendation after all spikes
    └── adr-status-review.md           # Updated ADR statuses
```

### 5.2 Spike Design Rules

1. **Every spike is independently loadable as an unpacked extension.** No spike depends on another spike's code.
2. **No shared production framework.** The `harness/` directory contains ONLY test infrastructure (message helpers, storage wrappers, capability probes). It must not evolve into a production library.
3. **Each spike has exactly one README.md** with: objective, success criteria, failure criteria, how to run, expected output, and a results section to be filled after execution.
4. **Spikes do NOT use shared state.** Each spike manages its own storage keys, its own message types, its own sandbox pages.
5. **Spikes are THROWN AWAY after validation.** They are not the foundation of the production codebase.
6. **Spikes target the lowest acceptable fidelity.** A spike that tests messaging doesn't need to verify C2PA manifests — it needs to verify that messages of the expected size and structure survive cross-context transit.

---

## Section 6: Validation Metrics

### 6.1 Performance Metrics

| Metric | Target | Measured In | Pass/Fail |
|--------|--------|-------------|-----------|
| C2PA first-load latency | < 2s | SPIKE-C1 | PASS if < 2s Chrome; WARN if 2-5s; FAIL if > 5s |
| C2PA warm-load latency | < 500ms | SPIKE-C1 | PASS if < 500ms; WARN if 500ms-1s; FAIL if > 1s |
| C2PA per-verification latency | < 200ms | SPIKE-C1 | PASS if < 200ms; FAIL if > 500ms |
| ONNX model first-load latency (10MB) | < 5s | SPIKE-C2 | PASS if < 5s; WARN if 5-10s; FAIL if > 10s |
| ONNX inference latency (512×512, WASM) | < 2s | SPIKE-C2 | PASS if < 2s; FAIL if > 5s |
| ONNX inference latency (512×512, WebGPU) | < 500ms | SPIKE-H2 | PASS if < 500ms; FAIL if > 1s |
| Content script sync budget | < 50ms | SPIKE-C4 | PASS if < 50ms; FAIL if > 50ms |
| First evidence to UI | < 500ms | SPIKE-C4 | PASS if < 500ms; WARN if 500ms-1s; FAIL if > 1s |
| Message round-trip CS→SW | < 10ms | SPIKE-C4 | PASS if < 10ms; FAIL if > 50ms |
| Capability detection latency | < 50ms | SPIKE-H5 | PASS if < 50ms; FAIL if > 100ms |

### 6.2 Memory Metrics

| Metric | Target | Measured In | Pass/Fail |
|--------|--------|-------------|-----------|
| ONNX model load peak memory (10MB model) | < 150MB | SPIKE-H1 | PASS if < 150MB; WARN if 150-300MB; FAIL if > 300MB |
| ONNX model load peak memory (50MB model) | < 300MB | SPIKE-H1 | PASS if < 300MB; FAIL if > 500MB or sandbox killed |
| ONNX model load peak memory (150MB model) | < 500MB | SPIKE-H1 | PASS if < 500MB; FAIL if sandbox killed |
| Sandbox memory after model eviction | Within 20% of pre-load | SPIKE-H1 | PASS if no leak; FAIL if > 20% growth after 10 load/evict cycles |
| Session storage per analysis session | < 2MB | SPIKE-H3 | PASS if < 2MB; WARN if 2-5MB; FAIL if > 5MB |
| IndexedDB available quota (standard device) | > 200MB | SPIKE-H4 | PASS if > 200MB; WARN if 100-200MB; FAIL if < 100MB |

### 6.3 Reliability Metrics

| Metric | Target | Measured In | Pass/Fail |
|--------|--------|-------------|-----------|
| SW 60-second survival with port | 100% of 20 trials | SPIKE-C3 | PASS if 100%; FAIL if any premature termination |
| SW 60-second survival without port (API calls) | 100% of 20 trials | SPIKE-C3 | PASS if 100%; FAIL if any premature termination |
| SW checkpoint-and-resume success | 95% of 20 trials | SPIKE-H6 | PASS if > 95%; WARN if 80-95%; FAIL if < 80% |
| Cross-context message integrity | 100% of 1000 messages | SPIKE-C4 | PASS if 100%; FAIL if any corruption detected |
| Offscreen doc 60-second survival | 100% of 20 trials | SPIKE-C6 | PASS if 100%; FAIL if any premature termination |
| IndexedDB blob round-trip integrity | 100% of 100 trials | SPIKE-C6 | PASS if 100%; FAIL if any hash mismatch |

### 6.4 Browser Compatibility Metrics

| Metric | Target | Measured In |
|--------|--------|-------------|
| Chrome all Critical spikes pass | 100% | All C-spikes |
| Edge all Critical spikes pass | 100% | All C-spikes |
| Firefox Critical spikes pass (with documented gaps) | C1, C2, C3, C4, C5, C7 pass. C6 documented as N/A (no offscreen). | All C-spikes |
| Safari Critical spikes pass (with documented gaps) | C1, C2, C4 pass. C3, C5, C6, C7 documented with constraints. | All C-spikes |

---

## Section 7: Exit Criteria

### 7.1 Mandatory Exit Criteria (Architecture Remains Frozen Until All Pass)

These must ALL pass before Phase 2 implementation begins. If any fails, architecture redesign is required.

| # | Criterion | Validated By |
|---|-----------|-------------|
| **EC-1** | c2pa-js loads and verifies C2PA manifests in MV3 sandbox page on Chrome and Firefox | SPIKE-C1 |
| **EC-2** | ONNX Runtime Web loads and executes inference in MV3 sandbox page on Chrome and Firefox | SPIKE-C2 |
| **EC-3** | SW survives 60+ second analysis sessions using keep-alive strategy on Chrome | SPIKE-C3 |
| **EC-4** | Cross-context messages of expected size and structure transmit reliably on Chrome and Firefox | SPIKE-C4 |
| **EC-5** | MessagePort supports persistent compute sessions (model stays loaded across requests) on Chrome | SPIKE-C5 |
| **EC-6** | Offscreen document serves as reliable IndexedDB proxy on Chrome | SPIKE-C6 |
| **EC-7** | Cross-origin image byte fetching works for metadata extraction on at least 60% of tested real-world image URLs | SPIKE-C7 |

### 7.2 Conditional Exit Criteria (Must Pass, But Architecture Can Absorb Failure With Modification)

| # | Criterion | Validated By | If Fails |
|---|-----------|-------------|----------|
| **EC-8** | ONNX model memory stays within browser limits for models up to 50MB | SPIKE-H1 | Reduce maximum model size. Accept that 150MB models require cloud fallback. |
| **EC-9** | Session storage supports at least 3 concurrent analysis sessions | SPIKE-H3 | Cap concurrent sessions at whatever limit is found. |
| **EC-10** | SW checkpoint-and-resume works in at least 80% of trials | SPIKE-H6 | Accept that analysis >5min is unreliable. Cap analysis tiers at 5min. |

### 7.3 Advisory Exit Criteria (Inform Architecture but Do Not Block)

| # | Criterion | Validated By |
|---|-----------|-------------|
| **EC-11** | WebGPU speedup is >2x over WASM | SPIKE-H2 |
| **EC-12** | IndexedDB quota > 200MB on standard devices | SPIKE-H4 |
| **EC-13** | Capability detection latency < 50ms | SPIKE-H5 |
| **EC-14** | Explainability templates produce coherent output for all tested edge cases | SPIKE-H7 |
| **EC-15** | Social media heuristics >80% accurate on current platform DOMs | SPIKE-M3 |

---

## Section 8: ADR Status Review

### 8.1 ADRs That Can Be Marked Accepted

These ADRs document architectural decisions that do NOT depend on spike validation. Their assumptions are either self-evident (design choices) or validated by existing knowledge (not requiring a spike).

| ADR # | Title | Current Status | Recommended Status | Rationale |
|-------|-------|---------------|-------------------|-----------|
| **ADR-001** | Extension Platform & Browser Context Strategy | Proposed | **Accepted** | Decision to use MV3, sandbox pages, and offscreen docs is validated by platform documentation. The specific behavior of these contexts is validated by spikes C1-C7. Architecture decision is sound; spike failures would change implementation, not the architectural choice. |
| **ADR-002** | Evidence Object Model | Proposed | **Accepted** | The 19-field interface is a design choice. Its serializability is validated by SPIKE-C4 and SPIKE-M6. The interface can be accepted as canonical; field adjustments from spike findings don't invalidate the ADR. |
| **ADR-003** | Analyzer Plugin API | Proposed | **Accepted** | The plugin manifest schema and lifecycle are design choices. Execution feasibility is validated by SPIKE-C1 (WASM plugins) and SPIKE-C2 (ONNX plugins). Sandboxing model is validated by SPIKE-C5 (MessagePort communication). |
| **ADR-007** | Storage Strategy | Proposed | **Accepted** | Storage backend allocation (session/local/IndexedDB) is a design choice. Feasibility of each backend is validated by SPIKE-C6 (IndexedDB), SPIKE-H3 (session), SPIKE-H4 (quota). The strategy is accepted; capacity numbers may be adjusted by spike results. |
| **ADR-008** | Browser Capability Layer | Proposed | **Accepted** | The CapabilityProvider abstraction is a design pattern. Its correctness is validated by SPIKE-H5 (detection performance). The abstraction itself doesn't depend on spike results. |
| **ADR-011** | Invocation Model | Proposed | **Accepted** | The unified AnalysisRequest and 5 invocation paths are design choices. Messaging feasibility is validated by SPIKE-C4. The model is accepted; permission requirements may be adjusted by spike results. |
| **ADR-012** | Explainability Engine | Proposed | **Accepted** | Three-tier template-based generation is a design choice. Template quality is validated by SPIKE-H7. The architecture is accepted; template coverage gaps are implementation details, not architectural flaws. |

### 8.2 ADRs That Remain Proposed Pending Spike Validation

These ADRs depend on assumptions that haven't been empirically validated. Their acceptance is contingent on spike results.

| ADR # | Title | Current Status | Blocking Spike(s) | What Must Be Proven |
|-------|-------|---------------|-------------------|---------------------|
| **ADR-004** | Fusion Engine Architecture | Proposed | SPIKE-C4, SPIKE-H7 | Bayesian fusion requires reliable evidence delivery (C4). Conflict detection logic requires coherent explanation output (H7). |
| **ADR-005** | Confidence Model | Proposed | SPIKE-C4, SPIKE-H1 | 8-dimension model requires complete evidence (C4). Level mapping logic must account for memory-constrained analyzers that can't run (H1). |
| **ADR-006** | Scheduler Design | Proposed | SPIKE-C3, SPIKE-C5, SPIKE-H1, SPIKE-H5 | Tiered execution model requires SW survival (C3), persistent compute (C5), memory budgeting (H1), and capability-aware scheduling (H5). |
| **ADR-009** | Cloud Privacy Architecture | Proposed | SPIKE-C7 | Cloud fallback decisions depend on what cannot be done locally. C7 determines the scope of "cannot be done locally." |
| **ADR-010** | Model Loading & Caching | Proposed | SPIKE-C6, SPIKE-H1, SPIKE-H4 | Model storage (C6), memory budget (H1), and quota (H4) determine loading strategy. |
| **ADR-013** | Dynamic Plugin Sandboxing | Proposed | SPIKE-C5 (foundation) | Depends on MessagePort compute model being viable (C5). Phase 3 ADR — can remain Proposed until Phase 3. |
| **ADR-014** | Enterprise Deployment Model | Proposed | None (Phase 3) | Phase 3 ADR. Can remain Proposed. |
| **ADR-015** | Cross-Device Sync | Proposed | None (Phase 3) | Phase 3 ADR. Can remain Proposed. |
| **ADR-016** | API Architecture | Proposed | None (Phase 3) | Phase 3 ADR. Can remain Proposed. |

---

## Appendix A: How to Run a Spike

Every spike follows this protocol:

1. **Read the spike README.** Understand the objective and success criteria.
2. **Load the spike as an unpacked extension** in Chrome, Firefox, Edge, and Safari (as specified in browser coverage).
3. **Follow the "How to Run" instructions** in the README.
4. **Record results** in the spike's `results.md`:
   - Pass/fail per success criterion
   - Quantitative measurements (latency, memory, throughput)
   - Qualitative observations (browser quirks, unexpected behavior)
   - Screenshots of any errors
5. **If the spike FAILS:** document the failure mode precisely. Do NOT attempt to fix it within the spike — spikes are validation, not problem-solving.
6. **If the spike PASSES:** document any caveats, edge cases, or browser-specific behaviors worth noting.
7. **Report results** at the next Go/No-Go gate.

---

## Appendix B: Spike Failure Protocol

If a Critical spike fails:

1. **DO NOT modify the spike to make it pass.** The spike is the test; the architecture is the subject.
2. **Document the failure mode** precisely: what was attempted, what failed, what error was observed.
3. **Assess impact:** does this failure require architecture redesign, or can it be absorbed with modification?
4. **Escalate to Architect.** A failing Critical spike triggers an architecture review.
5. **Propose alternatives** in a revised ADR or architecture amendment.
6. **Re-spike** the revised approach.

If a High spike fails:
1. Assess whether the architecture can absorb the failure (e.g., reduced concurrency, extended time budgets, browser-specific degradation).
2. If absorbable: document the constraint and proceed.
3. If not absorbable: escalate to Critical path — architecture redesign required.

---

*End of Validation Plan. No implementation begins until all Mandatory Exit Criteria are met. The architecture is frozen.*
