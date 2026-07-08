# `pdxinfo` — Full field catalog

Expands [pdxinfo.md](pdxinfo.md) with every observed field across
Purchased/Sideload/User/Season pdx bundles.

## Grammar

Plain UTF-8 text, `key=value` per line, LF or CRLF. Keys are
case-insensitive on lookup (firmware lower-cases both sides) but the
canonical form is lowercase.

## Complete field catalog

| Key | Type | Required | Notes |
|-----|------|----------|-------|
| `name` | UTF-8 | ✅ | Displayed on Launcher tile |
| `bundleid` | dotted string | ✅ | Reverse-DNS; Data/ dir key; unique |
| `author` | UTF-8 |  | Studio / individual |
| `description` | UTF-8 |  | Long text; wraps in Settings info panel |
| `version` | string |  | User-visible ("1.0", "2.3.1-beta") |
| `buildnumber` | int |  | Monotonic; Catalog update trigger |
| `pdxversion` | int |  | major*10000 + minor*100 + patch, e.g. 30100 = 3.1.0 |
| `buildtime` | int |  | UNIX seconds when `pdc` ran |
| `imagepath` | path |  | Launcher card dir; default `launcher/` |
| `launchsoundpath` | path |  | Sound played during launch animation (usually `.pda`) |
| `wrappingpattern` | path |  | Launcher wrap pattern PNG source (`pdc`-processed to `.pdi`) |
| `contentwarning` | UTF-8 |  | First-launch modal |
| `contentwarning2` | UTF-8 |  | Second line of same |
| `hash` | sha256hex |  | Catalog only — see [../os/drm.md](../os/drm.md) |
| `type` | enum |  | `"game"` (default), `"tool"`, `"gamegroup"` |
| `savesize` | int |  | Advisory max save-file size in bytes |
| `developer_name` | UTF-8 |  | (Catalog metadata; rare in local pdxinfo) |
| `studio_name` | UTF-8 |  | Same as `author` but distinct in Catalog |
| `pdxversion_min` | int |  | Minimum firmware `apiVersion` supported |

## Sample specimens

### Panic first-party (Setup.pdx)

```
name=Setup
author=Panic Inc.
description=First Launch Setup
bundleID=com.panic.setup
pdxversion=30100
buildtime=836082062
```

### Third-party paid Catalog game

```
name=BLOXZY
author=GEM
description=A bloxorz remake
bundleid=com.gem.bloxzy
imagepath=metadata
version=1.0
buildnumber=1
pdxversion=20706
buildtime=822501231
hash=ded0807a0c3e80032cc513cc3a0d93e47aa83c3b04ad0807b58e41fd50dc3688
```

### User-submitted (Catalog free)

```
name=Dig-Pit Runner
author=CrankWork Games
description=Arcade action game
bundleid=user.289230.com.crankwork.digpitrunner
version=0.931
buildnumber=3
imagepath=SystemAssets/
launchsoundpath=
wrappingpattern=wrapping-pattern.png
pdxversion=30002
buildtime=819731171
```

Note: `user.289230.` prefix on `bundleid` — first-party Catalog convention
for community uploads. The numeric ID is the uploader's Panic account.

### Pulp export

```
name=What time is it?
author=Amigos 500
description=Use logic to solve time puzzles
bundleid=com.amigos500.whattimeisit
version=1.0
buildnumber=20
imagepath=images
pdxversion=20706
buildtime=812665255
hash=e7f71dfbf4d0b3cae4f225728fc82d66b5bd80ccfd5929024b61978eae98eb40
```

Pulp exports use `pdxversion=11000` or `20706` depending on Pulp editor
version — always trails behind current SDK.

## Case sensitivity

The user-facing pdxinfo canon is lowercase (`bundleid` not `bundleID`),
but firmware lower-cases both key and comparison lookup. In practice
either works, but `pdc` output uses the canonical case per SDK version:

- SDK ≤ 2.x: `bundleID` (camelCase)
- SDK 3.x: `bundleid` (lowercase)

Both are observed in the wild.

## Runtime access from Lua

```lua
local m = playdate.metadata()
-- m is a table keyed by lowercased pdxinfo keys
print(m.name, m.bundleid, m.pdxversion)
-- Catalog-specific:
if m.hash then print("Catalog build; hash:", m.hash) end
```

`playdate.metadata()` returns fresh table each call. Values are strings
(no type coercion — `m.pdxversion` is `"30100"`, not `30100`).

## Runtime access from C

`system->getSystemInfo()->pdxversion` returns the `pdxversion` field
as `uint32_t`. Other fields have no dedicated C accessor — game code
must open `pdxinfo` directly if needed:

```c
SDFile* f = pd->file->open("pdxinfo", kFileRead);
char buf[512];
int n = pd->file->read(f, buf, sizeof buf);
pd->file->close(f);
// parse manually
```

## Encoding gotchas

- Values are **not trimmed**. Trailing spaces survive into the runtime.
- `\n` in a value is literal 2 chars (backslash + n) — runtime substitutes
  when rendering multiline (e.g. `contentwarning`).
- Non-ASCII allowed as raw UTF-8. Japanese authors write native.
- No comments. No escape mechanism. No quoting.

## `imagepath` conventions

- Default: `launcher/` (looks for `card.pdi`, `icon.pdi`, etc.).
- Set to `.` or `./` for card at bundle root.
- Set to `metadata` for Panic's "purchased game" convention (card lives
  in `metadata/`).

## Localization

`pdxinfo` itself is not localized. `name` and `description` in `pdxinfo`
are frozen at build time. To offer localized names, use `pds` string
tables + `playdate.getLocalizedText` for on-screen text; the Launcher
tile still uses the raw `pdxinfo.name`.
