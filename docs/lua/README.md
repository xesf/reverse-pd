# Playdate Lua Runtime

Playdate ships **Lua 5.4** (PUC-Rio fork, `lua_Integer = int32`,
`lua_Number = float32`). All game bytecode compiled by `pdc` and packed
into `main.pdz`. Runtime binds C API via `playdate.*` namespace.

- [corelibs_reference.md](corelibs_reference.md) — every `CoreLibs/*.lua`
  module (24 modules, ~5500 lines total). Function-by-function.
- [idioms.md](idioms.md) — `class()`, `import`, metatables, module scope,
  userdata retention.
- [playdate_globals.md](playdate_globals.md) — the top-level `playdate`,
  `playdate.graphics`, `playdate.sound`, etc. tree — Lua's mirror of the
  C `PlaydateAPI` vtable.

## Two-tier API

- **`playdate.*`** — native bindings, always present. One-to-one with
  the C `PlaydateAPI` struct entries. Provided by the runtime; no
  import needed.
- **`CoreLibs/*`** — pure-Lua helpers. Must be `import`ed by the game.
  `pdc` scans for `import` statements and packs only referenced modules
  into `main.pdz`.

Example minimum game (pure Lua):

```lua
import "CoreLibs/graphics"
import "CoreLibs/sprites"
import "CoreLibs/timer"

local gfx = playdate.graphics

function playdate.update()
    gfx.clear()
    playdate.timer.updateTimers()
    gfx.sprite.update()
end
```

The `playdate.update` global function is the Lua equivalent of the C
`update` callback registered via `system->setUpdateCallback`.

## Event handlers (Lua-side)

The Lua runtime installs default `eventHandler` and dispatches events to
free-standing Lua callbacks:

| C event | Lua callback |
|---------|--------------|
| `kEventInitLua` | (called before `main.lua` runs) |
| `kEventLock`    | `playdate.deviceWillLock()` |
| `kEventUnlock`  | `playdate.deviceDidUnlock()` |
| `kEventPause`   | `playdate.gameWillPause()` |
| `kEventResume`  | `playdate.gameWillResume()` |
| `kEventTerminate` | `playdate.gameWillTerminate()` |
| `kEventLowPower` | `playdate.deviceWillSleep()` |
| button pressed  | `playdate.AButtonDown()`, `playdate.leftButtonDown()`, etc. |
| button held/released | `playdate.AButtonHeld()`, `playdate.AButtonUp()`, etc. |
| crank changed   | `playdate.cranked(change, acceleratedChange)` |
| crank docked/undocked | `playdate.crankDocked()`, `playdate.crankUndocked()` |

Each is a free-standing global — leave undefined to ignore.
