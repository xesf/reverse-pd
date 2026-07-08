# playdate_sound — Audio engine

Source: `C_API/pd_api/pd_api_sound.h` (635 lines — largest header).

Playdate audio is a **44.1 kHz stereo bus** driven by a DMA-fed I2S
codec. Everything sums into a chain of `SoundChannel`s → the default
channel → speaker/headset. Chunks are processed in 512-frame cycles
(`AUDIO_FRAMES_PER_CYCLE`); latency budget ≈ 11.6 ms.

## Sample formats

```c
typedef enum {
    kSound8bitMono    = 0,
    kSound8bitStereo  = 1,
    kSound16bitMono   = 2,
    kSound16bitStereo = 3,
    kSoundADPCMMono   = 4,
    kSoundADPCMStereo = 5
} SoundFormat;

#define SoundFormatIsStereo(f) ((f)&1)
#define SoundFormatIs16bit(f)  ((f) >= kSound16bitMono)

static inline uint32_t SoundFormat_bytesPerFrame(SoundFormat f) {
    return (SoundFormatIsStereo(f) ? 2 : 1)
         * (SoundFormatIs16bit(f)  ? 2 : 1);
}
```

ADPCM = IMA-4bit. See [`../formats/pda.md`](../formats/pda.md) for the on-disk container.

## MIDI notes

```c
typedef float MIDINote;
#define NOTE_C4 60
static inline float pd_noteToFrequency(MIDINote n) { return 440.f * powf(2.f, (n-69)/12.f); }
static inline MIDINote pd_frequencyToNote(float f)  { return 12*log2f(f) - 36.376316562f; }
```

Fractional notes allowed (e.g. `60.5` = quarter-tone above C4).

## The class hierarchy

The C API models an OO tree; `void*` casts are legal (documented in the
header). Every player derives from `SoundSource`:

```
SoundSource
├── FilePlayer     — streamed .pda/.mp3
├── SamplePlayer   — in-RAM AudioSample
├── PDSynth        — waveform/sample synth with ADSR
└── DelayLineTap   — read head on a DelayLine
```

`playdate_sound_source` gives the common ops:

```c
struct playdate_sound_source {
    void (*setVolume)(SoundSource*, float lvol, float rvol);
    void (*getVolume)(SoundSource*, float* outl, float* outr);
    int  (*isPlaying)(SoundSource*);
    void (*setFinishCallback)(SoundSource*, sndCallbackProc*, void* ud);
};
```

## Root vtable

```c
struct playdate_sound {
    const struct playdate_sound_channel*      channel;
    const struct playdate_sound_fileplayer*   fileplayer;
    const struct playdate_sound_sample*       sample;
    const struct playdate_sound_sampleplayer* sampleplayer;
    const struct playdate_sound_synth*        synth;
    const struct playdate_sound_sequence*     sequence;
    const struct playdate_sound_effect*       effect;
    const struct playdate_sound_lfo*          lfo;
    const struct playdate_sound_envelope*     envelope;
    const struct playdate_sound_source*       source;
    const struct playdate_control_signal*     controlsignal;
    const struct playdate_sound_track*        track;
    const struct playdate_sound_instrument*   instrument;

    uint32_t     (*getCurrentTime)(void);   // sample counter, wraps ~27 h
    SoundSource* (*addSource)(AudioSourceFunction* cb, void* ctx, int stereo);
    SoundChannel* (*getDefaultChannel)(void);
    int (*addChannel)(SoundChannel*);
    int (*removeChannel)(SoundChannel*);

    int  (*setMicCallback)(RecordCallback*, void* ctx, enum MicSource);
    void (*getHeadphoneState)(int* hp, int* mic, void(*cb)(int hp, int mic));
    void (*setOutputsActive)(int hp, int spk);

    // 1.5
    int (*removeSource)(SoundSource*);
    // 1.12
    const struct playdate_sound_signal* signal;
    // 2.2
    const char* (*getError)(void);
    // 3.1
    enum accessReply (*requestMicAccess)(const char* purpose,
                                         AccessRequestCallback*, void* ud);
};
```

## FilePlayer — streamed playback

Handles files bigger than RAM budget. Streams from flash in chunks of
~16 KB (`setBufferLength`).

```c
struct playdate_sound_fileplayer {
    FilePlayer* (*newPlayer)(void);
    void        (*freePlayer)(FilePlayer*);
    int         (*loadIntoPlayer)(FilePlayer*, const char* path);
    void        (*setBufferLength)(FilePlayer*, float seconds);
    int         (*play)(FilePlayer*, int repeat);      // repeat=0 loop forever
    int         (*isPlaying)(FilePlayer*);
    void        (*pause)(FilePlayer*);
    void        (*stop)(FilePlayer*);
    void        (*setVolume)(FilePlayer*, float l, float r);
    void        (*getVolume)(FilePlayer*, float* l, float* r);
    float       (*getLength)(FilePlayer*);
    void        (*setOffset)(FilePlayer*, float sec);
    void        (*setRate)(FilePlayer*, float rate);   // 1.0 = normal, 2.0 = 2x
    void        (*setLoopRange)(FilePlayer*, float startSec, float endSec);
    int         (*didUnderrun)(FilePlayer*);
    void        (*setFinishCallback)(FilePlayer*, sndCallbackProc*, void*);
    void        (*setLoopCallback)  (FilePlayer*, sndCallbackProc*, void*);
    float       (*getOffset)(FilePlayer*);
    float       (*getRate)  (FilePlayer*);
    void        (*setStopOnUnderrun)(FilePlayer*, int flag);
    void        (*fadeVolume)(FilePlayer*, float l, float r, int32_t frames,
                              sndCallbackProc*, void*);
    void        (*setMP3StreamSource)(FilePlayer*,
                                      int (*ds)(uint8_t* data, int bytes, void* ud),
                                      void* ud, float bufSec);
    // 3.1
    void                (*setRateModulator)(FilePlayer*, PDSynthSignalValue*);
    PDSynthSignalValue* (*getRateModulator)(FilePlayer*);
};
```

`setMP3StreamSource` lets the caller feed compressed MP3 bytes on
demand — used for network radio, procedural playlists.

## AudioSample — in-RAM buffer

```c
struct playdate_sound_sample {
    AudioSample* (*newSampleBuffer)(int byteCount);   // preallocated silence
    int          (*loadIntoSample)(AudioSample*, const char* path);
    AudioSample* (*load)(const char* path);
    AudioSample* (*newSampleFromData)(uint8_t* data, SoundFormat, uint32_t rate,
                                      int nBytes, int freeData);
    void         (*getData)(AudioSample*, uint8_t** data, SoundFormat*,
                            uint32_t* rate, uint32_t* len);
    void         (*freeSample)(AudioSample*);
    float        (*getLength)(AudioSample*);
    // 2.4
    int (*decompress)(AudioSample*);   // .pda ADPCM -> PCM in place
};
```

## SamplePlayer

```c
struct playdate_sound_sampleplayer { // extends SoundSource
    SamplePlayer* (*newPlayer)(void);
    void          (*freePlayer)(SamplePlayer*);
    void          (*setSample)(SamplePlayer*, AudioSample*);
    int           (*play)(SamplePlayer*, int repeat, float rate);
    int           (*isPlaying)(SamplePlayer*);
    void          (*stop)(SamplePlayer*);
    void          (*setVolume)(SamplePlayer*, float l, float r);
    void          (*getVolume)(SamplePlayer*, float* l, float* r);
    float         (*getLength)(SamplePlayer*);
    void          (*setOffset)(SamplePlayer*, float sec);
    void          (*setRate)  (SamplePlayer*, float);
    void          (*setPlayRange)(SamplePlayer*, int startFrame, int endFrame);
    void          (*setFinishCallback)(SamplePlayer*, sndCallbackProc*, void*);
    void          (*setLoopCallback)  (SamplePlayer*, sndCallbackProc*, void*);
    float         (*getOffset)(SamplePlayer*);
    float         (*getRate)  (SamplePlayer*);
    void          (*setPaused)(SamplePlayer*, int);
    // 3.1
    void                (*setRateModulator)(SamplePlayer*, PDSynthSignalValue*);
    PDSynthSignalValue* (*getRateModulator)(SamplePlayer*);
};
```

## Signals — the modulation system

`PDSynthSignalValue` is the base: any object with a "current value" (LFO
output, envelope stage, control curve). `PDSynthSignal` is a subclass for
signals that step themselves each frame.

```c
typedef float (*signalStepFunc)  (void* ud, int* ioframes, float* ifval);
typedef void  (*signalNoteOnFunc)(void* ud, MIDINote note, float vel, float len);
typedef void  (*signalNoteOffFunc)(void* ud, int stopped, int offset);
typedef void  (*signalDeallocFunc)(void* ud);

struct playdate_sound_signal {
    PDSynthSignal* (*newSignal)(signalStepFunc, signalNoteOnFunc,
                                signalNoteOffFunc, signalDeallocFunc, void* ud);
    void  (*freeSignal)(PDSynthSignal*);
    float (*getValue)(PDSynthSignal*);
    void  (*setValueScale)(PDSynthSignal*, float);
    void  (*setValueOffset)(PDSynthSignal*, float);
    // 2.6
    PDSynthSignal* (*newSignalForValue)(PDSynthSignalValue*);
};
```

Any signal can be routed into any `PDSynthSignalValue`-taking slot on
another sound object (e.g. `setFrequencyModulator`).

## LFOs

```c
typedef enum {
    kLFOTypeSquare,
    kLFOTypeTriangle,
    kLFOTypeSine,
    kLFOTypeSampleAndHold,
    kLFOTypeSawtoothUp,
    kLFOTypeSawtoothDown,
    kLFOTypeArpeggiator,
    kLFOTypeFunction
} LFOType;

struct playdate_sound_lfo {
    PDSynthLFO* (*newLFO)(LFOType);
    void        (*freeLFO)(PDSynthLFO*);
    void        (*setType)(PDSynthLFO*, LFOType);
    void        (*setRate)(PDSynthLFO*, float hz);
    void        (*setPhase)(PDSynthLFO*, float);
    void        (*setCenter)(PDSynthLFO*, float);
    void        (*setDepth)(PDSynthLFO*, float);
    void        (*setArpeggiation)(PDSynthLFO*, int nSteps, float* steps);
    void        (*setFunction)(PDSynthLFO*,
                               float (*)(PDSynthLFO*, void* ud), void* ud, int interpolate);
    void        (*setDelay)(PDSynthLFO*, float holdoffSec, float rampSec);
    void        (*setRetrigger)(PDSynthLFO*, int flag);
    float       (*getValue)(PDSynthLFO*);
    // 1.10
    void (*setGlobal)(PDSynthLFO*, int);  // shared phase across voices
    // 2.2
    void (*setStartPhase)(PDSynthLFO*, float);
    // 3.1
    void (*setRandomSeed)(PDSynthLFO*, uint16_t);  // for sample-and-hold repro
};
```

## Envelopes

```c
struct playdate_sound_envelope {
    PDSynthEnvelope* (*newEnvelope)(float a, float d, float s, float r);
    void  (*freeEnvelope)(PDSynthEnvelope*);
    void  (*setAttack) (PDSynthEnvelope*, float);
    void  (*setDecay)  (PDSynthEnvelope*, float);
    void  (*setSustain)(PDSynthEnvelope*, float);
    void  (*setRelease)(PDSynthEnvelope*, float);
    void  (*setLegato) (PDSynthEnvelope*, int);
    void  (*setRetrigger)(PDSynthEnvelope*, int);
    float (*getValue)(PDSynthEnvelope*);
    // 1.13
    void (*setCurvature)         (PDSynthEnvelope*, float amount);
    void (*setVelocitySensitivity)(PDSynthEnvelope*, float velsens);
    void (*setRateScaling)       (PDSynthEnvelope*, float scale,
                                  MIDINote start, MIDINote end);
};
```

## PDSynth

```c
typedef enum {
    kWaveformSquare,
    kWaveformTriangle,
    kWaveformSine,
    kWaveformNoise,
    kWaveformSawtooth,
    kWaveformPOPhase,
    kWaveformPODigital,
    kWaveformPOVosim
} SoundWaveform;
```

`PO*` = Teenage Engineering Pocket Operator-style algorithmic
waveforms.

```c
// Custom generator callback (samples in Q8.24, phase in Q0.32)
typedef int  (*synthRenderFunc)(void* ud, int32_t* left, int32_t* right,
                                int nsamples, uint32_t rate, int32_t drate);
typedef void (*synthNoteOnFunc)(void* ud, MIDINote n, float vel, float len);
typedef void (*synthReleaseFunc)(void* ud, int stop);
typedef int  (*synthSetParameterFunc)(void* ud, int p, float v);
typedef void (*synthDeallocFunc)(void* ud);
typedef void*(*synthCopyUserdata)(void* ud);   // 2.4, for synth->copy()

struct playdate_sound_synth {   // extends SoundSource
    PDSynth* (*newSynth)(void);
    void     (*freeSynth)(PDSynth*);
    void     (*setWaveform)(PDSynth*, SoundWaveform);
    void     (*setGenerator_deprecated)(PDSynth*, int stereo,
                                        synthRenderFunc, synthNoteOnFunc,
                                        synthReleaseFunc, synthSetParameterFunc,
                                        synthDeallocFunc, void* ud);
    void     (*setSample)(PDSynth*, AudioSample*, uint32_t sStart, uint32_t sEnd);
    void     (*setAttackTime) (PDSynth*, float);
    void     (*setDecayTime)  (PDSynth*, float);
    void     (*setSustainLevel)(PDSynth*, float);
    void     (*setReleaseTime)(PDSynth*, float);
    void     (*setTranspose)  (PDSynth*, float halfSteps);
    void     (*setFrequencyModulator)(PDSynth*, PDSynthSignalValue*);
    PDSynthSignalValue* (*getFrequencyModulator)(PDSynth*);
    void     (*setAmplitudeModulator)(PDSynth*, PDSynthSignalValue*);
    PDSynthSignalValue* (*getAmplitudeModulator)(PDSynth*);
    int      (*getParameterCount)(PDSynth*);
    int      (*setParameter)(PDSynth*, int p, float);
    void     (*setParameterModulator)(PDSynth*, int p, PDSynthSignalValue*);
    PDSynthSignalValue* (*getParameterModulator)(PDSynth*, int p);
    void     (*playNote)    (PDSynth*, float freqHz, float vel, float len, uint32_t when);
    void     (*playMIDINote)(PDSynth*, MIDINote,     float vel, float len, uint32_t when);
    void     (*noteOff)     (PDSynth*, uint32_t when);
    void     (*stop)        (PDSynth*);              // immediate
    void     (*setVolume)   (PDSynth*, float l, float r);
    void     (*getVolume)   (PDSynth*, float* l, float* r);
    int      (*isPlaying)   (PDSynth*);
    // 1.13
    PDSynthEnvelope* (*getEnvelope)(PDSynth*);       // owned by synth, don't free
    // 2.2
    int  (*setWavetable)(PDSynth*, AudioSample*, int log2size, int cols, int rows);
    // 2.4
    void     (*setGenerator)(PDSynth*, int stereo, synthRenderFunc,
                             synthNoteOnFunc, synthReleaseFunc, synthSetParameterFunc,
                             synthDeallocFunc, synthCopyUserdata, void* ud);
    PDSynth* (*copy)(PDSynth*);
    // 2.6
    void (*clearEnvelope)(PDSynth*);
};
```

`when` is a future timestamp in samples (see
`sound->getCurrentTime()`); `0` = ASAP. `len == -1` = play until
`noteOff`.

## Sequences (piano-roll)

```c
struct playdate_control_signal { ... };  // per-parameter automation lane
struct playdate_sound_instrument { ... }; // bank of PDSynth voices with keysplits
struct playdate_sound_track { ... };      // notes + control lanes for one instrument
struct playdate_sound_sequence { ... };   // multi-track song, tempo, loops
```

Notable:
```c
int (*loadMIDIFile)(SoundSequence*, const char* path);
```
Standard MIDI File (SMF) support — imported as separate tracks per MIDI
channel, each with a default sine `PDSynth` instrument (change with
`setInstrument`).

## Effects

Common shape: `SoundEffect` base + custom `effectProc` for user effects,
prebuilt subclasses live in the same header.

```c
typedef int effectProc(SoundEffect*, int32_t* l, int32_t* r,
                       int nsamples, int bufactive);  // Q8.24

struct playdate_sound_effect {
    SoundEffect* (*newEffect)(effectProc*, void* ud);
    void         (*freeEffect)(SoundEffect*);
    void         (*setMix)(SoundEffect*, float);
    void         (*setMixModulator)(SoundEffect*, PDSynthSignalValue*);
    PDSynthSignalValue* (*getMixModulator)(SoundEffect*);
    void  (*setUserdata)(SoundEffect*, void*);
    void* (*getUserdata)(SoundEffect*);

    const struct playdate_sound_effect_twopolefilter*  twopolefilter;
    const struct playdate_sound_effect_onepolefilter*  onepolefilter;
    const struct playdate_sound_effect_bitcrusher*     bitcrusher;
    const struct playdate_sound_effect_ringmodulator*  ringmodulator;
    const struct playdate_sound_effect_delayline*      delayline;
    const struct playdate_sound_effect_overdrive*      overdrive;
};
```

Filter types:
```c
typedef enum {
    kFilterTypeLowPass, kFilterTypeHighPass, kFilterTypeBandPass,
    kFilterTypeNotch,   kFilterTypePEQ,
    kFilterTypeLowShelf, kFilterTypeHighShelf
} TwoPoleFilterType;
```

BitCrusher gained an **exponential** mode in 3.1
(`setExponential(true)`) plus new `setDepth`/`setDownsampling` params
that replace the older `setAmount`/`setUndersampling` with better curves.

## Sound channels

```c
struct playdate_sound_channel {
    SoundChannel* (*newChannel)(void);
    void          (*freeChannel)(SoundChannel*);
    int           (*addSource)     (SoundChannel*, SoundSource*);
    int           (*removeSource)  (SoundChannel*, SoundSource*);
    SoundSource*  (*addCallbackSource)(SoundChannel*, AudioSourceFunction*,
                                       void* ctx, int stereo);
    int           (*addEffect)     (SoundChannel*, SoundEffect*);
    int           (*removeEffect)  (SoundChannel*, SoundEffect*);
    void          (*setVolume)     (SoundChannel*, float);
    float         (*getVolume)     (SoundChannel*);
    void          (*setVolumeModulator)(SoundChannel*, PDSynthSignalValue*);
    PDSynthSignalValue* (*getVolumeModulator)(SoundChannel*);
    void          (*setPan)        (SoundChannel*, float);   // -1..+1
    void          (*setPanModulator)(SoundChannel*, PDSynthSignalValue*);
    PDSynthSignalValue* (*getPanModulator)(SoundChannel*);
    PDSynthSignalValue* (*getDryLevelSignal)(SoundChannel*);
    PDSynthSignalValue* (*getWetLevelSignal)(SoundChannel*);
    SoundSource*        (*getOutputAsSource)(SoundChannel*);
};
```

`getOutputAsSource` is a nice trick: it lets you feed a channel's mixed
output into another channel's effect chain, so you can bus-route.

## Microphone

```c
typedef int RecordCallback(void* ctx, int16_t* buf, int len);  // mono 16-bit

enum MicSource {
    kMicInputAutodetect = 0,   // headset if plugged, else internal
    kMicInputInternal   = 1,
    kMicInputHeadset    = 2
};
```

`setMicCallback` starts capture; the callback gets 512-sample buffers.
3.1 requires `requestMicAccess` for user consent (mirrors HTTP access
prompt UX).
