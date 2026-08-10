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

## 4 — The re-encode

Placing the incoming piece *inside* the composition re-encodes it. A sting that is mostly flat
gradient is the worst case for a compressor (banding). If it must be re-encoded, raise quality
(`--crf 16` territory); if no dissolve is needed, concatenating by stream copy leaves the piece
pixel-identical.
