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
