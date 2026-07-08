# .pdx — Playdate Bundle

**Not a file — a directory.** The `.pdx` extension is set on a
filesystem directory that acts as a self-contained game bundle. This is
the unit that `pdc` produces and that the Playdate OS launches.

## Layout

Mandatory:

```
Foo.pdx/
├── pdxinfo          plain-text KV metadata (see pdxinfo.md)
└── main.pdz         Playdate PDZ archive; entry point
```

Optional, per-convention:

```
├── images/          .pdi and .pdt assets not embedded in main.pdz
├── fonts/           .pft
├── sfx/             .pda (per-game sound effects)
├── systemsfx/       overrides for system sounds (used by Launcher.pdx)
├── video/           .pdv playback assets
├── videos/          alias of video/ (Setup.pdx uses this)
├── en.pds           localized strings — Japanese variant jp.pds, etc.
├── jp.pds
├── Server/          web resources served by network games (Setup uses this)
└── <bundle>.pdx.zip when distributed via Catalog (single zip'd bundle)
```

Every asset in the bundle is either:
- Discoverable at path `Foo.pdx/rel/path.pdi` via the file API, OR
- Included inside `main.pdz` under the same relative path, discoverable via the
  Lua/asset loaders that transparently search the PDZ table first, then the
  disk bundle.

## pdxinfo (bundle manifest)

Plain UTF-8 text, key=value per line, no comments, no whitespace around
`=`. Example (`Setup.pdx/pdxinfo`, verbatim):

```
name=Setup
author=Panic Inc.
description=First Launch Setup
bundleID=com.panic.setup
pdxversion=30100
buildtime=836082062
```

Known keys:

| Key           | Type    | Notes |
|---------------|---------|-------|
| `name`        | UTF-8   | Displayed on Launcher tile |
| `author`      | UTF-8   | |
| `description` | UTF-8   | Long text; wraps in Settings info panel |
| `bundleID`    | dotted  | Reverse-DNS; also the Data/ dir key |
| `version`     | string  | Free-form; user-visible |
| `buildNumber` | integer | Monotonic; Catalog update trigger |
| `pdxversion`  | integer | `major*10000 + minor*100 + patch`; runtime uses to gate features |
| `buildtime`   | integer | UNIX epoch seconds of `pdc` invocation |
| `imagePath`   | path    | Custom launcher-card image (default `launcher/card.png`) |
| `contentWarning` | UTF-8 | Shown on first launch |
| `contentWarning2` | UTF-8 | Second line |
| `launchSoundPath` | path | Sound to play during launch animation |

## Launcher-card assets

By convention (all optional, all under bundle root):

```
launcher/
├── card.pdi          380×230 static card
├── card-highlighted/ animated card (bitmap table)
├── card-pressed.pdi  frame shown briefly on select
├── icon.pdi          32×32 grid-view icon
├── launchImage.pdi   full-screen "loading"
└── card-launchImages/ animated launch sequence
```

The Launcher scans these at first-run and caches thumbnails in
`Data/com.panic.launcher/`.

## Compilation

Author-side files (`.lua`, `.png`, `.wav`, `.mid`) live in a **source
project** directory alongside a `pdxinfo`. `pdc` walks it and:

1. Compiles all `.lua` to Lua 5.4 bytecode, packs them into `main.pdz`.
2. Converts `.png` → `.pdi`, `.gif` → `.pdt` (frame table).
3. Converts `.wav` → `.pda` (ADPCM), `.mp3` passed through.
4. Converts `.mid` → keeps as `.mid` (loaded by `sound_sequence`).
5. Converts font sources (`.fnt`+`.png`) → `.pft`.
6. Emits `pdxinfo` verbatim (or with computed `buildtime`).
7. Wraps everything in the `Foo.pdx/` directory.

Non-Lua assets referenced by Lua go both into `main.pdz` **and** stay as
loose files in the bundle when the Lua source imported them by path.
The runtime prefers the PDZ table for imports; loose files back
`playdate.file.open()`.

## Simulator install

Simulator watches `~/Developer/PlaydateSDK/Disk/Games/` (or the Windows
equivalent). Any directory ending in `.pdx` shows up as an
installed title. Copy or symlink your build here.

## Device install

Over USB CDC, `pdutil install foo.pdx` (see `bin/pdutil`) speaks the
Playdate serial protocol to enter mass-storage or update mode. The pdx
tree is streamed as a tarball to `/Games/` on the internal flash.
