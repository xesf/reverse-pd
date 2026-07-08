# pdutil — Device USB Utility

Located at `bin/pdutil`. Mach-O universal. Talks to the Playdate over
USB CDC serial (`/dev/tty.usbmodemPD*` on macOS, `/dev/ttyACM0` on
Linux, `COM*` on Windows).

## Usage (verbatim from binary)

```
Usage: pdutil <device> <action> [options]
  device:
    path to Playdate serial port, ie; /dev/ttyACM0
  actions:
    datadisk           - Mount Playdate data partition
    recoverydisk       - Mount Playdate recovery partition
    bootdisk           - Mount Playdate boot partition
    install <path>     - Install .pdx bundle
    run <path>         - Run .pdx from device's data partition
```

## Serial protocol

The firmware exposes a line-oriented command shell over CDC ACM. Baud is
irrelevant (USB CDC is a virtual serial port, physical baud is ignored).

Observed protocol (deduced from strings + external community RE):

```
> <command>\n
< <response>\n
< ok\n            # or 'err <msg>\n'
```

Commands relevant to `pdutil`:

| Command       | Effect |
|---------------|--------|
| `datadisk`    | switch USB personality to Mass Storage, expose the Data partition |
| `bootdisk`    | expose the main system partition (OS + system pdx) |
| `recoverydisk`| expose the recovery partition |
| `run <path>`  | firmware launches `Games/<path>` immediately |
| `install`     | firmware enters upload mode; pdutil then streams a tarball |
| `stop`        | terminate the currently running game, return to Launcher |
| `?` / `help`  | list commands |
| `version`     | firmware `x.y.z` string |
| `echo on|off` | toggle serial echo |
| `serialecho on|off` | toggle in-game `logToConsole` mirroring |

## Install flow

```
pdutil /dev/tty.usbmodemPD1 install MyGame.pdx
```

Steps:
1. pdutil opens the port, sends `install\n`.
2. Firmware responds `ready\n` and waits.
3. pdutil sends a length-prefixed tar of the pdx directory.
4. Firmware unpacks into `/Games/<bundleID>/`.
5. Firmware responds `ok\n`.
6. pdutil closes.

`bundleID` comes from the pdxinfo inside the tar, not the on-disk
directory name — moving a source folder doesn't confuse the install.

## Mount flow

`datadisk` and friends:
1. pdutil sends `datadisk\n`.
2. Firmware unmounts internal filesystem, remaps USB to Mass Storage
   class exposing the requested partition.
3. Host OS mounts it as a normal removable drive (`PLAYDATE` volume on
   macOS).
4. When the user ejects, firmware remounts internal FS and returns to
   CDC serial mode.

## Errors

Verbatim from binary:

```
Device %s does not exist (Playdate not connected?)
Couldn't open %s (sudo required?)
Error %d from tcgetattr
Error %d from tcsetattr
Write failed
Please provide the path to an application on device, such as Games/MyGame.pdx
Unknown action: "%s"
```

Linux needs the invoking user in the `dialout` group (or `plugdev`)
for the serial port; macOS is permissive.

## Not exposed but present

The full serial shell has more commands (Wi-Fi provisioning, crash log
dump, calibration) not wrapped by `pdutil`. Poking around with a raw
serial terminal is the way to discover them, but Panic doesn't
guarantee stability.
