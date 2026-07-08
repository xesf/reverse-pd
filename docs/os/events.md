# Runtime Event Model

Every game process gets exactly one entry point:

```c
int eventHandler(PlaydateAPI* pd, PDSystemEvent event, uint32_t arg);
```

The Playdate OS pumps events onto this function in a strict order. The
runtime is **single-threaded and cooperative** — the OS does not
preempt the game.

## The 12 events

```c
typedef enum {
    kEventInit,          // 0
    kEventInitLua,        // 1
    kEventLock,           // 2
    kEventUnlock,         // 3
    kEventPause,          // 4
    kEventResume,         // 5
    kEventTerminate,      // 6
    kEventKeyPressed,     // 7  arg = HID keycode
    kEventKeyReleased,    // 8
    kEventLowPower,       // 9
    kEventMirrorStarted,  // 10
    kEventMirrorEnded     // 11
} PDSystemEvent;
```

## Startup

```
Power on
  ↓
Bootloader (STM32 ROM) → primary firmware image (from flash)
  ↓
Firmware userspace boots, mounts /System/*.pdx, loads Setup.pdx if
first boot, else Launcher.pdx
  ↓
User picks a game tile → firmware loads Games/<bundleID>.pdx
  ↓
For each game:
  Load main.pdz, mount its file tree.
  If C entry (contains pdex.bin ARM ELF):
      map into RAM, resolve eventHandler, call eventHandler(pd, kEventInit, 0)
  Else (pure Lua):
      call eventHandler(pd, kEventInitLua, 0)  [runtime provides default handler]
  ↓
Enter frame loop.
```

The runtime picks `kEventInit` vs `kEventInitLua` by looking at whether
`main.pdz` contains a compiled C module (`pdex.bin`). Lua-only games
never receive `kEventInit`; they receive `kEventInitLua`. Hybrid games
(C + Lua) receive both — `kEventInit` first, then `kEventInitLua`.

## Frame loop

```
loop:
    input->poll()                    # HID interrupt drives input queue
    dispatch input events            # button callback (2.4+)
    game->update()                   # via setUpdateCallback
    if returned non-zero:
        graphics->display()          # blit dirty rows to LCD
    audio DMA in background          # 512-sample IRQ, mix current channels
    sleep until next vsync tick
```

Vsync tick = `1/refresh_rate` seconds; if the update runs long, the
runtime logs "update overrun" to the console (see 30 FPS drop).

## Lock / Unlock

Fired when the OS idle-locks the display (dim → off).

- `kEventLock`: game receives it, must stop non-essential rendering.
  Sound is still playing. Buttons are disabled until unlock.
- `kEventUnlock`: game resumes. The screen was blanked; the runtime
  fully redraws (`markUpdatedRows(0, 240)` semantics).

`setAutoLockDisabled(1)` prevents idle-lock — used by games that expect
the crank to be the "active" input for extended still periods.

## Pause / Resume

Fired around the system menu (button held for ≥ 0.5 s).

- `kEventPause`: system menu is opening. The game's `update` still
  runs for one more tick (to let it snapshot state) but display flip is
  paused.
- `kEventResume`: menu closed. Game runs again.

Games commonly use these to pause music (`fileplayer->pause`) and to
snapshot a "resume from here" state.

## Terminate

`kEventTerminate` fires **once** when the game process is being torn
down (user exits to launcher, restart, low-power shutdown). Games must
free heap allocations here — the runtime does teardown but leaks are
logged.

Runtime guarantees a bounded time (~200 ms) before force-kill.

## Keyboard events (Simulator + Mirror)

`kEventKeyPressed` / `kEventKeyReleased` fire only when the Playdate is
being *mirrored* from a desktop keyboard (Simulator, Playdate Mirror).
`arg` is a HID keycode (USB Usage ID). Games that want to accept
keyboard input in the Simulator can handle these to map WASD→D-pad,
etc.

Return value from these two matters: `0` = pass through to default
handler, non-zero = "we consumed it, don't emit synthetic buttons".

## Low power

`kEventLowPower` fires once when battery < 5%. Games should save state
immediately; a hard shutdown may follow within seconds.

## Mirror events

`kEventMirrorStarted` fires when Playdate Mirror (or SDK dev-mirror)
attaches over USB. The game can then use
`system->sendMirrorData(cmd, buf, len)` for custom telemetry. Mirror
also grabs the framebuffer on each display flip and streams it — no
game action needed.

`kEventMirrorEnded` fires on detach.

## Cooperative discipline

The runtime **never** preempts the game. If `update` takes 40 ms, the
game misses frames but the OS survives; if `update` takes 40 seconds,
the watchdog fires (`system->error("update hang, terminating")`), fires
`kEventTerminate`, and returns to the launcher.

Watchdog timeout is ~10 s on device, disabled in Simulator.
