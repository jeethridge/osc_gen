# osc_gen

[![CircleCI](https://circleci.com/gh/harveyormston/osc_gen.svg?style=shield)](https://circleci.com/gh/harveyormston/osc_gen)
[![PyPI version](https://badge.fury.io/py/osc-gen.svg)](https://badge.fury.io/py/osc-gen)

# Table of Contents

<!-- vim-markdown-toc GFM -->

* [About](#about)
* [Installation](#installation)
* [Development](#development)
* [Getting Started](#getting-started)
  * [Morphing Between Waveforms](#morphing-between-waveforms)
  * [Generating Your Own Waves](#generating-your-own-waves)
  * [Pulse-width Modulation](#pulse-width-modulation)
  * [Processing Waveforms](#processing-waveforms)
* [Using Samples](#using-samples)

<!-- vim-markdown-toc -->

# About

osc_gen is a Python library for creating and managing oscillator wavetables.

Functionality includes:

* Generating common waveforms (sine, saw, square, etc.)
* Oscillator effects (waveshaping, distortion, downsampling, etc.)
* Resynthesising or slicing audio from wav files or other sources
* Saving wavetables to a wav file for use in samplers
* Saving wavetables in .h2p format for use in the u-he Zebra2 synthesiser


# Installation

osc_gen is [available on PyPI](https://pypi.org/project/osc-gen/#description) and can be installed using pip:

```sh
$ pip install osc_gen
```

# Development

Development requirements can be installed using the provided `requirements.txt`:

```sh
$ pip install -r requirements.txt
```

# Getting Started

These examples show how to:

- Use the sig module to generate oscillator shapes.
- Use the wavetable module to create a 16 slot wavetable.
- Use the zosc module to store a wavetable as a zebra oscillator file.

```python
import wavetable
import zosc
import sig
import dsp


# Create a signal generator.
sg = sig.SigGen()

# Create a wave table with 16 slots to store the waves.
wt = wavetable.WaveTable(num_waves=16)

# Generate a sine wave using our signal generator and store it in a new Wave object.
m = sg.sin()

# Put the sine wave into our wave table.
# As we're only adding one wave to the wave table, only the first slot of the
# resulting wavetable will contain the sine wave. The remaining slots will
# be empty, because we haven't added anything to those yet.
wt.waves = [m]

# Write the resulting oscillator to a file.
zosc.write_wavetable(wt, 'osc_gen_sine.h2p')

# To fill all 16 slots, repeat the sine wave 16 times in the wavetable.
wt.waves = [m for _ in range(16)]
zosc.write_wavetable(wt, 'osc_gen_saw16.h2p')
```

## Morphing Between Waveforms

We can use up all 16 slots in the wavetable, even with fewer than 16
starting waveforms, if we use morph() to morph from one waveform to the
other and fill in the in-between slots.

```python
# Morph from sine to triangle over 16 slots.
wt.waves = sig.morph((sg.sin(), sg.tri()), 16)
zosc.write_wavetable(wt, 'osc_gen_sin_tri.h2p')

# Morph from sine to triangle over 5 slots.
wt.waves = sig.morph((sg.sin(), sg.tri()), 5)
zosc.write_wavetable('osc_gen_sin_tri5.h2p')

# Morph between sine, triangle, saw and square over 16 slots:
wt.waves = sig.morph((sg.sin(), sg.tri(), sg.saw(), sg.sqr()), 16)
zosc.write_wavetable('osc_gen_sin_tri_saw_sqr.h2p')
```

## Generating Your Own Waves

You can create a custom signal yourself to use as an oscillator.
In this example, one slot is filled with random data, but you could
use any data you've generated or, say, read in from a wav file using the
wavfile module.

```python
# Generate some random data.
from random import uniform
random_wave = (uniform(-1, 1) for _ in range(128))

# Write to file.
wt.waves = [sg.arb(random_wave)]
zosc.write_wavetable('osc_gen_random.h2p')
```

The custom signal generator function automatically normaises and scales any
data you throw at it to the right ranges, which is useful.

## Pulse-width Modulation

SigGen has a pulse wave generator too. Let's use that to make a pwm wavetable.


Pulse widths are between 0 and 1 (0 to 100%).
0 and 1 are silent as the pulse is a flat line.

So, we want to have 16 different, equally spaced pulse widths, increasing in
duration, but also avoid any silence:

```python
pulse_widths = (i / 17. for i in range(1, 17))
wt.waves = [sg.pls(p) for p in pulse_widths]
zosc.write_wavetable('osc_gen_pwm.h2p')
```

## Processing Waveforms

The dsp module can be used to process waves in various ways.

```python
# Let's try downsampling a sine to produce aliasing distortion
ds = dsp.downsample(sg.sin(), 16)

# That downsampled sine from probably sounds pretty edgy.
# Let's try that again with some slew this time, to smooth it out a bit:
sw = dsp.slew(dsp.downsample(sg.sin(), 16), 0.8)

# Generate a triangle wave and quantize (bit crush) it.
qt = dsp.quantize(sg.tri(), 3)

# Apply inverse slew, or overshoot, to a square wave.
ss = dsp.slew(sg.sqr(), 0.8, inv=True)

# Overshoot might make the wave quieter, so let's normalize it.
dsp.normalize(ss)

# Morph between the waves over 16 slots and write out to a file
wt.waves = sig.morph((ds, sw, qt, ss), 16)
zosc.write_wavetable('osc_gen_dsp.h2p')
```

# Using Samples

Samples can be used to populate a wavetable using one of two methods: slicing
and resynthesis. Both methods involve finding the fundamental frequency of the
audio in the wav file and generating wavetable slots containing a multiple
single cycles of the waveform.

Slicing is relatively simple - the input audio is sliced at regular intervals
to extract individual cycles of the tone.

Resynthesis, on the other hand, uses Fourier analysis to reconstruct cycles of
the waveform based on the magnitude of the harmonics observed in the input.

Slicing gives results which will match the original audio exactly, but small
errors may result in unwanted harmonic content. Resynthesis gives more
predictable harmonic content but may discard information from the original
audio.

```python
# resynthesize
wt = wavetable.WaveTable().from_wav('mywavefile.wav', resynthesize=True)

# slice
wt = wavetable.WaveTable().from_wav('mywavefile.wav', resynthesize=False)
```
