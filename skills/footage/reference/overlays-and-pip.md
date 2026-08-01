# Overlays, zones and the PIP move

The composition is one HTML document: the footage plays underneath, graphics sit on tracks
above it, and one render produces the finished file.

> **REQUIRED BACKGROUND:** this file assumes the HyperFrames composition contract —
> `data-start` / `data-duration`, `class="clip"`, tracks, the paused master timeline. Read
> `hyperframes-core` first if those are unfamiliar; nothing here re-teaches them.

## Design from a frame, not from a template

Before choosing any position, extract frames across the take and look at them.

```bash
for t in 5 20 40 55; do ffmpeg -y -v error -ss $t -i in.mp4 -frames:v 1 -vf scale=405:-2 f$t.jpg; done
ffmpeg -y -v error -i f5.jpg -i f20.jpg -i f40.jpg -i f55.jpg -filter_complex hstack=inputs=4 grid.jpg
```

Where the face actually sits decides everything. A layout guide that says "cards go in the
bottom band" is useless if the speaker framed themselves low and their chin is in that band.

For a typical head-and-shoulders vertical take:

| Band (1080×1920) | What is there | Use for |
|---|---|---|
| 0 – 200 | Platform UI | nothing |
| 200 – 340 | Wall above the head | **corner chips** |
| 340 – 1350 | Face | nothing — unless the graphic is *meant* to take over |
| 1100 – 1400 | Chest / shoulders | **info panels** |
| 1400 – 1600 | Chest | **captions** |
| 1630 – 1920 | Platform UI | nothing |

Panels and captions both want the chest. Panels go above, captions below, and neither moves.

## Element budget

Aim for **one element every ~3 seconds**. Fewer and the piece reads as a plain talking head;
the difference between 9 elements and 18 across a ~1-minute piece is the difference between
"decorated" and "designed".

Mix the registers so it does not read as one repeated card:

| Register | Frequency | Example |
|---|---|---|
| Corner chip | one per section | step number + label |
| Info panel | 1–2 per section | a list, a table, a stat |
| Full takeover | 2–3 per piece | a big number, a hero graphic |
| Recap | once | everything so far, together |

**Full takeovers are scarce by design.** If every beat takes the frame, none of them is an
event, and the person disappears from their own video.

## Minimum duration

**No element under ~2.3 s.** A 0.3 s fade in and 0.3 s fade out leave a 1.1 s card readable for
half a second — it reads as a glitch. Compute the real readable time, not the declared
duration, and check the shortest element before rendering.

## Tracks

**Track number is both z-order and the overlap namespace.** A higher `data-track-index` paints
*above* a lower one — which is why a full-frame card on track 2 covers the video on track 1, and
why the PIP below needs an explicit `z-index` to climb back over it. Two elements on the same
track must not overlap in time; different tracks may.

Give each register its own track:

```
1   video          10  program audio      11  music bed
2   full takeovers  3  captions           5   corner chips
6   info panels    20+ SFX one-shots
```

## The three registers that are not PIP

PIP gets the long treatment below because its geometry is non-obvious. These three are simpler
but still need to be right.

### Corner chip — a persistent section marker

Sits in the wall band above the head, never moves, one per section.

```css
.chip {
  position: absolute; left: 48px; top: 52px;
  display: flex; align-items: center; gap: 18px;
  padding: 16px 28px 16px 20px;
  background: rgba(13,17,23,.82);
  border: 2px solid var(--accent);
  border-radius: 12px;
  box-shadow: 0 18px 46px -14px rgba(0,0,0,.7);
}
.chip .num { font: 700 56px/1 ui-monospace, monospace; color: var(--accent); }
.chip .lbl { font: 700 30px 'Inter', sans-serif; letter-spacing: .16em; text-transform: uppercase; }
```

Enter with a short stagger — number, then label, then an underline growing to a fixed width:

```js
tl.from(`${sel} .num`, { opacity: 0, y: 14, duration: 0.28 }, t + 0.08);
tl.from(`${sel} .lbl`, { opacity: 0, x: -18, duration: 0.30 }, t + 0.18);
tl.fromTo(`${sel} .bar`, { width: 0 }, { width: 300, duration: 0.45 }, t + 0.26);
```

### Info panel — a list, a table, a stat

Lives in the chest band, above the captions. One accent stripe on the left ties it to the
section's colour.

```css
.panel {
  position: absolute; left: 56px; right: 56px; top: 1088px;
  background: rgba(13,17,23,.90);
  border: 2px solid #2c333e; border-left: 8px solid var(--accent);
  border-radius: 16px; padding: 30px 36px;
  box-shadow: 0 30px 80px -20px rgba(0,0,0,.8);
}
.panel .kicker { font: 700 22px 'Inter'; letter-spacing: .22em; text-transform: uppercase; color: #7d8590; }
.panel .row { opacity: 0; }        /* ← animate with fromTo, never from */
```

Rows enter staggered, ~0.22–0.28 s apart. **Rows that start hidden in CSS must be animated with
`fromTo`** — see [silent-failures.md](silent-failures.md) §2.

A stat panel reveals in stages rather than all at once: the old value, the arrow, the new value,
then a counter. Any element whose text is written by a counter's `onUpdate` must **start at
`opacity: 0`** — otherwise its literal HTML text shows until the tween begins.

```js
(function () {
  const o = { v: 0 };
  tl.to(o, { v: -22, duration: 0.85, ease: 'power2.out', onUpdate() {
    document.querySelector('#pct').textContent = Math.round(o.v) + '%';
  }}, t + 1.15);
})();
tl.to('#pct', { opacity: 1, duration: 0.22 }, t + 1.05);   // reveal as it starts counting
```

### Recap — everything so far, once

Near the end, stack the sections as rows with their own accent colours, entering ~0.16 s apart.
Give it a dim over the footage: a recap covers the speaker's face by design, and without the dim
the rows fight the background. Keep it to one appearance — a recap that repeats is a list.

## The PIP move

The speaker shrinks to a corner square; a terminal, chart or animation takes the frame.

### Transforms only

Animating `left` / `top` / `width` / `height` snaps to integer device pixels and stutters under
a seek-by-frame capture engine. Use `scale`, `x`, `y`.

### Square from vertical footage

Uniform scale preserves aspect, so a 9:16 source shrinks to a 9:16 rectangle. For a square
without squashing the subject: **clip first, scale second.**

```css
.video-wrapper.pip {
  clip-path: inset(21.875% 0 21.875% 0 round 78px);  /* 1080×1920 → centred 1080×1080 */
  z-index: 50;                                        /* above the card that took the frame */
}
```

`(1920 − 1080) / 2 / 1920 = 21.875%` off the top and bottom.

### The translate

The clipped band's centre is the element's centre, so the translate is a pure delta:

```
scale       = 320 / 1080 = 0.2963              # target square ÷ source width
target ctr  = (684 + 160, 96 + 160) = (844, 256)
element ctr = (540, 960)
x, y        = (844 − 540, 256 − 960) = (+304, −704)
```

```js
tl.set('#video-wrap', { className: 'video-wrapper pip' }, IN - 0.55);
tl.to('#video-wrap', { scale: 0.2963, x: 304, y: -704, duration: 0.55, ease: 'power3.inOut' }, IN - 0.55);
tl.to('#video-wrap', { scale: 1, x: 0, y: 0, duration: 0.55, ease: 'power3.inOut' }, OUT - 0.55);
tl.set('#video-wrap', { className: 'video-wrapper' }, OUT);
```

### The z-index is not optional

A full-frame graphic with an opaque background sits on a higher track than the video and
**covers the PIP completely**. The graphic renders perfectly; the person is simply absent.
Raise the wrapper's `z-index` in the PIP class — and snapshot mid-PIP to confirm.

## Give the ending somewhere to land

A file that ends a few frames after the last word reads as if the speaker was cut off
mid-sentence. Run the composition past the clip:

```
composition duration : 60.0     # clip is 57.4
56.75  fade to black begins (0.85 s)
57.10  closing card enters over the black
57.60  black complete
60.00  end
```

The clip and the program audio keep their real duration; only the composition is longer. Two to
three seconds is enough for a closing line to land.

## Mockups: HTML, never generated imagery

A terminal, a chat notification, a code editor, a phone UI — build these in HTML/CSS.

Generated imagery produces text that *looks* like code and says nothing: invented commands,
inconsistent glyphs, garbled output. In HTML the text is real, the font is yours, it can type
out character by character, and it renders sharp at any resolution.

If the content is illustration, generate it. If the content is **information**, build it.
