# `.pdv` Encoding Pipeline

How `pdc` converts source video/GIF into a Playdate video file. See
[pdv.md](pdv.md) for the runtime format spec.

## Input formats

`pdc` accepts:

- **GIF** — animated GIF; each GIF frame becomes a video frame.
- **PDV directly** — precompiled by third-party tools.
- **Frame directory** — `movie-frame-0001.png`, `movie-frame-0002.png`
  (community convention; not sure if `pdc` supports natively).

Not accepted directly:
- `.mp4`, `.mov`, `.avi` — must be converted to GIF first (or use a
  third-party PDV encoder).

## Third-party encoder

Panic's SDK only ships GIF → PDV in `pdc`. For higher-quality
video, community tools:

- **jaames/panic-vid** (Node.js): encodes any FFmpeg-readable input
  into PDV.
- **Playdate Video Player Encoder** (Panic legacy tool): pre-SDK
  standalone utility.

Any tool must produce the container documented in [pdv.md](pdv.md).

## GIF → PDV pipeline

`pdc` steps for a `foo.gif`:

1. **Load GIF** using libpng-adjacent GIF decoder (bundled in `pdc`).
2. **For each frame**:
   - Extract 8-bit indexed pixels.
   - Downsample to 400×240 if source is larger (nearest neighbor,
     unless `-m` flag hints otherwise).
   - Convert to 1-bit via **Atkinson dither** (default) or ordered
     Bayer 8×8 (option).
3. **Frame-diff analysis**:
   - Compute per-frame XOR against the previous frame.
   - If diff is < 5% of pixels → mark as delta frame; store XOR.
   - If diff > threshold → mark as key frame; store full bitmap.
   - If diff = 0 → mark as repeat; store empty payload.
4. **Compress each frame** with zlib (level 9) independently.
5. **Assemble** the offset table + frames per the container spec.
6. **Emit** `foo.pdv` with `Playdate VID` header.

## Compression math

Typical size ratios observed:

- Raw 1-bit uncompressed: 400 × 240 / 8 = **12000 bytes/frame**.
- Zlib-compressed key frame: **1500 – 4000 bytes** (depends on
  complexity).
- Zlib-compressed delta frame: **50 – 500 bytes** for slow motion.
- Repeat frame: ~4 bytes header + 0 payload.

For the verified `Setup.pdx/videos/outro.pdv`: 179 frames, ~281 KB
total = 1.6 KB/frame average — dominated by key frames (frequent
scene changes in a menu transition video).

## Frame timing

`pdc` sets `frame_rate` in the header from GIF timing:

- GIF gce (`Graphic Control Extension`) frame delay in centiseconds.
- Averaged across all frames (single fixed rate).
- Clamped to Playdate's supported range (typically 15–60 Hz).

Non-uniform GIF timing is lost — Playdate `.pdv` has a single fixed
frame rate. For variable-rate playback, games must render frames
manually via `renderFrame(vp, n)` and manage timing themselves.

## Dithering choices

**Atkinson** (default): error-diffusion using Bill Atkinson's original
Mac coefficients. Produces "grainy" look with strong edges.

**Bayer 8×8**: ordered dither, produces "checkerboard" mid-tones.
Better for static screenshots than moving video (Atkinson's error
diffuses across frames producing crawling artifacts).

**None**: 50% threshold. Only for stylized/pixel-art source.

`pdc` flag `-d atkinson|bayer|none` selects (unverified — flag inferred
from Panic docs; not in `pdc --help` strings).

## Recommended source

For best Playdate video output:

- **Source**: 2× target resolution (800×480) to allow good downsampling.
- **Content**: high-contrast subject, strong lighting.
- **Frame rate**: 15–30 fps at capture; too many frames blows up file
  size fast.
- **Length**: keep under 30 s for playback smoothness.

## Streaming variant

For `playdate.videostream` (3.0+), the encoder pipeline is the same;
the runtime just doesn't require the whole file in RAM. Frames are
demuxed as they arrive from the transport (file or HTTP).

Audio track:

- Community convention: append a `.pda` after the last frame's data,
  set flag bit 0 in the header's `flags2`.
- Panic's spec allows it but `pdc` doesn't wire GIF audio (GIF has
  none). Manual muxing required.

## Manual muxing

For custom video pipelines:

```
[16B PDV header] [frame_count * 4B offset table] [frame data blocks]
                                                  [(optional) PDA audio blob]
```

Post-encoding, patch `flags2` bit 0 in the header if the audio is
appended.

## Verification tools

- `Simulator.app` opens `.pdv` files directly via drag-in.
- `playdate.graphics.video.new(path)` + `renderFrame` in a test game.

## Panic's own tools output

Panic used a legacy standalone encoder pre-SDK. Its output is
identical to `pdc`'s (same container). Files shipped with Season 1
Playdate games (e.g. `Casual Birder`'s intro cutscene) are compatible
byte-for-byte.
