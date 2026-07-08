# pdc — Playdate Compiler

Located at `bin/pdc`. Mach-O universal binary (x86_64 + arm64). Compiles
a source project directory into a `.pdx` bundle.

## Invocation

Extracted from the binary's help string:

```
usage: pdc [-sdkpath <path>] [-I <path>] [-s] [-u] [-m] [--version] <input> [output]
```

Flags (from strings + observed behavior):

| Flag        | Meaning |
|-------------|---------|
| `-sdkpath`  | Override SDK root (default = env `PLAYDATE_SDK_PATH` or `~/.Playdate/config` `SDKRoot`) |
| `-I <path>` | Extra library search path for `import` |
| `-s`        | Strip debug (implies `--strip-symbols`) |
| `-u`        | Force uncompressed pdz entries |
| `-m`        | Emit `main.lua`-only fast path (skip asset walking) |
| `--version` | Print `pdxversion` and exit |

## Pipeline

1. **Parse config**: read `~/.Playdate/config` (INI-style,
   `SDKRoot=/path/to/PlaydateSDK`). Missing → prints "SDK not found
   at path %@..." error and exits.
2. **Read `pdxinfo`** from source root; parse into KV map.
3. **Enumerate assets**:
   - `.lua`   → compile to bytecode.
   - `.png`, `.gif` → convert via libpng → 1-bpp `.pdi`/`.pdt`.
   - `.wav`   → resample / compress → `.pda` (ADPCM).
   - `.fnt` + `.png` → link into `.pft`.
   - `.mid`   → passthrough.
   - `.strings` → collapse into per-language `.pds`.
   - Anything unrecognized → copy verbatim.
4. **Compile Lua** using bundled Lua 5.4.3 compiler (`luac`-equivalent
   linked into `pdc`).
5. **Build `main.pdz`** with:
   - `CoreLibs/*.lua` from SDK (only those `import`-referenced).
   - The game's own `.lua` bytecode.
   - Embedded assets (any asset referenced by a Lua `import` or
     `playdate.graphics.image.new`).
6. **Write bundle**:
   - `main.pdz` (mandatory).
   - Loose assets under the bundle root (mirror source layout).
   - `pdxinfo` (with `buildtime` = now unless env var overrides).
7. **Enforce size limit**: `PDZ file too large! (%d bytes > 1MB)` — the
   `main.pdz` alone can't exceed 1 MB. Larger games must split assets
   into loose files.

## Version handling

Strings show `pdxversion=%d` gets written and checked. `pdc` writes the
running SDK version into every `pdxinfo` at build time. The firmware
uses it to gate API access — e.g. a game with `pdxversion=20400`
cannot call 3.0 APIs (they're NULL in that game's `PlaydateAPI`
vtable).

## Compression

Every non-Lua asset is zlib-deflated at level 9 by default. `-u`
disables — useful when iterating with a debugger on the raw bytes.

## Simulator hot-swap

When the Simulator is running and a game is loaded from a source
directory (not a pre-built `.pdx`), it monitors mtimes and re-invokes
`pdc` on change, then reloads the plugin. This gives the "F5 reload"
UX from Panic's own dev workflow.

## Common errors

Verbatim strings:

```
PDZ file too large! (%d bytes > 1MB)
incompatible version
version mismatch: app. needs %f, Lua core provides %f
invalid conversion '%s' to 'format'
compress failed: buffer is empty
compress failed: %i
unknown compression method
```

Version mismatch: happens when a Lua file was compiled with a different
`_VERSION` metric than the runtime's Lua VM understands. `_VERSION`,
`version`, `pdxversion`, `sdkversion` are all separate — `pdxversion`
is the important one for compatibility.
