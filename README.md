# footage-skills

**Real recorded footage into finished vertical shorts.**

You recorded yourself. This turns that take into something publishable — cut, captioned,
packaged with graphics, and mixed — without opening a non-linear editor.

The sibling of an AI-imagery video pack: same output format, opposite input. Here every pixel
of the subject is real; only the graphics are authored.

## The pipeline

| Stage | Tool |
|---|---|
| Cut silences and filler words | `ffmpeg` |
| Transcribe with word-level timestamps | any STT that returns per-word timings |
| Captions, overlays, PIP | HyperFrames (HTML → video, one encode) |
| Music bed and SFX | a TTS/music provider + a bundled SFX library |

One transcript feeds the cut list, the captions, the card timings and the graphic cues.
Transcribe once.

### Why this skill exists

Every defect it documents came from a run where **the tooling reported success** — lint clean,
render exit 0, valid output file:

- Timed `<audio>` without an `id` renders **silent**.
- `from({opacity:0})` on an element already at `opacity:0` animates 0 → 0: a panel that
  animates perfectly and is invisible for its entire window.
- Two caption lines alive in one slot print on top of each other; alternating tracks silences
  the warning without fixing the picture.
- A music level copied from published guidance sits **on top of** a raw voice, because that
  guidance assumes a normalized voiceover 10 LU louder.
- A correctly computed volume above 1.0 is **clamped without a warning**.
- An opaque full-frame card hides the speaker during a picture-in-picture move.
- `ffprobe` reports coded dimensions and ignores the rotation matrix, so vertical phone
  footage reads as landscape.
- A filler-word list matched a word the speaker was **quoting**, and cut it out of the sentence.

Two habits catch all of it: **measure numbers with a tool, and look at pixels.**

## The skill

| Skill | Description |
|-------|-------------|
| **footage** | Cut, caption, package and mix a real take into a short — plus the corpus of failures that never throw. |

```
skills/footage/
├── SKILL.md                        the pipeline, the timing floor, the density target
└── reference/
    ├── cut-and-transcript.md       silences, fillers, protected pauses, the language trap
    ├── captions-karaoke.md         windows, spacing, the karaoke mechanic, remapping
    ├── overlays-and-pip.md         zones, tracks, the PIP transform math, endings
    ├── audio-levels.md             measure-then-derive, the two silent clamps
    └── silent-failures.md          ten defects that produce no error
```

## Installation

```
/plugin → Marketplaces → Add Marketplace → milojarow/footage-skills
/plugin → Discover → footage-skills → Install
```

## Requirements

- `ffmpeg` and `ffprobe`
- A speech-to-text provider returning **word-level** timestamps
- HyperFrames for the composition stage

## License

MIT
