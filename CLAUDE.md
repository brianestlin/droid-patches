# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

DROID patches for the [DROID modular synthesizer system](https://der-mann-mit-der-maschine.de/droid/) by Der Mann mit der Maschine. Patches are `.ini` files deployed to the DROID master module via SD card.

There is no build step, linter, or test runner. Development is: edit `.ini` → copy to SD card → test in hardware.

## DROID Patch Format

Each `.ini` file is a patch. The structure is:
1. **Header comments** documenting inputs, outputs, and controller assignments
2. **Hardware declarations** (one line each, e.g. `[p2b8]`, `[e4]`)
3. **Circuit definitions** — named blocks like `[math]`, `[compare]`, `[buttongroup]`, etc., with key/value parameters

Internal signals use `_SCREAMING_SNAKE_CASE`. Hardware I/O uses:
- `I1`–`I8`: physical inputs
- `O1`–`O8`: physical outputs
- `N1`–`N8`: input normalizations (value when input is unpatched)
- `P<controller>.<number>`: pots (e.g. `P1.1`)
- `B<controller>.<number>`: buttons
- `L<controller>.<number>`: LEDs
- `R1`–`R8`: register outputs (for debugging)

## Voltage and Unit Conventions

DROID represents voltages internally as `voltage / 10`. So:
- 1V = `0.1`, 5V = `0.5`, 10V = `1.0`
- The literal `1V` in expressions evaluates to `0.1`
- Standard v/oct pitch: 0V = C0, 1V = C1, etc. (0.0–1.0 internally for C0–C9)
- Pitch class (0–11 for C–B): extracted via `floor(pitch * 120) % 12`
- DROID `[chord]` circuit `root` parameter takes a pitch class integer 0–11

## DROID language documentation 
- Documentation can be found in file "droid-manual-blue-6.pdf".

## Hardware Modules in Use

- **P2B8**: 2 full-size pots + 8 buttons with LEDs (controller `1` or `2`, etc.)
- **P10**: 10 trimpots
- **P8S8**: 8 pots + 8 sliders
- **E4**: 4 encoders with push-button and LED ring

## Recurring Idioms

**Detecting whether an input is patched** (normalization trick):
```ini
[copy]
    input = -1
    output = N2          # sets unpatched value of I2 to -1

[compare]
    input = I2
    compare = -1
    ifequal = <fallback> # used when unpatched
    else = I2            # used when patched
    output = _RESULT
```

**Sample-and-hold during "tie" gate:**
```ini
[sample]
    input = I5
    sample = I2
    output = _SAMPLED_ROOT
```

**Selecting from 10 chord types via a 0–1.0 CV:**
```ini
[math]
    input1 = _CV * 10
    floor = _CHORDTYPE   # yields integer 0–9

[switch]
    input1 = 10  # dim7 (DROID degree 10)
    input2 = 23  # ...
    ...
    offset = _CHORDTYPE
    output1 = _DEGREE
```

**Extrapress toggles** (long-press a buttongroup button to cycle a sub-mode):
```ini
[switch]
    input1 = 1
    input2 = 0.4
    forward = _EXTRAPRESS * _B1_SELECTED
    output1 = _B1_LED_LEVEL
```

## Harmony Generator Architecture

The `harmony-generator-*` family shares a common signal flow:

```
Melody CV (I1)
  → [minifonion] (scale-aware quantizer, uses _ROOTSEQ + _DEGREESEQ)
  → pitch class extraction (× 120, round, mod 12 → _MEL_INPUT_PITCH)

Root CV / E4 encoder
  → _ROOT_INPUT_PITCH (pitch class 0–11)

Chord CV / E4 encoder
  → _CHORDTYPE (0–9) → [switch] → _DEGREESEQ (DROID degree number)

[chord] circuit (_ROOTSEQ, _DEGREESEQ)
  → pitch classes of chord tones: _ROOT_PITCH, _THIRD_PITCH, _FIFTH_PITCH, _SEVENTH_PITCH
  → semitone deltas between adjacent chord tones

[multicompare] _MEL_INPUT_PITCH vs chord tone pitch classes
  → _MATCHING_INDEX: which chord tone the melody note is (or -1 if non-chord-tone)

If chord tone:  use _CHORDTONE_*_DELTA (sampled from last chord-tone event)
If non-chord-tone: use _ALTERNATIVE_*_DELTA (selected by algorithm buttongroup)

Algorithm buttongroup (B1.1–B1.8):
  1. Dim       — stacked minor 3rds down (-3, -3, -3) or dim-maj7 (-5, -3, -3)
  2. Rand      — random chord, new on each note
  3. Other     — fixed interval sets {-3,-2,-4} or {-2,-2,-5}
  4. RandLock  — random chord, held until button re-pressed
  5. Para Pass — same deltas as last chord tone
  6. Fuzzy     — para pass ± random ±1 semitone per voice
  7. Diatonic  — diatonic harmony for scale extensions, para pass for others
  8. Fixed/Uni — fixed: melody = chord root; unison: all voices in unison

Output: melody pitch ± deltas → v/oct on O1–O4
        chord tones direct → O5–O8
        optional "drop 2" / "drop 2-4" voicing via P1.1
```

The `_ROOTSEQ` signal is overridden in "fixed" mode (harmonizer = 7) to use `_MEL_INPUT_PITCH` instead of `_ROOT_INPUT_PITCH`.

## Patch Variants

| File | Root/Chord source | Notes |
|------|-------------------|-------|
| `harmony-generator-v1.ini` | CV on I5 (v/oct), I6 (0–1V) | Base version |
| `harmony-generator-v1-10v.ini` | CV on I5 (0–10V), I6 (0–10V) | Same math, different expected CV range |
| `harmony-generator-v1-E4s.ini` | E4 encoders; I5/I6 override if patched | Two `[e4]` modules declared |
| `harmony-generator-seq.ini` | 8-step sequencer via P10 trimpots | Advance/reset via I4/I8 or B1.1/B1.2 |
| `harmony-generator-seq-p8s8-chordcvs.ini` | 8-step sequencer via two P8S8 | Root pots on P2, chord pots on P3 |
| `harmony-generator-v1-E4s-seq.ini` | 4-step sequencer via two E4s | E4-1 encoders=roots, E4-2 encoders=chord types; push enc4=advance, push enc8=reset; I5/I6 CV override when patched |

The sequencer variants use DROID's `[sequencer]` circuit with `pitch1`–`pitch8` for root and `cv1`–`cv8` for chord type, advanced by a clock signal.
