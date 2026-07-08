# Playdate Simulator

The Simulator ships in the SDK at `bin/Playdate Simulator.app`
(universal Mach-O). It emulates the runtime on desktop macOS, Windows,
and Linux.

## Architecture

```
+----------------------------+
|  wxWidgets UI (PlaydateKitWx) |   Cross-platform GUI shell
+----------------------------+
|  Simulator core            |   C++, in-process
|    - PDLCDBox              |   virtual LCD widget
|    - Sound engine (miniaudio-ish) |
|    - File I/O (Disk/ overlay)     |
|    - Lua 5.4 VM            |   from PUC-Rio, same version as device
+----------------------------+
|  Game plugin (SHARED)      |   the game's pdex.dylib / .dll / .so
+----------------------------+
```

Strings extracted from the binary confirm:

- Built with `wxWidgets`, class `PWX::PDLCDBox` renders the LCD.
- Lua 5.4.3 (PUC-Rio) as of SDK 3.1.
- Analytics via TelemetryDeck (opt-out in Settings).
- OpenSSL for HTTPS emulation.
- libpng 1.6.37 for `.pdi` ↔ `.png` conversion in `pdc`.

Buildbot path leaked in strings:
`/Users/buildbot/Developer/ci-builds/8xz7CyNq/0/playdate/PlayDate/PlaydateKitWx/`
— Panic uses a private wx-based framework called `PlaydateKitWx`.

## Game loading

The Simulator loads a game's `pdex.dylib` (macOS), `pdex.dll`
(Windows), or `pdex.so` (Linux) using OS-native dlopen. It resolves
`eventHandler` and calls with the same `PlaydateAPI*` layout as the
device, populated with function pointers into the Simulator's own
implementations.

**Pure Lua games** don't ship a shared library; the Simulator's own Lua
VM runs `main.pdz`'s bytecode directly.

## LCD emulation

`PDLCDBox` is a wx `wxPanel` that:

- Holds a 400×240 1-bit array shadowing the "device" framebuffer.
- Repaints on `wxPaintEvent` — scaled up (typically 2× or 4×) to a
  visible pixel size.
- Interprets `graphics->display()` as an immediate blit.
- Optionally simulates the Sharp panel's LCD "off pixels are gray, not
  transparent" look via a slight background tint.

## Sound emulation

Runs a real host-side audio thread (usually via CoreAudio / WASAPI /
ALSA), pushes 44.1 kHz stereo. The synth, effects, and mixer are the
same code as the device — the port is just to a different DMA sink.

## File I/O

- **Data/** maps to `Disk/Data/<bundleID>/` (macOS default:
  `~/Developer/PlaydateSDK/Disk/Data/`).
- **Bundle** maps to the on-disk `.pdx` directory the user launched.

The Simulator does **not** decompress `.pdz` files at load time; it
walks entries lazily via the same reader used on device. Same
alignment, same zlib streams.

## Simulator-only APIs

Two graphics functions exist only in the Simulator:

- `graphics->getDebugBitmap()` — a diagnostic overlay bitmap (usually
  shows sprite bounds).
- The Simulator injects `kEventKeyPressed` / `kEventKeyReleased` from
  the host keyboard.

On device these entries in the vtable are `NULL` or no-op.

## Mirror

The Simulator can act as a Playdate Mirror sink — it opens a USB CDC
serial connection to a real Playdate, receives framebuffer snapshots
each flip, and displays them alongside the emulated ones for
side-by-side comparison. Toggle in Simulator menu → Device → Mirror
Connected Device.

## Telemetry

Simulator sends anonymized crash/version data to Panic via
TelemetryDeck. Endpoints hardcoded in the binary. Disable in
Preferences.
