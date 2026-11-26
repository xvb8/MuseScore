# Notation-First Audio Workflow: Project Notes

Nov 25, 2026 · @Karan Krishnan

## Central question

How can traditional notation be used in modern audio-first workflows, especially in orchestral and cinematic composition, and what are the advantages and disadvantages?

Today a composer must pick a starting point. Notation-first gives structural clarity (harmony, voice-leading, counterpoint) but a weaker-sounding render. DAW-first gives a convincing render but loses the clarity that centuries of notation convention provide. The project explores a tool that removes that tradeoff.

**Where notation still wins:** communicating with live players, seeing large-scale structure across many staves, precise pitch and rhythm intent, and delivery requirements (score and parts).

**Where audio-first wins:** client approval via realistic mockups, expressive nuance through CC automation, frame-accurate sync to picture, hybrid electronic layers, and fast iteration.

**Current industry practice:** a hybrid pipeline. Compose in a DAW for mockup fidelity, then notate separately for the session. This costs the composer two parallel representations of one piece.

## The friction and guiding principle

The friction is the forced choice of starting point: notation-first sacrifices render quality, DAW-first sacrifices the structural clarity notation evolved to provide.

**Guiding principle:** expand notation rather than compromise it. For example, a crescendo word becomes a CC curve shown directly in the score, or in an overlay.

**The gap:** no existing tool treats CC curves as first-class notated objects. Dorico converts hairpins and dynamics into CC curves through Expression Maps, but the curves are a derived byproduct. Cubase's Score Editor treats MIDI as the source of truth and notation as a view of it. MuseScore plus a DAW exists only as loose sync (JACK routing or MIDI round-trips) with two separate data models.

## Target user

A solo, notation-literate composer who needs realistic mockups for client approval. The render is the deliverable, not the score.

What this implies:

- **Simpler time model.** The composer controls the whole tempo map, so CC curves can live in musical time (bars and beats) with no conflict against picture sync.
- **Library fidelity is central.** Automatic articulation inference from notation, per-library mapping, and humanization are core features, because "sounds real enough to send" is the test.
- **No collaboration overhead.** One user, one score, no conductor-score versus parts split, no session-copyist engraving rigor.

Other personas considered and set aside for now: session-prep composer, game composer (adaptive cues), and film composer syncing to picture with tempo changes.

## Architecture

MuseScore stays the main app and UI. Tracktion Engine is embedded as a library and replaces MuseScore's playback, and a new glue layer connects them. Searching turned up no integration like this; existing MuseScore-plus-DAW setups are two separate apps synced loosely.

&#91;embedded content: architecture · 3 groups in 1 process\]

Score data flows from the kept MuseScore parts through the new layer into Tracktion, which hosts the sample libraries.

**Licensing.** Tracktion Engine is dual-licensed GPLv3 or commercial, and JUCE, which it uses, is licensed separately and is also dual-licensed. Keeping the whole project GPLv3, matching MuseScore, stays in the free lane. Verify current terms before committing.

**Risk 1: Qt and JUCE in one process.** Run JUCE headless (audio and plugin hosting only) and pump its message loop from Qt's, or run it on its own thread. Prototype this first; it could reshape the plan.

**Risk 2: keeping automation in sync with edits.** Overlay data must be anchored to musical position in the score model, not baked into absolute-time automation, or it drifts after inserting bars or changing time signatures.

## View, staves, and mixer

The score view is pageless like a DAW, but with traditional staves in place of tracks. Dorico's Galley View is prior art for pageless notation; the novel part is folding DAW-style tracks into it.

- **Instrument staves:** one traditional staff per instrument.
- **Effects staff (optional):** uses a new "audio clef" to hold sounds from file or DAW-style sources.
- **Record staff:** same audio clef, for recording directly into the score.
- **Vocals staff:** a generic staff for vocals. Possibly fold into the Record staff, unless built-in take-comping is wanted.
- **Mixer:** add FX and virtual instruments to any staff.
- **Lane overlays:** toggle velocity and CC lanes as graphs over a staff.

**Open: what the audio clef notates.** Options are a labeled clip region on a staff-shaped lane (simplest), a waveform drawn on the staff, or a spectrogram aligned to pitch so a sound-design layer's frequency content can be compared with the orchestral notes around it (a stretch feature). Also undecided: whether clips snap to the bar grid or can float in absolute time, with a per-clip override for hits that must land at an exact second.

**Tempo/rubato lane:** proposed as another overlay using the same mechanism, so tempo is editable as a curve in the same view.

## Dynamics: numbers, hairpins, velocity and CC

**Input.** Standard markings (pp, mf) still work and map to preset numbers. A direct number from 0 to 127 is the preferred, more precise input, shown in a distinct style so overrides are visible at a glance. The word "cresc." is replaced by hairpins, which define a start value, end value, and duration.

**Mapping lives in the expression map.** mf on a solo flute and mf on full brass need different underlying values, so symbol-to-number tables are per instrument and library, not global.

**Velocity and CC are separate lanes.** Velocity is set once at note-on and fixed for the note (attack character, dynamic layer). CC (usually CC1 or CC11) is continuous and can move during a held note. Both stay visible and independently toggleable.

**Hairpin handling (mf to f example).** A hairpin defines one ramp that drives both lanes in the way each can respond:

- CC lane: a continuous curve from mf's value to f's value across the whole span.
- Velocity lane: each note under the hairpin is sampled from the same ramp at its onset.
- Single held note: velocity is fixed at the starting value and CC does all the swell.
- Chord: all notes on one onset share one sampled velocity.
- Rest inside the span: CC keeps ramping by default, with a toggle to pause.
- Lone marking (no hairpin): sets the note's velocity and steps the CC lane to the same value so the lanes do not drift apart.

**Decoupling.** A loud attack with a quiet sustain, or the reverse, must be an easy, intentional action, since real players do both.

**Curve shaping.** Perceived loudness is not linear in CC value, so hairpins default to linear but get a bendable handle (ease-in, ease-out, exponential). Long straight-line swells are a common sign of a fake mockup.

**Point events.** Accents, sforzando, subito piano and fp are spikes or steps, not ramps. They need their own representation on the lanes, designed alongside hairpins.

**Defaults per instrument.** The expression map sets whether velocity affects dynamic layer, attack, or nothing, which CC number is the dynamics lane, and whether CC auto-follows velocity on note entry.

**Open: velocity ramp sampling.** Linear by note index versus linear by time position. Time-based is likely more musically correct for uneven rhythms; not yet decided.

## Articulations and expression maps

Notated articulations route to the right sample patch automatically. A dot over a note, for example, triggers the library's staccato keyswitch.

Libraries switch articulations three different ways: keyswitch notes, CC values, or separate MIDI channels per articulation. Spitfire's UACC is a CC-based scheme (typically CC32) where a given value always means the same articulation across Spitfire libraries, so instruments can be swapped without remapping. It is Spitfire-specific; other vendors such as Sonokinetic use their own systems.

**Design:** reusable, named, shareable expression-map profiles, one per instrument and library combination. Each profile holds:

- Notation-to-articulation routing (dot, slur, tremolo mark, and so on) and the target scheme (keyswitch, CC, or channel).
- Dynamics mapping (symbol-to-number tables, which CC is the dynamics lane).
- Velocity behavior and the default velocity/CC coupling.
- Humanization defaults (micro-timing drift, velocity randomization, round-robin variation).

Automatic inference from notation should be good by default, with the CC overlay as the manual override for anything it gets wrong.

## Layout: beat-proportional, tempo-independent

Horizontal spacing is proportional to duration in beats, not optical engraving spacing, so notation and the CC/velocity lanes share one x-axis. Note width does not change with tempo, matching a DAW piano roll; tempo is shown in its own lane instead.

- **Noteheads:** extended to show relative duration. Flags and filled/open status stay as the exact symbolic cue.
- **Rests:** scale the same way, otherwise lanes misalign.
- **Tuplets:** keep traditional bracket and number as the symbolic marking. Width falls out of the fractional beat duration (a triplet eighth is 1/3 of a quarter).
- **Nested tuplets:** the duration model should store fractions recursively.
- **Long notes:** get genuinely wide, like a DAW region. Horizontal zoom handles this.
- **Print view:** a separate view reverts to optical spacing and collapses numeric overrides to standard dynamics, if a legible score is ever needed for live players.

**Open: base unit.** Width per quarter note (or per smallest supported subdivision) drives the whole layout engine and should be chosen early. Also undecided: whether proportional spacing is always on or a toggleable mode.

## Feature ideas under consideration

- Tempo/rubato lane as an editable curve overlay.
- Humanization controls per staff (micro-timing, velocity randomization, round-robin).
- Hairpin curve-shape handle (linear, ease-in, ease-out, exponential).
- Distinct style for numeric dynamic overrides versus symbolic markings.
- Point-event markers on lanes (accent, sforzando, subito piano, fp).
- Spectrogram view on the audio clef (stretch goal).
- Take-comping on the Record/Vocals staff (only if wanted).
- Performance view versus print view toggle.
- Shareable expression-map profiles per instrument and library.

## Open questions

- [ ] What does the audio clef actually show: clip region, waveform, or spectrogram?
- [ ] Do audio clips snap to the bar grid, float in absolute time, or both with a per-clip override?
- [ ] Is Vocals a separate staff type, or part of Record (with or without take-comping)?
- [ ] Velocity ramp under a hairpin: linear by note index or by time position?
- [ ] Base horizontal unit for proportional spacing (width per quarter note or per smallest subdivision)?
- [ ] Is proportional spacing always on, or a toggleable mode?
- [ ] Are CC overlay data attached per note or phrase, or drawn freely across the timeline?
- [ ] How do CC curves behave when bars or time signatures are inserted or changed later?
- [ ] Final licensing path (all-GPLv3 versus commercial) and what to verify for Tracktion Engine and JUCE?

## Suggested order of work

1. Rough sketch of the core interaction (CC overlay, hairpins, lanes). This informs both the data model and the technical spike. Polished UI design waits.
2. Qt/JUCE feasibility spike: run Tracktion Engine headless inside MuseScore's process and pump its message loop from Qt's. This is the biggest unknown and could reshape the architecture.
3. Data model for CC-overlay objects in `libmscore`, anchored to musical position so edits do not break them.
4. Expression-map format (articulation routing, dynamics mapping, velocity/CC behavior).
5. Polished UI design, once the foundation is settled.
