# Audio levels — derive, never copy

Published mix guidance assumes a **normalized voiceover**. Raw footage is not that. Copying a
level from a tutorial is how a music bed ends up on top of the speaker.

**Rule: measure the voice first. Every other gain is arithmetic from that number.**

## Measure everything

```bash
ffmpeg -hide_banner -nostats -i asset.mp3 -af ebur128 -f null - 2>&1 | grep -E '^\s+(I|LRA):'
```

A real spread from one production:

| Source | Integrated |
|---|---|
| Voice, as muxed in the source video | **−26.1 LUFS** |
| Music bed (generated, mastered) | −15.7 LUFS |
| SFX: impact | **−4.5 LUFS** |
| SFX: keypress | **−36.3 LUFS** |

The music is **10 LU louder than the speaker** at unity. The SFX library spans **32 dB**
between its own files. No single hand-picked number can be right across that.

## The formula

```python
def level(source_lufs, below_lu, voice_lufs):
    """Linear gain that puts `source` `below_lu` under the speaking voice."""
    return 10 ** (((voice_lufs - below_lu) - source_lufs) / 20)
```

| Element | LU under the voice |
|---|---|
| Music bed | 16 – 18 |
| Transition whoosh | 8 – 10 |
| Impact / stinger | 6 – 8 |
| Typing ticks, texture | 12 – 14 |

## Two clamps that ruin a mix in silence

**1 — `volume` is capped at 1.0.** A correct calculation against a quiet asset can legitimately
demand `2.34`. It becomes `1.0`, plays far under the intended level, and nothing warns you.

**Fix:** normalize every asset to a common floor **once**, then compute:

```bash
ffmpeg -y -i sfx.mp3 -af loudnorm=I=-20:TP=-1.0:LRA=11 -c:a libmp3lame -q:a 2 sfx-norm.mp3
```

Then assert before rendering:

```python
assert max(volumes) <= 1.0, "a computed volume was clamped — normalize the assets first"
```

Note `loudnorm` cannot always reach the target on very short transients — a keypress may land
at −31 LUFS when asked for −20. Re-measure after normalizing rather than assuming.

**2 — Raising the voice invalidates every derived gain.** Cleaning and normalizing a −26 LUFS
voice to −16 moves it 10 LU. Every level derived from the old number is now 10 LU too quiet.
**Recompute after any change to the voice**, and keep the voice measurement in one named
constant so there is a single place to update.

## Cleaning the voice

Two operations, in this order:

1. **Isolation / denoise** — removes room tone and hum. A good one is sample-aligned: verify
   the duration before and after differ by only milliseconds, or every downstream timestamp is
   wrong.
2. **Normalize** — `loudnorm=I=-16:TP=-1.5:LRA=11`.

Measured effect of isolation on one phone take:

Measured on the audio **extracted to mono MP3** — hence −26.6 here against the −26.1 the
same voice measures inside the container. Both are the same take; the extraction and the
re-encode account for the 0.5 LU.

| | Before | After |
|---|---|---|
| Noise floor | −47.2 dB | **−84.2 dB** |
| Integrated | −26.6 LUFS | −25.8 LUFS |
| Duration | 57.429 s | 57.423 s |

37 dB of room tone removed, 6 ms of drift. That drift figure is the acceptance test — if an
isolation step returns a materially different duration, do not use it in a lip-synced pipeline.

Then remux the cleaned audio onto the untouched video:

```bash
ffmpeg -y -i video.mp4 -i voice-clean.wav -map 0:v -map 1:a \
  -c:v copy -c:a aac -b:a 192k -shortest video-clean.mp4
```

## Do not replace the voice with a clone

Voice cloning is tempting for a take with poor audio. It breaks lip sync.

Speech-to-speech follows the source prosody but does not preserve duration exactly — a measured
**6.037 s returned for a 6.000 s source** is 0.6% drift, which compounds to ~350 ms over a
minute. Visible desync starts around 80 ms.

Clones are for audio the camera never saw: an intro, an outro, a corrected line over a graphic,
a translated version. Not for re-voicing a shot where the mouth is visible.

## Target the platform

Short-form platforms normalize to about **−14 LUFS**. Landing the final mix between −14 and
−13 means the platform leaves it alone.

```bash
ffmpeg -hide_banner -nostats -i final.mp4 -af ebur128 -f null - 2>&1 | grep -E '^\s+(I|LRA):'
```

## Verify the bed actually made it

Phase-cancelling the voice out of the final mix **does not work** — the render re-encoded the
audio, so the waveforms no longer align and the subtraction returns the full mix.

Compare band energy where the bed lives and speech does not:

```bash
ffmpeg -hide_banner -nostats -i "$FILE" \
  -af "highpass=f=40,lowpass=f=90,astats=metadata=1:reset=0" -f null - 2>&1 | grep 'RMS level'
```

Run it on the voice-only track and on the final mix. More low-band energy in the final is the
bed. On one production: voice-only −33.8 dB, final −31.0 dB — 2.8 dB of bed, present and well
under the speech.
