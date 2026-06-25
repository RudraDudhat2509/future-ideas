# Future Project Ideas

Ideas worth building later. Each one has a real technical core and a real user base.

---

## Idea 1: Prompt Component Attribution

When you write instructions for an AI, those instructions have multiple parts. Nobody knows which part is actually causing the AI to behave a certain way. This tool figures that out by systematically removing parts and measuring what changes. Like figuring out which ingredient is making a recipe taste weird by leaving them out one by one, except it also tests combinations.

**Who uses it:** Engineers who maintain complex AI system prompts.

**Why it is deep:** Exact computation is 2^N evaluations for N sections, so you need an approximation algorithm (similar to KernelSHAP). The evaluation function is noisy because LLMs are non-deterministic, so you need statistical averaging. Builds naturally on diffprompt.

**Technical core:** Shapley value attribution applied to prompt structure, not input tokens.

---

## Idea 2: Behavioral Fingerprinting for LLM Apps

AI companies update their models constantly without telling you. Your product works fine on Monday and starts behaving weirdly on Thursday because the underlying AI quietly changed. You only find out when users complain. This tool watches your AI application and tells you the moment its behavior meaningfully shifts, before users notice.

**Who uses it:** Any team shipping a product on top of GPT-4, Claude, or any hosted model.

**Why it is deep:** Generating a behavioral fingerprint requires embedding + clustering across a generated input distribution. Drift detection on text outputs is harder than on scalar predictions. You need to separate model drift from data drift. Natural evolution of diffprompt from point-in-time comparison to continuous monitoring.

**Technical core:** Distribution-level behavioral fingerprinting using HDBSCAN clustering over LLM output embeddings, with statistical drift detection on the cluster structure.

---

## Idea 3: Instruction Provenance for Agent Reliability

When an AI agent does something wrong, you cannot easily tell why. Was it your instructions? Something the user said? Something it read in a document? This tool tracks where every piece of information came from and how much it influenced what the agent decided to do. So if an agent makes a bad decision because it followed instructions hidden inside a document it was supposed to just review, you can see that, trace it, and catch it.

**Who uses it:** Teams building RAG agents in legal, healthcare, or finance where decision traceability matters.

**Why it is deep:** Influence attribution without direct access to attention weights requires counterfactual ablation or model self-reflection. The trust model design is non-trivial: trust levels need to compose across multi-hop retrieval chains. Reframes prompt injection as a reliability problem, not just a security problem.

**Technical core:** Lightweight middleware that wraps LangGraph agents, tracks information provenance through the context window, and scores how much each source influenced each agent action.

---

## Idea 4: Glyph — Steganographic Instruction Persistence for LLMs

CLAUDE.md, memory files, and design specs are plain markdown loaded once at session start. LLMs drift from them over long conversations — 65% of enterprise AI failures trace to context drift. The files are also plaintext-readable, accidentally editable, and not portable. Glyph replaces all of these with UUID-encrypted PNGs using LSB steganography. Each instruction file becomes an image indistinguishable from real artwork. The LLM decodes it on demand via a CLI command. An Ed25519 signature and JWT-style payload verify the content came from the legitimate encoder, closing the image-as-attack-surface threat vector.

**Who uses it:** Anyone running LLM agents with persistent instructions — Claude Code users, developers with CLAUDE.md setups, teams managing LLM memory and design specs across long conversations.

**Why it is deep:** Steganography + AES-256-GCM authenticated encryption + Ed25519 asymmetric signing in one pipeline. The JWT-like header with expiry claims makes instructions time-bounded. The real unsolved problem is seamless re-injection — making instruction refresh feel automatic rather than a manual CLI call. Full design in glyph.md.

**Technical core:** LSB steganography in lossless PNGs as a secure opaque container. UUID → PBKDF2 → AES-256-GCM for confidentiality. Ed25519 keypair for content authenticity. ZSTD compression before encryption. ~1 week to build.

---

## Idea 5: Semantic Cache False-Hit Benchmark

LLM gateways (Portkey, Helicone, Redis LangCache) use semantic caching: if a new query is "similar enough" to a cached one, return the cached answer instead of calling the model. But similarity is not the same as needing the same answer. "Refund policy for Europe?" and "Refund policy for the US?" are ~0.92 cosine-similar and need different answers. Everyone warns about "bad cache hits" and hand-waves "keep your threshold above 0.85" — nobody publishes the actual false-hit curve. This builds the canonical open benchmark: false-hit rate vs hit rate tradeoff across embedding models and thresholds, on a real QA dataset.

**Who uses it:** Anyone running an LLM gateway with semantic caching. Gateway vendors themselves for tuning defaults.

**Why it is deep:** Validation is the product. Needs an answer-equivalence judge (two answers semantically equivalent or not) layered on top of query similarity, a threshold sweep, and defensible measurement free/local. The novel artifact is the tradeoff curve nobody has published.

**Technical core:** Embedding similarity + LLM-as-judge answer-equivalence scoring, swept across thresholds, producing false-hit/hit-rate curves and a public leaderboard.

**Gauntlet (2026-06-26):** Problem is vendor-acknowledged (Portkey, TrueFoundry, Percona, Redis all blog about it; MeanCache is the academic anchor — arxiv.org/html/2403.02694). Saturation LOW — pain is named everywhere, the rigorous open benchmark is missing. Strongest novelty of the cache/MCP/judge trio; survived the gauntlet intact. Outreach target: Portkey (their own blog hand-waves the threshold; you produce the real curve). Source: portkey.ai/blog/semantic-caching-thresholds

---

## Idea 6: MCP-AttackBench — Benchmark That Grades the Scanners

MCP (Model Context Protocol) tool poisoning is a confirmed attack class: malicious instructions hidden in tool descriptions that are invisible to users but read by the agent as commands. It is real enough that the NSA published a security doc on it and OWASP has a formal entry. The naive idea — "build an MCP scanner" — is dead on arrival: mcp-scan, MCP Scanner, Ramparts, CyberMCP, and Proximity already ship. The surviving move is the meta layer: build the standardized attack benchmark and grade every existing scanner against it. "I tested mcp-scan, Ramparts, and CyberMCP against 200 crafted attacks — here are their true detection rates."

**Who uses it:** Scanner maintainers (to prove/improve coverage), gateway and AI-security companies, anyone deploying MCP servers who needs to pick a scanner.

**Why it is deep:** Requires a taxonomy of poisoning vectors (tool-description injection, output poisoning, rug pulls, cross-origin shadowing), a labeled attack corpus, and a harness that drives each scanner and scores catch rate. Meta-evaluation is unsaturated where scanners are saturated.

**Technical core:** A reproducible labeled attack corpus + a scanner-driver harness computing per-vector detection/precision/recall, published as a leaderboard.

**Gauntlet (2026-06-26):** Attack class confirmed at NSA/OWASP level (Invariant Labs original disclosure, CyberArk "Poison Everywhere", arxiv.org/html/2508.12538). Saturation of scanners HIGH (5+ exist) — so "another scanner" fails novelty exactly like a me-too project. Saturation of scanner-benchmarks LOW — that is the gap. Best outreach play of the set: you can DM both the scanner makers AND the gateway companies. Dead center of the agentic-attack-surface niche. Sources: invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks, owasp.org/www-community/attacks/MCP_Tool_Poisoning

---

## Idea 7: Free-Tier LLM-Judge Reliability + Mitigation Library

LLM-as-judge is the dominant eval paradigm (Langfuse, Arize, Braintrust all ship it), but judges are biased — position bias, verbosity bias, self-preference. The catch: every academic study measures frontier models (GPT-4, Claude). Students and startups run free judges (Groq llama, Gemini flash, local Ollama) and have zero data on how unreliable those are. Two pivots dodge the academic saturation: measure free-tier judges specifically, and ship a drop-in library that auto-corrects position bias (swap-and-average) rather than just publishing another measurement.

**Who uses it:** Anyone running cheap LLM judges in an eval pipeline — exactly the diffprompt user base.

**Why it is deep:** The measurement is statistical (swap positions, vary verbosity, measure verdict flip rate). The library is the novel product piece: it wraps any judge call and returns a debiased verdict. Direct continuity with diffprompt's LLM-judge cascades.

**Technical core:** Bias measurement harness (flip-rate metrics) + a debiasing wrapper that runs swap-and-average over free-tier judges.

**Gauntlet (2026-06-26):** Problem real but academically VERY saturated (multiple arXiv papers on self-preference, position bias, "Justice or Prejudice" has its own site; Adaline blogs "frontier models fail 50%+ bias tests"). Naive "another bias benchmark" fails novelty hardest of the three. Only survives via the free-tier focus + mitigation-library angle. Outreach: Langfuse/Arize. Weakest of the three for standing out, strongest for diffprompt continuity. Source: llm-judge-bias.github.io

---

## Notes

- Ideas 1, 2, 3 build on or complement diffprompt
- Idea 4 is standalone — different problem space, different user
- Idea 2 has the fastest path to real users
- Idea 1 has the deepest interview story
- Idea 3 is the most unique, lowest competition
- Idea 4 has product potential — CLAUDE.md ecosystem is growing fast
- Ideas 5, 6, 7 were run through a novelty/saturation gauntlet on 2026-06-26 (see Gauntlet lines)
- Idea 5 (semantic cache) survived the gauntlet intact — lowest saturation, vendor-confirmed pain, best Portkey outreach
- Idea 6 (MCP-AttackBench) is the best founder-outreach play but only in its repositioned "benchmark the scanners" form, not as another scanner
- Idea 7 (judge reliability) is the most saturated; only stands out via free-tier focus + mitigation library
- IntentMap (LLM chatbot dead-zone analytics) was considered and dropped — failure scorer was heuristic-not-ML, benchmark numbers unvalidated, and bot platforms already ship "misunderstood message" analytics. Full critique in the altagic session spec docs.
