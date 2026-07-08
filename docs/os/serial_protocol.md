# USB CDC Serial Protocol

Playdate exposes a line-oriented ASCII shell over USB CDC ACM. Baud is
irrelevant (virtual serial). This is the mechanism used by `pdutil`,
`pdc`, Simulator's Mirror mode, and the Panic Setup app.

## Attaching

```
macOS:   /dev/tty.usbmodemPD*   (or /dev/cu.usbmodemPD*)
Linux:   /dev/ttyACM0            (needs dialout/plugdev group)
Windows: COMN                    (Device Manager)
```

Any terminal works: `screen`, `minicom`, `putty`. Press `?` at the
prompt to enumerate commands.

## Prompt

```
> 
```

Every command:

```
> <verb> [args...]\r\n
<response line(s)>\r\n
ok\r\n
```

or on failure:

```
err <message>\r\n
```

## Verified commands (from `pdutil` binary strings)

| Verb | Args | Effect |
|------|------|--------|
| `datadisk` | — | Detach CDC; expose Data partition as USB Mass Storage. |
| `bootdisk` | — | Same but expose System (OS) partition. |
| `recoverydisk` | — | Same for recovery partition. |
| `install` | (streams tar) | Firmware enters upload mode; receives pdx tarball on the same CDC channel. |
| `run` | `<path>` | Launch a pdx already on device; `<path>` is bundle relative to `/Games/`. |

## Additional commands (community-documented, partially verified)

Derived from Setup/Launcher UI flows and firmware string dumps:

| Verb | Args | Effect |
|------|------|--------|
| `?` / `help` | — | List commands |
| `version` | — | Print firmware version (e.g. `3.0.4`) |
| `serial` | — | Print device serial number |
| `echo` | `on|off` | Toggle host input echo |
| `serialecho` | `on|off` | Mirror in-game `logToConsole` to serial |
| `screencap` | — | Send framebuffer over serial (400×240 raw bits + rows) |
| `screenstream` | `on|off` | Continuous framebuffer stream (Mirror uses this) |
| `stats` | — | Runtime stats: heap, uptime, etc. |
| `hibernate` | — | Enter deep sleep (test only) |
| `stop` | — | Terminate running game; back to launcher |
| `wifi` | `scan | connect <ssid> <psk> | status | forget <ssid>` | Wi-Fi management |
| `bt` | (various) | BLE (unused publicly) |
| `crashlog` | — | Print stored crashlog.txt |
| `resetgame` | — | Clear current game's Data/ (rare; use carefully) |
| `changetimezone` | `<offset>` | Set TZ (Settings.pdx uses this) |
| `settime` | `<epoch>` | Set RTC |
| `btntest` | — | Diagnostic input loop |
| `crank` | `docked | undocked` | Simulate crank state |
| `format` | — | Wipe user data (dangerous) |
| `msc off` | — | Force MSC end (mostly for exit-from-datadisk) |

**Status**: only the first table is verified from `pdutil` strings. The
rest are inferred from usage patterns; poke your own device with `?` to
enumerate the real set on your firmware.

## Framing during `install`

The `install` verb switches the CDC channel from ASCII shell mode to
binary. Firmware then expects:

```
<length u32 LE>
<tarball bytes>
```

Where tarball is a POSIX ustar of the pdx directory. Firmware unpacks
into `/Games/<bundleID>/`, sends `ok\r\n`, returns to shell.

`pdutil install` handles this; not documented for third-party clients.

## Mass Storage mode

`datadisk` / `bootdisk` / `recoverydisk` switch the USB device class:

1. Firmware un-registers CDC ACM.
2. Registers USB Mass Storage Class exposing the requested partition.
3. Host OS enumerates the new device (may prompt / auto-mount).
4. When host cleanly ejects, firmware re-registers CDC.

The Playdate volume name is `PLAYDATE`.

## Serial console log

Any in-game `logToConsole` (or Lua `print`) writes to the CDC channel
if `serialecho` is on. Line format:

```
<UTC timestamp> [<bundleID>] <message>\r\n
```

Used by `pdutil` for tail-style tracing during development.

## Reset behavior

- Hard reset button-combo (Menu + Lock, hold ~5 s) → full boot.
- Watchdog reset from firmware assert → recorded in `rcccsr` in crashlog.
- USB unplug does **not** reset — device continues on battery.

## Baud & flow control

USB CDC ACM ignores physical baud. Some hosts still respect DTR/RTS for
line events — Playdate ignores those too. No hardware flow control.

## Rate limits

The firmware's CDC ring buffer is ~2 KB. Overrunning it during `install`
or `screenstream` causes drops without notification. `pdutil` throttles
by watching the `ok` acks.
