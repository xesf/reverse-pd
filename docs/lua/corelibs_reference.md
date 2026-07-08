# CoreLibs Reference

Every module under `PlaydateSDK-3.1.0-beta.7/CoreLibs/`. Import with
`import "CoreLibs/<name>"` (extension optional).

## `__stub.lua` — native stubs

Not user-facing. Declares native-implemented functions so the Lua LSP /
type-checker sees them. Includes:

```lua
playdate.table.indexOfElement(t, element)   -- linear search
playdate.table.getsize(t)                   -- # of elements (not just array part)
playdate.table.create(arrCount, hashCount)  -- pre-allocated
playdate.table.shallowcopy(src, dst?)       -- dst optional
playdate.table.deepcopy(src)
playdate.apiVersion()                        -- (major, minor)
playdate.metadata()                          -- pdxinfo table
playdate.wait(ms)                            -- blocking; do NOT use in update
```

## `3d.lua` — vector3d

`vector3d.new(x, y, z)` with operators:

```lua
v1 + v2     -- __add
v1 - v2     -- __sub
v * s       -- __mul (scalar)
v / s       -- __div
v1:dot(v2)
v1:cross(v2)
v:length()
```

Not a full 3D engine — just vector math for games doing pseudo-3D.

## `accelerometer.lua`

```lua
playdate.getDeviceOrientation()   -- "flat" / "portrait" / "landscape" / "landscapeLeft" / "landscapeRight"
playdate.getPitchAndRoll()        -- returns pitch, roll in radians
```

Wraps `system->getAccelerometer` in higher-level orientation categories.
Auto-enables the accel peripheral on first call.

## `animation.lua` — `playdate.graphics.animation.loop`

Frame-loop animation over an `LCDBitmapTable`.

```lua
local loop = playdate.graphics.animation.loop.new(delayMs, imageTable, shouldLoop)
loop:setImageTable(it)
loop:isValid()             -- false when not looping and finished
loop:image()               -- current frame (LCDBitmap)
loop:draw(x, y, flipped)   -- convenience blit
```

Frame index advances with wallclock, not update-count, so pause/resume
snaps forward.

## `animator.lua` — value tweening

`playdate.graphics.animator` interpolates between two values over time,
with optional easing.

```lua
local a = playdate.graphics.animator.new(durationMs, startVal, endVal, easingFn?)
-- 4-arg variant with easing from CoreLibs/easing
a:reset(newDuration?)
a:valueAtTime(t)          -- 0..1 of duration
a:currentValue()
a:progress()              -- 0..1
a:ended()
a:reverses()              -- ping-pong
```

Multiple ctor overloads exist — line/arc paths accept extra params.

## `crank.lua`

```lua
playdate.getCrankTicks(ticksPerRotation)
```
Discretized crank input. `ticksPerRotation=12` gives clock-style clicks.

## `easing.lua` — `playdate.easingFunctions`

Table of 30+ easing curves compatible with `animator` and `timer`:

```
linear, inQuad, outQuad, inOutQuad, outInQuad,
inCubic, outCubic, inOutCubic, outInCubic,
inQuart, outQuart, inOutQuart, outInQuart,
inQuint, outQuint, inOutQuint, outInQuint,
inSine, outSine, inOutSine, outInSine,
inExpo, outExpo, inOutExpo, outInExpo,
inCirc, outCirc, inOutCirc, outInCirc,
inElastic, outElastic, inOutElastic, outInElastic,
inBack, outBack, inOutBack, outInBack,
inBounce, outBounce, inOutBounce, outInBounce
```

Each takes `(t, b, c, d)` where `t` = current time, `b` = begin value,
`c` = change amount, `d` = total duration. Robert Penner formulas.

## `frameTimer.lua` — frame-count based timer

Like `playdate.timer` but tied to update-count, not wallclock.

```lua
playdate.frameTimer.new(durationFrames, startVal?, endVal?, easingFn?)
playdate.frameTimer.performAfterDelay(delayFrames, fn, ...)
playdate.frameTimer.updateTimers()   -- call in playdate.update
playdate.frameTimer:pause() / start() / reset()
```

Fields: `paused`, `duration`, `currentTime`, `timeLeft`, `startValue`,
`endValue`, `easingFunction`, `repeats`, `reverses`. Metatable enforces
read-only-ness on time-tracking fields.

## `graphics.lua` — misc gfx helpers

Adds Lua-only convenience over the C `playdate.graphics` bindings:

```lua
playdate.graphics.drawCircleInRect(x, y, w, h)
playdate.graphics.fillCircleInRect(...)
playdate.graphics.drawCircleAtPoint(x, y, radius)
playdate.graphics.fillCircleAtPoint(...)
playdate.graphics.drawArc(x, y, r, startAngleDeg, endAngleDeg)
playdate.graphics.drawSineWave(x1, y1, x2, y2, a1, a2, period, phaseShift)

-- LCDBitmap:drawAnchored(x, y, cx, cy, flip)
--   cx, cy in 0..1 for anchor point
```

## `keyboard.lua` — on-screen keyboard

Full modal input widget with default Roobert font. Uses assets:

```lua
import 'CoreLibs/assets/keyboard/Roobert-24-Keyboard-Medium-table-36-36.png'
import 'CoreLibs/assets/keyboard/menu-{del,cancel,ok,space}.png'
import 'CoreLibs/assets/sfx/{click,denial,key,selection*}.wav'
```

API:

```lua
playdate.keyboard.show(initialText?)
playdate.keyboard.hide()
playdate.keyboard.text           -- current buffer
playdate.keyboard.textChangedCallback = function() end
playdate.keyboard.keyboardWillHideCallback = function(ok) end   -- ok=true if committed
playdate.keyboard.selectedColumn -- 0/1/2 A-Z, a-z, punctuation
```

Crank scrolls current column; A commits letter, B deletes, Menu opens
special-keys row.

## `logic.lua` — boolean helpers

```lua
playdate.math.logic.nor(a, b)
playdate.math.logic.xor(a, b)
playdate.math.logic.nand(a, b)
playdate.math.logic.nxor(a, b)
```

## `math.lua`

```lua
playdate.math.lerp(min, max, t)   -- t in 0..1
```

Everything else is stdlib `math`.

## `nineslice.lua` — `playdate.graphics.nineSlice`

9-slice scalable UI panel. Cuts a source image into 9 rectangles by
inner-rect coords, then tiles/stretches the middle.

```lua
local ns = playdate.graphics.nineSlice.new(path, innerX, innerY, innerW, innerH)
ns:drawInRect(x, y, w, h)
ns:getSize()   -- native corner size
ns:getMinSize()
```

## `object.lua` — `class()` / `Object`

The Playdate OO model. See [idioms.md](idioms.md) for details. Provides:

```lua
class('Foo').extends(Base)                -- declares Foo class inheriting Base
class('Foo').extends()                    -- inherits Object
Foo.className, Foo.super, Foo.super.init
function Foo:init(...) end
Foo:isa(Bar)
```

## `qrcode.lua` — QR encoding

Wraps `3rdparty/qrencode.lua` (MIT-licensed pure-Lua QR encoder) with
Playdate-friendly bitmap output.

```lua
playdate.graphics.generateQRCode(stringToEncode, desiredSize?, callback)
-- callback(bitmap, errorCode?)
-- runs incrementally — split across many frames — because full QR
-- encode is too slow for one update tick.
```

## `save.lua` — savegame helpers

Just a wrapper convention:

```lua
playdate.datastore.write(table, path?, pretty?)   -- serialize to path (default "data")
playdate.datastore.read(path?)                    -- returns table or nil
playdate.datastore.delete(path?)
playdate.datastore.writeImage(bmp, path)
playdate.datastore.readImage(path)
```

Data lives under `/Data/<bundleID>/`. Default filename `data.json`.

## `sprites.lua` — sprite class extensions

Wraps the C `playdate.graphics.sprite.*` with Lua-friendly OO:

```lua
class('MySprite').extends(playdate.graphics.sprite)
function MySprite:init()
    MySprite.super.init(self)
    self:setImage(playdate.graphics.image.new(...))
end
function MySprite:update() end
function MySprite:draw(x, y, w, h) end
```

Auto-registers to the collision world.

## `strict.lua` — undefined-global guard

Sets a global metatable trap that errors on any read of an undeclared
global. Enable with `import 'CoreLibs/strict'` at the top of `main.lua`.

## `string.lua` — string helpers

```lua
string.split(s, delimiter)         -- returns table
string.trimTrailingZeros(numStr)   -- "1.50000" -> "1.5"
string.trimLeadingZeros(s)
```

Attached to the `string` metatable, so callable as `s:split(",")`.

## `timer.lua` — `playdate.timer`

Real-time timer/tween.

```lua
playdate.timer.new(durMs, startVal?, endVal?, easingFn?)
playdate.timer.performAfterDelay(delayMs, fn, ...)
playdate.timer.keyRepeatTimerWithDelay(startDelayMs, holdDelayMs, fn)
playdate.timer.updateTimers()   -- call in playdate.update
-- Instance:
t.paused, t.timerEndedCallback, t.updateCallback, t.timerEndedArgs
t.repeats, t.reverses, t.discardOnCompletion
t.duration, t.currentTime, t.timeLeft, t.startValue, t.endValue, t.value
t:pause(), t:start(), t:reset(), t:remove()
```

Same metatable trap as `frameTimer` — read-only time fields error on
assignment.

## `ui.lua`

Empty stub — loads only what's under `ui/`.

## `ui/crankIndicator.lua`

Draws the "please use crank" HUD arrow + pulse animation.

```lua
playdate.ui.crankIndicator:start()
playdate.ui.crankIndicator:update()   -- call in playdate.update
```

Auto-hides when crank is undocked and being turned.

## `ui/gridview.lua` — `playdate.ui.gridview`

Grid-scrolling widget (used by Launcher, Settings, Catalog UI).

```lua
local gv = playdate.ui.gridview.new(cellW, cellH)
gv:setNumberOfRows(n)
gv:setNumberOfColumns(n)
gv:setNumberOfSections(n)
gv:setNumberOfRowsInSection(section, n)
gv:setSectionHeaderHeight(h)
gv:setContentInset(left, right, top, bottom)
gv:setCellPadding(l, r, t, b)
gv:setSelection(section, row, col)
gv:selectPreviousRow(wrap?, scrolls?, forceScroll?)
gv:selectNextRow(...)
gv:drawInRect(x, y, w, h)
-- Callbacks user overrides:
gv.drawCell = function(sec, row, col, selected, x, y, w, h) end
gv.drawSectionHeader = function(sec, x, y, w, h) end
```

## `utilities/sampler.lua` — CPU sampler

Frame-time and CPU profiling helper. Emits per-frame timing to the
console for later graphing.

## `utilities/where.lua`

Prints file:line at each call — cheap `console.trace`-style debugging.

## `3rdparty/qrencode.lua` + `qrencode_panic_mod.lua`

Pure Lua QR encoder. `qrencode.lua` is Kevin Lange's MIT-licensed
implementation; `_panic_mod` patches it to yield periodically so the
main thread stays responsive during encoding.

## Import chains observed

- `frameTimer.lua` → `CoreLibs/easing`
- `graphics.lua`   → `CoreLibs/string`
- `logic.lua`      → `CoreLibs/object`
- `animation.lua`  → `CoreLibs/object.lua`
- `animator.lua`   → `CoreLibs/easing.lua`
- `keyboard.lua`   → assets + sfx (see above)

`pdc` follows these transitively to include only what the game uses.
