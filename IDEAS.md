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

## Notes

- Ideas 1, 2, 3 build on or complement diffprompt
- Idea 4 is standalone — different problem space, different user
- Idea 2 has the fastest path to real users
- Idea 1 has the deepest interview story
- Idea 3 is the most unique, lowest competition
- Idea 4 has product potential — CLAUDE.md ecosystem is growing fast
