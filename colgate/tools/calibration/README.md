# Calibration

How Basic and Master were tuned, so it can be done again with more material. No audio is kept in this repository. Keep it that way, since neither the mix nor the reference renders are ours to publish.

## Make the renders

Bounce each song through the reference plug-in at 48 kHz, once per setting, and keep them in one folder per song. Names and subfolders say what a render is. A name or folder with `master` in it counts as Master mode, and everything else as Basic:

```
ref 1/original.wav
ref 1/basic/recover 50 tame 50.wav
ref 1/basic/recover 15 tame 15 bias 15 brighten 15 boost 30.wav
ref 1/master/tame 50.wav
```

Delay between the files doesn't matter, because it's measured. Leave the plug-in's Gain at 0 dB and everything you aren't testing at its default. Check that the plug-in is really processing during each bounce: a render nearly identical to its original gets reported and left out. Check that each file has sound in it, and that every other control went back to its default before the next one.

Name each render after what was really set, because the name decides what the fit assumes. Two Master renders labelled Boost 30 were really at Boost 50, and that made Master's Boost about twice too strong in version 1.2.

## Fit and score

Build the tools (`cmake -S . -B build -DCOLGATE_BUILD_PLUGIN=OFF && cmake --build build`), install numpy and scipy, then:

```sh
# Fit, and print new constants for source/engine/PerceptualField.cpp
python3 tools/calibration/calibrate.py "ref 1" "ref 2" "ref 3" "ref 4" "ref 5" "ref 6" --build build/tests

# Compare the engine's actual audio with every render
python3 tools/calibration/calibrate.py "ref 1" "ref 2" "ref 3" "ref 4" "ref 5" "ref 6" --build build/tests --score
```

Fitting first scores each song with constants fitted only on the others, so those scores are a fair guess at how a new song would do. Then it refits on everything for the constants. It fits in two stages, because renders with large gains dominate a joint fit and pull it away from the settings most people use. Ordinary renders set most of the constants, up to 50% and without negative Brighten or Bias with the controls apart. All the renders then set the five for amounts above 50%, Bias and negative Brighten. A song held out takes its share of those five with it, so if only one song has such renders, its scores for them say little. Boost has one curve for each mode, and Basic borrows Boost's growth from Master and a fixed scale for Tame's movement, because it has too few songs and Boost values to fit them itself.

## What more renders would answer

Today's numbers come from six songs of 12 to 26 seconds, four of them in Basic. These would help most, in this order:

1. Two files from ref 5 again in Master: Boost 25 came out dry, and Tame 100, Bias 50 looks like a Recover render.
2. Amounts above 50 on other songs, such as 100/100 with no Bias, and 200/200. Only one song has been measured there, and only up to 110.
3. Boost alone at other values, such as +10, -15 and -30. So far: -10, +25, +30 and +50.
4. Bias below zero with Recover and Tame apart, and Tame alone at other Bias values.
5. Negative Brighten on other songs, such as -100, and Brighten +100. Only one song has been measured beyond -16.
6. Basic on the songs that only have Master renders so far.
7. Single sources rather than full mixes, such as one guitar track and one iPhone vocal. Sparse material is the least tested.
