# CONTRIBUTING.md — Sentinel Contribution Guide

> **Project:** Sentinel | **Owner:** Byrd Ventures LLC

---

## Welcome

Sentinel is an open-core project. The browser extension core, evidence pipeline framework, C2PA verification library, signal definition format, and SDK are open source (Apache 2.0). Proprietary components (detection models, trust graph, cloud infrastructure) are licensed separately.

We welcome contributions to the open-source components. This guide defines how.

---

## Development Workflow

1. **Fork** the repository
2. **Create a branch:** `feature/description`, `fix/description`, `spike/description`, or `docs/description`
3. **Implement** your change
4. **Write tests** for your change
5. **Run tests** to verify nothing is broken
6. **Submit a Pull Request** to the `main` branch
7. **Wait for review** — all PRs require at least one approval

---

## Branch Strategy

- **`main`** — Production-ready code. Protected branch. No force push.
- **`feature/*`** — New features
- **`fix/*`** — Bug fixes
- **`spike/*`** — Technical validation (disposable, not merged to main)
- **`docs/*`** — Documentation changes

---

## Commit Conventions

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add C2PA verification to evidence pipeline
fix: resolve SW termination during long analysis chains
docs: update architecture diagram for scheduler
chore: update dependencies
test: add fusion engine deterministic output tests
refactor: extract CapabilityProvider from scheduler
```

---

## Testing Expectations

1. **Every analyzer plugin must have unit tests** verifying:
   - Output format (conforms to Evidence interface)
   - Confidence calibration (reported confidence reflects actual accuracy)
   - Error handling (produces structured EvidenceError on failure)
   - Edge cases (empty input, malformed input, oversized input)

2. **Every pipeline stage must have integration tests** verifying input/output contracts.

3. **The Fusion Engine must have deterministic tests** — same evidence graph produces same confidence assessment.

4. **Cross-context messaging must have end-to-end tests** across all browser contexts.

5. **Performance budgets are verified in CI.** Any regression blocks merge.

---

## Documentation Expectations

1. New features require documentation updates.
2. Public interfaces require JSDoc comments.
3. ADRs required for architectural changes.
4. Undocumented code is incomplete.

---

## PR Requirements

- [ ] Branch named according to convention
- [ ] Commit messages follow Conventional Commits
- [ ] Tests pass locally
- [ ] New tests added for new functionality
- [ ] Documentation updated if applicable
- [ ] ADR included if architectural change
- [ ] No generated files committed (build output, node_modules)
- [ ] No secrets, API keys, or credentials
- [ ] PR links to relevant ADRs if implementing architecture decisions

---

## Code Standards

- **TypeScript** strict mode. No `any` without documented justification.
- **Immutability by default.** Evidence objects are frozen after construction.
- **Explicit error handling.** No swallowed exceptions.
- **No `eval()`, `new Function()`, or dynamic code execution.**
- **Minimal dependencies.** Every dependency must be justified.

Full standards: `AGENTS.md`

---

## Contributor License Agreement (CLA)

By submitting a contribution, you agree that your contribution is licensed under the Apache 2.0 license and that Byrd Ventures LLC may re-license your contribution under proprietary terms if needed for the open-core business model.

A formal CLA will be introduced before significant community contributions are accepted. For now, the Apache 2.0 license applies to all open-source components.

---

## Getting Help

- Read `PROMPT.md` for project context
- Read `ARCHITECTURE.md` for system overview
- Read domain-specific architecture documents in `docs/architecture/`
- Open an issue for questions

---

*End of CONTRIBUTING.md*
