# `pdex.bin` Loader

The mechanism by which the Playdate firmware maps a game's compiled C
ELF into memory and jumps into it.

## Author-side output from `pdc`

`pdc` produces per-platform native binaries:

- `pdex.bin` — ARM Cortex-M7 ELF (position-independent) for on-device.
- `pdex.dylib` — Mach-O for macOS Simulator.
- `pdex.dll` — PE/COFF for Windows Simulator.
- `pdex.so` — ELF for Linux Simulator.

All export the same symbol: `int eventHandler(PlaydateAPI*, PDSystemEvent, uint32_t)`.

## On-device wrapper

The on-device `pdex.bin` is not a raw ELF — it's an ELF wrapped in a
Playdate container:

```
0x00  12  "Playdate PDX"          magic
0x0C   4  flags (u32 LE)          0x00000000 sideload; 0x40000000 Catalog-encrypted
0x10  --  ELF payload (or ciphertext)
```

See [../formats/container.md](../formats/container.md) for flag semantics.

Verified: `Games/User/user.289230.com.crankwork.digpitrunner.pdx/pdex.bin`
has magic + flags 0x00, followed by what looks like fixed-up ARM code
(not raw ELF — likely a pre-relocated flat binary — see below).

## ELF vs flat binary

Two possible interpretations of the payload:

### A. Raw ELF (`EM_ARM`, hard-float ABI)

Firmware would parse the ELF at load, respect segment sizes, apply
`R_ARM_RELATIVE` relocations, resolve `PlaydateAPI*` symbol references,
and jump to `e_entry`.

### B. Pre-linked flat binary

`pdc` links the ELF at a **fixed virtual address** (like `0xC0100000`
in SDRAM) with all relocations resolved at build time. Firmware just
memcpy's the payload to that address and jumps.

**Evidence for B**: the first 4 bytes after the container header on
digpitrunner are `d1 d1 71 a7` — not `\x7fELF`. This is either:
- Encrypted (but flags say plain 0x0), or
- A flat binary (probable), or
- A stripped ELF with unusual header.

At `0x20` onwards we see `c8 73 00 00 0c 38 01 00` which could be a
segment table pointer or the start of executable code.

Working model: **flat binary + small custom header**. `pdc` link script
targets `0xC0100000`, static relocations resolved, PLT/GOT baked in
against a fixed offset within the runtime's `PlaydateAPI` mapping.

## Load sequence

1. Firmware mounts pdx bundle.
2. Reads `pdex.bin`, validates magic.
3. If encrypted (bit 30) → decrypts to a scratch buffer.
4. Copies payload to fixed load address in SDRAM
   (`0xC0100000` or similar).
5. Flushes D-cache (write-back) + invalidates I-cache for the range.
6. Resolves `PlaydateAPI*` — either by patching a known offset in the
   payload with the runtime's vtable pointer, or by loading it into
   a specific register before the call.
7. Calls `eventHandler(pd, kEventInit, 0)`.
8. On return, waits for next event.

## `PlaydateAPI*` resolution

The C runtime hands the game a stable pointer. Two ways this could
work:

### Approach 1: patch on load

`pdc` reserves a specific memory location in `pdex.bin` (e.g. the first
u32 of `.data`) for the API pointer. Firmware writes its own vtable
address there before jumping. Game code reads through this fixed
location.

### Approach 2: register pass

Firmware puts the API pointer in `r0` before calling — matches AAPCS
convention. Game's `eventHandler` receives it as first argument.

Approach 2 matches the C ABI cleanly and needs no linker cooperation.
Approach 1 is used by some other embedded games consoles (e.g. GBA
sysroot pointers).

**Working model: Approach 2.** `pdc` produces a normal
`int eventHandler(PlaydateAPI*, ...)` signature; firmware calls it as
if it were a normal C function. Any subsequent calls back into the
runtime go through the `pd` pointer the game cached.

## Symbol resolution

The game's `pdex.bin` links against **nothing** — no dynamic linker,
no shared libraries. Everything the game calls goes through the
`PlaydateAPI*` vtable.

Standard-library calls (`printf`, `malloc`, `memcpy`, etc.) are
either:
- Provided by the runtime through a compatibility shim (weak
  symbols),
- Statically linked from a bundled `libc` (newlib/nano), or
- Rejected at link time — game must use `pd->system->realloc`
  instead.

The SDK's `Makefile` for C games links with `-nostdlib` and provides its
own minimal `libplaydate.a` for essential runtime helpers.

## Instruction cache

After copying code into RAM, firmware must invalidate I-cache before
executing (or it may fetch stale bytes). This is why 2.0 added
`clearICache()` — the game itself can force this when it emits code
(rare) or after loading dynamic assets that contain code.

## Termination

On `kEventTerminate`:

1. Runtime calls game's `eventHandler(pd, kEventTerminate, 0)`.
2. Game frees resources, returns.
3. Runtime tears down: unmaps the code region, invalidates I-cache,
   clears the API vtable in registers.
4. Firmware returns to the Launcher process.

If the game hangs (>10s watchdog), firmware force-kills without
`kEventTerminate` — resources may leak but the launcher resumes.

## Simulator loader

macOS Simulator uses `dlopen("pdex.dylib") + dlsym("eventHandler")`.
Windows uses `LoadLibrary + GetProcAddress`. Linux uses `dlopen`
(same as macOS).

Simulator has **no** Catalog-encrypted `pdex.dylib` support — it can't
decrypt bit-30-encrypted binaries because it lacks the on-device keys.
Developers ship a separate unencrypted `pdex.dylib` for Simulator
testing.

## The `_WINDLL` conditional

```c
#ifdef _WINDLL
__declspec(dllexport)
#endif
int eventHandler(PlaydateAPI* playdate, PDSystemEvent event, uint32_t arg);
```

Windows requires the explicit `dllexport` to make `GetProcAddress`
find the symbol. Mach-O and ELF export by default.

## Relocation gotchas

Because `pdc` links at a fixed address, running the ELF on hardware
with a different memory map (e.g. Rev A vs Rev B where SDRAM might
be at a different address) would fail. Firmware likely picks one of:

1. Same SDRAM address on both revs (safest, wastes address space on Rev B).
2. `pdc` emits a small relocation table for firmware to fix up.
3. `pdc` emits truly position-independent code (PIC/PIE), no
   relocations needed.

Verifiable by disassembling one `pdex.bin` and looking for
`ldr Rn, [pc, #imm]` vs absolute addressing patterns.

## Format not standardized

Panic hasn't published the `pdex.bin` container format. The header
bytes after the magic are a Panic-private structure — likely:

```c
struct pdex_header {
    char     magic[12];         // "Playdate PDX"
    uint32_t flags;             // container flags
    uint32_t entry_offset;      // offset into payload of eventHandler
    uint32_t load_address;      // where to copy the payload
    uint32_t payload_size;      // bytes
    uint32_t bss_size;          // uninitialized data to reserve after payload
    uint32_t reserved[3];
    uint8_t  payload[payload_size];
};
```

**Status**: inferred layout, not verified against the actual bytes of
`pdex.bin`. Would need to disassemble the loader in firmware to
confirm.
