# AGENTS.md — AI Agent Governance

> **Project:** Sentinel | **Owner:** Byrd Ventures LLC
> **Document Type:** Agent Rules (binding for all AI agents)
> **Version:** 1.0

---

## Project Mission

Sentinel is a **Trust Layer for the Internet** — a privacy-first platform that analyzes digital content for authenticity, provenance, and trustworthiness using explainable, evidence-based confidence scoring rather than binary AI detection.

The browser extension is the first product. It is not the company.

---

## Agent Responsibilities

Every AI agent working on Sentinel must:

1. **Read this document before any work.**
2. **Read the referenced architecture documents** relevant to the task.
3. **Respect the authority of previously made decisions.** Do not silently overturn ADRs or architecture specifications.
4. **Produce work that aligns with the project's principles:** privacy-first, explainable, evidence-based, user-controlled.
5. **Document assumptions.** If you assume something not in the architecture, state it explicitly.

---

## Architecture Protection Rules

The following documents are **authoritative and binding.** Agents must not contradict them:

| Document | Authority |
|----------|-----------|
| `SENTINEL_PROJECT_PLAN.md` | Project governance, phases, principles |
| `VISION.md` | Product vision, mission, philosophy |
| `docs/discovery/DISCOVERY_REPORT.md` | Phase 0 consolidated research findings |
| `docs/TRUST_ENGINE_SPEC.md` | Product reasoning specification (what the Trust Engine does) |
| `docs/architecture/TRUST_ENGINE_ARCHITECTURE_MASTER.md` | System architecture (how the Trust Engine is built) |
| `docs/architecture/BROWSER_CAPABILITY_ASSESSMENT.md` | Browser capability assessment (binding technical constraints) |
| `docs/architecture/EVIDENCE_PIPELINE.md` | Evidence object model and pipeline |
| `docs/architecture/FUSION_ENGINE_ARCHITECTURE.md` | Fusion engine and confidence model |
| `docs/architecture/SCHEDULER_AND_PLUGIN_ARCHITECTURE.md` | Scheduler and plugin architecture |
| `docs/architecture/TRUST_ENGINE_ARCHITECTURE.md` | Invocation, privacy, performance, failure |
| `docs/architecture/VALIDATION_PLAN.md` | Technical spike plan and exit criteria |
| `docs/architecture/VALIDATION_HARNESS.md` | Validation platform design |
| `REQUIREMENTS.md` | Functional and non-functional requirements |
| `ARCHITECTURE.md` | Architecture index and overview |

**If an architecture document and its implementation contradict, the architecture document is correct until an ADR amends it.**

---

## Coding Standards

1. **TypeScript** for all extension code. Strict mode. No `any` without explicit justification.
2. **Immutability by default.** Evidence objects, fusion results, and assessment outputs are frozen after construction.
3. **Explicit error handling.** No swallowed exceptions. Every failure path must produce structured output.
4. **No `eval()`, `new Function()`, or dynamic code execution.** MV3 CSP prohibits these.
5. **Minimal dependencies.** Every dependency must be justified in an ADR or architecture document.
6. **Model weights must be signed and verified** before loading.
7. **All inter-context communication must use typed message interfaces.**

---

## Documentation Requirements

1. **Every subsystem must have a README or architecture document** explaining its purpose, inputs, outputs, and dependencies.
2. **Every public interface must be documented** with JSDoc comments describing parameters, return values, and error conditions.
3. **Every ADR must follow the template:** Problem → Context → Options Considered → Decision → Tradeoffs → Consequences.
4. **Documentation is part of the product.** Undocumented code is considered incomplete.

---

## Testing Requirements

1. **Every analyzer plugin must have unit tests** verifying output format, confidence calibration, and error handling.
2. **Every pipeline stage must have integration tests** verifying input/output contracts.
3. **The Fusion Engine must have deterministic tests** — same evidence graph → same confidence assessment.
4. **Cross-context messaging must have end-to-end tests** across all browser contexts.
5. **Performance budgets must be verified in CI** — any regression blocks merge.

---

## Forbidden Actions

Agents must **NEVER:**

1. **Modify architecture documents without an ADR.**
2. **Add permissions to the extension manifest without security review.**
3. **Send user content to cloud services without explicit opt-in consent architecture.**
4. **Access password fields, form data, cookies, or browsing history.**
5. **Embed API keys, secrets, or credentials in source code.**
6. **Use `any` type in TypeScript without documented justification.**
7. **Silently change confidence scoring thresholds or evidence weights.**
8. **Remove or disable safety overrides (C2PA floor, known synthetic hash floor).**
9. **Collect telemetry without opt-in consent architecture.**
10. **Output binary "AI" or "Human" verdicts** — always confidence assessments.

---

## Git Workflow Rules

1. **Branch naming:** `feature/description`, `fix/description`, `spike/description`, `docs/description`
2. **Commit messages:** Conventional Commits format (`feat:`, `fix:`, `docs:`, `chore:`, `test:`, `refactor:`)
3. **No force push to main.**
4. **PRs require at least one review** before merge.
5. **PRs must link to relevant ADRs** if they implement architecture decisions.
6. **No committing generated files** (build output, node_modules, coverage reports).

---

## Security Rules

1. **All provenance evidence must be cryptographically verified** before acceptance.
2. **Model weights must be signed.** Unsigned models must not be loaded.
3. **Reference data bundles must be signed and verified.**
4. **chrome.storage.local is unencrypted.** Sensitive cached data must be application-layer encrypted.
5. **No secrets in source code.** Use environment variables or secure storage APIs.
6. **Dependencies must be audited.** Every new dependency must be reviewed for supply chain risk.

---

## Privacy Rules

1. **Local-first is the default.** Cloud processing requires explicit user opt-in.
2. **Content analyzed must never be stored** — process and immediately discard.
3. **Full URLs must never be stored or transmitted** — domain hashing for network queries.
4. **Telemetry is opt-in only.**
5. **User personal allow/block lists never leave the device.**
6. **No browsing history collection.** Analyze only the content the user explicitly requests.

---

## Design Principles (from Phase 0 Discovery)

1. **Privacy First** — Local processing preferred. Cloud requires justification.
2. **Explainability** — Every confidence score must be explainable.
3. **User Control** — Users decide actions. Sentinel provides information.
4. **Evidence-Based** — Multiple independent signals. Never rely on one detector.
5. **Modular Design** — Every subsystem is replaceable. Plugin architecture throughout.

---

*End of AGENTS.md. All AI agents working on Sentinel must comply with these rules.*
