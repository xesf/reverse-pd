# MIDI Support

Playdate consumes standard **Standard MIDI Files** (`.mid`, `.midi`)
via `playdate.sound.sequence`. Files are stored as-is in the pdx bundle
— `pdc` does **not** transcode.

## Loading

```c
SoundSequence* seq = pd->sound->sequence->newSequence();
int ok = pd->sound->sequence->loadMIDIFile(seq, "music/theme.mid");
```

Returns 1 on success, 0 on parse failure.

Or via Lua:

```lua
local seq = playdate.sound.sequence.new("music/theme")
seq:play()
```

## Supported SMF features

Verified via SDK example `Examples/MIDIPlayer/`:

- SMF format 0 (single track) and format 1 (multi-track).
- Standard 480-PPQ timing (and other PPQ values via header).
- Note on / note off / running status.
- Program change → maps to instrument index (see below).
- Control change (see supported CC list).
- Tempo change (`FF 51 03 tt tt tt`) — SDK 2.5+ handles float tempo.
- End-of-track (`FF 2F 00`).
- Time signature (`FF 58 04 nn dd cc bb`) — parsed but only affects
  reporting.
- SysEx: skipped (parser walks past but ignores).

## Not supported

- Meta text events (lyrics, marker, cue) — parsed and dropped.
- SysEx-based instrument changes.
- MIDI channel voice messages beyond Note On/Off and Program Change on
  the default instrument.
- SMPTE-based timing (only ticks-per-quarter supported).

## Instrument mapping

By default, each track loaded from an SMF gets a **PDSynthInstrument**
containing one voice: a plain sine `PDSynth`. To use real instruments,
replace after load:

```c
SequenceTrack*    t = pd->sound->sequence->getTrackAtIndex(seq, 0);
PDSynthInstrument* i = pd->sound->instrument->newInstrument();
PDSynth*          s = pd->sound->synth->newSynth();
pd->sound->synth->setWaveform(s, kWaveformSquare);
pd->sound->instrument->addVoice(i, s, 0, 127, 0);   // full-range voice
pd->sound->track->setInstrument(t, i);
```

Or Lua:

```lua
local track = seq:getTrackAtIndex(1)
local inst  = playdate.sound.instrument.new()
local synth = playdate.sound.synth.new()
synth:setWaveform(playdate.sound.kWaveformSquare)
inst:addVoice(synth, 0, 127, 0)
track:setInstrument(inst)
```

MIDI Program Change events **do not** switch instruments automatically —
game code has to listen and swap.

## Control Change (CC) messages

`SequenceTrack` exposes each observed CC as a `ControlSignal`. The
signal timeline is populated at load time from the SMF events.

Look up a controller lane:

```c
ControlSignal* cs = pd->sound->track->getSignalForController(track, 7, 0);
// CC 7 = channel volume
```

The signal returns the "current" value at any point during playback and
can be routed as a modulator into a synth's parameters:

```c
pd->sound->synth->setAmplitudeModulator(synth, (PDSynthSignalValue*)cs);
```

Common CCs (all interpreted only if game wires them up):

| CC | Meaning |
|----|---------|
| 1  | Modulation wheel |
| 7  | Channel volume |
| 10 | Pan |
| 11 | Expression |
| 64 | Sustain pedal |
| 71 | Filter resonance |
| 74 | Filter cutoff |
| 91 | Reverb send |
| 93 | Chorus send |
| 120..127 | Channel mode messages |

Playdate has no built-in "GM" (General MIDI) mapping — the sound engine
is a modular synth; games choose their own wiring.

## Pitch bend

Standard 14-bit pitch bend (E<sub>n</sub> `bb bb`). Applied to the
instrument via:

```c
pd->sound->instrument->setPitchBend(inst, bend);          // -1 .. +1
pd->sound->instrument->setPitchBendRange(inst, halfSteps); // default 2
```

## Tempo

```c
pd->sound->sequence->getTempo(seq);                // 2.5+, float BPM/steps-per-second
pd->sound->sequence->setTempo(seq, stepsPerSecond);
```

SMF tempo events change the sequence tempo mid-play if `setTempo`
isn't called externally.

## Timing

Sequence step = SMF tick. `setTime(seq, step)` seeks. Steps map to
audio-mixer sample time via the current tempo.

## Loop control

```c
pd->sound->sequence->setLoops(seq, loopStart, loopEnd, count);
```

- `count = 0` → loop forever.
- `count = N` → loop N times then stop.
- Both `loopStart` and `loopEnd` in SMF ticks.

## Format compatibility notes

- Multi-track SMF files: each track becomes a `SequenceTrack` at the
  same index. MIDI channel is preserved as `track:getMIDIChannel()`
  (not exposed in current header, but used internally for splits).
- Format 2 SMF (independent tracks): supported but no known
  authoring tool emits these for Playdate.

## Panic's own MIDI tools

- `pdtracker` (community) — a Playdate-side tracker that outputs SMF.
- Standard DAWs (Logic, Ableton, LMMS) export usable SMF.
- The [Pulp editor](https://play.date/pulp) has a built-in
  chiptune editor that outputs SMF-compatible sequences.
