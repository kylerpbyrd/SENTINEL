# TRUST_ENGINE_SPEC.md — Trust Engine Product Reasoning Specification

> **Document Type:** Product Reasoning Specification (NOT implementation)
> **Phase:** 1 — Pre-Architecture
> **Owner:** Lead Architect (consolidated from 7 specialized agents)
> **Status:** Draft — Awaiting Stakeholder Review
> **Depends On:** Phase 0 Discovery Report
> **Prerequisite for:** ARCHITECTURE.md, ADR-003 (Trust Engine Architecture)

---

## Purpose

This document defines WHAT the Sentinel Trust Engine does, WHY it works the way it does, and WHAT guarantees it provides. It does NOT define HOW it is implemented.

Every architectural decision about Sentinel's reasoning system must trace back to the principles, rules, and constraints established in this specification.

### What This Document Answers

1. What is the purpose of the Trust Engine?
2. What evidence types exist?
3. Which evidence is strongest? (Confidence hierarchy)
4. Which evidence should NEVER be used by itself?
5. How should evidence be combined?
6. How should uncertainty be represented?
7. How should conflicting evidence be handled?
8. How should explainability work?
9. How should users interact with the results?
10. How can the Trust Engine evolve without architectural redesign?

### What This Document Excludes

- Programming languages, frameworks, APIs, databases, or implementation details
- UI mockups, pixel layouts, or color schemes
- Browser extension manifest structure
- Specific model architectures or training procedures
- Deployment infrastructure or cloud provider selection

---

## Section 1: Purpose of the Trust Engine

### 1.1 What the Trust Engine Is

The Trust Engine is the core reasoning system of Sentinel. It ingests evidence signals from multiple independent sources, fuses them into calibrated confidence assessments, and produces explainable output that users can understand, evaluate, and act upon.

### 1.2 What the Trust Engine Is NOT

| The Trust Engine is NOT | Why |
|--------------------------|-----|
| An AI detector | Does not output binary "AI" or "human" classifications |
| A fact-checker | Does not determine truth or falsehood of claims |
| A content filter | Does not hide, blur, or block content (that is a separate UX layer) |
| A reputation system | Source reputation is one input among many, not the product |
| A decision-maker | Provides assessments; the user makes decisions |

### 1.3 Core Principles

1. **Evidence over certainty.** The Trust Engine never determines truth. It communicates confidence based on available evidence.

2. **Transparency over authority.** Every assessment must show its work. Users must understand WHY a confidence level was reached.

3. **Honesty over precision.** It is better to say "insufficient evidence" than to guess with false precision. Confidence levels must be calibrated — "85% confidence" must actually mean ~85% accuracy.

4. **User sovereignty.** The user remains in control. The Trust Engine provides information; the user decides what to do with it.

5. **Privacy by default.** Evidence gathering favors local processing. Cloud analysis requires explicit opt-in. Content never leaves the device without the user knowing and consenting.

### 1.4 Who the Trust Engine Serves

**Primary (Phase 2 MVP):** Journalists and educators — professional content evaluators who need evidence they can cite in consequential decisions.

**Secondary (Phase 3):** Content marketers, enterprise trust & safety teams, developers integrating the Trust Engine API.

**Future (Phase 4+):** General consumers, platform partners.

---

## Section 2: Evidence Types

### 2.1 Complete Evidence Taxonomy

The Trust Engine consumes evidence from these categories. Each evidence type has a defined role, weight, and constraint set.

#### Provenance Evidence (Cryptographic & Metadata-Based)

| Evidence Type | What It Provides | Acquisition |
|---------------|-----------------|-------------|
| **C2PA manifest verification** | Cryptographically signed manifest confirming content origin, creation tool, edit history, and publisher identity. Verification is deterministic — the signature validates or it doesn't. | Local (WASM via `c2pa-js`). Same-origin images or CORS-enabled cross-origin. |
| **Hardware-attested provenance** | C2PA signed at capture moment inside a hardware secure element (Leica M11-P, Sony Alpha). Key never leaves the chip. | Local (WASM). Same constraints as C2PA. |
| **C2PA crJSON (text provenance)** | JSON-based C2PA manifest for text content. New standard (v2.4), minimal adoption as of July 2026. | Local (parsing). |
| **EXIF metadata** | Camera make, model, timestamp, GPS, capture settings. Informative when present but trivially fabricated. | Local (binary parsing). Same-origin or CORS-enabled. |
| **IPTC/IIM metadata** | Creator, copyright, caption, credit line. Professional photography standard. | Local. Same constraints as EXIF. |
| **XMP metadata** | Adobe-standard metadata container. Edit history, software used. | Local. Same constraints. |
| **IPTC Digital Source Type** | Metadata field explicitly declaring content type: captured by camera, generated by AI, etc. | Local. Currently very rare. |

#### Forensic Evidence (Content-Based Analysis)

| Evidence Type | What It Provides | Acquisition |
|---------------|-----------------|-------------|
| **Image frequency-domain artifact analysis** | Detects VAE decoder artifacts, GAN fingerprints, and frequency anomalies in AI-generated images. 75-90% accuracy on known generators. | Local (ONNX/WASM). Requires pixel access (same-origin only). |
| **Noise pattern analysis** | Identifies unnatural noise patterns inconsistent with camera sensor noise. | Local (WASM). Same-origin pixel access required. |
| **Error Level Analysis (ELA)** | Detects splicing and editing by analyzing compression inconsistency. | Local (JS/WASM). Same-origin pixel access. |
| **AI generator fingerprint detection** | Lightweight classifier models detect telltale fingerprints from specific generators (DALL-E, Midjourney, SD). | Local (ONNX, ~5-20 MB models). Same-origin pixel access. |
| **Known synthetic media hash matching** | Hash-level match against database of known AI-generated content. Near-deterministic when positive. | Local (periodic hash list download) or cloud (real-time lookup). |
| **Face-swap detection** | Xception-based analysis + temporal consistency + physiological signals (blinking, pulse). 80-90% on face-swaps. | Local (ONNX). Requires tabCapture for video (Chrome/Edge only). |
| **Audio vocoder artifact detection** | Detects neural vocoder artifacts in raw, uncompressed audio. 80-90% on source audio. | Local (ONNX, ~10-30 MB models). Requires tabCapture or media element source. |
| **Text burstiness/perplexity analysis** | Zero-shot analysis of statistical writing patterns. Model-agnostic but converges to random as LLMs improve. | Local (ONNX small model or pure JS statistics). |

#### Watermark Detection Evidence

| Evidence Type | What It Provides | Acquisition |
|---------------|-----------------|-------------|
| **DALL-E visible watermark** | Visible color band in bottom-right corner. Binary signal. | Local (pixel inspection). Same-origin. |
| **Midjourney invisible watermark** | Frequency-domain watermark. Detectable with known algorithm. | Local. Same-origin pixel access. |
| **Adobe Firefly Content Credentials** | C2PA manifests on Firefly outputs. | Local (C2PA verification). |
| **Stable Diffusion watermark** | Multiple approaches (SDXL-watermark, Tree-Ring). Detection method-dependent. | Local (method-dependent). Partial coverage. |
| **SynthID (Google)** | Proprietary watermark across all modalities. | **NOT locally verifiable.** Requires Google API cooperation. Currently not publicly available for third-party use. |

#### Reputation & Contextual Evidence

| Evidence Type | What It Provides | Acquisition |
|---------------|-----------------|-------------|
| **Domain reputation** | Historical trustworthiness of the domain publishing the content. | Local (cached list, refreshed daily) or cloud (real-time lookup by domain only). |
| **Publisher/source credibility** | Verified publisher identity, C2PA signing history, editorial track record. | Cloud lookup (domain-level only). |
| **Domain registration age** | Newly registered domains (<6 months) are disproportionately associated with disinformation. | Cloud lookup. |
| **Fact-check database lookup** | Match claims on page against known fact-check databases. | Cloud (sending claim hashes, not full text). |
| **Reverse image/audio search** | Prior occurrences on the web. Detects repurposing, identifies original source. | Cloud (content sent to search provider). |

#### Community & User Signals

| Evidence Type | What It Provides | Acquisition |
|---------------|-----------------|-------------|
| **Community/user reports** | Aggregated trust/dispute signals from other Sentinel users. | Cloud (anonymized). |
| **User trust designations** | User explicitly trusts or distrusts specific domains/sources. | Local (never transmitted). |
| **User calibration feedback** | User reports that an assessment was correct or incorrect. | Opt-in cloud submission. |

#### Cross-Modal & Derived Evidence

| Evidence Type | What It Provides | Acquisition |
|---------------|-----------------|-------------|
| **Cross-modal consistency** | Lip-sync mismatch, image-caption semantic gaps, physics violations. Architecture-independent signal. | Local (lightweight CLIP-ViT for image-text) or cloud (multi-modal models). |
| **Signal agreement score** | How many independent sources concur. Weighted by source independence. | Computed internally by Trust Engine. |
| **Signal disagreement detection** | Direct conflict between sources flagged as important meta-evidence. | Computed internally. |
| **Coverage completeness** | What percentage of applicable evidence sources returned results. | Computed internally. |

---

## Section 3: Confidence Hierarchy

Every evidence type is assigned to a tier based on its inherent reliability, durability against spoofing, and expected longevity. The Trust Engine weights evidence proportionally to its tier.

### Tier 1: Ground Truth Evidence (Dominant Weight)

These signals provide cryptographic or hardware-level guarantees. They cannot be forged without compromising cryptographic infrastructure. When present and valid, they should dominate the confidence assessment.

| Rank | Evidence Type | Weight | Can Function Alone? |
|------|--------------|--------|---------------------|
| **1** | Hardware-attested C2PA provenance (camera secure element) | Maximum | **Yes.** Absolute floor on authenticity confidence. Cannot be overridden by any detection signal except CA revocation. |
| **2** | C2PA with valid certificate chain from recognized CA | Dominant | **Yes**, but can be overridden by known synthetic signature match (Tier 2) or definitive physics violation (Tier 2). |
| **3** | Known synthetic media hash match (verified) | Near-deterministic when positive | **Yes** for AI-generation signal. Hash match is fingerprint evidence, not inference. |

**Why Tier 1 is strongest:** Verification is deterministic, not probabilistic. A signature either validates or it doesn't. Spoofing requires CA compromise or breaking SHA-256. The signal provides a complete, auditable provenance chain.

**When Tier 1 becomes unreliable:**
- Stripped manifests (social media, screenshots, format conversion)
- Compromised signer (valid certificate held by bad actor)
- Screenshot/air gap (analog capture severs all digital provenance)

### Tier 2: Strong Corroborating Evidence (High Weight)

Statistically robust but probabilistic. Provide strong evidence under known conditions but degrade outside those conditions. Should never override Tier 1 evidence.

| Rank | Evidence Type | Weight | Can Function Alone? |
|------|--------------|--------|---------------------|
| **4** | Cross-modal consistency violations | High | **No.** Requires multi-modal content. Can be manually corrected post-generation. |
| **5** | Image frequency-domain artifact analysis | Medium-High | **No.** Degrades 10-20% per new model generation. Post-processing reduces accuracy. |
| **6** | DALL-E / Midjourney / Adobe watermark detection | Medium-High | **No.** Platform-specific. Degrades under transformation. SynthID is NOT independently verifiable. |

**When Tier 2 is strongest:** On unmodified, high-resolution content from known AI generators. **When it fails:** After compression, cropping, filtering, or on content from new/unknown generators.

### Tier 3: Moderate Corroborating Evidence (Medium Weight)

Useful but volatile. Accurate under ideal conditions, degrade significantly under real-world conditions. Only meaningful in combination.

| Rank | Evidence Type | Weight | Can Function Alone? |
|------|--------------|--------|---------------------|
| **7** | Face-swap detection (Xception + temporal + physiological) | Medium | **No.** Only applies to face-swap deepfakes, not fully AI-generated video. |
| **8** | Audio vocoder artifact detection | Medium (conditional) | **No.** Only viable on raw source audio. Degrades to near-random after telephony compression. |
| **9** | IPTC/EXIF metadata consistency analysis | Medium | **No.** Metadata is trivially fabricated. Inconsistency detection is more valuable than presence. |
| **10** | Reverse image/audio search | Medium | **No.** Coverage gaps. Novel AI content has no prior web existence. Ambiguous interpretation. |

**When Tier 3 is useful:** Multiple Tier 3 signals in agreement add meaningful weight. **When it's misleading:** Any Tier 3 signal alone. Audio detection after phone compression. Metadata on content from platforms that routinely strip it.

### Tier 4: Weak / Aggregate-Only Evidence (Low Weight, Never Alone)

Volatile, narrow, or easily bypassed. Provide value ONLY in aggregate with higher-tier signals. Some must never be used at all (see Section 4).

| Rank | Evidence Type | Weight | Can Function Alone? |
|------|--------------|--------|---------------------|
| **11** | Text burstiness/perplexity analysis (zero-shot) | Low | **Never.** Converges to random as LLMs improve. Biased against non-native English. |
| **12** | Publisher/domain reputation | Low | **Never.** Lagging indicator. Domains churn rapidly. Reputable domains can be compromised. |
| **13** | Absence of provenance | Context-dependent | **Never as negative.** Default state for >95% of web content. Informational only. |
| **14** | Community/user signals | Very Low | **Never.** Easily gamed. High latency. Requires critical mass of users. |
| **15** | Domain registration age | Very Low | **Never.** Many legitimate sites use new domains. Corroborating only. |

---

## Section 4: Evidence That Must Never Be Used Alone

### 4.1 Prohibited Entirely (Never Use At All)

| Evidence Type | Reason | Authority |
|---------------|--------|-----------|
| **Model-specific text classifiers** (trained to detect specific LLM output) | Mathematically proven to converge to random as models improve. AUROC → 0.5. | Sadasivan et al. (2023), TMLR |
| **Binary "AI/Human" verdicts** for any modality | Creates false sense of precision. Misleading framing. No detector is reliable enough for binary classification. | Perkins et al. (2024), IJETHE |
| **Short text detection** (< 100 words) | Insufficient statistical structure for any method. Functionally random. | Empirical |
| **Non-native English language proficiency as a signal** | 70% false positive rate. Discriminatory. | Liang et al. (2024), Patterns |

### 4.2 Prohibited as Standalone (Must Have Corroboration)

| Evidence Type | Minimum Required Corroboration |
|---------------|-------------------------------|
| Text burstiness/perplexity | At least one signal from Tier 1 or Tier 2 |
| Face-swap detection | At least one additional signal from any tier |
| Audio vocoder artifacts | At least one provenance or metadata signal |
| IPTC/EXIF metadata alone | At least one forensic or provenance signal |
| Reverse search alone | At least one forensic or metadata signal |
| Publisher/domain reputation | At least two signals from Tier 2 or higher |
| Community/user signals | At least three signals from Tier 2 or higher |

### 4.3 The "Never Alone" Principle

The Trust Engine must never produce ANY confidence assessment above "Low Confidence" when only a single evidence signal is available. **One signal = insufficient.** The minimum for "Moderate Confidence" is 2 signals from different evidence classes. For "High Confidence" and above: 3 signals including at least one from Tier 1 or Tier 2.

---

## Section 5: How Evidence Should Be Combined

### 5.1 Recommended Approach: Bayesian Evidence Accumulation with Explicit Conflict Flagging

**Primary fusion method: Bayesian evidence accumulation.**

Each evidence signal contributes a calibrated likelihood ratio — the relative probability of observing this evidence if the content is authentic vs. AI-generated. Starting from a prior belief (base rate of AI content for the content type), each independent signal updates the posterior probability using Bayes' rule. The final output is a posterior probability with a credible interval.

**Why Bayesian:**
- **Mathematically honest.** Bayes' rule is the correct way to update beliefs given independent evidence.
- **Explainable.** Each signal's contribution to the final score is transparent. "The C2PA credential shifted our confidence from 40% to 85%. Pixel analysis shifted it from 85% to 78%."
- **Handles missing signals naturally.** A missing signal provides zero information — it simply doesn't update the posterior.
- **Supports calibrated output.** Credible intervals reflect genuine uncertainty, not arbitrary padding.

### 5.2 Why Not Other Approaches

| Approach | Rejected Because |
|----------|-----------------|
| **Weighted linear fusion** | Averages away conflict. Missing signals require awkward zero-filling. Cannot model conditional dependencies. |
| **Pure rule-based combination** | Brittle. Rules that make sense today may be wrong in 2028. Combinatorial explosion of rule interactions. Continuous human review burden. |
| **Conformal prediction alone** | Prediction sets are difficult for non-technical users to interpret. Better used as an optional wrapper for enterprise, not the primary consumer-facing output. |

### 5.3 Hybrid Architecture

The Bayesian core is supplemented by three mechanisms:

**1. Conflict Detection Layer.** After Bayesian fusion, a separate mechanism identifies when signals disagree beyond a defined threshold. Conflict is surfaced to the user — NOT smoothed into an uninformative average. (See Section 7.)

**2. Safety Override Rules.** Certain evidence creates hard floors on confidence:
- Valid hardware-attested C2PA → authenticity confidence floor (cannot be pushed below a minimum by detection signals)
- Verified known synthetic signature hash match → AI-generation confidence floor
- These are NOT "provenance always wins" — they are "when the highest-certainty signals fire, the system enforces a floor"

**3. Conformal Prediction Wrapper (Optional).** For enterprise/API users needing statistical guarantees: wraps the Bayesian output in a conformal prediction set with guaranteed coverage at a specified error rate.

### 5.4 Signal Independence Requirement

Bayesian fusion assumes conditional independence. This assumption is routinely violated — C2PA-verified images typically have consistent EXIF data; multiple frequency-domain detectors are correlated. Violating independence causes overconfidence.

**Mitigation:** The Trust Engine classifies signals into **evidence classes** and caps the ensemble weight from any single class:

| Evidence Class | Examples |
|----------------|----------|
| Cryptographic Provenance | C2PA, hardware attestation, Content Credentials |
| Metadata Analysis | EXIF, IPTC, XMP, file structure |
| Content Forensics | Frequency analysis, noise analysis, ELA, generator fingerprints |
| Behavioral/Statistical | Text perplexity, burstiness, style consistency |
| Reputation/Contextual | Domain reputation, source credibility, community signals |
| Watermark Detection | DALL-E watermark, Midjourney watermark, Adobe Firefly CC |

**Rule:** No single evidence class may contribute more than 60% of total weight. "High Confidence" requires contributions from at least 2 classes. No single signal may exceed 40% of total weight.

### 5.5 Content-Type-Specific Priors

The Trust Engine uses different Bayesian priors based on content type:

| Content Type | Default Prior (AI-generated) | Rationale |
|-------------|------------------------------|-----------|
| News photography | Lower prior | Most news photography is still human-captured |
| Social media images | Higher prior | Higher prevalence of AI-generated/synthetic images |
| Academic text | Moderate prior | Growing AI use in academic writing |
| Social media text | Higher prior | Substantial AI-generated social content |
| Professional video | Lower prior | Production video is predominantly human-created |
| User-generated video | Higher prior | Growing AI-generated short-form video |

Priors are periodically recalibrated based on measured base rates. Users may adjust priors as a trust preference.

---

## Section 6: How Uncertainty Should Be Represented

### 6.1 Uncertainty Representation by User Type

| User Type | Primary Output | Secondary Output |
|-----------|---------------|-----------------|
| **Consumer** (browser extension) | Bayesian credible interval: "Likely authentic (67-82% confidence)" | Signal agreement summary: "3 of 5 signals agree. 2 strongest signals concur." |
| **Professional** (explainability panel) | Full posterior probability distribution over categories | Per-signal likelihood ratios with individual confidence, signal quality tier breakdown |
| **Enterprise/API** | Bayesian credible interval + optional conformal prediction set | Raw per-signal scores for integration into enterprise workflows |

### 6.2 Discrete Confidence Levels

Confidence is communicated in discrete levels, NEVER as continuous percentages (which create false precision and enable model inversion attacks):

| Level | Probability Range | Consumer Label | Professional Label |
|-------|-------------------|----------------|-------------------|
| **5** | >90% | Very High Confidence | Extremely strong evidence |
| **4** | 75-90% | High Confidence | Strong preponderance of evidence |
| **3** | 55-75% | Moderate Confidence | Evidence favors this conclusion |
| **2** | 35-55% | Uncertain | Evidence is ambiguous or conflicting |
| **1** | 10-35% | Low Confidence | Weak evidence; treat with caution |
| **0** | <10% or no evidence | Insufficient Evidence | Cannot assess; not enough data |

### 6.3 What Must NEVER Be in the Output

- "100%" or "0%" confidence (implies certainty, which is impossible)
- "This is AI-generated" (binary claim; Sentinel provides assessments, not verdicts)
- "Verified authentic" ("verified" implies a guarantee)
- "Trust score" (implies trustworthiness judgment rather than evidence assessment)
- "Fake" or "Real" (binary, judgmental language)

### 6.4 Missing Signal Communication

When evidence signals are unavailable, the absence is communicated explicitly:

**"Not checked" vs. "Checked and found nothing":**
- "Not checked": The signal was not invoked (user disabled, browser doesn't support, requires cloud and user opted out)
- "Checked, nothing found": The signal ran but found no evidence either way (inconclusive, not absence)
- "Checked, unavailable": The signal attempted to run but failed (network error, model not loaded, timeout)

### 6.5 Temporal Confidence Decay

For time-sensitive content, confidence assessments include a recency indicator:

- "Analysis from 2 hours ago. This content relates to a rapidly developing event — confidence may have changed."
- "Analysis from 3 days ago. Models may have improved since then. Re-analysis recommended for critical decisions."
- Assessments are timestamped and frozen. Improving models do not retroactively change past assessments.

---

## Section 7: How Conflicting Evidence Should Be Handled

### 7.1 Philosophy: Conflict Is Information, Not Noise

When evidence signals disagree, the Trust Engine must NOT smooth the disagreement into an uninformative average. A bland "52% confidence" that hides a war between +90% (C2PA says authentic) and -85% (forensics say AI-generated) is actively misleading. The conflict IS the story.

### 7.2 Conflict Tiering

| Conflict Tier | Definition | Behavior |
|---------------|-----------|----------|
| **Critical Conflict** | Tier 1 vs. Tier 1 (e.g., two conflicting C2PA manifests) | Surface prominently. "Content has conflicting provenance claims." Both claims displayed. Confidence capped at Uncertain. |
| **High Conflict** | Tier 1 vs. Tier 2 (e.g., C2PA valid, forensics say AI-generated) | Conflict surfaced with explanation. Tier 1 dominates but Tier 2 is not hidden. "Valid provenance, but detection signals disagree." |
| **Moderate Conflict** | Tier 2 vs. Tier 2 (e.g., frequency analysis says AI, watermark says authentic) | Noted as disagreement. Confidence reflects weighted average of signals. Both signals visible. |
| **Low Conflict** | Tier 3/4 vs. anything higher | Lower-tier signal noted but does not create a conflict. The higher-tier signal carries. |

### 7.3 Resolution Rules

| Scenario | Rule |
|----------|------|
| Hardware-attested C2PA vs. Any detection signal | Hardware attestation wins. Creates confidence floor. Detection signal noted but does not override. |
| Standard C2PA vs. Known synthetic signature match | Conflict. Both are near-deterministic. Capped at Uncertain. User must apply judgment. |
| Standard C2PA vs. Frequency-domain artifacts | C2PA receives precedence but conflict is surfaced. "Valid provenance with contradicted forensic analysis. Possible explanations: AI generation with retroactive signing, edited content introducing artifacts, or detector false positive." |
| Multiple forensic signals disagree internally | Trust the higher-trust source (based on historical calibration). Surface disagreement. |

### 7.4 User Guidance for Conflicts

Every conflict must include suggested next steps:
- "Consider seeking additional evidence: verify with the content creator, check for other copies online."
- "If this content is critical to your decision, we recommend independent verification."
- "Report this assessment if you have ground truth information about this content."

---

## Section 8: How Explainability Should Work

### 8.1 Three-Tier Explainability Model

**Tier 1: Quick Glance (1-3 seconds)**
- Confidence level badge (discrete: "High Confidence" / "Uncertain" / etc.)
- Direction indicator (likely authentic ↔ likely synthetic)
- Top 1-2 strongest evidence points in natural language
- Timestamp of analysis
- Privacy mode indicator (local-only vs. cloud-assisted)

**Tier 2: Summary View (10-30 seconds)**
- Confidence level with credible interval
- Per-signal breakdown: every evidence signal listed with its individual finding and tier
- Signal agreement summary
- Missing signals listed with reason for absence
- Key findings in natural language paragraphs
- Actionable recommendations

**Tier 3: Detailed Report (2-5 minutes)**
- Full evidence chain: every signal, its source, its version, its confidence, its contribution
- Methodology notes: how signals were combined, calibration history, signal independence assessment
- Raw per-signal likelihood ratios (for technical users)
- Chain of custody: timestamp, model versions, content hash, platform information
- Exportable as signed PDF or JSON for institutional use
- Cryptographic signature for third-party verification

### 8.2 Explainability Language Principles

**DO:**
- Natural language: "Patterns consistent with AI generation were detected in this image"
- Calibrated certainty: "The evidence suggests this content is likely authentic"
- Transparency about limitations: "Text analysis is the least reliable signal and may produce incorrect results"
- Signal weight communication: "C2PA provenance (strong signal) indicates this image was captured by a camera"

**DON'T:**
- Technical metrics in consumer-facing output: "Perplexity: 234.7, burstiness: 0.43"
- Binary language: "This is fake" / "This is real"
- Certainty language: "Verified" / "Confirmed" / "Guaranteed"
- Hidden disagreement: never present an averaged score that obscures conflicting signals

### 8.3 Evidence Visual Language

Six categories of evidence, visually distinguished:

| Category | Visual Weight | Indicates |
|----------|---------------|-----------|
| Provenance | Highest prominence | Cryptographic or hardware evidence of origin |
| Detection | Medium prominence | ML-based forensic analysis |
| Reputation | Lower prominence | Source/domain historical trustworthiness |
| Corroborative | Supporting role | Additional signals that agree with primary |
| Missing | Neutral indicator | Signal was not checked or was unavailable |
| Inconclusive | Neutral indicator | Signal ran but found nothing either way |

Direction indicators: positive (supports authenticity) vs. negative (suggests synthetic) — visually distinct but never red/green alone (accessibility).

### 8.4 Actionable Outcomes

Every assessment must answer the "so what?" question with concrete next steps:

| Confidence Level | Authenticity Suggested | AI Generation Suggested |
|-----------------|----------------------|------------------------|
| **Very High / High** | "Evidence strongly supports authenticity. You could: cite this content with confidence, publish with Sentinel evidence attached, share with attribution." | "Strong evidence of AI generation. You could: seek original source, verify with creator, flag for review, do not cite as factual without corroboration." |
| **Moderate** | "Evidence favors authenticity with some uncertainty. You could: cross-check with additional sources, note the confidence level if citing." | "Evidence suggests AI generation. You could: request original/unedited version from source, check for alternative copies, apply additional scrutiny." |
| **Uncertain** | "Evidence is ambiguous. You could: seek additional verification methods, check metadata manually, consult a subject matter expert." | Same as Uncertain-authentic. |
| **Low / Insufficient** | "Not enough evidence to assess. You could: try re-analysis with cloud features enabled, seek provenance information from the publisher, manually verify." | Same as Low-authentic. |

---

## Section 9: How Users Should Interact With Results

### 9.1 Persona-Specific Behavior

| Persona | Default View | False Positive Sensitivity | Confidence Threshold for Action |
|---------|-------------|---------------------------|-------------------------------|
| **Journalist** | Summary View | Moderate — needs evidence to defend publication decisions | Moderate Confidence to publish; Uncertain triggers additional verification |
| **Educator** | Summary View with natural language | **Very High** — false accusation of AI use is catastrophic | High Confidence to open integrity case; Moderate to discuss with student |
| **Content Marketer** | Quick Glance for batch; Summary for flagged | Low-Moderate — false positive means asking freelancer to revise | Customizable AI allowance threshold (configurable % tolerance) |
| **Consumer** (future) | Quick Glance only | Low — mild annoyance if wrong | Personal judgment only |

### 9.2 User Feedback Mechanism

Users can provide feedback on assessments:

- **"This assessment seems correct"** — positive reinforcement. Feeds calibration monitoring.
- **"This assessment seems incorrect"** — with reason: content is actually authentic / content is actually AI-generated / evidence was misinterpreted / other.
- **Override:** User can annotate an assessment with their own judgment. This does NOT retroactively change the Trust Engine's assessment — it adds a user annotation. The original assessment is preserved for chain of custody.
- **Report:** User can flag particularly problematic assessments for human review.

All feedback is opt-in. No feedback data contains content — only assessment IDs and user judgments.

### 9.3 Export & Citation

Professional users can export assessments as:

- **Evidence Report (PDF):** Signed, verifiable, self-contained. Includes full evidence chain, methodology, and cryptographic signature. Suitable for editorial notes, academic integrity boards, and legal proceedings.
- **Evidence Record (JSON):** Machine-readable format for integration into CMS, LMS, or editorial workflows.
- **Assessment Link:** A shareable reference that allows third parties to verify the assessment (requires Sentinel verification service).

### 9.4 Privacy-Confidence Tradeoff

Users are explicitly shown the tradeoff between privacy and confidence:

- **Local-Only Mode:** Maximum privacy. Zero content leaves device. Some evidence unavailable (reverse image search, cloud reputation, community signals). Confidence ceiling may be lower.
- **Network-Assisted Mode:** Domain/URL lookups (not content) sent to Sentinel servers. Additional reputation and contextual evidence available.
- **Full Analysis Mode:** Content may be sent to cloud services for advanced analysis (reverse image search, deepfake detection, threat intelligence). Opt-in per analysis. Content processed and immediately discarded.

The current privacy mode is always visible. Users are informed when enabling a less-private mode would increase confidence.

---

## Section 10: Evolution Without Architectural Redesign

### 10.1 Design Invariants

The Trust Engine's core fusion logic must satisfy these invariants to remain stable as evidence sources evolve:

| Invariant | Guarantee |
|-----------|-----------|
| **Source-agnostic fusion** | The fusion algorithm operates on Evidence Results — structured output from any source. It never branches on source identity. It reasons over signal values, evidence types, and confidence scores only. |
| **Variable evidence count** | Handles any number of evidence results, from 0 (returns "insufficient evidence") to N (all sources ran). |
| **Variable evidence types** | Does not assume specific evidence types exist. Works with whatever is available. |
| **Graceful degradation** | When a source is unavailable, the Trust Engine proceeds with remaining sources. Missing evidence is noted but does not block assessment. |
| **No hardcoded thresholds per source** | Confidence thresholds, conflict rules, and weight caps are expressed in terms of evidence tiers and classes — never specific sources. |

### 10.2 Adding New Evidence Sources

1. Source registers with its capability declaration: content types, evidence class, latency profile, privacy tier
2. Source enters **shadow evaluation** — results collected but not contributed to confidence for first 1,000 analyses
3. After shadow period: if accuracy, calibration, and reliability meet thresholds → promoted to ACTIVE
4. No change to fusion algorithm. No code change required in Trust Engine.

### 10.3 Handling Obsolete Evidence Sources

1. Trust Engine detects obsolescence through: declining accuracy, increasing inconclusive rate, systematic disagreement with better-calibrated sources, version age > 12 months
2. Source weight automatically decays. ML-based sources decay; cryptographic sources (C2PA verification) do not.
3. When weight drops below threshold → DEPRECATED. Users notified.
4. After deprecation period → RETIRED. Removed from active orchestration. Historical analyses retain reference.

### 10.4 Handling Improved Evidence Sources

1. New version registered with incremented version identifier
2. A/B evaluation: both old and new versions run in parallel; new version in shadow mode
3. If new version outperforms → promoted to ACTIVE, old version deprecated
4. If new version underperforms → marked DEGRADED, old version retained
5. Users informed: "Image Forensics Model v3.1 → v3.2 (improved accuracy)"

### 10.5 Model Degradation Strategy

AI detection models degrade 10-20% per new model generation. The Trust Engine is designed for this:

1. **Weights reflect durable accuracy, not snapshot accuracy.** A model at 90% accuracy today that will be at 60% in 12 months carries less weight than a provenance signal at 99% permanently.

2. **Per-signal recalibration, not retraining.** When a model degrades, its likelihood ratio is recalibrated downward. The model stays the same; its weight changes.

3. **Ensemble diversity as degradation insurance.** At least two detection approaches per modality using fundamentally different architectures ensure uncorrelated failure modes.

4. **Tier 1 signals as stability anchors.** C2PA verification and hardware attestation do not degrade. The worst case with failed detection signals is "uncertain, but provenance data is available."

5. **Continuous calibration monitoring.** Each signal's real-world accuracy is tracked. When accuracy diverges from assumed calibration, weight is automatically reduced.

---

## Section 11: Evidence Source Contract

Every evidence source that integrates with the Trust Engine must fulfill this contract. Sources that do not satisfy this contract cannot be used.

### 11.1 What Every Source Must Produce

| Field | Required | Description |
|-------|----------|-------------|
| `signal_value` | Yes | What was detected. Discrete: AUTHENTIC, SYNTHETIC, MANIPULATED, INCONCLUSIVE, ERROR. Or continuous: score in documented range. |
| `confidence` | Yes | Self-assessed certainty (0.0–1.0). 1.0 = source is certain. Must be calibrated — confidence 0.95 must mean ~95% accuracy. |
| `evidence_type` | Yes | Classification: PROVENANCE, FORENSIC, REPUTATION, COMMUNITY, DERIVED |
| `timestamp` | Yes | ISO 8601. When evidence was gathered. |
| `source_identifier` | Yes | Unique detector/model ID including version. |
| `explanation` | Yes | Human-readable description (max 280 chars). Suitable for Quick Glance display. |
| `evidence_detail` | No | Source-specific structured data for Detailed Report. |
| `error` | No | Present only if signal_value is ERROR. |

### 11.2 Error Contract

| Failure Mode | Required Behavior |
|-------------|-------------------|
| Source crash/exception | Return ERROR (INTERNAL_ERROR). Never throw exceptions. |
| Timeout | Return ERROR (TIMEOUT). Self-enforce declared timeout. |
| Ambiguous result | Return INCONCLUSIVE with confidence < 0.5. Not an error — valid signal that source has no information. |
| Source unavailable | Return ERROR (UNAVAILABLE). Trust Engine may retry. |
| Content not processable | Return ERROR (UNSUPPORTED_CONTENT). Trust Engine should not re-route this type. |
| Partial result | Return best available result with reduced confidence and PARTIAL_RESULT error. |

### 11.3 Privacy Tiers for Evidence Sources

| Tier | Description | Consent Required | Default |
|------|-------------|-----------------|---------|
| **Tier 1: Zero Data Exfiltration** | Content never leaves device. C2PA verification, metadata extraction, on-device image forensics, text analysis, local reference data queries. | None beyond extension installation | ON |
| **Tier 2: Anonymized Query** | Network requests that do NOT send user content. Domain reputation lookup by hash, certificate revocation check, known AI hash lookup. | Implicit (first-run notice) | ON (single opt-out toggle) |
| **Tier 3: Content Transmission** | Actual content sent to server. Reverse image search, cloud deepfake detection, threat intelligence, full-model LLM analysis. | **Explicit per-analysis opt-in** | OFF |

### 11.4 Data That the Trust Engine MUST NOT Access

Even if technically possible, the Trust Engine must never access:
- User identity / PII
- Browsing history beyond the current page
- Form data, passwords, payment information
- Cookies, session tokens
- Geolocation
- WebRTC/media streams unless explicitly invoked
- Content from other browser tabs
- Content from private/incognito browsing unless explicitly enabled

### 11.5 Cache & Retention Policy

| Data Type | Maximum Retention | Storage Location |
|-----------|-------------------|-----------------|
| Analysis results (local) | 30 days or until user clears | Encrypted local storage |
| Domain reputation cache | 7 days (then marked stale) | Local storage |
| User preferences/thresholds | Until user changes | Local storage |
| User override history | 90 days | Local storage |
| Content analyzed | **NEVER stored** | N/A |
| Full URLs analyzed | **NEVER stored** | N/A |
| Cloud API analysis payloads | **Process and immediately discard** | N/A |

---

## Section 12: Security Requirements (Summary)

The Trust Engine must satisfy these security invariants. Full requirements are defined in the Security Requirements Specification (`docs/TRUST_ENGINE_SPEC.md` security section).

### 12.1 Critical Security Invariants

1. **Every provenance assertion must be cryptographically verified before acceptance.** Presence alone is not enough. Signature, certificate chain, and content binding must all validate.

2. **No single signal may exceed 40% of total weight.** No single evidence class may exceed 60%.

3. **"High Confidence" requires evidence from at least 2 of the 6 evidence classes.**

4. **Confidence levels must be discrete, never continuous percentages.** Anti-inversion, anti-liability measure.

5. **All detection models must be cryptographically signed.** Verified before loading. Updates must be resistant to downgrade attacks.

6. **When the Trust Engine detects it may be compromised (modified files, failed signature checks): stop producing assessments.** Fail closed, not open.

7. **The safe default is to flag as uncertain/suspicious, not to trust by default.** Erring toward flagging is safer than erring toward trust.

8. **All exported confidence reports must be cryptographically signed** so third parties can verify they were genuinely produced by Sentinel.

---

## Section 13: Self-Review

### Product Perspective

**Strengths:** The specification aligns with the Phase 0 Discovery Report's recommendation to narrow focus. The Trust Engine is explicitly designed for professional content evaluators first. The evidence hierarchy provides clear product differentiation from binary AI detectors.

**Weaknesses:** The specification is ambitious. The number of evidence types may create an expectation of completeness that the Phase 2 MVP cannot deliver. The spec must communicate that "Tier 1-4 is the MAP, not the current build."

**Recommendation:** Add an explicit "Phase 2 MVP Evidence Scope" section that identifies which evidence types ship in MVP vs. Phase 3 vs. Phase 4. Currently, everything is specified but nothing is prioritized for shipping.

### Security Perspective

**Strengths:** The security invariants directly address the 4 showstopper risks from Phase 0. The discrete confidence levels, signal weight caps, evidence class diversity requirement, and model signing requirements create defense-in-depth.

**Weaknesses:** The security requirements assume the Trust Engine runs in a trusted execution environment (browser extension sandbox). If the browser itself is compromised, all guarantees collapse. This is inherent to client-side analysis and should be acknowledged.

**Recommendation:** Add an explicit statement: "The Trust Engine's security guarantees assume the browser extension runtime is not compromised. Sentinel cannot protect against compromised operating systems, browser zero-days, or physical device access."

### Privacy Perspective

**Strengths:** The three-tier privacy model for evidence sources (Zero Data Exfiltration / Anonymized Query / Content Transmission) is clear and enforceable. Local-first processing with explicit opt-in for cloud features aligns with the Phase 0 recommendation.

**Weaknesses:** The specification defines WHAT data must not be accessed but does not define HOW to prevent accidental access. The "content analyzed: never stored" rule requires implementation-level guarantees (data loss prevention in logging, telemetry, error reporting) that are beyond this specification's scope but critical in practice.

**Recommendation:** The architecture phase must include a Data Loss Prevention (DLP) specification for the Trust Engine. "Never stored" is a policy; DLP is the enforcement mechanism.

### Machine Learning Perspective

**Strengths:** The evidence hierarchy correctly reflects the mathematical reality of AI detection. Text detection is correctly relegated to a weak corroborating signal. The Bayesian fusion approach is mathematically sound. The signal independence requirement addresses the overconfidence problem.

**Weaknesses:** Bayesian priors (Section 5.5) are specified as "periodically recalibrated based on measured base rates" but the mechanism for measuring base rates is undefined. If base rate measurement is itself biased (e.g., sampling from news websites overestimates human content), the prior is wrong and all posteriors are systematically shifted.

**Recommendation:** Add explicit guidance on base rate estimation methodology. The prior must be transparent, versioned, and independently auditable. Users should be informed what prior was used.

### Enterprise Perspective

**Strengths:** The evidence source contract, versioning, and deprecation model create a pluggable architecture that enterprises need for custom integrations. The conformal prediction wrapper provides statistical guarantees for regulated industries. Chain of custody requirements support compliance use cases.

**Weaknesses:** The specification assumes all users are individuals. Enterprise deployment scenarios (fleet management, mandatory policies, audit log access, admin visibility into user assessments) are mentioned in privacy requirements but not integrated into the main Trust Engine specification.

**Recommendation:** Add an "Enterprise Deployment Model" section that specifies: admin policy controls, fleet-wide source enablement/disablement, audit log requirements, and the boundary between individual user privacy and organizational visibility.

### UX Perspective

**Strengths:** The three-tier explainability model (Quick Glance / Summary / Detailed Report) is well-structured. The actionable outcomes framework addresses the "so what?" problem identified in Phase 0. The conflict handling approach (surface, don't smooth) aligns with transparency principles.

**Weaknesses:** The specification defines WHAT must be communicated but not the priority of communication. When there are 8 evidence signals, 2 conflicts, 3 missing signals, and recency warnings — what does the user see FIRST? Information hierarchy during high-complexity assessments is undefined.

**Recommendation:** Add an "Information Priority Order" that defines: (1) confidence level, (2) top 2 signals, (3) critical conflicts, (4) missing signal warnings, (5) recency, (6) remaining signals. Progressive disclosure should respect this order.

---

## Section 14: Unanswered Questions

The following questions must be resolved before ARCHITECTURE.md begins:

### Evidence & Reasoning

1. **Base rate estimation:** How will the Trust Engine measure the actual prevalence of AI-generated content for Bayesian priors? What is the calibration methodology?

2. **Signal calibration dataset:** What ground truth dataset will be used to calibrate each signal's likelihood ratio? How will this dataset be kept current as new AI models emerge?

3. **Prior transparency:** Should users be able to see and adjust the Bayesian prior? Should the prior be user-configurable or system-managed?

### Product & Scope

4. **Phase 2 MVP evidence scope:** Which specific evidence types ship in the MVP? Which are Phase 3? Phase 4? The spec currently describes the complete vision — prioritization is needed.

5. **Browser capability communication:** If Chrome gets "High Confidence" and Safari gets "Moderate Confidence" on the same content because Safari lacks tabCapture — is this acceptable? Or does the product need to maintain consistent confidence across browsers?

6. **Offline-first vs. online-enhanced:** Should the MVP require network for ANY features (reference data downloads, domain lookups), or should it function fully offline with reduced confidence?

### Security & Legal

7. **Discrete confidence levels — how many?** The spec defines 6 levels. Is 6 the right number? Too many? Too few? This directly affects user trust and model inversion resistance.

8. **Confidence floor for safety overrides:** What is the numerical confidence floor when hardware-attested C2PA fires? "Cannot be overridden" is a rule; the confidence value produced is undefined.

9. **Liability disclaimer placement:** The spec requires a legal disclaimer on every output. Should this be visible at all times or collapsible? Permanent visibility degrades UX; collapsibility risks legal exposure.

### Architecture & Integration

10. **Evidence source sandboxing:** How are dynamically registered (user-installed, community) evidence sources sandboxed? The spec says "must be sandboxed" but the mechanism is undefined.

11. **Cross-browser service worker vs. Event Page strategy:** Firefox Event Pages enable longer background analysis than Chrome service workers. Should the Trust Engine exploit this to run more evidence sources on Firefox?

12. **Model download and update UX:** Large ONNX models (5-150 MB) must be downloaded after extension install. What is the UX for "your trust assessment quality will improve after models download"?

---

## Appendix A: Key Terms

| Term | Definition |
|------|-----------|
| **Confidence assessment** | The Trust Engine's output: a discrete-level estimate of content authenticity based on fused evidence, with supporting explanation. Never a truth determination. |
| **Evidence signal** | An independent piece of information about content authenticity from a specific detector or verifier. |
| **Evidence class** | A category of signals sharing fundamental methodology. Signals within a class tend to fail together. |
| **Fusion** | The process of combining multiple evidence signals into a single confidence assessment. |
| **Bayesian prior** | The base rate assumption about AI-generated content prevalence, before any evidence is evaluated. |
| **Likelihood ratio** | P(evidence \| AI-generated) / P(evidence \| authentic). The contribution of a single signal in Bayesian updating. |
| **Credible interval** | A Bayesian range within which the true classification probability falls with specified certainty. |
| **Calibration** | Alignment between stated confidence and actual outcome frequency. |
| **Conformal prediction set** | A set of possible classifications with a statistical guarantee that the true class is in the set at a specified rate. |
| **Chain of custody** | Verifiable record of analysis: what was analyzed, when, by which model versions, with what result. |
| **Air gap** | Severing of digital provenance chains through analog conversion (screenshots, screen recordings). |

---

## Appendix B: Reference Documents

- Phase 0 Discovery Report: `docs/discovery/DISCOVERY_REPORT.md`
- Content Provenance Standards Research: `research/content-provenance-standards.md`
- Security, Privacy & Legal Research: `research/SECURITY_PRIVACY_LEGAL_RESEARCH.md`
- Product Strategy & UX Research: `research/product-strategy-and-ux-research.md`
- Business Plan: `BUSINESS_PLAN.md`

---

*End of TRUST_ENGINE_SPEC.md. This document is the authoritative product reasoning specification for Sentinel. All architecture, implementation, and design decisions must trace back to the principles, rules, and constraints defined herein.*
