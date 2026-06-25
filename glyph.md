# Glyph — Steganographic Instruction Persistence for LLMs

**Date:** 2026-06-26
**Status:** Idea — not started

---

## Problem

LLMs forget their instructions over long conversations. CLAUDE.md, memory files, and design specs are plain markdown loaded once at session start. Over multi-turn interactions models drift from these rules — 65% of enterprise AI failures in 2025 were traced to context drift. The files are also plaintext readable by anyone, accidentally editable, not portable, and provide zero tamper detection.

---

## Core Concept

Replace CLAUDE.md, memory files, and design specs with UUID-encrypted PNGs using LSB steganography. Each instruction file becomes an image indistinguishable from real artwork or a photo. The LLM decodes it on demand via a CLI command using a UUID secret key. An Ed25519 signature and JWT-style payload verify the content came from the legitimate encoder — closing the image-as-attack-surface threat.

```
encode once:   CLAUDE.md  →  instructions.png  +  UUID (you keep this)
decode anytime: UUID + instructions.png  →  exact original markdown (stdout)
```

The image is shareable (looks like real content), portable (PNG goes anywhere — Drive, git, iMessage), and meaningless without the UUID.

---

## Technical Approach

**Encoding pipeline:**
```
markdown
→ ZSTD compress (70-80% reduction for text)
→ AES-256-GCM encrypt (UUID → PBKDF2-SHA256 → 32-byte key + random salt)
→ sign plaintext with Ed25519 private key
→ build header: magic + version + salt + IV + auth_tag + sig_len + timestamps + expiry
→ embed [header + signature + ciphertext] in LSBs of carrier PNG (1-bit per channel)
→ output PNG
```

**Decoding pipeline:**
```
PNG + UUID
→ extract LSBs → raw bytes
→ parse header (magic check, version, expiry check)
→ PBKDF2(UUID, salt) → AES key
→ AES-256-GCM decrypt (auth_tag check → tamper detection)
→ ZSTD decompress → plaintext
→ Ed25519 verify(public_key, plaintext, signature)
→ output markdown
```

**JWT-style header claims:**
- `typ`: "GLYPH"
- `alg`: "AES256GCM+ED25519"
- `kid`: UUID fingerprint (first 8 chars — identifies key without revealing it)
- `iat`: encoded timestamp
- `exp`: expiration (optional — instructions auto-invalidate after N days)
- `ver`: format version

---

## Security Model

| Secret | Purpose | Who holds it |
|--------|----------|-------------|
| UUID (128-bit) | Decryption key | You + LLM env var |
| Ed25519 private key | Signing key — never in the image | You only |
| Ed25519 public key | Verification | Stored openly in manifest |

Two attack mitigations:
1. **AES-GCM auth tag** — detects any pixel-level tampering in the carrier image
2. **Ed25519 signature** — verifies plaintext came from the legitimate encoder (attacker with UUID cannot forge a valid signature without the private key)

---

## File Structure

```
.glyph/
  instructions.png   ← encodes CLAUDE.md
  memory.png         ← encodes memory/MEMORY.md
  spec-project.png   ← encodes design specs
  manifest.yaml      ← lists files + types (NOT UUIDs)
  .keys              ← UUIDs (gitignored)
  public.pem         ← Ed25519 public key (safe to commit)
```

`.keys` format:
```
instructions=<uuid>
memory=<uuid>
spec-project=<uuid>
```

---

## CLI

```bash
glyph encode <file.md>                     # Generates UUID, outputs PNG, prints UUID
glyph decode <uuid> <file.png>             # Decodes to stdout
glyph decode <uuid> <file.png> -o out.md   # Decodes to file
glyph init [dir]                           # Encodes all .md files, creates manifest + .keys
glyph list                                 # Shows manifest
glyph rotate <uuid> <file.png>             # Re-encrypts with new UUID
glyph verify <uuid> <file.png>             # Check integrity without fully decoding
```

---

## LLM Integration

UUIDs as env vars. SessionStart hook decodes on session open:

```bash
glyph decode $GLYPH_INSTRUCTIONS_UUID .glyph/instructions.png
glyph decode $GLYPH_MEMORY_UUID .glyph/memory.png
```

LLM can also re-decode mid-conversation to refresh drifted instructions.

---

## Pros

- Re-decodable on demand — LLM refreshes instructions at any point in a long conversation
- Hidden in plain sight — carrier image looks like real content, shareable anywhere
- Tamper detection — AES-GCM + Ed25519 dual-layer
- Universal — any LLM that can run a shell command can use it
- Portable — PNG works everywhere (git, Drive, iCloud, email)
- Time-bounded — expiry claim forces re-encoding, prevents stale instructions
- Compact — 5KB CLAUDE.md → ~1KB ciphertext → tiny PNG

## Cons

- UUID loss = permanent data loss (no recovery without UUID)
- Platform recompression — WhatsApp/Twitter/Instagram JPEG-compress uploaded PNGs, destroying LSB data. Must stay lossless end-to-end.
- LSB steganalysis — detectable by dedicated tools (StegExpose etc.). Security through obscurity at the carrier level, not true cryptographic hiding.
- Bootstrap problem — still need a thin plaintext wrapper to tell the LLM to run the decoder at session start
- No key management system — .keys file is fragile, no multi-device sync built in

---

## Build Complexity

**Stack:** Python, `cryptography`, `Pillow`, `zstandard`, `click`, `PyYAML`

All dependencies are mature and well-documented. No exotic setup.

**Tricky parts:**
- Binary header alignment (struct.pack byte boundaries)
- LSB bit math (easy off-by-one errors)
- Ed25519 key serialization (PEM vs DER format)
- Capacity guard (carrier image must be large enough for payload)

**Time estimate (experienced Python dev):**
- MVP (encode/decode/sign/verify): 2-3 days
- Full CLI + manifest: +2 days
- Tests + PyPI packaging: +2 days
- Total v1: ~1 week

---

## Open Questions

- How to handle carrier image generation (user-provided vs procedural art generator)?
- Key backup/recovery strategy for UUID loss
- Multi-device UUID sync (password manager integration?)
- Should public.pem be embedded in the PNG metadata for self-contained verification?
- Package name — "glyph" likely taken on PyPI
