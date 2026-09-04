# Kerrigan-Fantasma

**A custom security research LLM and autonomous fuzzing framework — built from scratch, not a fine-tuned wrapper.**

Built by **Brian Tushae Thomas** | Anthos Intelligence

📝 [Blog post — Building a Recurrent-Depth Transformer for Security Research](https://medium.com/@brian.thomas.t/building-a-recurrent-depth-transformer-for-security-research-on-a-2013-macbook-1b534101df31)

> **⚠️ Authorized use only.** Fuzzing and exploit generation require explicit per-target invocation against systems you own or have written permission to test.

---

## What It Is

Kerrigan-Fantasma is a security research framework built around a custom **Recurrent-Depth Transformer (RDT)** trained from scratch on security domain data. It combines a trained native model, an autonomous fuzzing loop, and a full defensive/OSINT tooling suite into one terminal-driven system.

**The problem it solves:** Commercial AI assistants refuse legitimate security research mid-engagement — fuzzing harness generation, crash analysis, CVE triaging, exploit chain reasoning. The refusals slow down defenders without stopping attackers. Kerrigan is tuned to allow all legitimate security work while hard-blocking live weaponized payloads and attacks on unowned systems.

**Long-term goal:** Scale KerriganCore to a 30B+ parameter model with validated real-world security reasoning, coverage-guided fuzzing against real targets, and native tool-calling. What exists today is the foundation — the architecture, training pipeline, and framework that 30B will be built on.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      KerriganCore                           │
│              Recurrent-Depth Transformer (RDT)              │
├──────────────────┬──────────────────────────────────────────┤
│  Prelude Layers  │  Initial context encoding                │
├──────────────────┼──────────────────────────────────────────┤
│  Recurrent Block │  Shared weights × 8 loops                │
│  (×N loops)      │  Per-loop LoRA adapters                  │
│                  │  MoE FFN (8 experts)                     │
│                  │  ACT halting logic                       │
├──────────────────┼──────────────────────────────────────────┤
│  Coda Layers     │  Final output encoding                   │
├──────────────────┼──────────────────────────────────────────┤
│  Tokenizer       │  CharTokenizer — character-level         │
│                  │  No vocabulary dependencies              │
└──────────────────┴──────────────────────────────────────────┘
```

**Specs:** **948M parameters** (confirmed from checkpoint) — hidden_size=1024, 8 MoE experts, 8 recurrent loops, 4 prelude + 2 coda layers, BPETokenizer (tiktoken cl100k_base, 100,277 vocab).

**Why recurrent depth:** The same weights are reused N times before producing output. This means the model can allocate more compute to hard reasoning problems (hardware-software boundary questions, exploit chain analysis) without growing parameter count. A standard transformer of equivalent reasoning depth costs 3–4× more to run. At 30B parameters with 8 loops, effective depth exceeds a naive 30B static transformer on hard inputs.

**Why char-level tokenization:** No vocabulary brittleness on novel malware strings, hex dumps, binary-adjacent content. Handles encodings that subword tokenizers fragment unpredictably. Tradeoff: slower convergence on high-level reasoning.

**MoE FFN:** 8 experts with aux loss at 0.01× weight in training to prevent expert collapse. Each expert specializes over the course of training on different security sub-domains.

**ACT halting:** Adaptive Computation Time logic is present in the architecture. The ponder cost is not yet active in the loss function — loop count doesn't vary by problem difficulty in practice. This is a known gap; adding ponder cost is on the roadmap.

---

## Training

All 5 training tiers completed on Vast.ai RTX 5090. Checkpoints stored at `TushaeBXN/kerrigan-fantasma` on HuggingFace (10.1 GB).

| Tier | Steps | Loss | Status |
|------|-------|------|--------|
| Smoke | 100 | 4.62 → 2.14 | ✅ Complete |
| SFT | 50,000 | 0.4986 | ✅ Complete |
| Adversarial hardening | 15,000 | 0.7438 | ✅ **Best checkpoint — inference base** |
| Instruct | 10,000 | 5.3235 | ⚠️ Bad run — 16 pairs, catastrophic forgetting; redoing |
| DPO | 500 | — | ⚠️ Cut short — no real data; redoing |
| Final combined | 20,000 | 4.3690 | ⚠️ Bad run — skipped in v2 |
| Instruct v2 | 2,000 | — | 🔲 Pending — 108 quality-filtered pairs |
| DPO v2 | 1,000 | — | 🔲 Pending — 75 preference pairs |

**Training data:** Linux kernel internals, UEFI/firmware specs, CPU architecture docs (x86_64, ARM64, RISC-V), NVD CVE corpus, 18 APT chain analyses (CISA AA, Mandiant, Dragos), OPSEC/anonymity guides, Borges PDFs (60 Q&A + 2.86MB pretraining text), Hitchhiker's Guide (380 Q&A + 11.9MB pretraining text), 54-tool AI security ecosystem.

**A note on validation:** Training loss confirms the model learned its training distribution. It does not confirm real-world security reasoning quality. Formal evaluation against external benchmarks (CTF challenges, CVE reasoning tasks, exploit chain construction) has not yet been run. This is the next credibility milestone.

---

## Autonomous Fuzzing Loop

The evolutionary loop (`loop/evolution.py`) is the core operational capability:

```
Target surface description
        ↓
  Hypothesis engine
  (attack surface decomposition,
   vulnerability class selection)
        ↓
  C harness generation
        ↓
  Compile with ASan/UBSan
  (ClosedLoopCompiler)
        ↓
  Mutation fuzzer runs
  (MutationFuzzer / AggressiveMutationFuzzer)
        ↓
  Crash triage
  (CrashTriageEngine, CISA 4-variable profile)
        ↓
  Sigma/YARA rule generation
        ↓
  Harness mutation → next iteration
```

**Current state:** The loop generates and fuzzes self-authored C programs with intentional bugs. `loop/afl_backend.py` for coverage-guided fuzzing against real third-party codebases is written and wired — connecting it to real corpora (libpng, libxml2, SQLite, OpenSSL) is the next step.

**Swarm mode:** N agents run in parallel with different mutation strategies, sharing crashing inputs back into a common seed pool. A heap overflow agent finds things a format string agent misses. Cross-pollination means a find by one agent immediately benefits all others in the same round.

```bash
# Single agent
python3 kerrigan.py --evolve "HTTP request parser" --iterations 5

# Swarm — 10 agents, 5 rounds
python3 kerrigan.py --swarm --evolve "HTTP parser" --agents 10 --rounds 5

# Auto-scale on CRITICAL/HIGH crashes, live dashboard
python3 kerrigan.py --swarm --evolve "kernel driver" \
  --specialties heap stack hardware --agents 9 --auto-scale --dashboard
```

| Specialty | What it hunts | Mutation bias |
|-----------|---------------|---------------|
| `heap` | Heap overflow, use-after-free, double-free | Long strings, boundary values |
| `stack` | Stack overflow, buffer overrun | Bit flips, boundary values, format strings |
| `format` | Format string vulnerabilities | `%s %p %x %n` patterns |
| `race` | TOCTOU, concurrency bugs | Structure mutation, input doubling |
| `hardware` | Memory-mapped I/O, DMA | Boundary values, `0xff` padding |

---

## Safety System

**Overmind** (`verifier/overmind.py`) gates every output before delivery. The DEFENDER_ALLOWLIST ensures legitimate security work is never blocked in any mode.

Always allowed: fuzzing, CVE analysis, harness generation, crash triage, malware analysis, static analysis, threat hunting, OSINT on authorized targets.

Hard-blocked (all modes): curl-pipe-to-shell, netcat reverse shells, live payload generators, msfvenom against real IPs, mass credential attack automation.

**Adversarial guard** (`core/adversarial_guard.py`) runs on all inputs — 9 threat classes with a pressure tracker and session risk state (RED/ORANGE/YELLOW):
- Prompt injection, jailbreak attempts, identity override
- Authority spoofing, urgency pressure, social engineering
- Gradual erosion (slow drift across turns), insider framing, data injection

Known weak point: single-word EDUCATIONAL_MARKERS match can bypass Overmind. Fix is on the roadmap.

---

## Intelligence Modules

| Module | What it does |
|--------|-------------|
| `core/hypothesis_engine.py` | Ingest a codebase → extract danger signals → generate ranked attack hypotheses with harness hints for the evolution loop |
| `core/exploit_chain.py` | Bug primitives → 20+ chain templates → 4-phase C/Python PoC scaffold → feeds back into evolution loop for proof testing |
| `core/vuln_triage.py` | 3-stage triage: extract → score → falsify → rank |
| `core/ip_intel.py` | IP geo, ASN topology, DNS contamination, RDAP Whois — no keys required |
| `core/world_intel.py` | 113 tools: Feodo/CISA KEV/SANS live feeds + NLP + classification |
| `core/osint.py` | 27-source live intel: conflicts, sanctions, radiation, markets |
| `core/recon.py` | Dorks, JS secrets scanner, DMARC/SPF/DKIM, crt.sh, CORS, S3, subdomain takeover, attack chains |
| `core/self_protect.py` | Live manifest of all 19 modules with known attack surfaces; probe campaign detection; SHA-256 file integrity; LD_PRELOAD/PYTHONPATH detection; debugger attachment detection; AI-vs-AI detection (Trojan triggers, special token injection, agentic framework attacks, AI-generated malware) |
| `agents/osint_agent.py` | 8 modules: subdomain enum, SPF/DMARC, S3 exposure, IdP recon, GitHub dorking, breach intel, web recon, full org surface scan |
| `loop/sigma_gen.py` | Auto-generates Sigma rules, YARA, Splunk/Elastic hunt queries from every finding |
| `skills/registry.py` | 15 skills — 7 security + 8 OSINT skills with keyword routing |

---

## Current Inference

Two paths:

**Native KerriganCore** (`scripts/chat.py`):
```bash
python3 scripts/chat.py --checkpoint checkpoints/kerrigan_final/final.pt
```
Includes auto web search (duckduckgo, no API key), 4-turn conversation memory, `!search`/`!nosearch`/`!clear` commands.

**Ollama-backed** (`kerrigan.py`) — current default:
```bash
ollama create kerrigan-fantasma -f config/Modelfile
python3 kerrigan.py
```
Routes through whichever Ollama model is installed. Making native KerriganCore the default path is a near-term milestone once inference quality is verified.

---

## What's Next

**Immediate (credibility blockers):**
1. Wire `loop/afl_backend.py` for real-corpus fuzzing (libpng, libxml2, SQLite)
2. Run `scripts/eval.py` against external security benchmarks — CTF challenges, CVE reasoning
3. Make native KerriganCore the default inference path
4. Fix Overmind EDUCATIONAL_MARKERS bypass

**Medium-term:**
- Add ponder cost to ACT halting
- Tool-calling training tier (structured nmap/searchsploit/CVE output)
- Sequence length: 512 → 2048 tokens
- Run distillation end-to-end (`scripts/distill.py` is ready)
- Human feedback loop on real security reasoning quality

**Scale milestones:**

| Milestone | What it means |
|-----------|--------------|
| 1B parameters | Larger hidden_size + more layers on same architecture; A100/H100 run |
| Validated benchmarks | Formal eval on CTFBench, CVEBench, exploit reasoning tasks |
| Real-target fuzzing | AFL backend against actual library corpora |
| Tool-calling native | Structured JSON tool calls trained into the model |
| **30B parameters** | **The actual goal.** Same architecture, the data and compute to match. At 30B with proper tool-calling and real-target fuzzing, this becomes a genuine autonomous security research system. |

---

## Responsible Use

**Intended for:**
- Authorized penetration testing environments
- Security research on systems you own or have explicit written permission to test
- Academic and defensive security research
- Red team tooling development within sanctioned programs

**Not intended for:** unauthorized system access, offensive operations outside legal authorized engagements, or any activity violating computer fraud and abuse laws.

---

## Links

- **Code:** [github.com/TushaeBXN/kerrigan-fantasma](https://github.com/TushaeBXN/kerrigan-fantasma)
- **Checkpoints:** [huggingface.co/TushaeBXN/kerrigan-fantasma](https://huggingface.co/TushaeBXN/kerrigan-fantasma)
- **Blog post:** [Building a Recurrent-Depth Transformer for Security Research](https://medium.com/@brian.thomas.t/building-a-recurrent-depth-transformer-for-security-research-on-a-2013-macbook-1b534101df31)
- **CyberGuard AI:** Product frontend powered by this system

---

**Brian Thomas (TushaeBXN)** — Cloud Security Engineer · AI Systems Builder · Charlotte, NC

[GitHub](https://github.com/TushaeBXN) · [LinkedIn](https://www.linkedin.com/in/brian-t-24748719/) · [ORCID](https://orcid.org/0009-0002-9633-3469)

MIT License.
