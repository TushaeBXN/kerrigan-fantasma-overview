# Kerrigan-Fantasma

> **A custom-built AI security research system designed for autonomous vulnerability discovery, exploit harness generation, and low-level system analysis — built from scratch, not a wrapper.**

📝 **Blog post:** [Building a Recurrent-Depth Transformer for Security Research on a 2013 MacBook](https://medium.com/@brian.thomas.t/building-a-recurrent-depth-transformer-for-security-research-on-a-2013-macbook-1b534101df31) — full writeup on Medium.

---

## ⚠️ Responsible Disclosure Notice

This repository is a **public overview only**. The core implementation, training pipeline, model weights, fuzzing harnesses, and exploit logic are maintained in a private repository. This project is shared for research discussion and professional outreach purposes. Use of any described techniques outside of authorized, sandboxed research environments is not endorsed.

---

## Proof of Concept

The autonomous fuzzing loop is functional today. Real results from 4 sessions on an HTTP request parser target:

| Metric | Result |
|--------|--------|
| Sessions run | 4 |
| Total inputs generated | 380 |
| Unique crash signatures | 2 |
| Crash types | SIGILL (CWE-121 stack overflow), SIGABRT (CWE-122 heap overflow) |
| CVE parallels | Heartbleed (OOB read pattern), Kaminsky (boundary condition) |

These were found on a 2013 MacBook Pro with no GPU — the C mutation engine runs at 2.95M mutations/sec via a compiled native engine, and a compilation cache drops repeated harness builds to 0ms.

---

## What is Kerrigan-Fantasma?

Kerrigan-Fantasma is an AI security research system built around a custom **Recurrent-Depth Transformer (RDT)** architecture — designed from first principles for low-level vulnerability research. It operates as an autonomous loop: given a target surface, it reasons about the attack space, generates C exploit harnesses, executes them in isolation, analyzes crashes, and refines its hypotheses.

This is not a fine-tuned general-purpose LLM applied to security. The architecture, tokenizer, and training corpus were each designed specifically for the demands of this domain.

---

## Architecture Overview

```
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
```

---

## Core Capabilities

### Autonomous Fuzzing Loop
The system writes its own C exploit harnesses targeting a supplied attack surface, executes them inside an isolated sandbox, captures and classifies crashes, and feeds results back into the reasoning layer. No human-in-the-loop required between iteration cycles. The C mutation engine generates 2.95M mutations/sec; AFL++ can be wired in as a coverage-guided backend for real binaries.

### Low-Level Domain Coverage
Trained on:
- Linux kernel source (syscall interfaces, memory subsystems, scheduler)
- UEFI/EDK2 firmware source (DXE core, SMM core, SecurityPkg)
- CPU architecture documentation (x86_64, ARM64, RISC-V ISA)
- NVD CVE corpus (500+ entries across memory corruption, hardware, firmware)
- Project Zero research and arxiv cs.CR papers
- 12 programming and assembly languages (C, C++, Rust, Assembly, Python, Go, JS, Java, Verilog, VHDL, SystemVerilog, Bash)

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

### OSINT Suite
9 integrated modules: email intel, domain intel, social media, phone, website fingerprinting, EXIF extraction, image search, dark web lookups, and full-target aggregation. All output is gated through the Overmind safety verifier and logged to persistent memory.

### Persistent Memory
ChromaDB vector store persists findings across sessions. Every crash, OSINT result, and reasoning chain is stored and retrieved on future runs — the system gets more effective the more it is used.

---

## Training Status

| Phase | Focus | Status |
|-------|-------|--------|
| Smoke (100 steps) | Architecture verification | ✅ Complete — loss 4.62 → 2.14 |
| Proof (1,000 steps) | Security corpus, loss validation | ✅ Complete |
| SFT (50,000 steps) | Combined code + hardware + security corpus | 🔲 Pending GPU (RunPod) |
| Hardware (20,000 steps) | Kernel + firmware + CPU specs fine-tune | 🔲 Pending GPU |
| Instruct (10,000 steps) | Q&A pairs: Spectre, ROP, UEFI, TrustZone | 🔲 Pending GPU |

Current inference backbone: **Ollama + deepseek-coder:6.7b**. The fuzzer, compiler cache, mutation engine, and safety systems are fully functional with this backbone today. The native KerriganCore model runs and needs GPU time to replace it.

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
| Framework | PyTorch 2.2.2 |
| Harness language | C (primary), Python (orchestration) |
| Mutation engine | Native C via ctypes — 2.95M mutations/sec |
| Compile cache | SHA-256 content hash, 0ms on hit |
| Execution environment | Docker (multi-layer isolated) |
| Memory backend | ChromaDB + MySQL (KerriganDB) |
| CI | GitHub Actions |
| Primary domains | Linux kernel, UEFI, x86_64 / ARM64 / RISC-V, CVE corpus |

---

## Status

Active research project. The autonomous fuzzing loop, mutation engine, compilation cache, OSINT suite, and safety verifier are all functional. Native KerriganCore architecture is built and smoke-trained — full training is pending GPU time on RunPod. The system currently operates on isolated lab infrastructure and is not deployed externally.

---

## Researcher

**Brian Thomas (TushaeBXN)**  
Cloud Security Engineer · AI Systems Builder · Charlotte, NC

[GitHub](https://github.com/TushaeBXN) · [LinkedIn](https://www.linkedin.com/in/brian-t-24748719/) · [ORCID](https://orcid.org/0009-0002-9633-3469)

> *For research collaboration, responsible disclosure discussions, or professional inquiries — reach out via LinkedIn.*

---

*Core implementation is maintained in a private repository. This overview exists for professional outreach and research discussion only.*
