# Voice Leveler

**A voice cleanup and levelling tool that runs entirely in your browser. No upload, no account, no length cap.**

👉 **[Open it](https://JonathanGeorgeGatz.github.io/voice-leveler/)** — or download `index.html` and double-click it.

---

## What it's for

Most one-click "enhance voice" features do a single pass over your whole file with one setting. That works if the whole thing came off one microphone, in one room, on one afternoon.

Narration usually doesn't. It gets recorded in chunks, on different days, with the HVAC on and then off. Matching the loudness of those chunks isn't enough — the listener still hears the *room* change underneath the voice, and the result sounds cheap without anyone being able to say why.

Voice Leveler measures every section of your file separately and pulls them toward each other on three axes at once:

| | what it is | target |
|---|---|---|
| **Loudness** | how loud it feels (LUFS, the broadcast scale) | under 1 LU of spread |
| **Noise floor** | how loud the background is between words | under 6 dB of spread |
| **Tonal balance** | the fingerprint of the mic and room, in 7 bands | under 2.5 dB of spread |

Then it mixes one single layer of gentle room tone under the whole file, so the background never changes at all.

## Measured results

Run on real narration — ten separate takes concatenated, 8 minutes 47 seconds:

```
loudness spread      2.04  ->  0.03 LU
noise floor spread  39.4   ->  2.2  dB
worst tonal band    13.5   ->  0.9  dB
output              -16.1 LUFS, peak -1.2 dBFS
processing          28 s for 527 s of audio   (~19x realtime)
samples in / out    23,238,936 / 23,238,936   (exact)
```

That 39 dB noise-floor spread is what "different rooms, different mics" sounds like. Two decibels is what one room sounds like.

And because "no length cap" has to mean something:

```
44-minute file    14/14 checks    106 s    start-vs-end drift 0.18 dB
79-minute file    14/14 checks    196 s    start-vs-end drift 0.12 dB
```

## Why browser-only matters

- **No length cap.** A two-hour file is a longer array. That's all.
- **Nothing is uploaded.** Your audio is read by your browser and never leaves the machine. Unplug your internet — it still works.
- **No meter.** It costs nothing per minute because nothing runs on anyone's server.
- **Sample-exact output.** What comes out is the same length as what went in, down to the sample, so every edit already sitting on your timeline stays where it is.

## Using it

1. Open `index.html` (or the hosted link above).
2. Drag in a WAV, MP3, M4A or OGG file.
3. Read the BEFORE numbers.
4. Pick a target loudness, press **Analyse and enhance**.
5. Compare the two players, then download the WAV.

**Target loudness:** -14 for YouTube, -16 for podcasts and voiceover, -19 for quiet listening, -23 for EBU R128 broadcast.

**Section length:** set it near the length of your recording takes. Longer is safer; shorter corrects faster-changing problems.

**Match sections:** leave it on if the audio was recorded in more than one sitting. Turn it off for a single continuous take.

## What it will not do

It filters; it does not reconstruct. It will not remove a lawnmower, rescue a phone recorded in a restaurant, or separate two people talking over each other.

For those, use [Adobe Podcast Enhance Speech](https://podcast.adobe.com/enhance) — it's free, it uses a trained model that rebuilds speech rather than filtering it, and on genuinely bad recordings it will beat this comfortably. The open-source equivalents ([Resemble Enhance](https://github.com/resemble-ai/resemble-enhance), [DeepFilterNet](https://github.com/Rikorose/DeepFilterNet)) are excellent too, and mean installing Python and finding a GPU.

Voice Leveler is for a *decent* recording that keeps changing rooms. That is a different problem, and nothing else seems to solve it.

## How it was built

Every line of this was built with AI, and the exact prompts are in [PROMPTS.md](PROMPTS.md) — including the fourth one, which is the correction after the first approach turned out to be wrong.

The short version of that mistake: each section was being expanded by an amount scaled to its own noise level. That's proportional, so it *preserves* the gap it's supposed to close. Two sections 26 dB apart, both reduced by the same proportion, are still 26 dB apart. Sections only converge if every one of them is aimed at the same absolute number.

Built for **Build It With AI**, Episode 2.

## Under the hood

ITU-R BS.1770 loudness (K-weighting, 400 ms blocks, 75% overlap, absolute and relative gating), noise floor as the 1st percentile of 100 ms RMS windows, 7-band FFT tonal profile, an envelope-follower expander solved against an absolute floor target, iterative spectral matching, compressor and look-ahead limiter.

62 automated checks across six suites — DSP core, page, single-file build, edge cases, a real 8:47 recording, and 44- and 79-minute files — every one run with all network requests blocked and counted, because a tool that claims to work offline should be tested that way rather than described that way.

The release download includes `test_fixture.wav`, an 80-second file with four deliberately mismatched sections. Run it through and watch the spread columns collapse.

## License

MIT. Yours to keep, change, sell, or rebuild. It's one HTML file and it's all commented — open it in a text editor.
