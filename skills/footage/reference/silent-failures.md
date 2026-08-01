# Silent failures

Every entry here was found on a real run **after** the tooling reported success: lint clean,
render exit 0, output file valid. None of them throws. They are ordered by how expensive they
are to discover late.

The unifying lesson: **a validator proves the document is well-formed, not that the frame is
right.** Two habits catch all of it — measure numbers with a tool, and look at pixels.

---

## 1. Timed `<audio>` without an `id` renders silent

An audio element carrying `data-start` but no `id` is never discovered by the renderer. It
appears in the DOM, it validates, and it produces nothing.

```html
<!-- silent -->
<audio src="sfx/whoosh.mp3" data-start="8.48" data-duration="2.0" data-volume="0.3"></audio>
<!-- audible -->
<audio id="sfx-2" src="sfx/whoosh.mp3" data-start="8.48" data-duration="2.0" data-volume="0.3"></audio>
```

**Catch it:** lint for it if your renderer offers the rule; otherwise grep that every `<audio>`
with `data-start` also has `id`.

**Confirm after render** — for *"is there any audio at all"*, measure the whole file. A fully
mute track reads `-inf`:

```bash
ffmpeg -hide_banner -nostats -i final.mp4 -af ebur128 -f null - 2>&1 | grep -E '^\s+I:'
```

That is a different question from *"did the music bed land"*, which needs the band-energy
comparison in §5 — do not substitute one for the other.

---

## 2. `from({opacity: 0})` on an element already at `opacity: 0`

A tween that animates *from* a value animates toward the element's **current** value. If CSS
already sets `opacity: 0`, the tween runs 0 → 0. The animation executes perfectly and nothing
appears — for the element's entire window.

```js
// no-op when CSS says .row { opacity: 0 }
tl.from(sel, { opacity: 0, duration: 0.34 }, t);
// correct
tl.fromTo(sel, { opacity: 0 }, { opacity: 1, duration: 0.34 }, t);
```

**Rule:** any element that starts hidden in CSS must be animated with `fromTo`, never `from`.
**Catch it:** snapshot the middle of the element's window and look.

---

## 3. Two caption lines alive in the same slot

Giving consecutive caption lines a lead-in and a tail makes them overlap. Alternating them
across tracks silences the overlap warning but leaves **both painted at the same y** — the
text renders on top of itself and is unreadable.

**Rule:** captions are a single readable slot. Windows must be strictly sequential:
line *n* ends one frame before line *n+1* begins. Quantize start and end **inside** the same
loop iteration — rounding them independently afterwards can push a start below the previous
end and reintroduce the overlap.

---

## 4. A `scale` on the active word eats the space between words

Highlighting the spoken word with `scale: 1.09` grows it ~9% in both directions. Two adjacent
highlighted words then consume the flex gap between them and read as one long token
(`terminalcorriendo`).

**Fix:** give each word its own horizontal padding so the growth expands into the padding
rather than into the neighbour, and keep the scale modest (≤1.06).

---

## 5. Music levels copied instead of derived

Published guidance for a music bed assumes a **normalized voiceover** (~−16 LUFS). Raw
recordings sit far lower — a phone selfie take can measure −26 LUFS. Applying the "safe"
default to that voice puts the bed **above** the speaker.

```bash
ffmpeg -hide_banner -nostats -i voice.wav -af ebur128 -f null - 2>&1 | grep -E '^\s+I:'
```

Derive instead of copying — the arithmetic is in [audio-levels.md](audio-levels.md).

**Verify the bed actually landed** — phase-cancelling against a re-encoded track does not work
(the encoder changed the waveform). Compare band energy where the bed lives and the voice does
not:

```bash
ffmpeg -hide_banner -nostats -i final.mp4 \
  -af "highpass=f=40,lowpass=f=90,astats=metadata=1:reset=0" -f null - 2>&1 | grep 'RMS level'
```

Run it on the voice-only track too. More energy in the final = the bed is present.

---

## 6. Computed volume above 1.0 is clamped in silence

Audio `volume` is clamped to `[0, 1]`. A bundled SFX library can span 30 dB between files, so a
correct calculation against a quiet source can legitimately demand `2.34` — and gets `1.0`,
playing far under the intended level, with no warning.

**Fix:** normalize every asset to a common loudness floor once, then compute gains. Assert
every computed volume is ≤ 1.0 before rendering.

---

## 7. An opaque full-canvas card hides the video during PIP

Shrinking the speaker to a corner square only works if the speaker is **above** the graphic
that took the frame. Cards usually sit on a higher track than the video, so a card with an
opaque background covers the PIP completely.

**Fix:** raise the video wrapper's `z-index` for the PIP window (a class swap, not a tween).
**Catch it:** snapshot mid-PIP. The graphic will look perfect and the person will be absent.

---

## 8. Animating layout properties makes motion stutter

`left` / `top` / `width` / `height` snap to integer device pixels during layout. Under a
seek-by-frame capture engine, a slow move or an ease-out tail visibly steps.

**Fix:** animate `x` / `y` / `scale` / `opacity`. They interpolate sub-pixel.

**Consequence for a square PIP from vertical footage:** uniform scale preserves aspect, so a
9:16 source shrinks to a 9:16 rectangle. To get a square without squashing the subject, clip
first and scale second — `clip-path: inset(21.875% 0 21.875% 0)` on a 9:16 element leaves a
centred square band; the transform then sizes it.

---

## 9. A filler-word list that matches a real word

Filler removal in English is safe: `um`, `uh`, `hmm`, `mhm`, `uh-huh`, `ah`, `huh`, `hm`, `m`
are never real words. Other languages are not so lucky — in Spanish `este`, `pues`, `bueno`
and `o sea` are all legitimate vocabulary.

Worse: a speaker **quoting** a filler ("all the *uh*s I just said are gone") has that token in
the transcript as meaningful content. Cutting it silently breaks the sentence.

**Rule:** ship only unambiguous sounds by default, make the list extensible per run, and print
exactly which tokens matched in the dry run. Read that list before rendering.

---

## 10. The container lies about orientation

`ffprobe -show_entries stream=width,height` returns **coded** dimensions and ignores the
display matrix. Phone footage is commonly stored 1920×1080 with `rotation=90`.

**Fix:** decode a frame and measure that. Everything downstream — canvas size, safe zones,
caption position — depends on getting this right the first time.
