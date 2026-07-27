# VISION.md — Sentinel Product Vision

> **Project:** Sentinel | **Owner:** Byrd Ventures LLC
> **Version:** 2.0 (revised per Phase 0 Discovery)

---

## What Sentinel Is

Sentinel is a **Trust Layer for the Internet.**

It is a privacy-first platform that helps users understand the authenticity, provenance, and trustworthiness of digital content they encounter online.

Sentinel does not decide truth. It does not censor. It provides **explainable confidence assessments** — showing users what evidence exists, how strong it is, and why the system reached its conclusion.

---

## The Problem

The internet is rapidly filling with AI-generated and AI-assisted content. Users cannot distinguish authentic photography from synthetic imagery, human writing from language model output, or genuine video from deepfakes.

Existing solutions fail because:

- **AI detectors** output binary scores ("87% AI") with no explanation — users don't know why, don't trust the result, and can't act on it
- **Fact-checkers** are reactive, slow, and don't scale to the volume of content
- **Provenance standards** (C2PA, Content Credentials) exist but are invisible to consumers — no tool surfaces them during browsing
- **Source reputation tools** rate websites, not individual content pieces

No existing tool combines provenance verification, forensic analysis, source credibility, and cross-modal consistency into a single, explainable confidence score.

---

## The Solution

Sentinel combines multiple independent evidence signals:

- **Provenance:** C2PA/Content Credentials verification, hardware attestation, cryptographic signatures
- **Forensics:** Image artifact analysis, AI generator fingerprinting, face-swap detection, audio vocoder analysis
- **Reputation:** Domain history, publisher credibility, community signals
- **Cross-modal:** Physics violations, lip-sync mismatches, image-caption semantic gaps
- **Contextual:** Metadata consistency, reverse search results, fact-check database lookups

These weak signals are fused into a **calibrated confidence assessment** — not a binary verdict. Users see the evidence, understand the reasoning, and decide what action to take.

---

## Product Philosophy

1. **Evidence over certainty.** Sentinel never claims to know truth. It communicates what the evidence suggests and how confident it is.

2. **Explainability over authority.** Every assessment shows its work. Users understand WHY a conclusion was reached.

3. **Privacy by default.** Processing happens locally whenever possible. Cloud features require explicit opt-in. Content is never stored.

4. **User sovereignty.** Sentinel provides information. The user decides. No content is hidden, blocked, or filtered without user choice.

5. **Platform independence.** Sentinel is not tied to any browser, AI model provider, or platform. Independence is the competitive moat.

---

## Target Users (Phased)

| Phase | Primary Users | Value |
|-------|---------------|-------|
| **Phase 2 (MVP)** | Journalists, Educators | Verify source images and student work with citable evidence |
| **Phase 3 (Platform)** | Content Marketers, Enterprise T&S | Batch verification, API access, team dashboards |
| **Phase 4 (Expansion)** | General Consumers | Passive browsing protection, mobile apps |

---

## Why This Exists

Sentinel exists because digital content is the foundation of modern information — and that foundation is cracking.

When a journalist cannot verify a source photograph, when a teacher cannot assess student originality, when a voter cannot distinguish authentic video from synthetic propaganda — the information ecosystem fails.

Sentinel exists to give people the tools to navigate digital content with confidence. Not by deciding for them, but by showing them the evidence and letting them decide.

---

## Long-Term Mission

Sentinel should become **infrastructure** — the standard layer for digital content trust assessment, integrated into browsers, platforms, publishers, and developer tools.

The browser extension is the first product. It is not the company.

---

*End of VISION.md. This vision was revised based on Phase 0 Discovery findings. The original framing of "Trust Layer for the Internet" remains aspirational; Phase 2 execution targets professional users with a narrower, validated value proposition.*
