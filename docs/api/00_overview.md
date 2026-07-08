# Playdate C API — Overview

The Playdate C runtime exposes a single fat vtable, `PlaydateAPI`, to the
game. All entry points are function pointers on nested `playdate_*` structs.
The layout is stable-append: new fields are only added at the end of a
struct (marked with `// 1.1`, `// 2.4`, `// 3.0` comments in the SDK header),
so games built against an older SDK keep working on newer OS firmware.

## The vtable

```c
// pd_api.h (verbatim)
struct PlaydateAPI
{
    const struct playdate_sys*         system;
    const struct playdate_file*        file;
    const struct playdate_graphics*    graphics;
    const struct playdate_sprite*      sprite;
    const struct playdate_display*     display;
    const struct playdate_sound*       sound;
    const struct playdate_lua*         lua;
    const struct playdate_json*        json;
    const struct playdate_scoreboards* scoreboards;
    const struct playdate_network*     network;   // added in 3.0
};
```

`PlaydateAPI*` is passed to the game each time the OS calls `eventHandler`.
Games **must** cache the pointer at `kEventInit` (or `kEventInitLua`) and
must not assume any address stability between reboots — but the pointer
is stable for the lifetime of the process.

## Entry point

```c
#ifdef _WINDLL
__declspec(dllexport)
#endif
int eventHandler(PlaydateAPI* pd, PDSystemEvent event, uint32_t arg);
```

Symbol name **must** be exactly `eventHandler`. On device the ELF is
loaded with a fixed export table lookup; on Windows Simulator the DLL is
loaded via `LoadLibrary` and `GetProcAddress("eventHandler")`; on macOS
Simulator, `dlsym`.

Return value is ignored except for `kEventKeyPressed`/`kEventKeyReleased`
which return an accept/reject hint to the OS input queue (0 = default).

## Event codes

```c
typedef enum {
    kEventInit,           // called once, pd stable from here
    kEventInitLua,         // Lua-based games get this instead of kEventInit
    kEventLock,            // display auto-locked
    kEventUnlock,
    kEventPause,           // system menu opened
    kEventResume,          // system menu dismissed
    kEventTerminate,       // game closing, free resources
    kEventKeyPressed,      // arg = USB keycode (Simulator mirror only)
    kEventKeyReleased,
    kEventLowPower,        // battery critical
    kEventMirrorStarted,   // Playdate Mirror capture started
    kEventMirrorEnded
} PDSystemEvent;
```

The `arg` parameter is only used for key events and for future extension.

## Registering the update callback

`kEventInit` is where the game installs its per-frame callback:

```c
static int update(void* ud) {
    PlaydateAPI* pd = ud;
    // ... draw ...
    return 1; // 1 = display was updated (schedule flip); 0 = skip flip
}

int eventHandler(PlaydateAPI* pd, PDSystemEvent e, uint32_t arg) {
    if (e == kEventInit)
        pd->system->setUpdateCallback(update, pd);
    return 0;
}
```

The OS calls `update` at the configured refresh rate (default 30 Hz, set
via `display->setRefreshRate`). Returning `1` triggers `graphics->display()`
implicitly; returning `0` leaves the previous frame on-screen (saves CPU
and battery).

## Version tags in the header

`pd_api_*.h` marks each additive field with the SDK version at which it
was introduced. Cross-reference table:

| Tag       | Approx. date | Highlights |
|-----------|--------------|------------|
| baseline  | 2022-04      | initial launch |
| 1.1       | 2022-05      | elapsed-time, screen-clip |
| 1.5       | 2022-08      | removeSource |
| 1.7       | 2022-11      | rotated bitmap, `getDisplayBufferBitmap` |
| 1.8       | 2023-01      | bitmap mask |
| 1.10      | 2023-04      | LFO global, tiled stencil |
| 1.12      | 2023-06      | fonts-from-data, signal generic |
| 1.13      | 2023-07      | timezone, envelope curvature |
| 2.0       | 2023-10      | Rev B firmware, icache clear |
| 2.1       | 2023-12      | text tracking getter |
| 2.2       | 2024-01      | wavetable, LFO startPhase |
| 2.4       | 2024-03      | button callback, mp3 stream, adpcm decompress |
| 2.5       | 2024-04      | pixel poke, bitmap-table info, sequence tempo float |
| 2.6       | 2024-06      | wrapped text, envelope clear, signal-from-value |
| 2.7       | 2024-09      | round rect, mirror-data, launch args |
| 3.0       | 2025-02      | network, tilemap, video stream, `getSystemInfo` |
| 3.1       | 2025-06      | localized text, power status, `exitToLauncher`, TCP sent-pending, bit-crusher exp, font glyph API |

## Calling conventions

- On device (Cortex-M7, ARMv7-M) the AAPCS ABI is used with hardware FPU
  (VFPv5). `float` returns in `s0`, `double` in `d0`.
- On Simulator the host ABI is used (SysV/AMD64 on macOS/Linux,
  Microsoft x64 on Windows). All function-pointer signatures use plain
  C types — no calling-convention modifiers.
- All strings are UTF-8 unless annotated `PDStringEncoding` (see
  `pd_api_gfx.h`), which allows `kASCIIEncoding`, `kUTF8Encoding`,
  `k16BitLEEncoding` (UCS-2 LE, used by Japanese fonts).

## Threading

Single-threaded. `update`, event handler, all callbacks (mic, TCP, HTTP,
sequence-finished, sprite update/draw) run on the same thread. Sound
mixing runs on a DMA IRQ but never invokes user callbacks directly; it
queues events onto the main thread.
