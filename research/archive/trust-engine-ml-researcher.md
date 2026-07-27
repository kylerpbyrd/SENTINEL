# TRUST_ENGINE_SPEC.md — Trust Engine Product Reasoning Specification

> **Document Type:** Product Reasoning Specification (NOT implementation)
> **Phase:** 1 — Architecture
> **Owner:** Machine Learning Researcher
> **Audience:** Architecture Team, Product Manager
> **Status:** Draft
> **Depends On:** Phase 0 Discovery Report, Content Provenance Research

---

## Purpose

This document defines the logical reasoning model for how Sentinel combines independent evidence signals into **confidence assessments** about digital content authenticity. It specifies:

- Which evidence signals exist, how they rank in reliability, and why
- Which signals must never serve as standalone detectors
- How signals combine into a coherent confidence output
- How uncertainty is represented honestly to users
- How conflicting evidence is resolved
- How the system adapts as individual detection models degrade

**This is a product reasoning specification.** It does not discuss programming languages, frameworks, APIs, databases, or implementation details. It focuses exclusively on the logic of evidence, confidence, and uncertainty.

---

## 1. Evidence Confidence Hierarchy

The Trust Engine must weight evidence by its inherent reliability. The hierarchy below ranks evidence types from strongest (most durable, hardest to spoof, highest-evidentiary value) to weakest (most volatile, easiest to spoof, lowest-evidentiary value). Weak signals are still useful; they must simply carry appropriate weight.

### Tier 1: Ground Truth Evidence

These signals provide **cryptographic or hardware-level guarantees.** They cannot be forged without compromising cryptographic infrastructure or physical hardware. When present and valid, they should dominate the confidence assessment.

| Rank | Evidence Type | Rationale | Weight |
|------|--------------|-----------|--------|
| **1** | **Cryptographic provenance (C2PA with valid certificate chain)** | Cryptographically signed, tamper-evident manifest attesting to content origin, creation tool, edit history, and publisher identity. Verification is a deterministic computation, not a probabilistic inference. Trust transfers from the user to the signer via PKI — the same infrastructure that secures HTTPS. | **Dominant.** When C2PA is present with a valid cert chain from a recognized CA, it provides the single strongest signal available. Should outweigh all detection signals combined. |
| **2** | **Hardware-attested provenance (Leica M11-P, Sony Alpha signed capture)** | A subset of C2PA where signing occurs at capture moment inside a hardware secure element. The cryptographic key never leaves the chip. This eliminates the gap between "content created" and "content signed" — the most vulnerable interval in any provenance chain. | **Dominant, slightly above standard C2PA.** Hardware attestation eliminates the possibility that unsigned content was retroactively signed by a malicious actor. The certificate chain starts at the silicon level. |

**Why Tier 1 is strongest:**
- Verification is deterministic, not probabilistic. A signature either validates or it doesn't.
- Spoofing requires either CA compromise, hardware tampering, or breaking SHA-256 — all nation-state-level attacks.
- The signal provides a complete provenance chain: who created it, with what tool, when, whether it was edited, and who published it.

**When Tier 1 becomes unreliable:**
- **Stripped manifests:** Social media platforms, screenshots, or format conversions routinely strip C2PA data. The absence of a manifest is not evidence of inauthenticity — it's evidence that the provenance was lost.
- **Compromised signer:** A malicious actor who obtains a valid signing certificate from a CA can sign fake content with valid provenance. C2PA verifies WHO signed, not that the signer's claims are truthful.
- **Signer trust assumption:** Trust in C2PA is transitive. If a CA issues certificates to bad actors, the signal is invalid. Trust in the CA itself must be continuously evaluated.
- **Screenshots / air gap:** Any analog capture (photograph of screen, screenshot) creates new content with no provenance, severing the chain.

**Can Tier 1 function alone?**
- **Yes.** C2PA verification combined with CA reputation assessment is sufficient as a standalone positive signal. Content with valid, trusted C2PA manifests can receive a high-confidence "authentic" assessment without additional signals. Content WITHOUT C2PA must not be downgraded for that absence alone (see "Absence of Provenance" in Tier 4).

---

### Tier 2: Strong Corroborating Evidence

These signals are **statistically robust but probabilistic.** They provide strong evidence under known conditions but degrade measurably outside those conditions. They should never override Tier 1 evidence but carry substantial weight in its absence.

| Rank | Evidence Type | Rationale | Weight |
|------|--------------|-----------|--------|
| **3** | **Cross-modal consistency** (lip-sync mismatch, image-caption semantic gaps, physics violations) | AI generation models often produce content with subtle inconsistencies between modalities: mouths that don't match speech, shadows pointing in physically impossible directions, text in images that is garbled, captions that don't match visual content. These violations are not an artifact of any specific model architecture — they are fundamental to the current generation paradigm and survive model improvements. | **High.** Physics violations and cross-modal inconsistencies are among the most durable AI detection signals because they are not model-dependent. However, they require multi-modal analysis (cannot analyze a still image for lip-sync issues) and are vulnerable to post-generation editing. |
| **4** | **Known synthetic media signatures** (pixel-perfect matches to known AI outputs) | When content matches exactly (hash-level) known AI-generated media in a reference database, this is near-deterministic evidence of AI origin. This is not statistical inference — it's fingerprint matching. | **Very high when positive.** A verified match is near-certain. However, coverage is inherently limited: novel AI outputs won't be in the database. This is a high-precision, low-recall signal. |
| **5** | **Image frequency-domain artifact analysis** (DIRE, NPR, etc.) | AI-generated images leave frequency-domain artifacts distinct from natural photographs and human-created digital art. These artifacts arise from the upsampling, decoding, and generation process. Detection accuracy on known generators: 75-90%. | **Medium-High.** Strongest of the ML-based detection signals. However, accuracy drops 10-20% per new model generation and degrades with post-processing (resizing, compression, filters). Not reliable after heavy editing. |
| **6** | **Generative watermark detection** (DALL-E, Midjourney, SynthID watermarks) | Watermarks embedded at generation time by specific platforms. SynthID (Google) embeds imperceptible statistical watermarks in pixel values. DALL-E uses C2PA manifests. Midjourney uses visible and invisible watermarks. | **Medium-High for DALL-E/C2PA watermarks** (verifiable locally). **Limited for SynthID** (proprietary detection, requires Google cooperation — independent verification is not possible). Signal value degrades when content passes through compression, cropping, or screenshots that damage the watermark. |

**Why Tier 2 is strong but not dominant:**
- Cross-modal consistency is architecture-independent and survives model evolution — a physicist doesn't need to know which AI model generated an image to spot physically impossible lighting.
- Frequency-domain artifacts are the best-studied ML detection signal for images but are in an arms race with generators.
- Watermark detection is solid but limited to specific platforms and degrades under transformation.

**When Tier 2 becomes unreliable:**
- **Cross-modal consistency:** Requires multi-modal content (video with audio, image with caption). Single-image analysis cannot detect lip-sync issues. Sophisticated actors can manually fix inconsistencies post-generation.
- **Known synthetic signatures:** Zero coverage for novel generations. Database must be continuously updated. Adversaries can add imperceptible noise to evade hash matching.
- **Frequency-domain artifacts:** Detection accuracy degrades with each new model generation. Post-processing (Instagram filters, compression artifacts from messaging apps, resizing for web) reduces signal. DIRE and similar models trained on 2024 generators may be near-random accuracy on 2027 generators.
- **Watermark detection:** SynthID is not independently verifiable. DALL-E watermarks are C2PA-based (belongs in Tier 1 when valid, Tier 2 when detection is uncertain). Watermarks can be stripped via screenshot or transcoding.

**Can Tier 2 function alone?**
- **No.** Cross-modal consistency should not be used as the sole signal because it suffers from coverage gaps (requires specific content types) and can be manually corrected. Frequency-domain artifacts should not be used alone due to accuracy degradation and model-dependence. Known synthetic signatures are too high-precision/low-recall to be useful alone. **All Tier 2 signals should be used in combination with at least one other signal from Tier 2 or higher.**

---

### Tier 3: Moderate Corroborating Evidence

These signals provide **useful but volatile evidence.** They are accurate under ideal conditions but degrade significantly under real-world conditions. They add confidence when multiple Tier 3 signals agree, but individually they are insufficient.

| Rank | Evidence Type | Rationale | Weight |
|------|--------------|-----------|--------|
| **7** | **Face-swap detection** (Xception + temporal consistency + physiological signals) | Deepfake face-swaps are detectable at 80-90% accuracy using a combination of: Xception-based frame analysis, temporal inconsistencies between frames (flickering, identity instability), and physiological signal analysis (inconsistent pulse signal, unnatural eye movement, irregular blinking patterns). | **Medium.** This is the strongest video-specific detection signal. However, it applies only to face-swap deepfakes, not fully AI-generated video (Sora, Runway), which requires a fundamentally different detection approach with much lower accuracy. |
| **8** | **Audio vocoder artifact detection** (raw source quality) | AI-generated audio from neural vocoders leaves specific spectral artifacts detectable in raw, uncompressed audio. Viable at 80-90% on source audio. | **Medium, highly conditional.** Viable only for raw source audio. Degrades to near-random accuracy after telephony compression, background noise, or re-encoding at low bitrates — which describes most audio encountered on the web. |
| **9** | **IPTC/EXIF metadata analysis** (creator, timestamp, device info consistency) | Metadata embedded in media files provides information about the capture device, creation time, software used, and editing history. Inconsistencies (e.g., "Created with iPhone" but EXIF shows a Canon camera; creation date predates the device model's release) are strong negative signals. | **Medium for inconsistency detection.** Metadata consistency checking is deterministic logic, not ML inference. However, metadata is trivially edited, routinely stripped, and unreliable on its own. Consistency is more informative than presence — a mismatch is suspicious; correct metadata is easily fabricated. |
| **10** | **Reverse image/audio search** (prior existence on the web) | Content that appears earlier on the web under different context is suspicious. Content that has been online for years under a specific attribution and is now being presented as novel is suspicious. | **Medium, narrow applicability.** Works only for content that has been published before. Does not detect novel AI generations. Cannot distinguish between "re-shared authentic content" and "re-shared AI content." Makes a strong signal when combined with metadata inconsistency. |

**Why Tier 3 is moderate:**
- Detection accuracy degrades under real-world conditions (compression, editing, noise).
- Face-swap detection is well-studied but doesn't generalize to fully AI-generated video.
- Metadata analysis is trivially bypassed by competent adversaries.
- Reverse search has limited coverage and ambiguous interpretation.

**When Tier 3 becomes unreliable:**
- **Face-swap detection:** Fully AI-generated video (Sora, Runway, Veo) does not involve face-swapping and requires entirely different detection approaches — which are essentially non-functional at useful accuracy currently. Temporal consistency degrades with low-quality source video. Physiological signals require high-resolution, unobstructed faces.
- **Audio vocoder artifacts:** Telephony compression, background noise, and re-encoding destroy the artifacts. Most content on social media, messaging apps, and news clips has been through multiple compression stages.
- **IPTC/EXIF metadata:** Any metadata can be fabricated or stripped. A negative finding (mismatch) is meaningful; a positive finding (metadata looks correct) is not — because correct-looking metadata is easy to forge.
- **Reverse search:** Novel AI generations have no prior web existence to detect. Archived authentic content can be repurposed for misinformation (prior existence = authentic, current use = misleading).

**Can Tier 3 function alone?**
- **Face-swap detection:** No. Only applies to a subclass of deepfakes. Should require corroboration from temporal consistency or metadata signals.
- **Audio vocoder artifacts:** No. Only useful on raw/uncompressed source audio. Should require corroboration from metadata or provenance signals.
- **IPTC/EXIF analysis:** No. Metadata is too easily fabricated. Use only as a cross-check against other signals.
- **Reverse search:** No. Coverage gaps and ambiguous interpretation. Use only as corroborating context.

---

### Tier 4: Weak/Aggregate-Only Evidence

These signals are **too volatile, too narrow, or too easily bypassed to function as independent evidence.** They provide value ONLY when combined with signals from higher tiers. They must never be the sole basis for any confidence assessment. **Critically, some of these must never be used at all (see Section 2).**

| Rank | Evidence Type | Rationale | Weight |
|------|--------------|-----------|--------|
| **11** | **Text statistical burstiness/perplexity analysis** (zero-shot, model-agnostic) | Human writing exhibits natural variation in sentence length, complexity, and word choice (burstiness). Text generated by language models tends toward uniform, "smoothed" output with unnaturally consistent sentence structure. Zero-shot perplexity analysis (measuring how "surprising" the text is to a language model of similar training distribution) provides a model-agnostic signal. | **Low, conditional.** Burstiness/perplexity analysis is model-agnostic and therefore more durable than model-specific classifiers. However, as LLMs improve, they produce increasingly human-like burstiness patterns. Cannot distinguish between "written by AI" and "written by a human with a formal, consistent style." False positive rate disproportionately affects non-native English speakers. |
| **12** | **Publisher/domain reputation** (historical accuracy, C2PA signing history) | Domains with a history of publishing verifiably false or manipulated content are less trustworthy sources. Domains that consistently publish C2PA-signed content from verified creators are more trustworthy. | **Low, decaying.** Reputation is a lagging indicator. Malicious domains are short-lived and churn rapidly. Reputable domains can be compromised. A single excellent reputation score means little; a pattern of poor reputation scores adds cumulative negative weight. |
| **13** | **Absence of provenance** (no C2PA, no metadata, no watermark) | Most content on the web lacks provenance data. The absence of provenance is the default state, not an anomaly. However, in contexts where provenance is expected (e.g., a news organization that typically publishes C2PA-signed photos suddenly publishes unsigned content), absence becomes informative. | **Context-dependent.** Must not be treated as a negative signal by default — that would penalize the vast majority of authentic content on the web. Absence should be flagged as "No provenance information available" (informational context) rather than "Suspicious because no provenance." Only carries negative weight when provenance is expected based on context (publisher norms, content type, platform conventions). |
| **14** | **Community signals** (user reports, social validation) | User reports of suspicious content, community notes, content moderation flags. Social validation signals (likes, shares, engagement patterns) provide weak indicators of authenticity. | **Very low, high-latency, easily manipulated.** Community signals are susceptible to brigading, coordinated inauthentic behavior, and astroturfing. They are reactive (require content to be seen and reported first) and therefore useless for novel content. Should only contribute when volume is high and verified by cross-referencing with non-community signals. |

**Why Tier 4 is weakest:**
- Text analysis is in a losing arms race with language models (see Section 2 for mathematical bounds).
- Reputation is always a lagging indicator and domain behavior changes faster than reputation scores.
- Absence of provenance is the default state for web content — penalizing absence would produce false positives on most authentic content.
- Community signals are trivially gamed and arrive too late for breaking content.

**When Tier 4 becomes unreliable:**
- **Text burstiness:** As LLMs improve, burstiness converges to human-level. Formal human writing (academic, legal, technical) already exhibits low burstiness similar to LLM output. Non-native English speakers produce writing patterns that detectors flag as AI-generated at elevated rates.
- **Publisher reputation:** Domain names churn rapidly. A domain with "excellent reputation" can become a misinformation source overnight (compromise, editorial change, content injection).
- **Absence of provenance:** Currently >95% of web content lacks C2PA. This ratio will decline over years, not months. Treating absence as a negative signal today would make Sentinel flag nearly everything.
- **Community signals:** Coordinated inauthentic behavior can flood any community signal with false reports. Social validation can be purchased.

**Can Tier 4 function alone?**
- **None of these signals may ever function alone.** See Section 2 for which must never be used at all.

---

## 2. Evidence That Must Never Be Used Alone

### 2.1 Text Perplexity-Based Classifiers (Model-Specific, Trained)

**Must never be used at all — not even in combination.**

The mathematical proof from **Sadasivan et al. (2023)** is definitive: as language models improve, the AUROC of even the best text detector converges to 0.5 — equivalent to random guessing. This is not an engineering problem to be solved with better models; it is a **mathematical inevitability.**

The proof's logic: a sufficiently capable language model generates text indistinguishable from the training distribution. When the model's output distribution matches human text distribution, no detector can distinguish between them with accuracy better than random. As models approach this capability, all text detectors degrade toward uselessness.

**Real-world confirmation:**
- Baseline detection accuracy: 39.5% on current models
- With adversarial paraphrasing: drops to 17.4% (worse than guessing)
- False positive rate on non-native English writers: up to 70% (**Liang et al., 2024**)
- OpenAI discontinued their own text classifier in July 2023, citing "low rate of accuracy"

**What this means for Sentinel:**
- Model-specific text classifiers (e.g., "trained to detect GPT-4.5 output") must NOT be included in the Trust Engine in any capacity — not as primary signals, not as corroborating signals, not even with low weight.
- Zero-shot burstiness/perplexity analysis (Rank 11 in the hierarchy) MAY be used as a weak corroborating signal but ONLY when: (a) the user is explicitly shown that text analysis is the least reliable signal, (b) it never contributes more than 10% of any confidence score, and (c) it is never the sole basis for any negative assessment.

### 2.2 Short Text Detection

Text samples under 50 words contain insufficient statistical structure for any detection method. Perplexity, burstiness, and linguistic pattern analysis all require minimum text lengths to produce meaningful signals. Short text detection is functionally random.

**Rule:** Text samples under 100 words must not receive any text-based AI detection assessment. The Trust Engine may still analyze metadata, provenance, or source reputation, but text-specific signals must be suppressed with the explicit message: "Text too short for reliable analysis."

### 2.3 Non-Native English Classification

**Liang et al. (2024)** demonstrated that GPT detectors falsely classify non-native English writing as AI-generated at rates up to 70%. The underlying cause is that non-native writing tends toward simpler vocabulary, more regular sentence structures, and lower burstiness — patterns that statistically resemble LLM output.

**Rule:** The Trust Engine must not use language proficiency as a detection signal. Text burstiness/perplexity analysis must be calibrated to account for non-native English writing patterns, and any text assessment must include the caveat: "Text analysis is less reliable for non-native English writing."

### 2.4 Binary "AI/Human" Text Classification as a Primary Signal

**Perkins et al. (2024)** confirmed that available GenAI detection tools produce inconsistent results, with accuracy varying wildly across models, domains, and languages. Binary classification ("AI" vs. "Human") is a misleading framing that creates a false sense of precision.

**Rule:** The Trust Engine must never output a binary "AI-generated" or "Human-written" verdict for any modality. Even for strong signals (C2PA, cross-modal consistency), the output must always be a confidence assessment with explicit uncertainty quantification.

### 2.5 Summary: The Never-Alone and Never-Use Rules

| Signal | Rule | Rationale |
|--------|------|-----------|
| Model-specific text classifiers | **NEVER USE** | Mathematically proven to converge to random (Sadasivan 2023) |
| Short text detection (<100 words) | **NEVER USE** for text signals | Insufficient statistical structure |
| Non-native English classification | **NEVER USE** for language proficiency signal | 70% false positive rate (Liang 2024) |
| Face-swap detection alone | **NEVER USE ALONE** | Only applies to face-swap deepfakes, not fully AI-generated video |
| Audio vocoder artifacts alone | **NEVER USE ALONE** | Degrades to near-random after compression |
| IPTC/EXIF metadata alone | **NEVER USE ALONE** | Trivially fabricated |
| Reverse search alone | **NEVER USE ALONE** | Coverage gaps, ambiguous interpretation |
| Text burstiness alone | **NEVER USE ALONE** | Converges to random as models improve |
| Publisher reputation alone | **NEVER USE ALONE** | Lagging indicator, domain churn |
| Absence of provenance | **NEVER USE AS NEGATIVE** | Default state for most authentic web content |
| Community signals alone | **NEVER USE ALONE** | Easily gamed, high-latency |

---

## 3. Evidence Combination Approaches

The Trust Engine's core function is combining multiple weak and strong signals into a coherent confidence assessment. Below are four candidate approaches, evaluated against criteria that matter for Sentinel's product philosophy.

### Approach A: Bayesian Evidence Accumulation

**How it works:** Each evidence signal contributes a likelihood ratio — the probability of observing this evidence given that the content is authentic vs. the probability of observing it given that the content is AI-generated. Starting from a prior belief (based on base rates of AI content), each signal updates the posterior probability using Bayes' rule. The final output is a posterior probability with a confidence interval derived from the variance of the log-odds.

**Mathematical honesty:** **Excellent.** Bayes' rule is the mathematically correct way to update beliefs given independent evidence. The approach makes its assumptions explicit: prior probability, conditional independence of signals, and the calibration of each signal's likelihood ratio. Users (or the system itself) can inspect whether signals are truly independent or whether dependencies inflate confidence.

**Explainability:** **Excellent.** The Bayesian framework inherently supports explainability. "We started with a base rate of 40% that this content might be AI-generated. The valid C2PA credential is very strong evidence for authenticity — it shifted our confidence to 85%. The pixel artifact analysis added moderate evidence. The final assessment of 78% reflects these combined factors." Each signal's contribution to the final score is transparent and decomposable.

**Robustness to missing signals:** **Excellent.** Missing signals simply don't update the posterior. If C2PA is unavailable, that signal provides zero information — the posterior is based only on available signals. This is the correct behavior: absence of a signal is not evidence (except for the "absence of provenance" signal, which is its own signal with its own likelihood ratio).

**Handling conflicting signals:** **Moderate.** Bayesian updating resolves conflict by weighted averaging. If C2PA strongly favors authenticity and pixel analysis strongly favors AI-generation, the posterior settles between them. The approach does not naturally flag "these signals are in conflict" — it produces a smoothed average. Additional machinery is needed to detect and surface signal conflict explicitly (see Section 5).

**Calibration quality:** **Good, with effort.** Bayesian calibration requires accurate likelihood ratios for each signal. This means each detector must be calibrated — when a detector says "80% confident," it must genuinely be correct 80% of the time. Calibration is achievable through held-out test sets and temperature scaling, but it degrades as models evolve (see Section 6). Recalibration must be continuous.

**Key weakness:** The conditional independence assumption. Evidence signals are not truly independent. A C2PA-signed image from a trusted camera will also have consistent EXIF metadata. Pixel artifacts and frequency-domain artifacts are correlated. Violating the independence assumption causes Bayesian updating to over-count evidence and produce overconfident scores. Mitigation requires explicit correlation modeling or conservative prior weighting.

---

### Approach B: Weighted Linear Fusion

**How it works:** Each signal produces a score in a common range (e.g., -1 to +1, where negative = evidence for AI-generation, positive = evidence for authenticity). Each signal has a learned or configured weight. The final confidence score is a weighted sum of all contributing signals, optionally passed through a calibration function (e.g., Platt scaling). Late fusion: signals are scored independently, then combined; each signal's internal process is opaque to the fusion layer.

**Mathematical honesty:** **Moderate.** Linear fusion is the simplest possible combination method. It makes no claims beyond "these signals, weighted by their past accuracy, sum to this score." It does not model uncertainty propagation, signal dependency, or likelihood ratios. It is honest in its simplicity — it does not pretend to be more sophisticated than it is.

**Explainability:** **Good.** Weights are interpretable: "C2PA verification carries 3x the weight of text burstiness analysis." Users can understand which signals matter most. However, linear fusion does not naturally decompose why a specific score was reached beyond listing the weighted contributions. It says "these signals contributed this much" but not "this evidence shifted our belief from X to Y."

**Robustness to missing signals:** **Moderate.** Missing signals must be represented as zero (neutral) or excluded with renormalization. Both approaches are defensible but qualitatively different: zero-weighting missing signals assumes missing = neutral; renormalization assumes available signals fully capture the evidence. Neither is ideal.

**Handling conflicting signals:** **Poor.** Linear fusion averages away conflict. A +0.9 signal and a -0.8 signal produce a +0.05 combined score — near neutral — which obscures the fact that two strong, opposite signals exist. The user sees "uncertain" but doesn't see "two strong signals violently disagree." This violates Sentinel's explainability principle.

**Calibration quality:** **Good, with effort.** Linear scores can be calibrated to probabilities via logistic calibration (Platt scaling) or isotonic regression. However, calibration degrades as individual signal accuracy changes, requiring continuous recalibration.

**Key weakness:** The "garbage in, garbage out" problem. If a user or automated system sets weights for signals based on outdated accuracy assumptions, the fusion produces confidently wrong scores. Weight configuration is a maintenance burden and a source of systematic error. Linear fusion also cannot model conditional dependencies — if Signal A and Signal B are highly correlated, a linear model double-counts their shared information.

---

### Approach C: Decision Tree / Rule-Based Combination

**How it works:** Transparent precedence rules determine how signals combine. Example: "If C2PA is valid with recognized CA → HIGH CONFIDENCE AUTHENTIC, regardless of other signals. If C2PA is absent and frequency analysis detects artifacts → MODERATE CONFIDENCE AI-GENERATED, pending additional checks. If text classifier and publisher reputation both flag → LOW CONFIDENCE AI-GENERATED, with explicit caveats."

**Mathematical honesty:** **Excellent.** There is no math to misrepresent. Each rule is an explicit conditional. The system's behavior is deterministic and fully auditable. "We reached this conclusion because Rule 3 applied: [conditions] → [result]."

**Explainability:** **Excellent.** This is the most explainable approach. The reasoning is a trace through a decision tree that users could theoretically inspect. "Your image triggered Rule 2a: valid provenance with recognized camera → probable authentic. No other rules modified this assessment."

**Robustness to missing signals:** **Excellent.** Rules are inherently conditional on signal presence. "If C2PA present AND valid → apply Rule 1. If C2PA absent → fall through to Rule 5." Missing signals trigger different paths, not awkward zero-filling.

**Handling conflicting signals:** **Good — with explicit design.** Conflict handling is a first-class design decision in rule-based systems. Rules can explicitly flag conflict: "C2PA valid (Rule 1) BUT pixel artifacts detected (Rule 7) → CONFLICT: This content has valid provenance but shows AI generation artifacts. Possible explanations: AI generation with retroactive signing, legitimate editing that introduced artifacts, or false positive in artifact detection."

**Calibration quality:** **Poor by default, improvable.** Rule thresholds must be set by engineering judgment, not learned from data. It's easy to produce rules that are intuitively reasonable but poorly calibrated. "We only trigger the deepfake rule when 3+ frames show face-swap artifacts" — is 3 the right number? This requires empirical tuning that starts to erode the simplicity benefit.

**Key weakness:** Brittleness. Rules that make sense in 2026 may become wrong in 2028 as the threat landscape changes. A rule like "C2PA always overrides detection signals" is correct today but may become wrong if C2PA certificate authorities are compromised or if detection signals improve dramatically. Rules require continuous human review and are expensive to maintain as the number of signals grows (rule interactions become combinatorially complex).

---

### Approach D: Conformal Prediction

**How it works:** Rather than outputting a single confidence score, conformal prediction outputs a **prediction set** with a statistical guarantee. Given a specified error rate (α, e.g., 0.05), the system outputs "with 95% confidence, this content is one of: {authentic, uncertain}" or "{authentic}" or "{AI-generated}" or "{authentic, AI-generated}" (the last meaning "we cannot distinguish at this confidence level"). The guarantee holds under minimal assumptions: the calibration data must be exchangeable with future data.

**Mathematical honesty:** **Excellent.** Conformal prediction provides the strongest mathematical guarantee available in machine learning: a bound on error rate that holds under exchangeability. Unlike Bayesian or linear methods, the error guarantee is not contingent on distributional assumptions being correct. If the guarantee says "95%," the error rate genuinely cannot exceed 5% (subject to exchangeability).

**Explainability:** **Moderate to Poor.** Conformal prediction sets are mathematically clean but difficult to explain to non-technical users. "There's a 95% guarantee that this content is either authentic or we can't tell" is harder to interpret than "78% confidence." The prediction set abstraction is less intuitive than a single number, even if it's more honest.

**Robustness to missing signals:** **Moderate.** Conformal prediction requires a scoring function that produces a single "nonconformity score" from all available signals. This scoring function could be any of the approaches above (Bayesian, linear, rule-based). Missing signals are handled by whatever the underlying scoring function does.

**Handling conflicting signals:** **Poor.** Conformal prediction's output is a set — it does not inherently surface WHY the set is wide. A broad prediction set ({authentic, AI-generated, uncertain}) tells the user "we can't tell" but doesn't explain that it's because two strong signals are in conflict.

**Calibration quality:** **Excellent (within the guarantee).** The error guarantee is mathematically proven, not empirically estimated. However, the guarantee's value depends on the user's interpretation. "95% of assessments will have the truth inside the prediction set" does not mean "each individual assessment is 95% accurate." It means the system will rarely exclude the true label from the set.

**Key weakness:** The exchangeability assumption. The guarantee holds only if future data is exchangeable with the calibration set. In AI detection, this assumption is routinely violated — new AI models produce content from different distributions than the calibration set. Conformal prediction cannot maintain its guarantee when the data distribution shifts (see Section 6). The guarantee also says nothing about "conditional coverage" — it may be 95% accurate overall but 60% accurate on challenging cases.

---

### Recommended Approach: **Bayesian Evidence Accumulation with Explicit Conflict Flagging**

**Justification:**

| Criterion | Winner | Why |
|-----------|--------|-----|
| Mathematical honesty | Bayesian / Conformal (tie) | Bayes' rule is the correct update mechanism; conformal provides guarantee bounds |
| Explainability | Bayesian / Rule-Based (tie) | Bayesian decomposes contributions naturally; rules are inherently transparent |
| Robustness to missing signals | Bayesian / Rule-Based (tie) | Both handle missing signals gracefully without awkward zero-filling |
| Handling conflicting signals | Rule-Based / Bayesian with conflict flags | Rules handle conflict explicitly; Bayesian needs added conflict detection |
| Calibration quality | Conformal / Bayesian | Conformal guarantees bounds; Bayesian calibrates to probabilities |
| Maintenance burden | Bayesian | Rules need continuous human review; Bayesian learns weights from calibration data |

**Recommended fusion architecture:**

1. **Primary engine: Bayesian evidence accumulation.** Each signal provides a calibrated likelihood ratio. Starting from a content-type-specific prior, the posterior updates with each available signal. The system computes both the posterior mean and a credible interval.

2. **Conflict detection layer: Conflict flagging.** After Bayesian fusion, a separate conflict detection mechanism identifies when signals disagree beyond a threshold. Conflicting assessments are not smoothed out — they are surfaced to the user as "Conflicting evidence: [Signal A suggests X, Signal B suggests Y]" (see Section 5).

3. **Fallback predicate rules: Safety overrides.** Certain hard rules must override Bayesian fusion:
   - Valid C2PA from a recognized hardware-attested camera with an unbroken chain → minimum floor on authenticity confidence, regardless of detection signals.
   - Known synthetic signature (hash-level match) → minimum floor on AI-generation confidence.
   - These are not "C2PA always wins" — they are "when the highest-certainty signals fire, the system enforces a floor on the confidence range."

4. **Calibration: Conformal prediction wrapper.** For enterprise and API users who need guaranteed error bounds, an optional conformal prediction layer wraps the Bayesian output. Consumer users see Bayesian probabilities with confidence intervals; enterprise users can request conformal prediction sets with guaranteed coverage.

---

## 4. Uncertainty Representation

Sentinel must communicate what it knows and what it doesn't know. An overconfident wrong answer is worse than an uncertain right answer. The following approaches are evaluated for how to represent uncertainty honestly.

### Approach A: Confidence Intervals

**Representation:** "We estimate this content is authentic with 67-82% confidence" (rather than a single number).

**Evaluation:**
- **Intuitive:** Users understand ranges from weather forecasts, polls, and financial projections. "60% chance of rain" is a well-established mental model.
- **Reduces over-reliance on point estimates:** A single "78%" feels precise; "67-82%" feels appropriately uncertain.
- **But:** Ranges can be misleading if not properly calibrated. A 67-82% interval should be an 85% credible interval, meaning the true value falls within this range 85% of the time. Producing calibrated intervals requires the Bayesian posterior variance to be accurate.
- **Visual design concern:** Confidence intervals require more screen space and are harder to scan quickly than a single number. Progressive disclosure: show the midpoint with an uncertainty indicator expandable to the full interval.

### Approach B: Conformal Prediction Sets

**Representation:** "With 95% probability, this content is one of: {authentic, uncertain}"

**Evaluation:**
- **Mathematically strongest guarantee:** The error rate is bounded, not estimated.
- **But:** Prediction sets are difficult for non-technical users to interpret. "{authentic, uncertain}" reads as "we don't know" to most users, even though it's a precise statistical statement.
- **Use case fit:** Conformal prediction sets are most appropriate for enterprise/API users who need statistical guarantees for automated decision-making. Consumer users need the Bayesian confidence interval.

### Approach C: Ensemble Disagreement as Uncertainty Metric

**Representation:** Not a single uncertainty measure, but an observation: "Individual detection signals disagree on this content."

**Evaluation:**
- **Honest:** Reflects genuine model uncertainty rather than statistical abstraction.
- **Interpretable:** Users understand "our methods disagree" better than "the posterior variance is 0.13."
- **But:** Ensemble disagreement only captures one type of uncertainty (model uncertainty). It cannot capture inherent data uncertainty (cases where even perfect models would disagree). It also requires maintaining multiple diverse detectors for each signal type.
- **Recommendation:** Use as a secondary uncertainty indicator alongside confidence intervals. "Signal agreement: 3 of 5 signals concur" provides useful context.

### Approach D: Signal Count / Signal Quality Ratio

**Representation:** "3 of 5 signals agree, but 2 strongest signals disagree" or "4 signals available, all low-quality — treat assessment with caution."

**Evaluation:**
- **Most honest framing:** Acknowledges that confidence is a function of both signal quantity and signal quality.
- **Avoids false precision:** "We have limited information" is more honest than "42% confidence" when the 42% is based on two weak, correlated signals.
- **But:** Signal count is a poor proxy for confidence. One C2PA verification is stronger than 5 weak text signals combined. Signal count must always be paired with signal quality tier information.
- **Recommendation:** Use as the primary explainability framing. Show users the signal quality distribution: "Strong signals: 1 valid (C2PA). Moderate signals: 2 supporting. Weak signals: 1 conflicting."

### Approach E: Temporal Confidence Decay

**Representation:** For time-sensitive content (breaking news, election claims, viral posts), confidence should decay with elapsed time since the assessment was made. "Confidence assessment from 2 hours ago: 78%. This content relates to a rapidly developing event — confidence may have changed."

**Evaluation:**
- **Addresses a real failure mode:** A perfectly calibrated assessment made at T=0 may be wrong at T=+6 hours because new evidence emerged, the content was edited, or the source's behavior changed.
- **But:** Temporal decay adds complexity. How fast should confidence decay? Linear? Exponential? Content-type-dependent? News decays faster than documentary photography.
- **Recommendation:** Use for time-sensitive content categories. Apply a simple recency warning — not a mathematical decay function. "This assessment is [X time] old. For this type of content, assessments older than [Y time] should be treated as approximate." The Trust Engine should support re-analysis on demand.

### Recommended Uncertainty Representation

**For consumer users (browser extension):**
- **Primary:** Bayesian credible interval (confidence range) with a central tendency: "Likely authentic (67-82% confidence)"
- **Secondary:** Signal agreement/disagreement summary: "Strong signals agree (1/1). Moderate signals: 2 agree, 1 disagrees."
- **Tertiary:** Recency timestamp with content-category-specific aging note

**For professional users (explainability panel):**
- Full Bayesian posterior visualization: probability distribution over {authentic, uncertain, AI-generated} categories
- Per-signal likelihood ratios with individual confidence intervals
- Signal quality tier breakdown
- Recency and staleness indicators

**For enterprise/API users:**
- Bayesian credible intervals
- Optional conformal prediction set with coverage guarantee
- Raw per-signal scores for integration into enterprise workflows

---

## 5. Conflicting Evidence Handling

When signals disagree — C2PA says authentic, pixel analysis says AI-generated — the Trust Engine must not smooth the disagreement into an uninformative average. Conflict is itself valuable information.

### Approach A: Signal Precedence (Hard Rules)

**How it works:** When signals conflict, a fixed precedence order resolves the conflict. "C2PA always wins over pixel analysis." "Hardware attestation always wins over text burstiness."

**Evaluation:**
- **Simple and predictable:** Users and developers can understand the rules.
- **But:** Wrong when the dominant signal is wrong. A compromised CA issuing certificates to bad actors would make C2PA the wrong signal to trust. A false positive in C2PA (valid cert from bad actor) could override a correct detection signal.
- **But:** Prevents learning. If pixel analysis improves to 95%+ accuracy while C2PA certificate infrastructure weakens, hard precedence rules prevent the system from adapting.

### Approach B: Weighted Conflict Resolution

**How it works:** Both conflicting signals are visible in the confidence assessment. The weighted average appears as the confidence score, but the disagreement is not explicitly highlighted. The user sees "52% confidence" — an average of +90% (C2PA authentic) and -85% (pixel analysis AI-generated).

**Evaluation:**
- **Mathematically clean:** Produces a single number.
- **But:** Actively misleading. "52% confidence" implies the system is uncertain in a boring, information-poor way — when in reality, the system has two very strong, very contradictory pieces of evidence. This is the most interesting and important kind of assessment, and it should be surfaced as such.
- **Verdict: Do not use weighted conflict resolution alone.**

### Approach C: Conflict Flagging (Explicit Surfacing)

**How it works:** When signals disagree beyond a threshold, the system flags the conflict explicitly: "CONFLICT: This content has valid C2PA provenance indicating authentic capture, but pixel-level analysis detected AI generation artifacts at high confidence. These signals are in disagreement. Possible explanations include: AI-synthesized content retroactively signed, edited content where editing introduced artifacts, or a false positive in either signal."

**Evaluation:**
- **Most honest:** Does not pretend the system knows which signal is correct.
- **Most useful to users:** A conflict is actionable in a way that a bland "52% confidence" is not. The user knows to investigate further, seek additional evidence, or apply their own judgment.
- **Most aligned with Sentinel's philosophy:** "Evidence, not certainty."
- **But:** Requires additional UX to help users navigate conflict. What should a journalist do when C2PA and pixel analysis disagree? The system should provide suggested next steps (see below).
- **Risk:** Too many conflicts could erode trust. If 30% of assessments show conflicts, users learn to ignore them. Conflict flagging must be combined with severity tiers.

### Approach D: Tiered Trust

**How it works:** Trust in signals varies by context. "Trust provenance over detection for known CAs." "Trust detection over provenance for unknown/new CAs." "Trust hardware attestation over everything." This is signal precedence with context-dependent thresholds.

**Evaluation:**
- **More nuanced than pure precedence:** Acknowledges that C2PA from Adobe (a known, audited CA) is different from C2PA from an unknown entity.
- **But:** Requires maintaining a trust registry of CAs, hardware manufacturers, and signing entities. This is maintenance-intensive and creates a new attack surface (the trust registry itself could be poisoned).
- **But:** Creates a two-class system where content signed by "known" entities gets preferential treatment — potentially reinforcing existing power structures in media and publishing.

### Recommended Approach: **Tiered Trust with Conflict Flagging (Hybrid)**

1. **Tier 1 signal dominance with context-dependence:**
   - Valid C2PA with cert chain from a recognized, audited CA → minimum floor on authenticity confidence, regardless of conflicting detection signals. The C2PA signal CAN be overridden only by: (a) hardware-level known synthetic signature match (Tier 2, Rank 4), or (b) definitive cross-modal physics violation (Tier 2, Rank 3).
   - Hardware-attested provenance (Leica M11-P, Sony Alpha) → absolute floor on authenticity. Cannot be overridden by any detection signal. The only override is CA revocation or detection of physical tampering with the secure element.
   - When Tier 1 and a Tier 2 signal conflict HARD (both >80% contradictory confidence): flag conflict explicitly rather than defaulting to precedence.
   - When Tier 1 and Tier 3/4 signals conflict: Tier 1 wins. Tier 3/4 signal is noted but does not modify the assessment.

2. **Conflict tiering:**
   - **Tier 1 vs. Tier 1 conflict** (e.g., two conflicting C2PA manifests): Critical conflict. Surface prominently. "Content has conflicting provenance claims."
   - **Tier 1 vs. Tier 2 conflict** (e.g., C2PA valid, pixel analysis says AI): High conflict. Surface with explanation. "Valid provenance but detection signals disagree."
   - **Tier 2 vs. Tier 2 conflict** (e.g., frequency analysis says AI, watermark says authentic): Moderate conflict. Note disagreement, but confidence reflects the averaged evidence.
   - **Tier 3/4 vs. anything:** Not flagged as conflict. Lower-tier signals do not create conflicts with higher-tier signals — they are simply noted and outweighed.

3. **User guidance on conflict:** Every conflict flag includes suggested next steps:
   - "Consider seeking additional evidence: verify with the content creator, check for other copies online, examine metadata for editing history."
   - "If this content is critical to your decision, we recommend independent verification."
   - "Report this assessment if you have ground truth information."

4. **Trust registry transparency:**
   - The list of "recognized CAs" and "trusted hardware manufacturers" must be public, versioned, and independently auditable.
   - Users can see WHY a certificate was trusted ("Issued by DigiCert, which has been audited and has zero recorded compromises" vs. "Issued by unknown CA — treat provenance with reduced confidence").
   - The trust registry must support revocation and downgrade. A CA that issues bad certificates must be promptly downgraded.

---

## 6. Model Degradation & Evolution

AI detection models degrade 10-20% per new model generation. The evidence combination logic must remain stable even as individual signal accuracy fluctuates. The Trust Engine's architecture must be designed for a world where every detection model has an expiration date.

### 6.1 The Degradation Problem

**Quantified degradation:** When StyleGAN3 replaced StyleGAN2, detection accuracy for StyleGAN2-trained detectors dropped from ~85% to ~60%. When a new LLM generation is released, text detectors trained on the previous generation degrade from ~70% to near-random. This pattern repeats with every major generative model release.

**Degradation is asymmetric:** Provenance signals (C2PA, hardware attestation) do not degrade. Detection signals (frequency analysis, vocoder artifacts, text classifiers) do. The Trust Engine's fusion logic must be designed so that detection signal degradation does not corrupt confidence assessments that rely primarily on provenance.

### 6.2 Strategies for Stability

**Strategy 1: Weight signals by durability, not just accuracy.**

A detection model that achieves 90% accuracy today but will degrade to 60% in 12 months should carry less weight than a provenance signal that achieves 99% accuracy permanently. Weights should reflect **expected accuracy over the signal's useful lifetime,** not snapshot accuracy.

**Implication:** The Tier hierarchy (Section 1) already encodes durability — Tier 1 signals persist indefinitely; Tier 2 signals degrade slowly; Tier 3 signals degrade quickly. The Bayesian fusion framework should use expected accuracy (with a degradation projection) rather than current accuracy when computing likelihood ratios.

**Strategy 2: Per-signal recalibration, not per-signal retraining.**

When a detection model degrades, the solution is not necessarily to retrain the model (expensive, slow). It is to **recalibrate the likelihood ratio** for that signal. If pixel artifact detection drops from 85% to 70% accuracy, the Bayesian prior should be adjusted so that this signal contributes proportionally less to the posterior. The model stays the same; its weight in the fusion changes.

**Implication:** The fusion layer must support dynamic weight adjustment without modifying the detection models. Accuracy benchmarks on new generator outputs should trigger automatic weight decay for affected signals.

**Strategy 3: Ensemble diversity as degradation insurance.**

If all detection models use the same underlying architecture (e.g., all image detectors are DCT-based frequency analyzers), they degrade together when generators change their frequency profiles. Ensemble diversity across detection approaches (DCT-based, CNN-based, transformer-based, physics-based) ensures that some signals degrade while others remain stable.

**Implication:** For each modality, at least two detection signals should use fundamentally different approaches. This is not about accuracy — it's about uncorrelated failure modes.

**Strategy 4: Tier 1 signals as stability anchors.**

C2PA verification and hardware attestation do not degrade. Well-formed Bayesian priors should weight these signals so heavily that even complete failure of all detection signals cannot produce a completely wrong confidence assessment on content with valid provenance. The worst case with missing/destroyed detection signals should be "uncertain, but provenance data is available" — not "confident and wrong."

**Strategy 5: Continuous calibration monitoring.**

The Trust Engine must track each signal's real-world accuracy against a ground truth validation set that is regularly updated with content from new AI models. When a signal's empirical accuracy diverges from its assumed accuracy by more than a threshold, the system should:
- Automatically reduce that signal's weight in the fusion
- Flag the signal for human review / recalibration
- Notify users that "detection model accuracy has degraded for newer content types"

**Strategy 6: Model versioning and temporal transparency.**

Every assessment should record which versions of which models produced which scores. When models are updated, old assessments should not be retroactively changed. Users should see: "Assessed on [date] using [model version]. This model has since been superseded by [new version]. Re-analysis may produce different results."

### 6.3 Evolution Without Revolution

The Trust Engine's fusion architecture should remain stable even as individual signals are replaced. This requires:

- **Signal interface stability:** A "pixel artifact detector" is a signal that takes an image and returns a calibrated likelihood ratio. Whether that detector is DIRE v1, DIRE v3, or a completely different architecture is irrelevant to the fusion layer.
- **Weight/calibration externalization:** Signal weights and calibration parameters should be data, not code. Updating a weight or swapping a calibration curve should not require changes to the fusion engine.
- **A/B signal evaluation:** Before a new detection model replaces an existing one, it should run in shadow mode — evaluated against the validation set and compared to the existing model's performance — before contributing to live assessments.

---

## 7. Calibration Philosophy

### 7.1 What "Calibrated" Means

A confidence assessment is **calibrated** if, among all content assigned "X% confidence" of a given label, approximately X% of that content genuinely belongs to that label. If the system says "78% likely authentic" for 1,000 pieces of content, approximately 780 of them should be authentic.

**Why calibration matters more than raw accuracy:** An 85% accurate detector that reports "99% confidence" on every assessment is actively harmful — it's confidently wrong 15% of the time. A 70% accurate detector that correctly reports "65-75% confidence" is useful — the user knows the system's limitations and can act accordingly.

### 7.2 Calibration vs. Discrimination

- **Discrimination** is the ability to separate authentic from AI-generated content (measured by AUROC).
- **Calibration** is the alignment between confidence scores and actual outcomes (measured by Expected Calibration Error).
- They are independent properties. A detector can have good discrimination but terrible calibration, or poor discrimination but perfect calibration.

**Sentinel's calibration standard:** The Trust Engine must be well-calibrated even if individual signals have poor calibration. Bayesian fusion with calibrated likelihood ratios enforces this: if each signal's likelihood ratio is calibrated, the posterior is calibrated. Sentinel invests calibration effort at the signal level so that fusion-level calibration follows naturally.

### 7.3 Calibration Targets

| Confidence Tier | Target Calibration | Max ECE (Expected Calibration Error) |
|-----------------|-------------------|--------------------------------------|
| >90% ("Very High Confidence") | Well-calibrated | ECE < 0.05 |
| 70-90% ("High Confidence") | Well-calibrated | ECE < 0.05 |
| 50-70% ("Moderate Confidence") | Well-calibrated | ECE < 0.08 |
| <50% ("Low Confidence / Uncertain") | Looser tolerance acceptable | ECE < 0.12 |

High-confidence assessments (which users are most likely to act on) must be tighter calibrated than low-confidence assessments (which users treat as informational).

### 7.4 Never Report 100% or 0% Confidence

**Rule:** The Trust Engine must never output 0% or 100% confidence for any assessment. These imply logical certainty, which is impossible given:
- Detection signals are probabilistic, not deterministic
- Even C2PA verification, while deterministic in signature validation, relies on the assumption that the signer is honest (non-deterministic)
- The system can always be wrong

The maximum reportable confidence should be 99.5% (acknowledging the infinitesimal but nonzero possibility of error). The minimum should be 0.5%.

---

## 8. Summary of Recommendations

### Evidence Hierarchy

Trust Engine must weight evidence by tier. Tier 1 signals (cryptographic provenance, hardware attestation) dominate when present. Tier 2 signals (cross-modal consistency, frequency analysis, watermark detection) provide strong corroboration but degrade with model evolution. Tier 3 signals (face-swap detection, audio artifacts, metadata, reverse search) are useful only in combination. Tier 4 signals (text analysis, reputation, community signals, absence of provenance) carry minimal weight and must never stand alone.

### Signals That Must Never Be Used Alone (or At All)

Model-specific text classifiers must never be used at all — mathematical proof of convergence to random. Short text, non-native English classification, and binary "AI/Human" verdicts are prohibited. Seven additional signal types must never serve as standalone signals (see Section 2.5).

### Fusion Approach

Bayesian evidence accumulation is the recommended primary fusion method, supplemented by explicit conflict flagging for contradictory signals, safety override rules for Tier 1 signals, and an optional conformal prediction wrapper for enterprise users who need statistical guarantees.

### Uncertainty Representation

Consumer users see Bayesian credible intervals with signal agreement summaries. Professional users see the full posterior with per-signal decomposition. Enterprise users have access to conformal prediction sets with guaranteed coverage.

### Conflicting Evidence

Conflict is surfaced explicitly, not smoothed. A tiered precedence model resolves most conflicts transparently. Unresolvable conflicts (Tier 1 vs. Tier 1, or Tier 1 vs. strong Tier 2) are flagged with context and suggested next steps. Users are never left with a conflict-free but misleading average.

### Model Evolution

The Trust Engine is designed for continuous signal degradation. Weights reflect durable accuracy, not snapshot accuracy. Calibration is per-signal and continuously monitored. The fusion architecture remains stable while individual models are hot-swapped. Tier 1 signals provide stability anchors that prevent detection signal collapse from producing confident wrong answers.

---

## Appendix A: Mathematical Context References

- **Sadasivan et al. (2023),** "Can AI-Generated Text be Reliably Detected?" — TMLR. The formal proof that text detection AUROC → 0.5 as language models approach the human text distribution.
- **Perkins et al. (2024),** "GenAI Detection Tools" — IJETHE. Empirical survey confirming inconsistent, degrading detection tool accuracy across models and languages.
- **Liang et al. (2024),** "GPT detectors are biased against non-native English writers" — Patterns. Demonstrated 70% false positive rate on non-native English text.
- **C2PA Specification v2.4** — Cryptographic provenance standard. Verification is deterministic; trust model is transitive via PKI.
- **SynthID (Google DeepMind)** — Proprietary watermarking. Not independently verifiable. Detection requires Google cooperation.

---

## Appendix B: Key Terms for Trust Engine

| Term | Definition |
|------|-----------|
| **Likelihood ratio** | P(evidence \| AI-generated) / P(evidence \| authentic). The Bayesian contribution of a single signal. |
| **Prior probability** | The base rate of AI-generated content for a given content type and context, before any signals are evaluated. |
| **Posterior probability** | The updated confidence after incorporating all available evidence signals via Bayes' rule. |
| **Credible interval** | A Bayesian range within which the true classification probability falls with specified certainty (e.g., 67-82% at 85% credibility). |
| **Calibration** | Alignment between stated confidence and actual outcome frequency. A "78% confidence" assessment should be correct ~78% of the time. |
| **ECE (Expected Calibration Error)** | A metric measuring the gap between stated confidence and empirical accuracy across confidence bins. |
| **Conformal prediction set** | A set of possible classifications with a statistical guarantee that the true class is in the set at a specified rate. |
| **Exchangeability** | The assumption that calibration data and future data come from the same distribution. Required for conformal prediction guarantees. |
| **Ensemble diversity** | The degree to which different detection models make uncorrelated errors. Higher diversity → more robust fusion. |
| **Conflict flagging** | Explicit surfacing of signal disagreement to users, rather than averaging away contradictions into a single score. |

---

*End of Trust Engine Product Reasoning Specification. This document defines the logical model for evidence combination, uncertainty, and confidence. Implementation decisions (language, framework, API design, data structures) are deferred to ADR-003: Trust Engine Architecture.*
