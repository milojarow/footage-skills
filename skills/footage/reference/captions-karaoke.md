# Captions — windows, spacing, karaoke

Word-level timestamps are the whole game. An `.srt` collapses words into blocks and throws the
granularity away, so keep the word array as the source of truth and treat `.srt` as an export.

## Remap to the cut timeline first

After the cut, original timings are wrong. Map every word onto the new timeline before writing
a single caption:

```python
def remap(seconds, segments):
    """segments = the (start, end) spans that survived the cut, in order."""
    elapsed = 0.0
    for start, end in segments:
        if seconds < start:
            return elapsed          # fell inside a removed span
        if seconds <= end:
            return elapsed + (seconds - start)
        elapsed += end - start
    return elapsed
```

Emit a word file that says which timeline it is on — a caption renderer that silently receives
original timings produces a package that drifts further from the audio the longer it plays.

```json
{ "timeline": "cut", "words": [{ "text": "Hola", "start": 0.08, "end": 0.34 }] }
```

## Windows must be strictly sequential

Captions occupy **one readable slot**. Two lines alive at once render on top of each other and
are unreadable — and alternating them across tracks silences the overlap warning without fixing
the picture (see [silent-failures.md](silent-failures.md) §3).

```
start[n] = first word's start − 0.10          (clamped above previous end + 1 frame)
end[n]   = start[n+1] − 1 frame               (last line: last word's end + 0.40)
```

**Quantize inside the loop**, not afterwards. Rounding start and end independently can push a
start below the previous end and reintroduce the overlap you just removed.

## Grouping

| Canvas | Words per line | Max span |
|---|---|---|
| 9:16 vertical | **3–4** | ~2.0 s |
| 16:9 | 6–8 | ~3.5 s |

Vertical is narrow. Seven words per line wrap badly on a phone and read as a subtitle file
rather than as designed captions.

## The karaoke mechanic

Each word is a span. On its `start`, tween colour (and optionally a small scale) to the accent;
on its `end`, tween back.

```js
tl.to(`#cap${l}-w${i}`, { color: ACCENT, scale: 1.06, duration: onDur }, word.start);
tl.to(`#cap${l}-w${i}`, { color: BASE,   scale: 1,    duration: 0.16   }, word.end);
```

**Short words need a shorter highlight.** A fixed 0.10 s "on" tween overlaps its own "off"
tween when the word lasts 0.12 s, which the linter flags and which looks like a flicker:

```python
on_dur = min(0.10, max(word_span, 0.06) * 0.45)
```

**Give each word its own padding.** A `scale` grows the word into its neighbour and eats the
flex gap, so two consecutive highlighted words merge into one token. Padding on the word plus a
small gap survives the growth:

```css
.capline { display:flex; flex-wrap:wrap; justify-content:center; gap:0 6px; }
.capline .kw { display:inline-block; padding:0 11px; }
```

## Placement

Put captions where the body is, not where the face is. Extract a few frames and look before
choosing a `y`.

For a 1080×1920 canvas with a head-and-shoulders framing:

| Band | Content |
|---|---|
| 0 – ~200 px | Platform UI (avatar, name, close) — keep clear |
| ~200 – ~1350 px | Face — no graphics |
| ~1400 – ~1600 px | **Captions** |
| ~1630 – 1920 px | Platform UI (reply, audio strip) — keep clear |

Anchoring captions around **y ≈ 1500** clears both UI strips and sits over the chest, where a
dark shirt gives free contrast. Legibility still needs a hard text-shadow — a caption over a
light wall or a bright shirt disappears without one.

Add a full-canvas legibility gradient (dark at top and bottom, transparent through the middle)
under the caption layer. It costs nothing and rescues frames where the background goes pale.

## Hard-kill every exit

A fade-out that ends on the next clip's start boundary leaves stale opacity when a seek lands
past it — the renderer seeks non-linearly, so this is not theoretical.

```js
tl.to(sel, { opacity: 0, duration: 0.16 }, end - 0.16);
tl.set(sel, { opacity: 0 }, end + 1 / FPS);   // hard kill
```
