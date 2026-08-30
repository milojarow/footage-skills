# Real footage dropped into a generated background — the seam nobody owns

A generated animated background, a clip the client shot (or was given), and a typographic
layer sitting on top don't fit cleanly into "AI-generated visuals" or "real recorded footage"
— they're both, at once, on the same timeline. Treat a mixed piece as normal, not as an edge
case to improvise: whichever pipeline you started from, the join is where the defects
concentrate.

## What the join needs

1. **Decide which side is authoritative.** The client's footage is the fixed point; the
   generated side is the one that adapts. Generated material is re-runnable at a price; the
   client's take is not re-runnable at all.
2. **Measure separability before promising the composite** — corner standard deviation of the
   footage's background, and the subject's share of the frame:

   ```
   corner patches → per-channel standard deviation   (3–4 on a 0–255 scale = flat)
   object pixels  → share of the frame               (roughly 15-25% = a real subject)
   ```

   A flat background at that deviation separates cleanly by distance-to-colour (soft ramp +
   light blur). A textured one doesn't, and that's a different, much more expensive job.
3. **Match the destination, not the source.** Scale and speed of the borrowed clip are set by
   the slot it has to fill in the edit, not by its native duration.
4. **Audit what the borrowed clip carries in.** Baked logos, baked headlines, a baked
   voiceover, a distinctive garment — decisions another studio made for a different brand.
   Accepting them by omission looks identical to having chosen them.
5. **Remove what doesn't belong per frame, not by rectangle.** A fixed kill-band (zeroing the
   alpha over a measured logo box or headline band) clips the subject wherever it grows into
   the killed region over the clip — the signature is a perfectly straight edge cutting an
   organic subject. Measure the keep/kill boundary per frame instead: label the connected
   components of the subject mask, take the one that reaches an always-present anchor (a stem,
   a base, the bottom edge), and read its topmost row per frame. Everything above that row is
   overlay; everything below is subject.

Step 2's separability check and step 5's per-frame boundary measurement are the same
techniques worth reaching for on any borrowed clip that has to sit inside a scene it wasn't
shot for, generated background or not.
