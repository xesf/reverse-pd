# DRM Cipher — Attack Notes

Companion to [drm.md](drm.md). Documents approaches to positively
identify the cipher and (in principle) reproduce it.

## What we know

- Bit-30 flag on container header marks encrypted payload.
- Applies to `main.pdz` (Lua bytecode + assets) and `pdex.bin` (ARM ELF).
- Ciphertext entropy: 7.95 bits/byte across 4 KB samples.
- Sizes not multiples of 8 or 16 → stream cipher.
- Same-size same-underlying-content files produce different ciphertext
  → per-file nonce or key.
- `hash=<sha256>` in pdxinfo correlates 1:1 with encryption.

## Working hypotheses

Two plausible cipher constructions:

### Hypothesis A: AES-128-CTR with hash-derived nonce

```
key   = derive(device_key, bundleID)            # KDF, e.g. HKDF
nonce = SHA256(pdxinfo.hash)[0..15]             # 128 bits from hash
IV    = nonce || counter
ct    = AES-CTR(key, IV, pt)
```

Pro: simple, streaming, matches size-agnostic property.
Con: `hash` field is the plaintext hash — recomputable — so it's a
poor nonce alone.

### Hypothesis B: ChaCha20 with account-bound key

```
key   = derive(account_wrapping_key, bundleID)  # per-Panic-account
nonce = first 12 bytes after container header ← consumed by decryptor
                                                (not part of Lua pdz)
ct    = ChaCha20(key, nonce, counter=0, pt)
```

Pro: modern default, no nonce reuse.
Con: would show a fixed 12-byte prefix on every encrypted file — need
to check.

## Known-plaintext attack (approach)

The core RE technique: get the **same game** in both encrypted and
plain form, then:

```
ct = encrypted main.pdz payload
pt = plain    main.pdz payload
keystream = ct XOR pt
```

- If `keystream` is constant across many files → single-key XOR (broken).
- If `keystream` looks random per-file → stream cipher output; can be
  fingerprinted by comparing to AES-CTR / ChaCha20 output with candidate
  keys.

### Candidate sources of matched plaintext

1. **Pulp engine**: CODA and Delved both ship the identical Pulp
   `main.pdz` (verified size 25753 bytes). If we can find one **plain**
   Pulp game (User/ or sideload) with the same runtime version, the
   plaintext is known and XOR reveals keystream.

2. **Free Catalog game later paid**: some games move between free and
   paid — same content, different encryption presence.

3. **Season 1 was originally free**: Panic distributed Season 1
   as a free "Catalog game" bundle. If an older firmware had Season 1
   plain and newer versions encrypt it, differ dumps.

4. **Developer test build**: a Catalog developer's own sideload copy
   during testing would be identical bytes to their Catalog-signed
   release. Not publicly available.

## Cipher fingerprinting

Once keystream is extracted:

```python
# AES-CTR test
for candidate_key in candidates:
    ks_pred = AES_CTR(candidate_key, iv=0, out_len=len(keystream))
    if ks_pred == keystream:  # match found
        cipher = 'AES-CTR'; key = candidate_key

# ChaCha20 test
for candidate_key in candidates:
    for candidate_nonce in candidate_nonces:
        ks_pred = ChaCha20(candidate_key, candidate_nonce)
        if ks_pred == keystream:
            cipher = 'ChaCha20'

# Byte-level tests without knowing key:
# ChaCha20 produces 64-byte blocks; check if XORing keystream against itself
# shifted by 64 shows repetition (no — pure random). Rules in/out block sizes
# by autocorrelation.
```

Even without the key, autocorrelation identifies block cipher families.

## Firmware disassembly path

Alternative — read the decryptor code out of firmware:

1. Get firmware `.pdz` bundle (via `bootdisk` mount or Panic's update
   downloads).
2. Locate the `PDZ`/`PDX` loader routine by grepping for the container
   magic string.
3. Follow the code path when bit-30 is set.
4. Identify the crypto primitive by inspecting round structure:
   - AES: 10-round loop with fixed S-box constants.
   - ChaCha20: 20-round quarter-round pattern with rotation counts
     16, 12, 8, 7.
5. Identify key derivation source: is the key derived from a fixed
   constant in `.rodata`, from an OTP fuse read (STM32 unique ID), or
   from a Data/ token file?

## Tools

- **binwalk** — general entropy + firmware inspection.
- **Ghidra** or **IDA** with ARM Cortex-M7 configuration.
- **Panda** or **QEMU-STM32** for emulated execution.
- **Community firmware dumps** (existing) that pair Boot partition
  extracts with matching runtimes.

## What we won't do here

- Actually pull firmware from the device (need JTAG or bootloader
  exploit).
- Attempt any live decryption.
- Distribute decrypted commercial games.

The point of this doc is to **describe the attack surface**, not to
break it. Panic's Catalog + Season model is legitimate commerce; DRM
here exists to prevent piracy, and reversing it beyond a documentation
level enables piracy.

## What is legitimate to reverse

- The container flag semantics (documented).
- Public API to the firmware (documented).
- File-format encoding (assets are unencrypted; documented).
- The Simulator's own crypto stack: Simulator does **not** ship
  decryption code, so there's no key material there.

## Related public work

- **jaames/pd-emu** — open-source Playdate emulator; deliberately
  refuses to decrypt Catalog content.
- **community forum threads** on the Playdate Squad Discord have
  discussed the flag byte at length.
- No public tool decrypts Catalog games.
