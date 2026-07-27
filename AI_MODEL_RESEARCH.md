# AI_MODEL_RESEARCH.md — Sentinel ML Strategy

> **Project:** Sentinel | **Phase 0 Discovery**
> **Note:** This document summarizes ML research findings. Full technical research in Phase 0 agent outputs.

---

## ML Philosophy

Sentinel does not build AI detection models. Sentinel **combines evidence signals** from multiple sources — some of which are ML-based. The Trust Engine reasons about evidence; it does not perform detection directly.

This separation is intentional: ML models degrade, become obsolete, and require continuous retraining. The Trust Engine's fusion logic remains stable as individual models are replaced.

---

## Planned ML Components

### Image Forensics

| Model Type | Purpose | Runtime | Accuracy | Limitations |
|-----------|---------|---------|----------|-------------|
| **Frequency domain artifact detection** | Detect VAE decoder artifacts, GAN fingerprints in AI-generated images | ONNX (5-20MB, WASM/WebGPU) | 75-90% on known generators | Degrades 10-20% per new model generation. Post-processing reduces accuracy. |
| **Noise pattern analysis** | Identify unnatural noise patterns inconsistent with camera sensor noise | ONNX (WASM) | Moderate-High | Requires same-origin pixel access (CORS constrained) |
| **Error Level Analysis (ELA)** | Detect splicing and editing via compression inconsistency | JavaScript (lightweight) | Moderate | Not AI-generation-specific |
| **AI generator fingerprint detection** | Detect telltale fingerprints from specific generators (DALL-E, Midjourney, SD) | ONNX (5-20MB) | 75-90% on known generators | Must be updated as new generators emerge |

### Text Analysis (Weak Corroborating Signal Only)

| Model Type | Purpose | Runtime | Accuracy | Limitations |
|-----------|---------|---------|----------|-------------|
| **Perplexity/burstiness analysis** | Zero-shot statistical analysis of writing patterns | ONNX (small LM ~50MB) or pure JS statistics | LOW-MEDIUM | Mathematically proven to converge to random as LLMs improve (Sadasivan et al. 2023). Biased against non-native English. NEVER a standalone signal. |

### Audio Analysis (Chrome/Edge, User-Invoked)

| Model Type | Purpose | Runtime | Accuracy | Limitations |
|-----------|---------|---------|----------|-------------|
| **Vocoder artifact detection** | Detect neural vocoder artifacts in raw audio | ONNX (10-30MB) | 80-90% (raw source) | Degrades to near-random after telephony compression |

### Video Analysis (Chrome/Edge, User-Invoked)

| Model Type | Purpose | Runtime | Accuracy | Limitations |
|-----------|---------|---------|----------|-------------|
| **Face-swap detection** | Detect deepfake face-swaps via temporal consistency + physiological signals | ONNX (WASM/WebGPU) | 80-90% (face-swaps) | Does NOT work on fully AI-generated video (Sora, Runway) |

### Cross-Modal Analysis

| Model Type | Purpose | Runtime | Accuracy | Limitations |
|-----------|---------|---------|----------|-------------|
| **Image-text semantic consistency** | Detect gross mismatches between image content and surrounding text | ONNX (CLIP-ViT ~150MB) or cloud | LOW-MEDIUM | Computationally heavy. Good for egregious mismatches only. |

---

## Model Selection Criteria

1. **Must run in browser** — ONNX Runtime Web with WASM backend (WebGPU acceleration where available)
2. **Must fit in IndexedDB** — Models >50MB are deferred to cloud or Phase 3+
3. **Must produce calibrated confidence** — Output must include accuracy estimate, not just classification
4. **Must degrade gracefully** — Model weight automatically decays as accuracy declines against new generators
5. **Must be signed** — All model weights cryptographically signed and verified before loading

---

## Update Strategy

- **ML-based models decay 10-20% per new model generation.** Continuous recalibration required.
- **Fusion weights are adjusted, not models retrained.** As a signal degrades, its weight in Bayesian fusion is reduced. The model itself may or may not be updated.
- **Shadow evaluation for new models.** Before a new model version replaces an existing one, it runs in evaluation mode for 1,000+ analyses. Results compared against existing model before promotion.
- **Reference data (known hashes, domain lists) refreshed periodically** via signed bundles.

---

## Privacy Considerations

- All Tier 1 forensic models run locally. Content never leaves the device.
- Model weights are downloaded once, cached in IndexedDB, verified by signature.
- Cloud-based ML inference is Tier 3 — requires explicit per-analysis opt-in.
- Chrome built-in AI APIs (Gemini Nano) used when available — zero model download overhead, on-device only.

---

## Key Research Finding

**Text detection is fundamentally unreliable as models improve.** The mathematical proof (Sadasivan et al. 2023, TMLR) shows that as language models approach human text distribution, AUROC of any detector converges to 0.5 (random). This is not an engineering limitation — it's a mathematical inevitability.

Sentinel's architecture accounts for this: text analysis is a weak corroborating signal (Tier 4), capped at 10% contribution, and never used as a standalone assessment.

---

*End of AI_MODEL_RESEARCH.md. Full ML research available in Phase 0 agent outputs.*
