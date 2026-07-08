# Playdate Hardware

The Playdate console has gone through three hardware revisions since
launch. All expose the same firmware API to games.

## Rev A / B / C — original series

- **SoC**: STMicroelectronics STM32-family Cortex-M7. Original launch
  units documented as **STM32F746IGH6** (216 MHz + OC to 240 MHz,
  1 MB flash, 320 KB SRAM). Later revs likely moved to a larger
  **STM32H7** part — crashlogs on 3.0.3+ firmware show `pc:24000000+`
  (STM32H7 AXI flash region) and heap of 5 MB, incompatible with
  F746's 320 KB SRAM. See "Memory map from crashlog" below.
- **External RAM**: 8 MB SDRAM (typically ISSI IS42S16400J).
- **External flash**: 16 MB QSPI NOR (Winbond W25Q128).
- **LCD**: Sharp Memory-in-Pixel LS027B7DH01A, 400×240, 1-bit reflective.
- **Audio**: TI TLV320DAC3100 codec + Class-D speaker.
- **IMU**: STMicro LIS3DH accelerometer (±2 g, I²C).
- **Crank**: AMS AS5600 magnetic rotary encoder, I²C, 12-bit angle.
- **Buttons**: 6 (D-pad + A + B + Menu + Lock).
- **USB**: USB 2.0 full-speed, CDC ACM (serial console) + mass storage
  in recovery mode.
- **Battery**: LiPo 740 mAh, 3.7 V nominal.
- **Headset**: 3.5 mm TRRS, CTIA pinout.

## Rev D — 2024+ (Wi-Fi capable)

Adds:

- **Wi-Fi/BT**: ESP32-C3 co-processor on internal UART, exposed via
  `playdate_network`. 2.4 GHz 802.11 b/g/n, BLE 5.0 (BLE unused
  publicly).
- **Firmware ≥ 3.0** required for the network APIs.
- Older units (Rev A/B/C) get an "N/A" from `network->getStatus()`
  returning `kWifiNotAvailable`; all HTTP calls fail with `NET_NO_DEVICE`.

## Memory map (from crashlog)

Extracted from `/Volumes/PLAYDATE/crashlog.txt` on a real device
running firmware 3.0.3 / 3.0.4:

```
0x00000000  ITCM view (F/H series)
0x08000000  Legacy flash bank    (F7 units only)
0x20000000  SRAM1 / SRAM2        (heap fragments seen at 0x20010000+)
0x24000000  AXI-mapped flash     ← current firmware runs from here (pc: 0x24061bxx)
0x38000000  ? (r2 seen = 0x300002cc — SRAM4/DTCM on H7)
0x40000000  Peripheral regs      (bfar seen at 0x40022000 = flash controller)
0x60000000  FMC bank 1           (QSPI / external NOR)
0x90000000  QSPI memory-mapped   (r0 = 0x9023e078 in one crash)
0xB0000000  ? (mmfar seen = 0xB024EF98 — likely bad address, fault region)
0xC0000000  FMC bank 5 SDRAM     (8 MB game heap; heap allocated up to 5.5 MB)
0xE0000000  Cortex-M private peripheral bus
```

The `pc:24061bb2` values consistently seen in fault frames confirm
firmware runs from AXI-mapped flash at `0x24000000` — a **STM32H7-only
memory mapping**. Rev D units (Wi-Fi capable) probably ship an H7A3
or H750 variant. Earlier F7 units would show `pc:08xxxxxx`.

## Memory layout (device)

See [../os/memory.md](../os/memory.md) for the full address map. Summary:

- Firmware runs from internal flash `0x08000000`.
- Games load into SDRAM at `0xC0100000+`.
- Framebuffer × 2 lives in internal SRAM for latency.

## LCD driver

The Sharp panel talks a 3-wire SPI:

- `SCLK`, `SI`, `SCS` (chip select active-high, oddly).
- 8 MHz clock; 400 bits + 16-bit dummy per row.
- Command byte format: `[M2 M1 M0 · · · · A]` where the low bit is
  address MSB.
- VCOM inversion signal must toggle at ~1 Hz.

Firmware drives it via SPI DMA; the CPU only pays for row-diff
computation.

## Power

- Battery monitor via STM32 ADC + resistor divider.
- Charger: TI BQ25895 I²C charge controller (USB VBUS in).
- Screws contact (stereo dock) shows up via GPIO — reported as
  `kPDPowerStatusScrews` bit.

## USB serial protocol

pdutil talks to the firmware over `/dev/tty.usbmodemPD*`. The console
speaks line-oriented commands (from strings dump):

```
run <path>            # launch a pdx from /Games/
datadisk              # switch to Mass Storage class, mount data partition
bootdisk              # switch to MSC, mount main OS partition
recoverydisk          # MSC, mount recovery partition
install               # accept a pdx stream (tarball, followed by 'install' ack)
```

There are more commands not exposed via `pdutil` but visible via a raw
`screen /dev/tty.usbmodemPD* 115200` session — press `?` at the prompt
to enumerate. Some (partial) inventory:

```
help
version
echo on|off
serialecho on|off
btlp                  # BLE low-power mode
crashlog              # print last crash dump
```

## Crash reporting

On hard fault, firmware writes a `crashlog.txt` to the root of the
data partition (visible under `/Volumes/PLAYDATE/crashlog.txt` when
`datadisk` is active). Contains registers (`r0..r3, r12`, `lr`, `pc`,
`psr`), fault-status regs (`cfsr`, `hfsr`, `mmfar`, `bfar`,
`rcccsr`), heap counter, and Lua VM state. See
[../os/device_disk.md](../os/device_disk.md#crashlogtxt-analyzed) for
the full field breakdown.

Firmware build identifier line format:

```
build:<git-hash-short>-<version>-release.<build#>-gitlab-runner
```

Panic uses a self-hosted GitLab CI runner.

Use `firmware_symbolizer.py` + `arm-none-eabi-addr2line` against your
game's ELF:

```
python3 firmware_symbolizer.py crashlog.txt Game.pdx/pdex.elf
```

The symbolizer just extracts `lr:` and `pc:` from each stanza and
pipes through addr2line.
