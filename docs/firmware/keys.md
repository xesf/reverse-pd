# On-device Key Storage (Hypothesis)

Where does the Catalog decryption key live? Not on the data partition
(verified). Must be on-device. Enumerating candidate locations, with
plausibility rankings.

## 1. Immutable factory OTP

STM32 chips include a **one-time programmable** region:

- **STM32F7**: OTP at `0x1FFF7800`, 1024 bytes, byte-write-once.
- **STM32H7**: OTP at `0x08FFF000`, 1024 bytes similar.

Panic could burn a device-unique wrapping key during factory
provisioning. Read from firmware boot code; never exposed to userland.

**Plausibility**: High. Matches Nintendo, Sony, Apple factory-provisioning
patterns.

**Verification**: dump OTP over JTAG (not accessible via USB).

## 2. STM32 unique ID (96-bit factory serial)

Every STM32 has a **read-only, factory-programmed** UID:

- **STM32F7**: 3× u32 at `0x1FF07A10`
- **STM32H7**: 3× u32 at `0x1FF1E800`

Not a secret (any code can read it), but excellent **derivation salt**.
The Catalog wrapping key could be:

```
wrapping_key = KDF(root_secret, STM32_UID)
```

where `root_secret` is either baked into firmware `.text` or fetched
from Panic on first pairing.

**Plausibility**: Very high — this is textbook device-binding.

**Verification**: read UID via serial protocol (`pdutil <port> version`
may expose it; or read from a running game via
`playdate.system.getSerial()` if such an API exists).

## 3. Firmware `.text` constant

The decryption key is a literal 16-byte constant compiled into
firmware. All Playdates share it. Trivially extractable from any
firmware dump.

**Plausibility**: Low — Panic engineers know better.

**Verification**: entropy scan of firmware binary. High-entropy 16-byte
blob adjacent to AES routines = probable key. But this alone wouldn't
explain device-binding (all Playdates would decrypt identically, and
any leaked firmware breaks the whole store).

## 4. Data partition encrypted keystore

A file under `Data/com.panic.catalog/` (or similar) holds a per-account
wrapping key, itself encrypted by a device-bound key from OTP.

**Plausibility**: Medium. Would require a per-Panic-account "master
key" fetch during Setup.pdx.

**Verification**: audit every non-plaintext file in Data/com.panic.*/.
Nothing suspicious found in current scan (all JSON or images).

## 5. Boot partition file

A file on the Boot partition (not exposed under `datadisk`) — e.g.
`/System/panic-catalog.key`. Written once during Setup.pdx pairing,
never modified.

**Plausibility**: Medium.

**Verification**: mount Boot partition via `bootdisk`, inspect.

## 6. QSPI flash reserved sector

The 16 MB QSPI has room for a reserved sector outside the exposed
filesystem. Firmware could carve out a small area for keys.

**Plausibility**: Medium.

**Verification**: dump full QSPI, look for high-entropy blob outside
FS metadata.

## Layered scheme (most likely)

Combining the plausible options:

```
device_root_key   = OTP fuse (factory-burned)              # ~256 bits
account_key       = KDF(device_root_key, panic_account_id) # bound to Panic acct
game_wrapping_key = KDF(account_key, bundleID)             # per-game
per-file_key      = KDF(game_wrapping_key, hash)           # per-download
```

Delivery:
1. During Setup pairing, device sends `STM32_UID` + attestation to
   Panic backend.
2. Backend verifies + returns account-binding token, encrypted to
   `device_root_key`.
3. Device decrypts and stashes `account_key` in Boot partition.
4. Catalog purchase: server returns `hash` in pdxinfo of the encrypted
   bundle. Runtime derives per-file key from stored `account_key` +
   `hash`.

This scheme has the properties Playdate DRM exhibits:
- **Offline decrypt** after install (all keys derivable on-device).
- **Device-bound** (STM32_UID → different device can't derive same key).
- **Panic-account-bound** (root secret from Panic backend, needed to
  derive account_key).
- **Per-file distinguishability** (`hash` field varies).

## Software-level access

None of the C API or Lua API exposes:
- Device serial.
- OTP contents.
- Panic account ID.
- Catalog wrapping keys.

`system->getSystemInfo()` (3.0+) gives OS version + language +
pdxversion only. No serial.

The system does have `/System/serial` (from public support docs, unverified)
which some diagnostic firmware exposes, but it's the device's model
serial (P/N), not the OTP UID.

## What breaks in what scenarios

| Attack | Break |
|--------|-------|
| Get firmware `.text` root secret | Break all devices if scheme has no per-device layer |
| Extract STM32 UID via JTAG | Enable **that one device** clones only |
| Compromise Panic backend | New downloads leak; existing installs unaffected |
| Bruteforce SHA-256 hash preimage | Not directly useful — hash is public |
| Class break of AES/ChaCha20 | Cryptography-wide problem |

Panic's likely design goal: single-device leaks stay contained.
Firmware-key extraction requires physical device access; Simulator has
no keys.

## Concrete verification steps (not done here)

1. Dump Boot partition. Look for a small (<1 KB), high-entropy file.
2. `pdutil <port> serial` — see if a serial-side API exposes UID.
3. Read STM32 UID via a native pdex.bin function that calls
   `SCB->CPUID` (or the specific UID address). Requires knowing the
   memory map for the shipping chip.
4. Compare same-account, two-devices: if the same purchased pdx bundle
   ends up with different ciphertext on each device, per-device
   binding is confirmed.

Step 4 is the cleanest experiment — requires two Playdates paired to
the same Panic account.
