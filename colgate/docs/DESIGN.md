# How Colgate works

Colgate started from three public descriptions of perceptual equalisation: an operation manual, an interview with a developer in the field, and patent application US 2005/0065781. Versions 1.1 to 1.4 were then tuned by measurement against 57 renders of six songs through Gullfoss: all six in its Master mode and four of them in its standard mode. Neither that plug-in's code nor its formulas were available, so this is an independent model fitted to what its output does to the audio. It is close on that material, and it isn't a copy.

Measuring changed the design, sometimes against what the texts suggested.

## What the renders showed

| Measured in the reference | Colgate does | Where |
| --- | --- | --- |
| The EQ is minimum phase. Phase matches the Hilbert transform of log magnitude within about 1 degree. | The log magnitude and its Hilbert transform are built per band and added up. | `ColgateEngine::buildResponseBasis`, `runHop` |
| Both channels get one curve. Gains on left, right, mid and side agree. | One curve from the mean power of both channels. | `ColgateEngine::runAnalysisHop` |
| Recover and Tame add up. On all six pairs measured, 50/50 is the sum of 50/0 and 0/50 to within 0.05 dB RMS, frame by frame. | Each control contributes its own part, and they are summed. | `PerceptualField::compute` |
| Each control both lifts and cuts. Recover alone dips the mids by about 0.7 to 1 dB and Tame alone lifts the treble by about 1.4 to 2 dB. The manual says the same: the combination keeps perceived loudness. | One broadband gain follows the two controls. | `keepLoudnessDb` |
| That broadband gain holds the sum over the bands of power^0.35 constant, with no ear weighting. Frame by frame this stays within about 0.2 dB RMS on three of the four songs with those renders. On the other, 95% of frames are within 0.5 dB, and a few outliers lift its RMS to 1.7 dB. It follows the music over time: correlation about 0.99 for Tame alone and 0.84 to 0.92 for Recover alone. | The same rule, solved every hop. | `keepLoudnessDb` |
| The gain depends on a band's level relative to the total and to its near neighbours. Wider scales add nothing once the broadband gain is modelled. A masking gradient, tonality, pitch and rhythm layers add under 1%. | Two weights and a floor at -60 dB. The masking model and both layers are gone, although the patent describes them. | `PerceptualField::compute` |
| Recover and Tame have different zero points across the spectrum. In the bass both act on the same band at once. Tame moves further than Recover for the same change in level, about 1.8 times in Master. | A static curve per control, a soft knee at its zero, and a scale for Tame's movement. | `ModeTuning` |
| Bias has no effect with Recover equal to Tame. With them apart it moves the curve modestly, and differently from song to song. | Bias enters only through the difference between the two, so the first is exact. For the second it moves a line and softens a knee, by amounts fitted per mode, which explains only part of the change. | `PerceptualField::compute` |
| Above 50%, gain grows far more slowly than below it: by an exponent of about 0.63 to 0.65, against about 1.05 to 1.1 up to 50%. | A second exponent above 50%. Measured on one song only, up to 110%. | `amountScale` |
| Brighten is a tilt around 900 Hz that levels off with amount. Upward it levels off quickly in the treble and calms the treble's movement. Downward it's a different shape: the cut keeps deepening up to about 8 kHz, and it barely touches the low side directly. | A separate, nearly straight slope for negative Brighten, in a form that stays smooth through zero. | `PerceptualField::compute` |
| Gains ease off above about 8 kHz, including the broadband gain, however the signal behaves. | The same fade, by frequency. | `kFadeKnotZ` |
| Boost follows the ISO 226 contour difference closely, about 1.2 times as large in both modes at -10, +25 and +30. Only above +30 does it grow faster than the contours do, by (Boost / 30)^0.16. It keeps rising above the standard's 12.5 kHz. | The contour difference, times a scale per mode and that growth, plus a measured treble correction. | `ColgateEngine::updateBoostCurve` |
| Boost holds a different measure: the sum of power^0.5. That reproduces how far the mids drop on each song, within about 0.7 dB at +50 and 0.2 dB at +30. | A second broadband gain for Boost alone. | `ColgateEngine::runAnalysisHop` |
| Energy reacts over 100 to 150 ms. The broadband gains read a faster rise and a slower fall. | Band energy rises with a 120 ms time constant and falls with 40 ms. The broadband gains read a copy that rises with 30 ms and falls with 80 ms, the best of those tried. | `ModeTuning` |
| Master differs mainly in the 3 to 10 kHz area, where Recover lifts about 0.5 to 1 dB less and moves less. Boost is close to the same in both. | A separate set of constants for each mode. | `kBasic`, `kMaster` |

Earlier versions got some of this wrong. Version 1.0 used a masking model and a linear-phase filter. Version 1.1 used a single field and treated Recover and Tame as one line through it, which was fine with them equal and wrong with them apart. Version 1.2 made Master's Boost about twice as strong as it should be, because two renders labelled Boost +30 were really at +50. The renders of songs 4 and 5 showed that, and the earlier two fit +50 far better than +30. Version 1.3 ran too hot above 50%, and its negative Brighten levelled off in the treble, which song 6 showed.

## From the sources to the code

| The source says | Colgate does | Where |
| --- | --- | --- |
| Time and frequency relate like position and momentum. A distribution from the phase-space form of quantum theory is used instead of an FFT. | A smoothed pseudo Wigner-Ville distribution, a member of Cohen's class. Publicly known mathematics from Wigner (1932), Moyal (1949) and Cohen (1966). | `PhaseSpaceAnalyzer` |
| Analyse with logarithmic resolution, like the ear (patent [0027]). | Three lag lengths: about 43 ms below 500 Hz, 11 ms up to 3.5 kHz, 3 ms above. | `PhaseSpaceAnalyzer` |
| Bias classifies the range: positive gives Recover more space, negative gives Tame more. | A line moved in the field and a softer knee on one side, acting on the difference between the two controls. | `PerceptualField::compute` |
| Boost is a global parameter that simulates listening level, in dB, and keeps perceived loudness. | Works without Recover or Tame and ignores the range limiters. | `ColgateEngine::updateBoostCurve` |
| Master has fixed best quality and finer control at the same latency. | Master always runs at the Best overlap, shows a decimal more and drags slower unless Shift is down. | `wantedQuality`, `setFineMode` |

## Signal flow

```
                    +-------------------------------------------------+
 main or sidechain  |  phase-space distribution  ->  band energies    |
 ------------------>|  relative level -> Recover part + Tame part     |
                    |  Brighten tilt, broadband gain, top fade         |
                    |  Boost curve and its own broadband gain          |
                    +------------------------+------------------------+
                                             | one set of band gains
                                             v
 main ----> Hann frames ----> minimum-phase filter ----> overlap-add ----> Gain ----> out
   |                                                                    ^
   +-------------------------- delay of the same latency -------------- + crossfade for Bypass
```

The analyser never touches the audio. With every control at zero the engine crossfades to the delay line, which makes that state bit-exact. Latency is one analysis window: 256 samples at 48 kHz, 5.3 ms.

## How close it is

All numbers compare with Gullfoss. The songs are a 26-second clip, two 12-second clips, two 25-second clips and an 18-second clip. Basic has songs 1, 2, 3 and 6.

**The fair test: songs the fit never saw.** Each song is scored with constants fitted only on the other songs, through the finished engine's actual audio, and the table averages over songs 1 to 5 and the settings of each kind. R² is the share of the reference's per-band gain, frame by frame, that the model explains. *Curve* is the error of the average EQ curve. *Movement* is the correlation of the gain's movement around that average. *Total* is the RMS gain difference per band and frame, and *dry* is the same for no processing at all. *Null* is the output minus the render, relative to the render minus the dry signal, after correcting any delay of a fraction of a sample between them, so 0 dB would mean the plug-in got nothing right.

| Setting | Mode | Renders | R² | Curve | Movement | Total | Dry | Null |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 15/15 | Basic | 1 | 0.86 | 0.07 dB | 0.90 | 0.17 dB | 0.49 dB | -6.3 dB |
| 50/50 | Basic | 3 | 0.92 | 0.24 dB | 0.92 | 0.48 dB | 2.14 dB | -8.7 dB |
| Equal, with Bias and Brighten | Basic | 6 | 0.88 | 0.21 dB | 0.87 | 0.33 dB | 1.18 dB | -8.3 dB |
| Recover 50 alone | Basic | 2 | 0.92 | 0.28 dB | 0.88 | 0.43 dB | 1.68 dB | -7.5 dB |
| Tame 50 alone | Basic | 2 | 0.87 | 0.31 dB | 0.85 | 0.38 dB | 1.21 dB | -6.7 dB |
| Boost +30, with 15/15 | Basic | 2 | 0.98 | 0.37 dB | 0.94 | 0.60 dB | 5.13 dB | -15.3 dB |
| 15/15 | Master | 3 | 0.87 | 0.27 dB | 0.80 | 0.23 dB | 0.73 dB | -6.8 dB |
| 50/50 | Master | 5 | 0.91 | 0.39 dB | 0.88 | 0.56 dB | 2.19 dB | -8.1 dB |
| Equal, with Bias and Brighten | Master | 7 | 0.86 | 0.25 dB | 0.81 | 0.29 dB | 0.90 dB | -7.3 dB |
| Recover 50 alone | Master | 4 | 0.91 | 0.32 dB | 0.74 | 0.46 dB | 1.56 dB | -7.0 dB |
| Tame 50 alone | Master | 4 | 0.85 | 0.39 dB | 0.85 | 0.44 dB | 1.26 dB | -7.0 dB |
| Recover 50 alone, Bias +50 | Master | 2 | 0.87 | 0.44 dB | 0.70 | 0.56 dB | 1.65 dB | -6.0 dB |
| Boost +30 or +50, with 15/15 | Master | 4 | 0.97 | 0.70 dB | 0.90 | 1.07 dB | 7.47 dB | -13.4 dB |

**Version 1.3 against 1.4, on song 6.** This song added the first renders above 50%, the first of negative Boost and of Boost alone, and the first of negative Brighten at high amounts. The 1.3 column is that version's audio from a Mac, and it had never heard this song. The 1.4 column uses a tuning fitted without this song, with one exception: this song is the only or main source for the constants for amounts above 50%, Bias and negative Brighten, so those five come from its renders in the other mode. That's a fair check of those constants on new renders, but not on a new song.

| Setting | Mode | R² 1.3 | R² 1.4 | Total 1.3 | Total 1.4 | Dry | Null 1.3 | Null 1.4 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Recover 100, Bias 100 | Basic | 0.34 | 0.65 | 2.23 dB | 1.62 dB | 3.18 dB | -3.9 dB | -5.9 dB |
| Tame 100, Bias 50 | Basic | 0.79 | 0.77 | 0.65 dB | 0.68 dB | 1.60 dB | -10.4 dB | -10.1 dB |
| 110/100, Bias 35 | Basic | 0.74 | 0.85 | 2.04 dB | 1.59 dB | 4.70 dB | -6.1 dB | -8.8 dB |
| 110/100, Bias 35, Brighten -40 | Basic | 0.04 | 0.78 | 3.10 dB | 1.48 dB | 3.20 dB | -3.0 dB | -8.3 dB |
| 17/15, Bias 34, Brighten -16, Boost -10 | Basic | 0.85 | 0.92 | 0.56 dB | 0.42 dB | 1.62 dB | -9.3 dB | -15.7 dB |
| Boost +25 alone | Basic | 0.96 | 0.96 | 0.66 dB | 0.69 dB | 3.71 dB | -19.1 dB | -17.4 dB |
| Recover 100, Bias 100 | Master | 0.87 | 0.87 | 0.83 dB | 0.82 dB | 2.48 dB | -6.9 dB | -5.5 dB |
| 110/100, Bias 35 | Master | 0.73 | 0.90 | 1.91 dB | 1.17 dB | 4.15 dB | -6.0 dB | -9.8 dB |
| 110/100, Bias 35, Brighten -40 | Master | 0.10 | 0.79 | 3.06 dB | 1.48 dB | 3.25 dB | -3.4 dB | -9.0 dB |
| 17/15, Bias 34, Brighten -16, Boost -10 | Master | 0.85 | 0.89 | 0.52 dB | 0.44 dB | 1.50 dB | -9.2 dB | -15.1 dB |

Three of this song's Gullfoss renders didn't match their labels. Basic's Recover 16, Tame 14 fits best with Boost +25 still on, so it's used that way. Master's Boost +25 is dry, and Master's Tame 100, Bias 50 looks like a Recover render, so both are left out.

The tuning for amounts up to 50% hasn't changed since 1.3. Fitting it again with this song in made Basic worse on the earlier songs, with 50/50 falling from 0.92 to 0.87, because renders with large gains dominate a joint fit.

**The shipped constants, fitted on all the songs,** so this flatters it a little:

| | Renders | R² | Curve | Movement | Total | Dry | Null |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Basic, no Boost | 18 | 0.88 | 0.33 dB | 0.88 | 0.54 dB | 1.80 dB | -7.9 dB |
| Basic, with Boost | 5 | 0.96 | 0.29 dB | 0.91 | 0.60 dB | 3.85 dB | -16.0 dB |
| Master, no Boost | 29 | 0.88 | 0.37 dB | 0.82 | 0.46 dB | 1.54 dB | -7.6 dB |
| Master, with Boost | 5 | 0.96 | 0.48 dB | 0.90 | 0.87 dB | 6.28 dB | -14.0 dB |

The null's best case is -18.9 dB, on song 6 with Boost +25 alone, and its worst -3.1 dB, on song 3 with Tame alone. Until version 1.3 these scores lined up each render with a plain correlation, which a large bass lift can drag more than 100 samples off, so older published nulls on Boost shouldn't be compared with these.

`tools/calibration` repeats the fit and these scores from the audio files, and its constants come out close to the shipped ones rather than identical.

## What the tests check

All of this runs on every build, using synthetic test material so no audio files are needed: `tests/engine_tests.cpp` and `tests/plugin_tests.cpp`. Values are from x86_64 Linux and may differ slightly on Apple silicon. The test mix has drums, bass, a pad, a voice-like lead and top-end air, with stereo placement.

| Promise | Result |
| --- | --- |
| All controls at zero is no processing | Bit-exact, at 16, 44.1, 48, 96, 192 and 384 kHz, mono and stereo |
| Reported latency is true | The impulse response peaks there, and less than -59 dB of its energy comes earlier, even at the most demanding setting tested |
| EQ is minimum phase | The anti-causal part of its cepstrum is below -120 dB |
| The graph shows the truth | Within 0.01 dB of the measured gain, and within 0.05 dB for the copy the display reads |
| Recover and Tame add up, as in the reference | 50/50 is the sum of 50/0 and 0/50 within 0.02 dB |
| The broadband gain does its job | The sum of power^0.35 holds to 0.13 dB |
| Stems add up to the processed mix (sidechain) | Null at -129 dB |
| Block size doesn't matter | Identical down to 1 sample |
| Both channels get one curve | A level difference between them passes unchanged, to -121 dB |
| Modes share a shape and a latency | Average curves correlate at 0.985, differ by at most 1 dB, and latency is equal |
| Switching mode while playing | No click: the largest step stays within 1.1 times the sine's own |
| Bias does nothing with Recover equal to Tame | 0.00 dB, as in the reference. With them apart it moves the curve |
| A masked tone is uncovered | A tone 32 dB under a neighbour 400 Hz below rises about 5 dB while the masker falls about 6.3 dB, at 100/100 |
| Boost makes room in the mids | -7.0 dB in Basic and -6.8 dB in Master at Boost +30 on pink noise, and -16.4 dB and -15.9 dB at +50 |
| Pink noise isn't reshaped | Largest band gain 0.9 dB at 100/100, not counting the lowest two bands |
| Entering a number | A double click selects the whole number, so typing replaces it, and Enter sets exactly that. Option-click returns any number or range limiter to its original value |
| Better resolution than a spectrogram | Valley between partials 188 Hz apart: 5.7 dB, against 1.2 dB for a Hann spectrogram of the same span |
| No allocation in the audio path | Counted, including moving every control |
| Bad input is contained | NaN and infinity never reach the output, and audio continues, in both modes |
| Runs in real time | One stereo instance on a 2.8 GHz Xeon cloud core: about 18 times real time at Normal (6% of a core), 10 times at Better, 5 to 6 times at Best and in Master (about 18%) |

The finished VST3 is also loaded through its public interface by a small host (`tools/vst3_host_check.cpp`), which checks parameters, state, audio and the editor.

## Limits worth knowing

- **Six songs is still six songs.** Basic has four, and only three of them with Boost. Leaving one out at a time gives a fair estimate for music like these, and says less about very different styles, levels or arrangements.
- **High amounts rest on one song.** Everything above 50%, most of what's known about Bias with the controls apart, and negative Brighten beyond -16 come from song 6. At 110/100 the lowest bands still lift about 3 to 4 dB too much, and Colgate's peak with Brighten -40 is about 3 dB over the reference's.
- **Settings nobody measured.** Amounts above 110%. Boost at values other than -10, +25, +30 and +50, anything below -10, and Boost alone in Master. Bias below zero with the controls apart. Brighten beyond +50 or below -40. Those may differ from the reference.
- **Nobody has listened to it.** The match is in numbers, and error can sit in what the numbers miss.
- **It moves too little at low settings.** At 15/15 its gain moves about 0.7 times as much as the reference in Basic and about 0.6 times in Master, though the average curve is right. This setting is the one most people will use, so it's the first thing to check by ear. Counting low settings more heavily in the fit didn't help on unseen songs.
- **Its movement runs late above the bass.** On nearly every render the gain in the mids and treble moves about 10 to 20 ms after the reference's, while the bass is on time. Shorter time constants close most of that gap, but no score improved when they did, so they stayed.
- **The Mac build gets partial testing.** Version 1.3 in Logic agreed with this Linux build's gain to 0.12 dB RMS at 110/100, on the one render checked that closely. Version 1.4's Mac build hasn't been run yet. GitHub Actions builds it on an Intel Mac and runs the tests, auval and pluginval there, so its Apple silicon half is only tested if you start that job by hand.
- **Sparse material.** The field was fitted on full mixes. At 15/15 a lone sine stays within 0.8 dB, but at 100/100 it loses 4 to 6 dB, partly because the analyser's leakage counts as energy in the loudness rule. At 110 Hz, sidebands sit about 41 dB below the tone at 100/100 and 65 dB below at 15/15. The reference's behaviour on this isn't known. A single guitar or vocal is the next thing worth measuring.
- **Loudness on a meter can move.** The rule that matches the reference isn't BS.1770. On the synthetic mix that means 0.6 LU at 15/15, 2.2 LU at 50/50 and 3.2 LU at 100/100. In its own renders the reference moved up to 1.5 LU at 50/50, and without Boost Colgate tracked it to within 0.4 LU up to 50% and within 1 LU above that.
- **Boost's peaks run high.** Colgate's peak is above the reference's on all eight renders with positive Boost, by 0.3 to 3.5 dB. The worst is song 3 in Master at +50: +1.3 dBFS against -2.2 dBFS. The average gain is close. At the bass hits examined, the reference's broadband gain drops sooner. A broadband gain that falls more slowly narrowed the gap, but slowing it further hurt the match. Watch the output when using Boost. Above 40 dB per band, Boost's curve rises at a quarter of the rate.
- **The deep bass can't be shaped finely.** The filter can't form details narrower than about 45 Hz either side, so below about 100 Hz the EQ acts as a broad move. The graph shows this as it really is.

## Not included

Windows and AAX builds, and the low-latency edition that the manual describes.

## Where to change things

- **Version:** the `project(Colgate VERSION ...)` line in `CMakeLists.txt`.
- **The tuning of each mode:** `kBasic` and `kMaster` in `source/engine/PerceptualField.cpp`. `tools/calibration` prints new values from new renders.
- **Presets:** `kFactoryPresets` in `source/plugin/Parameters.h`.
- **Colours and type:** `source/plugin/Theme.h`.
- **See what the engine does to a signal:** `colgate_measure` and `colgate_render`.
