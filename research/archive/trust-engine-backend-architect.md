# TRUST_ENGINE_SPEC.md

> **Project:** Sentinel | **Phase:** 1 — Architecture
> **Document:** Trust Engine & Evidence Source Ecosystem Specification
> **Domain:** Evidence Source Interface Design
> **Status:** Draft
> **Owner:** Backend Architect

---

## Purpose

This document defines the **contract between the Trust Engine and every evidence source it consumes.** The Trust Engine does not generate evidence itself. It aggregates, weights, and reasons over evidence produced by independent sources. This separation is foundational: evidence sources are pluggable subsystems that the Trust Engine orchestrates but does not own.

The question this document answers:

> **How does the Trust Engine discover, invoke, receive results from, and reason about evidence sources — without coupling its core logic to any specific source implementation?**

---

## 1. Evidence Source Taxonomy

Every evidence source that the Trust Engine might consume falls into exactly one of these five categories. The category determines the source's execution constraints, trust model, and orchestration rules.

### 1.1 Local Detectors

**Definition:** Evidence sources that execute entirely on the user's device. No network access. No data leaves the device.

| Detector Type | What It Analyzes | Signal Produced |
|---|---|---|
| Image forensics | Pixel-level artifacts (noise patterns, frequency anomalies, lighting inconsistencies) | Forensic signal — artifact presence, manipulation indicators |
| Text statistical analysis | Perplexity, burstiness, token distribution patterns, stylometric features | Forensic signal — statistical anomaly score |
| Metadata extraction | EXIF, IPTC, XMP, C2PA manifest location, file structure | Provenance signal — metadata presence, consistency, anomalies |
| C2PA verification | Cryptographic manifest validation, signer identity, ingredient chain integrity | Provenance signal — signature validity, signer trust, manifest chain |
| Audio analysis | Vocoder artifacts, frequency spectrum anomalies, temporal inconsistencies | Forensic signal — synthesis artifact score |
| Face-swap detection | Facial boundary artifacts, blending inconsistencies, physiological signal violations | Forensic signal — manipulation probability |

**Execution constraints:**
- Must run in browser sandboxed context (WASM, ONNX Runtime Web, or equivalent)
- Must complete within 5 seconds for interactive UX; 30 seconds maximum for background analysis
- Model weights bundled with extension or downloaded once on install
- No runtime network requests

**Trust model:** Local detectors are the most trusted sources because they are verifiable — the user's device executes them, the model weights are signed, and no third party intermediates the analysis.

### 1.2 Local Reference Data

**Definition:** Datasets downloaded periodically and queried locally. The data is fetched from a network source but analysis/querying happens on-device.

| Data Type | What It Contains | Refresh Cadence |
|---|---|---|
| Domain reputation lists | Known disinformation domains, impersonation domains, reliable news sources | Daily |
| Known AI content hashes | Perceptual hashes of confirmed AI-generated content (images, video frames) | Weekly |
| Publisher credibility database | Verified publisher identities, C2PA signer reputations, trust rankings | Weekly |
| Certificate revocation lists | Revoked C2PA signing certificates, compromised CA certificates | Hourly |
| AI model fingerprint database | Known artifacts/signatures of specific AI generators (Stable Diffusion, DALL-E, Midjourney) | Weekly |
| Known watermark patterns | SynthID detection patterns (if publicly available), Steg.ai, IMATAG signatures | Monthly |

**Execution constraints:**
- Data downloaded as signed bundles, verified before use
- Queries execute locally with sub-millisecond latency
- Stale reference data degrades gracefully — the Trust Engine weighs signals from stale data lower
- Reference data bundles must not exceed 50 MB combined (extension store size limits)

**Trust model:** Reference data is trusted proportionally to recency and source integrity. Signed bundles from Sentinel's infrastructure are trusted more than community-contributed lists. Stale data is automatically discounted.

### 1.3 Cloud APIs

**Definition:** Evidence sources that require real-time network queries. Content or content-derived features MAY be sent to external services. These are the most capable but least private evidence sources. **All Cloud API sources must be opt-in with explicit user consent before first use.**

| API Type | What It Provides | Privacy Impact |
|---|---|---|
| Reverse image search | Finds prior occurrences of an image on the web; detects repurposing, identifies original source | Content sent to search provider |
| Threat intelligence | Real-time domain/IP/phishing reputation, coordinated inauthentic behavior detection, malware association | URLs/domains sent to intelligence provider |
| Community signal aggregation | Aggregated trust/dispute signals from other Sentinel users about domains, content, publishers | Anonymized signal data sent |
| Heavyweight ML inference | Deep models too large for local execution — video deepfake detection, multimodal analysis, adversarial detection | Content sent to inference endpoint |
| Cross-reference validation | Fact-checking database queries, claim verification against known datasets | Claim data sent to validator |

**Execution constraints:**
- Must complete within 10 seconds (with progressive results acceptable)
- Must declare latency SLA at registration time
- Must return partial/graceful-degradation results if timeout exceeded
- Network failures, timeouts, or HTTP 5xx must not block other evidence sources
- Rate limits, quota, and cost tracking must be declared by the source

**Privacy contract (mandatory for all Cloud API sources):**
- The source must declare exactly what data it transmits and to which endpoint
- The Trust Engine must present this declaration to the user before first invocation
- Users must be able to disable all Cloud API sources with a single control
- Users must be able to per-source toggle Cloud API sources
- No Cloud API source may be invoked silently or without prior consent

**Trust model:** Cloud API sources are the least trusted because the Trust Engine cannot verify their internal operation. They are external black boxes. Their signals are weighed lower than local detector signals unless their historical accuracy is independently verified. Cloud API sources from competing or unvetted providers are weighed lower than those from trusted partners.

### 1.4 User-Provided Signals

**Definition:** Explicit signals that originate from the user's own judgment. These are not evidence ABOUT content — they are user preferences and assertions that inform how the Trust Engine weighs other evidence.

| Signal Type | User Action | Effect |
|---|---|---|
| Source trust designation | "I trust content from example.com" | Elevates provenance and reputation signals for this domain |
| Source distrust designation | "Content from example.com is unreliable" | Depresses all signals for this domain |
| Content suspicion flag | "This specific content seems suspicious to me" | Flags content for heightened scrutiny; increases sensitivity |
| Manual override | "Ignore all warnings for this content" | Suppresses all evidence below a severity threshold for specific content |
| Personal allow/block list | Maintained list of domains, publishers, content hashes | Applied before evidence source invocation as a routing decision |
| Calibration feedback | "This assessment was wrong — this content is actually authentic/synthetic" | Fed back to source trust scoring and model improvement |

**Execution constraints:**
- User signals take precedence over ALL automated evidence when they conflict
- User signals must be explicitly reversible (no permanent "trust" or "distrust" without an undo path)
- User signals must be time-bound by default (e.g., "trust this domain for 30 days") with user-selectable permanence option
- Personal allow/block lists are stored locally, never transmitted

**Trust model:** User signals are treated as ground truth for THAT user. They do not affect other users' assessments. They override evidence source outputs when in direct conflict. They are never questioned or second-guessed by the Trust Engine.

### 1.5 Derived Signals

**Definition:** Signals computed internally by the Trust Engine from the outputs of other evidence sources. These are NOT evidence sources themselves — they are reasoning products that inform confidence calibration.

| Derived Signal | How It Is Computed | Purpose |
|---|---|---|
| Signal agreement score | Count of independent sources that agree on a finding, weighted by source independence | Detects consensus; high agreement across diverse sources increases confidence |
| Signal disagreement score | Count and magnitude of conflicting signals | Detects ambiguity; high disagreement triggers "uncertain" classification |
| Confidence trajectory | Change in confidence over sequential analysis (early signals vs. full analysis) | Indicates stability; rapidly shifting confidence suggests unreliable signals |
| Temporal consistency | Whether the same content analyzed at different times produces consistent results | Detects time-sensitive manipulation (e.g., domain that changes behavior post-analysis) |
| Source independence metric | Pairwise correlation between source outputs over historical analyses | Detects sources that are redundant (highly correlated) vs. truly independent |
| Coverage completeness | What percentage of applicable evidence sources returned results | Indicates analysis thoroughness; incomplete coverage depresses confidence |

**Execution constraints:**
- Computed entirely within the Trust Engine — no external dependencies
- Recalculated whenever new evidence is added or evidence is invalidated
- Not treated as evidence but as confidence calibration modifiers

---

## 2. Evidence Source Contract

Every evidence source that integrates with the Trust Engine must fulfill this contract. The contract defines the minimum input/output specification, error handling, and latency requirements. Any source that does not satisfy this contract cannot be used by the Trust Engine.

### 2.1 Input Specification

Every evidence source receives an **Evidence Request** containing:

| Field | Required | Description |
|---|---|---|
| `content_reference` | Yes | A handle to the content to be analyzed. Not the raw content itself — a reference the source uses to access content (e.g., a blob URL, a content hash, a DOM element reference). The Trust Engine provides content access; the source does not fetch it independently. |
| `content_type_hint` | Yes | MIME type or modality hint (`image/jpeg`, `text/plain`, `video/mp4`, `audio/wav`, `application/pdf`). Used for routing to appropriate sources. |
| `analysis_context` | No | Optional metadata about the analysis context: URL where content was found, surrounding page content type (news article, social media post, direct upload), user's trust preferences for this domain. |
| `previous_evidence` | No | Results from evidence sources that have ALREADY executed. Enables sources to focus on what hasn't been covered. Sources MUST NOT modify their behavior based on what other sources found (independence requirement). |
| `priority` | Yes | Execution tier: `REQUIRED` (always run), `STANDARD` (run unless user disabled), `SUPPLEMENTAL` (run only if needed). Set by the Trust Engine based on orchestration rules. |

**Anti-patterns (what evidence sources must NOT receive):**
- Raw user browsing history beyond the specific page being analyzed
- User identity, account information, or authentication tokens
- Results from other evidence sources that would bias their analysis (exempt: derived signals about which gaps remain)

### 2.2 Output Specification

Every evidence source must return an **Evidence Result** containing:

| Field | Required | Type | Description |
|---|---|---|---|
| `signal_value` | Yes | Enumerated or continuous | What was detected. For categorical sources: `AUTHENTIC`, `SYNTHETIC`, `MANIPULATED`, `INCONCLUSIVE`, `ERROR`. For continuous sources: a score in a well-defined range (e.g., 0.0 to 1.0) with documented interpretation. |
| `confidence` | Yes | Float 0.0–1.0 | The source's self-assessed certainty in its own signal. 1.0 = source is absolutely certain. 0.0 = source has no useful information. This is the source's own calibration, NOT the Trust Engine's confidence. |
| `evidence_type` | Yes | Enum | Classification of the evidence: `PROVENANCE`, `FORENSIC`, `REPUTATION`, `COMMUNITY`, `DERIVED` |
| `timestamp` | Yes | ISO 8601 | When the evidence was gathered. Used for freshness weighting and temporal consistency analysis. |
| `source_identifier` | Yes | String | Unique identifier of the detector/model that produced this result. Enables source reputation tracking and version-specific weighting. |
| `source_version` | Yes | String | Version of the detector/model. Enables the Trust Engine to track accuracy changes across versions and phase out old versions. |
| `explanation` | Yes | Human-readable string | A concise description of what was found, suitable for display to the user. Example: "No noise artifacts consistent with AI generation detected in this image." Maximum 280 characters. |
| `evidence_detail` | No | Structured data | Source-specific detail that the Trust Engine can surface for advanced explainability. Format is source-defined. Example: C2PA manifest chain, specific EXIF anomalies, model attention heatmap summary. |
| `latency_ms` | No | Integer | Actual time the source took to execute. Used for performance monitoring and UX (progressive result loading). |
| `error` | No | Error object | Present only if `signal_value` is `ERROR`. Contains error classification and a message. |

**Confidence calibration requirement:**
- A source reporting `confidence: 0.95` must be correct approximately 95% of the time across its outputs
- The Trust Engine will empirically measure each source's calibration and apply a calibration correction factor
- Sources with systematically miscalibrated confidence (e.g., always reporting 0.99) will have their confidence automatically down-weighted

### 2.3 Error Contract

Every evidence source must handle these failure modes explicitly:

| Failure Mode | Required Behavior |
|---|---|
| **Source execution failure** (crash, exception) | Return `signal_value: ERROR` with `error.classification: INTERNAL_ERROR`. Must not throw or propagate exceptions to the Trust Engine. |
| **Timeout exceeded** | Return `signal_value: ERROR` with `error.classification: TIMEOUT`. Must not leave the Trust Engine waiting indefinitely. Sources must self-enforce their declared timeout. |
| **Ambiguous/unusable result** | Return `signal_value: INCONCLUSIVE` with `confidence < 0.5` and an explanation of why no conclusion could be reached. This is NOT an error — it's a valid signal that the source has no information. |
| **Source unavailable** (network down, model not loaded, reference data corrupted) | Return `signal_value: ERROR` with `error.classification: UNAVAILABLE`. The Trust Engine may retry based on the source's retry policy or immediately mark the source as degraded. |
| **Content not processable by this source** (wrong format, unsupported encoding, content too large) | Return `signal_value: ERROR` with `error.classification: UNSUPPORTED_CONTENT`. The Trust Engine should not route this content type to this source in the future. |
| **Partial result** (source completed partial analysis before failure/timeout) | Return `signal_value` with best-available partial result, `confidence` reduced proportionally to what couldn't be analyzed, and `error` object with `classification: PARTIAL_RESULT`. |

**Error escalation by the Trust Engine:**
- A source that returns `ERROR` on >50% of invocations in a rolling window is marked DEGRADED
- A source marked DEGRADED is skipped unless no other source of its evidence type is available
- A source that returns `ERROR` on >90% of invocations is marked OFFLINE and removed from active orchestration
- A source marked OFFLINE for >7 days is recommended for removal to the user (or automatically removed for non-user-installed sources)

### 2.4 Latency Contract

Every evidence source must declare its expected latency at registration:

| Latency Class | Maximum Response Time | Source Examples |
|---|---|---|
| `IMMEDIATE` | < 100 ms | Metadata extraction, hash lookup, reference data query, allow/block list check |
| `FAST` | < 2 s | C2PA verification, text statistical analysis, lightweight image forensics |
| `STANDARD` | < 10 s | Face-swap detection, audio analysis, reverse image search, community signal query |
| `SLOW` | < 30 s | Heavyweight ML inference, video analysis, cross-reference validation |
| `BACKGROUND` | < 5 min | Deep forensic analysis, multi-frame video processing, threat intelligence deep scan |

**Latency enforcement:**
- Sources must self-terminate at their declared latency ceiling
- The Trust Engine runs a parallel timeout that returns `ERROR (TIMEOUT)` if the source fails to self-terminate
- A source that exceeds its declared latency in >20% of invocations has its latency class downgraded
- The orchestrator respects latency class when deciding execution order

---

## 3. Evidence Source Registry & Discovery

The Trust Engine must know which evidence sources exist, what they can do, and how to invoke them — without hardcoding references to specific implementations.

### 3.1 Registry Model

The Evidence Source Registry is a capability-based catalog. Each entry declares:

| Declaration | Description |
|---|---|
| **Source identity** | Unique identifier, human-readable name, provider/author, version |
| **Capabilities** | Content types this source can analyze, evidence types it produces, modalities supported |
| **Execution profile** | Latency class, whether it requires network access, whether it sends content to external services, estimated resource consumption |
| **Privacy declaration** | For cloud sources: what data is transmitted, to which endpoint, under what privacy policy |
| **Dependencies** | Other evidence sources that must complete before this one can run (e.g., metadata extraction must run before C2PA verification) |
| **Input constraints** | Minimum content size, maximum content size, required format, required preprocessing |
| **Trust baseline** | Initial trust weight for this source (calibrated by the Trust Engine over time), whether the source's results should be treated as more or less reliable for specific content types |
| **Retirement status** | `ACTIVE`, `DEPRECATED` (still usable but replaced by successor), `RETIRED` (no longer invoked), `EXPERIMENTAL` (available but results not used in final confidence) |

### 3.2 Source Discovery Mechanisms

The Trust Engine must support four discovery mechanisms:

| Mechanism | Description | Use Case |
|---|---|---|
| **Static configuration** | Built-in sources that are always available. Defined at build time. Cannot be removed by users, only disabled. | Core detectors (C2PA verification, metadata extraction, basic text analysis). Essential for baseline functionality. |
| **Dynamic registration** | Sources registered at runtime via a well-defined interface. | User-installed detectors, community plugins, enterprise custom models. Must be sandboxed and verified before use. |
| **Capability advertisement** | Sources declare their capabilities via a manifest that the Trust Engine reads at registration time. | Enables automatic routing — Trust Engine discovers a new image forensics detector and automatically routes images to it. |
| **Remote registry (optional, Phase 3+)** | A vetted catalog of community and third-party evidence sources that users can browse and install. | Ecosystem growth. Analogous to browser extension stores. Vetting ensures sources meet security, privacy, and accuracy standards. |

### 3.3 Source Identity & Versioning

Each evidence source identity is a compound key: `{source_id}@{version}`.

**Versioning rules:**
- When a source is updated, its `source_version` field changes
- The Trust Engine tracks accuracy metrics PER VERSION, not just per source
- If a new version underperforms the previous version, the Trust Engine MAY prefer the older version's results (if both are available) or flag the new version as DEGRADED
- A source update that changes the signal interpretation (e.g., a different scoring range or different evidence_type) is considered a NEW source, not a new version — it must register under a new `source_id`

### 3.4 Deprecation & Retirement

| State | Behavior |
|---|---|
| **ACTIVE** | Normal operation. Source is invoked per orchestration rules. |
| **DEPRECATED** | Source still runs but its results carry a deprecation penalty (confidence multiplier < 1.0). User is notified that the source is outdated. A successor source is recommended. |
| **RETIRED** | Source is never invoked. If a user has this source in allow lists or custom configurations, it must be explicitly migrated or removed. Retired sources remain in the registry for historical audit (past analyses reference them) but are excluded from all active orchestration. |
| **EXPERIMENTAL** | Source runs but its results are collected for evaluation only — they do NOT contribute to the final confidence assessment. Enables A/B testing of new sources against production sources. |

---

## 4. Evidence Source Orchestration

The Trust Engine must decide which evidence sources to invoke, in what order, for each piece of content — without hardcoding decision logic per source.

### 4.1 Content-Type Routing

The Trust Engine maintains a routing table mapping content types to applicable evidence sources. This table is built dynamically from the registry:

| Content Type | Applicable Sources |
|---|---|
| `image/*` | Image forensics, metadata extraction, C2PA verification, face-swap detection (if faces detected), reverse image search, AI model fingerprint lookup, known hash lookup, watermark detection |
| `text/*` | Text statistical analysis, C2PA crJSON verification, domain reputation lookup (if from a URL), publisher credibility query, community signal query |
| `video/*` | Video forensics (frame sampling), audio track analysis, C2PA verification, face-swap detection per frame, reverse image search (keyframes) |
| `audio/*` | Audio forensics, C2PA verification, watermark detection |
| `application/pdf` | Text analysis (extracted text), metadata extraction, image analysis (embedded images), C2PA verification |

**Dynamic routing rule:** A source that declares capability for a content type is automatically added to the routing table for that type. Removing or retiring a source automatically removes it from all routing tables. No code change required in the orchestrator.

### 4.2 Tier-Based Invocation

The orchestrator structures execution in tiers. Sources in earlier tiers are faster, cheaper, and more privacy-preserving. Sources in later tiers require network access, send data to external services, or consume significant resources.

| Tier | Execution Rule | Sources |
|---|---|---|
| **Tier 0: Bypass** | Check user's personal allow/block list. If content is explicitly allowed, skip ALL evidence sources and mark as user-trusted. If explicitly blocked, skip and mark as user-distrusted. | User-provided signals |
| **Tier 1: Immediate Local** | Execute synchronously before any other analysis. Results available for user display within 100ms. | Metadata extraction, hash lookups, reference data queries |
| **Tier 2: Fast Local** | Execute in parallel as soon as Tier 1 completes. Results within 2s. | C2PA verification, text statistical analysis, lightweight image forensics |
| **Tier 3: Standard Local** | Execute in parallel as soon as Tier 2's fast sources complete. Results within 10s. | Face-swap detection, audio analysis, deeper image forensics |
| **Tier 4: Cloud (opt-in)** | Execute ONLY if user has enabled cloud-assisted analysis. Executed sequentially to respect rate limits and costs. | Reverse image search, heavyweight ML, threat intelligence, community signals |
| **Tier 5: Deep Analysis (opt-in)** | Execute ONLY if user has explicitly requested deep analysis for a specific piece of content. | Full video frame-by-frame analysis, cross-reference validation, adversarial detection |

**User presentation:** Results are displayed progressively as each tier completes. The user sees an initial assessment within 100ms, a refined assessment within 2s, and a comprehensive assessment within 10s. Users are never shown a loading spinner with no information — partial results are always displayed.

### 4.3 Parallel vs. Sequential Execution

| Execution Rule | Applies To |
|---|---|
| **Parallel** | All sources within the same tier that have no dependencies on each other. All Tier 2 local sources run concurrently. |
| **Sequential** | Sources that declare a dependency on another source's output (e.g., C2PA manifest parsing depends on metadata extraction completing first). |
| **Parallel with shared dependency** | Sources that all depend on the same predecessor but not on each other. They all wait for the predecessor, then run concurrently. |
| **Sequential within external services** | Cloud API sources that share rate limits, cost budgets, or the same external service are serialized to avoid overwhelming the service. |

**Hard constraint:** A source must NEVER be blocked indefinitely by another source. If a dependency source returns `ERROR` or `TIMEOUT`, dependent sources run with incomplete input and must handle this gracefully.

### 4.4 Confidence-Based Early Termination

The orchestrator MAY skip later-tier sources if earlier tiers produce sufficiently high confidence:

| Condition | Action |
|---|---|
| Provenance sources (Tier 2: C2PA) return `AUTHENTIC` with confidence ≥ 0.95 AND the signer is in the trusted publisher list | Skip Tier 3 forensic sources for this content. Flag as "provenance-verified." |
| Provenance sources return `SYNTHETIC` with confidence ≥ 0.95 (e.g., C2PA manifest explicitly declares AI generation) | Skip Tier 3 forensic sources for this content. Flag as "declared AI-generated." |
| Three or more independent forensic sources agree on the same finding with confidence ≥ 0.80 each | Skip remaining forensic sources. The ensemble has converged. |
| Tier 2 sources produce highly conflicting results (one says `AUTHENTIC` at 0.90, another says `SYNTHETIC` at 0.90) | Do NOT early-terminate. Escalate to Tier 3. High conflict requires MORE evidence, not less. |

**Anti-pattern:** Never early-terminate when confidence is high but signal agreement is low. High confidence with high disagreement means the system is certain it's uncertain — this must be communicated to the user, not resolved by skipping evidence.

### 4.5 User-Configured Source Selection

Users have these controls over evidence source invocation:

| Control | Scope | Default |
|---|---|---|
| **Disable specific source** | The named source never runs for any content | All sources enabled (except Cloud API sources, which require opt-in) |
| **Disable evidence type** | All sources of a specific evidence_type (e.g., "disable all reputation-based sources") | All types enabled |
| **Disable cloud analysis entirely** | All Tier 4+ sources are skipped; only local sources run | Cloud sources OFF by default (require opt-in) |
| **Per-domain source profile** | Different source configurations for different websites (e.g., stricter analysis on news sites, relaxed on personal blogs) | Global defaults apply |
| **Minimum confidence threshold** | Only invoke Tier 3+ sources if Tier 2 confidence is below this threshold | 0.85 |

---

## 5. Evidence Source Evolution

The evidence source landscape will change. Detectors will improve, become obsolete, and be replaced by new approaches. The Trust Engine must survive this without core logic changes.

### 5.1 Adding a New Evidence Source

When a new evidence source is added to the registry:

1. **Automatic capability discovery** — The Trust Engine reads the source's capability declaration and adds it to all applicable routing tables.
2. **Initial trust bootstrap** — The new source starts with a default trust weight. This weight is conservative (low) until the source has a proven track record.
3. **Shadow evaluation period** — For a configurable period (default: first 1,000 analyses), the new source's results are collected and compared against established sources' results but do NOT contribute to the final confidence assessment. This is the experimental warm-up.
4. **Promotion to active** — After the shadow period, if the source's accuracy, calibration, and reliability meet thresholds, it is promoted to ACTIVE status and its results feed into confidence calculations.
5. **No code change** — The Trust Engine's core reasoning logic does not change when sources are added or removed. The fusion algorithm operates on whatever evidence is available.

### 5.2 Handling an Obsolete Evidence Source

When an evidence source no longer produces useful results (e.g., a text detector that cannot detect current LLM outputs):

1. **Detection** — The Trust Engine detects obsolescence through:
   - Declining accuracy against ground truth benchmarks (if available)
   - Increasing rate of `INCONCLUSIVE` results (source is hedging)
   - Systematic disagreement with other sources that have better track records
   - Source version has not been updated in >12 months (stale source penalty)
2. **Weight decay** — The source's trust weight automatically decays. No human intervention required; it's a continuous function of measured accuracy.
3. **Deprecation notification** — When weight drops below a threshold, the source is marked DEPRECATED. Users with this source enabled are notified. The source's results appear in explainability panels with a deprecation warning.
4. **Retirement** — After a deprecation period (default: 30 days), the source is marked RETIRED and removed from active orchestration. Historical analyses that used this source retain their reference for audit purposes.
5. **No core logic change** — Obsolescence is handled entirely at the source-management layer. The Trust Engine's fusion algorithm never needs to know a source was retired — it simply stops receiving that source's results.

### 5.3 Handling an Improved Evidence Source

When an evidence source updates (new model version, improved accuracy):

1. **Version tracking** — The new version is registered separately. The Trust Engine tracks per-version accuracy.
2. **Concurrent evaluation** — If both old and new versions are available, the Trust Engine runs both for a transition period and compares their outputs.
3. **Promotion** — If the new version outperforms the old version across accuracy, calibration, and latency, the new version is promoted to ACTIVE and the old version is marked DEPRECATED.
4. **Rollback** — If the new version underperforms (regression), it is marked DEGRADED and the old version remains ACTIVE. The Trust Engine never auto-promotes a version that performs worse.
5. **User transparency** — When a source version changes, users are informed in the explainability panel: "Image Forensics Model v3.1 → v3.2 (improved accuracy)."

### 5.4 Weight Decay Model

Evidence sources do not retain their trust weight indefinitely without revalidation:

| Decay Factor | Rate | Rationale |
|---|---|---|
| **Version age decay** | Source loses 5% of its trust weight per month without a version update (floor: 50% of original) | Older models are less effective against newer AI generators |
| **Reference data staleness decay** | Source loses weight proportionally to how stale its reference data is (linear decay from last refresh to maximum age) | Stale reference data produces outdated results |
| **Accuracy trajectory decay** | If a source's measured accuracy over the past 30 days is declining, its trust weight is multiplied by the downward slope | Detects sources that are actively degrading |
| **No-decay sources** | Cryptographic provenance (C2PA signature verification) does NOT decay — it either verifies or it doesn't. Statistical and ML-based sources decay. Rule-based and cryptographic sources do not. |

### 5.5 Plug-and-Play Guarantee

The Trust Engine's core reasoning logic (the fusion algorithm, confidence calibration, explainability generation) must satisfy these invariants:

| Invariant | Guarantee |
|---|---|
| **Source-agnostic fusion** | The fusion algorithm operates on a list of Evidence Results. It does not reference specific sources by name. It does not branch on source identity. It reasons over signal values, evidence types, and confidence scores only. |
| **Variable evidence count** | The fusion algorithm handles any number of evidence results, from 0 (no sources available — returns "no evidence") to N (all sources ran). |
| **Variable evidence type** | The fusion algorithm does not assume specific evidence types exist. If provenance evidence is missing, it works with what remains. |
| **Graceful degradation** | When a source is unavailable, the Trust Engine proceeds with the remaining sources. Missing evidence is noted in the explanation ("Forensic analysis unavailable — source was offline") but does not block the assessment. |
| **No hardcoded thresholds per source** | Thresholds for "high confidence," "conflicting evidence," etc. are expressed in terms of evidence types and signal values, never in terms of specific sources. A new image forensic source that produces the same evidence type is handled identically. |

---

## 6. Evidence Source Trust Model

The Trust Engine does not trust all evidence sources equally. Source trust is continuously evaluated and affects how much weight each source's results carry in the fusion calculation.

### 6.1 Source Reputation Scoring

Every evidence source has a trust score maintained by the Trust Engine, computed from:

| Factor | Weight | Description |
|---|---|---|
| **Historical accuracy** | 40% | How often the source's signal agreed with the final consensus (or with ground truth when available). Measured over the source's lifetime with higher weight on recent analyses. |
| **Calibration quality** | 25% | How well the source's self-reported confidence matches its actual accuracy. A source that reports 0.95 confidence but is correct 70% of the time is miscalibrated and penalized. |
| **Independence** | 15% | How uncorrelated this source's results are with other sources. Sources that always agree with every other source provide no marginal information and are down-weighted. |
| **Availability** | 10% | Uptime. Sources that are frequently unavailable or timeout are less reliable and down-weighted. |
| **Freshness** | 10% | Version recency. Older versions are down-weighted per the decay model. |

**Trust score range:** 0.0 (completely untrusted — results should be ignored) to 1.0 (fully trusted — results taken at face value).

**Application:** A source's Evidence Result confidence is multiplied by the source's trust score before being fed into the fusion algorithm. A source with trust score 0.5 reporting confidence 0.95 is treated as having effective confidence 0.475.

### 6.2 Source Independence Requirement

The Trust Engine MUST ensure that the evidence it fuses comes from methodologically independent sources.

**Definition of independence:** Two sources are independent if they use fundamentally different approaches to reach their conclusions. An image forensics model that analyzes noise patterns and one that analyzes frequency spectra are independent. Two models that both analyze noise patterns using different architectures are NOT independent.

**Independence enforcement:**
- When multiple sources of the same class report results, the Trust Engine identifies the most trusted source in that class and gives it full weight. Other sources in the same class receive diminishing weight.
- If five image forensics sources all report the same finding but all use similar noise-analysis techniques, their ensemble weight is capped at the weight of the single best source in the class (they are not truly independent).
- If two sources report the same finding but one is provenance-based (C2PA) and one is forensic (noise analysis), they receive FULL independent weight because their approaches are fundamentally different.

**Practical effect:** Adding a tenth forensic source that is highly correlated with the existing nine provides almost no additional confidence. Adding one cryptographic provenance source to nine forensic sources provides significant additional confidence.

### 6.3 Source Provenance & Integrity

The Trust Engine must verify that an evidence source is what it claims to be before trusting its results.

| Verification | How |
|---|---|
| **Source identity** | Each source is identified by a cryptographic hash of its code + model weights. The Trust Engine verifies this hash against a signed manifest before loading. |
| **Model weight integrity** | ML model files are signed by their provider. The Trust Engine verifies the signature before loading. Unsigned models are treated as EXPERIMENTAL (results not used in confidence assessment). |
| **Code integrity** | Any dynamically registered source (user-installed, community plugin) must pass a sandbox security review before activation. Sources that execute arbitrary code without sandboxing are rejected. |
| **Supply chain integrity** | Built-in sources are bundled with the extension and verified at install time via the extension store's code signing. No runtime code loading from external sources. |

### 6.4 Conflicting Source Resolution

When two sources of the same evidence type produce conflicting results:

| Scenario | Resolution Rule |
|---|---|
| **Same type, similar confidence, different signals** | Trust neither. Flag as `CONFLICTING` evidence. The fusion algorithm treats this as evidence of uncertainty, not evidence of either signal. Presented to user: "Forensic sources disagree — some indicate authentic, some indicate synthetic." |
| **Same type, one has significantly higher source trust** | The higher-trust source's signal receives proportionally more weight. The lower-trust source's signal is not ignored but is discounted. The degree of discount is proportional to the trust disparity. |
| **Same type, one is provenance-based, one is forensic** | Provenance (cryptographic) takes precedence over forensic (statistical) when they conflict. A valid C2PA manifest saying "created by human with a camera" outweighs a forensic model saying "looks AI-generated." The forensic model might be detecting AI post-processing or compression artifacts, not generation. |
| **Multiple sources conflict with no clear resolution** | The Trust Engine returns `INCONCLUSIVE` with high confidence in its uncertainty. The explanation enumerates the conflicting evidence. This is a valid and useful outcome: "Sentinel is confident that it cannot determine authenticity." |

**Critical rule:** When sources conflict, the Trust Engine must NEVER silently pick a winner. The conflict must be surfaced to the user in the explainability panel. The user must know that evidence is ambiguous.

---

## 7. Reference Data Management

Evidence sources that depend on locally cached reference data (domain lists, known hashes, publisher databases, certificate lists) must follow these rules.

### 7.1 Freshness Requirements

| Reference Data Type | Maximum Staleness | Refresh Cadence | Degradation |
|---|---|---|---|
| Certificate revocation lists (CRLs) | 24 hours | Every 6 hours | After 24h: source returns `ERROR (UNAVAILABLE)`. CRL staleness is a hard failure. |
| Domain reputation lists | 7 days | Daily | After 3 days: source trust weight begins linear decay. After 7 days: source returns `INCONCLUSIVE` with explanation that data is stale. |
| Known AI content hashes | 30 days | Weekly | After 14 days: trust weight begins linear decay. After 30 days: source returns `INCONCLUSIVE`. |
| Publisher credibility database | 30 days | Weekly | After 14 days: trust weight decays. After 30 days: `INCONCLUSIVE`. |
| AI model fingerprint database | 90 days | Monthly | After 45 days: trust weight decays. After 90 days: `INCONCLUSIVE`. |
| Known watermark patterns | 90 days | Monthly | After 45 days: trust weight decays. After 90 days: `INCONCLUSIVE`. |

**Refresh mechanism:** The Trust Engine periodically checks for updated reference data bundles. Bundles are downloaded as signed packages, verified, and atomically swapped with the active bundle. During the swap, queries continue against the previous bundle — no downtime.

### 7.2 Integrity Requirements

| Requirement | Implementation Rule |
|---|---|
| **All reference data bundles must be signed** | The bundle provider (Sentinel infrastructure or trusted third party) signs each bundle with a private key. The Trust Engine verifies the signature using the provider's public key (bundled with the extension) before loading. Unsigned bundles are rejected. |
| **Bundle hash verification** | Each bundle includes a SHA-256 hash manifest. The Trust Engine verifies that the downloaded content matches the manifest before loading. |
| **Atomic update** | A new bundle is downloaded to a staging location, verified completely, then atomically swapped with the active bundle. If verification fails at any point, the active bundle is retained and the download is retried on the next refresh cycle. |
| **Rollback capability** | The Trust Engine retains the previous valid bundle version. If a new bundle causes widespread source errors, the Trust Engine automatically reverts to the previous bundle and flags the new bundle as corrupted. |
| **Tamper detection** | If the Trust Engine detects that a loaded bundle has been modified on disk (hash mismatch at load time), it reverts to a known-good backup or disables the dependent sources until a fresh bundle is downloaded. |

### 7.3 Fallback & Degradation

| Scenario | Behavior |
|---|---|
| **Reference data is stale but within acceptable window** | Source runs normally with reduced trust weight proportional to staleness |
| **Reference data is beyond maximum staleness** | Dependent source returns `INCONCLUSIVE`. Trust Engine proceeds with other evidence sources. User sees: "Reputation check unavailable — reference data is out of date." |
| **Reference data corrupted (hash mismatch)** | Dependent source is immediately disabled. Trust Engine reverts to previous valid bundle. If no previous bundle exists, dependent sources return `ERROR (UNAVAILABLE)` until a fresh download succeeds. |
| **No network to refresh reference data** | Sources continue with the currently loaded data until it exceeds maximum staleness, at which point they return `INCONCLUSIVE`. Network-independent local detectors (forensics, metadata) are unaffected. |
| **First launch, no reference data downloaded yet** | All reference-data-dependent sources return `INCONCLUSIVE` with explanation: "Reference data not yet downloaded. Analysis will improve after initial sync." Tier 1 and Tier 2 local detectors still function normally. |

### 7.4 User Contribution to Reference Data

Users MAY contribute to reference data (domain reputation, content hash reporting) under these rules:

| Rule | Description |
|---|---|
| **Opt-in only** | Users must explicitly enable contribution. It is OFF by default. |
| **Anonymized** | Contributions are stripped of user identity. Only the signal (domain, hash, report type) is sent. |
| **Rate-limited** | Users cannot submit the same report type for the same target more than once per day. |
| **Verifiable** | Contributions that can be independently verified (e.g., "this image has a valid C2PA manifest that declares AI generation") are weighted higher than unverifiable contributions ("this content seems suspicious"). |
| **Weighted, not binary** | A single user report does not create a reputation entry. A threshold of independent reports is required before a user contribution affects the aggregate reputation signal. |
| **Decay** | Community-contributed reputation signals decay faster than authoritative reference data. A domain flagged by users but not re-flagged within 30 days loses its reputation penalty. |
| **Appeal mechanism** | Content owners who believe community signals are incorrect must have a path to appeal. Appeals are reviewed, not automated. |

---

## 8. Cross-Cutting Rules

These rules apply across all sections above. They are guaranteed by the Trust Engine's design, not by individual source compliance.

### 8.1 Privacy Invariants

| Invariant | Enforcement |
|---|---|
| No content leaves the device without explicit, per-source user consent | The Trust Engine checks consent state before every Cloud API invocation. Consent is per-source, not blanket. |
| User personal allow/block lists never leave the device | Stored in local encrypted storage. Never included in any network request. |
| Evidence results from Cloud API sources are annotated with a privacy impact indicator in the explainability panel | Users see: "This signal required sending content to [service name]. Your content was transmitted to their servers." |
| Users can see exactly which sources accessed their content and when | Local audit log, stored on-device, cleared on user request. |

### 8.2 Performance Invariants

| Invariant | Enforcement |
|---|---|
| The Trust Engine never blocks the browser's main thread | All analysis executes in sandboxed pages, service workers, or Web Workers. Content scripts only report DOM metadata — no computation. |
| First evidence is displayed within 100ms of analysis start | Tier 0 (user signals) and Tier 1 (immediate local) are guaranteed to return within this window. If Tier 1 sources are unavailable, a placeholder is displayed: "Analyzing..." |
| Progressive results never regress | Once a confidence assessment is displayed, it may be refined (more precise) or updated (new evidence added) but never completely reversed without explanation. If new evidence contradicts earlier evidence, the contradiction is explained: "Initial analysis suggested authenticity, but deeper forensic analysis found synthesis artifacts." |

### 8.3 Explainability Invariants

| Invariant | Enforcement |
|---|---|
| Every confidence assessment must cite specific evidence sources and their signals | "78% confidence — based on valid C2PA manifest (provenance), consistent noise patterns (forensic), and trusted publisher reputation." |
| When evidence is missing, the absence is explained | "Reputation analysis was not available — no network connection for reference data refresh." |
| When evidence conflicts, the conflict is surfaced | "Provenance indicates authenticity, but forensic analysis found manipulation artifacts. These signals conflict." |
| Users can drill into any evidence signal to see the raw result, source identity, and source trust score | The explainability panel is hierarchical: summary → evidence list → per-source detail. |

### 8.4 Security Invariants

| Invariant | Enforcement |
|---|---|
| Evidence sources run in sandboxed contexts with no access to extension APIs, storage, or network (except declared Cloud API sources) | Sandbox enforcement at the browser level (sandboxed pages with strict CSP). Cloud API sources get narrowly scoped fetch permissions. |
| Dynamic sources (user-installed, community) are more sandboxed than built-in sources | Built-in sources have access to local model files and reference data. Dynamic sources have access only to their own sandboxed storage and must request model/data access through the Trust Engine. |
| A compromised evidence source cannot poison the Trust Engine's trust calculations for other sources | Source trust scores are computed independently per source. One source's performance does not affect another's trust score. |
| Evidence source results are immutable once produced | Results are written to a local append-only log. A source cannot retroactively modify its past results. |

---

## 9. Summary of Guarantees

The Trust Engine guarantees the following to every entity that interacts with it:

| To Evidence Sources | To Users | To Developers |
|---|---|---|
| You will be invoked only for content types you declare capability for | Your private data stays on your device unless you explicitly opt in to cloud analysis | You can add a new evidence source without modifying the Trust Engine's core logic |
| Your results will be weighted by your measured accuracy, not by who developed you | You will always know WHY a confidence assessment was reached | You can remove an evidence source without breaking the system |
| Your self-reported confidence will be calibrated against your actual performance | You will see progressive results — never a blank loading screen | The fusion algorithm works with any number of sources of any type |
| You will not be blocked indefinitely by a predecessor source | You control which sources run and on which websites | Evidence sources are interchangeable if they produce the same evidence type |
| Your errors and timeouts will not crash the Trust Engine | Conflicting evidence will be shown to you, not silently resolved | Source trust scoring, decay, and deprecation are automatic — no manual maintenance |

---

*End of Trust Engine & Evidence Source Ecosystem Specification.*
