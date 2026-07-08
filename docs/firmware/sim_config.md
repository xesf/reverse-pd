# `~/.Playdate/config` — SDK toolchain config

Plain-text config file read by `pdc`, `pdutil`, and the Simulator at
startup. Location:

- macOS/Linux: `~/.Playdate/config`
- Windows: `%USERPROFILE%\.Playdate\config`

## Format

Tab-separated key/value, one per line, no comments, no quoting.

Verified live sample:

```
SDKRoot	/Users/xesf/Developer/PlaydateSDK
```

## Known keys

| Key | Value | Notes |
|-----|-------|-------|
| `SDKRoot` | absolute path | Points to the installed SDK directory; used by `pdc` if `-sdkpath` and `PLAYDATE_SDK_PATH` env var are not set |
| `SimulatorLastGame` | path | Last opened project in Simulator (auto-updated) |
| `SimulatorWindowScale` | int | 1..8, LCD upscale factor |
| `SimulatorInverted` | 0/1 | Sim palette inverted preview |
| `SimulatorFramerate` | int | Cap for Simulator (default 30) |
| `TelemetryEnabled` | 0/1 | Simulator crash/telemetry opt-out |
| `PlaydatePath` | path | Last-connected device path for `pdutil` shortcuts |

**Not all keys are verified** — only `SDKRoot` seen live on this
machine. Others inferred from Simulator UI and public support notes.

## Resolution order for `SDKRoot`

`pdc` looks for SDK path in order:

1. `-sdkpath <path>` CLI flag
2. `PLAYDATE_SDK_PATH` env var
3. `~/.Playdate/config` `SDKRoot` line
4. Default `~/Developer/PlaydateSDK`

If none of these resolves to a real dir, `pdc` errors:

```
SDK not found at path "%@", please check your ~/.Playdate/config
file to make sure SDKRoot is set properly.
```

## Auto-creation

Simulator creates the file (and `~/.Playdate/`) on first run if absent.
Directory permissions: `0700`. File permissions: `0644`.

## Sample expanded

Typical file after Simulator has been used for a while:

```
SDKRoot	/Users/user/Developer/PlaydateSDK
SimulatorLastGame	/Users/user/Documents/mygame/mygame.pdx
SimulatorWindowScale	2
SimulatorInverted	0
SimulatorFramerate	30
TelemetryEnabled	1
```

## Related

- `~/.Playdate/GamePlaydateSDK/` — Simulator-only sandbox for save
  data (mirrors the device's Data/ partition).
- `~/Developer/PlaydateSDK/Disk/Data/` — Data/ overlay for
  Simulator-run games.
