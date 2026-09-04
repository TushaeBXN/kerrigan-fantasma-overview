# Kerrigan-Fantasma

> A custom security research LLM and fuzzing framework — built from scratch, not a fine-tuned wrapper.

Built by **Brian Tushae Thomas** | Anthos Intelligence

📝 [Blog post — Building a Recurrent-Depth Transformer for Security Research](https://medium.com/@brian.thomas.t/building-a-recurrent-depth-transformer-for-security-research-on-a-2013-macbook-1b534101df31)

> **⚠️ Authorized use only.** Fuzzing and exploit generation require explicit per-target invocation against systems you own or have written permission to test.

---

## What It Is

Kerrigan-Fantasma is a security research framework built around a custom **Recurrent-Depth Transformer (RDT)** trained from scratch on security domain data. It combines a trained native model, an autonomous fuzzing loop, and defensive/OSINT tooling into one terminal-driven system.

**Why build a custom model instead of fine-tuning?** Fine-tuning a general LLM inherits its safety training and domain gaps. KerriganCore's architecture, tokenizer, and safety behavior are designed specifically for this domain from the start.

**What problem it solves:** Commercial AI assistants refuse legitimate security research mid-engagement — fuzzing harness generation, crash analysis, CVE triaging, exploit chain reasoning. The refusals slow down defenders without stopping attackers. Kerrigan is tuned to allow all legitimate security work while hard-blocking live weaponized payloads and attacks on unowned systems.

---

## Current Status (September 2026)

### Done

| Component | Status |
|-----------|--------|
| KerriganCore architecture (`core/model.py`) | ✅ Built — 742M params, CharTokenizer, 8-loop RDT, MoE FFN |
| Training — 5 tiers completed on Vast.ai RTX 5090 | ✅ Done |
| — Smoke (100 steps) | ✅ loss 4.62 → 2.14 |
| — SFT (50,000 steps, 234 security books) | ✅ best loss 0.2481 |
| — Instruct (10,000 steps) | ✅ best loss 0.8649 |
| — Adversarial hardening (15,000 steps) | ✅ best loss 0.2007 |
| — DPO alignment | ✅ Complete |
| — Final combined (20,000 steps) | ✅ best loss 0.7769 |
| Checkpoints on HuggingFace | ✅ 10.1 GB at `TushaeBXN/kerrigan-fantasma` |
| Evolutionary fuzzing loop | ✅ Generates harnesses, compiles, fuzzes, triages crashes |
| Swarm mode (parallel specialized agents) | ✅ Working with cross-pollination |
| Overmind safety verifier | ✅ DEFENDER_ALLOWLIST in all modes |
| Creep memory (ChromaDB) | ✅ Findings persist across sessions |
| OSINT and recon suite | ✅ Wired — subdomain, IP intel, breach intel, Sigma/YARA gen |
| Adversarial guard (9 threat classes) | ✅ Prompt injection, authority spoof, gradual erosion, etc. |

### Not Done Yet (honest gaps)

| Gap | Why it matters |
|-----|---------------|
| Fuzzer targets self-generated templates only | The evolutionary loop fuzzes programs it generates with intentional bugs. Coverage-guided fuzzing against real codebases (libpng, libxml2, SQLite) — the credibility milestone — is the next step. |
| Native model not validated on external benchmarks | Training loss numbers confirm the model learned its distribution. They don't confirm real-world security reasoning quality. No formal eval has run against CTF challenges or CVE reasoning tasks yet. |
| `kerrigan.py` routes through Ollama by default | The native KerriganCore checkpoint is available via `scripts/chat.py` and HuggingFace. The main entry point still uses Ollama as the inference backend. |
| ACT halting doesn't vary loop count in practice | The Adaptive Computation Time mechanism exists in the architecture but has no ponder cost in the loss — the model doesn't learn to allocate more loops to harder problems yet. |

---

## Architecture

**KerriganCore (Recurrent-Depth Transformer):**
- Prelude layers → recurrent block ×8 (shared weights + per-loop LoRA adapters) → Coda layers
- 742M parameters: hidden_size=1024, 8 MoE experts, 8 recurrent loops
- CharTokenizer — character-level, no vocabulary brittleness on malware strings or binary content
- MoE aux loss at 0.01× weight to prevent expert collapse
- ACT halting logic present; ponder cost not yet active

**Training data:** 234 security books (Linux kernel, UEFI firmware, CPU specs, CVE corpus, APT chains, OPSEC guides) → 20,468 SFT pairs, plus instruct, adversarial, and DPO preference pairs.

**Safety:** Overmind DEFENDER_ALLOWLIST — fuzzing, CVE analysis, harness generation, crash triage, malware analysis, threat hunting are always allowed. Live weaponization patterns (curl-pipe-to-shell, netcat reverse shells, msfvenom against real IPs) are hard-blocked in all modes.

---

## Running It

```bash
# Full code at github.com/TushaeBXN/kerrigan-fantasma
git clone https://github.com/TushaeBXN/kerrigan-fantasma.git
cd kerrigan-fantasma
pip install -r requirements.txt

# Native model inference (download from HuggingFace)
python3 scripts/chat.py --checkpoint checkpoints/kerrigan_final/final.pt

# Ollama-backed interface
ollama create kerrigan-fantasma -f config/Modelfile
python3 kerrigan.py

# Evolutionary fuzzing loop
python3 kerrigan.py --evolve "HTTP request parser" --iterations 3

# Swarm mode — 6 specialized agents, 5 rounds
python3 kerrigan.py --swarm --evolve "JPEG parser" --agents 6 --rounds 5
```

---

## What's Next

1. Wire AFL backend for coverage-guided fuzzing against real open-source targets
2. Formal eval against CTF challenges and CVE reasoning benchmarks
3. Make native KerriganCore the default inference path in `kerrigan.py`
4. Tool-calling training tier — structured nmap/searchsploit/CVE output
5. Longer sequences (512 → 2048 tokens), then 1B+ model on A100/H100

---

## Links

- **Code:** [github.com/TushaeBXN/kerrigan-fantasma](https://github.com/TushaeBXN/kerrigan-fantasma)
- **Checkpoints:** [huggingface.co/TushaeBXN/kerrigan-fantasma](https://huggingface.co/TushaeBXN/kerrigan-fantasma)
- **CyberGuard AI** (product frontend): powered by this system

---

## Researcher

**Brian Thomas (TushaeBXN)** — Cloud Security Engineer · AI Systems Builder · Charlotte, NC

[GitHub](https://github.com/TushaeBXN) · [LinkedIn](https://www.linkedin.com/in/brian-t-24748719/) · [ORCID](https://orcid.org/0009-0002-9633-3469)

MIT License. Security research use only within authorized environments.
