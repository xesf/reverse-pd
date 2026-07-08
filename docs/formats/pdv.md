# .pdv — Playdate Video

Compressed 1-bit video for `playdate_video`. Each frame is a fully or
delta-compressed 400×240 (typically) bitmap.

## File shape

```
0x00  12  magic "Playdate VID"
0x0C   4  flags (u32 LE)             usually 0 (frames self-compressed)
0x10   4  frame_count (u32 LE)
0x14   4  frame_rate  (f32 LE)       Hz
0x18   2  width  (u16 LE)
0x1A   2  height (u16 LE)
0x1C   4  flags2 (u32 LE)            bit0: has_audio_track (unconfirmed)
0x20  --  frame offset table: frame_count u32 LE entries, each is an
          absolute byte offset into the file where that frame starts
0x??  --  frame data blocks
```

Verified on `Setup.pdx/videos/outro.pdv`:

```
0x0C: 00 00 00 00              flags = 0
0x10: b3 00 00 00              frame_count = 179
0x14: 00 00 f0 41              frame_rate  = 30.0f
0x18: 90 01                    width  = 400
0x1A: f0 00                    height = 240
0x1C: 01 00 00 00              flags2 = 1
0x20: 8a 00 00 00              frame[0] offset = 0x0000008a
0x24: 12 01 00 00              frame[1] offset = 0x00000112
0x28: 9a 01 00 00              frame[2] offset = 0x0000019a
```

Offset gaps between successive entries give per-frame compressed size
(e.g. 0x112 − 0x8a = 0x88 = 136 bytes for frame 0). Final "offset" is
the file end / total size (last entry: `0x001121B8` = 1,122,872).

## Per-frame layout

```c
struct pdv_frame {
    uint8_t  header;        // low 2 bits: type (I/P/duplicate)
                            // upper bits: reserved / mask bit
    uint8_t  reserved[3];
    uint32_t compressed_size;  // matches offset[i+1] - offset[i] - 8
    uint8_t  zlib_stream[compressed_size];
};
```

Frame types (inferred from header nibble):

| Value | Kind | Meaning |
|-------|------|---------|
| 0x00  | Key  | Full 400×240 frame; decoder discards previous state |
| 0x01  | Delta | XOR against the previous frame's decoded bitmap |
| 0x02  | Repeat | Reuse previous frame verbatim (payload usually empty) |

**Key frame** payload after inflate: raw 1-bpp bitmap of
`(width*height)/8` bytes (12000 bytes for 400×240), same layout as
`.pdi`'s image plane (MSB-left, 1=white).

**Delta frame** payload after inflate: same size, XOR into last decoded
frame → new decoded frame.

## Streaming variant (3.0+)

`playdate_videostream` accepts a `SDFile*` or an `HTTPConnection*`. The
streamer needs the initial header + at least the first frame to start;
subsequent frames are pulled as they arrive. The frame offset table
lets it seek if the transport supports it (HTTP range requests, seek
on file).

## Audio track

The `flags2` bit `0x1` in the sample above hints an audio track is
present. When set, a parallel `.pda` stream is embedded at the end of
the file (offset = last frame end). `playdate_videostream` runs a
`FilePlayer` on it, synchronized to the video clock (`frame_rate`).

**Status**: audio-track packing is inferred from `videostream` API
existence (`getFilePlayer` on the stream) and not fully byte-verified.

## API round-trip

```c
LCDVideoPlayer* vp = pd->graphics->video->loadVideo("videos/outro");
int w, h, count, cur;
float rate;
pd->graphics->video->getInfo(vp, &w, &h, &rate, &count, &cur);
// w=400, h=240, rate=30.0, count=179
pd->graphics->video->useScreenContext(vp);
for (int i = 0; i < count; ++i) {
    pd->graphics->video->renderFrame(vp, i);
    pd->graphics->display();
}
```
