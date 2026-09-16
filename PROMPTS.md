# The prompts that built this

Every line of Voice Leveler was written by AI from the four prompts below, in order. Paste them into Claude (or any capable coding model) and you get the same tool.

Prompt 4 is the correction. It is here on purpose — the first approach to section matching was wrong, and the wrong version is more instructive than the right one.

---

## Prompt 1 — the spec

```
Build me a single-file HTML tool that cleans up and levels spoken-word
audio. Hard requirements:

- One .html file. No build step, no install, no account, no server.
  Everything inline: CSS, JS, no CDN links of any kind. It must work
  with the network switched off.
- All processing happens in the browser with the Web Audio API and
  plain JavaScript. The audio file never leaves the machine.
- No length limit. A two-hour file should work; it is just a longer
  array. Do not add a cap.
- Input: drag-and-drop or file picker, any format the browser can
  decode (wav, mp3, m4a, ogg). Output: a 16-bit WAV download.
- The output must be EXACTLY the same number of samples as the input.
  Not approximately. Exactly. If it drifts, every edit already on my
  video timeline moves.
- Before doing anything to the audio, measure it and show me the
  numbers. After processing, show the same numbers again so I can see
  what changed. A tool that can't tell me what was wrong can't tell me
  whether it helped.

Controls: target loudness (-14 / -16 / -19 / -23 LUFS), a strength
slider, a section-length slider, and a toggle for section matching.
Before and after playback, side by side.
```

---

## Prompt 2 — the measurements

```
Implement the analysis pass properly, not with rough approximations.

1. Integrated loudness in LUFS to ITU-R BS.1770: K-weighting
   (high-shelf + high-pass), 400 ms blocks with 75% overlap, absolute
   gate at -70 LUFS, then the relative gate at -10 LU below the
   ungated mean.

2. Noise floor: take the 1st percentile of RMS measured in 100 ms
   windows. Do not use a longer window. A 400 ms or 1 s window catches
   quiet speech and reports it as the floor, which is how you end up
   "fixing" a file that was never noisy.

3. Tonal balance in 7 bands (roughly 60, 150, 400, 1k, 2.5k, 6k, 12k)
   using an FFT of the whole signal, reported in dB relative to the
   overall level. This is the fingerprint of the mic and the room.

Report all three for the whole file AND for each section
independently, where a section is a fixed block of N seconds.
```

---

## Prompt 3 — section matching

```
Here is the part no one-click enhancer does.

My narration is recorded in chunks, on different days, sometimes with
the background HVAC on and sometimes off. Loudness matching alone does
not fix it — the listener still hears the room change under the voice.

So: split the file into sections, measure each one on its own, and pull
every section toward the average of all of them. Three things have to
converge:

- loudness, to under 1 LU of spread
- noise floor, to under 6 dB of spread
- each of the 7 tonal bands, to under 2.5 dB of spread

Use an expander (not a denoiser) for the floor — most of what is left
in a quiet recording is breath, and spectral denoisers barely move it
while doing audible damage to the consonants.

Then, so the background never changes at all, mix one single layer of
gentle room tone under the entire file at a level just above the
worst remaining section floor.

Iterate the matching. One pass overshoots, because correcting the
floor changes the tonal balance and correcting the tonal balance
changes the floor.
```

---

## Prompt 4 — the correction

This is the one that mattered.

```
The tests say the sections are still 26 dB apart after matching. The
bug is in the approach, not the code.

Right now each section is expanded by an amount scaled to its own
noise level — the noisy one gets a big reduction, the clean one gets a
small reduction. That is proportional, so it preserves the gap. Two
sections 26 dB apart, both reduced by the same proportion, are still
26 dB apart.

Rewrite it so every section is aimed at the SAME ABSOLUTE floor
target. Solve for the expander ratio each section needs to land on
that number, clamp the ratio to a sane range, and iterate up to three
times. The dirty sections should end up doing a lot of work and the
clean sections should end up doing almost none.

Also check the room tone level. I asked for -52 dB and the measured
output is -68. The tone is being generated and then passed through a
filter, and the filter changes the level of what goes through it.
Measure the tone after filtering and scale it to hit the requested
number.
```

---

---

## Prompt 5 — the long-file correction

Found only by testing a 44-minute file. Everything passed at 9 minutes.

```
On a 44-minute file the tonal matching stalls at 3.6 dB of spread and more
iterations don't help — 14 passes land in the same place as 5. On the
9-minute file the same code reaches 2 dB.

Before changing the matcher, check the measurement it depends on: measure
the same 30 seconds of audio twice with the analysis grid shifted by 50 ms.
Identical audio should give identical numbers. If it doesn't, the matcher
has been correcting measurement noise, and a long file with many sections
simply has more noise to chase.

If that's the cause, make the measurement stable: many more analysis frames
covering the whole section rather than a few sparse ones, and skip
near-silent frames so a section's result depends on the voice, not on how
much silence it happens to contain.

Separately: the file must not be copied at every processing step. A
79-minute file is 840 MB as floating point, and one copy per filter is a
crash. Filter in place wherever a sample is read before it is written, and
give the loudness meter and the limiter streaming implementations that
never allocate a full-length buffer. The limiter's look-ahead is also an
O(n x window) loop — replace it with a sliding-window maximum.

Add a "start of file vs end of file" figure to the results so the user can
see there is no drift, rather than being told there isn't.
```

What that found: identical audio measured **1.0 dB apart on average, up to
3 dB** in some bands. The matcher had been applying real EQ to imaginary
differences. After the fix, 0.1–0.4 dB.

---

## What the tests measured

On a synthetic file built with four deliberately mismatched sections:

```
loudness spread    5.38  ->  0.24 LU
noise floor spread 32.0  ->  1.4  dB
worst tonal band    9.6  ->  0.9  dB
```

On 8 minutes 47 seconds of real narration recorded as ten separate takes:

```
loudness spread     2.04  ->  0.03 LU
noise floor spread 39.4   ->  2.2  dB
worst tonal band   13.5   ->  0.9  dB
```

Long files, single-file build, all network requests blocked:

```
44 minutes    14/14 checks   106 s   712 MB peak heap
79 minutes    14/14 checks   196 s   1.2 GB peak heap
```

62 automated checks across six suites, every one run with network requests blocked and counted.
