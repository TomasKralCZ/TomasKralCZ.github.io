---
layout: post
title:  "Some notes about the NES APU"
tags: NES, audio
---
I recently implemented APU emulation in my NES emulator [Fearless-NES](https://github.com/TomasKralCZ/Fearless-NES) - one of my favourite hobby projects to work on.

The APU is not too complicated - my implementation consists of ~1200 LOC, ~300 of those being comments. The tricky part is actually understanding how it works.

The [NESdev wiki](https://www.nesdev.org/wiki/APU) contains extensive documentation of the inner-workings of the APU. The wiki precisely describes how all of the APU's components work, but doesn't necessarily explain the big picture.

This blog post is meant to be a sort-of "what I wish I had known about the APU before I started trying to emulate it" post.

Please note that I'm not an audio expert nor do I have a perfect understanding of the APU.

# Audio basics

I'm going to assume that you have at least rudimentary knowledge of how audio works. If not, [this video by Displaced Gamers](https://youtu.be/mJnz6dEWwIw) should be a good starting point.

# Channels

The APU contains 5 (mostly) independent audio channels: 2 "pulse" channels, a "triangle" channel, a "noise" channel and a delta modulation channel (DMC).

The pulse channels output a square waveform. The triangle channel outputs a "triangle" waveform imitating a sine wave. The noise channel outputs a pseudorandom wave, used for percussive sounds. The DMC channel outputs recorded samples.

Every channel contains a **timer**, which periodically advances the channel's position in the waveform. For example the triangle channel contains an internal *output level buffer*:
{% highlight c %}
[15, 14, 13, 12, 11, 10,  9,  8,  7,  6,  5,  4, 3,  2,  1,  0, 0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11, 12, 13, 14, 15].
{% endhighlight %}

The same buffer plotted:

![TriangleChannelOutputPlot](/assets/triangle-plot.svg){: width="250" }

Every time the triangle's timer *clocks*, it changes the channel's current output level to the next value in the buffer (if it's at the end of the buffer, it simply loops back to the start). The more often this *clock* happens, the faster the output level changes and the higher the channel's pitch is.

The other channels work in a simillar (although a bit more complicated) manner.

## Timers

Every timer needs to maintain two pieces of information:
1. Current timer value
2. Reload value

The timers are *ticked* on every CPU cycle with the exception of the Pulse channels' timers which are *ticked* every *APU cycle* (1 APU cycle = 2 CPU cycles).

Every time a timer is *ticked* one of 2 things happen:
1. Current timer value is 0. This is when the timer *clocks*. The current timer value is set to the reload value and some *action is performed* (such as changing the triange's channel output level).
2. Current timer value is > 0. The current timer value is decreased by 1.

The reload value is what is set by memory-mapped registers such as (`0x4002, 0x4006` etc..).

# The Mixer

On every *CPU clock*, the mixer takes the output of all of the channels and mixes them together into one sample. The way the output level is calculated for the other channels is going to be explained later.

# The Frame Counter

The job of the frame counter is to periodically update some properties of the channels. It is clocked on ever *APU cycle* and maintains a counter of these clocks. When specific values are reached, the frame counter *clocks* components such as length counters, envelopes and sweep units.

# The channels in-depth

## The pulse channels

The pulse channels output a square wave and are slightly more complicated than the triangle channel. The basic shape of the pulse channel's waveform is determined by the currently selected output waveform, which is selected by the 2 most-significant bits of the 0x4000 / 0x4004 register. The "currently selected waveform" is called the "duty cycle" in audio lingo.

| Selected waveform (duty cycle) | Output waveform   |
|--------------------------------|-------------------|
| `0`                            | `0 1 0 0 0 0 0 0` |
| `1`                            | `0 1 1 0 0 0 0 0` |
| `2`                            | `0 1 1 1 1 0 0 0` |
| `3`                            | `1 0 0 1 1 1 1 1` |
