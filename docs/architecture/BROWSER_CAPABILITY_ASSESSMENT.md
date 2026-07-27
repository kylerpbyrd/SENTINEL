# Trust Engine — Evidence Gathering Capability Assessment

> **Document Type:** Architecture Specification
> **Phase:** 1 — Architecture
> **Status:** Draft
> **Owner:** Browser Extension Engineer
> **Audience:** Architect Agent, ML Researcher, Trust Engine Designer, Product Manager
> **Scope:** Defines what evidence the Trust Engine can actually gather within a Manifest V3 browser extension context, and under what constraints.

---

## 1. Evidence Gathering Feasibility Matrix

Each evidence type is assessed on three axes: whether collection is technically possible, what constraints apply, and what confidence the resulting signal warrants.

### 1.1 Provenance & Metadata Evidence

| Evidence Type | Feasibility | Constraints | Signal Confidence |
|---|---|---|---|
| **C2PA manifest verification** | Feasible (local WASM via `c2pa-js`) | Must fetch image bytes via `fetch()` or `XMLHttpRequest`. Cross-origin images blocked by CORS unless server sends `Access-Control-Allow-Origin` matching extension origin. Same-origin images: full verification possible. Cross-origin images on cooperative servers (CDNs, news orgs): likely to work. Cross-origin images on uncooperative servers (social media): blocked. Manifest may be embedded in file (JUMBF box) or external via Soft Binding — external manifests require additional fetch. WASM bundle size ~1.5-3 MB for `c2pa-rs` compiled. | HIGH (when verifiable). Cryptographic signature verification provides strong, non-probabilistic signal. |
| **C2PA crJSON (text provenance)** | Feasible (local parsing) | Standard is new (C2PA 2.4). Near-zero adoption as of July 2026. Even when present, crJSON can be stripped by platforms. Fetch constraints same as image C2PA — cross-origin may be blocked. | HIGH (when present and verifiable). Currently near-zero availability. |
| **EXIF metadata reading** | Feasible (local, pure JS) | Can be read from same-origin images via `fetch()` of image bytes and parsing binary EXIF headers in JS. Cross-origin images: same CORS constraint. Many platforms (Twitter/X, Facebook, Instagram, WhatsApp) strip EXIF on upload. Screenshots have no EXIF. | MEDIUM (when present). Presence = signal. Absence = not informative alone (may have been stripped by platform, not creator). |
| **IPTC/IIM metadata reading** | Feasible (local) | Same CORS and stripping constraints as EXIF. IPTC is common in professional photography workflows but stripped by consumer platforms. | MEDIUM (when present). Absence has same caveat as EXIF. |
| **XMP metadata reading** | Feasible (local) | Embedded in file header. Same CORS and stripping constraints. XMP is the metadata container Adobe tools use. | MEDIUM (when present). |
| **ICC profile inspection** | Feasible (local) | Can be read from same-origin images. Minimal stripping (platforms rarely strip ICC profiles). Profile information indicates whether image passed through professional editing software. | LOW-MEDIUM. Corroborating signal only. |

### 1.2 In-Page Content Analysis Evidence

| Evidence Type | Feasibility | Constraints | Signal Confidence |
|---|---|---|---|
| **DOM text extraction** | Feasible (content script) | Can read `innerText`/`textContent` from DOM nodes the page exposes. Closed Shadow DOM (`mode: 'closed'`) is impenetrable — extension cannot access text inside closed shadow roots. Open Shadow DOM is accessible via `element.shadowRoot`. Dynamic content (SPAs, infinite scroll) requires MutationObserver or polling. `contenteditable` regions readable. | N/A (this is evidence extraction, not a signal itself). Feeds text analysis signals. |
| **DOM image element detection** | Feasible (content script) | Can enumerate `<img>`, `<canvas>`, `<video>` (poster frames), `<picture>` elements. CSS `background-image` requires `getComputedStyle()` — expensive if scanning all elements. Images loaded via `srcset` or `<picture>` with multiple resolutions: extension must determine which variant the browser selected. Lazy-loaded images (`loading="lazy"`, Intersection Observer-based) may not have loaded bytes yet. | N/A (extraction). Feeds image analysis signals. |
| **In-page image pixel analysis** | **PARTIAL** | **Same-origin images only.** Can draw image to `<canvas>` and read pixels via `getImageData()`. **Cross-origin images:** canvas becomes "tainted" — any `getImageData()` or `toDataURL()` call throws `SecurityError`. This is a hard browser security constraint with no workaround. `crossorigin="anonymous"` attribute only works if server sends `Access-Control-Allow-Origin` header. Social media, CDN images, and most web images are cross-origin and will fail. **Workaround:** `fetch()` the image bytes directly (if CORS permits), decode via `createImageBitmap()`, analyze in offscreen canvas. Same CORS dependency. **Conclusion:** Pixel analysis on arbitrary web images is unlikely to work without server cooperation. | HIGH (when same-origin). UNAVAILABLE (for cross-origin without CORS headers). |
| **In-page video frame analysis** | **Limited** | **tabCapture API:** Captures the current tab's video output including `<video>` elements. Chrome/Edge only (not Firefox, not Safari). Requires user gesture to initiate capture — cannot start silently. Returns a `MediaStream`; frames can be extracted via `VideoFrame` API or canvas draw. Canvas tainting does NOT apply to tabCapture (it captures rendered output, not cross-origin pixel data). **Constraints:** Resolution limited to viewport. Performance cost is high (decoding + frame analysis). User must explicitly invoke — cannot be always-on. Firefox: no tabCapture. Safari: no tabCapture. | MEDIUM (tabCapture). UNAVAILABLE (Firefox, Safari). |
| **In-page audio analysis** | **Limited** | **tabCapture API:** Same capture as video — can capture tab audio. Chrome/Edge only. Returns `MediaStreamTrack` for audio; can connect to `AudioContext` for analysis. **Alternative:** `AudioContext.createMediaElementSource()` can connect to an `<audio>` or `<video>` element on the page to analyze its audio output — but this works only for media elements on the page, not arbitrary page audio. CORS applies to media element sources. Firefox: `createMediaElementSource` works. Safari: works but limited AudioContext APIs. | MEDIUM (with tabCapture or media element source). LOW (Safari). |
| **Text from `<canvas>` elements** | **Limited** | Cannot read text rendered to canvas programmatically. Canvas is a pixel buffer. Would require OCR, which is computationally expensive and unreliable for arbitrary fonts/sizes/rotations. | VERY LOW. Not recommended as primary evidence source. |
| **Accessibility tree text** | **PARTIAL** | `aria-label`, `aria-description`, `alt` text on images, and other ARIA attributes are accessible via content script DOM traversal. This provides text descriptions of visual elements that may supplement analysis. Does not require special permissions. | LOW (supplementary only). Producers rarely provide accurate alt text for synthetic content. |

### 1.3 Image Forensics Evidence

| Evidence Type | Feasibility | Constraints | Signal Confidence |
|---|---|---|---|
| **Error Level Analysis (ELA)** | Feasible (local, JS/WASM) | Requires pixel-level access to image. Same CORS constraint as pixel analysis. Re-saving images at known quality level and comparing differences. Computationally moderate — can run in sandbox page or offscreen document. Works on same-origin images or images fetched with CORS. | MEDIUM. Can detect splicing and editing artifacts. Not AI-generation-specific. |
| **Noise pattern analysis** | Feasible (local, WASM) | Same pixel-level access requirement. Analyzing noise residuals for consistency. AI-generated images often have unnatural noise patterns distinct from camera sensors. Requires statistical computation in WASM for performance. | MEDIUM-HIGH. More specific to AI generation than ELA. |
| **JPEG compression artifact analysis** | Feasible (local) | Same pixel-level access requirement. Detects double-compression artifacts, grid anomalies consistent with inpainting/splicing. | MEDIUM. Corroborates with other forensics signals. |
| **Frequency domain analysis (DCT/FFT)** | Feasible (local) | Requires pixel access + moderate compute. DCT coefficient analysis can reveal GAN fingerprints. WASM implementation needed for performance. | MEDIUM-HIGH for known GAN architectures. Degrades on post-processed images. |
| **AI generator fingerprint detection** | Feasible (local, ONNX/WASM) | Lightweight classifier models (~5-20 MB) can detect telltale fingerprints from specific generators (DALL-E, Midjourney, Stable Diffusion, Adobe Firefly). Must be updated as new generators emerge. Same CORS constraint for pixel access. Can run in sandboxed page or offscreen document. | MEDIUM (model-dependent). Accuracy 75-90% on known generators. Drops 10-20% per new model generation. |

### 1.4 Text Analysis Evidence

| Evidence Type | Feasibility | Constraints | Signal Confidence |
|---|---|---|---|
| **Perplexity/burstiness analysis** | Feasible (local, ONNX) | Small language models (GPT-2-size, ~500 MB) compute perplexity in-browser via ONNX Runtime Web. Perplexity alone is not a reliable detector (Sadasivan et al., 2023). Text extracted from DOM via content script. Performance: moderate compute cost, can run asynchronously post-load. | LOW-MEDIUM. Mathematically proven to converge to random as LLMs improve. ALWAYS a corroborating signal, never standalone. |
| **Style consistency analysis** | Feasible (local) | Analyzes writing style consistency within a text block (sentence length variation, vocabulary diversity, transition patterns). Pure JS/statistical — no model required. Very lightweight. | LOW-MEDIUM. Corroborating only. Human writing can be inconsistent too. |
| **Repetition pattern analysis** | Feasible (local) | Detects formulaic structures, repeated phrases, template patterns. Extremely lightweight (pure JS). | LOW. Useful only as aggregate pattern across many samples. |
| **AI text watermark detection** | **PARTIAL** | **SynthID (Google):** Proprietary. Detection requires Google's API — not locally verifiable. No independent implementation. **OpenAI text watermark:** Not deployed as of July 2026. OpenAI has stated they have watermarking capability but are not using it due to accuracy concerns. **Other watermarks (Kirchenbauer et al.):** Statistical watermark detectable locally if algorithm is known and model output tokens are available. Not applicable to generic web text. | VERY LOW (practically unavailable for web content). |
| **Known AI content hash matching** | Feasible (local cache or cloud) | Local: periodic download of hash list for known AI-generated text samples. Match locally via rolling hash. Privacy-preserving. Cloud: real-time lookup against database. Requires sending hash, not content. | MEDIUM (when match found). LOW (no match = no signal). |

### 1.5 Video Forensics Evidence

| Evidence Type | Feasibility | Constraints | Signal Confidence |
|---|---|---|---|
| **Face manipulation detection (deepfake)** | Feasible (local, ONNX/WASM) | Requires tabCapture for frame access (Chrome/Edge only). Lightweight models analyze temporal consistency, eye blinking patterns, facial boundary artifacts. Performance: moderate-to-high compute. Can only analyze frames user is actively viewing. | MEDIUM (80-90% for face-swap deepfakes). LOW (fully AI-generated video — Sora, Runway). |
| **Video provenance verification** | Feasible (local if C2PA) | C2PA supports video manifests. Same `fetch()`/CORS constraints as images. Video files are large — downloading full video for C2PA verification is impractical. Manifest should be checked from initial bytes or external Soft Binding. | HIGH (when verifiable). Practically rare in web context. |
| **Video metadata analysis** | Feasible (local) | Container/codec metadata, creation timestamps, encoder fingerprints. Requires video file access (same CORS). Browser `<video>` element exposes limited metadata via `HTMLVideoElement` properties. | LOW-MEDIUM. Corroborating. |
| **Thumbnail/post-frame analysis** | Feasible (local if same-origin) | `<video poster>` attribute provides a still frame. Analyzable if same-origin. Cross-origin poster images: same canvas tainting. | LOW. Thumbnail is often manually selected, not representative. |

### 1.6 Audio Forensics Evidence

| Evidence Type | Feasibility | Constraints | Signal Confidence |
|---|---|---|---|
| **Voice cloning detection** | Feasible (local, ONNX/WASM) | Requires audio access via tabCapture (Chrome/Edge) or `AudioContext.createMediaElementSource()` from `<audio>`/`<video>`. Lightweight anti-spoofing models exist (~10-30 MB ONNX). Cannot analyze DRM-protected audio. | MEDIUM (80-90% for raw source audio). LOW after telephony compression or background noise. |
| **Audio deepfake detection** | Feasible (local, ONNX/WASM) | Same constraints as voice cloning detection. Fully synthetic speech detectable via vocoder artifacts when source is clean. Degrades rapidly with environmental noise, codec compression, or re-recording. | MEDIUM (clean source). LOW (compressed/noisy). |
| **Audio provenance verification** | Feasible (local if C2PA) | Same CORS constraint. Audio files smaller than video but still non-trivial. C2PA audio signing exists but adoption is minimal. | HIGH (when verifiable). Rare in practice. |

### 1.7 Reputation & Contextual Evidence

| Evidence Type | Feasibility | Constraints | Signal Confidence |
|---|---|---|---|
| **Domain reputation** | Feasible (local cached list + cloud) | Local: periodic download of domain reputation database. Stored in `chrome.storage.local`. Updated asynchronously. Cloud: real-time lookup by domain only (not full URL path). **Privacy consideration:** Cloud lookups reveal which domains the user is analyzing. Even domain-only lookups leak information. User must opt in. | MEDIUM (when from credible source). LOW for newly registered or obscure domains. |
| **Source/publisher credibility** | Feasible (cloud lookup) | Requires network request. Domain-level only (full URL path would be invasive). Trust graph must be built and maintained. | MEDIUM. Variable — credibility is subjective and domain history can change. |
| **Community/user reports** | Requires cloud | Sentinel cloud API for user-submitted trust assessments. Crowd-sourced reliability. Requires critical mass of users to be useful. Cold-start problem. Privacy: reports are user-contributed, not page content. | LOW (early stage). MEDIUM (with sufficient user base). |
| **Fact-check database lookup** | Requires cloud | Match claims on page against fact-check databases (e.g., Google Fact Check Tools API). Requires extracting factual claims from text, then sending claim hashes (not full text) to lookup service. | MEDIUM (when claim is in database). Not real-time — fact checks lag behind publication. |
| **Domain WHOIS/registration age** | Requires cloud | Newly registered domains (< 6 months) are disproportionately associated with disinformation. Can be cached locally with periodic updates. | LOW (corroborating only). Many legitimate sites use new domains. |
| **TLS certificate inspection** | Feasible (local, limited) | Extension can access `securityState` via `chrome.webNavigation` or `chrome.tabs`. Cannot directly inspect certificate chain details without additional APIs. EV/OV certificates indicate organizational identity. | VERY LOW. Let's Encrypt has democratized TLS — certificate type is barely informative. |

### 1.8 Watermark Detection Evidence

| Evidence Type | Feasibility | Constraints | Signal Confidence |
|---|---|---|---|
| **DALL-E visible watermark** | Feasible (local) | DALL-E images include a visible color band in the bottom-right corner. Detectable via pixel inspection on same-origin images. Cross-origin: requires CORS-enabled fetch. OpenAI announced C2PA migration — this watermark may be deprecated. | HIGH (when present). Binary signal. |
| **Midjourney invisible watermark** | Feasible (local) | Midjourney embeds a frequency-domain watermark. Detectable locally with known algorithm (Midjourney's approach is documented). Requires pixel access (same CORS constraint). | HIGH (when detected on full-resolution images). May not survive recompression. |
| **Adobe Firefly Content Credentials** | Feasible (local, C2PA) | Firefly outputs include C2PA manifests. Detection via standard C2PA verification. Same CORS constraint. | HIGH (when present and verifiable). |
| **Stable Diffusion invisible watermark** | **PARTIAL** | Stable Diffusion can embed watermarks via the SDXL-watermark or Tree-Ring approaches. Detectable locally if the embedding method is known. Multiple watermarking approaches exist — no single detection method works for all. | MEDIUM (method-dependent). LOW (if watermarking approach unknown). |
| **SynthID (Google)** | **UNAVAILABLE locally** | Proprietary technology. Detection requires Google's cooperation via their API. Not independently verifiable. Google has not opened the detection endpoint to third parties as of July 2026. | UNAVAILABLE. |
| **IPTC Digital Source Type** | Feasible (local) | IPTC standard metadata field indicating whether content was captured by camera, generated by AI, etc. Readable from image metadata if present. Same CORS constraint + platform stripping. | HIGH (when present). Currently very rare. |

### 1.9 Cross-Modal & Composite Evidence

| Evidence Type | Feasibility | Constraints | Signal Confidence |
|---|---|---|---|
| **Cross-modal consistency analysis** | **Limited** | Analyzing whether image content matches surrounding text context. Requires multi-modal model (vision-language). Computationally heavy. Local execution requires model ~200 MB+ via ONNX. Cloud: requires sending both text and image. Significant privacy implication. | MEDIUM (cloud with larger models). LOW (local lightweight models). |
| **Image-text semantic mismatch** | Feasible (local, lightweight) | Smaller vision-language models (CLIP-ViT, ~150 MB) can detect gross mismatches. "Article about Paris with photo of Tokyo" type errors. Can run in sandboxed page. | LOW-MEDIUM. Good for egregious mismatches. Not useful for subtle or adversarial cases. |
| **Multi-image consistency** | Feasible (local) | If a page has multiple images attributed to the same source, inconsistencies across them (different camera sensors, metadata discrepancies, stylistic drift) may indicate forgery. Requires pixel access (CORS) for each image. | LOW. Aggregate signal only. |
| **Temporal coherence** | **Limited** | Article published date vs. image capture date vs. domain registration date. Requires metadata access + cloud lookups for domain dates. Several independent asynchronous fetches. | LOW. Corroborating only. |

### 1.10 Writing Process & History Evidence

| Evidence Type | Feasibility | Constraints | Signal Confidence |
|---|---|---|---|
| **Google Docs version history** | **PARTIAL** | Only accessible if the user is viewing a Google Doc AND has authorized the extension to access Google Drive API. Cannot access without user authorization. Provides edit process replay, keystroke patterns, time spent — strong signal for human-authored content. | HIGH (when authorized). UNAVAILABLE (without authorization). |
| **Microsoft Word Online editing history** | **PARTIAL** | Similar constraint to Google Docs. Requires Microsoft Graph API authorization. | HIGH (when authorized). UNAVAILABLE (without authorization). |
| **Draft.js/ProseMirror edit history in-page** | **Limited** | If the page exposes edit history in a JavaScript object (e.g., collaborative editing tools), content script could potentially access it. Highly platform-specific. Not a general solution. | MEDIUM (when available). Rare in practice. |

---

## 2. Content Type Detection Specification

Before evidence gathering begins, the extension must determine what content types exist on the page to route to appropriate analyzers.

### 2.1 Content Type Taxonomy for Detection

The Trust Engine should recognize these content types:

| Content Type | Detection Method | Evidence Pipeline |
|---|---|---|
| **Static image** | `<img>` elements, `<picture>` elements, `<figure>` containing images, `<svg>` with embedded `<image>`, `<canvas>` elements | → Image forensics pipeline |
| **CSS background image** | `getComputedStyle(el).backgroundImage` parsing. **Expensive** — must be selectively applied to likely content containers, not every element. | → Image forensics pipeline (if same-origin) |
| **Video** | `<video>` elements, embedded players (YouTube iframe, Vimeo iframe, etc.) detected by domain pattern on `<iframe src>`. | → Video forensics pipeline (tabCapture required) |
| **Audio** | `<audio>` elements, embedded audio players. | → Audio forensics pipeline |
| **Text block** | `<p>`, `<article>`, `<blockquote>`, `<section>`, `<main>`, `<div>` with significant text content, social media post containers (identified by data attributes and class patterns). | → Text analysis pipeline |
| **Rich text / mixed** | Content with interleaved text and images (e.g., news articles, blog posts). | → All applicable pipelines, cross-modal analysis |
| **Document** | `<embed>` or `<iframe>` containing PDF, Google Docs, Office Online. Detection by MIME type or domain. | → Limited. PDF text extraction possible if same-origin `<embed>`. Cloud docs require API authorization. |

### 2.2 Social Media Content Container Detection

Social media platforms use platform-specific DOM structures. Detection heuristics:

| Platform | Image Container Selector | Text Container Selector | Notes |
|---|---|---|---|
| **Twitter/X** | `[data-testid="tweetPhoto"] img` | `[data-testid="tweetText"]` | Class names are hashed (CSS Modules). Must use `data-testid` attributes. |
| **Facebook** | `img[data-visualcompletion]`, images within `[role="article"]` | `[data-ad-comet-preview="message"]`, post content divs | DOM structure frequently changes. Heuristics fragile. |
| **Instagram** | `article img`, `[role="button"] img` | Post caption text within article elements | Heavily reliant on `<canvas>` and lazy loading. |
| **Reddit** | `shreddit-post img`, post media containers | `[slot="text-body"]`, markdown-rendered divs | Web component-based (Shadow DOM). Open shadow roots accessible. |
| **LinkedIn** | Post media images within feed articles | Post text containers, article content | Professional content. DOM relatively structured. |
| **TikTok** | Video/SVGs in feed | Caption text elements | Primarily video. Very limited text. |
| **YouTube** | Thumbnails, channel avatars | Video descriptions, comments | Text is user-generated and unstructured. |

**Critical caveat:** Platform DOM structures change without notice. Detection selectors break. The extension must degrade gracefully — if a social media container cannot be identified with high confidence, treat the content as generic web text/images rather than failing.

### 2.3 Content Types the Trust Engine Should IGNORE

Not all page content is meaningful for trust assessment. Intentional exclusions:

| Content to Ignore | Rationale |
|---|---|
| **Navigation elements** (`<nav>`, menus, breadcrumbs) | Not user-generated content. UI chrome of the site. |
| **Advertisements** | Third-party injected. Not the page's content. Can be heuristically identified by common ad container classes, iframe origins from known ad networks (`doubleclick.net`, `googlesyndication.com`, etc.). |
| **Comments sections** (by default) | User-generated commentary, not the content being analyzed. **Exception:** When user explicitly wants to assess community sentiment/trust. Then treat as separate content domain. |
| **Footer/header boilerplate** | Copyright notices, navigation links, site-wide repeated text. |
| **Sidebar widgets** | "Related posts," "Trending now," weather widgets, etc. |
| **Cookie consent banners** | Regulatory UI, not content. |
| **Chat widgets / intercom** | Third-party customer support overlay. |
| **Embeds from known non-content domains** | Analytics scripts, social share buttons, tracking pixels. |
| **SVG icons / UI sprites** | Decorative/functional, not content. |
| **Images smaller than 50x50 pixels** | Icons, avatars, tracking pixels. Below minimum threshold for meaningful forensics. |
| **`contenteditable` regions where user is actively typing** | User's own input, not content they're evaluating. |
| **Password/credit card input fields** | MUST NOT touch under any circumstances. Security/privacy. |
| **Form inputs with `type="hidden"`** | Not user-visible content. |

### 2.4 Content Type Detection Strategy

**Phase 1 (synchronous, on DOM ready):**
1. Identify main content region using semantic HTML5 landmarks (`<main>`, `<article>`, `role="main"`, `role="article"`).
2. If no semantic landmarks found, use heuristics: largest `<div>` by text content, element at center of viewport, element containing the most `<p>` tags.
3. Enumerate text blocks, images, videos, audio within the identified content region.
4. Classify page type (article, social media post, gallery, video page, etc.) based on DOM structure + URL patterns.

**Phase 2 (asynchronous, post-load):**
1. Wait for dynamic content to settle (MutationObserver-based).
2. Detect social media platform and apply platform-specific container selectors.
3. Classify content types within each container.
4. Prioritize analysis queue: text first (fastest), then metadata, then images, then video/audio (slowest, user-invoked).

---

## 3. Page Structure Challenges

### 3.1 Single Page Applications (SPAs)

**Challenge:** Content is rendered client-side via JavaScript after initial HTML load. Traditional `DOMContentLoaded` fires before content exists.

**Handling:**
- Use `MutationObserver` to detect DOM mutations and identify when content stabilizes.
- Apply a debounce window (~500ms of no mutations) before triggering analysis to avoid analyzing mid-render partial content.
- For navigation events within SPAs (URL change without page reload), use `chrome.webNavigation.onHistoryStateUpdated` or `popstate` event listeners to re-trigger content type detection.
- **Constraint:** MutationObserver on entire document body is expensive. Scope observation to content containers after initial detection.

**Confidence impact:** None, once content is captured. The challenge is timing, not accuracy.

### 3.2 Infinite Scroll

**Challenge:** New content continuously appended. Analysis of initial viewport content may be incomplete as more content loads.

**Handling:**
- Analyze content visible in viewport at time of invocation.
- For on-demand analysis: analyze what's present at click time. Do not wait for infinite scroll to "finish" — it never does.
- For automatic/per-domain analysis: observe newly appended content via MutationObserver. Only analyze new items, not the entire page again.
- Display confidence with a caveat: "Analysis based on X items visible. Scroll to load more for complete analysis."

**Confidence impact:** LOW. Incomplete content set limits assessment.

### 3.3 Shadow DOM

**Challenge:** Closed shadow roots (`mode: 'closed'`) prevent all access. Content within is invisible to the extension.

**Handling:**
- Open shadow roots: accessible via `element.shadowRoot`. Traverse recursively.
- Closed shadow roots: **no workaround exists.** Content inside is permanently inaccessible.
- Platform impact: Web components using closed shadow DOM (increasingly common) will have content gaps.
- Flag in analysis output: "X elements were inaccessible due to closed Shadow DOM."

**Confidence impact:** MEDIUM (if significant content is in closed shadow roots, assessment is unreliable). This should be communicated to the user.

### 3.4 Cross-Origin iframes

**Challenge:** Content scripts cannot access cross-origin iframe content. Each cross-origin iframe is an isolated browsing context.

**Handling:**
- Same-origin iframes: content script can access via `iframe.contentDocument`.
- Cross-origin iframes: **cannot access DOM, cannot read content.** Separate extension permissions for the iframe origin would be required (`"all_frames": true` + host permissions for each iframe origin).
- **Strategy:** Do not attempt to analyze cross-origin iframe content. Flag its presence as a gap.
- Common cross-origin iframe content: embedded YouTube videos, Twitter embeds, ad networks, payment forms.

**Confidence impact:** MEDIUM-HIGH (embedded content often critical — YouTube embeds in news articles are evidence).

### 3.5 Lazy-Loaded Images

**Challenge:** Images with `loading="lazy"` or Intersection Observer-based lazy loading have not been fetched when analysis begins.

**Handling:**
- For on-demand analysis: scroll to trigger load, then analyze. Or `fetch()` image URLs with CORS — may succeed even if browser hasn't loaded them for display.
- For automatic analysis: wait for images within viewport to load via `img.complete` and `img.naturalWidth > 0`. Flag non-loaded images as "not analyzed."
- Images far below the fold: not analyzed unless user scrolls.

**Confidence impact:** LOW-MEDIUM. Incomplete image analysis set limits overall assessment.

### 3.6 Paywalled Content

**Challenge:** Extension can read paywalled content if the user has access (logged in). But the extension cannot distinguish between what the user sees and what's behind the paywall.

**Handling:**
- Analyze what's accessible. Flag that content behind paywall was not analyzed.
- If the page is mostly paywalled (e.g., only a teaser paragraph is visible), the analysis is likely misleading — flag this prominently.

**Confidence impact:** HIGH. Analysis of a paywalled article's teaser paragraph is not representative of the full content. Must communicate this.

### 3.7 A/B Testing

**Challenge:** Different users see different content on the same URL. No reliable detection mechanism exists — the extension sees what the user sees and cannot know about variants.

**Handling:**
- No mitigation possible. This is an inherent limitation of client-side analysis.
- Analysis output should note that assessments are based on the content presented to this user at this time.

**Confidence impact:** LOW. But no practical mitigation exists.

### 3.8 Dynamic Ad Insertion / Personalization

**Challenge:** Content mixed with personalized ads and recommendations. The extension cannot distinguish "article content" from "personalized recommendation widget" with perfect accuracy.

**Handling:**
- Use content container heuristics (Section 2.3) to exclude known ad patterns.
- Flag residual uncertainty: "X% of analyzed text may include non-editorial content."

**Confidence impact:** LOW-MEDIUM.

---

## 4. Performance Boundary Constraints

The Discovery Report identified > 500ms page load impact = uninstall. This sets strict boundaries on what evidence can be gathered and when.

### 4.1 Evidence Gathering Timing Model

| Timing Window | Permitted Evidence | Rationale |
|---|---|---|
| **Synchronous (page load, < 50ms)** | Content type detection (DOM landmark scan only), page URL extraction, basic element enumeration (counts, types). | Must not block page rendering. Absolute budget: < 50ms in content script `run_at: document_start` or `document_idle`. |
| **Immediate async (0-300ms post-load)** | DOM text extraction from main content region, metadata extraction (EXIF/IPTC from same-origin images via `fetch()`), C2PA verification initiation (WASM load deferred). | Non-blocking. User sees page while analysis begins. |
| **Deferred async (300ms-2s post-load)** | C2PA verification completion, domain reputation lookup (local cache), lightweight text analysis (perplexity via pre-loaded model), content type classification refinement. | Background processing. User is already interacting with page. |
| **User-invoked only (> 2s or on-demand)** | Image pixel forensics (ELA, noise analysis, frequency domain), video frame analysis (tabCapture), audio analysis, cross-modal analysis with multi-modal models, cloud lookups. | These operations are expensive (> 100ms each) or require user interaction (tabCapture). Never run automatically. |

### 4.2 Synchronous vs. Asynchronous vs. Deferred — By Evidence Type

| Evidence Type | Timing Classification | Budget |
|---|---|---|
| Content type detection (DOM landmarks) | Synchronous | < 30ms |
| Page URL / domain extraction | Synchronous | < 1ms |
| DOM text extraction (main content only) | Immediate async | < 50ms |
| `<img>` enumeration + `src` collection | Immediate async | < 30ms |
| Same-origin image metadata fetch (EXIF/IPTC) | Deferred async | < 200ms per image |
| C2PA WASM load + verification | Deferred async | 500ms-2s for initial load; 50-200ms per subsequent verification |
| Local domain reputation cache lookup | Deferred async | < 5ms |
| Lightweight text style/pattern analysis | Deferred async | < 100ms |
| Text perplexity model inference (ONNX, small model) | Deferred async | 200-500ms for inference |
| Image pixel analysis (ELA/noise/frequency) | User-invoked | 200ms-2s per image |
| Image forensics model inference (ONNX) | User-invoked | 500ms-2s per image |
| Video frame analysis (tabCapture) | User-invoked | 2-10s |
| Audio analysis | User-invoked | 1-5s |
| Cross-modal analysis (CLIP-ViT) | User-invoked | 1-3s |
| Cloud lookups (domain reputation, fact-check, reverse image) | User-invoked or opt-in background | Network-dependent; 200ms-3s |
| Hash list download (known AI content) | Background (periodic) | Minutes; performed during idle |

### 4.3 Progressive Analysis Model

The Trust Engine should produce a **preliminary assessment** with fast signals, then refine with slower signals as they become available.

**Tier 1 — Preliminary (available in < 500ms):**
- Domain/URL classification
- Content type identification
- Content volume metrics (word count, image count, video count)
- Cached domain reputation (if available)
- "No provenance found" signal for images lacking C2PA

**Tier 2 — Refined (available in 1-5s):**
- C2PA verification results
- EXIF/IPTC metadata analysis
- Text style/pattern analysis
- Basic image metadata (dimensions, format, file size)
- Cloud domain reputation lookup (if enabled)

**Tier 3 — Deep (user-invoked or opt-in auto):**
- Image forensics (ELA, noise, frequency domain)
- AI generator fingerprint model results
- Video frame analysis
- Audio analysis
- Cross-modal consistency
- Reverse image search results

**UX implication:** The extension UI must show evidence being gathered progressively. A spinner with "Analyzing..." without showing intermediate results fails the explainability principle. Show: "Domain check: complete. Metadata: found X of Y images. C2PA: verifying... Image analysis: pending (click to run)."

### 4.4 WASM Model Loading Strategy

Models compiled to WASM (C2PA verification: ~1.5-3 MB; ONNX image forensics: ~5-20 MB; CLIP-ViT: ~150 MB) present a loading challenge:

- **C2PA (`c2pa-js`):** Load lazily on first C2PA verification request. Cache compiled WASM in `chrome.storage.session` (memory) for duration of browser session. Reload if service worker terminates.
- **ONNX Runtime Web:** Load at extension startup (service worker wake). Lightweight runtime (~5 MB). Specific models loaded on-demand.
- **ONNX image forensics models:** Load on first image analysis invocation. Cache in `chrome.storage.session`. Total model size must stay within `chrome.storage.session` limit (~10 MB per key, quota varies).
- **Large models (CLIP-ViT, ~150 MB):** Cannot fit in `chrome.storage.session`. Either use `IndexedDB` (available in offscreen documents, not service workers) or stream from bundled assets on each use. Defer to user-invoked only.
- **Chrome built-in AI APIs (Gemini Nano):** Emerging alternative for text analysis. No model download needed — uses device model. Chrome-only. Should be used when available as zero-overhead text analysis.

### 4.5 Memory Budget

| Component | Approximate Memory | Notes |
|---|---|---|
| Content script | < 5 MB | Minimal. Only DOM traversal + message passing. |
| Service worker | < 20 MB | Orchestration, message routing, cache management. No large models. |
| Sandbox page / offscreen document | 50-200 MB | Model inference. One at a time. Page can be terminated to free memory. |
| Total extension | < 300 MB peak | Acceptable for modern browsers. Must release memory after analysis completes. |

---

## 5. Browser-Specific Capability Gaps

The Trust Engine specification must acknowledge that evidence quality varies by browser. Sentinel's confidence system should integrate browser capability awareness.

### 5.1 Capability Matrix by Browser

| Capability | Chrome | Edge | Firefox | Safari |
|---|---|---|---|---|
| **Manifest version** | MV3 | MV3 | MV3 | Proprietary (Safari Web Extensions) |
| **Background context** | Service worker (30s idle timeout) | Service worker (30s idle timeout) | Event pages (persistent-ish, terminate when idle) | Background scripts (native app wrapper) |
| **`c2pa-js` WASM verification** | Supported | Supported | Supported | Supported (WASM works in Safari) |
| **ONNX Runtime Web** | Supported (WebGPU) | Supported (WebGPU) | Supported (WASM only) | Supported (WASM only) |
| **WebGPU** | Supported | Supported | Supported (behind flag) | Not supported |
| **Offscreen documents** | Supported | Supported | Not supported | Not supported |
| **tabCapture API** | Supported | Supported | Not supported | Not supported |
| **sidePanel API** | Supported | Supported | Not supported (use browserAction popup) | Not supported |
| **`chrome.storage.session`** | Supported | Supported | Not supported (use `browser.storage.local`) | Not supported |
| **Chrome built-in AI APIs** | Supported (experimental) | Not supported | Not supported | Not supported |
| **`declarativeNetRequest`** | Supported | Supported | Supported | Supported |
| **`activeTab` permission** | Supported | Supported | Supported | Supported |
| **`alarms` API** | Supported (min 30s) | Supported (min 30s) | Supported (min 1s) | Not supported |
| **Extension storage quota** | 10 MB (local), memory-only (session) | Same as Chrome | ~5 MB (local), unlimited with `unlimitedStorage` | ~5 MB |
| **Content script injection on file:// URLs** | Supported (with flag) | Supported (with flag) | Supported | Not supported |

### 5.2 Evidence Availability by Browser

| Evidence Type | Chrome/Edge | Firefox | Safari |
|---|---|---|---|
| **C2PA verification** | Full (WASM) | Full (WASM) | Full (WASM, slower) |
| **EXIF/IPTC metadata** | Same-origin only | Same-origin only | Same-origin only |
| **Image pixel forensics** | Same-origin only | Same-origin only | Same-origin only |
| **AI generator fingerprint models** | Full (ONNX + WebGPU) | Good (ONNX WASM) | Limited (ONNX WASM, slower) |
| **Text analysis (perplexity)** | Full (ONNX + Chrome AI) | Good (ONNX WASM) | Limited (ONNX WASM, slower) |
| **Video frame analysis** | Full (tabCapture) | **UNAVAILABLE** | **UNAVAILABLE** |
| **Audio analysis** | Full (tabCapture) | Limited (media element source only) | Limited (media element source only) |
| **Domain reputation (local cache)** | Full | Full | Full |
| **Chrome built-in AI (Gemini Nano)** | Supported | **UNAVAILABLE** | **UNAVAILABLE** |
| **Offscreen document for heavy compute** | Supported | **UNAVAILABLE** (must use sandbox page) | **UNAVAILABLE** |
| **Long-running background analysis** | Limited (30s SW timeout) | Better (Event pages) | Better (background script in native app) |
| **Cross-modal analysis (CLIP-ViT)** | Full (WebGPU) | Good (slower WASM) | Limited (slow WASM) |

### 5.3 Confidence Tier Degradation by Browser

The Trust Engine should report confidence tiers differently based on the browser:

| Browser | Confidence Profile | Evidence Reduced |
|---|---|---|
| **Chrome (full capabilities)** | Full confidence | None. All evidence types available (when user-invoked). |
| **Edge (near-full)** | Full confidence | None substantive. Chrome built-in AI APIs may be absent. |
| **Firefox (reduced)** | Moderate confidence | No video frame analysis (tabCapture missing). No audio tab capture. Heavier compute slower (no WebGPU, no offscreen docs). Permanent background processing slightly better (Event pages). |
| **Safari (degraded)** | Reduced confidence | No video frame analysis. No audio tab capture. No WebGPU. No offscreen documents. No sidePanel. Slower model inference. Extension store approval path more restrictive. |

**Specification requirement:** The Trust Engine output must include a `browser_capability_profile` field indicating which evidence types were unavailable due to browser limitations. This feeds into the overall confidence score: a "low trust" assessment on Safari should note that some evidence signals were unavailable and the assessment might differ on a full-capability browser.

### 5.4 Firefox Advantage: Event Pages

Firefox's Event Pages (not service workers) can persist longer than Chrome's 30-second service worker timeout. This enables:

- **Longer model warm-up:** ONNX models can stay loaded in memory across multiple page analyses.
- **Better progressive analysis:** Evidence gathering that exceeds 30 seconds (e.g., analyzing many images on a gallery page) won't be killed mid-operation.
- **Periodic background tasks:** Hash list updates, reputation list downloads can run to completion.

This should be noted in the architecture: the Trust Engine can process more evidence per invocation on Firefox than on Chrome due to the longer runtime window.

---

## 6. Offline Degradation Specification

### 6.1 Evidence Types by Online/Offline Availability

| Evidence Type | Online | Offline | Degradation |
|---|---|---|---|
| **C2PA manifest verification** | Full | Full | None. Entirely local. |
| **EXIF/IPTC/XMP metadata reading** | Full | Full | None. Entirely local. |
| **DOM text extraction** | Full | Full | None. Reads content already loaded in browser. |
| **Text style/pattern analysis** | Full | Full | None. Statistical, no model required. |
| **Text perplexity analysis** | Full | Full | None. Local ONNX model. |
| **Image pixel forensics** | Full | Full | None. Local computation. |
| **AI generator fingerprint detection** | Full | Full | None. Local ONNX model. |
| **Invisible watermark detection (DALL-E, Midjourney)** | Full | Full | None. Local algorithm. |
| **SynthID detection** | UNAVAILABLE | UNAVAILABLE | Requires Google API (online). Unknown if endpoint is publicly accessible. |
| **Domain reputation (local cache)** | Full | Full (stale cache) | Cache freshness degrades. Last update timestamp shown. |
| **Domain reputation (real-time)** | Full | UNAVAILABLE | Cannot refresh. Stale cache used if available. |
| **Source/publisher credibility lookup** | Full | UNAVAILABLE | No offline equivalent. |
| **Fact-check database lookup** | Full | UNAVAILABLE | No offline equivalent. |
| **Reverse image search** | Full | UNAVAILABLE | No offline equivalent. |
| **Known AI content hash matching (cloud)** | Full | UNAVAILABLE | Local hash cache still functions if recently synced. |
| **Community/user reports** | Full | UNAVAILABLE | No offline equivalent. |
| **Video/audio analysis** | Full (tabCapture) | Full (tabCapture) | tabCapture is local. No offline impact. |

### 6.2 Offline Confidence Communication

When the extension detects offline operation, the Trust Engine must:

1. **Tag assessment** with `"operational_mode": "offline"` indicating reduced evidence availability.
2. **List unavailable evidence** explicitly: which evidence types could not be gathered because they require network access.
3. **Adjust confidence indicators:** A score of "78% confidence" online might drop to "63% confidence" offline because 15 percentage points of evidence were unavailable.
4. **Show stale data warnings:** If cached domain reputation is > 7 days old, display: "Domain reputation data is X days old. Assessment may not reflect recent changes."
5. **Provide re-analysis prompt:** "This assessment was performed offline. Connect to the internet for complete analysis."

### 6.3 Offline-First Design Principles

- All evidence that CAN be gathered locally SHOULD be gathered regardless of online/offline state.
- Evidence that requires network SHOULD be gathered when online and skipped gracefully when offline.
- Users SHOULD NOT receive error messages when offline. They SHOULD receive reduced-confidence assessments.
- On reconnection, cached assessments SHOULD be automatically refreshed with cloud-only evidence.

---

## 7. Invocation Model vs. Evidence Availability

### 7.1 Invocation Models

| Model | Trigger | Permission Required | User Awareness | Privacy Risk |
|---|---|---|---|---|
| **On-demand ("Analyze this page")** | User clicks extension icon or sidebar button | `activeTab` | Explicit — user initiates | Lowest. Only current tab, only when asked. |
| **Per-domain automatic** | Page load on user-allowlisted domains | Host permissions (granted at domain-allowlist time) | Semi-explicit — user chose the domain | Medium. Content on allowlisted domains analyzed automatically. |
| **Always-on passive** | Every page load | `<all_urls>` or broad host permissions | Implicit — user installed extension | High. All browsing analyzed. Privacy concern. |
| **Content menu ("Analyze this image")** | User right-clicks specific image | `activeTab` + contextMenus | Explicit — user targets a single element | Lowest. Only the selected element. |
| **Share/flow (mobile)** | User shares content to Sentinel from another app | Share Extension (iOS) / Share Intent (Android) | Explicit — user action | Lowest. Only shared content. |

### 7.2 Evidence Available by Invocation Model

| Evidence Type | On-demand | Per-domain Auto | Always-on Passive | Context Menu | Mobile Share |
|---|---|---|---|---|---|
| Content type detection | Full | Full | Full | Single element only | Full |
| DOM text extraction | Full | Full | Limited (full text extraction on every page load is too expensive) | N/A | Full (content shared) |
| EXIF/IPTC metadata | Full | Full (deferred) | Metadata only (no pixel analysis) | Full (single image) | Full (single file) |
| C2PA verification | Full | Full (deferred) | Metadata only | Full (single image) | Full (single file) |
| Image pixel forensics | Full (user expects wait) | Deferred (background, may not complete before tab close) | **NOT SUITABLE** — too expensive per page | Full (single image) | Full (single image) |
| AI generator fingerprint models | Full | Deferred | **NOT SUITABLE** — per-image cost too high | Full | Full |
| Text analysis (style/pattern) | Full | Full | Full (lightweight only) | N/A | Full |
| Text perplexity (ONNX) | Full | Deferred (costly) | **NOT SUITABLE** — too expensive per page | N/A | Full |
| Video frame analysis (tabCapture) | Full (user-invoked) | **NOT SUITABLE** — requires user gesture per tabCapture | **NOT SUITABLE** | N/A | Limited (shared clip) |
| Audio analysis | Full (user-invoked) | **NOT SUITABLE** — requires user gesture per tabCapture | **NOT SUITABLE** | N/A | Limited (shared clip) |
| Domain reputation lookup (cached) | Full | Full | Full | Full | Full |
| Cloud lookups | Full (opt-in) | Per-domain opt-in | **NOT SUITABLE** — privacy violation | Full (opt-in) | Full (opt-in) |
| Reverse image search | Full (opt-in) | **NOT SUITABLE** — automatic image upload is privacy violation | **NOT SUITABLE** | Full (opt-in) | Full (opt-in) |

### 7.3 Recommended Invocation Model Tiers

**Tier 1 — On-demand ("Analyze this page"):**
- Default mode. Minimal permissions (`activeTab`). Highest privacy posture.
- All evidence types available, user-initiated.
- Best for: MVP launch, privacy-conscious users, journalists verifying specific content.
- Limitation: Extra click per page. User must remember to use it.

**Tier 2 — Per-domain automatic:**
- User explicitly adds domains (e.g., `twitter.com`, `facebook.com`, `reddit.com`) to an allowlist.
- Host permissions granted at allowlist time via `chrome.permissions.request()`.
- Lightweight evidence gathered automatically on page load (metadata, domain reputation, text patterns). Heavyweight analysis (pixel forensics, model inference) deferred until user invokes.
- Best for: Power users who want continuous monitoring on specific social platforms.
- Limitation: Heavier permissions. Must justify to store reviewers.

**Tier 3 — Always-on passive (deferred to Phase 3+):**
- NOT recommended for MVP. Requires strong privacy justification for store review.
- Evidence limited to fast, non-invasive signals: domain reputation (cached), page structure classification, content volume metrics.
- Full analysis only on explicit user invocation.
- Best for: Enterprise deployments where organization mandates continuous monitoring.
- Limitation: Privacy risk. Store rejection risk. User trust risk.

### 7.4 Privacy Implications of Each Invocation Model

| Model | Data Leaves Device | User Exposed | Store Review Risk |
|---|---|---|---|
| On-demand (`activeTab`) | No (local-only default); only if user enables cloud features | No (user controls exactly which pages) | LOW |
| Per-domain automatic | No by default; cloud lookups only if opted in | Domains on allowlist exposed to extension; user chose these | MEDIUM — must justify why specific domains need auto-analysis |
| Always-on passive | No by default; cloud lookups only if opted in | All browsing domains visible to extension | HIGH — broad host permissions scrutinized; privacy policy must be robust |
| Context menu | Same as on-demand | No (single element, not page) | LOW |
| Mobile Share | Depends on processing tier (local vs cloud) | Only content explicitly shared | LOW (user-initiated, no background access) |

---

## 8. Summary: What the Trust Engine Can Actually Do

### Evidence Tiers (What to Build)

| Tier | Evidence Signals | Availability | Performance | Privacy |
|---|---|---|---|---|
| **Tier 1: Always Available** | C2PA verification, EXIF/IPTC metadata, DOM text extraction, text style/pattern analysis, cached domain reputation, content type classification | All browsers, online/offline, on-demand | < 500ms combined | Local only |
| **Tier 2: Available with Constraints** | Image pixel forensics (same-origin only), AI generator fingerprint models (same-origin only), text perplexity (ONNX, deferred), invisible watermark detection (same-origin) | All browsers (slower without WebGPU), on-demand or per-domain deferred | 1-5s per image | Local only (models bundled) |
| **Tier 3: Browser-Dependent** | Video frame analysis (tabCapture: Chrome/Edge only), audio analysis (tabCapture or media element source), cross-modal analysis (WebGPU beneficial) | Chrome/Edge full; Firefox partial; Safari unavailable | 2-10s | Local (tabCapture) or local (models) |
| **Tier 4: Cloud-Dependent** | Reverse image search, real-time domain reputation, fact-check lookup, community reports, SynthID detection (if API accessible) | All browsers, online only, opt-in only | Network-dependent | Content may leave device |

### Key Architectural Constraints (Non-Negotiable)

1. **Never run pixel analysis on cross-origin images.** Canvas tainting is a hard browser constraint. Attempting to bypass it violates browser security model and Chrome Web Store policy.
2. **Never extract evidence from password/credit card/input fields.** Security requirement.
3. **Never make cloud requests without explicit user opt-in.** Privacy requirement. Also required by store policies.
4. **Never exceed 50ms synchronous work on page load.** Performance requirement (> 500ms impact = uninstall).
5. **Never assume C2PA/IPTC metadata will be present.** Most web content has no provenance metadata. Absence is a signal, not an error.
6. **Never present a single-signal verdict.** Always show evidence fusion. Explainability requirement. Legal risk mitigant.
7. **Never store full page content or URLs in `chrome.storage.local`.** Unencrypted storage. Privacy requirement.
8. **Test on Firefox and Safari, not just Chrome.** Architecture must accommodate browser capability gaps. Design for the lowest common denominator, enhance for capability.

### Evidence NOT Available (Hard Limits)

These evidence types the Trust Engine **cannot** gather, regardless of browser, regardless of permissions:

- **Cross-origin image pixels** (without server CORS cooperation) — canvas tainting is a web security primitive
- **Closed Shadow DOM content** — browser-enforced encapsulation
- **Network response bodies** — MV3 `declarativeNetRequest` is header-only
- **SynthID verification** without Google API access — proprietary technology
- **Writing process history** for arbitrary web text (only works with authorized Google Docs/Microsoft Word Online access)
- **Content behind login walls** the user hasn't authenticated to
- **DRM-protected audio/video content** — encrypted media extensions block access
- **Content on `chrome://`, `about:`, `edge://`, Firefox internal pages** — extension APIs blocked on browser-internal URLs
- **Chrome Web Store / AMO pages** — extensions cannot inject into store pages

---

*End of Trust Engine Evidence Gathering Capability Assessment. This document defines WHAT the Trust Engine can collect, not HOW it collects it. Implementation patterns, message passing design, and code architecture are separate concerns addressed in ADR-003 (Trust Engine Architecture).*
