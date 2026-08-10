# Cutting silences and filler words

The cut is the only destructive stage. Everything after it is drawing, so this is the one to
get right and the one to always dry-run.

## Ask for filler words, then throw the text away

Most speech-to-text providers strip `um`/`uh` by default. That gives a clean transcript for
free — and makes filler removal from *video* impossible, because absent words have no
timestamps.

**Request them.** Use their spans as cut regions, and filter them out of the text yourself.
The clean transcript is something you produce, not something the API hands you.

This is the single most counter-intuitive setting in the pipeline: you turn *on* the thing you
want gone.

## The keep-list

```
kept  = words that are not fillers
gaps  = spans between consecutive kept words
cut   = fillers ∪ { gaps longer than the threshold }
keep  = everything else, with `pad` seconds restored on each side of every cut
```

`pad` (≈0.08 s) is what stops the cut clipping the tail consonant of a word. Without it the
edit sounds clicky even when the timestamps are correct.

### Threshold

| Gap threshold | Result |
|---|---|
| 0.4 s | Aggressive. Punchy, but starts eating natural breaths |
| **0.6 s** | Sensible default. Removes dead air, keeps the rhythm of speech |
| 1.0 s | Conservative. Only long stalls |

Print the distribution before choosing — how many cuts each threshold implies and how many
seconds it removes. The right value depends on the speaker, not on the format.

## Deliberate pauses must be declared

**A gap detector cannot tell a dramatic beat from dead air.** The pause before a punchline is
usually the *longest* silence in the take, which means an aggressive threshold eats it first.

There is no signal in the audio that distinguishes them. The pause has to be declared —
by the operator, per run:

```
--keep-gap 67.2-69.2      # a silence overlapping this range survives the cut
```

Implementation: when deciding whether to cut a gap, skip it if it overlaps a protected range.

```python
protected = any(start > lo and previous_end < hi for lo, hi in protected_ranges)
if start - previous_end > threshold and not protected:
    ...cut...
```

## The gap detector hears; it cannot see

The silence detector works on **audio** — gaps between words with timestamps. It never looks at
the **image**. That makes it blind to a common and expensive case:

**Footage that already went through an editor usually carries a brand outro glued to the end —
logo, card, contact line. That outro is silent.** To the detector it is indistinguishable from
dead air, and it cuts the whole thing without a single error.

Measured case: a 24.07 s take, last word at 20.80. The detector reported "tail: 3.26 s of
silence". Cutting it looked obviously correct — and deleted the closing brand card, which ran
from 21.15 to 24.06.

The twin case lives at the head: 1.10 s of "silence" before the first word can be an opening
transition (fade, page-peel, wipe). In that same take the head *was* cut, but for a reason the
detector could not know: looking at the frames showed a stray mouse cursor left in by whoever
exported it. The frames decided the cut on evidence, not on a threshold.

**Rule: before cutting ANY span the detector marks as silence — above all the first and the
last — extract frames from that span and look at them.**

```bash
for t in 21.2 22.0 23.0 23.9; do
  ffmpeg -y -v error -ss $t -i in.mp4 -frames:v 1 -vf scale=200:-2 tail-$t.jpg
done
ffmpeg -y -v error -i tail-21.2.jpg -i tail-22.0.jpg -i tail-23.0.jpg -i tail-23.9.jpg \
  -filter_complex "[0][1][2][3]hstack=inputs=4" tail-sheet.jpg
```

A four-frame contact sheet answers it in five seconds. If there is content, the span is not dead
air — it is a piece, and keeping or dropping it is the operator's call, not the threshold's.

Cheap to automate if you want it: mean luminance and its variance across the span. Real dead air
over a static shot barely varies; an animated outro does. But **a static logo card does not vary
either**, so variance alone gives false negatives — which is why the rule is to LOOK, not to
measure.

### A take handed to you "to edit" may already BE a finished edit

Prove it before running the cutter. One measured case: a clip arrived "to have its silences
cut", and the timestamped transcript showed 1.22 s of gap at the head, 3.22 s at the tail and a
1.00 s gap mid-sentence — textbook raw take with dead air. It was nothing of the sort. Pulling
frames showed the 1.22 s head was a **designed opening shot** with a whip transition, the 3.22 s
tail was an **animated brand card**, and the piece already carried **karaoke captions burned
into the pixels**. The only genuinely cuttable gap was the 1.00 s one, worth 0.53 s recovered
out of 28.7 s — 1.9% of the runtime.

**Why this breaks gap cutters specifically.** A protected-range flag (`--keep-gap START-END`)
guards deliberate pauses *between words*. The edges are not gaps between words: they are the
head before the first word and the tail after the last one, and the typical implementation
trims them unconditionally:

```python
segmentStart = max(0, keptWords[0].start - pad)           # ALWAYS trims the head
segments.append((segmentStart, min(dur, prevEnd + pad)))  # ALWAYS trims the tail
```

Neither line consults the protected ranges. Run the tool as-is on a piece like that and it
deletes the opening AND the brand card — no error, no warning, success verdict. **Treat the
head and the tail as protected by default**, and clamp them against the protected ranges the
same way the interior gaps are.

**The pre-flight check.** A contact sheet of ~10 frames spread across the piece, looked at with
your eyes. It takes seconds and answers three things the transcript cannot:

1. Are the head and the tail black/static, or are they content?
2. Are there already burned-in captions? (If so you cannot add yours without stacking them, and
   any cut has to land where no caption is on screen.)
3. Where does the subject sit in the frame? (That decides where any graphic can go.)

**Cutting with burned-in captions present:** put the cut inside the blank window between two
caption groups. Find it by sampling frames every 0.2 s around the gap — if there is different
text on screen on either side, the cut produces a subtitle jump that no linter sees.

**Short rule:** the transcript tells you where there is no voice. It does not tell you where
there is no video. Before treating a gap as dead air, look at the frame — above all in the first
and last second, which is exactly where openings and brand cards live, and exactly where the
cutters have no guard.

**The general shape:** a detector that measures one dimension declares healthy whatever is
broken in the dimension it does not watch, and declares it with total confidence. The gap
detector hears and does not see; a linter validates structure and does not see; a screenshot
sees and does not hear. When a check says "there is nothing here", ask **in which dimension** it
looked.

## Filler words by language

The list a provider strips by default is **English only**:

```
um · uh · hmm · mhm · uh-huh · ah · huh · hm · m
```

None of those is ever a real word. Other languages have no such luck:

| Language | Safe to cut | **Never cut blind** |
|---|---|---|
| English | `um` `uh` `hmm` `mhm` `uh-huh` `ah` `huh` `hm` | — |
| Spanish | `eh` `ehh` `mmm` `mm` `em` `emm` | `este` `pues` `bueno` `o sea` — all real words |

Ship only the unambiguous sounds. Make the rest opt-in per run and **print which tokens
matched** so the operator can read the list before rendering.

### The quoted-filler trap

A speaker who *quotes* a filler — "all the **uh**s I just said are gone" — has that token in
the transcript as meaningful content. Cutting it leaves "all the s I just said are gone".

This is not hypothetical; it is the first thing that happens when a script talks about filler
removal. The dry-run list is the only defence.

## Cut video and audio in one graph

Cutting audio alone desynchronises the lips. Trim both streams in a single `filter_complex` so
they cannot drift:

```bash
# per segment i: [0:v]trim=start=S:end=E,setpts=PTS-STARTPTS[vi];
#                [0:a]atrim=start=S:end=E,asetpts=PTS-STARTPTS[ai];
# then:          [v0][a0][v1][a1]…concat=n=N:v=1:a=1[outv][outa]
ffmpeg -i in.mp4 -filter_complex "$GRAPH" -map '[outv]' -map '[outa]' \
  -c:v libx264 -crf 20 -c:a aac -b:a 192k out.mp4
```

Frame-accurate cuts at arbitrary timestamps cannot be done by stream copy — the output is
re-encoded. Budget one generation of loss here and **do not add a second** by burning captions
in a separate pass.

## Verify the cut

```bash
ffprobe -v error -show_entries format=duration -of csv=p=0 out.mp4          # matches the plan?
ffprobe -v error -select_streams v:0 -show_entries stream=duration -of csv=p=0 out.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=duration -of csv=p=0 out.mp4
```

Video and audio durations differing by a few milliseconds is normal (audio frame
quantisation). Tens of milliseconds is drift and means the streams were not cut together.

Then confirm orientation from a **decoded frame**, not the container fields — see
[silent-failures.md](silent-failures.md) §10.

## Keep the transcript

The transcript is the expensive artifact; everything else regenerates for free. Save the raw
provider response next to the output and allow re-running the cut against the saved file.
Re-tuning the threshold, the pad and the filler list then costs nothing.
