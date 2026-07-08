# Memory Model

## Hardware map (STM32F746IG)

```
0x00000000  ROM boot mirror                    (variable, see BOOT pins)
0x08000000  Internal Flash                     1 MB  (firmware image)
0x10000000  Data-TCM (DTCM) SRAM              64 KB  (fast, no cache)
0x1FF00000  System memory (bootloader)
0x20000000  SRAM1                            256 KB
0x20040000  SRAM2 + shared                    64 KB
0x60000000  FMC bank 1  (external NOR / QSPI) 16 MB  (game/asset storage)
0x90000000  QSPI direct-map                   (see fw config)
0xC0000000  FMC bank 5 (SDRAM)                8 MB   (heap / video / audio)
0xE0000000  Cortex-M private peripheral bus
```

Framebuffer, audio DMA rings, LCD command buffer, and the Lua VM stack
live in internal SRAM for latency. Game heap (Lua GC arena, C `malloc`,
big asset caches) lives in the SDRAM at `0xC0000000+`.

## Heap

`system->realloc(ptr, size)` is a wrapper over the runtime's internal
allocator:

- Backing storage: SDRAM (`0xC0000000..0xC07FFFFF`, 8 MB total).
- Allocator: dlmalloc-derived; free-list with size classes.
- Alignment: 8 bytes.
- Metadata: 8-byte header per block (size + status).

Games budget is soft — actually all 8 MB minus firmware carveouts (~1
MB reserved for framebuffer × 2, audio DMA, LCD command ring, and the
Lua VM). Practical ceiling: ~6.5 MB.

## Stack

- Main thread stack: 32 KB in SRAM1 (grows down from top).
- Sound IRQ stack: 4 KB, separate.
- Lua co-routine stacks: allocated from heap per coroutine (~4 KB min).

Stack overflow triggers a hard fault → firmware crash screen.

## Static

Firmware `.data` + `.bss` sit in DTCM (64 KB). Game code's static
storage (from the ELF's `.data` + `.bss`) lives in SDRAM.

## Executable code

Game C code (`pdex.bin` ELF inside the pdx bundle) is:

1. Loaded from flash storage into a reserved region of SDRAM
   (typically `0xC0100000+`).
2. Fixed-up (dynamic relocations against the runtime's export table).
3. Executed in place; the ARM icache is invalidated via
   `system->clearICache()` before jumping in.
4. On `kEventTerminate` unmapped.

`pdex.bin` is a **position-independent ELF** with `EM_ARM` machine
type, ARMv7-M ISA, hard-float ABI (`EF_ARM_ABI_FLOAT_HARD`). The
runtime resolves `PlaydateAPI*` symbols by pointing the ELF's GOT at
its own vtable pointer before invoking `eventHandler`.

## Cache

Cortex-M7 has:

- L1 I-cache: 16 KB
- L1 D-cache: 16 KB

D-cache is write-through and enabled on SDRAM regions but disabled on
DTCM (which is already single-cycle). The framebuffer sits in cached
SRAM; the LCD DMA reads from the same buffer, so an explicit clean is
needed before pushing (firmware handles this).

I-cache is enabled on both flash and SDRAM. After emitting code (JIT
generation) games **must** call `clearICache()` (2.0+) or the CPU may
execute stale bytes.

## Lua VM budget

Lua allocates through `system->realloc`. GC pressure hits the same
heap as C `malloc` — a game leaking Lua tables reduces C allocation
headroom identically. Practical Lua-only game footprint: ~2 MB for
code + tables, ~1 MB for asset caches loaded via `playdate.graphics`.

## Big-asset streaming

`FilePlayer`, `LCDStreamPlayer`, and `HTTPConnection.read` all operate
in-place: they don't grow with the file size, only with the buffer
size you set. Use these instead of `graphics->loadBitmap` on files >
100 KB when RAM is tight.
