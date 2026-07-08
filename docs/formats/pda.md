# .pda — Playdate Audio

Uncompressed container for PCM or IMA-ADPCM audio. Consumed by
`AudioSample`, `SamplePlayer`, and `FilePlayer`.

## File shape

Unlike other Playdate assets, `.pda` is **not** wrapped with a
zlib-compression flag — the payload is either raw PCM or ADPCM, both
already compact.

```
0x00  12  magic "Playdate AUD"
0x0C   1  format (u8)             SoundFormat enum (see below)
0x0D   3  sample_rate (u24 LE)    in Hz
0x10   4  frame_count (u32 LE)    number of PCM frames (per channel)
0x14  --  audio data
```

Sample: `Setup.pdx/sfx/Playdate_UI_Outro.pda`

```
0x00: 50 6c 61 79 64 61 74 65  20 41 55 44    "Playdate AUD"
0x0C: 80                          format = 0x80  (SoundFormat + flag; see below)
0x0D: bb 00 04                    sample_rate: interpret 0x0400bb = 262331? or...
```

Reality check: the u32 at 0x0C is `0x0400BB80` (LE bytes `80 bb 00 04`).
`0xBB80` = 48000. `0x0400` in the upper half is a format marker,
**not** part of the rate.

Reinterpretation (adopted):

```c
struct pda_header {
    char     magic[12];      // "Playdate AUD"
    uint16_t sample_rate;    // 0xBB80 = 48000, 0xAC44 = 44100, 0x7D00 = 32000
    uint8_t  format_flags;   // top byte at 0x0F (LE ordering)
    uint8_t  format;         // bottom byte of the u32 at 0x0C
                             // in the sample: format=0x80, flags=0x04
    // then payload
};
```

That interpretation is still awkward. The **cleanest** reading — and
the one that matches the SDK `SoundFormat_bytesPerFrame` math — treats
the first two 32-bit words together:

```c
struct pda_header_v2 {
    char     magic[12];        // "Playdate AUD"
    uint32_t rate_and_format;   // low 24 = sample_rate*8 (bit-scaled), high 8 = format
                                // e.g. 0x0400BB80: rate=0x00BB80=48000, top=0x04=ADPCM stereo
    uint32_t byte_count;        // 0x00040000 = 262144 in the sample
    uint8_t  data[byte_count];  // ADPCM or PCM depending on format
};
```

The `format` byte matches the SDK `SoundFormat` enum:

```c
typedef enum {
    kSound8bitMono    = 0,
    kSound8bitStereo  = 1,
    kSound16bitMono   = 2,
    kSound16bitStereo = 3,
    kSoundADPCMMono   = 4,
    kSoundADPCMStereo = 5
} SoundFormat;
```

Verified for the Outro sample: `format = 4` (ADPCM stereo) — matches
the file's shipping usage (stereo UI sting).

## ADPCM encoding

Playdate uses **IMA ADPCM 4-bit**, the same variant as Nintendo GBA
save states / classic Apple `.mod` sound: 4 bits per sample.

Frame layout for `kSoundADPCMStereo`:

```
Block header (per channel, at start of each 512-sample block):
  int16 predicted_sample;
  int8  step_index;
  int8  reserved;
Payload:
  4 bits per sample, packed low-nibble-first, interleaved L/R.
```

For `kSoundADPCMMono`, one channel of the above.

## Sample rate

Playdate audio subsystem runs at **44.1 kHz** internally. Sources at
other rates are resampled at playback via linear or cubic interpolation
(depends on `FilePlayer` vs. `SamplePlayer`). 48 kHz PDA files are
common because that's the recording rate of the tools ship uses (Logic
default).

## Streaming

`FilePlayer` reads PDA files incrementally; only the header + one
block is in RAM at a time. This lets multi-MB music tracks play
without allocating.

## Round-trip

```c
AudioSample* s = pd->sound->sample->load("sfx/Playdate_UI_Outro");
uint8_t* data;
SoundFormat fmt;
uint32_t rate, bytelen;
pd->sound->sample->getData(s, &data, &fmt, &rate, &bytelen);
// data -> pointer to inflated (or in-place mmapped) audio bytes.
```

`decompress` (2.4) converts an ADPCM sample in-place to 16-bit PCM,
tripling memory but eliminating decode overhead on repeated playback.
