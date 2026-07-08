# `playdate.*` Global Namespace

Lua-side mirror of the C `PlaydateAPI` vtable. Populated by the runtime
before `main.lua` executes. All subsystem structs from the C API
become sub-tables here.

## Structure

```
playdate                    -- root; sys functions here
├── graphics                -- gfx primitives, LCDBitmap wrappers
│   ├── sprite              -- retained-mode scene
│   ├── image               -- alias of LCDBitmap namespace
│   ├── imagetable          -- LCDBitmapTable
│   ├── font                -- LCDFont
│   ├── video               -- LCDVideoPlayer
│   ├── videostream         -- LCDStreamPlayer (3.0+)
│   ├── tilemap             -- LCDTileMap (3.0+)
│   ├── animation           -- Lua-only loop helper
│   ├── animator            -- Lua-only tweening
│   └── nineSlice           -- 9-slice panels
├── sound
│   ├── channel
│   ├── source              -- base class
│   ├── fileplayer
│   ├── sample
│   ├── sampleplayer
│   ├── synth
│   ├── instrument
│   ├── track               -- SequenceTrack
│   ├── sequence            -- SoundSequence
│   ├── controlsignal
│   ├── signal
│   ├── envelope
│   ├── lfo
│   ├── effect
│   │   ├── twopolefilter
│   │   ├── onepolefilter
│   │   ├── bitcrusher
│   │   ├── ringmodulator
│   │   ├── delayline
│   │   └── overdrive
│   └── micinput            -- setMicCallback wrapper
├── display                 -- LCD control
├── file                    -- file I/O
│   └── open, read, write, close, geterr, listFiles, etc.
├── datastore               -- higher-level save wrapper (CoreLibs/save)
├── json                    -- encode/decode
├── system                  -- alias — some SDK code uses playdate.system.*
├── network                 -- Wi-Fi (3.0+)
│   ├── http                -- HTTPConnection
│   └── tcp                 -- TCPConnection
├── scoreboards             -- Panic leaderboards
├── timer                   -- Lua-only real-time timers (CoreLibs)
├── frameTimer              -- Lua-only frame-count timers (CoreLibs)
├── math                    -- extends stdlib math
│   └── logic               -- boolean ops (CoreLibs/logic)
├── easingFunctions         -- 30+ easing curves (CoreLibs/easing)
├── ui                      -- UI widgets (CoreLibs/ui)
│   ├── crankIndicator
│   └── gridview
├── keyboard                -- on-screen keyboard (CoreLibs/keyboard)
├── string                  -- extends stdlib string
├── argv                    -- command-line-ish args from restartGame
└── metadata                -- pdxinfo as table
```

## Root functions

```lua
playdate.apiVersion()               -- returns (major, minor)
playdate.metadata()                 -- pdxinfo as {key = value} table
playdate.wait(ms)                   -- blocking (avoid)
playdate.getCurrentTimeMilliseconds()
playdate.getSecondsSinceEpoch()     -- returns (sec, ms)
playdate.getElapsedTime()
playdate.resetElapsedTime()
playdate.getBatteryPercentage()
playdate.getBatteryVoltage()
playdate.getPowerStatus()           -- {charging, USB, screws}
playdate.getFPS()
playdate.drawFPS(x, y)
playdate.isCrankDocked()
playdate.getCrankPosition()         -- 0..360
playdate.getCrankChange()           -- degrees since last call, (change, acceleratedChange)
playdate.getCrankTicks(ticksPerRotation)   -- CoreLibs
playdate.setCrankSoundsDisabled(bool)
playdate.getFlipped()
playdate.setAutoLockDisabled(bool)
playdate.getReduceFlashing()
playdate.getLocale()                -- "en" | "jp" | "auto"
playdate.getLocalizedText(key, lang?)
playdate.setNewlinePrinted(bool)
playdate.getLaunchArgs()
playdate.restart(argstr?)
playdate.exit()                     -- back to launcher
playdate.stop()                     -- suspend Lua VM
playdate.start()
playdate.getDebugImage()            -- Simulator only
playdate.getStats()                 -- {kernel=<us>, game=<us>, sprites=<us>, ...}
playdate.setStatsInterval(ms)
```

## Input state

```lua
playdate.buttonIsPressed(playdate.kButtonA)
playdate.buttonJustPressed(playdate.kButtonUp)
playdate.buttonJustReleased(playdate.kButtonB)
playdate.getButtonState()   -- (current, pushed, released) each as bitmask

-- Button constants
playdate.kButtonLeft, kButtonRight, kButtonUp, kButtonDown, kButtonA, kButtonB
```

## Menu items

Same shape as C API:

```lua
local item = playdate.getSystemMenu():addMenuItem("Restart", function() restart() end)
playdate.getSystemMenu():addCheckmarkMenuItem("Sound", 1, function(v) ... end)
playdate.getSystemMenu():addOptionsMenuItem("Difficulty", {"Easy","Med","Hard"}, "Med", cb)
playdate.getSystemMenu():removeAllMenuItems()
```

## `playdate.graphics`

Most C API methods are exposed as `playdate.graphics.<name>`. Bitmaps
have both a namespace API and OO methods:

```lua
-- Namespace style
local img = playdate.graphics.image.new("images/foo")
playdate.graphics.drawImage(img, 10, 10)

-- OO style (equivalent)
img:draw(10, 10)
img:drawRotated(x, y, deg)
img:drawScaled(x, y, sx, sy)
img:draw(x, y, flip, sourceRect)
img:getSize()
img:copy()
img:invertedImage()
img:blurredImage(radius, numPasses, ditherType)
img:fadedImage(alpha, ditherType)
```

## `playdate.sound`

C sound source classes are exposed as Lua classes:

```lua
local fp = playdate.sound.fileplayer.new("music/theme")
fp:play(0)              -- 0 = loop forever
fp:setVolume(0.8)
fp:setLoopRange(startSec, endSec)
fp:setFinishCallback(function(fp) end)

local sy = playdate.sound.synth.new()
sy:setWaveform(playdate.sound.kWaveformSine)
sy:playNote("A4", 0.5, 1.0)   -- freq (Hz or note name), vel, len
```

Note names: `"C4"`, `"A#4"`, `"Bb3"`, etc. Also accepts MIDI note number
(int) or Hz (float).

## `playdate.datastore` (from CoreLibs/save)

```lua
playdate.datastore.write(table, filename?, pretty?)
playdate.datastore.read(filename?)
playdate.datastore.delete(filename?)
playdate.datastore.writeImage(bmp, filename)
playdate.datastore.readImage(filename)
```

Default filename `"data"` (extension `.json` added automatically).
Location: `Data/<bundleID>/<filename>.json`.

## `playdate.file`

Direct file API. Lower level than `datastore`:

```lua
local f = playdate.file.open("levels/1.txt", playdate.file.kFileRead)
while true do
    local line = f:readline()
    if not line then break end
    -- ...
end
f:close()

-- Constants
playdate.file.kFileRead
playdate.file.kFileWrite
playdate.file.kFileAppend
playdate.file.kFileReadData     -- bundle-only, skip Data/
```

Instance methods: `read`, `write`, `readline`, `seek`, `tell`, `close`,
`flush`.

## `playdate.metadata` — pdxinfo access

`playdate.metadata()` returns a table with every key from `pdxinfo`:

```lua
local m = playdate.metadata()
print(m.name, m.author, m.bundleid, m.version, m.buildnumber)
```

Includes the `hash=` field for Catalog games — Lua code can compare it
against a runtime-computed digest for tamper detection.
