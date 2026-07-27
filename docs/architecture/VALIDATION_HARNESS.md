# Sentinel — Validation Harness Architecture

> **Document Type:** Engineering Infrastructure Design
> **Phase:** 1.5 — Pre-Implementation Validation
> **Status:** Draft
> **Owner:** Lead Architect
> **Audience:** Spike Implementors, QA, Release Engineer
> **Depends On:** `VALIDATION_PLAN.md`
> **Output:** Reusable validation platform for all 32 technical spikes

---

## Purpose

The Validation Harness is a reusable MV3 browser extension that provides ALL infrastructure needed to execute, benchmark, profile, log, and report on technical spikes. It is NOT production code. It is a disposable engineering validation platform.

**Design principle:** The harness is the platform. Spikes are plugins. Spikes implement test logic only; the harness handles everything else.

---

## Section 1: Validation Harness Architecture

### 1.1 Architectural Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     VALIDATION HARNESS (MV3 Extension)                    │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                     SPIKE REGISTRY                                    │ │
│  │  Spike manifests declare requirements. Harness discovers + wires.    │ │
│  │  spikes/spike-c0/manifest.json  →  registered as "spike-c0"         │ │
│  │  spikes/spike-c1/manifest.json  →  registered as "spike-c1"         │ │
│  └───────────────────────────────┬─────────────────────────────────────┘ │
│                                  │                                        │
│  ┌───────────────────────────────┼─────────────────────────────────────┐ │
│  │                      HARNESS CORE                                    │ │
│  │                                                                      │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │ │
│  │  │ Message  │  │  Logger  │  │Benchmark │  │  Profiler        │   │ │
│  │  │ Bus      │  │          │  │ Engine   │  │                  │   │ │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │ │
│  │       └──────────────┴────────────┴───────────────┘               │ │
│  │                          │                                         │ │
│  │  ┌─────────────────────────────────────────────────────────────┐  │ │
│  │  │                 SPIKE ORCHESTRATOR                           │  │ │
│  │  │  • Discovers registered spikes                              │  │ │
│  │  │  • Routes messages to active spike                          │  │ │
│  │  │  • Manages spike lifecycle (LOAD → INIT → RUN → DONE)       │  │ │
│  │  │  • Collects benchmark + profile data per spike              │  │ │
│  │  │  • Generates spike reports                                  │  │ │
│  │  └─────────────────────────────────────────────────────────────┘  │ │
│  │                                                                      │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │ │
│  │  │ Capability│  │ Fixture  │  │ Report   │  │  Storage         │   │ │
│  │  │ Detector │  │ Registry │  │ Generator│  │  (session/local) │   │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                     BROWSER CONTEXTS                                  │ │
│  │                                                                       │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │ │
│  │  │ Content      │  │ Service      │  │ Sandbox Page             │  │ │
│  │  │ Script       │  │ Worker       │  │ • WASM execution         │  │ │
│  │  │ • DOM access │  │ • Orchestr.  │  │ • ONNX inference         │  │ │
│  │  │ • Metadata   │  │ • Scheduling │  │ • Heavy compute          │  │ │
│  │  │ • Extraction │  │ • Routing    │  │ • MessagePort only       │  │ │
│  │  └──────┬───────┘  └──────┬───────┘  └────────────┬─────────────┘  │ │
│  │         │                 │                        │                 │ │
│  │  ┌──────┴───────┐  ┌──────┴───────┐  ┌────────────┴─────────────┐  │ │
│  │  │ Popup        │  │ Side Panel   │  │ Offscreen Document       │  │ │
│  │  │ • Trigger    │  │ • Persist UI │  │ • IndexedDB proxy        │  │ │
│  │  │ • Progress   │  │ • Results    │  │ • tabCapture consumer    │  │ │
│  │  │ • Control    │  │ • Reports    │  │ • Multi-modal compute    │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────────────────┘  │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Harness as the Extension

The harness IS the MV3 extension. There is exactly one `manifest.json` at the repository root. All browser contexts (content script, service worker, sandbox page, offscreen document, popup, side panel) are defined in this single manifest. Spikes do NOT have their own manifests — they declare requirements in a spike manifest that the harness reads.

**Single manifest approach rationale:**
- One extension to load per browser (not 32)
- Shared permissions — declared once, spikes request subsets
- Single content script per page (not N competing scripts)
- Single service worker for orchestration
- Consistent CSP, storage, and API access

### 1.3 Context Ownership

| Context | Owned By | Spike Access |
|---------|----------|-------------|
| **Service Worker** | Harness (orchestrator, router, logger, profiler) | Spikes register message handlers. Harness routes to active spike. |
| **Content Script** | Harness (DOM extraction harness) | Spikes request DOM data via message. Spikes do NOT inject their own content scripts. |
| **Sandbox Page** | Harness (WASM runtime, ONNX runtime) | Spikes provide compute payloads. Harness manages sandbox lifecycle and model loading. |
| **Offscreen Document** | Harness (IndexedDB proxy, tabCapture consumer) | Spikes request storage operations or media capture. Harness proxies. |
| **Popup** | Harness (spike selector, run button, progress display) | Spikes provide UI summary. Harness renders in popup. |
| **Side Panel** | Harness (detailed results, report viewer) | Spikes provide detailed output. Harness renders in side panel. |

**Design rationale:** Spikes never directly manage browser contexts. This prevents 32 different service worker implementations, 32 different sandbox page configurations, and 32 different content script injection strategies. The harness provides stable, tested context management; spikes provide test logic.

---

## Section 2: Repository Structure

```
sentinel-validation/
│
├── manifest.json                    # SINGLE MV3 manifest for entire harness
├── package.json                     # Dev dependencies (TypeScript, build tools, test runner)
├── tsconfig.json                    # Shared TypeScript configuration
├── README.md                        # How to load harness, run spikes, read reports
│
├── harness/                         # ===== HARNESS CORE (not disposable) =====
│   │
│   ├── core/                        # Core orchestration
│   │   ├── orchestrator.ts          # Spike lifecycle: discover → load → init → run → report
│   │   ├── registry.ts              # Spike manifest registry. Discovers spikes/, validates manifests
│   │   └── router.ts                # Message router: routes messages to active spike handlers
│   │
│   ├── messaging/                   # Cross-context communication
│   │   ├── bus.ts                   # MessageBus: typed publish/subscribe across contexts
│   │   ├── channels.ts              # Channel definitions: CS↔SW, SW↔Sandbox, SW↔Popup, SW↔Offscreen
│   │   └── serialization.ts         # structuredClone wrapper, size validation, integrity check
│   │
│   ├── contexts/                    # Browser context management
│   │   ├── service-worker.ts        # SW entry point. Initializes orchestrator, router, profiler.
│   │   ├── content-script.ts        # CS entry point. DOM extraction harness, fixture injection.
│   │   ├── sandbox/
│   │   │   ├── sandbox.html         # Sandbox page HTML (single page for all spikes)
│   │   │   └── sandbox-runtime.ts   # WASM/ONNX runtime manager. Executes spike payloads.
│   │   ├── offscreen/
│   │   │   ├── offscreen.html       # Offscreen document HTML
│   │   │   └── offscreen-runtime.ts # IndexedDB proxy, tabCapture consumer
│   │   ├── popup/
│   │   │   ├── popup.html           # Popup UI shell
│   │   │   ├── popup.ts             # Spike selector, run/stop controls, progress display
│   │   │   └── popup.css
│   │   └── sidepanel/
│   │       ├── sidepanel.html       # Side panel UI shell
│   │       ├── sidepanel.ts         # Detailed results, report viewer, spike history
│   │       └── sidepanel.css
│   │
│   ├── benchmark/                   # Benchmark framework
│   │   ├── engine.ts                # BenchmarkEngine: start/stop/lap. Auto-records metrics.
│   │   ├── metrics.ts               # Metric definitions (startup_latency, inference_time, etc.)
│   │   ├── timer.ts                 # High-resolution timer (performance.now() + performance.mark)
│   │   └── recorder.ts              # Records benchmark runs to structured results
│   │
│   ├── profiling/                   # Profiling system
│   │   ├── memory.ts                # Heap snapshot, peak RSS, allocation timeline
│   │   ├── cpu.ts                   # CPU utilization (rough, via timing)
│   │   ├── lifecycle.ts             # SW lifetime tracker, context creation/destruction events
│   │   └── events.ts                # Event counter: messages sent/received, context switches
│   │
│   ├── logging/                     # Structured logging
│   │   ├── logger.ts                # Logger: severity levels, context annotation, correlation IDs
│   │   ├── formats.ts               # Log entry schema (timestamp, context, spike, severity, message)
│   │   ├── buffer.ts                # Ring buffer for in-memory logs (survives SW termination)
│   │   └── replay.ts                # Log replay: reconstruct spike execution timeline from logs
│   │
│   ├── reporting/                   # Report generation
│   │   ├── generator.ts             # ReportGenerator: collects results → produces markdown/JSON
│   │   ├── templates.ts             # Report templates (pass/fail summary, metrics table, graphs)
│   │   ├── compare.ts               # Cross-spike comparison (e.g., "same metric across spikes")
│   │   └── export.ts                # Export report as .md, .json, .html
│   │
│   ├── fixtures/                    # Fixture system
│   │   ├── registry.ts              # FixtureRegistry: discover fixtures/, resolve by ID
│   │   ├── loader.ts                # FixtureLoader: load fixture into context (image→ArrayBuffer, HTML→DOM)
│   │   ├── catalog.ts               # FixtureCatalog: all available fixtures with metadata
│   │   └── generator.ts             # FixtureGenerator: programmatic fixture creation (e.g., generate CORS page)
│   │
│   ├── capabilities/                # Browser capability detection
│   │   ├── detector.ts              # CapabilityDetector: static + runtime probes
│   │   ├── profile.ts               # CapabilityProfile: typed result of all probes
│   │   └── probes/                  # Individual probe implementations
│   │       ├── webgpu.ts
│   │       ├── offscreen.ts
│   │       ├── tabcapture.ts
│   │       ├── onnx.ts
│   │       ├── wasm.ts
│   │       ├── session-storage.ts
│   │       ├── indexeddb.ts
│   │       ├── shared-array-buffer.ts
│   │       └── chrome-ai.ts
│   │
│   ├── storage/                     # Storage utilities
│   │   ├── session.ts               # chrome.storage.session wrapper (with quota tracking)
│   │   ├── local.ts                 # chrome.storage.local wrapper
│   │   ├── indexeddb.ts             # IndexedDB wrapper (via offscreen proxy)
│   │   └── quota.ts                 # Quota monitoring: warn when approaching limits
│   │
│   └── utils/                       # Shared utilities
│       ├── hash.ts                  # SHA-256 hashing (SubtleCrypto)
│       ├── assert.ts                # Assertion helpers (fail-fast with structured error)
│       ├── timing.ts                # performance.now() wrappers, timeout helpers
│       ├── ids.ts                   # UUID v7 generation
│       └── types.ts                 # Shared type definitions
│
├── spikes/                          # ===== SPIKE IMPLEMENTATIONS (disposable) =====
│   │
│   ├── spike-c0/                    # SPIKE-C0: End-to-End Architecture Validation
│   │   ├── manifest.json            # Spike manifest (declares requirements)
│   │   ├── spike.ts                 # Spike implementation (entry point)
│   │   ├── scenarios/               # Test scenarios
│   │   │   ├── message-roundtrip.ts
│   │   │   ├── context-switching.ts
│   │   │   ├── lifecycle-survival.ts
│   │   │   └── cancellation.ts
│   │   └── README.md
│   │
│   ├── spike-c1/                    # SPIKE-C1: C2PA WASM Validation
│   │   ├── manifest.json
│   │   ├── spike.ts
│   │   ├── scenarios/
│   │   │   ├── verify-valid-c2pa.ts
│   │   │   ├── verify-tampered-c2pa.ts
│   │   │   └── verify-no-c2pa.ts
│   │   └── README.md
│   │
│   ├── spike-c2/                    # SPIKE-C2: ONNX Runtime Web
│   │   └── ...
│   │
│   ├── spike-c3/                    # SPIKE-C3: SW Keep-Alive
│   │   └── ...
│   │
│   ├── spike-c4/                    # SPIKE-C4: Cross-Context Messaging
│   │   └── ...
│   │
│   ├── spike-c5/                    # SPIKE-C5: MessagePort Compute
│   │   └── ...
│   │
│   ├── spike-c6/                    # SPIKE-C6: Offscreen IndexedDB Proxy
│   │   └── ...
│   │
│   ├── spike-c7/                    # SPIKE-C7: Cross-Origin Fetch
│   │   └── ...
│   │
│   ├── spike-h1/                    # SPIKE-H1 through H7
│   │   └── ...
│   ├── spike-h2/
│   ├── spike-h3/
│   ├── spike-h4/
│   ├── spike-h5/
│   ├── spike-h6/
│   ├── spike-h7/
│   │
│   ├── spike-m1/                    # SPIKE-M1 through M7
│   │   └── ...
│   ├── spike-m2/
│   ├── spike-m3/
│   ├── spike-m4/
│   ├── spike-m5/
│   ├── spike-m6/
│   ├── spike-m7/
│   │
│   └── spike-l1/                    # SPIKE-L1 through L8
│       └── ...
│
├── fixtures/                        # ===== FIXTURE DATA =====
│   │
│   ├── catalog.json                 # Fixture catalog (index of all fixtures with metadata)
│   │
│   ├── images/
│   │   ├── authentic/
│   │   │   ├── photo-dslr.jpg       # DSLR photo with EXIF
│   │   │   ├── photo-phone.jpg      # Phone photo with EXIF
│   │   │   └── screenshot-real.png  # Authentic screenshot
│   │   ├── c2pa/
│   │   │   ├── firefly-valid.jpg    # Adobe Firefly: valid C2PA, declares AI-generated
│   │   │   ├── photoshop-valid.jpg  # Photoshop: valid C2PA, edit history
│   │   │   ├── leica-signed.jpg     # Leica M11-P: hardware-attested C2PA
│   │   │   ├── c2pa-tampered.jpg    # C2PA present but signature invalid (tampered)
│   │   │   └── c2pa-unknown-ca.jpg  # C2PA from unrecognized certificate authority
│   │   ├── ai-generated/
│   │   │   ├── dalle3-output.png    # DALL-E 3 output
│   │   │   ├── midjourney-v6.jpg    # Midjourney v6 output
│   │   │   ├── sd3-output.png       # Stable Diffusion 3 output
│   │   │   └── sora-frame.png       # Sora video frame
│   │   ├── manipulated/
│   │   │   ├── spliced.jpg          # Spliced composite (two real photos merged)
│   │   │   ├── inpainted.jpg        # AI-inpainted region on real photo
│   │   │   └── deepfake-face.jpg    # Face-swap deepfake
│   │   ├── edge-cases/
│   │   │   ├── tiny-10x10.png       # Below minimum analysis size
│   │   │   ├── huge-8000x6000.jpg   # Above typical web resolution
│   │   │   ├── grayscale.jpg        # No color channels
│   │   │   ├── animated.gif         # Multi-frame GIF
│   │   │   └── corrupted.jpg        # Truncated file
│   │   └── metadata/
│   │       ├── exif-complete.jpg    # All EXIF fields populated
│   │       ├── exif-stripped.jpg    # EXIF intentionally removed
│   │       ├── exif-inconsistent.jpg# EXIF claims iPhone but ICC profile says Canon
│   │       └── iptc-news.jpg        # IPTC fields with news credit/caption
│   │
│   ├── html/
│   │   ├── spa-react.html           # React SPA with client-side rendering
│   │   ├── infinite-scroll.html     # Infinite scroll page
│   │   ├── shadow-dom-open.html     # Open shadow DOM
│   │   ├── shadow-dom-closed.html   # Closed shadow DOM (inaccessible)
│   │   ├── cors-images.html         # Page with cross-origin images
│   │   ├── same-origin-images.html  # Page with same-origin images
│   │   ├── mixed-content.html       # Page with text + images + video
│   │   ├── paywall.html             # Simulated paywall
│   │   ├── social-twitter.html      # Simulated Twitter/X post
│   │   ├── social-instagram.html    # Simulated Instagram post
│   │   └── blank.html               # Empty page (control)
│   │
│   ├── video/
│   │   ├── real-clip.mp4            # Authentic video clip
│   │   ├── deepfake-face.mp4        # Face-swap deepfake
│   │   ├── ai-generated-sora.mp4    # AI-generated (Sora) clip
│   │   └── screen-recording.mp4     # Screen recording (no provenance)
│   │
│   ├── audio/
│   │   ├── real-speech.wav          # Authentic human speech
│   │   ├── tts-elevenlabs.wav       # ElevenLabs TTS output
│   │   ├── tts-openai.wav           # OpenAI TTS output
│   │   └── phone-quality.mp3        # Real speech, telephony compressed
│   │
│   └── cors/
│       ├── server/
│       │   ├── cors-enabled.js      # Simple Node server with CORS headers
│       │   └── cors-disabled.js     # Simple Node server without CORS headers
│       └── test-urls.json           # List of real-world URLs for fetch testing
│
├── reports/                         # ===== GENERATED REPORTS =====
│   ├── capabilities.json            # Capability profile (generated on harness load)
│   ├── spike-c0/
│   │   ├── report.md
│   │   ├── metrics.json
│   │   └── profile.json
│   ├── spike-c1/
│   │   └── ...
│   └── summary.md                   # Cross-spike summary report
│
├── scripts/                         # ===== BUILD & RUN SCRIPTS =====
│   ├── build.ts                     # Build harness + selected spike (esbuild/rollup)
│   ├── run.ts                       # Load extension in browser, execute spike, capture results
│   ├── serve-fixtures.ts            # Start local fixture server (for CORS testing)
│   ├── clean.ts                     # Clear reports, clear storage, reset state
│   └── capabilities.ts              # Detect capabilities and write capabilities.json
│
├── docs/                            # ===== HARNESS DOCUMENTATION =====
│   ├── HARNESS_ARCHITECTURE.md      # This document
│   ├── SPIKE_AUTHORING_GUIDE.md     # How to write a spike
│   ├── FIXTURE_GUIDE.md             # Available fixtures and how to use them
│   └── BENCHMARK_GUIDE.md           # How to use the benchmark framework
│
└── .gitignore
```

---

## Section 3: Spike Registration System

### 3.1 Spike Manifest Schema

Every spike directory contains a `manifest.json` that declares the spike's identity, requirements, and execution contract. The harness discovers these at build time and validates them at runtime.

```jsonc
{
  // ===== IDENTITY =====
  "spike_id": "spike-c1",                        // Unique. Matches directory name.
  "name": "C2PA WASM Validation",                // Human-readable
  "objective": "Can c2pa-js verify C2PA manifests in an MV3 sandbox page context?",
  "assumption_validated": "A-C1",                // Reference to VALIDATION_PLAN.md assumption ID
  "priority": "critical",                         // critical | high | medium | low

  // ===== BROWSER SUPPORT =====
  "browsers": {
    "chrome": "required",                        // required | optional | unsupported
    "firefox": "required",
    "edge": "optional",
    "safari": "optional"
  },

  // ===== PERMISSIONS =====
  "permissions": {
    "manifest_permissions": ["activeTab", "storage"],
    "host_permissions": ["<all_urls>"],
    "optional_permissions": [],
    "manifest_keys": {
      "sandbox": {
        "pages": ["harness/contexts/sandbox/sandbox.html"]
      }
    }
  },

  // ===== REQUIRED CONTEXTS =====
  "required_contexts": [
    "service_worker",                              // Always required
    "content_script",                              // If spike needs DOM access
    "sandbox_page",                                // If spike needs WASM/ONNX compute
    "offscreen_document",                          // If spike needs offscreen doc
    "popup"                                        // If spike has UI
  ],

  // ===== REQUIRED CAPABILITIES =====
  "required_capabilities": [
    "supportsWASM",                                // From CapabilityDetector
    "supportsSandbox",
    "supportsMessagePort"
  ],

  // ===== EXPECTED RUNTIME =====
  "expected_runtime": {
    "minimum_ms": 1000,                            // Minimum expected execution time
    "maximum_ms": 30000,                           // Timeout — spike fails if exceeded
    "target_ms": 5000                              // Benchmark target
  },

  // ===== TEST CONTRACT =====
  "scenarios": [                                   // Named test scenarios within the spike
    {
      "id": "verify-valid-c2pa",
      "description": "Verify a known-valid C2PA-signed JPEG",
      "expected_result": "pass"
    },
    {
      "id": "verify-tampered-c2pa",
      "description": "Reject a tampered C2PA JPEG",
      "expected_result": "pass"
    }
  ],

  // ===== BENCHMARK HOOKS =====
  "benchmarks": [
    {
      "id": "c2pa_init_time",
      "metric": "startup_latency",
      "target_ms": 2000,
      "description": "Time from sandbox page load to c2pa-js ready"
    },
    {
      "id": "c2pa_verify_time",
      "metric": "execution_time",
      "target_ms": 200,
      "description": "Time to verify a single C2PA manifest"
    }
  ],

  // ===== FIXTURES REQUIRED =====
  "fixtures": [
    "images/c2pa/firefly-valid.jpg",
    "images/c2pa/c2pa-tampered.jpg",
    "images/authentic/photo-dslr.jpg"
  ],

  // ===== DEPENDENCIES =====
  "depends_on_spikes": [],                         // Must run after these spikes pass
  "depends_on_capabilities": ["supportsWASM"],     // Skip if capability not available

  // ===== SKIP CONDITIONS =====
  "skip_if": {
    "capabilities_missing": ["supportsWASM"],      // Skip entirely if these are missing
    "browsers": ["safari"],                        // Skip on these browsers
    "spikes_failed": []                            // Skip if these prerequisite spikes failed
  }
}
```

### 3.2 Spike Implementation Interface

Every spike exports a single `SpikeImplementation` object:

```typescript
interface SpikeImplementation {
  /** Called once when the spike is loaded. Set up message handlers, allocate resources. */
  initialize(ctx: SpikeContext): Promise<void>;

  /** Called to execute the spike. Run scenarios, produce results. */
  run(ctx: SpikeContext): Promise<SpikeResult>;

  /** Called after run() completes or on cancellation. Release resources. */
  cleanup(ctx: SpikeContext): Promise<void>;
}

interface SpikeContext {
  spikeId: string;
  manifest: SpikeManifest;
  capabilities: CapabilityProfile;

  // Harness services available to the spike
  messaging: MessageBus;          // Send/receive messages across contexts
  benchmark: BenchmarkEngine;     // Start/stop/lap benchmarks
  profiling: Profiler;            // Memory snapshots, event counts
  logging: Logger;                // Structured logging
  fixtures: FixtureLoader;        // Load fixtures by ID
  storage: StorageService;        // session/local/IndexedDB access
  sandbox: SandboxProxy;          // Invoke sandbox compute
  offscreen: OffscreenProxy;      // Invoke offscreen document operations

  // Lifecycle
  signal: AbortSignal;            // Cancelled if spike times out or user stops
  onProgress: (update: string) => void;  // Report progress to UI
}

interface SpikeResult {
  spikeId: string;
  status: 'PASS' | 'FAIL' | 'SKIP';
  scenarios: ScenarioResult[];
  benchmarks: BenchmarkResult[];
  profile: ProfileSnapshot;
  logs: LogEntry[];
  error?: string;
  duration: number;
}
```

### 3.3 Registration Flow

```
BUILD TIME:
  1. Build script scans spikes/*/manifest.json
  2. Collects all required permissions → generates merged manifest.json
  3. Collects all required capabilities → generates capability check list
  4. Collects all required contexts → enables those contexts
  5. Generates spike registry module (static import map)

RUNTIME:
  1. Service worker starts → HarnessOrchestrator.initialize()
  2. Orchestrator loads CapabilityProfile
  3. Orchestrator reads spike registry
  4. For each spike: check skip_if conditions → mark ELIGIBLE or SKIPPED
  5. ELIGIBLE spikes appear in popup UI
  6. User selects spike → Orchestrator.load(spikeId) → spike.initialize()
  7. User clicks "Run" → Orchestrator.run(spikeId) → spike.run()
  8. spike.run() returns SpikeResult → Orchestrator generates report
```

### 3.4 Message Routing During Spike Execution

The harness owns all message handlers. Spikes register handlers for specific message types:

```typescript
// In spike.initialize():
ctx.messaging.on('sandbox:compute_result', (payload) => {
  // Spike-specific logic for handling sandbox results
  ctx.benchmark.lap('sandbox_roundtrip');
  collectedResults.push(payload);
});

// Spikes never call chrome.runtime.sendMessage directly.
// They use the harness messaging abstraction:
ctx.messaging.send('sandbox', 'compute_request', { model: 'c2pa', imageHash: 'abc123' });
```

The harness `MessageRouter` ensures:
- Only the active spike receives messages (no cross-spike interference)
- Messages are logged with correlation IDs
- Message latency is benchmarked automatically
- Unhandled messages produce warnings, not errors

---

## Section 4: Benchmark Framework

### 4.1 BenchmarkEngine API

```typescript
interface BenchmarkEngine {
  /** Start a named benchmark. Multiple benchmarks can run concurrently. */
  start(id: string): void;

  /** Stop a benchmark. Records elapsed time. */
  stop(id: string): BenchmarkSample;

  /** Record a lap within a running benchmark (for multi-phase measurements). */
  lap(id: string, label: string): void;

  /** Create a timer that auto-stops when the returned function is called. */
  measure(id: string): () => BenchmarkSample;

  /** Get all samples collected so far. */
  samples(): BenchmarkSample[];

  /** Reset all benchmarks for a fresh run. */
  reset(): void;
}

interface BenchmarkSample {
  id: string;
  startTime: number;        // performance.now() at start
  endTime: number;          // performance.now() at end
  duration: number;         // end - start (ms)
  laps: { label: string; elapsed: number }[];
}
```

### 4.2 Automatic Metrics

The harness automatically records these metrics for every spike without the spike requesting them:

| Metric | Description | Source |
|--------|-------------|--------|
| `spike_total_duration` | Time from initialize() to result | Orchestrator |
| `spike_init_duration` | Time spent in initialize() | Orchestrator |
| `spike_run_duration` | Time spent in run() | Orchestrator |
| `sw_startup_time` | Time from SW start to ready | SW lifecycle profiler |
| `capability_detection_time` | Time to complete CapabilityProfile | CapabilityDetector |
| `message_count_total` | Total messages sent/received during spike | MessageBus counter |
| `message_latency_p50` | Median message round-trip time | MessageBus timer |
| `message_latency_p95` | 95th percentile message round-trip | MessageBus timer |
| `sandbox_create_time` | Time to create and initialize sandbox page | SandboxRuntime |
| `offscreen_create_time` | Time to create offscreen document | OffscreenRuntime |
| `peak_memory_mb` | Peak JS heap size during spike | Profiler |
| `sw_terminations` | Number of SW terminations during spike | Lifecycle profiler |

### 4.3 Spike-Declared Benchmarks

Spikes declare additional benchmarks in their manifest:

```jsonc
"benchmarks": [
  {
    "id": "c2pa_init_time",
    "metric": "startup_latency",
    "target_ms": 2000
  }
]
```

The spike calls `ctx.benchmark.start('c2pa_init_time')` and `ctx.benchmark.stop('c2pa_init_time')` in their code. The harness validates that all declared benchmarks were executed and records whether targets were met.

---

## Section 5: Profiling System

### 5.1 Profiler API

```typescript
interface Profiler {
  /** Take a memory snapshot. Returns heap size, external memory, total. */
  snapshot(label: string): MemorySnapshot;

  /** Start tracking allocations between start/stop. */
  trackAllocations(): void;
  stopTrackingAllocations(): AllocationRecord[];

  /** Increment a named event counter. */
  count(event: string): void;

  /** Record a context switch event. */
  contextSwitch(from: BrowserContext, to: BrowserContext): void;

  /** Get the current profile snapshot. */
  getSnapshot(): ProfileSnapshot;
}

interface MemorySnapshot {
  label: string;
  timestamp: number;
  heapSize: number;           // performance.memory.usedJSHeapSize
  heapLimit: number;          // performance.memory.jsHeapSizeLimit
  totalHeap: number;          // performance.memory.totalJSHeapSize
  externalMemory?: number;    // If available from browser
}

interface ProfileSnapshot {
  memory: MemorySnapshot[];           // Timeline of memory snapshots
  allocations: AllocationRecord[];    // Objects allocated
  events: Record<string, number>;     // Event counters
  contextSwitches: number;            // Number of context transitions
  swTerminations: number;             // SW lifecycle terminations
  peakMemory: number;                 // Max heapSize observed
}
```

### 5.2 Automatic Profiling

The harness automatically profiles:
- **Memory:** Snapshot on spike start + every 5 seconds during execution + on spike end
- **Events:** Messages sent/received, context switches, SW lifecycle events
- **SW lifetime:** Creation time, termination time, termination reason (if available)

Spikes do NOT need to request profiling — it's always on. The overhead is minimal (memory snapshots via `performance.memory` are cheap).

---

## Section 6: Logging Architecture

### 6.1 Log Entry Schema

```typescript
interface LogEntry {
  /** ISO 8601 timestamp with milliseconds */
  timestamp: string;

  /** Correlation ID — same for all log entries within one spike run */
  correlationId: string;

  /** Which spike produced this log */
  spikeId: string;

  /** Which browser context produced this log */
  context: 'service_worker' | 'content_script' | 'sandbox' | 'offscreen' | 'popup' | 'sidepanel';

  /** Severity level */
  level: 'DEBUG' | 'INFO' | 'WARN' | 'ERROR' | 'FATAL';

  /** Subsystem: messaging, benchmark, sandbox, etc. */
  subsystem: string;

  /** Human-readable message */
  message: string;

  /** Structured data (optional) */
  data?: Record<string, unknown>;

  /** Duration in ms (for operation-timing logs) */
  duration?: number;

  /** Error object (for ERROR/FATAL logs) */
  error?: { name: string; message: string; stack?: string };
}
```

### 6.2 Logger API

```typescript
interface Logger {
  debug(message: string, data?: Record<string, unknown>): void;
  info(message: string, data?: Record<string, unknown>): void;
  warn(message: string, data?: Record<string, unknown>): void;
  error(message: string, error?: Error, data?: Record<string, unknown>): void;
  fatal(message: string, error?: Error, data?: Record<string, unknown>): void;

  /** Time an operation. Logs duration on completion. */
  time(operation: string): () => void;

  /** Get all logs for the current spike run. */
  getLogs(): LogEntry[];
}
```

### 6.3 Log Persistence

- Logs are stored in an in-memory ring buffer (last 10,000 entries)
- On SW termination, logs are flushed to `chrome.storage.session` (if available) or `chrome.storage.local`
- On spike completion, logs are included in the spike report
- Log replay: the `replay.ts` module reads a spike's logs and reconstructs the execution timeline

### 6.4 Log Replay

The harness can replay a spike execution from logs:

```
[+0ms]    SW: SPIKE_LOAD spike-c1
[+12ms]   SW: CAPABILITY_CHECK supportsWASM=true
[+45ms]   SW: SANDBOX_CREATE sandbox.html
[+230ms]  SANDBOX: WASM_INIT c2pa-js v0.5.0
[+1800ms] SANDBOX: WASM_READY (took 1570ms)
[+1805ms] SW: FIXTURE_LOAD images/c2pa/firefly-valid.jpg
[+1850ms] SW: SANDBOX_COMPUTE_REQUEST { fixture: 'firefly-valid.jpg' }
[+1980ms] SANDBOX: C2PA_VERIFY_START
[+2150ms] SANDBOX: C2PA_VERIFY_RESULT { valid: true, signer: 'Adobe Inc.' }
[+2160ms] SW: BENCHMARK c2pa_verify_time = 180ms PASS
[+2170ms] SW: SPIKE_RESULT PASS
```

---

## Section 7: Reporting System

### 7.1 Report Structure

Every spike run produces a report in `reports/{spike_id}/`:

```
reports/spike-c1/
├── report.md              # Human-readable report
├── metrics.json           # All metric values (machine-readable)
├── profile.json           # Profile snapshot
├── logs.json              # All log entries
├── capabilities.json      # Capability profile at time of run
├── benchmarks.json        # Benchmark samples
├── scenarios.json         # Per-scenario pass/fail
└── screenshots/           # Screenshots if spike captured any
    └── error-state.png
```

### 7.2 Report Template (Markdown)

```markdown
# SPIKE REPORT: spike-c1 — C2PA WASM Validation

**Date:** 2026-07-28T14:32:00Z
**Browser:** Chrome 140.0.0.0
**Status:** ✅ PASS
**Duration:** 4,230ms (target: <5,000ms)

## Assumption Validated
> A-C1: c2pa-js WASM module loads, initializes, and verifies C2PA manifests within an MV3 sandbox page context

## Scenario Results

| Scenario | Status | Duration | Notes |
|----------|--------|----------|-------|
| verify-valid-c2pa | ✅ PASS | 180ms | Signature verified. Signer: Adobe Inc. |
| verify-tampered-c2pa | ✅ PASS | 165ms | Correctly rejected with INVALID_SIGNATURE |
| verify-no-c2pa | ✅ PASS | 95ms | Correctly returned NO_MANIFEST |

## Benchmarks

| Benchmark | Target | Actual | Status |
|-----------|--------|--------|--------|
| c2pa_init_time | <2,000ms | 1,570ms | ✅ |
| c2pa_verify_time | <200ms | 180ms | ✅ |
| message_latency_p50 | <50ms | 12ms | ✅ |

## Profile

| Metric | Value |
|--------|-------|
| Peak memory | 48 MB |
| SW terminations | 0 |
| Context switches | 14 |
| Messages sent | 22 |

## Browser Capabilities
| Capability | Detected |
|------------|----------|
| supportsWASM | ✅ |
| supportsSandbox | ✅ |
| supportsMessagePort | ✅ |
| supportsWebGPU | ✅ |
| supportsOffscreen | ✅ |

## Architecture Notes
- c2pa-js loaded successfully from bundled assets in sandbox page.
- MessagePort communication was stable — no dropped messages.
- WASM module size: 2.8 MB. Acceptable.
- Warm cache load time: 320ms (significantly faster than first load).

## Recommendations
- Proceed to SPIKE-C2 (ONNX Runtime Web). Sandbox WASM foundation is validated.
- Consider pre-warming sandbox page on extension install to reduce first-analysis latency.
```

### 7.3 Cross-Spike Comparison

The harness can generate a summary report comparing metrics across all executed spikes:

```markdown
# CROSS-SPIKE SUMMARY

| Spike | Status | Duration | Peak Memory | SW Terms | Messages |
|-------|--------|----------|-------------|----------|----------|
| spike-c0 | ✅ | 8,230ms | 32 MB | 0 | 45 |
| spike-c1 | ✅ | 4,230ms | 48 MB | 0 | 22 |
| spike-c2 | ✅ | 12,500ms | 210 MB | 0 | 38 |
| spike-c3 | ⚠️ WARN | 65,000ms | 28 MB | 2 | 120 |
| spike-c4 | ✅ | 3,100ms | 24 MB | 0 | 80 |
```

---

## Section 8: Fixture System

### 8.1 Fixture Catalog

All fixtures are registered in `fixtures/catalog.json`:

```jsonc
{
  "images/c2pa/firefly-valid.jpg": {
    "id": "images/c2pa/firefly-valid.jpg",
    "type": "image/jpeg",
    "category": "c2pa",
    "subcategory": "valid",
    "description": "Adobe Firefly output with valid C2PA manifest declaring AI generation",
    "dimensions": "1024x1024",
    "size_bytes": 245760,
    "has_c2pa": true,
    "c2pa_valid": true,
    "c2pa_signer": "Adobe Inc.",
    "c2pa_claims": "AI-generated using Adobe Firefly",
    "has_exif": true,
    "has_iptc": false,
    "ai_generated": true,
    "source": "Generated via Adobe Firefly, July 2026"
  }
}
```

### 8.2 FixtureLoader API

```typescript
interface FixtureLoader {
  /** Load a fixture by ID. Returns appropriate type for the fixture category. */
  load(id: string): Promise<Fixture>;

  /** Load a fixture into a specific browser context (content script, sandbox, etc.) */
  loadInto(id: string, context: BrowserContext): Promise<FixtureHandle>;

  /** Get fixture metadata without loading the data */
  metadata(id: string): FixtureMetadata;

  /** List fixtures matching a filter */
  list(filter: FixtureFilter): FixtureMetadata[];
}

type Fixture =
  | ImageFixture      // { type: 'image', data: ArrayBuffer, metadata }
  | HtmlFixture       // { type: 'html', data: string, metadata }
  | VideoFixture      // { type: 'video', data: ArrayBuffer, metadata }
  | AudioFixture      // { type: 'audio', data: ArrayBuffer, metadata }
  | TextFixture;      // { type: 'text', data: string, metadata }
```

### 8.3 Fixture Loading Strategy

- **Images:** Loaded as `ArrayBuffer` via `fetch()` from bundled extension assets. Transferred to sandbox via MessagePort (zero-copy for ArrayBuffers).
- **HTML pages:** Injected into content script's tab via `chrome.scripting.executeScript` or loaded as data URIs. For CORS testing, served from a local fixture server (`scripts/serve-fixtures.ts`).
- **Video/Audio:** Loaded as `ArrayBuffer`. For tabCapture testing, played in a browser tab.
- **Large fixtures (>10MB):** Streamed from IndexedDB (stored once, reused across spikes).

---

## Section 9: Browser Capability Detection

### 9.1 CapabilityDetector

The detector runs at harness startup (service worker initialization). Results are cached for the session.

```typescript
interface CapabilityDetector {
  /** Run all probes. Returns complete profile. */
  detect(): Promise<CapabilityProfile>;

  /** Check a single capability. */
  check(capability: CapabilityId): Promise<boolean>;

  /** Get the cached profile (must call detect() first). */
  getProfile(): CapabilityProfile;
}

interface CapabilityProfile {
  browser: string;           // 'chrome' | 'firefox' | 'safari' | 'edge'
  browserVersion: string;    // '140.0.0.0'
  userAgent: string;
  platform: string;          // 'win32' | 'macos' | 'linux' | 'android' | 'ios'
  timestamp: string;

  // Compute
  supportsWASM: boolean;
  supportsWASMThreading: boolean;
  supportsSharedArrayBuffer: boolean;
  supportsWebGPU: boolean;

  // ML
  supportsONNX: boolean;           // WASM backend always available if WASM supported
  supportsONNXWebGPU: boolean;     // WebGPU backend
  supportsChromeAI: boolean;       // Chrome built-in AI APIs (Gemini Nano)

  // Browser APIs
  supportsTabCapture: boolean;
  supportsOffscreen: boolean;
  supportsSidePanel: boolean;
  supportsSandbox: boolean;
  supportsMessagePort: boolean;

  // Storage
  supportsSessionStorage: boolean;
  supportsLocalStorage: boolean;   // Always true in practice
  supportsIndexedDB: boolean;
  supportsUnlimitedStorage: boolean;

  // Quotas
  storageLocalQuota: number;       // bytes
  storageSessionQuota: number;     // bytes
  indexedDBQuota: number;          // bytes (estimated via navigator.storage.estimate)

  // Extras
  serviceWorkerLifetime: '30s_idle' | 'event_page' | 'background_script';
  alarmsIntervalMinimum: number;   // seconds
  maxOffscreenDocuments: number;   // 1 or 0
}
```

### 9.2 Probe Implementation Pattern

Each probe is a standalone function that returns `true`, `false`, or throws (caught and recorded as `false`):

```typescript
async function probeWebGPU(): Promise<boolean> {
  if (typeof navigator === 'undefined' || !('gpu' in navigator)) return false;
  try {
    const adapter = await navigator.gpu.requestAdapter();
    return adapter !== null;
  } catch {
    return false;
  }
}

async function probeTabCapture(): Promise<boolean> {
  return typeof chrome !== 'undefined' && 'tabCapture' in chrome;
}
```

### 9.3 Capability Auto-Annotation

Every spike report is automatically annotated with the capability profile at the time of execution. The harness uses capabilities to:
- Skip spikes that require unavailable capabilities
- Adjust benchmark targets (e.g., longer timeout for WASM-only vs WebGPU)
- Document capability gaps in reports (e.g., "tabCapture not available on Firefox — video analysis skipped")

---

## Section 10: SPIKE-C0 — End-to-End Architecture Validation

### 10.1 Purpose

SPIKE-C0 validates that the complete browser architecture works end-to-end before ANY specialized spike runs. It exercises every browser context in the architecture: content script → service worker → sandbox → offscreen document → popup → content script return path. It validates message passing, lifecycle management, serialization, context switching, cancellation, and timeout handling.

**SPIKE-C0 contains NO C2PA, NO ONNX, NO AI, NO forensics.** It tests architecture plumbing only.

### 10.2 Spike-C0 Manifest

```jsonc
{
  "spike_id": "spike-c0",
  "name": "End-to-End Architecture Validation",
  "objective": "Validate message passing, lifecycle, serialization, context switching, recovery, cancellation, and timeout handling across all browser contexts without any specialized analysis logic.",
  "assumption_validated": null,    // Validates architecture, not a specific assumption
  "priority": "critical",
  "prerequisite_for": "ALL OTHER SPIKES",  // Must pass before any other spike executes
  "browsers": { "chrome": "required", "firefox": "required", "edge": "optional", "safari": "optional" },
  "permissions": {
    "manifest_permissions": ["activeTab", "storage", "alarms", "offscreen"],
    "host_permissions": ["<all_urls>"]
  },
  "required_contexts": ["service_worker", "content_script", "sandbox_page", "offscreen_document", "popup"],
  "required_capabilities": ["supportsWASM", "supportsSandbox", "supportsMessagePort"],
  "expected_runtime": { "minimum_ms": 5000, "maximum_ms": 60000, "target_ms": 15000 },
  "scenarios": [],
  "benchmarks": [],
  "fixtures": ["html/blank.html"],
  "depends_on_spikes": [],
  "skip_if": {}
}
```

### 10.3 Spike-C0 Scenarios

| # | Scenario | What It Validates |
|---|----------|-------------------|
| **S0-1** | Content Script → SW message round-trip | Content script sends a payload. SW receives, modifies, returns. CS verifies modification. |
| **S0-2** | SW → Sandbox compute round-trip | SW sends compute request to sandbox. Sandbox performs trivial WASM addition. Returns result. SW verifies. |
| **S0-3** | SW → Offscreen storage round-trip | SW sends storage request to offscreen doc. Offscreen writes to IndexedDB, reads back, returns value. SW verifies. |
| **S0-4** | SW → Popup progress updates | SW sends 5 sequential progress updates to popup. Popup verifies all 5 received in order. |
| **S0-5** | Multi-context pipeline | Full round-trip: CS → SW → Sandbox → SW → Offscreen → SW → Popup. Each hop adds a token to a payload. Final payload verified. |
| **S0-6** | Large payload serialization | CS sends a 50KB payload (simulated DOM text) through all contexts. Each context verifies hash. Final hash matches original. |
| **S0-7** | Context recovery after SW termination | Intentionally idle SW for 35s. Verify SW restarts. Send message. Verify message delivered. |
| **S0-8** | Cancellation propagation | Start a long-running sandbox computation (simulated sleep). Cancel via AbortController. Verify sandbox stops and SW receives cancellation confirmation. |
| **S0-9** | Timeout handling | Send request to sandbox that intentionally never responds. Verify SW timeout fires and produces structured error. |
| **S0-10** | Concurrent spike isolation | Run two spike message handlers simultaneously. Verify no cross-spike interference. |

### 10.4 Spike-C0 Success/Failure Criteria

**PASS:** All 10 scenarios succeed on Chrome and Firefox.
**PASS WITH WARNINGS:** All 10 succeed on Chrome. Some fail on Firefox with documented gaps (e.g., offscreen doc N/A — S0-3 skipped with annotation).
**FAIL:** Any scenario fails on Chrome. **Architecture redesign required. No other spike may execute.**

### 10.5 Spike-C0 as Prerequisite

The harness enforces: if `spike-c0` result is not PASS, all other spike runs are rejected with the message: "Prerequisite SPIKE-C0 has not passed. Run SPIKE-C0 first."

---

## Section 11: Architecture Readiness Review (ARR)

### 11.1 Purpose

The ARR is the formal gate between validation and implementation. It occurs after ALL mandatory spikes (Critical + High) have been executed. It answers: is the architecture proven sound, or must it be revised?

### 11.2 ARR Document Template

```markdown
# Architecture Readiness Review — Sentinel Trust Engine

**Date:** YYYY-MM-DD
**Review Board:** Architect, Product Manager, Security Architect, ML Researcher, Release Engineer
**Spike Executions:** 22 of 32 (7 Critical + 7 High + 8 Medium)
**ARR Status:** GO / NO-GO / CONDITIONAL GO

---

## 1. Mandatory Spike Results

| Spike | Assumption | Status | Browser | Notes |
|-------|-----------|--------|---------|-------|
| SPIKE-C0 | Architecture E2E | ✅ PASS | Chrome, Firefox | — |
| SPIKE-C1 | C2PA WASM (A-C1) | ✅ PASS | Chrome, Firefox | — |
| SPIKE-C2 | ONNX Runtime (A-C2) | ✅ PASS | Chrome, Firefox | — |
| SPIKE-C3 | SW Keep-Alive (A-C3) | ✅ PASS | Chrome | ⚠️ WARN on Firefox (see below) |
| SPIKE-C4 | Messaging (A-C4) | ✅ PASS | Chrome, Firefox | — |
| SPIKE-C5 | MessagePort (A-C5) | ✅ PASS | Chrome | — |
| SPIKE-C6 | IndexedDB Proxy (A-C6) | ✅ PASS | Chrome | N/A on Firefox (expected) |
| SPIKE-C7 | Cross-Origin Fetch (A-C7) | ⚠️ PASS | Chrome | 62% of tested URLs accessible |

## 2. Critical Assumptions — All Validated?
✅ All 7 critical assumptions passed. Architecture foundation is sound.

## 3. Disproven Assumptions
(None — or list any assumptions that failed and require redesign)

## 4. Architecture Modifications Required
- A-C7: Cross-origin fetch success rate is 62%, below the 80% aspirational target. Metadata extraction will work on the majority of the web but not universally. This was anticipated in the architecture as "metadata best-effort." No redesign required. Document limitation in user-facing materials.

- A-C3: SW keep-alive produces WARNING on Firefox (SW terminated at 42s in 2 of 20 trials). Firefox Event Pages do not provide the expected persistence advantage. Accept Chrome keep-alive strategy for Firefox as well. No architecture change.

## 5. ADR Status Updates

| ADR | Previous | New | Rationale |
|-----|----------|-----|-----------|
| ADR-004 | Proposed | **Accepted** | Bayesian fusion validated by end-to-end messaging (SPIKE-C0) and explainability edge cases (SPIKE-H7) |
| ADR-005 | Proposed | **Accepted** | 8-dimension model works with complete evidence. Memory-constrained degradation validated by SPIKE-H1 |
| ADR-006 | Proposed | **Accepted** | Tiered scheduling validated. SW keep-alive sustains analysis chains. Capability-aware scheduling works. |
| ADR-009 | Proposed | **Accepted** | Cloud fallback decisions clarified by cross-origin fetch results (SPIKE-C7) |
| ADR-010 | Proposed | **Accepted** | Model loading from IndexedDB via offscreen proxy validated (SPIKE-C6, SPIKE-H1, SPIKE-H4) |

All 16 ADRs now Accepted.

## 6. Remaining Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Cross-origin fetch unreliable for 38% of URLs | Medium | Accept as design constraint. Metadata analysis is best-effort on cross-origin images. | 
| Firefox offscreen doc unavailable | Medium | Already designed for — Firefox uses sandbox page for compute, skips offscreen-dependent analyzers. |
| Safari comprehensive testing delayed | Low | Safari spikes (M2) show basic functionality works. Detailed Safari testing deferred to Phase 2 implementation. |
| Large model (150MB) peak memory near limit | Low | Spike H1 shows peak at 280MB. Acceptable. Monitor in production. |

## 7. Implementation Authorization

☑ **GO — Implementation is authorized.**

All mandatory exit criteria (EC-1 through EC-7) met.
All conditional exit criteria (EC-8 through EC-10) met.
All advisory exit criteria (EC-11 through EC-15) met.

## 8. Technical Debt Incurred During Validation
- Harness logging ring buffer implementation is naive (linear scan). Acceptable for validation, must be reimplemented for production.
- Fixture server is a simple Node script. Not needed for production.
- Spike implementations contain hardcoded test values. Not reusable for production.

## 9. Signatures
- Architect: _______________
- Product Manager: _______________
- Security Architect: _______________
- Release Engineer: _______________
```

### 11.3 ARR Go/No-Go Rules

| Verdict | Condition |
|---------|-----------|
| **GO** | All mandatory exit criteria passed. No critical assumptions disproven. All Priority 1+2 ADRs Accepted. |
| **CONDITIONAL GO** | All mandatory exit criteria passed. One or more High assumptions failed BUT architecture absorbed the failure with documented modification. ADR amendments required. Implementation may begin on unaffected subsystems. |
| **NO-GO** | Any mandatory exit criterion failed. OR any critical assumption disproven requiring architecture redesign. Implementation blocked until revised architecture passes re-validation. |

---

## Appendix A: Spike Authoring Guide (Reference)

### How to Create a New Spike

1. Create directory: `spikes/spike-{id}/`
2. Create `manifest.json` using the schema from Section 3.1
3. Create `spike.ts` implementing the `SpikeImplementation` interface
4. Add scenarios in `scenarios/` directory
5. Create `README.md` with objective, how to run, expected output
6. Register required fixtures in `manifest.json` (if using existing fixtures) or add new fixtures to `fixtures/`
7. Build: `npx tsx scripts/build.ts --spike spike-{id}`
8. Load extension in Chrome: `chrome://extensions` → "Load unpacked" → select `sentinel-validation/` root
9. Open popup → select spike from dropdown → click "Run"
10. View report: `reports/spike-{id}/report.md`

### Spike Implementation Template

```typescript
import { SpikeImplementation, SpikeContext, SpikeResult } from '../harness/core/registry';

const spike: SpikeImplementation = {
  async initialize(ctx: SpikeContext): Promise<void> {
    ctx.logging.info('Spike initializing');

    // Register message handlers
    ctx.messaging.on('sandbox:compute_result', (payload) => {
      ctx.benchmark.lap('sandbox_roundtrip', 'compute_result_received');
      ctx.profiling.count('sandbox_results');
      collectedResults.push(payload);
    });
  },

  async run(ctx: SpikeContext): Promise<SpikeResult> {
    ctx.logging.info('Spike running');
    ctx.onProgress('Starting scenario 1...');

    try {
      // Scenario 1: Verify valid C2PA
      ctx.benchmark.start('c2pa_verify_time');
      ctx.onProgress('Verifying valid C2PA manifest...');

      const fixture = await ctx.fixtures.load('images/c2pa/firefly-valid.jpg');
      ctx.messaging.send('sandbox', 'compute_request', {
        analyzer: 'c2pa',
        imageHash: await ctx.utils.hash(fixture.data)
      });

      // ... await result, assert, benchmark ...

      ctx.benchmark.stop('c2pa_verify_time');
      ctx.onProgress('Scenario 1 complete.');

      return {
        spikeId: ctx.spikeId,
        status: 'PASS',
        scenarios: [/* ... */],
        benchmarks: ctx.benchmark.samples(),
        profile: ctx.profiling.getSnapshot(),
        logs: ctx.logging.getLogs(),
        duration: 0 // computed by orchestrator
      };
    } catch (err) {
      ctx.logging.error('Spike failed', err as Error);
      return {
        spikeId: ctx.spikeId,
        status: 'FAIL',
        scenarios: [],
        benchmarks: ctx.benchmark.samples(),
        profile: ctx.profiling.getSnapshot(),
        logs: ctx.logging.getLogs(),
        error: (err as Error).message,
        duration: 0
      };
    }
  },

  async cleanup(ctx: SpikeContext): Promise<void> {
    ctx.logging.info('Spike cleanup');
    ctx.messaging.removeAllListeners();
    ctx.benchmark.reset();
  }
};

export default spike;
```

---

## Appendix B: mv3 manifest.json (Generated)

The harness build script generates the single `manifest.json` by merging spike requirements:

```jsonc
{
  "manifest_version": 3,
  "name": "Sentinel Validation Harness",
  "version": "0.1.0",
  "description": "Engineering validation platform for Sentinel Trust Engine technical spikes.",

  "background": {
    "service_worker": "harness/contexts/service-worker.js",
    "type": "module"
  },

  "permissions": [
    "activeTab",
    "storage",
    "alarms",
    "offscreen",      // From SPIKE-C6, H4
    "scripting"       // For content script injection
  ],

  "host_permissions": ["<all_urls>"],

  "content_scripts": [{
    "matches": ["<all_urls>"],
    "js": ["harness/contexts/content-script.js"],
    "run_at": "document_idle"
  }],

  "sandbox": {
    "pages": ["harness/contexts/sandbox/sandbox.html"]
  },

  "action": {
    "default_popup": "harness/contexts/popup/popup.html",
    "default_title": "Sentinel Validation"
  },

  "side_panel": {
    "default_path": "harness/contexts/sidepanel/sidepanel.html"
  },

  "content_security_policy": {
    "extension_pages": "script-src 'self'; object-src 'self'",
    "sandbox": "sandbox allow-scripts allow-modals; script-src 'self' 'wasm-unsafe-eval'"
  },

  "web_accessible_resources": [{
    "resources": [
      "fixtures/**",
      "spikes/**"
    ],
    "matches": ["<all_urls>"]
  }]
}
```

---

*End of Validation Harness Architecture. This document defines the engineering validation platform. All 32 spikes execute within this harness. No spike implements its own extension, service worker, content script, sandbox page, or messaging infrastructure.*
