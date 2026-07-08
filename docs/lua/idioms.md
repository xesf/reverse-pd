# Lua Idioms

Conventions used across Playdate's SDK Lua code.

## `class()` — OO on top of tables

Defined in `CoreLibs/object.lua`. Style:

```lua
class('Player').extends()   -- inherits Object (default)
class('Enemy').extends(Player)   -- inheritance

function Player:init(x, y)
    Player.super.init(self)   -- always call super:init
    self.x, self.y = x, y
end

function Player:move(dx, dy)
    self.x += dx  -- Lua-with-compound-ops (playdate patched Lua)
    self.y += dy
end

local p = Player(10, 20)     -- __call metamethod on class runs :init
p:move(1, 0)
print(p:isa(Player))         -- true
print(p:isa(Enemy))          -- false (upcast test only)
```

`class('Name')` creates a new global `Name` (yes, actually global). The
returned object is a class table with:

- `Name.className` = `'Name'`
- `Name.super` = the parent class table (or `Object`)
- `Name.new` — factory that calls `:init`
- `__index = Name` (methods resolve through class)

`Name(...)` uses the metatable's `__call` to run `Name.new(...)`.

## `import`

Playdate's `import` is **not** stock Lua `require`. It:

- Runs a source file **once**, at compile-time by `pdc`.
- Bakes the imported bytecode into the enclosing `main.pdz`.
- Supports both `"CoreLibs/foo"` (SDK paths) and `"src/mylib"` (project-local).
- Extension optional; `.lua` implied.
- Circular imports OK — second load is skipped.

Runtime `import` is a **noop** — everything's already loaded. Use it
freely at top-of-file; there's no perf cost.

## Compound assignment operators

Playdate's Lua 5.4 fork adds `+=`, `-=`, `*=`, `/=`, `%=`:

```lua
self.x += 1     -- lexed as self.x = self.x + 1
self.hp -= dmg
```

Stock Lua rejects these. `pdc` accepts them, then transpiles.

## Metatable field guards

`playdate.timer` and `playdate.frameTimer` use `__index` +
`__newindex` to expose derived read-only fields:

```lua
playdate.timer.__index = function(t, k)
    if k == "timeLeft" then
        return math.max(0, t.duration - t._currentTime)
    elseif ...
    else return rawget(playdate.timer, k) end
end

playdate.timer.__newindex = function(t, k, v)
    if k == "timeLeft" or k == "currentTime" then
        print("ERROR: playdate.timer."..k.." is read-only.")
    elseif ...
    else rawset(t, k, v) end
end
```

Pattern lets external users read `t.timeLeft` as if it were a stored
field while it's computed on access. Warning-based (not error) to keep
buggy games running.

## Table.create pre-sizing

`table.create(arrCount, hashCount)` is a Playdate/PUC-Lua extension
that pre-allocates the internal array + hash halves of a Lua table:

```lua
local grid = table.create(400, 0)   -- known 400 elements, avoids grow-rehash
for i = 1, 400 do grid[i] = 0 end
```

Big win for large fixed-size arrays. Zero-init still needed after
allocation.

## Global vs. local best practice

Playdate LSP does not enforce local-first. Games leak globals freely.
Enable `CoreLibs/strict` to catch typos:

```lua
import 'CoreLibs/strict'
-- from here on, reading an undeclared global raises an error
```

Also catches `Player = class(...)` (globals) vs `local Player = ...`
mistakes.

## Sprite-class inheritance

Standard idiom for game entities:

```lua
class('Enemy').extends(playdate.graphics.sprite)

function Enemy:init()
    Enemy.super.init(self)
    self:setImage(playdate.graphics.image.new("images/enemy"))
    self:setCollideRect(0, 0, self:getSize())
    self:add()   -- register into scene
end

function Enemy:update()
    self:moveWithCollisions(self.x + self.vx, self.y + self.vy)
end
```

The base `sprite` class's `:update()` no-ops; override to add per-frame
behavior.

## `playdate.update` main callback

The Lua VM auto-registers a wrapper for `system->setUpdateCallback`
that calls the global `playdate.update`:

```lua
function playdate.update()
    playdate.timer.updateTimers()
    playdate.frameTimer.updateTimers()
    playdate.graphics.sprite.update()
    -- draw HUD etc.
end
```

Returning nothing from `playdate.update` is fine; the wrapper triggers a
display flip automatically.

## Callbacks vs. inheritance

Most subsystems support both:

- **Callback**: `t.timerEndedCallback = function(t) end`
- **Class method**: `class('MyTimer').extends(playdate.timer)` with `:timerEnded()`.

Callbacks are per-instance and cheaper; class methods scale better for
lots of similar instances.

## Userdata retention

Native objects wrapped by the C API's `pushObject` are refcounted. When
returned to Lua, the refcount is bumped once for the Lua handle. When
Lua GC collects the wrapper, the refcount drops.

**Gotcha**: storing a raw `LCDBitmap` returned by `getTableBitmap(t, i)`
into a Lua field does not extend its lifetime beyond the table — the
docstring warns not to free it independently. In practice, hold a
reference to the table, not to the returned bitmap.

## Coroutines

Full Lua 5.4 coroutines available. Playdate games use them heavily for
per-entity state machines:

```lua
function Enemy:update()
    if not self.co or coroutine.status(self.co) == "dead" then
        self.co = coroutine.create(function() self:think() end)
    end
    coroutine.resume(self.co)
end
```

Yielding in `update` is cooperative — the update tick finishes, next
frame resumes the coroutine. Watch stack allocations: each coroutine
gets its own Lua stack (~4 KB minimum).
