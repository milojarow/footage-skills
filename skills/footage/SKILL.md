---
name: footage
description: Use when turning real recorded footage — a phone selfie take, a webcam recording, a screen capture, an interview — into a finished vertical short. Use when the user wants dead air or filler words cut out of a take, word-synced karaoke captions burned in, graphic overlays and lower-thirds timed to what is being said, a picture-in-picture moment where a terminal or graphic owns the frame, or a music bed and SFX mixed under a voice. Also use when a rendered video comes out with silent audio, captions drawn on top of each other, a bed louder than the speaker, a graphic that covers the person, or a cut that clips the last consonant of a word. Not for videos whose visuals are AI-generated images — that is a different pipeline.
---

# Footage — real takes into finished shorts

You recorded yourself. This turns that take into something publishable: cut, captioned,
packaged with graphics, and mixed — without opening a non-linear editor.

> **🎬 ACTIVE-SKILL MARKER:** While `footage` is active, begin every reply with 🎬 so the operator sees at a glance that this skill is engaged. Do not omit it.

## Overview

Four tools, each doing only what it is best at:

| Stage | Tool | Why this one |
|---|---|---|
| **Cut** | `ffmpeg` | Destructive edits on real pixels. Frame-accurate trim + concat |
| **Hear** | Speech-to-text with **word-level timestamps** | Every downstream stage is keyed to word timings |
| **Draw** | HyperFrames | HTML→video. Captions, overlays, PIP live in one composition, one encode |
| **Sound** | A music/SFX source — generated, catalogued, or a bundled library | Bed and one-shots under the voice |

**The through-line: one transcript feeds everything.** The cut list, the captions, the card
timings and the graphic cues all derive from the same word array. Transcribe once.

**The failure mode that defines this work: nothing throws.** A composition can lint clean,
render to completion, and still ship a silent audio track, an empty panel, or a bed louder
than the speaker. Every defect in [reference/silent-failures.md](reference/silent-failures.md)
came from a run where the tooling reported success. **Measure and look; do not assume.**

## When to use

- A recorded take (phone, webcam, screen capture, interview) that needs to become a short.
- Cutting silences and filler words out of a take without hand-scrubbing a timeline.
- Word-synced captions — karaoke, one-word-at-a-time, or plain blocks.
- Graphic overlays, lower-thirds, callouts, or a stat card timed to the speech.
- A PIP moment: the speaker shrinks to a corner and a terminal or graphic takes the frame.
- Mixing a music bed and SFX under a voice without drowning it.
- Debugging a render that "worked" but looks or sounds wrong.

**Not for:** videos built from AI-generated imagery (different pipeline), live streaming, or
non-linear editing tasks like colour grading and multi-camera sync.

## The pipeline

Run it in this order. Each stage's output is the next stage's input, and every gate below
exists because skipping it cost a re-render.

### 1 — Probe the source before anything

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,r_frame_rate:stream_tags=rotate:side_data=rotation \
  -show_entries format=duration -of default=nw=1 input.mp4
```

⚠️ **`stream=width,height` reports CODED dimensions and ignores the rotation matrix.** Phone
footage is routinely stored 1920×1080 with `rotation=90` and displays as 1080×1920. Read the
**decoded** frame, not the container fields:

```bash
ffmpeg -v error -i input.mp4 -frames:v 1 -y /tmp/probe.png   # then check the PNG's real size
```

Getting this wrong means designing a whole composition for the wrong aspect ratio.

**A take reported as "no audio" gets measured, not re-shot.** An existing audio stream proves
nothing was captured, and buried signal, a quiet room and a gate that erased the voice look
identical in a player — only one of the three is recoverable. Triage:
[reference/audio-levels.md](reference/audio-levels.md).

### 2 — Transcribe with word-level timestamps

Ask the STT provider for **filler words included**. This reads backwards and matters:
most providers strip "um"/"uh" by default, which gives a clean transcript for free — but
absent fillers have **no timestamps**, and without timestamps they cannot be cut from the
video. Request them, cut on their spans, and filter the text yourself.

Detail, and the language trap that comes with it: [reference/cut-and-transcript.md](reference/cut-and-transcript.md).

### 3 — Build the cut list, then cut

Silences are gaps between kept words. Fillers are words on a list. Both become spans to drop;
what remains becomes spans to keep, joined in **one** `filter_complex` so video and audio are
cut by the same graph and cannot drift.

**Always dry-run first.** Print the plan and read it. Automatic cutting on human speech
occasionally clips the tail of a word, and re-rendering costs more than reading a table.

**Deliberate pauses must be declared.** A gap detector cannot tell a dramatic beat from dead
air — the pause before a punchline is usually the *longest* silence in the take and the first
thing an aggressive threshold eats. See [reference/cut-and-transcript.md](reference/cut-and-transcript.md).

**Look at the frames of every span you are about to delete.** The detector hears; it does not
see. A silent tail is often a brand outro and a silent head is often an opening transition —
both get cut with no error at all. Contact sheet first: [reference/cut-and-transcript.md](reference/cut-and-transcript.md).

### 4 — Remap every timestamp onto the cut timeline

Once frames are removed, the original word timings are wrong. Every caption, card and cue
must be recomputed against the cut, or the whole package drifts against the audio.

### 5 — Compose: captions, overlays, PIP

Captions, graphics and the video live in **one** composition and **one** encode. Burning
captions in a second pass costs a second generation of compression and, worse, is blind to
what the first pass already put on screen.

Layout, zones, the PIP transform math and the collision rules: [reference/overlays-and-pip.md](reference/overlays-and-pip.md).
Caption windows, word spacing and the karaoke mechanic: [reference/captions-karaoke.md](reference/captions-karaoke.md).

### 6 — Source the sound, then mix by measurement

Two kinds of asset, sourced differently:

- **Bed** — one instrumental track, slightly longer than the piece so the composition can trim
  the tail rather than run out. Generating one from a mood prompt costs cents and beats hunting
  a catalogue; ask for **instrumental only**, since vocals fight the voice.
- **One-shots** — transitions, impacts, ticks. A curated library beats generation here: you want
  a whoosh that sounds like every other whoosh in the piece. Composition frameworks often ship a
  small bundled set; if not, any royalty-free pack works.

Whatever the source, **normalize every asset to a common loudness floor before computing any
gain** — see below.

### Mix by measurement

Never pick audio levels by feel. Measure the voice, measure each asset, and **derive** every
gain from the difference. A tutorial default assumes a normalized voiceover; a phone recording
can sit 10 LU below that, and the same "safe" music level then plays *over* the speaker.

The arithmetic, and the clamp that silently ruins a mix: [reference/audio-levels.md](reference/audio-levels.md).

### 7 — Review the render, frame by frame

Sample the finished file — roughly one frame every 3 seconds — and look at every one.
Lint and exit code 0 do not mean the frame is right.

```bash
for t in 1.5 4.5 8.0 11.0 14.5; do
  ffmpeg -y -v error -ss $t -i out.mp4 -frames:v 1 -vf scale=250:-2 f$t.jpg
done
ffmpeg -y -i f1.5.jpg -i f4.5.jpg ... -filter_complex hstack=inputs=N sheet.jpg
```

A contact sheet of 20 frames is one image and catches what no validator does: an empty panel,
a graphic over a mouth, two captions stacked, a counter showing a bare `0`.

## Timing floor — the one number to remember

**No visual element under ~2.3 seconds.** With a 0.3 s fade in and 0.3 s fade out, a 1.1 s
card is readable for half a second and reads as a glitch, not a design.

| Element duration | Actually readable |
|---|---|
| 1.1 s | ~0.5 s — feels like a flash |
| 2.3 s | ~1.7 s — the floor |
| 3–6 s | comfortable |
| >15 s | needs internal staging or it goes stale |

## Density

One element every ~3 seconds of runtime is a reasonable target for a fast social short. Below
that the piece reads as a plain talking head with a few decorations. Count them before
rendering — a ~1-minute piece with 9 elements has long bare stretches; the same piece with 18
feels designed.

**One deliberate exception: leave the closing line bare.** After a dense sequence, dropping
every graphic for the final beat is what makes it land.

## Silent failures — the corpus

These produce **no error, no warning, and a successful render**. Full catalogue with symptoms
and fixes: [reference/silent-failures.md](reference/silent-failures.md).

| Symptom | Cause |
|---|---|
| Rendered video has no sound | A timed `<audio>` element with no `id` — the renderer never discovers it |
| A panel is empty for its whole window | `from({opacity:0})` on an element already at `opacity:0` in CSS — animates 0 → 0 |
| Captions unreadable, letters run together | Two caption lines alive at once in the same slot, or a `scale` on the active word eating the gap |
| Music sits on top of the voice | A level copied from a tutorial instead of derived from the measured voice |
| A cue is inaudible or crushed | Computed volume above 1.0 — clamped in silence |
| The person vanishes during a PIP | An opaque full-canvas card on a higher track covering the video |
| Motion stutters on slow moves | Animating `left`/`top`/`width`/`height` instead of transforms |
| Output is sideways | Read the container's coded size instead of the decoded frame |
| A quoted word disappears from a sentence | A filler-word list that matched a real word |

## Reference files

| File | Read it when |
|---|---|
| [reference/cut-and-transcript.md](reference/cut-and-transcript.md) | Cutting silences and fillers; protecting deliberate pauses; the Spanish filler trap |
| [reference/captions-karaoke.md](reference/captions-karaoke.md) | Caption windows, word spacing, the karaoke highlight, remapping to the cut |
| [reference/overlays-and-pip.md](reference/overlays-and-pip.md) | Composition layout, safe zones, PIP transform math, z-order |
| [reference/audio-levels.md](reference/audio-levels.md) | Measuring loudness and deriving every gain from it; diagnosing a take reported as having no audio |
| [reference/joining-pieces.md](reference/joining-pieces.md) | Gluing a sting, end card or second render onto the finished piece without a visible seam |
| [reference/silent-failures.md](reference/silent-failures.md) | A render "worked" but looks or sounds wrong |

## Common mistakes

- **Trusting the exit code.** Every defect above shipped with a clean lint and a successful render.
- **Designing before looking at a frame.** Where the face sits in the frame decides where every
  graphic can go. Extract a few frames and look before choosing zones.
- **Picking levels by feel.** Measure the voice first; every other gain is arithmetic from there.
- **Transcribing twice.** The cut, the captions and the cues all come from one word array.
- **Cutting a filler list blind.** In some languages the common fillers are also real words.
  Dry-run and read what matched.
