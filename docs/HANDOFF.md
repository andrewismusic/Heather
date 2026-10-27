# Heather — Handoff for Claude Code

Oct 17, 2026 · @Andrew Huang

## What Heather is

Heather is a 3×3 chord-pad instrument. What sets it apart is per-note voice-leading glide: every note of a chord slides on its own path to a note of the next chord. When chords differ in size, notes split apart or merge back together instead of ever sounding a unison. The prototype proves the interaction model and the musical logic. Port the logic faithfully and rebuild the audio and UI natively.

What the prototype already settles:

- **The glide engine:** pairing modes, branch and join behaviour, spread, land-on-beat timing, and live re-voicing from the faders.
- **A performer layer:** a style slider that morphs Pad → Stabs → Broken → Arp, a rhythm library of 128 patterns, broken-chord groupings, and an arpeggiator with novel random modes.
- **A generator:** key and mode, Harmony and Complexity controls, locks, and die-slider randomizers with an amount set by how far you drag.
- **A progression builder:** slots that mirror the pads, held blanks, play from any slot, and randomizing with musical rules.
- **A two-layer synth:** four engines, amp and filter ADSR, a pitch envelope and presets.
- **Export:** WAV via offline render and MIDI that includes the playing style.

The reference implementation is one HTML file of about 2,300 lines: `glide-grid.html`. It's the Heather artifact, and it should sit in the repo as `prototype/glide-grid.html`. Its pure functions are the spec. Where this doc and the code disagree, the code shows what was actually built.

## Product direction

Build an iOS/iPadOS app whose engine is an AUv3 from day one, on top of a portable C++ core. The touch app ships first. The same core then becomes a desktop plugin without a rewrite.

- **iOS app + AUv3 together:** the standalone app hosts its own AUv3 extension. That gives you a plugin inside iOS DAWs and hosts (Logic Pro for iPad, GarageBand, AUM, Cubasis) at no extra cost. AUv3 also loads in Logic Pro on Apple-silicon Macs, an early taste of desktop.
- **Two plugin roles:** an instrument (Heather's own synth) and a MIDI generator (AUv3 MIDI processor, type `aumi`) that plays any other synth. Chord generation is the reason producers will want it, so the MIDI role matters as much as the sound.
- **Glide over MIDI needs MPE:** each voice goes on its own MIDI channel with its own pitch bend, so glides, splits and merges reach an external MPE synth. Without MPE, MIDI out falls back to plain chord changes. Design the glide planner's output as per-voice pitch curves so both audio and MPE can use it.
- **Desktop plugin later:** wrap the same core in JUCE for VST3, AU and CLAP on macOS and Windows. Check JUCE's current licence terms before committing.
- **Why not JUCE everywhere:** JUCE would give one codebase, but its touch UI is weaker than SwiftUI, and the touch faders are the product on iOS. Keep native SwiftUI on Apple and use JUCE only as the desktop plugin shell.

The diagram below shows how the pieces fit.

&#91;embedded content: architecture · 3 hosts, 6 core modules, 2 outputs\]

Everything musical lives in the core. Each host adds only a UI and an audio or MIDI wrapper. The glide planner is highlighted because its per-voice pitch curves feed both the synth and MPE output.

## Repo layout and stack

One monorepo, with the core shared by every host and the prototype kept as the reference.

```
heather/
  core/              C++20, no platform dependencies, CMake, unit tests
    theory/          chords, qualities, scales, names, roman numerals
    voicing/         voiceChord, invert, smart inversions, top voice
    glide/           pairing, branch/join planner -> per-voice pitch curves
    performer/       style morph, rhythm patterns, broken groups, arp
    generator/       pad generation, die rolls, progression rules
    sequencer/       clock, progression slots, scheduling, offline render
    synth/           layers, engines, envelopes, filter, reverb
    state/           data model, JSON, undo snapshots, presets
  apple/
    HeatherApp/      SwiftUI app (standalone)
    HeatherAU/       AUv3 extension: instrument + MIDI processor
    Bridge/          Objective-C++ wrapper exposing the core to Swift
  desktop/           JUCE plugin shell (later)
  prototype/         glide-grid.html (reference implementation)
  tests/fixtures/    golden outputs generated from the prototype
  docs/              this handoff, specs, CLAUDE.md
```

| Layer | Choice | Notes |
| --- | --- | --- |
| Core | C++20 + CMake | Pure, deterministic and seedable (every random call goes through an injected RNG) so tests can repeat |
| Tests | Catch2 or GoogleTest | Golden fixtures produced by running the prototype's pure functions in Node |
| Audio (Apple) | AUv3 render block, AVAudioEngine in the standalone app | No allocation, locks or Objective-C messaging on the render thread |
| UI (Apple) | SwiftUI, with a UIKit touch layer for pads and faders | Pads need raw multi-touch with low latency; scope drawn in Canvas or Metal |
| UI → audio | Lock-free single-producer queue + AUParameterTree | Faders and knobs become automatable AU parameters |
| State | Codable JSON, versioned | The same JSON is the AU `fullState` for DAW recall |
| Desktop plugin | JUCE wrapper around `core/` | VST3, AU, CLAP |

## Core engine spec

The engine turns a chord plus five performance values (voicing, inversion, top voice, style, glide) into per-voice pitch curves and note triggers. Prototype function names are given in brackets so each rule can be checked against the code.

### Chords and voicing

- **Chord:** a root pitch class (0–11) plus one of 28 qualities, from triads to 13ths and altered dominants (`QUAL`). Intervals of 12 or more are extensions, voiced above the core tones. Names use sharps or flats by key, and roman numerals are relative to the key.
- **Voicing 0–1** (`voiceChord`): the bass is the root, placed near MIDI 55 − 18×open. The remaining tones stack greedily with a minimum gap of 1 + round(open×11) semitones. The gap shrinks until the top note is at or below 98. Maximum 6 notes.
- **Inversion −6..+6** (`invert`): each step up moves the lowest note up an octave, and each step down moves the top note down an octave, skipping unisons and staying within MIDI 26–102.
- **Smart inversions** (`chordNotes`): try every inversion within 3 steps of the fader and pick the lowest of voice-leading cost from the previous chord ÷ note count + 0.9 × steps away from the fader.
- **Top voice −1..+1** (`topPitch`): aims the top note up to an octave up or down.
  - Candidates are the chord's own tones plus scale tensions: the key's scale if the chord is diatonic, otherwise a scale chosen from the chord's quality (`chordScale`).
  - It excludes tones a semitone above a chord tone, unisons with other voices, and anything at or below the bass.

### Glide planner

- **Pairing** (`groupsFor`, `partitionDP`): sources are the sounding voices in pitch order, targets the new chord's notes.
  - **Nearest:** a monotone partition that minimizes total movement.
  - **Farthest:** the same partition run against targets in reverse order, maximizing movement.
  - **Random:** random groups.
- **Branch (more targets than voices):** a new voice starts at its source's pitch, silent, and fades in over 60% of its glide as it moves away. **Join (fewer targets than voices):** surplus voices glide to a shared target and reach zero level exactly on arrival. A unison never sounds.
- **Timing:** glide time 0–1 s on an x² taper. Spread ±0–100% gives each voice its own duration, chosen evenly at random around that time. **Land on the beat:** each glide starts at T − its duration so all voices arrive on the beat. **Start on the beat:** glides start at T.
- **Quantization:** tapped pads are unquantized. Progression chords land on bar lines.
- **Memory:** with Hold off and "glide every chord", a new chord glides out of the last chord's pitches from silence. With "overlapping taps only", it glides only when the previous pad is still held.
- **Live faders** (`liveShape`): Inversion and Voicing (each set to live or next chord) and Top voice re-aim the sounding voices with their own glide times. Top voice vibrato bends that voice by up to ± the bend range within 18 px of the touch point, before switching to note changes.
- **Finger on a pad:** x/y movement bends the whole chord up to ± the bend range (default 50 cents), or slides between pads when that setting is on.

### Performer

- **Style slider:** morphs between the enabled styles (Pad, Stabs, Broken, Arp, each switchable in settings).
  - **From Pad toward any style:** sets a sustain floor of 1 − f between hits.
  - **Between two non-pad styles:** each tick picks style B when a hash of (tick mod 8 beats) < f, so the mix repeats every two bars instead of shuffling.
  - **Speed:** the style slider is always instant.
- **Rhythm** (`PATS`): all 128 eight-step patterns of 16th notes that start on step 1, ordered by number of hits, then by evenness (variance of the gaps). Each pattern repeats twice per bar.
- **Gate** = a fraction of the gap to the next hit. **Accent:** bar downbeat 1, other quarter notes 1 − 0.2a, everything else 1 − 0.6a. **Swing:** shown as 50–75%; the off-beat delay = swing × 0.5 × step length.
- **Notes filter** (stabs and arp): All, Upper, Outer, Bass, Top.
- **Broken groupings** (`brokenGroups`): Bass / rest, Odd / even, Low / high, Outer / inner, Three groups, Overlapping, Alberti, and Random groups. Random groups stay fixed until the Broken control or the style slider moves.
- **Arp** (`arpStep`): Up, Down, Up-down, Converge, Diverge, Random, Random other (never the same note twice in a row), Random other other (a note can repeat only after two others) and Random loop (a fresh order each chord, then repeated). Rates 1/4 to 1/32 including triplets; 1–3 octaves; keep place or restart on chord change.
- **Hits:** a hit retriggers the voice's envelopes, then releases to the floor after the gate. A newly launched chord always gets a hit on its beat. Releasing a pad with Hold off cancels hits already scheduled.

### Generator and randomizers

- **Pads** (`makeChord`): Key and Mode, Harmony (diatonic → borrowed → secondary dominants → tritone subs and chromatic mediants) and Complexity (triads → extended). Locked pads are skipped.
- **Die amount for pads** (`rollPads`):
  - 97% or more regenerates every unlocked pad.
  - Below that, each pad changes with probability 0.2 + 0.8a.
  - A changing pad becomes a new chord with probability 0.15 + 0.7a; otherwise it swaps to another quality from the same family.
- **Progression** (`rollProg`, `pickProgChord`): only filled, unlocked slots change; the number changed = max(1, round((0.2 + 0.8a) × free slots)). Chords come from the current pads.
  - The same chord back to back is allowed only about 10% of the time.
  - No root note may cover more than half the filled chords.
  - A final repair pass checks the whole loop against both rules.

## Synth spec

Two stackable layers, each with one of four engines. Up to 6 chord notes, plus a pool of 12 voices so splitting and merging notes have room. The prototype's Web Audio DSP is a sketch: keep the parameters and behaviour, but use proper native DSP.

| Engine | Sources per voice | Engine controls |
| --- | --- | --- |
| Analog | 2 oscillators, square sub an octave down, noise | Osc 1 wave, Osc 2 wave, Osc 2 level, detune 0–30 cents, sub, noise |
| Wavetable | Table A and table B crossfaded, sine sub | Table (Soft, Glass, Vowel, Organ, Hollow, Digital), position, position envelope, sub, drift |
| FM | 2-operator, sine modulator | Ratio (0.5 to 7), carrier wave, index 0–10, index envelope |
| Strings | 3 saws (centre, ±spread), saw an octave up, slow LFO on the side saws | Spread 0–40 cents, octave level, motion |

- **Per layer:**
  - Filter: LP, BP or HP, with cutoff 60 Hz–12 kHz, resonance, and envelope amount ±1 (±4 octaves).
  - Amp ADSR and filter ADSR. The filter ADSR also drives the wavetable position envelope and the FM index envelope.
  - Pitch envelope: ±24 semitones, decaying over 5 ms–2 s.
  - Level, fine tune ±50 cents, octave −1, 0 or +1.
  - A layer at zero level is not built at all, so it costs no CPU.
- **Envelope behaviour:** envelopes retrigger on every chord change, starting from their current level so pads don't dip, and on every performer hit. Changing octave or FM ratio re-points the pitch without retriggering; doing it by rebuilding caused glitches in the prototype.
- **Global:**
  - Glide time and spread.
  - Width pans the notes by pitch rank, lowest left to highest right, and updates live.
  - Reverb, bend range (0–100 cents) and volume.
- **Presets:** five per engine. Analog: Mellow pad (the default sound), Warm keys, Bright brass, Soft pluck, Dark drone. Wavetable: Glass pad, Vowel sweep, Organ, Hollow keys, Digital shimmer. FM: E-piano, Bell, Glass keys, Soft FM pad, Metal pluck. Strings: Ensemble, Solina, Slow strings, Bright section, Choir. Values are in `PRESETS`.
- **Layer randomizer:** a Character setting (Any, Pad, Keys, Pluck, Bright, Dark) sets ranges for filter and envelope values (`CHAR`). The die amount blends from the current values toward random ones, with an effective amount of 0.15 + 0.85a.

Native DSP upgrades to make during the port:

- Band-limited oscillators (PolyBLEP or mipmapped wavetables) instead of browser oscillators.
- A zero-delay-feedback state-variable filter instead of the browser's biquad filter.
- A real algorithmic reverb instead of the prototype's convolution with generated noise.
- Envelope curves you control directly. The prototype approaches each target exponentially, with time constants of attack/3, decay/4 and release/4.

## UI and interaction spec

The screen splits in two. The top half is fixed and is where you play. The bottom half is four swipeable panes where you shape things.

### Layout

- **Top, fixed:**
  - A title bar with undo, redo, Hold and settings.
  - A voice-trace scope, about 70 pt tall, that draws each voice's pitch over the last 3.5 s.
  - The 3×3 pads, about 31% of the screen height.
  - A row with the pencil button and the Generate die slider.
- **Bottom panes:** swiped or tapped from a tab strip.
  - **Chords:** Key, Mode, Harmony, Complexity.
  - **Progression:** slot grid, play/stop, clear (with confirmation), Randomize die slider, BPM, Audio and MIDI export.
  - **Play:** vertical Style, Inversion, Voicing and Top voice faders, then Rhythm, Gate, Accent, Swing, Notes, Broken and the arp controls.
  - **Sound:** Layer A/B, preset, engine, filter and envelope knobs, glide and output knobs, Character + Randomize die slider.
- **iPad landscape:** the pads on the left, the panes in a column on the right.

### Gestures

- **Pads:**
  - Tap or hold to play.
  - The lock sits top-right.
  - The active pad glows from the touch point.
  - Degree names can be hidden in settings.
- **Pencil:** tap toggles edit mode, where pad edges glow and a tap opens the pad's editor. Hold the pencil and tap a pad for one quick edit.
  - The editor grows out of the pad into a centred card.
  - It shows the chord name (tap to hear it), the voiced notes, root and quality chips, and Clear.
- **Die slider:**
  - Drag the die right, or tap the track, to roll by that amount.
  - Hold the die and tap a pad or slot to roll just that one. On a pointer device, tap the die to arm it first.
  - Targets glow while the die is in use, and whatever changed flashes.
- **Progression slots:**
  - Assign a chord by holding a pad and tapping a slot, holding a slot and tapping a pad, or dragging a pad into a slot.
  - Drag a slot onto another to swap them, or out of the grid to remove it.
  - Tap a slot to play from it.
  - Empty slots between chords hold the earlier chord and merge into its tile.
  - A new row of 3 appears once the last row fills.
- **Faders:** double-tap resets. The style slider has a detent at each style, and Top voice has one at centre.
- **Knobs:** drag vertically. Knobs for choices (waves, tables, filter type, engine) step through them and show the current one.
- **Haptics (new on iOS):** a tick on fader detents, on knob steps for choices, and when a die roll lands.

### Settings

| Setting | Options | Default |
| --- | --- | --- |
| Colours (Chords, Progression and Play panes / Sound pane) | Ember, Tide, Lilac, Rose, Brass, Frost | Ember / Lilac |
| Styles on the style slider | Pad, Stabs, Broken, Arp (at least one) | All |
| Inversion fader / Voicing fader | Change live, next chord | Live / live |
| Fader glide times | Top voice, Inversion, Voicing (0–1.5 s) | 0.2 s each |
| Top voice fader | Notes, notes + vibrato | Notes |
| Glide pairing | Nearest, random, farthest | Nearest |
| Progression glide timing | Land on the beat, start on the beat | Land |
| Gliding when Hold is off | Every chord, overlapping taps only | Every chord |
| Finger movement on a pad | Vibrato, slide between pads | Vibrato |
| Smart inversions | Off, on | Off |
| Arp & broken pattern on chord change | Keep place, restart | Keep place |
| Degree names / visualizer | Show, hide | Show / show |

**Visual identity:** dark, instrument-like palettes. Type is Bricolage Grotesque for names and Martian Mono for labels and values. An orbiting edge glow marks edit and pick modes. Colour tokens for every palette are in the prototype's CSS.

## Data model, persistence and undo

All musical state is one plain, serialisable object. Keep it that way: it drives saving, undo, presets and AU state recall.

```
Project {
  cells[9]: Chord|null      locks[9]: bool
  prog[n]: Chord|null       progLocks[n]: bool     // n = rows x 3, minimum 9
  key, mode, chroma, complexity, bpm
  voicing, inversion, topVoice, glide, dev
  perf { style, density, swing, gate, accent, notes, broken, arp, rate, oct }
  synth { layers[2] { engine, preset, level, oct, fine, ftype, cutoff, reso, fenv,
                      aA,aD,aS,aR, fA,fD,fS,fR, pAmt, pTime, e{engine params} },
          width, reverb, volume, bend }
}
Prefs { palettes, styles enabled, live inversion/voicing, fader glides, topVib,
        glideMode, landMode, glideAll, slide, smartInv, arpReset, showDegree, showScope }
Chord { r: 0-11, q: quality key }
```

- **Progression slots store chord copies**, not pad references, so re-rolling or editing pads never changes a built progression.
- **Undo and redo:** whole-project snapshots.
  - **What's covered:** everything in `Project`, including the performance faders and glide. Prefs and the current pane are excluded.
  - **When a step is saved:** continuous edits are saved 450 ms after they settle. A discrete action (roll, edit, assign) saves before and after, so each is its own step.
  - **Limit:** 120 steps.
  - **Restoring:** reapplies the sound and, if the sounding pad's chord changed, glides to the new chord.
- **Persistence:** the prototype uses browser storage under versioned keys with migrations. The product needs saved projects, factory and user preset banks per engine, and AU `fullState` holding the project JSON so a DAW reopens exactly as left.

## Export and DAW integration

Exports run the real scheduler offline. That way audio, MIDI and live playback can never drift apart. Keep that design: one sequencer, run against either the live clock or a simulated one.

- **Audio (prototype):** renders the progression twice and keeps the second pass, so the file loops seamlessly, glide back into the first chord included. WAV, 44.1 kHz, 16-bit.
  - **Product:** 48 kHz/24-bit, optional stems per layer, and the iOS share sheet, Files and drag-and-drop on iPad.
  - **Why a ZIP:** the prototype wraps its files in a ZIP only because the claude.ai viewer can't save .wav or .mid. Drop that wrapper.
- **MIDI (prototype):** the same offline run records each performer hit as a note with velocity. Pad style records a held chord per bar. No glide.
  - **Product:** add an MPE export that writes each voice's glide as per-channel pitch bend, alongside the plain version.
- **AUv3 instrument:**
  - Follows host tempo and transport.
  - Exposes faders and knobs as automatable parameters.
  - Takes incoming MIDI notes to trigger pads (for example, nine notes mapped to the nine pads).
  - Saves the project in `fullState`.
- **AUv3 MIDI processor (`aumi`):** sends chords, performer patterns and, optionally, MPE glides to another instrument. This is the producer-facing plugin.
- **Standalone sync:** Ableton Link and MIDI clock in and out, so the iOS app plays alongside other apps and hardware.
- **Desktop (JUCE):** the same instrument and MIDI-effect roles as VST3, AU and CLAP, with drag-to-DAW MIDI clips from the progression pane.

## Known gaps and open questions

The prototype was tested in headless Chromium and by ear in the browser, never on a real touchscreen.

**Gaps to close in the port:**

- **Untested gestures:** the two-handed ones (hold a pad and tap a slot, hold the die and tap a target) and pad vibrato under several fingers at once.
- **Untested CPU load:** two Strings layers across 12 voices on an iPhone.
- **No arp portamento:** consecutive arp notes within one chord don't glide; each arp note is a separate voice. Chord-to-chord glide under the arp does work.
- **Crossings in random and farthest pairing:** voices cross mid-glide and can briefly pass through the same pitch. Decide whether to add a crossing fade.
- **Progression limits:** one bar per slot, 4/4 only, no tempo changes. Removing one slot works only by dragging it out of the grid.
- **Web Audio workarounds not to port:** constant-source nodes for octave and vibrato offsets, envelopes built by cancelling and rescheduling targets, and reading parameter values back for the scope. A native engine computes these directly.

**Questions to settle:**

- [ ] iPhone or iPad as the primary design target? The fixed-top layout was tuned on a 390 × 844 phone.
- [ ] Business model: paid app, subscription, or free with an in-app purchase for AUv3 and exports?
- [ ] Should MPE be the default for MIDI out, with plain MIDI as the fallback?
- [ ] Desktop plugin format priority (VST3, AU, CLAP) and timing relative to the iOS launch.
- [ ] Should the pads in a DAW be triggered by incoming MIDI notes, chord-detected from played chords, or both?
- [ ] Should pad edits ever flow into the progression (it currently holds copies)?
- [ ] Accessibility: VoiceOver labels for pads and faders, plus a non-gesture way to assign slots.

## Milestones and first prompts for Claude Code

Port the logic before the sound, and the sound before the UI. Each step is testable on its own.

1. **Core logic + golden tests:** theory, voicing, glide planner, performer, generator, progression rules in `core/`, checked against fixtures produced from the prototype.
2. **Native synth + offline render:** four engines, envelopes, filter, reverb. A command-line tool that renders a project JSON to WAV and MIDI.
3. **iOS app shell:** fixed top area, panes, pads with multi-touch, faders, knobs, die sliders, progression gestures, undo.
4. **AUv3:** instrument and MIDI processor, parameter tree, host sync, `fullState`, testing in AUM, GarageBand and Logic Pro for iPad.
5. **Export, presets, polish:** WAV, MIDI and MPE export, preset banks, haptics, iPad layout, TestFlight.
6. **Desktop plugin:** a JUCE shell around `core/` for VST3, AU and CLAP.

Put this doc in the repo as `docs/HANDOFF.md` and the prototype as `prototype/glide-grid.html`, then start with these prompts:

```
Read docs/HANDOFF.md and prototype/glide-grid.html. Write a CLAUDE.md that summarises
the architecture (portable C++20 core, SwiftUI app, AUv3 extension, later JUCE shell),
the realtime rules (no allocation or locks on the audio thread), and how to run tests.
```

```
Extract the pure functions from prototype/glide-grid.html (voiceChord, invert,
partitionDP, chordNotes, topPitch, chordScale, PATS, brokenGroups, rollPads,
pickProgChord, rollProg) into a Node script with a seeded RNG. Generate JSON golden
fixtures for a few hundred inputs into tests/fixtures/.
```

```
Create core/ as a CMake C++20 library with Catch2 tests. Port theory/ and voicing/
first and make every golden fixture pass. Then port glide/ so it outputs per-voice
pitch curves (start time, end time, from, to, fade in/out) for a chord change, with
tests for branch, join and the three pairing modes.
```

```
Port performer/ and sequencer/ with an injectable clock. Build heather-render, a CLI
that takes a project JSON and writes WAV and MIDI, matching the prototype's offline
render. Then add synth/ with band-limited oscillators and a ZDF state-variable filter.
```
