# Playdate Reverse Engineering & SDK Documentation

Reverse-engineered documentation of Panic's Playdate OS, C API, file formats, and firmware.

Target SDK: **3.1.0-beta.7** (from `PlaydateSDK-3.1.0-beta.7/VERSION.txt`).
Hardware: STM32F746 (Cortex-M7, 240 MHz), 8 MB SDRAM, 16 MB NOR flash, 400×240 1-bit reflective LCD @ ~50 Hz refresh.
Style: **C11**, kernel-doc-ish comments, byte-level tables for binary formats.

## Layout

- [`api/`](api/) — C API reference, one file per `playdate_*` subsystem struct.
- [`formats/`](formats/) — file-format specifications, hex-tables, magic bytes, offsets.
- [`os/`](os/) — runtime model: event loop, framebuffer, memory map, cooperative dispatch.
- [`firmware/`](firmware/) — Simulator internals, symbolizer, MCU boot, syscall shape.
- [`samples/`](samples/) — annotated hex dumps used as evidence.

## Subsystems (from `PlaydateAPI`)

| # | Struct              | File                    | Purpose |
|---|---------------------|-------------------------|---------|
| 0 | `playdate_sys`      | [api/01_system.md]      | timers, buttons, crank, menu, i18n, power |
| 1 | `playdate_file`     | [api/03_file.md]        | POSIX-ish FS on Data/ and pdx read |
| 2 | `playdate_graphics` | [api/04_graphics.md]    | 1-bpp draw, bitmaps, fonts, framebuffer |
| 3 | `playdate_sprite`   | [api/05_sprite.md]      | dirty-rect scene, AABB collisions |
| 4 | `playdate_display`  | [api/02_display.md]     | LCD scale/mosaic/flip/offset/refresh |
| 5 | `playdate_sound`    | [api/06_sound.md]       | 44.1 kHz stereo mix, synth, MIDI, effects |
| 6 | `playdate_lua`      | [api/07_lua.md]         | Lua 5.4 bridge, class registration |
| 7 | `playdate_json`     | [api/08_json.md]        | streaming JSON encode/decode |
| 8 | `playdate_scoreboards` | [api/09_scoreboards.md] | Panic-hosted leaderboards |
| 9 | `playdate_network`  | [api/10_network.md]     | Wi-Fi status, HTTP, TCP (rev D+) |

## File formats

All Playdate binary formats share a **16-byte container header**:

```
offset  size  field
0x00    12    magic "Playdate XXX" (ASCII, space padded, no NUL)
0x0C     4    flags (little-endian u32); bit 31 (0x80000000) = payload zlib-compressed
0x10    ...   payload (type-specific)
```

| Magic          | Ext       | File | Content |
|----------------|-----------|------|---------|
| `Playdate PDZ` | `.pdz`    | [formats/pdz.md] | archive of compiled Lua + assets |
| `Playdate IMG` | `.pdi`    | [formats/pdi.md] | single 1-bit bitmap + mask |
| `Playdate IMT` | `.pdt`    | [formats/pdt.md] | bitmap table (sprite sheet) |
| `Playdate FNT` | `.pft`    | [formats/pft.md] | bitmap font (Cuberick, Asheville, etc.) |
| `Playdate AUD` | `.pda`    | [formats/pda.md] | 4-bit ADPCM audio (mono/stereo) |
| `Playdate VID` | `.pdv`    | [formats/pdv.md] | 1-bit indexed video, delta frames |
| `Playdate STR` | `.pds`    | [formats/pds.md] | localized string table |
| —              | `pdxinfo` | [formats/pdxinfo.md] | plain-text KV metadata |
| —              | `.pdx`    | [formats/pdx.md] | directory bundle (not a file) |

Not a binary file, but part of the boot chain:

- `pdxinfo` (plain text) — bundle manifest, see [formats/pdxinfo.md]
- `main.pdz` — mandatory entry archive at root of every `.pdx`

## OS notes

- [os/events.md](os/events.md) — the 12 `PDSystemEvent` codes, dispatch, and locked-state semantics.
- [os/framebuffer.md](os/framebuffer.md) — the 240×52-byte framebuffer, dirty rows, VBL.
- [os/memory.md](os/memory.md) — heap, stack, Lua VM alloc, malloc wrapper.
- [os/boot.md](os/boot.md) — power-on → Setup.pdx → Launcher.pdx → game.
- [os/device_disk.md](os/device_disk.md) — data partition layout: `Games/`, `Data/`, `Shared/`, `tmp/`, logs.
- [os/drm.md](os/drm.md) — Catalog protection: bit-30 encryption, `Playdate PDX` wrapper, `hash=` metadata.
- [os/catalog_api.md](os/catalog_api.md) — Catalog HTTPS API on `play.date/api/v2/`: endpoints, JSON shapes, purchase flow.

## Firmware

- [firmware/hardware.md](firmware/hardware.md) — SoC, peripherals, pin map, LCD driver.
- [firmware/symbolizer.md](firmware/symbolizer.md) — how `firmware_symbolizer.py` translates crash logs.
- [firmware/simulator.md](firmware/simulator.md) — how the desktop Simulator emulates the runtime.

## Legal

Documentation is derived from the publicly distributed Playdate SDK
(license: `PlaydateSDK-3.1.0-beta.7/SDK_LICENSE.md`) and observation of
publicly shipped binaries. No proprietary source code is reproduced.
