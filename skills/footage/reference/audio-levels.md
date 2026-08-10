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

## Broadband RMS lies about a tonal element over a noise bed

The lie always pushes you to add gain you should not add.

A brand sting (three rising notes plus a chord) mixed over a park-ambience tail, measured in
broadband RMS by windows — which is what one measures:

| Window | Broadband RMS |
|---|---|
| Ambience alone | −31.5 dBFS |
| The three notes | −30.0 dBFS — only +1.5 dB |
| The chord | −23.9 dBFS |

Read that way the notes are **buried**: a decibel and a half over the noise floor is nothing,
and the obvious reflex is to turn the sting up. It is false. The same measurement split by band:

| Band (Hz) | Ambience | Notes | Difference |
|---|---|---|---|
| 60–250 | −43.2 | −34.7 | +8.5 |
| 250–700 | −50.0 | −34.0 | **+16.1** ← the fundamentals |
| 700–1500 | −52.6 | −40.5 | +12.1 |
| 1500–3500 | −48.8 | −41.8 | +7.1 |
| 3500–8000 | −53.4 | −45.3 | +8.1 |

The notes win **+16 dB where they actually live** and +7 to +12 dB everywhere else. They are
perfectly audible. Adding gain would have made the sting shout over a voice that had just
finished speaking.

**Why it lies:** broadband RMS sums the whole spectrum into one number. A **tonal** element
concentrates its energy in a few narrow bands; a **noise** bed spreads it across all of them.
Compared with a single number, the noise accumulates energy from fifty bands where the note
contributes in three — so the number under-represents exactly what the ear separates, because
the ear does not average: it listens in critical bands.

This is one case of a general trap: **an average over a dimension the destination system does
not average does not describe that system.**

### Measure by band instead

Before touching the gain of a tonal element over a noise bed — sting, bell, synth line, a cue
over traffic or rain:

```python
def band_rms(a, sr, t0, t1, f0, f1):
    s = a[int(t0*sr):int(t1*sr)]
    F = np.fft.rfft(s * np.hanning(len(s)))
    f = np.fft.rfftfreq(len(s), 1/sr)
    m = (f >= f0) & (f < f1)
    return 20*np.log10(np.sqrt((np.abs(F[m])**2).sum())/len(s)*2 + 1e-12)
```

Bands that suffice: 60–250, 250–700, 700–1500, 1500–3500, 3500–8000. Comparing windows of
different length requires normalizing for duration (`− 10*log10(dur_a/dur_b)`), or the longer
segment looks louder just for being longer. **If the element wins more than ~6 dB in the band
holding its fundamentals, it is audible. Do not turn it up.**

### Derive gain from the file that exists, not from the buffer you produced

A companion case in the same job: the author of a sting reported his file peaking at −3 dBTP
and the measurement of the delivered file said −6.2. Both were right about different things —
he measured the **mono** buffer he handed the encoder, and an ffmpeg mono→stereo upmix
**preserves power and spreads −3 dB per channel**. Between the buffer and the file there is a
resample, a channel change and a quantization, and any of the three moves the peak.

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

## A number that does not move proves nothing — look for a marker that RISES

Adding a 2.4 s sting over a 26.75 s mix (voice + bed) left the file's aggregate metrics
**identical** before and after:

    before   mean −15.9 dB   max −0.6 dB   I −13.3 LUFS
    after    mean −15.9 dB   max −0.6 dB   I −13.3 LUFS

Read that way you conclude "the filter never applied, the layer never landed" and go debug an
`amix` that was perfect. The cause is arithmetic, not a fault: **the peak and the programme
integrated live in another section** (the 20 s of speech), so a short, moderate layer in the last
6 s cannot move either of them, and a 26 s mean does not notice 2.4 s at −27 dB.

**The verification that does prove it** — a marker that has to go **up**, scoped to the window
and the band the new layer lives in:

```python
raw = subprocess.run(['ffmpeg','-v','error','-ss',str(t0),'-t',str(dur),'-i',mix,
                      '-ac','1','-ar','48000','-f','f32le','-'], capture_output=True).stdout
x = np.frombuffer(raw, dtype=np.float32)
X = np.abs(np.fft.rfft(x*np.hanning(len(x))))**2
f = np.fft.rfftfreq(len(x), 1/48000)
for lo, hi in [(0,300),(300,1000),(1000,3000)]:
    m = (f >= lo) & (f < hi); print(lo, hi, 100*X[m].sum()/X.sum())
```

In the measured case the sting's band (300 Hz–3 kHz) went from **5.3% → 17.6%** inside that
window. That is proof. An unchanged number is not.

**Corollary:** run the same measurement on the **rendered MP4**, not only on the intermediate
audio file — it is the only way to know the layer survived the render.

### A three-minute average does not describe two seconds

Before mixing, the question was whether the sting (300–3000 Hz) would fight the music bed. The
answer "yes, the music lives in the upper mids" was true **for the whole track** and false for
the 2.4 s that mattered: measured in that exact window, the bed was **94.7% below 300 Hz** — a
low-pulse section with no melody. Zero overlap.

Measure **the window the layer falls in**, never the track average. And derive the level by
comparing energies **inside the shared band**, not global levels:

    bed in 300–3000 Hz, in that window    −30.1 dB
    sting at gain 1.0, in its band        −27.8 dB   → +2.3 dB over the bed

### Headroom is a separate question from masking

Two different questions, answered separately. In that case the sting was attenuated −3 dB
**not** because it covered anything, but because the mix already peaked at −0.6 dBFS and adding
a layer peaking at −6 dB risked clipping. With the −3 dB plus an `alimiter=limit=0.977`
(≈ −0.2 dBFS) the final peak landed at −0.8 dB. Conflating the two leads to attenuating for the
wrong reason and to missing the real clipping.
