# Scheduler & Analyzer Plugin Architecture

> **Document Type:** Architecture Design
> **Phase:** 1 — Architecture
> **Status:** Draft
> **Owner:** Trust Engine Architect
> **Depends On:** BROWSER_CAPABILITY_ASSESSMENT.md (Evidence Gathering), TRUST_ENGINE_SPEC.md (Product Reasoning), Evidence Source Ecosystem Spec
> **Audience:** Implementor Agent, ML Researcher, Product Manager

---

## Part 1: Scheduler Architecture

The Scheduler is the single orchestrator of all evidence gathering. It receives an analysis request (page, single image, batch), resolves which analyzers are eligible, determines execution order respecting dependencies and browser constraints, enforces resource budgets, manages timeouts, and reports progress. No analyzer runs without the Scheduler's authorization.

### 1.1 Priority System

#### 1.1.1 Evidence Type Priority Hierarchy

Priority is determined by two independent axes combined into a single scheduling priority integer:

**Axis 1: Latency Class priority (lower = runs first)**

| Latency Class | Priority Value | Max Wall Time | Examples |
|---|---|---|---|
| `IMMEDIATE` | 10 | 100ms | Metadata extraction, hash lookup, cached domain reputation, content type classification |
| `FAST` | 20 | 2s | C2PA verification, text style/pattern analysis, EXIF/IPTC/XMP parsing |
| `STANDARD` | 30 | 10s | Face-swap detection, audio vocoder analysis, lightweight ONNX inference (generator fingerprints) |
| `SLOW` | 40 | 30s | Heavyweight ONNX models (CLIP-ViT), reverse image search, fact-check lookup, multi-image consistency |
| `BACKGROUND` | 50 | 5min | Full video frame-by-frame, deep audio forensics, cloud threat intelligence, batch image forensics |

**Axis 2: Evidence Signal Tier priority (lower = runs first)**

This mirrors the Product Reasoning Spec's evidence hierarchy:

| Signal Tier | Priority Bias | Rationale |
|---|---|---|
| Tier 1 — Ground Truth (C2PA, hardware attestation) | -5 | Cryptographic signals are deterministic and dominant. Must run before probabilistic detectors so their results are available for early-termination decisions. |
| Tier 2 — Strong Corroborating (cross-modal, frequency artifacts, watermarks) | 0 | Strong probabilistic signals. Run after ground truth, before weak signals. |
| Tier 3 — Moderate Corroborating (face-swap, audio artifacts, EXIF consistency) | +5 | Moderate signals. Run later — useful for refinement, not triage. |
| Tier 4 — Weak/Aggregate (text burstiness, domain reputation, community signals) | +10 | Weak signals. Run last. Never block or delay higher-tier signals. |

**Combined Priority:** `latency_class_priority + signal_tier_bias = execution_priority`

*Example:* C2PA verification = 20 (FAST) + (-5) (Tier 1) = **15**. Text burstiness = 20 (FAST) + 10 (Tier 4) = **30**.
*Both are FAST, but C2PA runs first because its signal carries more weight.*

Lower `execution_priority` values run first.

#### 1.1.2 Invocation-Driven Priority Paths

The Scheduler supports three invocation models, each with a different priority profile:

**Path A: On-Demand (User clicks "Analyze this page")**

- ALL tiers run (local first, cloud opt-in)
- IMMEDIATE and FAST analyzers run immediately in parallel
- STANDARD and SLOW analyzers queued after FAST results arrive
- Progressive display: Tier 1 (provenance) → Tier 2 (forensics) → Tier 3 (deep)
- Early termination allowed (see Section 1.1.4)
- User sees a countdown: "Analyzing... (quick results in ~300ms, full results in ~5s)"

**Path B: Per-Domain Automatic (Background on allowlisted domains)**

- Only IMMEDIATE and FAST analyzers run automatically
- STANDARD, SLOW, BACKGROUND analyzers are **never started** until user invokes
- C2PA verification deferred to after page load (not on critical path)
- Results silently cached for display when user opens the panel
- Priority: lower than on-demand if both are active. On-demand analysis preempts background work.

**Path C: Content Menu ("Analyze this image" / "Analyze this selection")**

- Single target (one image OR one text block)
- ALL applicable analyzers for that single target run at elevated priority
- No tier filtering — user explicitly requested deep analysis on this item
- Priority: highest of all paths. Preempts any background work immediately.

**Path D: Always-On Passive (Phase 3+, enterprise)**

- Only IMMEDIATE analyzers (cached domain reputation, content type detection)
- NEVER pixel forensics, NEVER model inference, NEVER network requests
- Results purely informational; do not produce confidence assessments
- Priority: lowest. Must yield to all other scheduling work.

#### 1.1.3 Progressive Result Display Guarantees

The Scheduler guarantees the following display milestones, aligned with the Architecture Spec's progressive analysis model:

| Milestone | By When | What's Available | What's Still Running | UI Shows |
|---|---|---|---|---|
| **Initial** | <500ms after analysis start | Domain classification, content type, content volume, cached domain reputation, "no C2PA found" signals | C2PA verification (WASM loading), metadata extraction | "Domain: verified. Content: article with 3 images. Checking provenance..." |
| **Refined** | 1-5s after analysis start | C2PA results, EXIF/IPTC metadata, text style analysis, image format/dimensions, cloud domain reputation (if enabled) | Image forensics, model inference, cloud lookups | "C2PA: verified. EXIF: 2 of 3 images have metadata. Running deeper checks on 1 image..." |
| **Deep** | 5-30s (user-invoked) | Image forensics, AI generator fingerprints, video frame analysis, audio analysis, cross-modal | Cloud lookups, community signals, reverse image search | "Image analysis complete. ELA found no anomalies. Generator fingerprint: no match. Reverse image search: in progress..." |
| **Final** | After all eligible analyzers complete | All evidence gathered | None (quiesced) | Full explainability panel |

The Scheduler **must deliver the Initial milestone before any FAST or slower analyzer begins**. This is non-negotiable for UX. The user must never see a blank screen.

#### 1.1.4 Priority Promotion/Demotion Based on Partial Results

The Scheduler can adjust downstream priorities based on intermediate results:

**Promotion triggers (escalate priority):**
- C2PA returns `INCONCLUSIVE` → promote image forensics analyzers from STANDARD to run sooner (user needs answers)
- Domain reputation returns negative → promote ALL analyzers for this page (suspicious source warrants thorough check)
- Text burstiness returns anomalous → promote cross-modal and image analyzers (text may be covering for fake images)
- Three analyzers within same tier return conflicting results → promote deeper analyzers (conflict requires resolution)

**Demotion triggers (lower priority):**
- C2PA returns `AUTHENTIC` with confidence ≥ 0.95 AND signer is trusted → demote all image forensics (provenance won)
- Domain reputation returns strong positive AND all images have valid C2PA → demote deep forensics (all signals agree on authentic)
- Text analyzer finds nothing anomalous → demote cross-modal analysis (no reason to suspect mismatch)

**Early termination triggers (skip remaining analyzers):**
- C2PA hardware-attested provenance from recognized camera → skip all content forensics for that image (absolute ground truth)
- Hash-level match to known AI content → skip all other analyzers (deterministic evidence)
- User has explicitly allowlisted this domain → skip everything (user signal overrides all)
- **Anti-pattern guard:** High confidence but high signal disagreement must NOT trigger early termination. If two Tier 2 analyzers conflict at >80% confidence each, the system must run MORE analyzers, not fewer.

### 1.2 Parallel Execution

#### 1.2.1 Concurrency Rules

Analyzers are grouped into execution contexts. Each context has a maximum concurrency limit enforced by the browser:

| Execution Context | Max Concurrent | Browser Constraint | Affected Analyzers |
|---|---|---|---|
| **Content Script (main world)** | 1 (synchronous only) | Must execute on main thread. No parallel work. | DOM text extraction, image enumeration, element detection |
| **Content Script (isolated world)** | Unlimited (logical) | Extension's own JS context. Not main-thread-blocking. | Metadata parsing triggered from content script via fetch |
| **Service Worker** | 1 logical thread | Single-threaded event loop. Async I/O allows concurrent network requests. | Message routing, cache lookups, orchestration logic, cloud API fetches |
| **Offscreen Document** | **1 total** | Chrome/Edge: one offscreen document at a time. Firefox/Safari: N/A. Use sandboxed page fallback. | All WASM inference, ONNX model execution, pixel forensics |
| **Sandboxed Page** | 1 per page instance | Separate origin, no extension API access. Can have multiple iframes but only one is practical. | WASM computation (Firefox/Safari fallback when offscreen unavailable) |
| **tabCapture Stream** | **1 total** | Chrome/Edge: one active capture per extension. User gesture required. | Video frame analysis, audio capture |
| **Network (Cloud APIs)** | 6 concurrent fetches | Browser-wide HTTP connection limit (~6 per origin, ~10 total). Extensions share this pool. | Reverse image search, fact-check lookup, cloud reputation, community signals |

#### 1.2.2 Execution Groups (What Can Run Concurrently)

The Scheduler forms execution groups from the dependency graph:

**Group 0 — Synchronous (content script, main world, <50ms budget):**
- Content type detection (DOM landmark scan)
- Page URL extraction
- Basic element enumeration (count, types)
- **Execution:** Sequential (single-threaded), must complete within 50ms

**Group 1 — Independent IMMEDIATE (parallel, no shared context):**
- Cached domain reputation lookup (service worker, <5ms)
- `<img>` enumeration + `src` collection (content script, <30ms)
- DOM text extraction from main content (content script, <50ms)
- **Execution:** Parallel. These run in different contexts (SW + content script) or non-overlapping DOM operations.

**Group 2 — Independent FAST (parallel, offscreen document serialized):**
- EXIF/IPTC/XMP metadata parsing (offscreen or content script WASM, per-image)
- C2PA verification (offscreen WASM, per-image)
- Text style/pattern analysis (content script or dedicated JS, one text block)
- **Execution:** Metadata parsing and C2PA verification both require offscreen document for WASM. These must be **serialized** through the single offscreen document. Multiple images are batched: the offscreen document processes one image at a time (metadata → C2PA for image 1, then metadata → C2PA for image 2). Text analysis runs concurrently in content script context.

**Group 3 — Independent STANDARD (serialized through offscreen):**
- AI generator fingerprint detection (ONNX, offscreen, per-image)
- ELA analysis (offscreen, per-image)
- Noise pattern analysis (offscreen, per-image)
- Frequency domain analysis (offscreen, per-image)
- Watermark detection (offscreen, per-image)
- All serialize through the single offscreen document. For multiple images: process image by image, but within each image run independent analyzers concurrently if the offscreen document supports Web Workers or the models can share the ONNX runtime.

**Group 4 — Browser-Dependent (requires tabCapture, serialized):**
- Video frame analysis
- Audio analysis
- **Execution:** Sequential. Only one tabCapture stream. Audio and video capture from same stream if both requested.

**Group 5 — Cloud (parallel fetches, limited by browser connection pool):**
- Reverse image search
- Real-time domain reputation lookup
- Fact-check database lookup
- Community signal aggregation
- **Execution:** Parallel fetches (up to 6). Each is independent.

#### 1.2.3 Dependency Model

Dependencies are modeled as a **Directed Acyclic Graph (DAG)** where nodes are analyzers and edges are data dependencies.

| Dependency Type | Semantics | Example |
|---|---|---|
| **HARD_REQUIRES** | Dependent analyzer MUST NOT run until dependency completes with SUCCESS. If dependency fails/times out, dependent is skipped. | Face-swap detection HARD_REQUIRES video frame extraction. If frame extraction fails, face-swap cannot run. |
| **SOFT_REQUIRES** | Dependent analyzer prefers dependency's output but can run with partial/degraded input if dependency fails. | C2PA verification SOFT_REQUIRES metadata extraction (needs to locate JUMBF box). If metadata extraction fails, C2PA can still attempt to parse the raw bytes — may be slower or less reliable but can still function. |
| **ENRICHES** | Dependent analyzer produces better results with dependency's output but can produce useful results without it. | Cross-modal analysis ENRICHES from text analysis + image analysis. Can run with either alone but produces best results with both. |
| **PREFERS_ORDER** | No data dependency, but the dependency should complete first for UX/priority reasons. Does not block execution. | Domain reputation PREFERS_ORDER before cloud lookups (if domain is known-bad, skip expensive cloud analysis). |

**DAG validation at registration time:**
- Circular dependencies are rejected with a cycle trace
- Missing dependencies (referencing a non-existent analyzer) are rejected
- A dependency chain deeper than 5 levels is flagged as a warning (likely architectural issue)

#### 1.2.4 Maximum Concurrency Limits

| Resource | Hard Limit | Soft Limit (Recommended) | Enforcement |
|---|---|---|---|
| Total concurrent analyzers | 20 | 8 | Soft: Scheduler stops dequeuing new work when 8 analyzers are RUNNING. Hard: never exceed 20 (browser becomes unresponsive). |
| Content script DOM operations | 2 | 1 | Two DOM mutations at once risk layout thrashing. Serialize DOM reads. DOM writes (mutations) must be atomic. |
| Offscreen document operations | 1 | 1 | Only one WASM/ONNX inference at a time. Queue additional requests. |
| tabCapture streams | 1 | 1 | Browser-enforced. Second capture request will fail. |
| Concurrent cloud fetches | 6 | 4 | Leave headroom for browser's own requests. |
| Models loaded in memory | Unlimited (practical: 3) | 2 | Each ONNX model = 5-200 MB. Loading a third model may require evicting a cached model. |

### 1.3 Resource Budgeting

#### 1.3.1 CPU Budget Per Analysis Session

| Invocation Model | Total CPU Budget | Budget Window | Enforcement |
|---|---|---|---|
| On-demand | 30s cumulative main-thread + offscreen CPU time | Per analysis session | Soft cap: warn in UI if exceeded. Hard cap: terminate at 60s. |
| Per-domain automatic | 5s cumulative CPU time | Per page load | Hard cap. Exceeding 5s means automatic analysis is too expensive for this page — cancel remaining work. |
| Always-on passive | 500ms cumulative CPU time | Per page load | Hard cap. Must never impact page interactivity. |
| Content menu (single image) | 15s cumulative CPU time | Per invocation | Soft cap. User expects deep analysis on a single target. |

**CPU time is measured as:**
- Main thread blocking time (in content script or service worker event loop)
- Offscreen document execution time
- WASM execution time (wall clock * thread count)

**Enforcement mechanism:** The Scheduler wraps every analyzer execution in a timing measurement. After each analyzer completes, the Scheduler checks cumulative CPU time against the budget. If approaching budget (80% consumed), the Scheduler stops dequeuing new analyzers and allows in-flight analyzers to complete. If budget exceeded, in-flight analyzers are cancelled.

#### 1.3.2 Memory Budget Per Analysis

| Component | Budget | Monitoring | OOM Response |
|---|---|---|---|
| Evidence results cache | 5 MB | Track serialized size of all EvidenceResult objects in active session | Evict oldest results from non-critical analyzers (Tier 4 first) |
| Model weights (loaded) | 150 MB | Track sum of loaded model sizes via plugin manifests | Unload least-recently-used model. Reload on next use. |
| Intermediate analysis data | 50 MB | Image pixel buffers, decoded video frames, extracted audio samples | Discard intermediate data from completed analyzers. Stream large data instead of buffering. |
| Offscreen document heap | 200 MB | Browser-reported JS heap size via `performance.memory` (Chrome only, non-standard) | Terminate offscreen document and recreate. Reload currently needed models. |
| Total extension memory | 300 MB peak | Sum of all components | Hard ceiling. If approaching 300 MB, cancel lowest-priority analysis, unload models, force garbage collection hint. |

#### 1.3.3 Cost-Based Early Termination

The Scheduler makes a cost/benefit decision before starting each new analyzer:

```
ShouldRun(analyzer) = ExpectedValue(analyzer) > ExecutionCost(analyzer)

Where:
  ExpectedValue = ExpectedInformationGain × RemainingSignalWeight
  ExecutionCost = EstimatedCPU_ms × ResourceMultiplier × BudgetPressure
```

**ExpectedInformationGain:** Based on evidence tier and whether similar evidence already exists. If three image forensics analyzers already ran and agreed, a fourth offers diminishing marginal information.

**RemainingSignalWeight:** The weight this signal would have in the Bayesian fusion given signals already gathered. If C2PA returned dominant proof, remaining signal weight is near zero.

**BudgetPressure:** As CPU/memory budget is consumed, this multiplier increases → harder to justify new analyzers.

**Decision rules:**
- If `ExpectedValue < 0.2 × ExecutionCost`: skip analyzer
- If budget is 80% consumed: only run Tier 1 and Tier 2 analyzers
- If user is actively waiting (on-demand): be more aggressive (allow 1.5x budget)
- If this is background/passive: be conservative (0.5x budget)

#### 1.3.4 Per-Analyzer Memory Limits

Each analyzer declares `estimated_memory_mb` in its plugin manifest. The Scheduler:

1. Before starting an analyzer, checks: `current_loaded_models_mb + new_model_mb <= 150 MB`
2. If the new model would exceed the limit, the Scheduler evaluates whether to evict a currently loaded model:
   - Eviction candidates ranked by: (time since last use) × (1 / signal_tier)
   - The least-recently-used, lowest-tier model is evicted first
   - The evicted model's compiled WASM is retained in `chrome.storage.session` (if fits in ~10 MB) for faster reload
3. If the analyzer needs 200 MB but only 150 MB is available AND no model can be evicted (e.g., all loaded models are actively in use):
   - The analyzer is queued until a model is freed
   - If the queue wait would exceed the analyzer's timeout, the analyzer is **SKIPPED** with reason: `INSUFFICIENT_MEMORY`
   - The UI shows: "Deep image analysis skipped — insufficient device memory. Try closing other tabs."

### 1.4 Timeout Management

#### 1.4.1 Per-Analyzer Timeout Contract

Each analyzer declares a latency class. The Scheduler enforces a hard timeout based on that class:

| Latency Class | Declared Timeout | Scheduler-Enforced Hard Timeout | Notes |
|---|---|---|---|
| `IMMEDIATE` | <100ms | 200ms | Double the declared ceiling as buffer for measurement overhead. |
| `FAST` | <2s | 4s | Account for WASM cold-start latency. |
| `STANDARD` | <10s | 15s | Allow for browser scheduling jitter under load. |
| `SLOW` | <30s | 45s | Long-running computation. Signal to user that this is expected to take time. |
| `BACKGROUND` | <5min | 5min (SW hard limit) | Service worker will be killed at 5 minutes regardless. Must produce partial results before this. |

**Timeout enforcement mechanism:**
- The Scheduler wraps every analyzer execution in an `AbortSignal` with a timeout
- When timeout fires, the analyzer's `AbortSignal` is triggered
- The analyzer is expected to catch the abort, produce a partial result if possible, and return
- If the analyzer does not respond within 2x its hard timeout (grace period), the Scheduler terminates the execution context entirely (closes offscreen document, revokes tabCapture, aborts fetch)

#### 1.4.2 Timeout Exceeded Behavior

| Condition | Scheduler Action | Analyzer Result | UI Display |
|---|---|---|---|
| Analyzer self-terminates at hard timeout, produces partial result | Accept partial result with reduced confidence | `signal_value` + `error.classification: PARTIAL_RESULT` | "Analysis partially complete — timed out before finishing. Results may be less reliable." |
| Analyzer exceeds hard timeout, does not self-terminate | Scheduler forcefully terminates execution context | `signal_value: ERROR` with `error.classification: TIMEOUT` | "Analysis timed out. This check was skipped." |
| Analyzer exceeds hard timeout on >3 consecutive invocations | Scheduler downgrades the analyzer's effective latency class by one tier (FAST→STANDARD, STANDARD→SLOW) | The analyzer now gets more time budget but runs later in the schedule | No direct user notification. Scheduler internally adjusts. |
| Analyzer exceeds hard timeout on >20% of all invocations (rolling window of 50) | Scheduler marks analyzer DEGRADED. Increased timeout buffer applied. | As above. | If user opens advanced settings: "This analyzer frequently times out. Consider disabling it." |

#### 1.4.3 Service Worker 30-Second Idle Timeout — Keep-Alive Strategy

The MV3 service worker terminates after 30 seconds of inactivity. For analysis chains exceeding 30 seconds:

**Detection:** The Scheduler tracks idle time since the last message received. At 20 seconds of idle time (10s before termination), the Scheduler sends a keep-alive ping.

**Keep-alive mechanisms (in descending order of reliability):**

| Mechanism | How It Works | Reliability | Constraints |
|---|---|---|---|
| **Port-based messaging** | Open a `chrome.runtime.connect()` long-lived port between the offscreen document and the service worker. The port keeps the SW alive. | **High.** A connected port prevents SW termination. | Port can be disconnected if offscreen document is terminated. |
| **`chrome.alarms` periodic pulse** | Set an alarm every 25 seconds. Alarm wake event resets the idle timer. | **Medium.** Chrome: minimum 30s alarm interval (close to the 30s idle limit — race condition risk). Firefox: minimum 1s alarm interval (safer). | Does not work on Safari (no alarms API). |
| **`fetch` keep-alive request** | Service worker makes a `fetch()` to a local extension resource or a no-op endpoint. Outbound request keeps SW alive. | **Low.** Requires a pending fetch — completed fetches don't reset the timer. Must use streaming/chunked responses with delays between chunks. | Fragile. Network errors, browser throttling. |

**Recommended strategy:**
- **Primary:** Open a `chrome.runtime.connect()` port from the offscreen document to the SW at the start of any analysis session expected to exceed 30 seconds. Close the port when analysis completes.
- **Fallback:** On Firefox (no offscreen, uses sandboxed page), open a port from the sandboxed page to the SW.
- **Safari:** Safari has persistent background scripts (no 30s limit). No keep-alive needed.
- **If the port disconnects:** The SW has ~10 seconds to spin up a new connection before termination. If the offscreen document was terminated, the SW restores it and reconnects.

#### 1.4.4 Service Worker 5-Minute Hard Limit

Chrome MV3 enforces a 5-minute maximum for any single service worker operation. For analysis that legitimately exceeds this:

**Strategy A — Checkpoint & Resume (preferred):**
1. At 4 minutes 30 seconds, the Scheduler checkpoints all in-progress analysis state to `chrome.storage.session` (or `chrome.storage.local` for larger state).
2. The Scheduler returns partial results for all running analyzers that haven't completed.
3. The SW allows itself to be terminated.
4. On next SW wake (triggered by `chrome.alarms` or user interaction), the Scheduler reads the checkpoint and resumes where it left off.
5. Completed analyzers are not re-run. In-progress analyzers restart from the beginning (idempotency design: analyzers must be restartable).

**Strategy B — Split into sub-5-minute sessions (for BACKGROUND-class analysis):**
- Large batch analysis jobs (e.g., analyzing 100 images on a gallery page) are split into batches of ~10, each completing within 5 minutes
- After each batch, checkpoint and intentionally terminate the SW
- Schedule next batch via `chrome.alarms` (minimum 30s on Chrome; use a visible user prompt: "Analysis paused. Click to continue.")

**Strategy C — User prompts (for on-demand analysis):**
- If a single on-demand analysis exceeds 4 minutes, pause and prompt: "This analysis is taking longer than expected. Continue? [Yes] [Stop and show partial results]"
- User's click provides the event to restart the SW

**What happens if the SW is killed mid-analysis without checkpoint:**
- All in-flight analyzer results are lost
- The Scheduler detects the SW restart (no active analysis session in memory)
- The analysis session is marked `INTERRUPTED` in storage
- On next user interaction with the extension, the Scheduler prompts: "Your previous analysis was interrupted. Restart? [Yes] [Dismiss]"
- If user restarts, all analyzers re-run from scratch (checkpointing would have prevented this)

### 1.5 Browser Capability Detection

#### 1.5.1 Detection Architecture

Capability detection runs at **extension startup** (service worker activation) and is cached for the lifetime of the service worker. If the SW is terminated and restarted, detection runs again.

**Two-phase detection:**

**Phase 1: Static Manifest Inspection (runs once at SW startup, <1ms)**

Based on `navigator.userAgent` and browser API presence:

| Capability | Detection Method | Chrome | Edge | Firefox | Safari |
|---|---|---|---|---|---|
| `offscreen` | `typeof chrome.offscreen !== 'undefined'` | ✓ | ✓ | ✗ | ✗ |
| `tabCapture` | `typeof chrome.tabCapture !== 'undefined'` | ✓ | ✓ | ✗ | ✗ |
| `sidePanel` | `typeof chrome.sidePanel !== 'undefined'` | ✓ | ✓ | ✗ | ✗ |
| `storage.session` | `typeof chrome.storage.session !== 'undefined'` | ✓ | ✓ | ✗ | ✗ |
| `declarativeNetRequest` | `typeof chrome.declarativeNetRequest !== 'undefined'` | ✓ | ✓ | ✓ | ✓ |
| `alarms` | `typeof chrome.alarms !== 'undefined'` | ✓ (min 30s) | ✓ (min 30s) | ✓ (min 1s) | ✗ |
| `userScripts` (World Isolation) | `typeof chrome.userScripts !== 'undefined'` | ✓ | ✓ | ✓ | ✗ |

**Phase 2: Runtime Probing (runs once at SW startup OR on first use, ~10-100ms)**

| Capability | Probing Method | Reason for Probing |
|---|---|---|
| **WebGPU** | Attempt `navigator.gpu.requestAdapter()` in the offscreen document | `navigator.gpu` may exist but fail to create an adapter (driver issues, disabled flag). Firefox: exists behind `dom.webgpu.enabled` flag. Safari: not present. |
| **WebAssembly (threading)** | Attempt to compile a minimal WASM module with shared memory | Some browsers support WASM but not SharedArrayBuffer (required for pthreads). ONNX Runtime Web may fall back to single-threaded WASM. |
| **`createImageBitmap`** | Attempt `createImageBitmap()` with a minimal Blob | Required for off-main-thread image decoding. Present in all modern browsers but verify for graceful degradation. |
| **`AudioContext`** | Attempt `new AudioContext()` | Required for audio analysis. May be suspended until user gesture in some browsers. |
| **`OffscreenCanvas`** | Attempt `new OffscreenCanvas(1, 1)` | Required for off-main-thread pixel analysis. Firefox: supported. Safari: supported from 16.4. |
| **`VideoFrame` API** | Check `typeof VideoFrame !== 'undefined'` | Required for efficient video frame extraction from tabCapture. Chrome/Edge only. |
| **Chrome built-in AI** | Attempt `ai.languageModel.create()` or equivalent | Experimental. Chrome-only. May not be available even on Chrome if user hasn't enabled it. |

**Capability Profile Construction:**

The Scheduler builds a `BrowserCapabilityProfile` from both phases:

```
{
  browser: "chrome" | "edge" | "firefox" | "safari",
  browser_version: "131.0.0",
  supports: {
    offscreen_document: true | false,
    tab_capture: true | false,
    webgpu: true | false,
    wasm_threading: true | false,
    offscreen_canvas: true | false,
    video_frame_api: true | false,
    chrome_builtin_ai: true | false,
    storage_session: true | false,
    alarms_api: true | false,
    alarms_min_interval_seconds: 30 | 1 | null,
    background_lifetime: "service_worker_30s" | "event_page" | "persistent",
    max_offscreen_documents: 1 | 0,
    max_tabcapture_streams: 1 | 0
  },
  confidence_profile: "full" | "moderate" | "reduced"
}
```

#### 1.5.2 Scheduling Decisions Based on Capability Profile

**Before scheduling any analyzer, the Scheduler checks:**
1. Does the analyzer's `required_apis` list intersect with the capability profile's unsupported APIs? → If yes, mark analyzer `SKIPPED: UNSUPPORTED_BROWSER`
2. Does the analyzer's `supported_browsers` list include the current browser? → If no, mark `SKIPPED: UNSUPPORTED_BROWSER`

**Examples:**
- tabCapture analyzers on Firefox → `SKIPPED: UNSUPPORTED_BROWSER`. UI: "Video analysis unavailable in Firefox. Try Chrome or Edge."
- WebGPU-accelerated analyzers on Safari → run in WASM fallback mode with longer timeout. Analyzer declares both `required_apis: ["webgpu"]` AND `fallback_apis: ["wasm"]`.
- Offscreen document on Firefox → use sandboxed page instead. Behaviorally equivalent but less performant. Scheduler routes to sandboxed page execution context.

#### 1.5.3 Capability Change Detection

Browser capabilities can change at runtime (user enables a flag, browser update, extension update). The capability profile is **not** continuously polled — it's recomputed:

1. On service worker start (extension install, update, browser restart)
2. On `chrome.runtime.onInstalled` (extension update)
3. If an analyzer fails with an unexpected `UNSUPPORTED` error, the capability profile is re-probed for that specific capability

### 1.6 Offline Detection

#### 1.6.1 Detection Methods

| Method | Latency | Reliability | Notes |
|---|---|---|---|
| `navigator.onLine` | Instant | **Low.** Reports `true` even when behind a captive portal or when DNS is unavailable. Only reliable for complete disconnection (airplane mode, no network interface). | Use as a quick optimistic check. Never trust a negative (`false`) but be skeptical of a positive (`true`). |
| **Failed fetch probe** | Variable (uses cached timeout) | **Medium.** A recent failed `fetch()` is strong evidence of offline. But "recent" has a decay window. | Track the last 3 fetch failures. If 2+ failed in the last 30 seconds, treat as likely offline. |
| **Active health check** | 1-3s (timeout) | **High.** `fetch("https://clients3.google.com/generate_204")` with a 3s timeout. If it fails, confident offline. If it succeeds, confident online. | Run on-demand before first cloud analyzer. Cache result for 60 seconds. |

**Offline state determination algorithm:**
1. If `navigator.onLine === false` → **offline** (trust the negative)
2. If `navigator.onLine === true` AND recent fetch failures < 2 in 30s → **assume online** until proven otherwise
3. If `navigator.onLine === true` AND recent fetch failures ≥ 2 in 30s → **probe** with active health check
4. If first cloud analyzer fails with network error → **mark offline**, probe again in 60 seconds

#### 1.6.2 Scheduling Adjustments When Offline

The Scheduler maintains two routing tables: one for online, one for offline.

| Analyzer Type | Online Behavior | Offline Behavior |
|---|---|---|
| Local-only (C2PA, EXIF, text style, image forensics, ONNX models) | Run normally | Run normally — no impact |
| Locally cached reference data (domain reputation, known hashes, publisher credibility) | Run with fresh data | Run with cached data, weight decay applied based on data staleness |
| Cloud API (reverse image search, fact-check, community signals) | Run (if opted in) | **Skip.** Mark as `SKIPPED: OFFLINE` |
| Real-time domain reputation | Run | **Skip.** Falls back to cached domain reputation if available |
| SynthID detection | Run (requires Google API) | **Skip.** Mark as `SKIPPED: OFFLINE` |

**Priority adjustment when offline:**
- All local analyzers retain their normal priority
- Cloud analyzers are removed from the execution queue entirely (not just demoted)
- Text and metadata analyzers are promoted (they're now the only signals available, so their results matter more)
- The Scheduler immediately reports: "Offline mode — cloud verification unavailable. Local analysis in progress."

**On reconnection:**
- The Scheduler detects reconnection via `navigator.onLine` change event or successful active health check
- The current analysis session is tagged as "offline" for its lifetime (results are not retroactively modified)
- The UI shows: "You're back online! Re-run analysis for cloud verification? [Re-analyze] [Dismiss]"
- If user clicks "Re-analyze," only the skipped cloud analyzers run (local results are cached from the offline run)

#### 1.6.3 Cached Reference Data vs. Real-Time Cloud Sources

| Aspect | Cached Reference Data | Real-Time Cloud Source |
|---|---|---|
| Stored in | `chrome.storage.local` (persistent), `chrome.storage.session` (memory cache) | N/A — fetched on each request |
| Freshness | Versioned bundles. Each bundle has a `generated_at` timestamp and `expires_at` timestamp. | Always fresh (real-time query) |
| Staleness handling | Weight decay proportional to age: `effective_confidence = declared_confidence × max(0.5, 1 - (age/max_age))` | N/A |
| Offline behavior | Used with decay. Displayed with "Data last updated: 3 days ago." | Skipped entirely. |
| Privacy | Fully local. No data leaves device. | Domain/claim hash may be transmitted. User must opt in. |
| Failure mode | If bundle is corrupted (hash mismatch), revert to previous bundle. If no previous bundle, skip. | If fetch fails, skip. Retry on next page load (not immediately). |

### 1.7 Retry & Backpressure

#### 1.7.1 Failure Classification & Retry Policy

| Failure Mode | Retry? | Max Retries | Backoff | Rationale |
|---|---|---|---|---|
| **Network timeout** (fetch to cloud API timed out) | **Yes** | 2 retries | Exponential: 1s, 4s | Transient. Network congestion, server temporarily slow. |
| **Network error** (DNS failure, connection refused, HTTP 5xx) | **Yes** | 1 retry | 2s | May be transient (server restart). If 5xx persists, it's a server issue — don't hammer it. |
| **HTTP 4xx** (400, 401, 403, 404, 429) | **No** | 0 | N/A | Client error. Retrying won't help. 429 (rate limited) may be retried after a longer backoff (declared in Retry-After header). |
| **Model load failure** (WASM compilation error, ONNX model corrupted) | **No** | 0 | N/A | Persistent. Model file is likely corrupted on disk. Requires re-download of model bundle, which is a background task, not a per-request retry. Mark analyzer `UNAVAILABLE` until model is re-downloaded. |
| **CORS block** (cross-origin image fetch rejected) | **No** | 0 | N/A | Persistent. Server policy. Retrying won't change the CORS headers. Mark analyzer `SKIPPED: CORS_BLOCKED` for this specific image. |
| **OOM / out of memory** | **Yes** | 1 retry | Immediate (after evicting models) | The Scheduler evicts the LRU model and retries immediately. If fails again, mark `SKIPPED: INSUFFICIENT_MEMORY`. |
| **Internal analyzer error** (uncaught exception, assertion failure) | **No** | 0 | N/A | Bug in analyzer code. Should not retry — same input will produce same crash. Log error for debugging. Mark analyzer `FAILED: INTERNAL_ERROR`. |
| **Analyzer not applicable** (wrong content type, unsupported format) | **No** | 0 | N/A | Routing error. Should not have been scheduled. Mark `SKIPPED: UNSUPPORTED_CONTENT`. Feed back to routing table to prevent future mis-scheduling. |
| **Service worker termination** (mid-analysis SW kill) | **Yes** | 1 resume attempt | On next SW wake | On SW restart, read checkpoint and resume. If no checkpoint exists, restart analysis from beginning. If a second termination occurs, mark analysis `INTERRUPTED` and prompt user. |

#### 1.7.2 Backpressure and Navigation-Aware Cancellation

**Backpressure triggers (cancel pending work):**

| Trigger | Action |
|---|---|
| **User navigates away from analyzed tab** (tab URL changes or tab closes) | Cancel all in-flight and queued analyzers for that tab. Content script analyzers are immediately invalid (DOM is gone). Cloud fetches may complete but results are discarded. |
| **User starts new analysis while previous is running** | Previous analysis is cancelled (unless explicitly a batch job). New analysis takes priority. |
| **Service worker memory pressure** | The Scheduler monitors for memory pressure signals (if available). If memory is critically low, cancel lowest-priority analyzers, unload models, force GC hint. |
| **Browser is idle** (user hasn't interacted in 30s) | Downgrade priority of analysis. Only complete the current in-flight analyzer, don't start new ones. Resume when user interacts. |
| **Queue depth exceeds 20 pending analyzers** | Apply backpressure: stop accepting new analysis requests. Return "Analysis queue full. Please wait for current analysis to complete." |

**What happens to in-flight analysis when user closes tab:**

1. The `tabs.onRemoved` event fires in the service worker
2. The Scheduler looks up all analyzers associated with that `tabId`
3. For analyzers in content script context: already dead (content script destroyed with tab)
4. For analyzers in offscreen document: send cancel signal. Offscreen document continues existing (shared resource) but the specific analysis task is cancelled
5. For cloud fetches: abort the `AbortController`. If the request already went out, the response is discarded when it arrives
6. For tabCapture: stream is terminated by the browser when the tab closes. Scheduler cleans up the reference.
7. All partial results for that tab are discarded

#### 1.7.3 Retry Backoff Strategy

The Scheduler implements **exponential backoff with jitter** for retryable failures:

```
delay = min(cap, base_delay × 2^attempt) × (0.5 + random(0, 1))
```

| Failure Type | Base Delay | Cap |
|---|---|---|
| Network timeout | 1s | 8s |
| Network error (5xx) | 2s | 8s |
| OOM retry | 500ms | 1s (one retry only) |
| Rate limit (429) | Value from Retry-After header, or 5s default | 30s |

**Maximum retry budget per analysis session:** The combined retry delay for all analyzers in a session must not exceed 30 seconds. After 30 seconds of cumulative retry waiting, remaining retries are skipped and analyzers are marked `FAILED: MAX_RETRIES_EXCEEDED`.

### 1.8 Cancellation

#### 1.8.1 Cancellation Architecture

The Scheduler uses `AbortController` / `AbortSignal` as the universal cancellation primitive.

**Cancellation sources:**

| Source | Trigger | Scope |
|---|---|---|
| **User cancel** | User clicks "Cancel" in the analysis UI | Entire analysis session (all analyzers, all tabs) |
| **Tab close** | `tabs.onRemoved` | All analyzers for that specific `tabId` |
| **Timeout** | Per-analyzer timeout exceeded | Single analyzer |
| **Budget exceeded** | CPU or memory budget exhausted | Remaining queued analyzers + optionally in-flight |
| **Navigation** | `webNavigation.onCommitted` with transition type `link`/`typed`/`auto_bookmark` | All analyzers for that `tabId` |

#### 1.8.2 Cancel Signal Propagation

The Scheduler establishes a **cancellation tree**:

```
Session Cancel (root AbortController)
├── Tab-level Cancel (one per tab being analyzed)
│   ├── Analyzer A Cancel (derived from tab + own timeout)
│   ├── Analyzer B Cancel (derived from tab + own timeout)
│   └── Analyzer C Cancel (derived from tab + own timeout)
└── Tab-level Cancel (another tab)
    └── Analyzer D Cancel
```

- **Session cancel:** The root `AbortController.abort()` is called. All derived signals fire.
- **Tab-level cancel:** Only that tab's `AbortController` is aborted. Other tabs' analyzers continue.
- **Per-analyzer timeout:** The analyzer's derived `AbortController` (which is tied to BOTH the tab signal AND its own timeout) fires.

Each analyzer receives its unique `AbortSignal`. The signal is aborted if ANY of its parent signals are aborted.

**Analyzer contract for cancellation:**
- Every analyzer MUST accept an `AbortSignal` in its execution context
- Every analyzer MUST periodically check `signal.aborted` during long-running loops
- Every analyzer MUST pass its `AbortSignal` to any `fetch()` calls (via `fetch(url, { signal })`)
- When `signal.aborted` is detected, the analyzer MUST:
  1. Stop computation immediately
  2. Release any acquired resources (canvas contexts, file handles, audio contexts)
  3. Return `signal_value: CANCELLED` with any partial results available
  4. Must NOT throw — cancellation is a normal control flow path

#### 1.8.3 Resource Cleanup on Cancellation

| Resource | Cleanup Action | Timing |
|---|---|---|
| **Offscreen document tasks** | The specific task is cancelled via `AbortSignal`. The offscreen document itself is NOT closed — it may have queued tasks for other tabs. | Immediately |
| **Model weights (ONNX/WASM)** | Not unloaded on cancel. Models are expensive to reload and may be needed for subsequent analysis. Only unload on memory pressure or explicit session end. | Retained |
| **tabCapture stream** | `MediaStreamTrack.stop()` on all tracks. Revoke the capture. | Immediately |
| **Cloud fetch requests** | `AbortController.abort()` cancels in-flight fetches. Already-completed fetches are unaffected. | Immediately |
| **Content script DOM references** | Nulled out. Content script execution context may already be destroyed. | Immediately |
| **Intermediate pixel buffers / decoded frames** | Discarded. Memory freed. | Immediately |
| **Evidence results from completed analyzers** | Retained. Even if the session was cancelled, completed results are kept for display. User cancelled mid-analysis; they should see what was already done. | Retained for display |
| **Checkpoint state** | Discarded on explicit cancel. Retained on tab close (in case user reopens tab and wants to resume). | Depends on cancel source |

#### 1.8.4 Graceful Cancellation UX

| Cancel Source | User Sees |
|---|---|
| User clicked "Cancel" | "Analysis cancelled. Showing partial results from 3 of 7 checks." |
| Tab closed | No UI shown. Results silently discarded. |
| Timeout on one analyzer | "Image forensics check timed out. Other checks completed." (other analyzers unaffected) |
| Budget exceeded | "Analysis paused to maintain browser performance. Showing results from 5 of 9 checks." |
| Navigation | "Analysis stopped — page changed. Click 'Analyze' on the new page to continue." |

### 1.9 Dependency Resolution

#### 1.9.1 DAG Construction & Topological Sort

**At registration time:**
1. When a plugin is registered, its `required_evidence` list (dependencies) is validated
2. All referenced dependencies must exist in the registry (by `analyzer_id`)
3. A temporary DAG is constructed with all registered plugins
4. Cycle detection runs via depth-first search with a visited/recursion-stack algorithm
5. If a cycle is detected, registration is **rejected** with the cycle path printed:

```
Circular dependency detected:
  video_face_swap → video_frame_extract → video_metadata → video_face_swap
```

**At scheduling time:**
1. The Scheduler takes the set of eligible analyzers (after capability, privacy, and content-type filtering)
2. A sub-DAG is extracted containing only eligible analyzers
3. Topological sort orders analyzers by dependencies
4. Within each topological level (analyzers at the same depth with all dependencies satisfied), analyzers are sorted by their `execution_priority` (latency class + signal tier bias)
5. Execution groups are formed from same-depth analyzers (see Section 1.2.2)

**Example DAG for an on-demand page analysis:**

```
Level 0 (no dependencies, IMMEDIATE):
  - DOM text extraction
  - Image element enumeration
  - Cached domain reputation lookup
  - Content type detection

Level 1 (depends on Level 0, FAST):
  - EXIF metadata parsing (depends on: image enumeration)
  - C2PA verification (depends on: image enumeration, SOFT_REQUIRES metadata)
  - Text style analysis (depends on: text extraction)

Level 2 (depends on Level 1, STANDARD):
  - AI generator fingerprint (depends on: image enumeration, ENRICHES from C2PA)
  - ELA analysis (depends on: image enumeration)
  - Noise pattern analysis (depends on: image enumeration)
  - Watermark detection (depends on: image enumeration)

Level 3 (depends on Level 2, STANDARD/SLOW):
  - Cross-modal consistency (ENRICHES from: text analysis, image forensics)
  - Multi-image consistency (depends on: all image analyzers)

Level 4 (independent, SLOW, cloud):
  - Reverse image search (depends on: image enumeration)
  - Fact check lookup (depends on: text extraction)
  - Cloud domain reputation (depends on: URL extraction)
```

#### 1.9.2 Dependency Failure Handling

| Dependency Type | Dependency Result | Dependent Behavior |
|---|---|---|
| `HARD_REQUIRES` | SUCCESS | Dependent runs normally |
| `HARD_REQUIRES` | FAILED / TIMEOUT / CANCELLED | Dependent is **SKIPPED** with reason: `DEPENDENCY_FAILED: {analyzer_name}` |
| `SOFT_REQUIRES` | SUCCESS | Dependent runs normally with full input |
| `SOFT_REQUIRES` | FAILED / TIMEOUT / CANCELLED | Dependent runs with fallback/partial input. Confidence ceiling reduced by 15%. |
| `ENRICHES` | SUCCESS | Dependent runs with enriched input |
| `ENRICHES` | FAILED / TIMEOUT / CANCELLED | Dependent runs without enriched input. No confidence penalty. |
| `PREFERS_ORDER` | SUCCESS or FAILED | Dependent runs regardless. No data dependency. |

**Multiple dependency failure:** If an analyzer has 3 HARD_REQUIRES dependencies and 1 fails, the analyzer is skipped regardless of the other 2 succeeding. All HARD_REQUIRES must succeed.

**Cascading skip reporting:** When an analyzer is skipped due to dependency failure, all analyzers that HARD_REQUIRE it are also skipped. The Scheduler traverses the DAG forward to mark all transitively dependent analyzers as `SKIPPED: DEPENDENCY_FAILED`. The UI shows: "C2PA verification failed → Image provenance check skipped → Multi-image consistency skipped." This is reported as a linear chain, not a confusing tree.

#### 1.9.3 Circular Dependency Rejection at Registration

The Scheduler validates the entire plugin registry DAG at:
1. Extension startup (all built-in plugins registered)
2. Every dynamic plugin registration (user-installed plugin)
3. Every plugin update

**Rejection behavior:**
- If a built-in plugin has a circular dependency (should never happen; caught in CI), the extension fails to initialize with a console error
- If a user-installed plugin introduces a circular dependency, the plugin is rejected with an error message: "Cannot install {plugin_name}: creates a circular dependency with {other_plugin_name}."
- The user sees a notification: "Plugin installation failed — incompatible with your current plugin set."

### 1.10 Progress Reporting

#### 1.10.1 Progress Event Model

The Scheduler emits progress events to a **Progress Bus**. The UI (side panel, popup, or content script overlay) subscribes to the bus for the active analysis session.

**Event types:**

| Event | Payload | When Emitted |
|---|---|---|
| `SESSION_STARTED` | `{ session_id, invocation_model, tab_id, content_types }` | Analysis session begins |
| `SESSION_COMPLETED` | `{ session_id, total_analyzers, completed, failed, skipped }` | All analyzers finished (or early termination triggered) |
| `SESSION_CANCELLED` | `{ session_id, reason, partial_results_available }` | User cancelled or tab closed |
| `ANALYZER_QUEUED` | `{ session_id, analyzer_id, priority, queue_position }` | Analyzer added to execution queue |
| `ANALYZER_STARTED` | `{ session_id, analyzer_id, started_at }` | Analyzer begins execution |
| `ANALYZER_PROGRESS` | `{ session_id, analyzer_id, percent, message }` | Optional. For long-running analyzers that can report incremental progress. |
| `ANALYZER_COMPLETED` | `{ session_id, analyzer_id, result_summary, latency_ms }` | Analyzer finishes successfully |
| `ANALYZER_FAILED` | `{ session_id, analyzer_id, error, latency_ms }` | Analyzer fails |
| `ANALYZER_TIMED_OUT` | `{ session_id, analyzer_id, timeout_ms }` | Analyzer exceeds timeout |
| `ANALYZER_CANCELLED` | `{ session_id, analyzer_id }` | Analyzer cancelled (user or system) |
| `ANALYZER_SKIPPED` | `{ session_id, analyzer_id, reason }` | Analyzer excluded (offline, unsupported, dependency failure, etc.) |
| `MILESTONE_REACHED` | `{ session_id, milestone: "initial" \| "refined" \| "deep" \| "final" }` | Progressive display milestone reached |
| `BUDGET_WARNING` | `{ session_id, resource, consumed_percent }` | CPU or memory approaching budget limit |

#### 1.10.2 Progress Summary Model

The UI renders a progress summary from the current session state:

```
3 of 7 analyzers complete  •  2 running  •  1 queued  •  1 skipped
```

**Status per analyzer:**

| Status | Description | End State? |
|---|---|---|
| `PENDING` | In the execution queue, not yet started | No |
| `RUNNING` | Currently executing | No |
| `COMPLETED` | Finished successfully with a result | Yes |
| `FAILED` | Finished with an error (INTERNAL_ERROR) | Yes |
| `TIMED_OUT` | Exceeded its declared timeout | Yes |
| `CANCELLED` | Cancelled by user or system event | Yes |
| `SKIPPED` | Not run (offline, unsupported browser, dependency failed, privacy opt-out, content not applicable) | Yes |

#### 1.10.3 UI Contract

The Progress Bus delivers events to UI contexts via `chrome.runtime.sendMessage()` or long-lived port. The UI:

1. Subscribes to the session ID when the user opens the analysis panel
2. Receives a snapshot of all current analyzer statuses (including completed ones from before the panel was opened)
3. Receives incremental updates as events fire
4. Unsubscribes when the panel is closed (doesn't mean analysis stops)

**Progressive display rendering priority:**
- IMMEDIATE and FAST results render first (within the "Initial" and "Refined" milestones)
- STANDARD and SLOW results render as they complete (within the "Deep" milestone)
- The UI must never reflow aggressively — result cards should have fixed heights and animate in smoothly
- When the "Initial" milestone fires, the UI transitions from "Analyzing..." to showing the preliminary card set

---

## Part 2: Analyzer Plugin Architecture

### 2.1 Plugin Manifest Specification

Every analyzer plugin declares itself via a JSON manifest. This manifest is the single source of truth that drives the Scheduler's routing, capability checking, dependency resolution, and resource budgeting.

#### 2.1.1 Full Manifest Schema

```json
{
  "plugin": {
    "analyzer_id": "string — unique identifier, kebab-case (e.g., 'c2pa-verifier')",
    "display_name": "string — human-readable name (e.g., 'C2PA Manifest Verifier')",
    "version": "string — semantic versioning (e.g., '1.3.2')",
    "author": "string — provider name",
    "description": "string — 1-2 sentence description of what this analyzer does",
    "plugin_type": "built-in | dynamic | enterprise",
    "created_at": "ISO 8601",
    "updated_at": "ISO 8601",
    "homepage_url": "URL — optional documentation/source link"
  },

  "capabilities": {
    "content_types": ["image/jpeg", "image/png", "image/webp", "image/avif", "text/plain", ...],
    "max_content_size_bytes": "integer — largest file this analyzer can process (null = no limit)",
    "min_content_size_bytes": "integer — minimum file size (e.g., 1000 for meaningful analysis)",
    "min_text_length_words": "integer — minimum text length for meaningful analysis (null for non-text)",
    "produces_evidence_types": ["PROVENANCE", "FORENSIC", "REPUTATION", "COMMUNITY", "DERIVED"],
    "produces_signal_tier": "1 | 2 | 3 | 4 — matches Product Reasoning Spec evidence hierarchy",
    "can_handle_batch": "boolean — can this analyzer process multiple items efficiently in a batch?",
    "max_batch_size": "integer — maximum items per batch invocation"
  },

  "permissions": {
    "host_permissions": ["<all_urls>", "https://*.example.com/*", null],
    "extension_permissions": ["activeTab", "storage", "alarms", "webNavigation", ...],
    "api_access": ["tabCapture", "offscreen", "declarativeNetRequest", ...]
  },

  "browser_support": {
    "supported_browsers": ["chrome", "edge", "firefox", "safari"],
    "minimum_versions": {
      "chrome": "114",
      "edge": "114",
      "firefox": "115",
      "safari": "16.4"
    },
    "required_apis": ["offscreen", "webgpu"],
    "fallback_apis": ["wasm"],
    "browser_notes": {
      "firefox": "Uses sandboxed page instead of offscreen document. Performance reduced by ~30%.",
      "safari": "WASM execution only. No WebGPU acceleration. Expect 3-5x slower inference."
    }
  },

  "execution": {
    "context": "content_script | service_worker | offscreen_document | sandbox_page | tab_capture",
    "fallback_context": "sandbox_page",
    "latency_class": "IMMEDIATE | FAST | STANDARD | SLOW | BACKGROUND",
    "estimated_runtime_ms": {
      "best_case": 50,
      "typical": 200,
      "worst_case": 1500
    },
    "is_parallelizable": "boolean — can multiple invocations of this analyzer run concurrently?",
    "max_concurrent_instances": "integer — max parallel instances (1 = singleton)",
    "requires_user_gesture": "boolean — does this analyzer need a user gesture to start? (e.g., tabCapture)"
  },

  "resource_profile": {
    "model_size_mb": "number — size of ONNX/WASM model (0 if no model)",
    "runtime_memory_mb": "number — peak runtime memory excluding model (pixel buffers, intermediate tensors)",
    "estimated_memory_mb": "number — model_size_mb + runtime_memory_mb",
    "storage_requirement_mb": "number — persistent storage needed for reference data (0 if none)",
    "cpu_intensity": "low | medium | high | extreme",
    "gpu_accelerated": "boolean — benefits from WebGPU?",
    "network_required": "boolean — must have network access?",
    "network_bandwidth_estimate_kbps": "number — estimated bandwidth for cloud analyzers (0 if local)"
  },

  "evidence": {
    "confidence_range": {
      "min": 0.0,
      "max": 1.0,
      "typical_accuracy": 0.85,
      "calibration_confidence": 0.80
    },
    "requires_inputs": ["array of evidence_type strings that MUST be available before running"],
    "optional_inputs": ["array of evidence_type strings that ENRICH but don't block"],
    "produces_outputs": {
      "primary_evidence_type": "PROVENANCE | FORENSIC | REPUTATION | COMMUNITY | DERIVED",
      "secondary_evidence_types": ["array — additional evidence types produced"],
      "output_schema": "JSON Schema — structure of the evidence_detail field in the result"
    },
    "independence_class": "string — 'cryptographic' | 'ml_statistical' | 'heuristic' | 'reference_lookup' | 'network_query'",
    "correlated_with": ["array of analyzer_ids that produce correlated signals — used for independence discounting"]
  },

  "dependencies": {
    "hard_requires": [
      {
        "analyzer_id": "string",
        "evidence_type": "string",
        "reason": "string — why this dependency exists"
      }
    ],
    "soft_requires": [
      {
        "analyzer_id": "string",
        "evidence_type": "string",
        "fallback_behavior": "string — description of degraded behavior when dependency fails"
      }
    ],
    "enriches_from": [
      {
        "analyzer_id": "string",
        "evidence_type": "string",
        "how_enriched": "string — how the output improves with this input"
      }
    ],
    "prefers_after": ["array of analyzer_ids — UX ordering preference, no data dependency"]
  },

  "privacy": {
    "tier": "LOCAL_ONLY | ANONYMIZED_QUERY | CONTENT_TRANSMITTED",
    "data_transmitted": "string — exactly what leaves the device, if anything",
    "transmission_endpoint": "URL — where data is sent, if applicable",
    "retention_policy": "string — what the endpoint does with the data",
    "third_party_notice": "string — 'Data is sent to {service_name}. See {privacy_policy_url}'",
    "user_consent_required": "boolean — must prompt user before first invocation?",
    "consent_message": "string — the prompt text shown to user",
    "can_be_disabled": "boolean — can user disable this analyzer? (always true for dynamic, may be false for essential built-in)"
  },

  "failure_modes": [
    {
      "error_code": "CORS_BLOCKED",
      "classification": "UNSUPPORTED_CONTENT",
      "message": "string — user-facing explanation",
      "retryable": false,
      "user_action": "string — what the user can do about it"
    },
    {
      "error_code": "MODEL_LOAD_FAILED",
      "classification": "INTERNAL_ERROR",
      "message": "string",
      "retryable": false,
      "user_action": "string"
    }
  ],

  "lifecycle": {
    "status": "ACTIVE | DEPRECATED | RETIRED | EXPERIMENTAL",
    "deprecation_message": "string — shown to user when status is DEPRECATED",
    "successor_analyzer_id": "string — recommended replacement when DEPRECATED",
    "shadow_evaluation_required": "boolean — must this plugin pass shadow evaluation before ACTIVE?",
    "shadow_evaluation_min_invocations": "integer — minimum invocations before promotion consideration",
    "min_trust_score_for_active": "number — minimum source trust score to transition from EXPERIMENTAL to ACTIVE"
  }
}
```

#### 2.1.2 Manifest Validation Rules

At registration time, the Plugin Registry validates:

| Rule | Validation |
|---|---|
| `analyzer_id` uniqueness | Must not collide with any registered plugin (active or retired) |
| `version` format | Must be valid semver |
| `content_types` | Must include at least one valid MIME type |
| `required_apis` vs `supported_browsers` | If `tabCapture` is required, `supported_browsers` must not include Firefox or Safari |
| `hard_requires` references | All referenced `analyzer_id` values must exist in the registry |
| `execution.context` vs `browser_support` | If context is `offscreen_document`, Firefox/Safari must have a `fallback_context` |
| `dependencies` acyclicity | Must not create a cycle (see Section 1.9.3) |
| `privacy.tier` consistency | If `CONTENT_TRANSMITTED`, must have `user_consent_required: true` |
| `resource_profile.estimated_memory_mb` | If >200 MB, must log a warning (will frequently be starved) |
| `latency_class` vs `estimated_runtime_ms` | `worst_case` must be <= ceiling for declared latency class |
| `shadow_evaluation_required` | Must be `true` for all `dynamic` plugins. May be `false` for `built-in`. |

### 2.2 Plugin Lifecycle

#### 2.2.1 Lifecycle States

```
REGISTRATION → VALIDATION → LOADING → ACTIVE → DEPRECATED → RETIRED
                                  ↓         ↓
                            [LOAD_FAILED]  [DEGRADED]
                                  ↓
                            EXPERIMENTAL → (shadow evaluation) → ACTIVE
```

| State | Description | Scheduler Behavior |
|---|---|---|
| **REGISTRATION** | Plugin manifest received but not yet validated | No scheduling. |
| **VALIDATION** | Manifest being checked against rules (2.1.2). DAG cycle detection. | No scheduling. |
| **LOADING** | Plugin code being loaded. Models downloading (if applicable). WASM/ONNX runtime initializing. | No scheduling. Transitions to ACTIVE or LOAD_FAILED. |
| **ACTIVE** | Plugin is fully loaded, validated, and available for scheduling. | Normal scheduling. Results contribute to confidence. |
| **DEPRECATED** | Plugin has been superseded by a successor. Still functional but discouraged. | Still scheduled but results carry a deprecation confidence penalty (multiplier < 1.0). User prompted to upgrade. |
| **RETIRED** | Plugin no longer available. Not in active registry. | Never scheduled. Historical results still referenced for audit. |
| **EXPERIMENTAL** | New plugin (dynamic or built-in) undergoing shadow evaluation. | Scheduled normally but results **do not contribute to confidence assessment**. Results are logged for accuracy comparison against ACTIVE plugins. |
| **DEGRADED** | Plugin is ACTIVE but exhibiting elevated error/timeout rates. | Scheduled but with reduced trust weight. Results may be excluded if better alternatives exist. |
| **LOAD_FAILED** | Plugin code/model failed to load. | Not scheduled. Retried on next extension startup or when user manually retries. |

#### 2.2.2 State Transition Triggers

| Transition | Trigger |
|---|---|
| REGISTRATION → VALIDATION | Automatic on plugin registration (startup for built-in, user action for dynamic) |
| VALIDATION → LOADING | Validation passed |
| VALIDATION → retired (rejected) | Validation failed (circular deps, missing deps, incompatible browser) |
| LOADING → ACTIVE | Code loaded, model loaded, runtime initialized |
| LOADING → LOAD_FAILED | Model download failed, WASM compilation failed, runtime error |
| ACTIVE → DEPRECATED | New version installed, plugin provider marks as deprecated |
| DEPRECATED → RETIRED | Deprecation period expires (default: 30 days) or user manually removes |
| ACTIVE → DEGRADED | Error rate > 50% OR timeout rate > 20% in rolling window of 50 invocations |
| DEGRADED → ACTIVE | Error rate drops below threshold for 50 consecutive invocations |
| EXPERIMENTAL → ACTIVE | Shadow evaluation complete: `min_invocations` met AND trust score >= `min_trust_score_for_active` |
| EXPERIMENTAL → retired | Shadow evaluation failed: trust score below threshold after `min_invocations` + buffer |
| LOAD_FAILED → LOADING | User manually retries or extension restarts |
| Any state → retired | User uninstalls the plugin |

#### 2.2.3 Shadow Evaluation Period

For all `dynamic` and `experimental` built-in plugins:

1. Plugin runs normally — Scheduler invokes it, it produces results
2. Results are collected in a shadow evaluation log
3. After `shadow_evaluation_min_invocations` (default: 100) completed analyses:
   - Compare shadow results against ACTIVE plugins producing the same evidence type
   - Measure: accuracy agreement (do results correlate with trusted plugins?), calibration (are confidence scores well-calibrated?), false positive rate, false negative rate
   - If metrics meet thresholds: promote to ACTIVE
   - If metrics fail: extend shadow period or retire
4. During shadow evaluation, the plugin appears in the UI as: "New analyzer (evaluation mode) — results not yet used in confidence scores"

**Why shadow evaluation matters:**
- Prevents a buggy user-installed plugin from poisoning confidence assessments
- Allows A/B testing of new built-in models before deploying
- Builds a calibration history before the plugin's results carry weight

#### 2.2.4 Hot-Swapping and Mid-Session Replacement

**Can a plugin be replaced while analysis is running?**

| Scenario | Handling |
|---|---|
| **Plugin update while idle** (no active analysis) | Old version transitions DEPRECATED → RETIRED. New version validates, loads, enters EXPERIMENTAL (if dynamic) or ACTIVE (if built-in). Seamless. |
| **Plugin update while analysis is running** | Running instance of old plugin continues to completion. New version registered under new `analyzer_id@version`. Any queued but not-yet-started invocations of the old version are cancelled and re-queued to the new version. |
| **Plugin uninstalled while analysis is running** | Running instances complete (they're already loaded in memory). Queued invocations are cancelled → marked `SKIPPED: PLUGIN_REMOVED`. No new invocations scheduled. |
| **Model update (same plugin, new model version)** | The plugin declares a new `version`. The old version's model is retained until all in-flight invocations complete. The new model is loaded in parallel (if memory permits) or queued after old model is freed. Results from the old version are tagged with the old version number for audit. |

#### 2.2.5 Model Download Lifecycle

Large ONNX/WASM models (5-150 MB) are NOT bundled with the extension. They are downloaded post-install.

**Download flow:**
1. Plugin transitions from VALIDATION to LOADING
2. The Plugin Registry checks if the model is already cached in `chrome.storage.local` (IndexedDB for >10 MB)
3. If cached and version matches → load from cache → ACTIVE
4. If not cached or version mismatch → download from declared model URL
5. Download happens in the service worker or offscreen document (which persists during download)
6. SHA-256 hash of downloaded model is verified against the manifest's declared `model_integrity_hash`
7. If verification fails → discard download, retry once, then → LOAD_FAILED
8. If verification passes → store in IndexedDB (for offscreen document access) → LOADING complete → ACTIVE

**Download UX:**
- The UI shows: "Downloading analysis models... (12 MB of 45 MB)"
- User can cancel download
- If user goes offline during download, download pauses and resumes on reconnection
- Models download in the background; the extension is functional with basic analyzers (metadata, text) while models download

### 2.3 Plugin Sandboxing

#### 2.3.1 Trust Tiers

| Tier | Plugin Type | Execution Context | Access |
|---|---|---|---|
| **Tier 0: Built-in (Trusted)** | Core analyzers bundled with the extension | Offscreen document with full extension API access via `chrome.runtime` messaging | Access to: local models, reference data bundles, `chrome.storage`, results from other analyzers. Can read extension configuration. |
| **Tier 1: Built-in (Experimental)** | New core analyzers in shadow evaluation | Same as Tier 0 | Same access as Tier 0, but results are flagged EXPERIMENTAL and don't feed confidence. |
| **Tier 2: Verified Dynamic** | Third-party plugins that passed Sentinel's verification process (Phase 3+) | Sandboxed page (strict CSP: `sandbox allow-scripts`). No extension API access. | Communicates with Trust Engine via structured message channel. Requests content access through a capability broker. Cannot access storage, network (directly), or other analyzers' results. |
| **Tier 3: User-Installed (Untrusted)** | Plugins installed by the user from any source | Sandboxed page (strictest CSP: `sandbox allow-scripts`). No same-origin access. | Communicates via message channel. Every capability request is user-gated (user must approve "Plugin X wants to access image pixels"). Network access only through a proxy that strips cookies and enforces allowlisted endpoints. |

#### 2.3.2 Capability Broker

Dynamic/user-installed plugins do not have direct access to content, storage, or network. They request capabilities through a **Capability Broker** in the service worker:

| Capability Request | Broker Behavior |
|---|---|
| `ACCESS_IMAGE_PIXELS` | Broker fetches the image (with CORS), decodes it, and passes `ImageData` to the plugin. Plugin never sees the URL. Broker strips EXIF metadata before passing pixels (privacy: EXIF may contain GPS coordinates). |
| `ACCESS_TEXT_CONTENT` | Broker passes extracted text from the content script. Plugin never accesses the DOM directly. |
| `ACCESS_AUDIO_SAMPLES` | Broker captures audio, extracts raw PCM samples, passes to plugin. Plugin never accesses tabCapture directly. |
| `FETCH_URL` | Plugin declares: "I need to fetch from `https://api.example.com/check`". Broker shows the user: "Plugin X wants to connect to api.example.com. Allow? [Yes] [No] [Always for this plugin]". If approved, broker proxies the fetch, stripping cookies and adding rate limiting. |
| `QUERY_REFERENCE_DATA` | Plugin asks: "What is the domain reputation for example.com?" Broker queries the local reference data and returns the result. Plugin never accesses the reference database directly. |
| `LOAD_MODEL` | Plugin declares a model URL and integrity hash. Broker downloads the model, verifies the hash, and loads it into the plugin's sandboxed context. Plugin cannot load arbitrary models — only the declared model. |

**What plugins can NEVER access (regardless of tier):**
- User's browsing history
- User's bookmarks
- Cookies or local storage from any origin
- Passwords or form input fields (enforced at content script level — these are excluded from DOM extraction)
- Other extensions' data
- Raw page HTML (only extracted text content)
- Network requests to arbitrary URLs (only declared endpoints)

#### 2.3.3 Code Integrity Verification

| Plugin Type | Verification |
|---|---|
| Built-in | Code is bundled with the extension. Verified by the extension store's code signing at install time. No runtime verification needed. |
| Verified Dynamic | Code is signed by Sentinel's private key. Broker verifies the signature before loading into the sandboxed page. Signature verification uses the extension's bundled public key. |
| User-Installed | Code is hashed (SHA-256) at install time. The hash is stored. On each load, the broker re-hashes the code and compares. If the hash changed (tampering), the plugin is blocked from loading. User is notified: "Plugin X has been modified and cannot be loaded." |

### 2.4 Per-Plugin Specifications

For each analyzer type in the Trust Engine, the following tables define the key manifest fields. Fields not listed use sensible defaults derived from the analyzer type.

---

#### 2.4.1 C2PA Analyzer

| Field | Value |
|---|---|
| `analyzer_id` | `c2pa-verifier` |
| `plugin_type` | `built-in` |
| `content_types` | `["image/jpeg", "image/png", "image/webp", "image/avif", "image/tiff", "video/mp4", "audio/wav", "application/pdf"]` |
| `max_content_size_bytes` | `52428800` (50 MB — larger files impractical for web) |
| `produces_evidence_types` | `["PROVENANCE"]` |
| `produces_signal_tier` | `1` (Ground Truth) |
| `required_apis` | `["wasm"]` (c2pa-rs compiled to WASM) |
| `execution.context` | `offscreen_document` |
| `fallback_context` | `sandbox_page` |
| `latency_class` | `FAST` |
| `estimated_runtime_ms` | `{ best_case: 50, typical: 200, worst_case: 1500 }` (first verification includes WASM instantiation; cached WASM makes subsequent verifications fast) |
| `model_size_mb` | `3` (c2pa-rs WASM) |
| `runtime_memory_mb` | `5` (image bytes + manifest parsing) |
| `estimated_memory_mb` | `8` |
| `independence_class` | `cryptographic` |
| `hard_requires` | `[]` (no dependency — can verify raw bytes directly) |
| `soft_requires` | `[{ analyzer_id: "exif-extractor", evidence_type: "METADATA", fallback_behavior: "C2PA can locate the JUMBF box by scanning the file binary. Metadata extraction provides a faster path but is not required." }]` |
| `privacy.tier` | `LOCAL_ONLY` |
| `failure_modes` | `[{ error_code: "CORS_BLOCKED", classification: "UNSUPPORTED_CONTENT", message: "Cannot fetch image. The server does not allow cross-origin access.", retryable: false }, { error_code: "NO_MANIFEST_FOUND", classification: "INCONCLUSIVE", message: "No C2PA manifest found in this content.", retryable: false }, { error_code: "SIGNATURE_INVALID", classification: "COMPLETED", message: "C2PA manifest found but signature verification failed. The manifest may have been tampered with.", retryable: false }, { error_code: "CERTIFICATE_UNTRUSTED", classification: "COMPLETED", message: "C2PA manifest has a valid signature but the signing certificate is from an untrusted authority.", retryable: false }]` |
| `lifecycle.shadow_evaluation_required` | `false` (built-in, tested before release) |

---

#### 2.4.2 EXIF Analyzer

| Field | Value |
|---|---|
| `analyzer_id` | `exif-extractor` |
| `plugin_type` | `built-in` |
| `content_types` | `["image/jpeg", "image/tiff", "image/webp", "image/png"]` |
| `produces_evidence_types` | `["PROVENANCE"]` |
| `produces_signal_tier` | `3` (Moderate — trivially fabricated) |
| `required_apis` | `[]` (pure JS binary parsing) |
| `execution.context` | `content_script` (can parse in content script via `fetch` of same-origin images) |
| `fallback_context` | `offscreen_document` (for cross-origin images fetched with CORS) |
| `latency_class` | `IMMEDIATE` |
| `estimated_runtime_ms` | `{ best_case: 5, typical: 20, worst_case: 100 }` |
| `model_size_mb` | `0` (no model — pure parsing) |
| `runtime_memory_mb` | `2` (image header bytes) |
| `estimated_memory_mb` | `2` |
| `independence_class` | `heuristic` |
| `hard_requires` | `[{ analyzer_id: "image-enumerator", evidence_type: "IMAGE_LIST", reason: "Needs image URLs to fetch and parse" }]` |
| `privacy.tier` | `LOCAL_ONLY` |
| `failure_modes` | `[{ error_code: "CORS_BLOCKED", classification: "UNSUPPORTED_CONTENT", message: "Cannot read metadata from cross-origin image without CORS headers.", retryable: false }, { error_code: "NO_EXIF", classification: "INCONCLUSIVE", message: "No EXIF metadata found. This is common — many platforms strip EXIF data.", retryable: false }]` |

---

#### 2.4.3 Image Metadata Analyzer (IPTC/XMP/ICC)

| Field | Value |
|---|---|
| `analyzer_id` | `image-metadata-extractor` |
| `plugin_type` | `built-in` |
| `content_types` | `["image/jpeg", "image/tiff", "image/png", "image/webp"]` |
| `produces_evidence_types` | `["PROVENANCE"]` |
| `produces_signal_tier` | `3` |
| `required_apis` | `[]` |
| `execution.context` | `content_script` |
| `latency_class` | `IMMEDIATE` |
| `estimated_runtime_ms` | `{ best_case: 10, typical: 30, worst_case: 150 }` |
| `model_size_mb` | `0` |
| `runtime_memory_mb` | `2` |
| `independence_class` | `heuristic` |
| `hard_requires` | `[{ analyzer_id: "image-enumerator", evidence_type: "IMAGE_LIST" }]` |
| `correlated_with` | `["exif-extractor"]` (both parse file headers — not truly independent signals) |
| `privacy.tier` | `LOCAL_ONLY` |
| `failure_modes` | `[{ error_code: "NO_IPTC", classification: "INCONCLUSIVE", message: "No IPTC/XMP metadata found.", retryable: false }]` |

---

#### 2.4.4 Text Analyzer (Perplexity/Burstiness/Style)

| Field | Value |
|---|---|
| `analyzer_id` | `text-pattern-analyzer` |
| `plugin_type` | `built-in` |
| `content_types` | `["text/plain", "text/html"]` |
| `min_text_length_words` | `100` (below 100 words: suppressed per Product Reasoning Spec) |
| `produces_evidence_types` | `["FORENSIC"]` |
| `produces_signal_tier` | `4` (Weak — converges to random as LLMs improve) |
| `required_apis` | `["onnx"]` (for perplexity model) |
| `execution.context` | `offscreen_document` (for ONNX inference) |
| `fallback_context` | `sandbox_page` |
| `latency_class` | `FAST` |
| `estimated_runtime_ms` | `{ best_case: 50, typical: 300, worst_case: 800 }` |
| `model_size_mb` | `15` (small perplexity model, e.g., GPT-2 distilled) |
| `runtime_memory_mb` | `10` (token window) |
| `estimated_memory_mb` | `25` |
| `independence_class` | `ml_statistical` |
| `hard_requires` | `[{ analyzer_id: "text-extractor", evidence_type: "TEXT_BLOCKS" }]` |
| `enriches_from` | `[{ analyzer_id: "domain-reputation", evidence_type: "REPUTATION", how_enriched: "Known AI-content domains increase prior probability for text generation." }]` |
| `privacy.tier` | `LOCAL_ONLY` |
| **Critical UX constraint (from Product Reasoning Spec):** Must display: "Text analysis is the least reliable signal. It should never be the sole basis for any conclusion. Text under 100 words is not analyzed." |
| **Never Use Rule:** Model-specific text classifiers (e.g., "GPT-4.5 detector") must NEVER be registered. The text-pattern-analyzer uses zero-shot statistical methods only. |
| `failure_modes` | `[{ error_code: "TEXT_TOO_SHORT", classification: "INCONCLUSIVE", message: "Text too short for reliable analysis (minimum 100 words).", retryable: false }, { error_code: "NON_NATIVE_ENGLISH", classification: "COMPLETED", message: "Note: Text analysis is less reliable for non-native English writing. Results should be interpreted with additional caution." }]` |

---

#### 2.4.5 Domain Reputation Analyzer

| Field | Value |
|---|---|
| `analyzer_id` | `domain-reputation` |
| `plugin_type` | `built-in` |
| `content_types` | `["text/html", "*/*"]` (applies to any page content) |
| `produces_evidence_types` | `["REPUTATION"]` |
| `produces_signal_tier` | `4` (Weak — lagging indicator) |
| `required_apis` | `[]` |
| `execution.context` | `service_worker` |
| `latency_class` | `IMMEDIATE` |
| `estimated_runtime_ms` | `{ best_case: 1, typical: 5, worst_case: 20 }` (local cache lookup) |
| `model_size_mb` | `0` |
| `runtime_memory_mb` | `5` (domain list in memory) |
| `estimated_memory_mb` | `5` |
| `storage_requirement_mb` | `10` (compressed domain reputation bundle in `chrome.storage.local`) |
| `network_required` | `false` (cached mode) — `true` (real-time lookup mode) |
| `independence_class` | `reference_lookup` |
| `hard_requires` | `[{ analyzer_id: "url-extractor", evidence_type: "PAGE_URL" }]` |
| `privacy.tier` | `LOCAL_ONLY` (cached) / `ANONYMIZED_QUERY` (real-time — domain name transmitted, not full URL) |
| `user_consent_required` | `true` (for real-time lookup mode) |
| `consent_message` | "Real-time domain reputation requires sending the website domain to Sentinel's servers. No page content or URL path is sent. [Enable] [Use cached data only]" |
| `failure_modes` | `[{ error_code: "CACHE_STALE", classification: "INCONCLUSIVE", message: "Domain reputation data is {X} days old and may not reflect recent changes.", retryable: false }, { error_code: "NETWORK_UNAVAILABLE", classification: "INCONCLUSIVE", message: "Cannot refresh domain reputation — offline. Using cached data from {X} days ago.", retryable: false }]` |

---

#### 2.4.6 Noise Analyzer

| Field | Value |
|---|---|
| `analyzer_id` | `noise-analyzer` |
| `plugin_type` | `built-in` |
| `content_types` | `["image/jpeg", "image/png", "image/webp"]` |
| `produces_evidence_types` | `["FORENSIC"]` |
| `produces_signal_tier` | `2` (Strong Corroborating — frequency-domain artifacts) |
| `required_apis` | `["wasm", "offscreen_canvas"]` |
| `execution.context` | `offscreen_document` |
| `fallback_context` | `sandbox_page` |
| `latency_class` | `STANDARD` |
| `estimated_runtime_ms` | `{ best_case: 100, typical: 500, worst_case: 2000 }` |
| `model_size_mb` | `0` (pure statistical computation — wavelet decomposition, residual analysis) |
| `runtime_memory_mb` | `20` (pixel buffers + intermediate matrices) |
| `estimated_memory_mb` | `20` |
| `is_parallelizable` | `false` (single-threaded WASM) |
| `independence_class` | `ml_statistical` |
| `hard_requires` | `[{ analyzer_id: "image-enumerator", evidence_type: "IMAGE_LIST" }]` |
| `soft_requires` | `[{ analyzer_id: "image-metadata-extractor", evidence_type: "METADATA", fallback_behavior: "Metadata about camera sensor can inform noise expectation model." }]` |
| `privacy.tier` | `LOCAL_ONLY` |
| `failure_modes` | `[{ error_code: "IMAGE_TOO_SMALL", classification: "UNSUPPORTED_CONTENT", message: "Image too small for noise analysis (minimum 200x200 pixels).", retryable: false }, { error_code: "HEAVILY_COMPRESSED", classification: "INCONCLUSIVE", message: "Image compression artifacts obscure noise patterns. Analysis may be unreliable.", retryable: false }]` |

---

#### 2.4.7 ELA Analyzer

| Field | Value |
|---|---|
| `analyzer_id` | `ela-analyzer` |
| `plugin_type` | `built-in` |
| `content_types` | `["image/jpeg"]` (ELA is JPEG-specific) |
| `produces_evidence_types` | `["FORENSIC"]` |
| `produces_signal_tier` | `2` |
| `required_apis` | `["offscreen_canvas"]` |
| `execution.context` | `offscreen_document` |
| `latency_class` | `STANDARD` |
| `estimated_runtime_ms` | `{ best_case: 80, typical: 300, worst_case: 1200 }` |
| `model_size_mb` | `0` |
| `runtime_memory_mb` | `15` |
| `estimated_memory_mb` | `15` |
| `independence_class` | `heuristic` |
| `hard_requires` | `[{ analyzer_id: "image-enumerator", evidence_type: "IMAGE_LIST" }]` |
| `correlated_with` | `["noise-analyzer"]` (both detect editing artifacts — not fully independent) |
| `privacy.tier` | `LOCAL_ONLY` |
| `failure_modes` | `[{ error_code: "NOT_JPEG", classification: "UNSUPPORTED_CONTENT", message: "Error Level Analysis only works on JPEG images.", retryable: false }]` |

---

#### 2.4.8 Fingerprint Analyzer (AI Generator Detection)

| Field | Value |
|---|---|
| `analyzer_id` | `ai-fingerprint-detector` |
| `plugin_type` | `built-in` |
| `content_types` | `["image/jpeg", "image/png", "image/webp"]` |
| `produces_evidence_types` | `["FORENSIC"]` |
| `produces_signal_tier` | `2` (Strong — frequency-domain artifacts) |
| `required_apis` | `["onnx"]` |
| `execution.context` | `offscreen_document` |
| `latency_class` | `STANDARD` |
| `estimated_runtime_ms` | `{ best_case: 200, typical: 800, worst_case: 3000 }` |
| `model_size_mb` | `20` (lightweight classifier, e.g., EfficientNet-B0 finetuned) |
| `runtime_memory_mb` | `15` |
| `estimated_memory_mb` | `35` |
| `gpu_accelerated` | `true` (WebGPU reduces inference by 3-5x) |
| `independence_class` | `ml_statistical` |
| `hard_requires` | `[{ analyzer_id: "image-enumerator", evidence_type: "IMAGE_LIST" }]` |
| `enriches_from` | `[{ analyzer_id: "noise-analyzer", evidence_type: "FORENSIC", how_enriched: "Noise residuals can be used as additional features for the classifier." }]` |
| `privacy.tier` | `LOCAL_ONLY` |
| **Critical constraint:** Model accuracy degrades 10-20% per new AI model generation. The plugin must have a `version` that corresponds to its training data recency. A model trained on 2024 generators receives increasing weight decay after 12 months (per Product Reasoning Spec). |
| `failure_modes` | `[{ error_code: "MODEL_LOAD_FAILED", classification: "INTERNAL_ERROR", message: "AI fingerprint model failed to load. Try reinstalling the extension.", retryable: false }, { error_code: "LOW_CONFIDENCE", classification: "INCONCLUSIVE", message: "Could not match image to any known AI generator fingerprint. This does not mean the image is authentic — novel generators may not be in our database.", retryable: false }]` |

---

#### 2.4.9 Watermark Analyzer

| Field | Value |
|---|---|
| `analyzer_id` | `watermark-detector` |
| `plugin_type` | `built-in` |
| `content_types` | `["image/jpeg", "image/png", "image/webp"]` |
| `produces_evidence_types` | `["FORENSIC"]` |
| `produces_signal_tier` | `2` (Strong when detected — DALL-E/Midjourney are high-confidence) |
| `required_apis` | `[]` |
| `execution.context` | `offscreen_document` |
| `latency_class` | `FAST` |
| `estimated_runtime_ms` | `{ best_case: 20, typical: 100, worst_case: 500 }` |
| `model_size_mb` | `0` (algorithmic detection — checks for known watermark patterns by pixel/frequency domain inspection) |
| `runtime_memory_mb` | `10` |
| `estimated_memory_mb` | `10` |
| `independence_class` | `heuristic` |
| `hard_requires` | `[{ analyzer_id: "image-enumerator", evidence_type: "IMAGE_LIST" }]` |
| `privacy.tier` | `LOCAL_ONLY` |
| **Detects:**
- DALL-E visible color band (bottom-right corner)
- Midjourney frequency-domain watermark
- Adobe Firefly C2PA manifest (delegates to C2PA analyzer)
- Stable Diffusion SDXL invisible watermark (if embedding method known)
- SynthID: **NOT DETECTABLE locally** (proprietary Google technology). The watermark analyzer produces `INCONCLUSIVE` for SynthID with message: "SynthID watermark detection requires Google's API (unavailable)." |
| `failure_modes` | `[{ error_code: "NO_WATERMARK", classification: "INCONCLUSIVE", message: "No known watermark detected. Many AI generators do not watermark their outputs.", retryable: false }]` |

---

#### 2.4.10 Video Analyzer (Face-Swap / Temporal)

| Field | Value |
|---|---|
| `analyzer_id` | `video-face-swap-detector` |
| `plugin_type` | `built-in` |
| `content_types` | `["video/mp4", "video/webm"]` |
| `produces_evidence_types` | `["FORENSIC"]` |
| `produces_signal_tier` | `3` (Moderate — face-swap only, not fully AI-generated video) |
| `required_apis` | `["tabCapture", "onnx"]` |
| `supported_browsers` | `["chrome", "edge"]` (Firefox and Safari excluded — no tabCapture) |
| `browser_notes` | `{ "firefox": "Unavailable. Firefox does not support tabCapture.", "safari": "Unavailable. Safari does not support tabCapture." }` |
| `execution.context` | `tab_capture` (MediaStream from tab) → `offscreen_document` (frame analysis) |
| `latency_class` | `SLOW` |
| `estimated_runtime_ms` | `{ best_case: 2000, typical: 5000, worst_case: 15000 }` (processes sampled frames, not every frame) |
| `model_size_mb` | `25` (Xception-based face forensics model) |
| `runtime_memory_mb` | `50` (decoded video frames, face crops, temporal buffers) |
| `estimated_memory_mb` | `75` |
| `requires_user_gesture` | `true` (tabCapture requires user gesture) |
| `network_required` | `false` |
| `independence_class` | `ml_statistical` |
| `hard_requires` | `[{ analyzer_id: "video-detector", evidence_type: "VIDEO_ELEMENTS", reason: "Needs to know which video elements exist on the page to capture the correct region." }]` |
| `privacy.tier` | `LOCAL_ONLY` |
| **Critical constraint:** Only detects face-swap deepfakes (80-90% accuracy). Does NOT detect fully AI-generated video (Sora, Runway, Veo) — this requires fundamentally different (and currently unreliable) detection approaches. The UI must display this limitation. |
| `failure_modes` | `[{ error_code: "NO_FACE_DETECTED", classification: "INCONCLUSIVE", message: "No faces detected in video to analyze.", retryable: false }, { error_code: "TABCAPTURE_DENIED", classification: "ERROR", message: "Video capture requires your permission. Please grant access when prompted.", retryable: false }, { error_code: "LOW_RESOLUTION", classification: "INCONCLUSIVE", message: "Video resolution too low for reliable face analysis (minimum 480p).", retryable: false }]` |

---

#### 2.4.11 Audio Analyzer (Vocoder Artifacts)

| Field | Value |
|---|---|
| `analyzer_id` | `audio-vocoder-detector` |
| `plugin_type` | `built-in` |
| `content_types` | `["audio/wav", "audio/mp3", "audio/mpeg", "audio/ogg", "audio/webm"]` |
| `produces_evidence_types` | `["FORENSIC"]` |
| `produces_signal_tier` | `3` (Moderate — degrades rapidly with compression) |
| `required_apis` | `["tabCapture", "onnx"]` |
| `supported_browsers` | `["chrome", "edge"]` (tabCapture) |
| `browser_notes` | `{ "firefox": "Limited. Can analyze audio from <audio>/<video> elements via AudioContext.createMediaElementSource(), but not arbitrary page audio.", "safari": "Limited. Same constraint as Firefox plus fewer AudioContext APIs." }` |
| `execution.context` | `tab_capture` (MediaStreamTrack) → `offscreen_document` (audio buffer analysis) |
| `fallback_context` | `offscreen_document` (when using `createMediaElementSource()` — Firefox/Safari fallback) |
| `latency_class` | `STANDARD` |
| `estimated_runtime_ms` | `{ best_case: 500, typical: 2000, worst_case: 5000 }` (3-10 second audio samples) |
| `model_size_mb` | `15` (lightweight anti-spoofing model) |
| `runtime_memory_mb` | `15` (audio sample buffer, spectrogram) |
| `estimated_memory_mb` | `30` |
| `requires_user_gesture` | `true` (tabCapture requires user gesture) |
| `independence_class` | `ml_statistical` |
| `hard_requires` | `[{ analyzer_id: "audio-detector", evidence_type: "AUDIO_ELEMENTS" }]` |
| `privacy.tier` | `LOCAL_ONLY` |
| **Critical constraint:** Detection accuracy degrades to near-random after telephony compression, background noise, or re-encoding. Most web audio has been through multiple compression stages. UI must display: "Audio analysis works best on original source audio. Compressed or noisy audio may produce unreliable results." |
| `failure_modes` | `[{ error_code: "NO_AUDIO_SOURCE", classification: "INCONCLUSIVE", message: "No audio source available for analysis.", retryable: false }, { error_code: "DRM_PROTECTED", classification: "UNSUPPORTED_CONTENT", message: "Cannot analyze DRM-protected audio.", retryable: false }, { error_code: "HIGH_COMPRESSION", classification: "INCONCLUSIVE", message: "Audio is heavily compressed — vocoder artifact detection may be unreliable.", retryable: false }]` |

---

#### 2.4.12 Fact Check Analyzer

| Field | Value |
|---|---|
| `analyzer_id` | `fact-check-lookup` |
| `plugin_type` | `built-in` |
| `content_types` | `["text/plain", "text/html"]` |
| `min_text_length_words` | `50` (need enough text to extract a claim) |
| `produces_evidence_types` | `["REPUTATION"]` |
| `produces_signal_tier` | `3` (Moderate — when claim is in database) |
| `required_apis` | `[]` |
| `execution.context` | `service_worker` (network fetch) |
| `latency_class` | `STANDARD` |
| `estimated_runtime_ms` | `{ best_case: 300, typical: 1500, worst_case: 8000 }` |
| `model_size_mb` | `0` |
| `runtime_memory_mb` | `5` |
| `network_required` | `true` |
| `network_bandwidth_estimate_kbps` | `10` (small claim hash + response) |
| `independence_class` | `network_query` |
| `hard_requires` | `[{ analyzer_id: "text-extractor", evidence_type: "TEXT_BLOCKS" }]` |
| `privacy.tier` | `CONTENT_TRANSMITTED` (claim hashes sent, not full text) |
| `user_consent_required` | `true` |
| `consent_message` | "Fact-checking requires sending extracted claims (not your full text) to a fact-check database. No personally identifying information is sent. [Enable] [Skip]" |
| `data_transmitted` | "Claim hashes (SHA-256) and anonymized claim metadata. Full text content is never transmitted." |
| `failure_modes` | `[{ error_code: "NO_CLAIM_EXTRACTED", classification: "INCONCLUSIVE", message: "Could not extract factual claims from text for checking.", retryable: false }, { error_code: "OFFLINE", classification: "SKIPPED", message: "Fact-check database unavailable — offline.", retryable: false }, { error_code: "NO_MATCH", classification: "INCONCLUSIVE", message: "No matching fact-checks found. This does not mean the claim is true — it may not have been checked yet.", retryable: false }]` |

---

#### 2.4.13 Reverse Image Analyzer

| Field | Value |
|---|---|
| `analyzer_id` | `reverse-image-search` |
| `plugin_type` | `built-in` |
| `content_types` | `["image/jpeg", "image/png", "image/webp", "image/gif"]` |
| `produces_evidence_types` | `["FORENSIC"]` (contextual — prior web existence) |
| `produces_signal_tier` | `3` (Moderate — narrow applicability) |
| `required_apis` | `[]` |
| `execution.context` | `service_worker` (network fetch) |
| `latency_class` | `STANDARD` |
| `estimated_runtime_ms` | `{ best_case: 500, typical: 2000, worst_case: 10000 }` |
| `model_size_mb` | `0` |
| `runtime_memory_mb` | `5` |
| `network_required` | `true` |
| `network_bandwidth_estimate_kbps` | `50` (image perceptual hash uploaded, not full image) |
| `independence_class` | `network_query` |
| `hard_requires` | `[{ analyzer_id: "image-enumerator", evidence_type: "IMAGE_LIST" }]` |
| `privacy.tier` | `CONTENT_TRANSMITTED` (image perceptual hash transmitted, not the image itself) |
| `user_consent_required` | `true` |
| `consent_message` | "Reverse image search sends a fingerprint of the image (not the image itself) to a search service to check if it has appeared elsewhere on the web. [Enable] [Skip]" |
| `data_transmitted` | "Image perceptual hash (pHash/DCT hash). Thumbnail preview may be uploaded for visual matching services. Full-resolution image is never transmitted." |
| `failure_modes` | `[{ error_code: "OFFLINE", classification: "SKIPPED", message: "Reverse image search unavailable — offline.", retryable: false }, { error_code: "NO_MATCHES", classification: "INCONCLUSIVE", message: "Image not found elsewhere on the web. This is expected for newly created or never-published images.", retryable: false }, { error_code: "SEARCH_ERROR", classification: "ERROR", message: "Image search service returned an error. Try again later.", retryable: true }]` |

---

### 2.5 Summary: Plugin Manifest Declarations Table

| Analyzer | Plugin Type | Context | Latency | Signal Tier | Model (MB) | Memory (MB) | Network | Key Constraint |
|---|---|---|---|---|---|---|---|---|
| C2PA Verifier | built-in | offscreen | FAST | 1 — Ground Truth | 3 | 8 | No | CORS-dependent |
| EXIF Extractor | built-in | content_script | IMMEDIATE | 3 — Moderate | 0 | 2 | No | Often stripped by platforms |
| Image Metadata | built-in | content_script | IMMEDIATE | 3 — Moderate | 0 | 2 | No | Correlated with EXIF |
| Text Pattern | built-in | offscreen | FAST | 4 — Weak | 15 | 25 | No | Min 100 words; never standalone |
| Domain Reputation | built-in | service_worker | IMMEDIATE | 4 — Weak | 0 | 5 | Cached (no) / Cloud (yes) | Staleness decay |
| Noise Analyzer | built-in | offscreen | STANDARD | 2 — Strong | 0 | 20 | No | Degrades with compression |
| ELA Analyzer | built-in | offscreen | STANDARD | 2 — Strong | 0 | 15 | No | JPEG only |
| Fingerprint Detector | built-in | offscreen | STANDARD | 2 — Strong | 20 | 35 | No | Accuracy decays 10-20%/year |
| Watermark Detector | built-in | offscreen | FAST | 2 — Strong | 0 | 10 | No | Coverage limited to known watermarks |
| Video Face-Swap | built-in | tab_capture | SLOW | 3 — Moderate | 25 | 75 | No | Chrome/Edge only; user gesture required |
| Audio Vocoder | built-in | tab_capture | STANDARD | 3 — Moderate | 15 | 30 | No | Degrades to near-random after compression |
| Fact Check Lookup | built-in | service_worker | STANDARD | 3 — Moderate | 0 | 5 | Yes | Claim hashes transmitted; opt-in |
| Reverse Image Search | built-in | service_worker | STANDARD | 3 — Moderate | 0 | 5 | Yes | Perceptual hash transmitted; opt-in |

---

## Appendix A: Analyzer ID Reference

All analyzer IDs referenced in the dependency graph:

| Analyzer ID | Purpose |
|---|---|
| `c2pa-verifier` | C2PA manifest verification |
| `exif-extractor` | EXIF metadata extraction |
| `image-metadata-extractor` | IPTC/XMP/ICC metadata extraction |
| `text-pattern-analyzer` | Perplexity, burstiness, style analysis |
| `domain-reputation` | Domain/publisher reputation (cached + cloud) |
| `noise-analyzer` | Noise pattern/consistency analysis |
| `ela-analyzer` | Error Level Analysis (JPEG editing detection) |
| `ai-fingerprint-detector` | AI generator fingerprint classification |
| `watermark-detector` | Known watermark detection (DALL-E, Midjourney, etc.) |
| `video-face-swap-detector` | Face-swap deepfake detection in video |
| `audio-vocoder-detector` | Vocoder artifact detection in audio |
| `fact-check-lookup` | Fact-check database claim verification |
| `reverse-image-search` | Reverse image search for prior web existence |
| `image-enumerator` | DOM image element detection (pre-requisite for all image analyzers) |
| `text-extractor` | DOM text block extraction (pre-requisite for text analyzers) |
| `url-extractor` | Page URL/domain extraction (pre-requisite for domain reputation) |
| `video-detector` | Video element detection on page |
| `audio-detector` | Audio element detection on page |
| `content-classifier` | Page type classification (article, social media, gallery, etc.) |

---

*End of Scheduler & Analyzer Plugin Architecture.*
