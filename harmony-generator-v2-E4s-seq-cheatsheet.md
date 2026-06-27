# DROID Patch Cheatsheet — harmony-generator-v2-E4s-seq

A dual-mode patch: a **Harmonizer** and a 4-channel **Quantizer** that share one 4-step root/chord sequence and one note-density control.

**Toggle modes by pushing Encoder 1.** Encoder 1's ring is lit in Quantizer mode. The mode persists across power cycles.

- **Harmonizer mode:** the melody on I1 is harmonized into 3 extra voices (O2–O4).
- **Quantizer mode:** 4 independent melodies on I1–I4 are each quantized to the shared scale/chord, output on O1–O4.

---

## Inputs

| Jack | Harmonizer mode | Quantizer mode |
|------|-----------------|----------------|
| I1 | Melody CV in | Channel 1 melody in |
| I2 | Tie gate (holds chord while high) | Channel 2 melody in |
| I3 | — (ignored) | Channel 3 melody in |
| I4 | — (ignored) | Channel 4 melody in |
| I5 | Advance chord sequencer | Advance chord sequencer |
| I6 | Reset chord sequencer | Reset chord sequencer |
| I7 | Root CV override (overrides sequencer when patched) | Same |
| I8 | Chord-type CV override (overrides sequencer when patched) | Same |

---

## Outputs

| Jack | Harmonizer mode | Quantizer mode |
|------|-----------------|----------------|
| O1 | Quantized melody (1V/oct) | Quantized channel 1 |
| O2 | Harmony voice 1 | Quantized channel 2 |
| O3 | Harmony voice 2 | Quantized channel 3 |
| O4 | Harmony voice 3 | Quantized channel 4 |
| O5 | Underlying chord root | Same |
| O6 | Underlying chord third | Same |
| O7 | Underlying chord fifth | Same |
| O8 | Underlying chord seventh | Same |

O1 is identical in both modes. O5–O8 expose the raw chord for layering (pad, bass) in either mode.

---

## Controls

### Controller 1 — P2B8 (2 pots + 8 buttons)

| Control | Function |
|---------|----------|
| P1.1 | Voicing: close / drop-2 / drop-2-4 (harmonizer mode only) |
| P1.2 | Density via harmonicshift: full chromatic down to root only (both modes) |
| B1.1–B1.8 | Harmony algorithm select (harmonizer mode only; buttons are inert and dark in quantizer mode) |

### Controller 2 — E4 #1 (first encoder bank)

| Control | Function |
|---------|----------|
| Encoders 1–4 | Root note for sequence steps 1–4 |
| Push Encoder 1 | Toggle Harmonizer / Quantizer (ring lit = quantizer) |
| Push Encoder 4 | Manual advance (same as I5) |

### Controller 3 — E4 #2 (second encoder bank)

| Control | Function |
|---------|----------|
| Encoders 5–8 | Chord type for sequence steps 1–4 |
| Push Encoder 8 | Manual reset (same as I6) |

---

## Harmony Algorithms (B1.1–B1.8, harmonizer mode)

Selects how non-chord-tones are harmonized. Chord tones always use the chord's own intervals.

| # | Button | Behavior |
|---|--------|----------|
| 1 | Dim | Stacked minor-3rds down (dim7). Long-press → dim maj7 |
| 2 | Rand | New random chord on every note |
| 3 | Other | Fixed interval set −3,−2,−4. Long-press → variant −2,−2,−5 |
| 4 | RandLock | One random chord held on all non-chord-tones; re-press for a new one |
| 5 | Para Pass | Parallel passing — reuse the last chord-tone's intervals |
| 6 | Fuzzy | Para pass, ±1 random semitone per voice |
| 7 | Diatonic | Diatonic harmony for scale tones; para pass for the rest |
| 8 | Fixed/Uni | Fixed: melody note = chord root. Long-press → Unison (all voices = melody) |

**Long-press (extrapress) sub-modes:** B1.1 (dim vs dim maj7), B1.3 (the two "Other" variants), B1.8 (fixed vs unison). A half-lit LED indicates the alternate sub-mode.

---

## Chord Types (Encoders 5–8 / I8)

Each step's chord type selects one of 10 as the encoder sweeps its range:

| Value | Chord | Scale |
|-------|-------|-------|
| 0 | dim7 | whole-half diminished |
| 1 | min7♭5 | locrian |
| 2 | min7 | dorian |
| 3 | sus7♭9 | mixolydian sus4 ♭9 |
| 4 | sus7 | mixolydian sus4 |
| 5 | 7♭9 | mixolydian ♭9 |
| 6 | 7 | mixolydian |
| 7 | maj7 | lydian |
| 8 | altered | altered |
| 9 | maj7♯5 | maj7♯5 |

---

## Chord Sequencer

- 4 steps. Roots come from Encoders 1–4, chord types from Encoders 5–8.
- Advance: I5 or push Encoder 4. Reset to step 1: I6 or push Encoder 8.
- Step indication: the current step is bright on the E4 LED rings, others dim. Encoder 1's ring shows mode instead of step 1, so read step 1 from E4 #2 (Encoder 5's ring).
- Patch I7 (root) or I8 (chord) to override the sequencer with external CV.

---

## Density Control (P1.2)

P1.2 drives the minifonion `harmonicshift` (0–7), which only ever *removes* notes. The patch enables all 12 chromatic notes (7 scale degrees + 5 fill notes), so at the CCW end nothing is filtered and harmonicshift thins from there as you turn CW. Applies in both modes, across all voices/channels:

| Pot position | harmonicshift | Result |
|--------------|:-------------:|--------|
| Fully CCW | 0 | **Full chromatic** — all 12 notes (no quantizing) |
| Just off CCW | 1 | **Diatonic scale (7 notes)** — drops the non-scale "fill" notes |
| | 2 | Also drop 11th |
| | 3 | Also drop 13th |
| | 4 | Also drop 9th → 7th chord (root/3/5/7) |
| | 5 | Also drop 7th → triad |
| | 6 | Also drop 3rd → root + 5th |
| Fully CW | 7 | **Root only** |

Removal order as you turn CW: fill (non-scale) notes → 11th → 13th → 9th → 7th → 3rd → 5th. The root is the last note to go.

> **Note:** the scale set by the chord type begins at harmonicshift **1**, *not* fully CCW. At fully CCW every chromatic note passes — there is no scale filtering at all.

---

## Notes & Quirks

- **Tie is harmonizer-only.** In quantizer mode I2 becomes channel 2 and the tie sample-and-hold is bypassed.
- **Unpatched channel 2:** because I2 normalizes to −1 for tie detection, an unpatched I2 in quantizer mode quantizes to a low note on O2. Harmless if O2 is unused.
- **Buttons frozen in quantizer mode:** the algorithm buttons and their long-press sub-modes can't be changed while quantizing, so flipping back to harmonizer restores your exact settings. Advance, reset, and mode encoder-pushes stay active.
- Known limitation (per patch TODO): tie currently also freezes the chord outputs O5–O8; ideally those would move immediately.
