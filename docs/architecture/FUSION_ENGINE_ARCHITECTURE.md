# Evidence Fusion Engine & Confidence Model Architecture

> **Document Type:** Architecture Design
> **Phase:** 1 — Architecture
> **Status:** Draft
> **Owner:** Architect Agent
> **Audience:** Trust Engine Designer, ML Researcher, Browser Extension Engineer
> **Depends On:** TRUST_ENGINE_SPEC.md (authoritative), DISCOVERY_REPORT.md
> **Scope:** Defines the evidence graph structure, fusion algorithm, conflict detection, uncertainty propagation, confidence model, and edge case handling for Sentinel's Trust Engine.

---

## Table of Contents

1. [Evidence Graph Structure](#1-evidence-graph-structure)
2. [Fusion Algorithm Architecture](#2-fusion-algorithm-architecture)
3. [Conflict Detection & Resolution](#3-conflict-detection--resolution)
4. [Uncertainty Propagation](#4-uncertainty-propagation)
5. [Confidence Model](#5-confidence-model)
6. [Edge Cases](#6-edge-cases)
7. [Architecture Decision Summary](#7-architecture-decision-summary)

---

## 1. Evidence Graph Structure

### 1.1 Graph Topology

The Fusion Engine operates on a **directed acyclic graph (DAG)** where nodes are immutable Evidence objects and edges encode relationships between them. The graph is the single source of truth for the current state of all evidence gathered for a content assessment.

**Graph constraints:**
- The DAG is per-assessment (one assessment = one graph instance)
- Nodes are append-only — once added, an Evidence node is never modified
- Edges are created based on computed relationships; they are also append-only
- The graph supports partial construction — the engine can produce assessments from incomplete graphs
- No cycles are permitted; `supersedes` edges must point forward in time (newer version → older version)

### 1.2 Evidence Nodes

Each Evidence node is an immutable object produced by an analyzer in the pipeline. A node is created when an analyzer completes its computation and emits an Evidence record.

**Node identity:**
- `evidence_id`: Content-hash of the signal's output (deterministic, reproducible)
- `assessment_id`: Which assessment this evidence belongs to (ties evidence to a specific page/content analysis)

**Node payload (immutable, set at creation time):**

| Field | Type | Description |
|-------|------|-------------|
| `signal_type` | Enum | One of ~20 evidence types (C2PA_VERIFICATION, ELA_ANALYSIS, PERPLEXITY_ANALYSIS, etc.) |
| `evidence_class` | Enum | One of 6 classes: PROVENANCE, IMAGE_FORENSICS, TEXT_ANALYSIS, VIDEO_FORENSICS, AUDIO_FORENSICS, REPUTATION_CONTEXTUAL |
| `tier` | 1-4 | Evidence tier per the hierarchy in TRUST_ENGINE_SPEC.md |
| `direction` | -1, 0, +1 | -1 = evidence for AI-generated, 0 = neutral/informational only, +1 = evidence for authentic |
| `likelihood_ratio` | Float > 0 | P(evidence \| AI) / P(evidence \| authentic), calibrated from detector performance |
| `lr_variance` | Float > 0 | Variance of the log-likelihood ratio (the signal's measurement uncertainty) |
| `raw_score` | Float [0,1] or null | The detector's raw output before calibration to LR |
| `confidence` | Float [0,1] | The signal's self-reported confidence in its own measurement |
| `timestamp` | Integer (ms) | When this evidence was produced (wall clock, for ordering and staleness) |
| `detector_version` | String | Version identifier of the analyzer that produced this signal |
| `detector_durability` | Float [0,1] | Expected signal lifetime before accuracy degradation (1.0 = permanent like C2PA, 0.3 = degrades quickly like text classifiers) |
| `analysis_target` | Object | What was analyzed (e.g., `{url: "...", element_hash: "...", content_type: "image"}`) — no raw content stored |
| `unavailability_reason` | Enum or null | Why this evidence was NOT gathered (CORS_BLOCKED, SHADOW_DOM, BROWSER_UNSUPPORTED, OFFLINE, PLATFORM_STRIPPED). Present only for "absence evidence" nodes. |

**Node lifecycle:**
1. **Created** when an analyzer emits evidence
2. **Added** to the graph immediately (no batching at the graph level)
3. **Never modified** after creation
4. **Never removed** — if a user disables a signal class, that class's nodes remain in the graph but are marked as `excluded: true` via the graph's filter context

### 1.3 Edge Types

The graph supports five edge types, each encoding a distinct relationship:

#### `supports(A, B, strength)`
Both nodes point in the same direction. The strength is the cosine similarity of their directional contribution vectors in log-odds space. Used during fusion to identify convergent evidence.

**Creation rule:** Automatically created when two nodes have the same `direction` sign (±1) and the dot product of their normalized log-LR contributions exceeds 0.3. Created at node insertion time via pairwise comparison.

#### `contradicts(A, B, severity)`
Nodes point in opposite directions. Severity maps to the 4-tier conflict model (Critical/High/Moderate/Low) determined by multiplying their individual confidences.

**Creation rule:** Automatically created when two nodes have opposite `direction` signs and both have `confidence > 0.3`. The severity is: `min(confidence_A, confidence_B)` mapped through thresholds: ≥0.8 → Critical, ≥0.6 → High, ≥0.4 → Moderate, otherwise Low.

#### `derived_from(B, A)`
Node B's analysis consumed or depended on the output of node A. Example: cross-modal consistency analysis derives from both text extraction and image forensics nodes.

**Creation rule:** Explicitly declared by the analyzer that produces B. The analyzer lists which evidence IDs it consumed as inputs. No automatic derivation detection.

#### `correlates_with(A, B, coefficient)`
Nodes A and B measure overlapping phenomena and their outputs are statistically correlated. The correlation coefficient ρ ∈ [0,1] comes from a pre-computed correlation matrix maintained offline from calibration data.

**Creation rule:** Looked up from the correlation matrix at node insertion time. If the matrix contains an entry for `(signal_type_A, signal_type_B)` with ρ > 0.1, the edge is created. The correlation matrix is versioned and shipped as configuration data alongside the extension.

#### `supersedes(B, A)`
Node B is a newer version of the same signal type as A (e.g., ELA model v2 replaces v1). During fusion, only the latest version contributes weight; older versions are informational.

**Creation rule:** Created when two nodes share the same `signal_type` and `analysis_target` but B has a newer `detector_version`. The older node is NOT removed — it remains in the graph for audit trail and explainability.

### 1.4 Graph Construction

**When nodes are added:** Immediately upon evidence emission from any analyzer. The Fusion Engine subscribes to an evidence event stream. Each arrival triggers graph insertion.

**Graph insertion algorithm (prose):**

1. The new Evidence node receives a unique `evidence_id` (deterministic hash of its contents)
2. The node is added to the graph's node set
3. For every existing node already in the graph:
   - Check `direction` signs: if opposite AND both confidences > 0.3 → create `contradicts` edge
   - Check `direction` signs: if same AND normalized log-LR dot product > 0.3 → create `supports` edge
   - Check `signal_type` and `analysis_target`: if same → create `supersedes` edge (newer version wins)
   - Check correlation matrix: if `(new.signal_type, existing.signal_type)` has ρ > 0.1 → create `correlates_with` edge
4. If the new node has declared `derived_from` inputs, create those edges
5. The graph integrity is validated (no cycles introduced — `derived_from` and `supersedes` always respect temporal ordering)

**Computational bounds:** Pairwise comparison with all existing nodes is O(n) per insertion where n is the number of existing nodes. Evidence nodes per assessment are bounded (realistically < 50 nodes even with all analyzers firing). This is trivially fast.

### 1.5 Asynchronous Evidence Arrival

Evidence arrives progressively per the timing model defined in BROWSER_CAPABILITY_ASSESSMENT.md:

| Wave | Timing | Evidence Types |
|------|--------|---------------|
| Wave 1 | < 500ms | Content type detection, domain/URL extraction, content volume metrics, cached domain reputation, C2PA absence flag |
| Wave 2 | 1-5s | C2PA verification results, EXIF/IPTC metadata, text style/pattern analysis, image metadata, cloud reputation lookup |
| Wave 3 | User-invoked | Image forensics (ELA, noise, frequency), AI generator fingerprints, video frame analysis, audio analysis, cross-modal consistency |

**The graph handles asynchrony natively:**
- The graph does not "wait" for any wave to complete
- Nodes are added as they arrive, regardless of expected timing
- The fusion engine can compute from a partially populated graph at any moment
- There is no concept of a "complete" graph — only the current state
- Late-arriving evidence (e.g., a slow cloud lookup returning after 8 seconds) is inserted exactly like any other node, and triggers re-computation

### 1.6 Graph Querying for Fusion

The fusion algorithm queries the graph for:
- All nodes not marked `excluded`
- All `supports` edges to compute agreement clusters
- All `contradicts` edges to compute conflict
- All `correlates_with` edges to compute dependency adjustments
- The single active node per `signal_type + analysis_target` (latest version via `supersedes` traversal)

The graph does NOT perform the fusion computation itself. It is a pure data structure. The fusion algorithm reads the graph as input.

---

## 2. Fusion Algorithm Architecture

### 2.1 High-Level Fusion Pipeline

```
Evidence Graph (current state)
        │
        ▼
  ┌─────────────────────────┐
  │ 1. DEPENDENCY RESOLVER  │  Group correlated signals, cap class weights
  └───────────┬─────────────┘
              │
              ▼
  ┌─────────────────────────┐
  │ 2. BAYESIAN ACCUMULATOR │  Compute posterior from likelihood ratios
  └───────────┬─────────────┘
              │
              ▼
  ┌─────────────────────────┐
  │ 3. CONFLICT DETECTOR    │  Identify and tier contradictions
  └───────────┬─────────────┘
              │
              ▼
  ┌─────────────────────────┐
  │ 4. SAFETY OVERRIDE      │  Apply confidence floors from Tier 1 signals
  └───────────┬─────────────┘
              │
              ▼
  ┌─────────────────────────┐
  │ 5. CONFIDENCE CLASSIFIER│  Map posterior + constraints → discrete level
  └───────────┬─────────────┘
              │
              ▼
  ┌─────────────────────────┐
  │ 6. DIMENSION COMPUTER   │  Compute all 8 confidence dimensions
  └───────────┬─────────────┘
              │
              ▼
       Fusion Output
```

### 2.2 Step 1: Dependency Resolver

**Purpose:** Detect and account for correlated signals so that dependent evidence is not double-counted in the Bayesian accumulator.

**Algorithm (prose):**

1. **Resolve superseded signals.** For each `(signal_type, analysis_target)`, select only the node with the latest `detector_version` for fusion contribution. Superseded nodes are retained for audit trail but contribute zero weight.

2. **Group nodes by evidence class.** Partition all active nodes into their 6 evidence classes.

3. **Detect correlations within each class.** For each pair of nodes in the same class, check the `correlates_with` edge. If correlation coefficient ρ > 0.3, the pair is flagged as dependent.

4. **Cap individual signal weight at 40%.** For each node, compute its raw log-LR contribution. If any single node would contribute more than 40% of the total absolute log-LR sum, its contribution is clamped to that ceiling. The excess is redistributed proportionally to other uncapped nodes in the same class. If all nodes in the class are capped, the excess is discarded (lost to the dependency penalty).

5. **Cap evidence class weight at 60%.** After individual capping, compute the total absolute log-LR contribution per evidence class. If any class exceeds 60% of the total, scale all nodes in that class down proportionally until the class total is exactly 60%.

6. **Apply correlation penalty.** For each correlated pair (ρ > 0.3), reduce their combined effective contribution. The adjusted combined log-LR for a dependent pair A and B is:

   ```
   adjusted_combined = log_LR_A + log_LR_B * (1 - ρ_A,B)
   ```

   where the node with the smaller individual contribution is the one discounted. This penalizes the weaker signal in a correlated pair rather than averaging both down, preserving the stronger signal's full contribution.

7. **Output adjusted log-LR values** for each node, ready for the Bayesian accumulator.

**Pre-computed correlation matrix location:** Shipped as a JSON configuration file bundled with the extension. Updated via periodic background sync (not per-analysis). Versioned alongside detector versions.

**Example correlation matrix entries:**

| Signal A | Signal B | ρ | Rationale |
|----------|----------|---|-----------|
| JPEG_ARTIFACT_ANALYSIS | FREQUENCY_DOMAIN_ANALYSIS | 0.65 | Both operate on DCT coefficients |
| FREQUENCY_DOMAIN_ANALYSIS | AI_GENERATOR_FINGERPRINT | 0.45 | GAN fingerprints manifest in frequency domain |
| ELA_ANALYSIS | JPEG_ARTIFACT_ANALYSIS | 0.35 | Both detect compression inconsistencies |
| PERPLEXITY_ANALYSIS | STYLE_CONSISTENCY | 0.30 | Both measure statistical properties of text |
| C2PA_VERIFICATION | EXIF_METADATA | 0.10 | C2PA is cryptographic, EXIF is declarative — essentially independent |

### 2.3 Step 2: Bayesian Accumulator

**Purpose:** Combine independent evidence signals into a posterior probability using Bayesian evidence accumulation.

**Algorithm (prose):**

1. **Determine the prior probability.** The prior P(AI) — the base rate belief that content is AI-generated before any evidence is evaluated — is determined from:
   - Content type: image, text, video, audio, or mixed. Each has a different base rate.
   - Domain context: social media platforms have higher AI base rates than news organization domains.
   - The prior is intentionally conservative: it defaults to P(AI) = 0.35 for unknown contexts (slightly below the estimated web average of ~40%, biasing toward "presume authentic unless evidence suggests otherwise").
   - Domain-specific priors are adjusted by the domain reputation signal when available.

   **Per-content-type default priors:**

   | Content Type | P(AI) Default | Rationale |
   |-------------|---------------|-----------|
   | Social media image | 0.45 | High AI content prevalence on social platforms |
   | News organization image | 0.15 | News orgs have editorial standards; lower AI prevalence in journalism |
   | Generic web image | 0.35 | Unknown context, conservative estimate |
   | Social media text | 0.50 | Significant AI-generated text on social platforms |
   | News article text | 0.20 | Editorial standards; AI use is growing but not dominant |
   | Generic web text | 0.35 | Unknown context |
   | Video (any context) | 0.30 | AI video is less prevalent than images due to compute cost |
   | Audio (any context) | 0.25 | AI audio is growing but not yet dominant |

2. **Compute log-prior-odds:**

   ```
   log_prior_odds = ln(P(AI) / (1 - P(AI)))
   ```

3. **Accumulate adjusted log-likelihood ratios.** For each node with adjusted log-LR value (from Step 1):

   ```
   log_posterior_odds = log_prior_odds + Σ (direction_i * log(adjusted_LR_i))
   ```

   Where `direction_i` is +1 (evidence for AI) or -1 (evidence for authentic). The sign convention: positive log_posterior_odds → favors AI-generation; negative → favors authenticity.

4. **Compute posterior probability:**

   ```
   P(AI | evidence) = exp(log_posterior_odds) / (1 + exp(log_posterior_odds))
   ```

   This is the raw Bayesian posterior. It is NOT the final confidence level — it feeds into the Confidence Classifier (Step 5) which maps it to a discrete level.

5. **Compute posterior variance (for credible intervals).** The variance of the log-posterior-odds is:

   ```
   var(log_posterior_odds) = Σ var(log_LR_i) + 2 * Σ Σ ρ_jk * sqrt(var_j * var_k)
   ```

   Where `var(log_LR_i)` is the `lr_variance` from each Evidence node, and `ρ_jk` is the correlation coefficient for each correlated pair. This accounts for the fact that dependent signals' variances are not additive.

   From the variance, the 85% credible interval is computed:

   ```
   lower_log_odds = log_posterior_odds - 1.44 * sqrt(var_log_posterior)
   upper_log_odds = log_posterior_odds + 1.44 * sqrt(var_log_posterior)
   
   credible_lower = exp(lower_log_odds) / (1 + exp(lower_log_odds))
   credible_upper = exp(upper_log_odds) / (1 + exp(upper_log_odds))
   ```

   (Where 1.44 is the z-score for an 85% interval — chosen over 95% because narrower intervals are less likely to mislead non-technical users into over-trusting the precision.)

6. **Handle degenerate cases:**
   - If **zero evidence nodes exist:** posterior = prior, variance = ∞ (total uncertainty), credible interval = [0, 1]. This maps to "Insufficient Evidence."
   - If **only Tier 4 nodes:** posterior is computed normally, but the tier quality constraint in Step 5 will cap the discrete confidence level.
   - If **all nodes have the same direction:** posterior may converge to very high or very low values. The variance comes only from individual signal uncertainties, not from disagreement (no conflict penalty). This is the desired behavior — strong agreement among independent signals SHOULD produce a tight posterior.
   - If **likelihood ratio is exactly 1.0** (signal provides zero information): that node contributes 0 to the log-posterior-odds, which is correct behavior.

### 2.4 Why Not Simple Weighted Averaging

The spec explicitly rejects simple weighted averages. Beyond that rejection, the mathematical justification:

- **Averages hide conflict.** A +0.9 and a -0.8 signal average to +0.05 — a number that reads as "neutral" but actually encodes violent disagreement.
- **Averages misrepresent uncertainty.** A single strong signal and five weak signals produce the same average as six moderate signals, but the certainty is radically different.
- **Averages cannot model conditional dependence.** Bayesian accumulation with correlation penalties explicitly accounts for the fact that JPEG artifact analysis and frequency domain analysis overlap.
- **Averages produce continuous scores that users interpret as percentages.** A "78% score" invites the user to think "78% certain." Bayesian posteriors feed into discrete levels, preventing the false-precision problem.

### 2.5 Re-computation Triggers

The fusion pipeline re-computes the assessment output when any of the following events occur:

| Trigger | Latency | Behavior |
|---------|---------|----------|
| Single evidence node added | Immediate (< 5ms) | Full re-computation from scratch (fast — reads graph, not raw evidence) |
| Multiple nodes added in same wave | Batched (200ms debounce) | Wave 1 completion triggers one re-computation; Wave 2 arrivals debounced |
| User disables a signal class | Immediate | Re-computes with nodes of that class marked `excluded` |
| User requests Tier 3 analysis | Immediate on each Tier 3 result arrival | Each forensics model result triggers re-computation (sparse, user-initiated) |
| Browser transitions to offline | Immediate | Cloud-dependent in-flight evidence is cancelled; re-computes with local-only nodes |
| Browser transitions to online | Immediate | Triggers refresh of any stale cloud evidence; re-computes if new evidence arrives |
| Time-based staleness check | Periodic (every 60s for active assessments) | For assessments visible in the UI, re-evaluates whether cloud evidence has gone stale |

**Debouncing rationale for Wave 2:** Wave 2 signals (C2PA verification, EXIF, text style, image metadata) complete within a ~3-second window. Re-computing after every single arrival would cause the confidence level to flicker 5-8 times in rapid succession. A 200ms debounce window means the UI updates at most ~15 times in that window — perceptible as a smooth refinement rather than a flicker.

**The debounce does NOT delay the final result.** When the debounce timer expires, it processes all queued evidence arrivals in one batch. If a new arrival occurs during the debounce window, the timer resets. The worst-case additional latency from debouncing is 200ms, which is imperceptible on top of the 1-5s Wave 2 window.

### 2.6 Progressive Refinement UX Mapping

The fusion output transitions through these states as evidence arrives:

| State | Timing | Available Evidence | Discrete Confidence Level Limits |
|-------|--------|-------------------|----------------------------------|
| **Initial** | 0ms | Prior only (no evidence) | Capped at "Insufficient Evidence" |
| **Preliminary** | < 500ms | Wave 1 complete (domain, content type, volume, cached reputation) | Capped at "Low Confidence" |
| **Refined** | 1-5s | Wave 2 complete (C2PA, EXIF, text style, image metadata, cloud reputation) | Up to "Very High Confidence" if Tier 1 signals present |
| **Deep** | User-invoked | Wave 3 (image forensics, AI fingerprints, video, audio, cross-modal) | Full range available |
| **Final** | All waves complete | All applicable evidence gathered | Final assessment locked until re-analysis requested |

The UI displays the current state label alongside the confidence level: "Preliminary — Low Confidence" → "Refined — High Confidence" → "Deep — Very High Confidence."

---

## 3. Conflict Detection & Resolution

### 3.1 Conflict Detection Algorithm

**Definition:** Two evidence nodes are in conflict when they point in opposite directions AND both carry sufficient individual weight that neither can be dismissed as noise.

**Step 1: Pairwise direction check.**
For each pair of active nodes (A, B) where both are NOT excluded:
- If `direction_A ≠ direction_B` AND neither direction is 0 (neutral), the pair is a candidate conflict.

**Step 2: Significance threshold.**
A candidate pair is a "detected conflict" only if both nodes clear a minimum significance bar:
- `confidence_A > 0.3` AND `confidence_B > 0.3`

A signal with confidence ≤ 0.3 is too uncertain to constitute a meaningful conflict. It is simply a weak contradictory whisper, noted but not escalated.

**Step 3: Conflict severity calculation.**
For each detected conflict pair, the severity is determined by the geometric mean of the two confidences:

```
conflict_score = sqrt(confidence_A * confidence_B)
```

Mapped to the 4-tier model:

| conflict_score | Severity Tier | Label |
|---------------|---------------|-------|
| ≥ 0.85 | CRITICAL | "Evidence is in critical contradiction" |
| ≥ 0.65 | HIGH | "Evidence strongly disagrees" |
| ≥ 0.45 | MODERATE | "Evidence partially disagrees" |
| < 0.45 | LOW | "Minor evidence disagreement" |

**Step 4: Tier-based escalation modifier.**
The tier of the conflicting nodes modifies the severity:

| Conflict Pair Tiers | Severity Adjustment |
|--------------------|---------------------|
| Tier 1 vs Tier 1 | Escalate to CRITICAL regardless of score |
| Tier 1 vs Tier 2 | Escalate by one tier (capped at CRITICAL) |
| Tier 1 vs Tier 3+ | No escalation (Tier 1 dominates) |
| Tier 2 vs Tier 2 | No adjustment |
| Tier 3+ vs anything | De-escalate by one tier (lower-tier signal is expected to be weaker) |

**Step 5: Aggregated conflict state.**
The assessment has one overall conflict state, derived from the most severe conflict pair present in the graph:

```
overall_conflict = max(severity of all conflict pairs)
```

If zero conflicts are detected, `overall_conflict = NONE`.

### 3.2 The 4-Tier Conflict Model in Detail

#### CRITICAL Conflict
**Definition:** Two cryptographic or near-cryptographic signals disagree, or two very-high-confidence detection signals directly contradict.

**Triggers:**
- Two conflicting C2PA manifests exist for the same content (e.g., content signed by two different entities making contradictory claims)
- Hardware-attested C2PA conflicts with known synthetic signature hash match
- Any two signals both with ≥ 0.85 individual confidence pointing in opposite directions

**System behavior:**
- Confidence level capped at "Low Confidence" regardless of other evidence
- The credible interval is widened to its maximum extent
- The conflict is flagged prominently in the UI as the primary message
- Suggested next steps: "This content has irreconcilable provenance claims. Manual investigation required."
- The fusion does NOT attempt to resolve the conflict — it surfaces it

#### HIGH Conflict
**Definition:** A Tier 1 signal conflicts with a strong Tier 2 signal, or two strong Tier 2 signals disagree.

**Triggers:**
- Valid C2PA verification (Tier 1, authentic) vs. strong frequency-domain artifact detection (Tier 2, AI-generated)
- Cross-modal physics violation (Tier 2, AI) vs. known watermark detection (Tier 2, authentic)

**System behavior:**
- Confidence level capped at "Moderate Confidence"
- Both perspectives are presented with their evidence
- Possible explanations are offered: "Valid provenance but AI artifacts detected. Possible: AI content was signed after generation, editing introduced artifacts, or one signal is a false positive."
- The posterior is still computed but flagged as "disputed"

#### MODERATE Conflict
**Definition:** Two Tier 2 signals disagree, or a Tier 2 signal conflicts with aggregated Tier 3 signals.

**Triggers:**
- Frequency-domain analysis suggests AI but audio vocoder artifacts not detected (both Tier 2, opposite directions)
- AI generator fingerprint (Tier 2) suggests AI but IPTC metadata looks consistent (Tier 3)

**System behavior:**
- Noted in the explainability panel
- Does not cap the confidence level (the Bayesian posterior already reflects the disagreement)
- Listed under "Disagreeing Signals" in the evidence summary

#### LOW Conflict
**Definition:** A weak signal (Tier 3/4) disagrees with a strong signal (Tier 1/2), or two weak signals disagree.

**Triggers:**
- Tier 1 C2PA says authentic, Tier 4 text burstiness suggests AI
- Tier 3 face-swap detection disagrees with Tier 4 community reports

**System behavior:**
- NOT surfaced as a "conflict" to the user (would create noise)
- The lower-tier signal is simply noted in the evidence detail and effectively outweighed
- Appears in the evidence breakdown as "Outweighed: [signal name] suggested [X] but was overridden by stronger signals"

### 3.3 "Conflict" vs. "Merely Disagree" Threshold

The distinction between a conflict (actionable contradiction) and mere disagreement (normal noise in a probabilistic system) is:

| Characteristic | Mere Disagreement | Conflict |
|---------------|-------------------|----------|
| Signal tier differential | Tier 4 disagrees with Tier 1 | Tier 1 disagrees with Tier 1 or Tier 2 |
| Confidence product | < 0.3 × 0.3 (at least one signal is weak) | ≥ 0.45 × 0.45 (both signals carry weight) |
| User impact | The assessment is still useful — one signal is clearly less reliable | The assessment's conclusion is genuinely unclear — both signals are credible |
| System action | Lower-tier signal noted, not escalated | Escalated per severity tier |

The threshold is: **both signals must have confidence ≥ 0.45 AND the tiers must be within one level of each other** for the disagreement to be treated as a conflict rather than noise.

### 3.4 Safety Override Mechanism (Non-Brittle Design)

The spec mandates that hardware-attested C2PA creates a confidence floor that cannot be overridden by detection signals. Rather than implementing this as brittle hard-coded rules ("if hardware C2PA, set confidence to HIGH"), the safety overrides are implemented as **posterior floor constraints** — parameterized, transparent, and revisable.

**Floor Constraint Model:**

Each safety override is a constraint of the form:
```
IF condition THEN P(authentic) ≥ floor_value
```

The condition is a predicate over the evidence graph. The floor value is a configuration parameter.

| Override | Condition | Floor | Rationale | Overridable By |
|----------|-----------|-------|-----------|----------------|
| Hardware-attested C2PA | Node exists with `signal_type=C2PA_HARDWARE_ATTESTED` AND `direction=+1` AND cert chain valid | P(authentic) ≥ 0.70 | Cryptographic signing at silicon level. Near-impossible to forge without hardware compromise. | CA revocation, known synthetic hash match, conflicting hardware attestation |
| Standard C2PA (recognized CA) | Node exists with `signal_type=C2PA_VERIFICATION` AND `direction=+1` AND CA in recognized trust registry | P(authentic) ≥ 0.55 | Valid cert chain from audited CA. Strong but not absolute — the signer could be malicious. | Known synthetic hash match, Tier 2 cross-modal physics violation |
| Known synthetic hash match | Node exists with `signal_type=KNOWN_SYNTHETIC_HASH` AND `direction=-1` AND hash match verified | P(AI) ≥ 0.75 | Content matches known AI-generated output exactly. Near-deterministic evidence of AI origin. | Hardware-attested C2PA (the only signal that can override — if both fire, it's a CRITICAL conflict) |

**Enforcement mechanism:**
1. The Bayesian accumulator computes the raw posterior P(authentic | evidence)
2. The safety override layer checks each active constraint
3. If any constraint's condition is met and the raw posterior violates the floor, the posterior is clamped to the floor value
4. The clamped value is annotated with which constraint was applied and why
5. Multiple constraints can apply simultaneously — they combine via the most restrictive floor

**Why this is not brittle:**
- The floor values are data (in a configuration file), not hard-coded rules in the fusion logic
- The conditions are simple predicates over evidence node existence and properties
- Adding a new safety override does not require changing the fusion algorithm — it requires adding a new entry to the constraints configuration
- The trust registry (list of recognized CAs) is independently versioned and auditable
- If a CA becomes compromised, removing it from the trust registry is a data update, not a code change
- The floor values themselves can be tuned based on empirical data about real-world C2PA reliability

**What safety overrides do NOT do:**
- They do not short-circuit the fusion algorithm. The full Bayesian computation still runs — the override clamps the output, it doesn't skip computation.
- They do not hide evidence. If the raw posterior was 0.30 and the floor raises it to 0.70, the explainability panel shows BOTH numbers: "Raw assessment: 30% → Adjusted to 70% due to hardware attestation floor."
- They do not create false precision. The floor is a floor, not a target. The posterior can be higher than the floor if other evidence supports authenticity.

### 3.5 Conflict Resolution UX

Every conflict flag in the output includes:

1. **Conflict severity label** (Critical/High/Moderate/Low)
2. **Conflicting signal summary:** "Signal A (C2PA Verification) indicates authentic. Signal B (Frequency Domain Analysis) indicates AI-generated."
3. **Why these signals matter:** Each signal's tier, evidence class, and historical accuracy range
4. **Possible explanations:** 2-3 hypotheses that could explain the conflict, ranked by plausibility
5. **Suggested next steps:** Concrete actions the user can take
6. **Confidence impact:** How this conflict affected the final confidence level

Example output for a HIGH conflict:
```
⚠ CONFLICT: Valid provenance indicates authentic, but detection suggests AI-generated.

C2PA Verification (Tier 1, Provenance class): Valid certificate from Adobe. 
Content signed by Photoshop 2026. Confidence: 95%.

Frequency Domain Analysis (Tier 2, Image Forensics class): DCT anomalies consistent 
with GAN-based generation detected. Confidence: 82%.

WHY THESE DISAGREE:
• The C2PA certificate is cryptographically valid — the signing is not in dispute
• But C2PA verifies WHO signed, not WHAT they signed. A malicious actor with 
  a valid certificate could sign AI-generated content
• The frequency analysis could be a false positive — post-processing filters 
  can create GAN-like artifacts in authentic images

NEXT STEPS:
• Verify the signer's identity independently (contact the claimed creator)
• Check if this content appears elsewhere with different provenance claims
• Run additional forensics (ELA, noise pattern analysis) for corroboration

Confidence capped at "Moderate" due to unresolved conflict.
```

---

## 4. Uncertainty Propagation

### 4.1 Uncertainty Sources

Uncertainty enters the fusion engine at multiple levels:

| Source | Where It Originates | How It's Quantified |
|--------|--------------------|---------------------|
| **Measurement uncertainty** | Individual analyzer outputs | `lr_variance` field on each Evidence node — each analyzer reports its own uncertainty |
| **Calibration uncertainty** | Detector calibration to likelihood ratios | Expected Calibration Error (ECE) measured offline; included in the `lr_variance` |
| **Model degradation uncertainty** | Accuracy drift as generators evolve | `detector_durability` field; older models have increased effective variance |
| **Missing evidence uncertainty** | Evidence that couldn't be gathered (CORS, offline, browser limits) | Propagated as a confidence ceiling cap rather than a variance term |
| **Dependency uncertainty** | Correlation between signals imperfectly estimated | The correlation matrix has its own uncertainty; correlated pairs use conservative (higher) ρ estimates |

### 4.2 Propagation Path

The propagation follows this chain:

```
Individual signal uncertainty (lr_variance)
        │
        ▼
    ┌──────────────┐
    │ DEPENDENCY    │  Combine variances with correlation adjustments
    │ ADJUSTMENT    │  var_combined = Σ var_i + 2*Σ ρ_jk*sqrt(var_j*var_k)
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │ BAYESIAN      │  var(log_posterior_odds) from combined variances
    │ POSTERIOR VAR │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │ CREDIBLE      │  85% credible interval from posterior Beta distribution
    │ INTERVAL      │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │ LEVEL IMPACT  │  Wider intervals → potential downgrade in discrete level
    │               │  (very wide intervals cap at "Low Confidence")
    └──────────────┘
```

### 4.3 Credible Interval Computation

The 85% credible interval is the primary uncertainty communication for consumer users. It is computed as:

1. The posterior Beta distribution: Beta(α, β) where:
   - α = prior_pseudo_count_AI + effective_AI_signal_strength
   - β = prior_pseudo_count_authentic + effective_authentic_signal_strength
   
   Where pseudo-counts encode the prior strength and effective signal strength comes from the accumulated likelihood ratios.

2. The 85% highest-density interval (HDI) is the narrowest interval containing 85% of the posterior probability mass.

3. For computational simplicity, the interval is approximated using the normal distribution on the log-odds scale, then transformed back to probability space.

**Why 85% rather than 95%:** The TRUST_ENGINE_SPEC recommends 85% credible intervals. A 95% interval conveys an impression of near-certainty about the uncertainty ("we are 95% sure the true value is in this range") that is itself misleading when signals are sparse. An 85% interval is wide enough to capture reasonable variation but narrow enough to be useful for decision-making.

### 4.4 Missing Evidence Ceiling Effect

Missing evidence does NOT widen the credible interval (which tracks uncertainty in what WAS measured). Instead, missing evidence imposes a **confidence ceiling** — a maximum achievable discrete confidence level regardless of how strong the available evidence is.

**Ceiling computation:**

The ceiling is the highest discrete confidence level that can be reached given the evidence that COULD have been gathered but was NOT.

| Missing Evidence Scenario | Ceiling | Rationale |
|--------------------------|---------|-----------|
| All 6 classes have coverage, Tier 1 signals present | No ceiling (Very High achievable) | Full evidence spectrum available |
| 4-5 classes covered, no Tier 1 | "High Confidence" max | Good coverage but no ground-truth anchoring |
| 2-3 classes covered, Tier 2 signals present | "Moderate Confidence" max | Adequate coverage but significant gaps |
| Only 1 class covered | "Low Confidence" max | Can't corroborate across classes |
| Only Tier 4 signals, any class count | "Low Confidence" max | All signals are weak — no amount of them creates confidence |
| Browser missing key capability (e.g., Firefox no tabCapture) | "Moderate Confidence" max for video/audio assessments | Key evidence class unavailable due to platform |
| Offline operation, cached data stale > 7 days | Downgrade by one level | Stale reputation data reduces reliability |
| > 50% of images on page are cross-origin, CORS-blocked | "Moderate Confidence" max | Majority of image evidence unavailable |

**The ceiling is applied AFTER the full fusion pipeline.** The pipeline computes the discrete level normally, then the ceiling clamps it if the computed level exceeds what's achievable with the available evidence.

**Ceiling is additive across missing-evidence categories:**

```
effective_ceiling = min(
    evidence_class_ceiling,
    tier_quality_ceiling,
    browser_capability_ceiling,
    offline_staleness_ceiling,
    cors_shadow_dom_ceiling
)
```

The most restrictive ceiling wins. A page with good provenance but 80% of images behind CORS on Firefox gets capped by the browser capability ceiling, not given a pass because C2PA worked on the one same-origin image.

---

## 5. Confidence Model

### 5.1 The 8 Independent Dimensions

Each dimension is computed independently and tracked separately. They are displayed to the user as orthogonal facets, not collapsed into a single score.

#### Dimension 1: Confidence
**What it measures:** How strongly the evidence supports a specific conclusion (authentic or AI-generated).

**Computation:** Bayesian posterior probability P(authentic | evidence) or P(AI | evidence), computed by the fusion pipeline.

**Range:** 0.0 to 1.0, mapped to the 6 discrete levels (see Section 5.2).

**Display:** The discrete confidence level is the headline. The posterior probability is available in the explainability panel for professional users.

#### Dimension 2: Trust
**What it measures:** How reliable the evidence SOURCES are, independent of what the evidence says.

**Computation:** Aggregated source reputation scores:
- For each provenance source (C2PA signer, domain, publisher), a reputation score [0,1] is maintained
- Reputation is computed from: historical accuracy, recency of verification, CA audit status, community trust signals
- Unknown sources get a neutral starting score (0.5) — neither trusted nor distrusted
- The trust dimension is the weighted average of all source reputations contributing to the current assessment

**Range:** 0.0 (all sources unknown/untrustworthy) to 1.0 (all sources verified, long positive history).

**Display:** "Sources: X of Y verified. Publisher: [Trusted/Unknown/Unverified]."

#### Dimension 3: Certainty
**What it measures:** How much the evidence signals AGREE with each other.

**Computation:** Ensemble agreement metric:
- For each pair of active nodes with definite direction, compute whether they agree (same sign) or disagree (opposite sign)
- Certainty = (pairs_agreeing / total_pairs) weighted by the geometric mean of their confidences
- Adjusted for tier: agreement between two Tier 1 signals carries more weight than agreement between two Tier 4 signals
- Range: 0.0 (all signals contradict) to 1.0 (all signals converge)

**Display:** "Signal agreement: X/Y signals concur." If conflict exists, "2 signals disagree (see conflict details)."

#### Dimension 4: Coverage
**What it measures:** What percentage of APPLICABLE evidence types were successfully gathered.

**Computation:**
- Determine the set of evidence types applicable to the content (e.g., a text-only page cannot have image forensics — those are not "missing")
- Count how many of those applicable types produced evidence nodes
- Coverage = gathered_applicable_types / total_applicable_types
- The "absence of provenance" flag counts as coverage (it IS the evidence for provenance when no C2PA exists)

**Range:** 0.0 (no applicable evidence gathered) to 1.0 (all applicable types gathered).

**Display:** "Evidence gathered: X/Y applicable types."

#### Dimension 5: Quality
**What it measures:** What TIER of evidence was available — the strength of the best signals.

**Computation:** Weighted tier score:
```
quality = (count_tier1 * 4 + count_tier2 * 3 + count_tier3 * 2 + count_tier4 * 1) / (total_signals * 4)
```
Normalized to [0,1] where 1.0 means all signals are Tier 1.

**Range:** 0.25 (all Tier 4) to 1.0 (all Tier 1).

**Display:** "Best available signal: [Tier N — Name]." Tier distribution shown as a breakdown.

#### Dimension 6: Browser Completeness
**What it measures:** What evidence was UNAVAILABLE due to browser limitations.

**Computation:** From the browser capability matrix (BROWSER_CAPABILITY_ASSESSMENT.md Section 5.1):
- Determine which evidence types are impossible on the current browser
- Count how many of those would have been applicable to this content
- Browser completeness = 1.0 - (unavailable_due_to_browser / total_applicable_types)

**Range:** 0.0 (all evidence types blocked by browser) to 1.0 (full capabilities).

**Display:** List of specific evidence types unavailable: "Video frame analysis unavailable in Firefox. Image analysis reduced (no WebGPU)."

#### Dimension 7: Offline Completeness
**What it measures:** What evidence was UNAVAILABLE due to operating offline.

**Computation:** When in offline mode:
- Cloud-dependent evidence types (real-time reputation, fact-check, reverse image search) are unavailable
- Locally-cached versions may be used with a staleness penalty
- Offline completeness = 1.0 - (unavailable_due_to_offline / total_applicable_types)

**Range:** 0.0 (all cloud-dependent evidence unavailable, no caches) to 1.0 (online, or all applicable evidence is local-only).

**Display:** "Operating offline. X evidence types unavailable. Cached reputation data is Y days old."

#### Dimension 8: Evidence Completeness
**What it measures:** What evidence was UNAVAILABLE due to CORS, Shadow DOM, platform stripping, or other page-structure barriers.

**Computation:**
- Count evidence nodes with non-null `unavailability_reason`
- Evidence completeness = 1.0 - (unavailable_due_to_page_barriers / total_applicable_types)

**Range:** 0.0 (all evidence blocked by page structure) to 1.0 (no barriers).

**Display:** "X images could not be analyzed (CORS). Y elements inaccessible (closed Shadow DOM). Metadata stripped by platform."

### 5.2 The 6 Discrete Confidence Levels

The spec mandates 6 discrete levels — never continuous percentages. These are:

| Level | Label | User-Facing Description |
|-------|-------|------------------------|
| L6 | **Very High Confidence** | "Multiple strong, independent signals consistently support this conclusion." |
| L5 | **High Confidence** | "Strong evidence supports this conclusion with minor uncertainty." |
| L4 | **Moderate Confidence** | "Evidence supports this conclusion but is incomplete or partially mixed." |
| L3 | **Low Confidence** | "Limited evidence available; this conclusion is tentative." |
| L2 | **Very Low Confidence** | "Evidence is weak, sparse, or contradictory." |
| L1 | **Insufficient Evidence** | "Not enough evidence to reach any conclusion." |

### 5.3 Mapping from Dimensions to Discrete Levels

The mapping uses a **decision matrix** with the Bayesian posterior as the primary axis and the remaining 7 dimensions as constraint-based downgrades.

**Step 1: Raw Posterior Band**

Map the Bayesian posterior P (the posterior for whichever direction has more support — authentic or AI) to a raw level:

| Posterior Range | Raw Level |
|-----------------|-----------|
| P > 0.90 | L6 (Very High) |
| P > 0.75 | L5 (High) |
| P > 0.60 | L4 (Moderate) |
| P > 0.40 | L3 (Low) |
| P > 0.20 | L2 (Very Low) |
| P ≤ 0.20 | L1 (Insufficient) |

**Step 2: Constraint-Based Downgrades**

Apply the following constraints in order. Each can only lower the level, never raise it. The most restrictive downgrade wins.

**Constraint A: Signal Count**
| Condition | Downgrade |
|-----------|-----------|
| Fewer than 2 active evidence nodes | Cap at L3 (Low Confidence) |
| Only 1 active evidence node | Cap at L2 (Very Low Confidence) |
| 0 active evidence nodes | Force L1 (Insufficient Evidence) |

**Constraint B: Evidence Class Diversity**
| Condition | Downgrade |
|-----------|-----------|
| "Very High Confidence" (L6) but evidence from < 3 classes | Downgrade to L5 (High) |
| "High Confidence" (L5) but evidence from < 2 classes | Downgrade to L4 (Moderate) |
| "Moderate Confidence" (L4) but evidence from < 2 classes | Downgrade to L3 (Low) |

**Constraint C: Tier Quality Floor**
| Condition | Downgrade |
|-----------|-----------|
| All active signals are Tier 4 only | Cap at L3 (Low Confidence) |
| Best active signal is Tier 3, no Tier 1 or 2 | Cap at L4 (Moderate Confidence) |
| Best active signal is Tier 2, no Tier 1 | Cap at L5 (High Confidence) |

**Constraint D: Conflict Severity**
| Condition | Downgrade |
|-----------|-----------|
| CRITICAL conflict detected | Cap at L3 (Low Confidence) |
| HIGH conflict detected | Cap at L4 (Moderate Confidence) |
| MODERATE conflict detected | No cap (informational only) |

**Constraint E: Coverage Floor**
| Condition | Downgrade |
|-----------|-----------|
| Coverage < 30% of applicable evidence types | Cap at L3 (Low Confidence) |
| Coverage < 50% of applicable evidence types | Cap at L4 (Moderate Confidence) |
| Coverage < 70% of applicable evidence types | Cap at L5 (High Confidence) |

**Constraint F: Trust Floor**
| Condition | Downgrade |
|-----------|-----------|
| All contributing sources have "Unknown" trust status | Cap at L5 (High Confidence) |
| Any contributing source has "Compromised" or "Blacklisted" status | Cap at L4 (Moderate Confidence) |

**Constraint G: Ceiling from Dimensions 6-8 (Browser/Offline/Evidence Completeness)**

The minimum of the three completeness dimensions imposes an overall ceiling:
| Minimum Completeness Score | Ceiling |
|---------------------------|---------|
| < 0.30 | Cap at L2 (Very Low Confidence) |
| < 0.50 | Cap at L3 (Low Confidence) |
| < 0.75 | Cap at L4 (Moderate Confidence) |
| < 0.90 | Cap at L5 (High Confidence) |
| ≥ 0.90 | No ceiling from completeness |

**Step 3: Final Level Assignment**

```
final_level = raw_level
for each constraint in [A, B, C, D, E, F, G]:
    final_level = min(final_level, constraint_cap)
```

### 5.4 Worked Examples

**Example 1: Ideal Scenario**
- Evidence: C2PA valid hardware attestation (Tier 1, Provenance) + EXIF consistent (Tier 3, Provenance) + DOM text extraction + text style analysis (Tier 4, Text) + cached domain reputation (Tier 4, Reputation)
- Posterior: P(authentic) = 0.94 → Raw L6
- Constraint A: 4 signals → no downgrade
- Constraint B: 2 classes (Provenance, Reputation) — not 3 → L6 downgraded to L5
- Wait — actually we need to carefully count. EXIF and C2PA are both Provenance class? Let me reconsider. The 6 evidence classes are: PROVENANCE, IMAGE_FORENSICS, TEXT_ANALYSIS, VIDEO_FORENSICS, AUDIO_FORENSICS, REPUTATION_CONTEXTUAL. EXIF and C2PA are both PROVENANCE. Text style is TEXT_ANALYSIS. Domain reputation is REPUTATION_CONTEXTUAL. So 3 classes. → Constraint B satisfied. L6 maintained.
- Constraint C: Tier 1 present → no downgrade
- Constraint D: No conflict → no downgrade
- Constraint E: Coverage high → no downgrade
- Constraints F, G: All good
- **Final: L6 — Very High Confidence**

**Example 2: Only Weak Signals**
- Evidence: Text style analysis (Tier 4, Text), text burstiness (Tier 4, Text), domain reputation unknown (Tier 4, Reputation)
- Posterior: P(AI) = 0.68 → Raw L4
- Constraint A: 3 signals → no downgrade
- Constraint B: 2 classes → L4 maintained (need 2 for Moderate, have 2)
- Constraint C: All Tier 4 → Cap at L3
- Constraint D: No conflict → no downgrade
- Constraint E: Coverage partial → L4 cap
- Final: min(L4, L3, L4) = **L3 — Low Confidence**

**Example 3: Single Strong Signal**
- Evidence: C2PA valid with recognized CA (Tier 1, Provenance) — only signal
- Posterior: P(authentic) = 0.88 → Raw L5
- Constraint A: Only 1 signal → Cap at L2
- Constraint B: 1 class → If L5 but < 2 classes → downgrade to L4; then L2 cap wins
- **Final: L2 — Very Low Confidence**

Note: This is correct behavior. The spec says C2PA can function as a standalone positive signal for AUTHENTICITY, but the confidence LEVEL must reflect that a single signal cannot produce high confidence in the discrete level system. A single C2PA verification tells you "this content was signed" — but without corroboration from other classes, you can't be highly confident about authenticity. The user sees "Very Low Confidence: Only one signal available. C2PA verification confirms signing, but no corroborating evidence was found."

**Example 4: Conflicting Strong Signals**
- Evidence: C2PA valid (Tier 1, Provenance, direction authentic) + AI generator fingerprint 87% (Tier 2, Image Forensics, direction AI)
- Posterior: P(authentic) = 0.52 → Raw L3
- Conflict: HIGH conflict (Tier 1 vs Tier 2, both > 0.65 confidence)
- Constraint D: HIGH conflict → Cap at L4
- Final: min(L3, L4) = **L3 — Low Confidence** (conflict + near-even posterior)

The UI shows: "Low Confidence — Strong conflicting evidence. Valid provenance indicates authentic, but AI detection indicates generated. Investigation recommended."

### 5.5 Dimension Display in the UI

For consumer users (quick glance):
```
████████████████████████░░░░  HIGH CONFIDENCE
3 of 4 signals agree  |  Best signal: C2PA (Tier 1)  |  2 sources verified
```

For professional users (explainability panel):
```
CONFIDENCE: High (L5) — Strong evidence supports authenticity
  Posterior: 0.82  |  85% Credible Interval: [0.73, 0.89]

TRUST: Moderate — 1 of 2 sources verified
  • C2PA signer: Adobe Inc. — Trusted (audited CA)
  • Domain: example.com — Unknown (no history)

CERTAINTY: High — 3 of 4 signals agree
  • Agreeing: C2PA, EXIF, Domain Reputation
  • Disagreeing: Text Burstiness (weak, outweighed)

COVERAGE: 67% — 4 of 6 applicable evidence types gathered
  • Gathered: Provenance (2 signals), Text (1), Reputation (1)
  • Not applicable: Image Forensics (no images on page), Video (no video)

QUALITY: Good — Best signal Tier 1
  • Tier 1: 1 signal (C2PA)  |  Tier 3: 1 signal (EXIF)  |  Tier 4: 2 signals

BROWSER: Chrome — Full capabilities available

OFFLINE: Online — All evidence types available

EVIDENCE GAPS: None — No CORS, Shadow DOM, or platform stripping detected
```

---

## 6. Edge Cases

### 6.1 All Signals Are Tier 4 — No Tier 1-3 Evidence Available

**Scenario:** The page has only text (no images for forensics), no provenance metadata, no reputation data cached. The only available signals are text burstiness, style consistency, and repetition pattern analysis — all Tier 4.

**Fusion behavior:**
- The Bayesian accumulator computes normally. The posterior may drift in one direction if the Tier 4 signals agree, but the variance will be high (Tier 4 signals have high uncertainty).
- Constraint C (Tier Quality Floor) activates: all Tier 4 → cap at L3 (Low Confidence).
- The credible interval will be wide because individual signal variances are large.
- Coverage dimension will be low (only TEXT_ANALYSIS class with applicable evidence; other classes have no evidence because no analysable content exists for them).

**User sees:** "Low Confidence — Limited to weak text analysis signals. No provenance, image forensics, or reputation data was available for this content."

**Context note:** If the page genuinely has no analysable images, video, or audio, the Coverage dimension should NOT penalize for that — Coverage measures applicable types that WERE gathered vs. applicable types that COULD have been gathered but weren't. A text-only page with text analysis has high Coverage, but low Quality because all signals are Tier 4. Both dimensions tell the user the full story.

### 6.2 Only 1 Signal Available

**Scenario:** A single image on a page with no surrounding text. Only one signal fires (e.g., EXIF metadata is present, but C2PA absent, no domain reputation, no text to analyze).

**Fusion behavior:**
- Constraint A: 1 active evidence node → Cap at L2 (Very Low Confidence).
- Even if that single signal is Tier 1 (C2PA hardware attestation), the cap applies.
- The spec's statement that "Tier 1 can function alone" is respected in the Bayesian posterior computation — the posterior will be high — but the discrete confidence LEVEL is capped per the constraint that confidence assessment requires corroboration.

**User sees:** "Very Low Confidence — Only one signal available (EXIF Metadata). Multiple independent signals are required for a reliable assessment."

**Single C2PA exception rationale:** The C2PA signal being strong (Tier 1) is reflected in the posterior being high and the credible interval being narrow. The L2 cap reflects the meta-uncertainty about single-signal assessments: "The one signal we have says authentic, but with only one data point, we can't be confident in the overall assessment."

### 6.3 Every Signal Disagrees With Every Other Signal

**Scenario:** 4 signals, all pointing in different directions or contradicting each other in pairs. No two signals agree.

**Fusion behavior:**
- The Bayesian accumulator processes all signals. The posterior will be near 0.5 (maximally uncertain) because equal evidence pushes in both directions.
- Conflict detector: Multiple CRITICAL or HIGH conflict pairs detected.
- Constraint D: CRITICAL conflict → Cap at L3 (Low Confidence).
- Certainty dimension: 0.0 (complete disagreement).
- The credible interval will span nearly [0,1] — maximum uncertainty.

**User sees:** "Low Confidence — All evidence signals disagree. No consensus. Manual investigation required. Possible explanations: content has been heavily edited, multiple AI generation techniques were used, or detection signals are producing false positives."

**This is the correct behavior.** A page where every signal disagrees SHOULD produce a low-confidence assessment, not a split-the-difference "52%." The conflict itself is the primary information.

### 6.4 Evidence Arrives That Contradicts an Already-Displayed Assessment

**Scenario:** The user sees "High Confidence — Likely Authentic" based on Wave 1 + Wave 2 evidence. Then they invoke Tier 3 deep analysis. The AI generator fingerprint model returns a strong AI signal (confidence 0.85).

**Fusion behavior:**
- The new evidence node is added to the graph.
- A `contradicts` edge is created between the new node and the existing C2PA/exif nodes.
- The conflict detector identifies a HIGH conflict.
- The Bayesian posterior shifts toward lower authenticity.
- Constraint D: HIGH conflict → Cap at L4 (Moderate Confidence).
- The assessment transitions: L5 "High Confidence" → L4 "Moderate Confidence."

**UI behavior:**
- The confidence level indicator animates a transition (NOT an abrupt flash — a 300ms eased transition).
- A notification appears: "Confidence updated. New deep analysis evidence contradicts previous assessment."
- The conflict detail is expandable: "C2PA indicated authentic. AI generator fingerprint detected DALL-E 4 artifacts. These signals disagree."
- Previous assessment is preserved in the timeline: "Initial assessment (2s ago): High Confidence → Updated (now): Moderate Confidence."

### 6.5 User Disables a Signal Class Mid-Analysis

**Scenario:** User is on a page with image + text analysis. They go to settings and disable the "Text Analysis" signal class entirely.

**Fusion behavior:**
- All Evidence nodes with `evidence_class = TEXT_ANALYSIS` are marked `excluded = true` in the graph's filter context.
- The fusion engine re-computes the assessment excluding those nodes.
- The `excluded` nodes remain in the graph — they can be re-included if the user re-enables the class.
- Coverage and Quality dimensions update to reflect the reduced signal set.
- If the excluded class was the only class providing evidence, the assessment may drop to L1 or L2.

**UI behavior:**
- "Assessment updated. Text Analysis signals excluded per your settings."
- A small indicator: "2 signals hidden (Text Analysis disabled)."
- The explainability panel shows which signals are excluded and why.

### 6.6 Browser Goes Offline Mid-Analysis

**Scenario:** User initiates analysis while online. Wave 1 and Wave 2 complete. During Wave 3, the browser goes offline. Cloud-dependent evidence (real-time domain reputation refresh, fact-check lookup) is in-flight.

**Fusion behavior:**
1. **In-flight cloud evidence is cancelled.** Any pending cloud requests are aborted. The evidence nodes for those signals are created with `unavailability_reason = OFFLINE`.
2. **Local-only evidence gathered so far is preserved.** The graph retains all nodes gathered before the offline transition.
3. **Cached cloud evidence is used as fallback.** If a domain reputation cache exists (even if stale), it is used as a proxy with reduced weight (the `confidence` field on the cached reputation node is lowered based on staleness).
4. **Offline Completeness dimension (Dimension 7) updates.** Reflects which evidence types are now unavailable.
5. **The ceiling recomputation may lower the confidence level.** If cloud-dependent evidence was critical to the assessment, the level may drop.

**User sees:**
- "Assessment incomplete — you are offline. 3 cloud-dependent signals unavailable."
- If cached data is used: "Using cached domain reputation (updated 5 days ago). Accuracy may be reduced."
- "Re-analyze when back online for complete assessment."

**On reconnection:**
- When the browser transitions back online, the extension detects the state change.
- If the assessment is still active, cloud evidence is automatically refreshed.
- A prompt appears: "Back online. Update this assessment with fresh cloud data?"

### 6.7 Evidence Class Weight Capping in Practice

**Scenario:** A page has 5 provenance signals (C2PA, EXIF, IPTC, XMP, ICC profile) and 1 reputation signal (domain reputation). Without capping, provenance would dominate.

**Fusion behavior (Step 1 — Dependency Resolver):**
1. Individual signal cap at 40%: No single provenance signal exceeds 40% of total log-LR sum — all pass.
2. Class cap at 60%: The PROVENANCE class has 5 signals contributing, say, 85% of total weight. This triggers the 60% cap.
3. All provenance signals are scaled down proportionally until PROVENANCE = 60% of total.
4. The REPUTATION signal now accounts for 40% of the remaining weight — still within its class cap.
5. The adjusted log-LR values are passed to the Bayesian accumulator.

**This prevents the scenario where a content type with many analysable images (e.g., a photo gallery) receives artificially high confidence simply because there were more images to analyze.** The class cap ensures diversity of evidence classes matters as much as volume.

### 6.8 Very High Posterior but Tier Quality Constraint Caps It

**Scenario:** Three Tier 4 text signals all strongly agree (direction = AI, each with 85% signal confidence). The Bayesian posterior reaches P(AI) = 0.92. Raw level = L6.

**Constraint C fires:** All signals are Tier 4 → Cap at L3 (Low Confidence).

**Result:** "Low Confidence — Evidence consistently suggests AI generation, but all available evidence is from weak signal types. Strong confirmation requires provenance or forensic analysis."

**This is correct.** The system is saying: "The weak signals we have all point the same way, but they're still weak signals. We can't be highly confident based on weak signals alone, even if they agree." Agreement among unreliable witnesses doesn't create reliability.

---

## 7. Architecture Decision Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Primary fusion method** | Bayesian evidence accumulation with log-odds | Mathematically correct for independent evidence; supports explainability via decomposed contributions; handles missing signals gracefully |
| **Graph structure** | DAG with 5 edge types | Captures all evidence relationships (support, contradiction, derivation, correlation, versioning) without cycles |
| **Dependency handling** | Pre-computed correlation matrix + weight capping | Correlation matrix is data, not code — updatable without changing the fusion engine. Weight caps enforce the spec's 40%/60% limits |
| **Conflict detection** | Pairwise comparison with tier-based escalation | Deterministic, explainable. Each conflict pair has a severity score. The 4-tier model surfaces what matters, suppresses noise |
| **Safety overrides** | Posterior floor constraints (config data) | Not brittle hard-coded rules. Floor values are parameterized. Trust registry is independently auditable. Override rationale is transparent |
| **Uncertainty propagation** | Variance accumulation through Bayesian pipeline | Each signal reports its own uncertainty. Variances combine with correlation adjustments. Propagated to credible interval |
| **Missing evidence** | Confidence ceiling model | Missing evidence caps achievable confidence level. Multiple ceiling sources combine via minimum. Does not widen credible interval (avoids "we're 30% sure but the range is 20-80%" confusion) |
| **Confidence levels** | 6 discrete levels with constraint-based mapping | Bayesian posterior determines raw level. 7 constraints can only downgrade. Never produces a continuous percentage that invites false precision |
| **Confidence dimensions** | 8 independent, always-computed dimensions | Separates what's being measured (confidence) from why it might be limited (trust, certainty, coverage, quality, browser, offline, evidence gaps). Prevents a single number from hiding critical context |
| **Re-computation timing** | Immediate on new evidence, 200ms debounce for waves | Balances UI stability with responsiveness. Wave 1 batch-computed, Wave 2 smooth refinement, Wave 3 immediate. Debounce worst-case latency is imperceptible |
| **Progressive refinement** | 4 states: Initial → Preliminary → Refined → Deep | Aligns with the timing model defined in BROWSER_CAPABILITY_ASSESSMENT. Each state has a known evidence set and achievable confidence ceiling |
| **Edge case: single signal** | Capped at L2 (Very Low) per constraint A | Even Tier 1 signals require corroboration for confident assessment. The signal's strength is reflected in the posterior — the level cap reflects meta-uncertainty about single-source assessments |
| **Edge case: all Tier 4** | Capped at L3 (Low) per constraint C | Agreement among weak signals doesn't create reliability. Users must understand the evidence quality limitation |
| **Model degradation** | `detector_durability` field + per-signal recalibration | Detection models degrade over time. The fusion engine weights by durability, not snapshot accuracy. Calibration updates are data changes, not code changes |
| **Browser/offline gaps** | Dimensions 6-8 + confidence ceiling | Gaps are tracked independently and surfaced explicitly. Users know WHY confidence is limited, not just THAT it's limited |

---

*End of Fusion Engine Architecture. This document defines the graph structure, fusion algorithm, conflict detection, uncertainty propagation, confidence model, and edge case handling for Sentinel's Trust Engine. Implementation details (data structures, class design, message passing, serialization) are separate concerns.*
