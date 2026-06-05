# Kerrigan-Fantasma

> **A custom-built AI security research system designed for autonomous vulnerability discovery, exploit harness generation, and low-level system analysis — built from scratch, not a wrapper.**

---

## ⚠️ Responsible Disclosure Notice

This repository is a **public overview only**. The core implementation, training pipeline, model weights, fuzzing harnesses, and exploit logic are maintained in a private repository. This project is shared for research discussion and professional outreach purposes. Use of any described techniques outside of authorized, sandboxed research environments is not endorsed.

---

## What is Kerrigan-Fantasma?

Kerrigan-Fantasma is an AI security research system built around a custom **Recurrent-Depth Transformer (RDT)** architecture — designed from first principles for low-level vulnerability research. It operates as an autonomous loop: given a target surface, it reasons about the attack space, generates C exploit harnesses, executes them in isolation, analyzes crashes, and refines its hypotheses.

This is not a fine-tuned general-purpose LLM applied to security. The architecture, tokenizer, and training corpus were each designed specifically for the demands of this domain.

---

## Architecture Overview

\`\`\`
┌─────────────────────────────────────────────────────┐
│                  Kerrigan-Fantasma                  │
│              Recurrent-Depth Transformer            │
├─────────────────────────────────────────────────────┤
│  Input Layer    │  Multi-domain tokenizer           │
│                 │  C / ASM / UEFI / CVE-aware vocab │
├─────────────────┼─────────────────────────────────────┤
│  Core Model     │  Custom RDT architecture          │
│                 │  Recurrent depth blocks            │
│                 │  Multi-scale attention             │
├─────────────────┼─────────────────────────────────────┤
│  Reasoning      │  Exploit hypothesis engine        │
│  Layer          │  Attack surface decomposition      │
│                 │  Crash signature analysis          │
├─────────────────┼─────────────────────────────────────┤
│  Execution      │  Autonomous harness generation     │
│  Layer          │  7-layer code validation pipeline  │
│                 │  Docker-isolated sandbox           │
├─────────────────┼─────────────────────────────────────┤
│  Feedback       │  Crash → learning loop            │
│  Loop           │  CVE pattern reinforcement         │
└─────────────────────────────────────────────────────┘
\`\`\`

---

## Core Capabilities

### Autonomous Fuzzing Loop
The system writes its own C exploit harnesses targeting a supplied attack surface, executes them inside an isolated sandbox, captures and classifies crashes, and feeds results back into the reasoning layer. No human-in-the-loop required between iteration cycles.

### Low-Level Domain Coverage
Trained on:
- Linux kernel source (syscall interfaces, memory subsystems, scheduler)
- UEFI firmware specifications
- CPU architecture documentation (x86_64, ARM64)
- Historical CVE corpus with patch diffs
- 12 programming and assembly languages

### Defense-in-Depth Sandbox
Before any generated harness executes, it passes through a 7-layer validation pipeline:
1. Static syntax validation
2. Dangerous syscall detection
3. Resource ceiling enforcement (memory, CPU time, file descriptors)
4. Network isolation assertion
5. Filesystem scope restriction
6. Docker container isolation
7. Post-execution artifact review

### Exploit Harness Generation
Given a target (kernel module, firmware image, binary), the model decomposes the attack surface, selects candidate vulnerability classes (buffer overflows, use-after-free, race conditions, type confusion, etc.), and generates targeted C harnesses for each hypothesis.

---

## Training Approach

| Phase | Focus | Status |
|-------|-------|--------|
| Pre-training | Low-level domain corpus ingestion | ✅ Complete |
| Identity hardening | System identity & behavioral grounding | ✅ Complete |
| Instruction tuning | Security task alignment | ✅ Complete |
| Exploit reasoning | CVE-to-harness mapping | 🔄 In Progress |
| Adversarial fine-tuning | Sandbox evasion robustness | 🔲 Planned |

---

## Design Principles

**Not a wrapper.** Every component — architecture, tokenizer, training pipeline, and execution harness — was built for this domain. There is no underlying commercial model being prompted.

**Isolation first.** The system is designed under the assumption that generated code is potentially dangerous. Execution never occurs outside the defined sandbox boundary.

**Failure is signal.** Crashes are not errors to be suppressed — they are data points. The feedback loop is the core mechanism for capability improvement.

**Scope-aware.** The system tracks which vulnerability classes have been explored per target surface and explicitly avoids redundant hypothesis generation.

---

## Threat Model & Intended Use

Kerrigan-Fantasma is designed for:
- Authorized penetration testing environments
- Security research on systems you own or have explicit written permission to test
- Academic and defensive security research
- Red team tooling development within sanctioned programs

It is **not** designed for, and the author does not endorse use in:
- Unauthorized access to systems
- Offensive operations outside of legal authorized engagements
- Any activity that violates applicable computer fraud and abuse laws

---

## Stack Snapshot

| Component | Detail |
|-----------|--------|
| Model architecture | Custom Recurrent-Depth Transformer (RDT) |
| Framework | PyTorch |
| Harness language | C (primary), Python (orchestration) |
| Execution environment | Docker (multi-layer isolated) |
| CI | GitHub Actions |
| Primary domains | Linux kernel, UEFI, x86_64 / ARM64, CVE corpus |

---

## Status

Active research project. Core autonomous loop is functional. Exploit reasoning fine-tuning is ongoing. The system currently operates on isolated lab infrastructure and is not deployed externally.

---

## Researcher

**Brian Thomas (TushaeBXN)**
Cloud Security Engineer · AI Systems Builder · Charlotte, NC

[GitHub](https://github.com/TushaeBXN) · [LinkedIn](https://www.linkedin.com/in/brian-t-24748719/) · [ORCID](https://orcid.org/0009-0002-9633-3469)

> *For research collaboration, responsible disclosure discussions, or professional inquiries — reach out via LinkedIn.*

---

*Core implementation is maintained in a private repository. This overview exists for professional outreach and research discussion only.*
