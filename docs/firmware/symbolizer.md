# firmware_symbolizer.py

Small Python helper distributed at `bin/firmware_symbolizer.py`. Turns
a raw crash log into symbolic backtrace using `addr2line`.

## Source (verbatim, 43 lines)

```python
import re
import subprocess
try:
    import click
except ModuleNotFoundError as e:
    print(f"Module not found: {e}, please install the required package using pip.")
    exit(1)

"""
    SETUP:
        1. pip3 install click
        2. make sure arm-none-eabi-addr2line is in your $PATH

    USAGE:
        python3 firmware_symbolizer.py crashlog.txt game.elf

"""


@click.command()
@click.argument("crashlog", type=click.Path(exists=True))
@click.argument("elf",       type=click.Path(exists=True))
def symbolize(crashlog, elf):
    cl_contents = open(crashlog, "r").read()
    cl_blocks   = re.split(r"\n\n", cl_contents)

    for block in cl_blocks:
        matches = re.search(r"lr:([0-9a-f]{8})\s+pc:([0-9a-f]{8})", block)
        if matches:
            print(block, "\n")
            lr = matches.group(1)
            pc = matches.group(2)
            cmd = f"arm-none-eabi-addr2line -f -i -p -e \"{elf}\" '0x{pc}' '0x{lr}'"
            stack = subprocess.check_output(cmd, shell=True).decode("ASCII")
            print(stack)


if __name__ == "__main__":
    symbolize()
```

## Crash log format

The firmware writes crash logs as newline-separated stanzas
(**empty-line separated**). Each stanza contains a `lr:hexhex pc:hexhex`
pair. Between stanzas: register dumps, task name, cause of fault.

Skeleton (observed on device):

```
Task: game_main
Reason: HardFault (BusFault, precise data access, at 0xC0100000)
r0:00000000 r1:c0100000 r2:00000010 r3:0000ffff
r4:20030200 r5:00000001 r6:00000000 r7:c0200048
r8:00000000 r9:00000000 sl:00000000 fp:20030260
ip:00000000 sp:20030200 lr:c010037d pc:c010030a psr:61000000

Backtrace:
lr:c010037d pc:c010030a
lr:c01005a1 pc:c010037c
lr:c0100721 pc:c01005a0
lr:c0100000 pc:c0100720
```

Each `lr:...  pc:...` line pair identifies one stack frame. The
symbolizer just extracts each and pipes both addresses through
`addr2line -f -i -p`, giving:

```
game_hit_wall at src/collision.c:132
 (inlined by) game_move at src/player.c:45
main_loop at src/main.c:88
```

## Bit 0 note

ARMv7-M Thumb instructions have bit 0 of the PC set to `1` to indicate
Thumb state — the symbolizer does **not** clear this bit before passing
to addr2line. Modern addr2line ignores it, so no issue in practice, but
if you get "no such address" errors, try masking `pc & ~1`.

## Enhancement ideas

- Print register dump inline.
- Emit source with `-a` flag for architecture context.
- Handle `armv7em` prefix if you swap toolchains.
- Batch: iterate a directory of crash logs.
