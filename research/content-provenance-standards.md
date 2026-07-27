# Content Provenance Standards & Ecosystem Research

**Project:** Sentinel | **Phase:** 0 - Discovery
**Date:** July 2026
**Audience:** Architecture Team, Product Team

---

## 1. C2PA (Coalition for Content Provenance and Authenticity)

### 1.1 Specification Status

The C2PA specification is at **version 2.4** (released mid-2025), with 2.3 as the current stable downloadable version. The spec is royalty-free, open standard, and hosted as a Joint Development Foundation (JDF) project under the Linux Foundation.

**Key spec additions in 2.4:**
- Content Credentials JSON (crJSON) format - a JSON serialization for use cases where binary embedding isn't possible (social media, web, streaming)
- Soft Binding API - enables binding C2PA manifests to content that can't have manifests embedded directly (decoupled manifests)
- Human and Organizational Identity Recommendation

**Spec structure:**
- **Manifests**: JSON-LD data structures containing assertions about content
- **Assertions**: Claims about the content (creator, actions, ingredients, etc.)
- **Signatures**: Cryptographic signatures (W3C Verifiable Credentials compatible) that bind manifests to content
- **Ingredient relationship**: A chain of manifests showing the content's edit history

### 1.2 Technical Architecture

C2PA works through **cryptographically signed manifests** embedded in or associated with media files:

1. **Content is created** → A "creation" claim is generated with metadata (timestamp, device, creator identity)
2. **Content is edited** → Each editing tool appends a new manifest referencing the previous manifest (ingredient chain)
3. **Content is published** → A publisher assertion is added, signed by a certificate from a trusted Certificate Authority

**The trust model:**
- C2PA does NOT verify "truth" of content
- It verifies **who** created/published it, **when**, with **what tool**, and whether it's been **tampered with** since signing
- Trust is transitive: you trust the content because you trust the signer
- This matches Sentinel's philosophy perfectly ("evidence, not truth")

**Technologies used:**
- CBOR (Concise Binary Object Representation) for compact binary encoding
- JUMBF (JPEG Universal Metadata Box Format) for embedding in media
- X.509 certificates for signer identity
- Cryptographic hashing (SHA-256) for tamper detection
- W3C Verifiable Credentials data model

### 1.3 Who Is Implementing It

**Steering Committee Members (10 companies):**
| Company | Role |
|---------|------|
| Adobe | Co-founder; ships CC in Photoshop, Lightroom, Firefly |
| Microsoft | Co-founder; integrates into Azure, Edge research |
| Google | DeepMind SynthID + C2PA cross-compatibility work |
| OpenAI | DALL-E generated images get C2PA manifests |
| Meta | AI-generated content labeling with C2PA manifests |
| Amazon | AWS integration, cloud signing services |
| BBC | News content signing (via Project Origin) |
| Sony | Camera hardware signing (Alpha series) |
| Intel | Hardware attestation, chip-level signing |
| Truepic | Secure camera capture SDK; C2PA pioneers |
| Publicis Groupe | Advertising supply chain provenance |

**Notable General/Contributor Members (60+ organizations):**
- **Camera manufacturers:** Leica, Nikon, Canon, Fujifilm, Samsung
- **AI companies:** OpenAI, ElevenLabs, HeyGen, Resemble AI
- **Media:** NYT, AP, AFP, Reuters, CBC, NHK, WDR, France Televisions
- **Infrastructure:** Akamai, Fastly, Cloudflare, DigiCert, Sectigo
- **Blockchain:** Numbers Protocol, Polygon, Story Protocol, OriginTrail, Aptos
- **Enterprise:** Bank of America, New York Times, Shutterstock, Canva

### 1.4 Real-World Adoption (Mid-2026)

**Strong adoption:**
- Adobe Creative Cloud (Photoshop, Lightroom): Content Credentials are built in. All Firefly AI outputs are signed.
- OpenAI: DALL-E 3 and GPT image outputs include C2PA manifests. GPT text outputs use C2PA crJSON.
- Microsoft: Built into Designer, Copilot image generation. Azure has C2PA signing services.
- Leica M11-P: First camera with hardware-based C2PA signing (2023). Sony Alpha series followed.
- Google: SynthID watermarks on all Gemini-generated images; working on C2PA interoperability.

**Growing adoption:**
- Meta: Started labeling AI-generated images on Facebook/Instagram with C2PA metadata (2024)
- BBC: Project Origin implementation for all published news photos
- Nikon, Canon, Fujifilm: Announced C2PA support in camera firmware

**Still early:**
- Most consumer apps don't display C2PA data
- Social platforms inconsistently preserve C2PA metadata on upload
- Video signing is technically supported but rarely deployed

### 1.5 Browser Support Status

- **No native browser support as of July 2026.** No browser renders or validates C2PA manifests natively.
- The `c2pa-js` library (WebAssembly-based, MIT licensed) enables browser-side verification using the Rust `c2pa-rs` library compiled to WASM.
- Chrome/Edge have shown interest but no shipped feature. Google is a steering committee member.
- The verification flow works entirely client-side in the browser via WASM - no server needed.
- **Key observation:** A browser extension is the primary way to bring provenance verification to browsers today. This is exactly Sentinel's first planned product.

### 1.6 SDK/API Availability

| SDK | Language | Status | Notes |
|-----|----------|--------|-------|
| c2pa-rs | Rust | Mature | Core library; all other bindings derived from this |
| c2pa-js | JavaScript/TS | Mature | Browser (WASM) + Node.js; read+verify+sign |
| c2pa-python | Python | Mature | Pip-installable |
| c2pa-c | C++ | Mature | Desktop integration |
| c2pa-iOS | Swift | Stable | Native iOS SDK |
| c2pa-android | Kotlin/Java | Stable | Native Android SDK |
| c2patool | CLI | Mature | Reference CLI for signing/verifying |
| Content Credentials Verify | Web App | Live | Online verification tool at verify.contentauthenticity.org |

**All open source, MIT licensed, maintained under the CAI GitHub organization (contentauth).**

### 1.7 Limitations and Criticisms

1. **Privacy concern:** Manifests can contain extensive metadata (location, device info, creator identity). A signed manifest that includes personally identifiable information cannot be easily redacted.
2. **Signer trust assumption:** C2PA doesn't verify if the signer's claims about the content are true. A bad actor with a valid certificate could sign fake content.
3. **Metadata stripping:** If C2PA metadata is stripped (by social platforms, screenshots, etc.), the provenance is lost. There's no cryptographic way to recover it without the manifest.
4. **Single-vendor governance:** Despite being multi-stakeholder, Adobe drives most of the open-source tooling and development.
5. **Screenshot/screen recording bypass:** A screenshot of a C2PA-signed image creates a new image without the manifest. This is the "air gap" problem.
6. **Certificate authority centralization:** Trust roots back to traditional CAs, which some consider a centralization risk.
7. **File format dependency:** C2PA works with JPEG, PNG, TIFF, AVIF, WebP, MP4, M4A, WAV, and PDF. Not all formats are supported yet (GIF, BMP, some raw formats).

---

## 2. Content Credentials (Adobe-led branding)

### 2.1 Relationship to C2PA

- **C2PA** is the technical specification and standards body
- **Content Authenticity Initiative (CAI)** is the community/membership organization (5,000+ participants)
- **Content Credentials** is the consumer-facing brand name for C2PA manifests
- The "CR" (Content Credentials) pin icon is the visual indicator

### 2.2 Implementation in Creative Cloud

- **Photoshop:** Built-in Content Credentials panel. Auto-sign on export. "CR" pin toggle.
- **Lightroom:** Content Credentials on export for photos
- **Adobe Firefly:** All AI-generated outputs automatically signed with C2PA manifests
- **Adobe Express:** Content Credentials support for templates and designs

### 2.3 Consumer Awareness

- The "CR" pin icon rollout is underway but **consumer awareness remains low**
- Adobe reports that most users don't know what the pin means
- Verification is available through verify.contentauthenticity.org but **few consumers know to check**
- The gap between "content has credentials" and "users know to look for them" is significant

### 2.4 Camera Manufacturer Adoption

| Manufacturer | Product | Year | Notes |
|-------------|---------|------|-------|
| Leica | M11-P | 2023 | First with hardware-based C2PA signing |
| Sony | Alpha 1 II, Alpha 9 III | 2024 | Firmware update adds C2PA signing |
| Nikon | Z9 | 2024 | Firmware update adds C2PA signing |
| Canon | EOS R1 | 2025 | Announced support |
| Fujifilm | GFX series | Announced | Committed to C2PA |

Truepic provides SDKs for secure camera capture on mobile devices that integrate C2PA signing from capture moment.

### 2.5 Verification Tools

- **verify.contentauthenticity.org** (web app): Upload images to inspect C2PA manifests
- **C2PA Tool (CLI):** `c2patool` for reading, signing, verifying
- **Browser library:** `c2pa-web` npm package for in-browser verification
- **Adobe Content Credentials panel:** Built into Photoshop/Lightroom
- **IPTC Origin Verify:** Verification tool focused on news publishers

---

## 3. SynthID (Google DeepMind)

### 3.1 How It Works Technically

SynthID embeds **imperceptible digital watermarks** directly into content at generation time:

- **Images/Video:** Modifies pixel values at generation in ways imperceptible to humans but statistically detectable. Robust to screenshots, compression, cropping, and filters. (Specific algorithm not fully disclosed.)
- **Audio:** Embeds inaudible watermark into audio waveforms. Survives MP3 compression, noise addition, and speed changes.
- **Text:** Operates at the **token probability level**. During generation, the LLM subtly adjusts which tokens are selected based on a watermark key. Detection works by analyzing the statistical pattern of token choices. This is the "tournament sampling" approach - for each token, candidate words are assigned to "green" or "red" lists based on a hash of preceding tokens, and green-list tokens are slightly favored.

### 3.2 Modalities Covered

| Modality | Status | Used In |
|----------|--------|---------|
| Images | Production | Imagen, Gemini image gen, Nano Banana |
| Audio | Production | Lyria music model, NotebookLM podcasts |
| Video | Production | Veo video generation |
| Text | Production | Gemini app and web (all text outputs) |

### 3.3 Availability as a Verification Tool

- **SynthID Detector** (portal): Verification tool for journalists/media professionals. Waitlist-based access.
- **Gemini chatbot:** Users can upload content and ask "was this created by Google AI?" and Gemini checks for SynthID watermarks.
- **Google AI Studio:** SynthID watermarking available for developers using Gemini API.
- **Not publicly available as an independent library or API** for third-party use.

### 3.4 Integration with Google Products

- All Gemini-generated images, audio, video, and text are watermarked
- Nano Banana (Gemini 2.5/3.1 Flash Image) watermarks all generated images
- Veo watermarks all generated video
- Lyria watermarks all generated music
- NotebookLM watermarks audio in generated podcasts

### 3.5 Detectability Without Google Cooperation

- **Not independently detectable without Google's cooperation.** The detection algorithm is proprietary.
- Google has not released the detection code publicly.
- There is no open-source implementation of SynthID detection.
- Third-party verification relies on Google's detector portal (waitlisted) or Gemini chatbot.
- This is a **key limitation**: SynthID creates a walled garden for verification, unlike C2PA which is fully open.

---

## 4. Other Watermarking/Provenance Technologies

### 4.1 Steg.ai

- **Type:** Deep learning-based forensic watermarking (steganography)
- **Modalities:** Images, video, documents, audio
- **Key capabilities:**
  - Invisible, robust watermarks that survive compression, screenshots, and transformations
  - Leak tracing (unique watermark per viewer to trace source of leaks)
  - Deepfake detection (watermarking content to detect unauthorized AI manipulation)
  - C2PA integration for provenance
- **Business model:** SaaS + API + on-premise deployment
- **Memberships:** C2PA member, Adobe CAI member
- **Notable:** NSF-funded, Nvidia Inception Program

### 4.2 IMATAG

- **Type:** Invisible digital watermarking for brand protection and content tracking
- **Modalities:** Images primarily
- **Key capabilities:**
  - Invisible watermark embedding and detection
  - C2PA integration
  - Image monitoring across web
- **Status:** C2PA member, focused on brand protection and copyright

### 4.3 Digimarc

- **Type:** Digital watermarking for both physical products and digital content
- **Modalities:** Images, video, audio, printed materials, packaging
- **Key capabilities:**
  - C2PA provenance and authenticity solution
  - Piracy prevention
  - Anti-counterfeiting
  - Product digitization
  - Media watermarking (Emmy Award-winning technology)
- **Business model:** Enterprise, with major clients (P&G, AstraZeneca, Schnucks, Loreal)

### 4.4 IPTC Photo Metadata Standard

- **Type:** Embedded metadata standard for images (administrative, descriptive, rights)
- **History:** Industry standard since 1990s. The most widely used photo metadata standard.
- **Integration with C2PA:** IPTC is a liaison partner to C2PA. C2PA manifests can carry IPTC metadata fields.
- **Key properties:** Creator, Copyright Notice, Caption/Description, Credit Line (the "4Cs")
- **Key insight:** IPTC metadata is widely embedded in photos but **routinely stripped** by social media platforms.

### 4.5 EXIF-based Approaches

- **Type:** Technical metadata embedded in image files (camera make, model, GPS, timestamp, settings)
- **Status:** Universal in digital photography. Reliable for capturing device info but **easily stripped or edited.**
- **Not a provenance solution:** EXIF doesn't provide cryptographic assurance. Can be modified with any metadata editor.
- **Use in Sentinel:** Valuable as one of many signals (e.g., EXIF mismatch with claimed origin is suspicious).

### 4.6 Blockchain-based Provenance

#### Numbers Protocol
- **Type:** Blockchain-based content provenance and tracking
- **Mechanism:** Content is hashed and registered on-chain. Every edit/transformation is tracked.
- **C2PA member:** Yes, integrating C2PA with blockchain anchoring
- **Key insight:** Blockchain provides immutability for provenance records but doesn't solve the "first mile" problem (was the original content authentic when registered?)

#### Other Blockchain Efforts
- **OriginTrail:** Decentralized Knowledge Graph for supply chain and content provenance; C2PA member
- **Story Protocol:** On-chain IP and provenance registry; C2PA member
- **Polygon:** Layer-2 blockchain; C2PA member exploring content provenance
- **Numbers Protocol:** Full provenance tracking with C2PA compatibility

### 4.7 Microsoft's Provenance Efforts

- **Co-founder of C2PA** (Feb 2021)
- **Microsoft Designer/Copilot:** AI-generated images include C2PA manifests
- **Azure:** Cloud-based C2PA signing services
- **Project Origin:** Partnered with BBC, NYT, CBC for news provenance (now merged into IPTC/C2PA ecosystem)
- **Edge browser research:** Investigating native C2PA display in browser chrome
- **No Microsoft-specific proprietary alternative to C2PA** - they've committed fully to the C2PA standard

---

## 5. Cryptographic Approaches

### 5.1 Digital Signatures for Media

C2PA uses **X.509 certificate chains** for signing:
- Content creators, editors, and publishers each sign with their private key
- Verification follows standard PKI trust chains to trusted root CAs
- This is the same infrastructure that powers HTTPS, code signing, and document signing

### 5.2 Hardware Attestation

- **Truepic:** Secure camera SDK that verifies the capture environment is genuine (not a screenshot or screen recording). Uses device attestation.
- **Leica M11-P:** Hardware-secure element signs images at the moment of capture. Tied to the specific camera.
- **Sony/Canon/Nikon:** Using firmware-based signing rather than dedicated hardware security modules.
- **Intel:** Exploring chip-level attestation for content provenance at the silicon level.

### 5.3 Verifiable Credentials for Content

- C2PA specification is compatible with **W3C Verifiable Credentials** data model
- C2PA identity assertions use the VC model for expressing who signed the content
- **IPTC Verified News Publishers list:** A certificate list of verified news organizations. Content signed by a publisher on this list signals that the content came from a verified news source (not that it's "true").
- **Decentralized Identity (DID):** C2PA supports DID-based identities alongside traditional X.509 certificates.

---

## 6. Adoption & Ecosystem Status

### 6.1 Social Media Metadata Handling

Based on IPTC's 2019 survey (last comprehensive test), social platforms were still destructive:

| Platform | Displays Metadata | Preserves on Download | C2PA Status (2026) |
|----------|-------------------|----------------------|---------------------|
| Instagram | NO - strips all | N/A | Testing C2PA labels for AI content (Meta) |
| Facebook | NO - strips most | Strips XMP, retains some EXIF | Testing C2PA labels for AI content |
| Twitter/X | NO - strips all | Strips all | Unknown |
| Flickr | Partial | Preserves originals | Unknown |
| Google Photos | Some IPTC fields | Preserves originals | Unknown |
| Dropbox | None displayed | Preserves originals | N/A |
| Google Drive | None displayed | Preserves originals | N/A |

**Key finding:** Social media platforms historically strip metadata. This is the #1 obstacle to provenance adoption. Meta (Facebook/Instagram) has started labeling AI-generated content using C2PA manifests, but it's unclear if the manifests survive re-upload/download.

**2026 update:** IPTC's more recent research and updates suggest platforms are improving due to regulatory pressure, but comprehensive 2025/2026 data isn't widely published.

### 6.2 Screenshot/Screen Recording Impact

- **Screenshots destroy C2PA manifests.** A screenshot creates a new image file with no provenance data.
- SynthID claims resilience to screenshots for images (the watermark persists at the pixel level), but this is proprietary and unverified independently.
- This is **the fundamental "air gap" problem** - any content displayed on screen can be captured via screenshot/screen recording, severing the provenance chain.

### 6.3 The "Air Gap" Problem (Analog Hole)

Content that passes through an analog stage (screenshot, photograph of screen, recording of speakers) loses all digital provenance:
- C2PA manifests don't survive
- SynthID watermarks may partially survive but detection degrades
- Blockchain provenance points to a hash that no longer matches
- **No existing standard solves this effectively**

### 6.4 Publisher Adoption

- **Reuters, AP, AFP, NYT, BBC, CBC, NHK:** All C2PA members. Various stages of implementation.
- **IPTC Verified News Publishers list:** Growing but not universal. Publishers must apply and verify identity.
- **Most publishers have NOT implemented C2PA signing** as of mid-2026. It remains early adopter territory.

### 6.5 Regulatory Mandates

#### EU AI Act
- **Adopted May 2024, in effect in phases through 2026-2027**
- **Article 50 (Transparency):** AI-generated content must be labeled as artificially generated or manipulated
- **Does NOT mandate C2PA specifically**, but C2PA is the leading candidate for compliance
- **Requires** that AI systems generating synthetic audio, image, video, or text be designed to output in a machine-readable format that is detectable as artificially generated/manipulated
- Watermarking requirements for AI-generated content

#### US Executive Orders
- **Executive Order 14110** (Oct 2023, Biden): Safe, Secure, and Trustworthy AI
  - Requires NIST to develop standards for authenticating content and tracking provenance
  - Requires watermarking of AI-generated content
  - Commerce Department to develop guidance for content authentication
- Subsequent AI executive actions have reinforced provenance requirements

#### Other Jurisdictions
- **China:** Required labeling of AI-generated content since 2023 (deep synthesis provisions)
- **UK:** Online Safety Act + AI Safety Summit commitments to labeling AI content
- **South Korea:** Legislation under consideration for AI content labeling
- **California:** State-level AI transparency bills (SB 942, others)

**Key insight:** Regulatory mandates are a powerful driver for provenance adoption. All major regulations require "labeling" of AI content, which creates demand for both C2PA and SynthID.

---

## 7. Standards Gaps

### 7.1 Content Types Lacking Provenance Standards

| Content Type | C2PA Support | SynthID Support | Gap |
|-------------|-------------|-----------------|-----|
| Images (JPEG, PNG, AVIF, WebP) | Yes | Yes | Good coverage |
| Video (MP4) | Yes | Yes (Veo) | Video signing complex (fragmented), low adoption |
| Audio (WAV, M4A) | Yes | Yes (Lyria) | Low adoption outside Google |
| PDF documents | Yes | No | C2PA signing works but rarely used |
| Plain text | crJSON only | Yes (Gemini) | No universal standard; crJSON is new |
| Live streaming video | No (Soft Binding API emerging) | No | Major gap |
| Social media posts | crJSON (new) | No | Not widely adopted |
| GIFs | No | No | Gap |
| 3D models | No | No | No standard |
| Code | No | No | No provenance standard |
| SVG | No | No | Gap |

### 7.2 Text Provenance Standard

**There is no widely adopted text provenance standard.** Current approaches:
- **C2PA crJSON** (v2.4): JSON-based manifest for text. Very new, minimal adoption.
- **SynthID for text:** Statistical watermarking in token selection. Only works on text from Google's models.
- **OpenAI:** Uses C2PA manifests for GPT text outputs but adoption is limited.
- **No standard for provenance of user-generated text** (the vast majority of text online).

**This is a major gap Sentinel could address** through multi-signal approaches rather than relying on a single standard.

### 7.3 Competitive vs. Complementary Standards

The standards are largely **complementary but operationally competitive**:

- **C2PA + SynthID:** Both aim to solve provenance. Google supports both (C2PA member + SynthID developer). Potential convergence but currently separate.
- **C2PA + IPTC:** Complementary. IPTC metadata fields can be included within C2PA manifests.
- **C2PA + Blockchain:** Complementary. Blockchain can provide immutable anchoring of C2PA manifest hashes.
- **C2PA + Watermarking:** Complementary. Steg.ai, Digimarc, IMATAG integrate with C2PA for combined provenance + protection.

**Concern:** If SynthID becomes dominant, it creates a Google-controlled verification ecosystem. If C2PA becomes dominant, it relies on traditional PKI/CA infrastructure. Neither is ideal from a decentralization perspective.

### 7.4 Interoperability Challenges

1. **C2PA ↔ SynthID:** No standard way to combine or reconcile the two. Different detection mechanisms.
2. **Manifest versioning:** C2PA 2.0-2.4 manifests may not be fully forward-compatible.
3. **Cross-platform:** C2PA tooling from Adobe works on Adobe products but doesn't guarantee identical behavior on non-Adobe implementations.
4. **Certificate trust stores:** Different CAs trusted in different contexts. No universal C2PA certificate trust list.
5. **File format transformation:** Converting a signed JPEG to a WebP may or may not preserve the manifest depending on the tool used.

---

## 8. Integration Opportunity for Sentinel

### 8.1 Browser Extension Verification

**This is a strong opportunity.** A browser extension can:

1. **Detect C2PA manifests** in images on any webpage using `c2pa-web` (WASM-based, runs entirely client-side)
2. **Verify cryptographic signatures** locally (no server needed)
3. **Display provenance information** inline - who created it, with what tool, when
4. **Combine C2PA verification with other signals** (Sentinel's multi-signal approach)
5. **Work across all Chromium browsers** (Chrome, Edge, Brave, Arc) via Manifest V3; Firefox via WebExtensions

**Technical feasibility:** High. The `c2pa-js` library (MIT license) provides browser-side reading and verification. The extension would need to:
- Scan `<img>`, `<video>`, `<audio>` elements on page load
- Fetch the media file to inspect metadata (may require CORS handling)
- Verify manifests client-side using WASM
- Display an overlay/indicator for content with provenance data

### 8.2 Available APIs

| API | Local/Cloud | Cost | Use For |
|-----|------------|------|---------|
| c2pa-web (npm) | Local (WASM) | Free, MIT | Read + verify C2PA manifests |
| c2pa-node (npm) | Local | Free, MIT | Read + verify + sign on server |
| c2pa-python | Local | Free, MIT | Server-side verification |
| c2patool (CLI) | Local | Free, MIT | Command-line verification |
| verify.contentauthenticity.org | Cloud | Free | Web-based inspection |
| SynthID Detector | Cloud (Google) | Waitlist | Detect SynthID watermarks |

### 8.3 Local vs. Cloud Verification

**C2PA verification can be done entirely locally.** The `c2pa-rs` Rust library compiles to WebAssembly and runs in the browser. No server, no API calls, no data leaving the user's machine. This aligns perfectly with Sentinel's privacy-first principles.

**SynthID detection requires Google's cooperation** (either through Gemini chatbot or the waitlisted detector portal). This is a limitation for Sentinel's independence.

### 8.4 What Sentinel Would Add

Existing provenance tools focus on **verifying one standard** (C2PA OR SynthID). Sentinel's differentiating value:

1. **Multi-standard aggregation:** Show C2PA, SynthID, IPTC, EXIF, and other signals in one unified view
2. **Context beyond provenance:** Combine provenance data with AI detection confidence, reputation signals, and cross-reference checks
3. **Explainability:** Unlike a simple "verified/not verified" badge, Sentinel would explain what each signal means and how confident it is
4. **Cross-platform coverage:** Browser extension works on all sites, not just those that natively display Content Credentials
5. **User choice:** Let users decide thresholds for hiding/blurring/warning about content
6. **The "no provenance" signal:** Flag content that lacks provenance data as "No provenance information available" - this alone is valuable context
7. **Gap-filling for text:** Where C2PA and SynthID are weak (user-generated text, social posts), Sentinel can use other detection signals

---

## 9. Assessment Summary

### What Sentinel Can Leverage (Build on existing infrastructure)

| Component | Leverage? | How |
|-----------|----------|-----|
| C2PA manifest reading | Yes - fully | `c2pa-web` library (free, local, WASM) |
| C2PA manifest verification | Yes - fully | Same library handles cryptographic verification |
| C2PA signing certificates | Yes - awareness | Use existing cert chains for trust assessment |
| IPTC/EXIF metadata reading | Yes - fully | Standard libraries available |
| SynthID detection | No | Proprietary; requires Google cooperation |
| Blockchain provenance | Partial | Can query public chains for registered hashes |
| C2PA UX guidelines | Yes - reference | C2PA has published UX guidance for progressive disclosure |

### What Sentinel Must Build

| Component | Build? | Why |
|-----------|--------|-----|
| Multi-signal scoring engine | Yes | Core Sentinel IP - combining provenance + AI detection signals |
| Unified verification UI | Yes | No existing tool combines all signals into one view |
| Browser extension framework | Yes | Core product; no existing extension does multi-signal provenance |
| Text provenance analysis | Yes | Gap in standards; needs multi-signal approach |
| No-provenance signal | Yes | Flagging absence of provenance is novel value |
| Trust graph / reputation system | Yes | Long-term: associating provenance with publisher reputation |
| Privacy-preserving verification | Yes | Local-first architecture per Sentinel principles |

### Timeline Projections

| Milestone | Estimated Timing | Confidence |
|-----------|-----------------|------------|
| Major browsers ship C2PA display | 2027-2028 | Medium |
| Social platforms preserve C2PA manifests | Gradual; 2026-2028 | Low-Medium |
| C2PA becomes widespread (>30% images) | 2028-2030 | Low |
| Regulatory mandates in effect | 2026-2027 (EU) | High |
| SynthID detector becomes public | 2026-2027 | Medium |
| Text provenance standard emerges | 2027-2029 | Low |

### Key Gaps Sentinel Could Fill

1. **The verification UX gap:** Provenance data exists but nobody sees it. A browser extension that surfaces it everywhere is immediate value.
2. **The text gap:** Text provenance has no good standard. Multi-signal text analysis (writing style + metadata + known AI patterns) could be Sentinel's killer feature.
3. **The "no provenance" signal:** In a world where most content lacks provenance, simply flagging "no provenance" and explaining what that means is valuable.
4. **The standards integrator:** Nobody currently shows C2PA + SynthID + IPTC + EXIF + blockchain + detection scores in one coherent view. Sentinel as the "trust aggregator" is a unique position.
5. **The privacy-first approach:** All existing verification tools either run in the cloud (verify.contentauthenticity.org) or are product-specific (Photoshop). A browser extension that verifies locally is novel.

---

## 10. Key References

### Specifications
- C2PA Specification v2.4: https://spec.c2pa.org/specifications/specifications/2.4/index.html
- C2PA Explainer v2.2: https://spec.c2pa.org/specifications/specifications/2.2/explainer/Explainer.html
- C2PA Security Considerations v2.4: https://spec.c2pa.org/specifications/specifications/2.4/security/Security_Considerations.html
- C2PA Guidance for AI/ML: https://spec.c2pa.org/specifications/specifications/2.3/ai-ml/ai_ml.html
- C2PA UX Recommendations v2.2: https://spec.c2pa.org/specifications/specifications/2.2/ux/UX_Recommendations.html

### Open Source Tooling
- c2pa-rs (Rust core): https://github.com/contentauth/c2pa-rs
- c2pa-js (JavaScript/TypeScript): https://github.com/contentauth/c2pa-js
- C2PA Tool (CLI): https://github.com/contentauth/c2pa-rs/tree/main/cli
- c2pa-python: https://opensource.contentauthenticity.org/docs/c2pa-python/

### Learning Resources
- Content Credentials Foundations Course: https://learn.contentauthenticity.org/
- CAI Docs: https://opensource.contentauthenticity.org/docs

### Organizations
- C2PA: https://c2pa.org/
- Content Credentials: https://contentcredentials.org/
- Content Authenticity Initiative: https://contentauthenticity.org/
- IPTC: https://iptc.org/
- SynthID: https://deepmind.google/technologies/synthid/

### Related Standards
- IPTC Photo Metadata: https://iptc.org/standards/photo-metadata/
- IPTC Video Metadata Hub: https://iptc.org/standards/video-metadata-hub/
- IPTC Media Provenance: https://iptc.org/media-provenance/
- W3C Verifiable Credentials: https://www.w3.org/TR/vc-data-model/

---

*Research compiled for Sentinel Phase 0 Discovery. All information sourced from official specifications, organizational websites, and verified public data as of July 2026.*
