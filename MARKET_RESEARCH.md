# MARKET_RESEARCH.md — Sentinel Market Analysis

> **Project:** Sentinel | **Phase 0 Discovery**
> **Note:** This document consolidates Phase 0 market research findings. Full research available in `research/` directory.

---

## Market Landscape

### The Problem

AI-generated content now equals or exceeds human-generated content in many online domains. By mid-2026, the average internet user encounters synthetic text, images, video, and audio daily without reliable tools to distinguish authentic from generated content.

### Market Drivers

1. **AI Content Proliferation** — Copyleaks research confirms ~50/50 AI-to-human content split online. Europol's "90% by 2026" projection, while hyperbolic, is directionally accurate.
2. **Regulatory Mandates** — EU AI Act requires AI content labeling. US executive orders mandate watermarking. C2PA adoption is accelerating as a compliance mechanism.
3. **Platform Liability** — Publishers and platforms face increasing pressure to verify content before publication.
4. **Deepfake Fraud** — Synthetic media is being used for financial fraud, political manipulation, and impersonation at growing scale.
5. **Search Engine Penalties** — Google's 2025 Search Quality Rater Guidelines penalize unlabeled AI content, creating demand for verification tools before publishing.

---

## Competitor Landscape

### Direct Competitors (AI Detection)

| Competitor | Users | Approach | Weakness |
|-----------|-------|----------|----------|
| **GPTZero** | 17M+ users, 1M+ educators | AI text detection, Chrome extension, LMS integration | Binary scores, no provenance, no explainability |
| **Originality.ai** | Enterprise (Stanford, Harvard, NYT, Reuters) | Content Integrity Suite (AI + Plagiarism + Fact Check) | Dashboard-first, not browser-first. Credit-based pricing. |
| **Turnitin** | 16,000+ institutions | Embedded in LMS (Canvas, Moodle, Blackboard) | Institution-only, no consumer product |
| **Copyleaks** | 500+ enterprise/edu, 30M+ monthly scans | Multi-modal (text, image, video, code), 30+ languages | Complex credit-based pricing, compliance positioning |
| **Winston AI** | 10M+ users | Highest claimed accuracy (99.98%), Chrome + WordPress integration | Less enterprise credibility |

### Adjacent (Provenance & Trust)

| Player | Approach | Gap |
|--------|----------|-----|
| **C2PA / Content Authenticity Initiative** | Open cryptographic provenance standard. 500+ member companies. | No consumer browser extension surfaces Content Credentials during browsing |
| **Truepic** | Proprietary secure camera + AI manipulation detection | Enterprise-only. Not a consumer product. |
| **Google SynthID** | Watermarking for all modalities (Gemini, Veo, Lyria) | Proprietary. Not independently verifiable. No consumer tool. |
| **NewsGuard** | Human-rated source reliability (36,000+ sources), browser extension | Only covers news sources. No AI detection. No provenance. Slow to scale. |

---

## Competitive Gaps (Sentinel's Opportunity)

1. **No unified trust score exists.** Every competitor does one thing: AI detection OR source rating OR fact-checking. No one combines all signals into a single explainable assessment.

2. **No consumer provenance verification.** C2PA infrastructure exists but no browser extension surfaces Content Credentials during browsing. This is unclaimed territory.

3. **Binary detection is mistrusted.** Users and institutions increasingly recognize that "87% AI-generated" without evidence is meaningless. Explainability is a genuine differentiator.

4. **No browser extension combines** AI detection + provenance verification + source credibility + fact corroboration. Users would need 3-5 separate extensions for equivalent coverage.

5. **"Trust Layer" positioning is unclaimed.** Every competitor calls themselves an "AI detector" — a shrinking, commoditizing category.

---

## User Segments

| Segment | Need | Willingness to Pay | Priority |
|---------|------|-------------------|----------|
| **Journalists** | Verify source images/video before publication with citable evidence | High (institutional budget) | P0 (MVP) |
| **Educators** | Assess student work originality with explainable evidence for academic integrity boards | High (institutional budget) | P0 (MVP) |
| **Content Marketers** | Verify freelance submissions before publication. Protect SEO from AI penalties. | Medium (existing tool budget) | P1 (Phase 3) |
| **Enterprise T&S** | Programmatic trust signals at scale for moderation decisions | Very High (enterprise contracts) | P2 (Phase 3-4) |
| **General Consumers** | Quick, effortless trust assessment while browsing | Low (expects free) | P3 (Phase 4) |

---

## Adoption Challenges

1. **Browser extension fatigue** — Average user has 5-10 extensions, actively uses 2-3. New extension must justify itself in the first session.
2. **Trust barrier** — "Why should I trust Sentinel's assessment?" Solved through: open-source client, explainability, institutional partnerships, never overclaiming.
3. **Privacy concerns** — Core personas (journalists, educators) are the MOST privacy-conscious users. Local-first architecture is not optional — it's a competitive requirement.
4. **The "so what?" problem** — Confidence scores without actionable outcomes are a curiosity, not a product. MVP must provide concrete action paths per persona.

---

## Key Market Risks

| Risk | Severity |
|------|----------|
| Google/Apple/Microsoft build trust features natively into browsers | High |
| AI detection becomes mathematically unsolvable (converges to random) | High |
| Consumer demand doesn't materialize (B2B path is the fallback) | Medium |
| C2PA adoption slower than projected | Medium |

---

*End of MARKET_RESEARCH.md. Full competitor analysis and pricing research available in Phase 0 agent outputs.*
