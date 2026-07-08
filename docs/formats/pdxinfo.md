# pdxinfo — bundle manifest

Plain-text UTF-8, one `key=value` per line, LF or CRLF. No quoting, no
comments, no escapes. Whitespace around `=` is **not** trimmed — trailing
whitespace on the value survives into the runtime.

See [pdx.md](pdx.md) for the field catalog.

## Grammar

```
pdxinfo := line ( "\n" | "\r\n" )+
line    := key "=" value
key     := [A-Za-z_][A-Za-z0-9_]*
value   := any UTF-8 up to end-of-line
```

## Real-world example

```
name=Setup
author=Panic Inc.
description=First Launch Setup
bundleID=com.panic.setup
pdxversion=30100
buildtime=836082062
```

## Runtime handling

- `system->getSystemInfo()->pdxversion` returns the `pdxversion` value.
- The bundleID is used as the Data/ directory name: `/Data/<bundleID>/`.
- Unknown keys are ignored (forward-compat).
- Duplicate keys: last one wins.

## Encoding gotchas

- `buildtime` is UNIX epoch **seconds**, stored as decimal ASCII. If
  reproducible builds are desired, override with `SOURCE_DATE_EPOCH`
  before invoking `pdc`.
- `version` is user-visible; `buildNumber` is what the OS compares for
  update prompts.
- `contentWarning` supports `\n` **literally** (as two chars); the
  runtime turns them into newlines when rendering.
