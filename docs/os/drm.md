# Catalog DRM

Reverse-engineered from live device disk mounted at `/Volumes/PLAYDATE`
(SDK 3.0.x era). Two protection layers.

## Layer 1 — Transport ("_protected.zip")

Catalog downloads land in `/tmp/` as `<gameID>_protected.zip`. Verified
sample:

```
/Volumes/PLAYDATE/tmp/media_W_C_..._What_time_is_it.pdx_protected.zip
```

Structure:
- Standard **PKZip** (`PK\3\4`), deflate method 8, no ZipCrypto flag.
- Payload: full `.pdx` directory tree.
- Zip itself **not** password-protected. "protected" refers to what's
  *inside* the zip.

## Layer 2 — Container-level bit-30 encryption flag

Playdate binary formats have a 16-byte container header with 4 flag
bytes at offset 0x0C:

- Bit **31** (`0x80000000`) = payload zlib-compressed. [See container.md]
- Bit **30** (`0x40000000`) = **payload encrypted**. New. Catalog-only.

The encryption flag is set on:

| File | Applies to |
|------|------------|
| `main.pdz` | top-level PDZ (Lua bytecode + core assets) |
| `pdex.bin` | ARM ELF for C games (magic changes from `PDZ` to **`PDX`**) |

**Not** encrypted (even in Catalog builds):
- Nested `.pdz` (`data.pdz`, `animation_utils.pdz` in ClayDate, Delved, CODA)
- Loose assets: `.pdi`, `.pdt`, `.pft`, `.pda`, `.pdv`
- `pdxinfo` (plain text)

Rationale: engine + logic protected; assets alone can't be resold.

## The `Playdate PDX` magic

Any `pdex.bin` on device begins with `Playdate PDX` (not `PDZ`):

```
Sideload user pdex.bin:   50 6c 61 79 64 61 74 65 20 50 44 58 | 00 00 00 00
Catalog purchased pdex:   50 6c 61 79 64 61 74 65 20 50 44 58 | 00 00 00 40
                                                              flags
                                                              bit 30 set = encrypted
```

`PDX` is the wrapper for compiled ARM ELF, independent of DRM. Setting
bit 30 on it produces an encrypted variant. Sideload/User games use the
same magic with flags=0.

## The `hash=` field in pdxinfo

Every Catalog `pdxinfo` (Purchased + Seasons) gains one extra line:

```
hash=ded0807a0c3e80032cc513cc3a0d93e47aa83c3b04ad0807b58e41fd50dc3688
```

Verified coverage on this device:

| Location                    | Has `hash=` |
|-----------------------------|-------------|
| `Games/Purchased/*.pdx`     | 68 / 68     |
| `Games/Seasons/Season-002/*`| 12 / 12     |
| `Games/User/*.pdx`          | 0 / 9       |
| Sideload `Games/*.pdx`      | 0           |

64 hex chars = 32 bytes = **SHA-256**. Working model:
- Digest of the **decrypted** `main.pdz` (or a canonical manifest
  including it). Serves as post-decrypt integrity check.
- May also be used as **per-file key derivation salt** (see cipher below).

## Cipher analysis

Sample ciphertext from `CODA.pdx/main.pdz` and `Delved.pdx/main.pdz`:

- Both files: **exact same size (25753 bytes)** → same underlying
  Pulp-runtime `main.pdz` template.
- Ciphertext bytes 0x10..0x30 completely different →
  **per-file key or per-file nonce**.
- Sizes are **not** multiples of 16 or 8 → rules out AES-CBC / DES-CBC
  with PKCS7 padding. → **stream cipher** (AES-CTR, ChaCha20) or
  raw-block variant (AES-ECB is trivially broken; not this).
- Entropy of first 4 KB of ciphertext: **7.95 bits/byte** (max is 8) →
  indistinguishable from random → real cipher, not XOR-with-key.

Working model: **AES-CTR-128 or ChaCha20** with per-file nonce.
Where's the nonce? Not in the file's plain header — the 16-byte
container header is identical across encrypted PDZs (same magic + same
flags). Options:

1. Nonce = first N bytes of encrypted body (typical of ChaCha20 poly1305 boxes).
2. Nonce = `hash` field from pdxinfo (SHA-256 truncated).
3. Nonce = deterministic from (bundleID, buildnumber, device serial).

Option 2 fits best: `hash=` is fresh per build and per bundle, and its
presence exactly correlates with encryption. Testing hypothesis needs
firmware disasm.

## Key material

No key files on the exposed data partition. Not in `/System/`, not in
`Data/com.panic.*/`. Key must live in one of:

- Firmware `.text` on internal STM32 flash (`0x24000000+`).
- OTP/BOR area of the STM32 (device-unique burn).
- Panic backend server, per-download unlock delivered via HTTPS session,
  cached in an encrypted keystore on flash outside the exposed data
  partition.

Given Catalog games run **offline** after install (no wifi required),
the key must be on-device. First install of any Catalog game after
device pairing pulls a device-bound wrapping key from Panic; individual
game keys are then decrypted from `hash=`-derived material at runtime.

That's the working model. Not verified.

## Access control on the disk itself

Panic mounts the data partition read-writable to the host during
`datadisk` mode. This gives the user visibility into their purchased
bundles (as we're using here) — but the encrypted `main.pdz`/`pdex.bin`
are meaningless without the on-flash key. The DRM is not "hide the
files"; it's "the files are cryptographically opaque".

## Simulator implications

The Simulator has **no** Catalog DRM path. `pdex.dylib`/`pdex.dll`/`pdex.so`
are never encrypted — the simulator would need the runtime key to
decrypt, which Panic doesn't ship. Catalog `.pdx` bundles cannot be run
on desktop Simulator; developers ship a separate unprotected build for
Simulator testing (typically the `pdex.dylib` next to a plain `pdex.bin`
in sideload builds).

## Anti-swap protections observed

- The `Purchased/` and `Seasons/` subdirectories are firmware-managed;
  the launcher shows them under different UI sections.
- `pdxinfo` `hash=` presence is likely the trigger for firmware to
  attempt decryption. Manually deleting `hash=` from a Catalog `pdxinfo`
  would presumably cause the runtime to try loading `main.pdz` as
  plain text and fail with `PDZ magic` error (since the payload is
  ciphertext, magic check would still pass but inflate would fail).

## Distribution channels observed

| Directory                        | Kind                            | Protection |
|----------------------------------|---------------------------------|------------|
| `Games/*.pdx`                    | Sideload via pdutil install     | none |
| `Games/User/user.<uid>.*.pdx`    | Catalog "user submissions"      | none |
| `Games/Purchased/*.pdx`          | Catalog paid store              | bit-30 encrypted |
| `Games/Seasons/Season-00N/*.pdx` | Season-1 & Season-2 games       | bit-30 encrypted |

User-submissions in `User/` are the free community-uploaded pdx served
by Catalog but **not** DRM-protected. They carry the `user.<accountID>.`
prefix (`user.289230.com.crankwork.digpitrunner.pdx`) — the numeric ID
is the Panic account that uploaded the game.

## Open questions

- Confirm AES-CTR vs ChaCha20 by comparing ciphertext under known
  plaintext (need a decrypted sample or a decompile of the runtime's
  loader).
- Locate the on-device wrapping key (firmware disasm or emulation of
  the STM32 boot sequence).
- Verify the `hash=` role (nonce? integrity check? key derivation
  salt?) by editing one byte of ciphertext and observing the failure
  mode.

## What this doc does NOT claim

- No key material extracted.
- No decryption performed.
- Cipher identification is a **plausibility argument** from
  entropy + size divisibility, not a positive identification.
