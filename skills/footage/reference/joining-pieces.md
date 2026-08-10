# Joining two rendered pieces without a visible seam

When a finished edit gets another piece glued to it — a brand sting, an end card, a wipe — the
join is where the work shows. The order of operations matters more than the codecs, and the
frame that decides the result is not the one you would guess.

## 1 — Concatenate video first; mux the audio LAST, as one single track

This is the heaviest point and the least obvious.

An MP4 almost never has the same video and audio duration: in one measured case, 28.200 s of
video against 28.224 s of audio — 24 ms of difference from audio frame quantization. If you
concatenate two files that **already carry** audio, the demuxer has to splice both tracks too,
and that error accumulates exactly at the seam.

The form that cannot fail:

```bash
# 1. strip the audio from both parts
ffmpeg -y -i part-a.mp4 -an -c:v copy a.mp4
ffmpeg -y -i part-b.mp4 -an -c:v copy b.mp4

# 2. concatenate video only, by stream copy
printf "file '%s'\nfile '%s'\n" "$PWD/a.mp4" "$PWD/b.mp4" > list.txt
ffmpeg -y -f concat -safe 0 -i list.txt -c copy video.mp4

# 3. build the COMPLETE audio track at the exact total duration
#    (here: 28.200 + 6.500 = 34.700), and only then mux
ffmpeg -y -i video.mp4 -i full.wav -map 0:v -map 1:a \
  -c:v copy -c:a aac -b:a 192k -ar 48000 -movflags +faststart final.mp4
```

With the audio as one continuous track, **at the junction point there is no junction** — it is
just the next sample. There cannot be a jump because the boundary does not exist. The seam is
left only in the video, where it is exact by construction. No `-shortest`, no `apad`, no offset.

## 2 — The second file MUST start on a keyframe

This is the load-bearing condition of the stream copy. If the second piece opens on a P-frame,
the splice has no reference and renders as garbage.

```bash
ffprobe -v error -select_streams v:0 -show_frames \
  -show_entries frame=pts_time,key_frame,pict_type -of json final.mp4
```

At the seam frame you must see `key_frame=1`, `pict_type=I`.

**Corollary that explains a false positive:** comparing packets with `framemd5` turned up two
differences, which were exactly the two keyframes of the second file — the demuxer rewrites
their SPS/PPS headers on concat. **A different packet is not a different pixel.** Decode and
compare the image before accusing anything of re-encoding.

## 3 — Parameters to match (and why they usually already do)

Measured side by side: `profile`, `codec_tag`, `pix_fmt`, `time_base`, `sample_aspect_ratio`,
`display_aspect_ratio`, `color_range`, `color_primaries`, `color_transfer`, `color_space`,
`chroma_location`, `has_b_frames`. All matched with nothing forced, because both pieces came out
of the same encoder at the same resolution and fps. The only mismatch was `level` (40 vs 50),
which the demuxer tolerates by taking the first one's.

`has_b_frames=0` on both helps: with no B-frames there is no reordering across the splice.

## 4 — 🔴 The frame that decides the join is the LAST one, and it is measured, not judged

When the incoming piece exists in a light and a dark variant, the one that goes is whichever
does not flash against the **last frame of the edit**. That decision is a scalar:

```bash
# last frame of the edit
ffmpeg -v error -sseof -0.05 -i edit.mp4 -frames:v 1 -y /tmp/last.png
# first frame of the piece being glued on
ffmpeg -v error -i piece.mp4 -frames:v 1 -y /tmp/first.png
python3 -c "
from PIL import Image; import numpy as np
for f in ['/tmp/last.png','/tmp/first.png']:
    print(f, np.asarray(Image.open(f).convert('L')).mean())"
```

Decision rule, measured on a real case:

| Mean-luminance difference (0–255) | Verdict |
|---|---|
| < ~20 | hard cut, reads clean |
| 20 – 60 | hard cut acceptable; dissolve if you want it |
| > 60 | **no good variant exists** → dissolve |

### The mistake that makes this necessary

"Does your edit close light or dark?" invites a description of the **scene**, and the scene is
not the number. A sunlit shot with a subject, shadows and foliage averages **112/255** even with
its highlights pinned; a flat brand card averages **246**. Answering "light, a park in full sun"
led to picking the light variant, and the result jumped **+127 of mean luminance in ONE frame** —
half the scale, and it reads as a flash.

    piece A   last frame 245.5  →  first frame of the sting 245.8   jump   0.3
    piece B   last frame 112.5  →                           244.4   jump 131.9

No codec improvement fixes 131.9. And measured afterwards, neither variant worked for piece B:
133.3 against the light one, 93.3 against the dark. **A mid-light close is not fixed by picking a
variant — it is fixed with a dissolve.**

**The general shape:** a question that can be answered with a scalar must not be answered with
an adjective, even when the asker accepts the adjective. In the measured case the number had
already been written down twenty minutes earlier and the impression was given instead. When you
ask anyone — person or agent — for a datum an irreversible decision depends on, **ask for the
command, not for the judgement.**

### The dissolve

8–10 frames spread a jump of ~130 into ~16 per frame, which reads as a dissolve rather than a
cut. Beyond ~10 it starts eating the incoming piece's own opening animation, so the ceiling is
set by the content, not by taste.

Verify frame by frame, not by eye:

```python
raw = subprocess.run(['ffmpeg','-v','error','-ss',str(t0),'-t','0.8','-i',V,
                      '-vf','scale=192:-2,format=gray','-f','rawvideo','-'],capture_output=True).stdout
n = 192*340; prev = None
for i in range(len(raw)//n):
    m = np.frombuffer(raw[i*n:(i+1)*n], dtype=np.uint8).astype(float).mean()
    if prev is not None: print(f"{t0+i/30:.3f}  {m:6.1f}  {m-prev:+6.1f}")
    prev = m
```

Healthy output: the delta column climbs evenly, with no step that stands out.

### Two traps in the method

- **Do not run it on the ALREADY-finished piece** if that one ends in black: it returns 0.0,
  which sends you to the dark variant — a correct answer to a question you did not ask. Measure
  the last frame of part A, before gluing anything.
- **Snapshot directories are not cleared between runs** by most composition tooling. PNGs from a
  previous run are still there, and a script globbing them mixes old timestamps with new ones —
  that produced a "black" frame that did not exist and sent a healthy composition to be debugged.
  Delete the snapshot directory before measuring.

## 5 — The re-encode

Placing the incoming piece *inside* the composition re-encodes it. A sting that is mostly flat
gradient is the worst case for a compressor (banding). If it must be re-encoded, raise quality
(`--crf 16` territory); if no dissolve is needed, concatenating by stream copy leaves the piece
pixel-identical.
