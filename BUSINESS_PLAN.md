# BUSINESS_PLAN.md

> **Project:** Sentinel | **Phase:** 0 – Discovery & Research
> **Document:** Business Strategy & Monetization Research
> **Owner:** Byrd Ventures LLC
> **Version:** 1.0

---

## Executive Summary

Sentinel is positioned as a **"Trust Layer for the Internet"** — an infrastructure play, not an AI detector. This positioning matters enormously for commercial viability. Companies that sell "AI detection" compete in a crowded, price-sensitive market ($13–$180/mo). Companies that sell **trust infrastructure** compete in a market with far higher ceilings.

The key commercial insight: **Sentinel's explainable, multi-signal confidence model is a structural differentiator** in a market where competitors provide black-box scores with no provenance. This differentiator should anchor every monetization decision.

---

## 1. Competitive Landscape & Existing Pricing

### 1.1 Direct Competitors (AI Detection Tools)

| Company | Personal | Pro/Team | Enterprise | Model |
|---|---|---|---|---|
| **Originality.ai** | $12.95/mo (2K credits) | — | $136.58/mo (15K credits) | Credit-based subscription |
| **GPTZero** | Free tier | Undisclosed (contact sales) | Undisclosed (contact sales) | Freemium, edu-focused |
| **Copyleaks** | $13.99/mo (100 credits) | $74.99/mo (1K credits) | Custom (contact sales) | Credit-based subscription |
| **Grammarly** (AI detect) | Free (100 prompts) | $12/mo Pro (2K prompts) | Contact sales | Bundled with writing tools |

**Key observations:**

- The credit/word-count model is standard for text-based detection tools
- Enterprise pricing is opaque — custom quoting is the norm
- All competitors position as "AI detectors," not trust infrastructure
- None offer a trust graph, multi-modal provenance, or C2PA integration
- **Market gap:** No tool combines text, image, video, audio detection with cryptographic provenance verification

### 1.2 Adjacent Infrastructure Companies (Open-Core Models)

These are more relevant comparables for Sentinel's long-term business model:

| Company | Free Tier | Pro/Starter | Team/Business | Enterprise | Model Type |
|---|---|---|---|---|---|
| **Sentry** | $0 (5K errors) | — | $26/mo | Custom | Open-core, usage-based |
| **PostHog** | $0 (generous) | Pay-as-you-go | — | Custom | Open-core, usage-based |
| **Supabase** | $0 | $25/mo | $599/mo | Custom | Open-core, usage-based |
| **GitLab** | $0 | — | $29/user/mo | Custom | Open-core, per-seat |

**Pattern across all successful open-core companies:**

1. **Generous free tier** — drives adoption, developer love, community
2. **Usage-based Pro tier** — converts power users without friction
3. **Team/Business tier** — SSO, RBAC, compliance, collaboration features
4. **Enterprise tier** — custom pricing, SLAs, dedicated support, on-prem/private cloud

### 1.3 Trust & Safety Infrastructure (Closest Analogy)

| Company | Approach | Pricing |
|---|---|---|
| **OpenAI Moderation API** | Free moderation endpoint (text + images) | $0 — it's a loss leader for their API platform |
| **Cloudflare** | DDoS, bot management, WAF | Freemium → $20–$200/mo → Enterprise |
| **VirusTotal** | File/URL scanning, community intel | Free basic → paid API (Google acquired) |
| **Hive Moderation** | Content moderation API | Usage-based, ~$0.01–$0.05 per image |

**Critical insight:** OpenAI's moderation API being free is both a threat and proof of demand. They subsidize it to protect their AI platform. Sentinel's advantage vs. OpenAI: explainability, provenance, independence (not tied to any AI model provider), and privacy-first architecture.

---

## 2. Enterprise Use Cases & Willingness to Pay

### 2.1 Enterprise Buyer Personas (By Willingness to Pay)

| Buyer | Use Case | Willingness to Pay | Decision Trigger |
|---|---|---|---|
| **Publishers/News Organizations** | Verify UGC, detect synthetic media before publishing | High ($10K–$100K+/yr) | Reputation risk, liability from publishing fake content |
| **Brand Protection Teams** | Monitor for deepfakes, impersonation, brand misuse | High ($10K–$50K+/yr) | Brand damage, fraud costs |
| **Legal/Compliance** | Evidence authentication, regulatory compliance, discovery | Very High ($50K–$200K+/yr) | Regulatory mandates, litigation risk |
| **HR/Recruiting** | Verify candidate submissions, detect AI-generated applications | Medium ($1K–$10K/yr) | Hiring quality, fairness concerns |
| **Education** | Academic integrity | Low-Medium ($500–$5K/yr) | Budget-constrained, price-sensitive |
| **Cybersecurity Teams** | Detect AI-generated phishing, social engineering | High ($10K–$50K+/yr) | Direct security threat |
| **CMS/Publishing Platforms** | White-label trust layer for their users | Very High ($100K+/yr) | Competitive differentiation, trust |
| **Government/Intelligence** | Information warfare defense, foreign influence detection | Very High (classified budgets) | National security mandate |

### 2.2 Enterprise Purchasing Patterns

Enterprises do **not** purchase browser extensions through procurement. The extension is the **wedge** — it proves value to individual users who then advocate for organizational adoption.

The enterprise sales path:
```
Browser Extension (individual adoption)
    → Team Dashboard (team manager sees value)
        → Enterprise API/SDK (IT/procurement purchases)
            → Enterprise Platform (organization-wide deployment)
```

This is the **Grammarly playbook**: free browser extension → individual Pro subscriptions → enterprise deals with SSO, admin controls, analytics. Grammarly now has 50,000+ organizational customers and was last valued at $13B.

### 2.3 Addressable Market Sizing Framework

Rather than precise numbers, think in layers:

1. **TAM (Total Addressable Market):** Every organization that publishes, consumes, or verifies digital content — essentially the entire internet economy
2. **SAM (Serviceable Addressable Market):** Organizations with active trust/safety teams, compliance requirements, or content verification workflows
3. **SOM (Serviceable Obtainable Market):** Early adopters reached through developer channels, browser extension store, and content marketing

Growth vector: Start with publishers and cybersecurity teams (highest acute need), expand to brand protection and legal, then move downmarket to education and consumers.

---

## 3. Recommended Business Model

### 3.1 Model: Open-Core + Usage-Based SaaS + Enterprise Platform

This is the dominant model among successful infrastructure companies and aligns with Sentinel's philosophy.

#### Tier Structure

**Tier 0: Free / Community (Browser Extension + Open Source Core)**
- Browser extension with local analysis (text)
- Basic confidence scores with simple explanations
- C2PA content credential verification (read-only)
- Community support (GitHub Discussions, Discord)
- MIT/Apache 2.0 license for detection orchestration framework
- **Goal:** Distribution, community, trust-building, developer adoption

**Tier 1: Pro / Individual ($9–15/mo or $99–149/yr)**
- Cloud-assisted analysis (all modalities: text, image, video, audio)
- Advanced explainability (per-signal breakdowns)
- History & saved scans
- Custom trust thresholds & per-site rules
- Export reports (PDF/JSON)
- **Goal:** Convert power users, generate initial revenue, validate willingness to pay

**Tier 2: Team / Business ($25–50/user/mo)**
- Team dashboard with shared scan history
- RBAC, audit logs, admin controls
- API access (rate-limited)
- Priority processing queue
- Integration with Slack/Teams (alerts on detected content)
- SAML SSO, basic compliance features
- **Goal:** Organizational adoption, recurring revenue, land-and-expand

**Tier 3: Enterprise (Custom, $50K–$500K+/yr)**
- Full API/SDK access (high rate limits)
- Private cloud or on-prem deployment
- Custom detection models / fine-tuning
- Trust graph API (query reputation of domains, sources, content hashes)
- White-label / embedded trust layer for platforms
- Dedicated customer success, SLAs (99.9%+), security review support
- FedRAMP/IL5 readiness for government
- **Goal:** High-value contracts, platform revenue, defensibility

**Tier 4: Trust Graph / Data API (Usage-Based, Separate Product)**
- Query the Sentinel trust graph: domain reputation, content hash lookup, source reliability scores
- Per-query pricing with volume discounts
- For partners, platforms, and data integrators
- **Goal:** Network-effect revenue, data moat

### 3.2 Justification

Why this model over alternatives:

| Alternative | Why Not |
|---|---|
| **Pure freemium browser extension (like Honey)** | Honey monetized via affiliate/referral fees — incompatible with trust positioning. Can't take commissions from content sources you're analyzing. |
| **Pure subscription SaaS only** | Loses the developer/community adoption that open-core drives. Higher customer acquisition costs. |
| **Marketplace model** | No clear two-sided marketplace dynamic in trust verification. |
| **Advertising/data selling** | Explicitly violates the privacy-first philosophy and destroys trust. |
| **Consulting/services-heavy** | Doesn't scale. Can be an add-on but not primary model. |

### 3.3 Revenue Projections Framework

Instead of precise numbers, model structure:

```
Revenue = (Free Users × Conversion Rate × ARPU) + (Enterprise Deals × ACV)

Where:
- Free Users: Driven by extension store, content marketing, developer advocacy
- Conversion Rate: 2-5% is strong for browser extensions (Grammarly: ~5-7%)
- ARPU (Pro): $100-150/yr (annual plan)
- ARPU (Team): $300-600/user/yr
- Enterprise ACV: $50K-500K/yr depending on scale

Example framework (end of Year 2):
  100K free extension users × 3% conversion × $120/yr = $360K
  20 Team customers × 10 seats × $400/yr = $80K
  5 Enterprise deals × $100K/yr = $500K
  API/Trust Graph early revenue = $60K
  Total ≈ $1M ARR

  This is a framework, not a forecast. Actual numbers depend on execution.
```

---

## 4. Open-Core Boundary Recommendation

### 4.1 What Should Be Open Source (MIT or Apache 2.0)

| Component | License | Rationale |
|---|---|---|
| **Detection orchestration framework** | MIT | Core composability layer — developers integrate their own detectors. Drives adoption. |
| **Browser extension (core logic)** | MIT | Transparency builds trust. Users can verify no telemetry/spying. |
| **C2PA verification library** | MIT | Standards implementation should be open. Contributions from C2PA community. |
| **Signal definition format** | MIT | Like OpenTelemetry for trust signals — industry standard potential. |
| **SDK for custom detectors** | MIT | Enables ecosystem. Community contributes detectors. |
| **CLI tools** | MIT | Developer tooling that drives API adoption. |

### 4.2 What Should Be Proprietary

| Component | Rationale |
|---|---|
| **Proprietary detection models** | Core IP. Training data, model weights, fine-tuning. Different from open-source orchestrator — the orchestrator can run any model, but Sentinel's best models are proprietary. |
| **Trust graph database** | Network-effect data asset. Domain reputations, source reliability, content hash registry. |
| **Cloud infrastructure & API** | Hosted service is the primary monetization. Sentry model: open-source core, proprietary cloud. |
| **Enterprise dashboard & analytics** | Admin/team features that enterprises pay for. |
| **Multi-modal fusion engine** | How signals are combined into confidence scores — the "special sauce." |
| **Automated threat intelligence** | Active monitoring, crawling, alerting — operational infrastructure. |

### 4.3 Preventing Cloud Provider Competition

This is the standard open-core risk: AWS/GCP/Azure host your open-source code and compete with you.

**Sentinel's protections:**

1. **Proprietary detection models** — The orchestration framework is open-source but the best models are not. Self-hosters need to provide their own models or use inferior open-source ones.
2. **Trust graph network effects** — The more users contribute to and query the trust graph, the more valuable it becomes. A hosted clone has an empty graph.
3. **AGPL for edge components** — Consider AGPL for server-side components to require modifications to be open-sourced (prevents proprietary forks by cloud providers). MongoDB/Redis precedent.
4. **Trademark enforcement** — "Sentinel Certified" marks for API compatibility.
5. **Cloud-provider terms** — Explicitly restrict managed hosting of Sentinel by cloud providers using license terms (like Elastic's strategy that forced AWS to fork).
6. **Frequent releases** — The cloud version gets models, updates, and features first. Self-hosted lags.

### 4.4 Contributor License Agreement (CLA)

Require CLA for all contributions. This allows Byrd Ventures LLC to:
- Re-license contributions under proprietary terms
- Defend against patent claims
- Maintain unified IP ownership

Options:
- **Individual CLA** — Standard for most open-core companies
- **Corporate CLA** — For companies assigning employees to contribute
- **DCO (Developer Certificate of Origin)** — Lighter weight, but weaker IP protection

**Recommendation:** Start with DCO during community building (low friction). Add CLA before any major fundraising or enterprise deals.

---

## 5. Go-to-Market Strategy

### 5.1 Initial Target Audience: Developers & Privacy-Conscious Power Users

The browser extension should launch targeting:

1. **Developers (primary)** — They influence organizational tooling decisions. They care about explainability and extensibility. They contribute to open-source.
2. **Privacy-conscious power users** — Early adopters who already use ad blockers, password managers, VPNs. The "trust layer" concept resonates.
3. **Technical journalists & researchers** — Need verification tools. Will write about them if they're good.

**Rationale:** This follows the PostHog/Sentry/GitLab playbook. Developers adopt first, then bring tools into their organizations. Enterprise sales is easier when engineers already use and advocate for the product.

### 5.2 Distribution Channels

**Primary:**
- **Chrome Web Store / Firefox Add-ons / Edge Add-ons** — Direct distribution. SEO within extension stores.
- **GitHub** — Open-source repos drive discovery. Stars, contributions, discussions.
- **Developer content** — Technical blog posts, tutorials, comparisons with other tools.
- **Hacker News / Reddit** — Direct engagement with technical audience.

**Secondary:**
- **API marketplaces** — RapidAPI, Postman collections for developer onboarding.
- **Cloud marketplaces** — AWS Marketplace, GCP Marketplace for enterprise deals.
- **LMS integrations** — Canvas, Moodle (education channel, lower priority).

### 5.3 Partnership Opportunities

| Partner Type | Value | Priority |
|---|---|---|
| **C2PA (Content Credentials)** | Standards alignment, credibility | Immediate |
| **Browser vendors** | Built-in integration, distribution | Long-term (risky — they may compete) |
| **CMS platforms (WordPress, Drupal)** | Plugin ecosystem, massive distribution | Medium |
| **News organizations (Reuters, AP, NYT)** | Credibility, case studies, design partners | Medium |
| **Social platforms** | Integration into content moderation pipelines | Long-term |
| **Cybersecurity vendors (CrowdStrike, Cloudflare)** | Complementary product, channel partnership | Medium |
| **Academic institutions** | Research collaboration, model validation | Medium |

### 5.4 Content Marketing Approach

Content strategy for a trust platform:

1. **Transparency reports** — Regular publications on detection accuracy, false positive rates, model limitations. Builds trust through honesty.
2. **Educational content** — "How to spot deepfakes," "Understanding C2PA," "Digital provenance explained."
3. **Research collaborations** — Partner with universities on AI detection benchmark studies.
4. **Case studies** — Real examples of synthetic content caught, with evidence trails.
5. **Comparison content** — Sentinel vs. black-box AI detectors, emphasizing explainability difference.
6. **Open data** — Publish benchmark datasets. Community contributions improve them.

**Key principle:** Content should teach, not sell. The "trust layer" positioning means Sentinel's own marketing must model transparency and evidence.

---

## 6. Competitive Moat Analysis

### 6.1 What Prevents Big Tech From Building This?

They are. C2PA steering committee includes Adobe, Google, Microsoft, OpenAI, Meta, Amazon, BBC, Sony, Truepic.

But they're building **different things:**

| Company | What They're Building | Why Sentinel is Different |
|---|---|---|
| **Google** | SynthID watermarking, C2PA in Android/Chrome | Platform-level, not content-agnostic. Tied to their AI models. |
| **Adobe** | Content Credentials in Creative Cloud | Tooling for creators, not verification for consumers. |
| **Microsoft** | C2PA in Bing/Edge | Search integration, not a universal trust layer. |
| **Meta** | AI-generated content labeling | Platform-specific. Users don't trust Meta to tell them what's real. |
| **OpenAI** | C2PA in DALL-E, moderation API | Tied to their models. Not independent verification. |
| **Truepic** | Content provenance SDK | Infrastructure play (closest competitor). But focused on media capture, not universal analysis. |

**Big Tech's structural disadvantage:** They cannot be a neutral trust layer. Each has platform interests. Google benefits from AI content in search results. Meta benefits from engagement regardless of authenticity. OpenAI can't credibly audit its own models.

**Sentinel's independence is a moat.** A trust layer must be independent of content creators, platforms, and AI model providers.

### 6.2 Moats Sentinel Can Build

| Moat | Type | Strength | Time to Build |
|---|---|---|---|
| **Trust Graph** (domain reputation, source reliability, content hash registry) | Data network effects | Very Strong | 2-3 years |
| **Detection Model Quality** (proprietary training data, multi-modal fusion) | Technical IP | Strong | 1-2 years |
| **Community Reputation** (transparent methodology, open benchmarks) | Brand/Trust | Strong | 1-3 years |
| **Standards Leadership** (signal format, orchestration standard) | Ecosystem lock-in | Very Strong | 3-5 years |
| **Integration Breadth** (browsers, CMS, platforms, APIs) | Switching costs | Medium-Strong | 2-4 years |
| **C2PA/Content Credentials Expertise** | Standards expertise | Medium | 1-2 years |
| **Independent Stance** (not tied to any AI model provider or platform) | Positioning | Strong | Immediate |
| **Privacy-First Architecture** (local processing preference) | Regulatory/Trust | Strong | Immediate |

### 6.3 Platform Risk

**Real risks:**
- Browsers could integrate trust features natively (Chrome already adding C2PA support)
- Browser extension APIs could be restricted (MV3 already limits some capabilities)
- Apple/Google could block or remove the extension from stores

**Mitigations:**
1. **Multi-platform from day one** — Chrome, Firefox, Edge, Safari. No single platform dependency.
2. **API-first architecture** — If extensions are blocked, the API and SDK survive.
3. **Standards, not platform features** — Build toward C2PA and open standards integration so browser vendors adopt Sentinel-compatible standards.
4. **User advocacy** — A trusted user base makes removal politically costly for platforms.
5. **Desktop/mobile apps** (Phase 4) — Reduce extension dependency.

---

## 7. Enterprise Feature Tier Recommendations

### 7.1 Enterprise-Grade Features (Required for Tier 2-3)

| Feature | Tier 2 (Team/Business) | Tier 3 (Enterprise) |
|---|---|---|
| **SAML/OIDC SSO** | Yes | Yes |
| **SCIM provisioning** | No | Yes |
| **RBAC / custom roles** | Basic (Admin/Member) | Custom roles |
| **Audit logs** | 30 days | 1 year+, exportable |
| **API access** | Rate-limited | High volume, custom SLAs |
| **Private cloud / on-prem** | No | Yes |
| **Data residency** | US/EU | Custom region |
| **Custom detection models** | No | Yes (fine-tuning) |
| **Trust graph API** | Read-only | Read-write, contribute |
| **White-label / embedded** | No | Yes |
| **Dedicated support** | Email, 24hr SLA | Phone/Slack, 1hr SLA |
| **Security review support** | Standard questionnaire | Custom, FedRAMP package |
| **Usage analytics dashboard** | Basic | Advanced, exportable |
| **Compliance certifications** | SOC 2 | SOC 2, ISO 27001, HIPAA |
| **Integration with SIEM/SOAR** | No | Yes |

### 7.2 API Pricing Benchmarks

Based on comparable services:

| Usage Tier | Monthly API Calls | Price (suggested) | Comparable |
|---|---|---|---|
| Free | 1,000 calls/mo | $0 | Sentry free tier, PostHog free tier |
| Starter | 10,000 calls/mo | $49/mo | AI detection API market rate |
| Growth | 100,000 calls/mo | $399/mo | Volume discount |
| Scale | 1M calls/mo | $1,999/mo | Enterprise API pricing |
| Custom | 1M+ calls/mo | Custom quote | Negotiation-based |

Trust Graph API can be priced separately, per query, at higher rates since it provides unique network-effect value.

---

## 8. Investment & Growth Strategy

### 8.1 Bootstrapping Viability

Sentinel **can** be bootstrapped through Phase 2 (MVP) with minimal capital:
- Small team (2-3 engineers, 1 business)
- Cloud costs are compute-dependent but manageable for browser extension + API
- Open-source community can supplement development

Phase 3-4 (Platform, Enterprise) likely requires external capital for:
- Dedicated ML engineers for proprietary model development
- Enterprise sales team
- Infrastructure for trust graph at scale
- Compliance certifications (SOC 2, ISO 27001 — $50K-100K+ each)

### 8.2 Venture Capital Interest (2026 Landscape)

As of mid-2026, AI safety, trust, and content authenticity are significant VC themes:

- **AI detection/watermarking** startups have raised significant rounds (GPTZero, Originality.ai)
- **Content provenance** is a regulatory priority (EU AI Act, US AI executive orders mandate labeling)
- **Trust & safety as a service** is a growing category
- **C2PA adoption** is accelerating (Google, Meta, Adobe, OpenAI all implementing)

**Investor narrative:** "Sentinel is building the trust infrastructure for the AI-generated internet — independent, explainable, and privacy-first. As regulators mandate AI content labeling and enterprises face liability from synthetic media, Sentinel becomes the standard layer for verification, analogous to what SSL/TLS became for secure connections."

### 8.3 Revenue Milestones for Validation

| Milestone | Metric | What It Validates |
|---|---|---|
| **Product-Market Fit (Consumer)** | 10K active extension users, >4 star rating | Individual users find value |
| **Initial Monetization** | 500 paying Pro users | Willingness to pay exists |
| **Team Adoption** | 10 organizations on Team plan | Organizational budget exists |
| **Enterprise Validation** | 2-3 design partners at $25K+ | Enterprise use case confirmed |
| **Platform Revenue** | API revenue exceeds consumer revenue | Infrastructure play working |
| **Scale Signal** | $1M ARR | Viable business, fundable |
| **Growth Signal** | $5M ARR with >100% YoY growth | Venture-scale opportunity |

---

## 9. Key Monetization Risks

| Risk | Severity | Mitigation |
|---|---|---|
| **AI detection becomes unsolvable** (models indistinguishable from human) | Existential | Pivot to provenance/cryptographic verification (C2PA). Detection is one signal, not the product. |
| **Google builds this into Chrome** | High | Multi-platform, API-first, independence positioning. Google can't be neutral. |
| **Open-source models erode proprietary model value** | Medium | Trust graph and explainability layer are the moat, not just detection accuracy. |
| **False positive liability** (wrongly flagging human content as AI) | High | Explainability over binary classification. "Confidence scores, not decisions." Legal disclaimers. |
| **Enterprise sales cycles too long for startup** | Medium | Start with self-serve Pro/Team revenue. Enterprise sales later. |
| **Privacy regulations restrict analysis capabilities** | Medium | Local-first architecture aligns with privacy trends. GDPR-friendly by design. |
| **Cloud providers host open-source core** | Medium | Proprietary models, trust graph, AGPL licensing, trademark enforcement. |
| **Free tier cannibalizes paid conversion** | Low-Medium | Usage-based limits. Power users naturally hit limits. |

---

## 10. References & Comparable Companies

### Open-Core Precedents

| Company | Open Source | Monetization | Valuation/Revenue |
|---|---|---|---|
| **Sentry** | MIT license | Cloud hosting, enterprise features | $3B+ valuation (Series E) |
| **PostHog** | MIT license | Usage-based cloud, enterprise | $1B+ valuation, self-sustaining |
| **Supabase** | MIT license | Managed hosting, enterprise | $2B+ valuation |
| **GitLab** | MIT/EE license | Subscription tiers, enterprise | Public ($10B+ market cap at peak) |
| **Elastic** | Apache 2.0 → Elastic License | Cloud hosting (conflict with AWS) | Public, $5B+ market cap |

### Trust/Safety Market Precedents

| Company | Focus | Model | Scale |
|---|---|---|---|
| **Hive Moderation** | Content moderation API | Usage-based | $2B valuation (2021) |
| **ActiveFence** | Trust & safety intelligence | Enterprise SaaS | $500M+ valuation |
| **Safeguard (by DataDome)** | Bot detection | Enterprise SaaS | $1B+ valuation |
| **Truepic** | Content provenance | Enterprise SDK | $100M+ raised |

### Browser Extension Monetization Precedents

| Product | Free → Paid Path | Model |
|---|---|---|
| **Grammarly** | Free extension → Pro $12/mo → Enterprise | Freemium SaaS, $13B valuation |
| **1Password** | Free extension (view-only) → $2.99/mo → Business $7.99/user | Freemium, enterprise |
| **Honey** | Free → affiliate commissions (acquired by PayPal $4B) | Not applicable to trust tools |
| **Loom** | Free extension → Business $12.50/user | Freemium SaaS (acquired by Atlassian $975M) |

---

## 11. Final Recommendations

### Recommended Business Model
**Open-Core + Usage-Based SaaS + Enterprise Platform**, with the browser extension as the free distribution wedge.

### Open-Core Boundary
- **Open:** Detection orchestration framework, browser extension core, C2PA verification library, signal format, SDKs, CLI tools
- **Proprietary:** Detection models, trust graph, multi-modal fusion engine, cloud infrastructure, enterprise dashboard, threat intelligence

### Pricing Structure
- **Free:** Browser extension (local processing, basic text analysis)
- **Pro:** $9-15/mo (cloud-assisted, all modalities, advanced explainability)
- **Team:** $25-50/user/mo (dashboard, API, SSO, collaboration)
- **Enterprise:** Custom $50K-500K+/yr (full platform, private cloud, trust graph API)
- **Trust Graph API:** Usage-based, separate product

### Primary Revenue Driver (Years 1-3)
Enterprise deals with publishers, cybersecurity teams, and brand protection — these buyers have acute needs and high willingness to pay. Consumer Pro subscriptions provide validation and base revenue, but enterprise is where material revenue lives.

### Critical Success Factors
1. **Independence** — Never tie to any single AI model provider or platform
2. **Explainability** — The transparency differentiator must be real, not marketing
3. **C2PA integration** — Standards alignment is the path to ecosystem lock-in
4. **Developer love** — Open-source, great docs, extensibility. Developers are the enterprise buying committee.
5. **Privacy by design** — GDPR/regulatory compliance isn't a feature, it's table stakes

### Single Most Important Strategic Principle
**Sentinel must never appear to decide what is true. It provides evidence. Users decide.** This is what separates Sentinel from every "AI detector" in the market and what makes the "trust layer" positioning commercially defensible. If Sentinel is perceived as a censorship tool or a truth arbiter, both the consumer and enterprise propositions collapse.
