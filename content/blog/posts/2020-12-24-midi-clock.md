---
title: "Calculating BPM: The Mysterious 24"
date: 2020-12-13T16:44:49-08:00
Description: "Measuring BPM via Midi Clock"
Tags: ["Javascript", "Midi", "Node",]
Categories: ["code"]
draft: false

---

I've been working on a Web Midi project lately that relies on the BPM of the midi device. In my case, the OP-Z as a midi source.


The MIDI clock is not tied to the sequencer play/stop and is always being outputted. As I could not think of another way to retrieve the BPM, I decided to measure it from the clock.

----
## Measuring BPM

```js
// First clock input
const clock_1 = new Date();

// Second clock input
const clock_2 = new Date();

// Period (s)
const period = (clock_2 - clock_1) / 1000;

// Frequency (Hz)
const hz = 1 / period;

// Frequency per minute
const bpm = hz*60;
```

At this stage I expected to have the BPM, but **what I was seeing on my midi device was off by a factor of 24**.

Why 24? Was it device specific?

It took me a while to figure this out (and so I had a variable named `MAGIC_NUM` for a while), but I was pointed to "PPQ" by my friend Drew.

[Pulses Per Quarter Note (wiki)](https://en.wikipedia.org/wiki/Pulses_per_quarter_note)

Of importance is:

> Purposefully quantised music can have resolutions as low as 24 (the standard for Sync24 and MIDI, which allows triplets, and swinging by counting alternate numbers of clock ticks)

So, the BPM is really:

```js
const bpm = hz*60/24;
```
----
## Getting accurate BPM

Simply the method above yeilds fluctuations. In otherwords, if the BPM is 120 then you'll get readings like 119, 121, 122, etc.. It's close, but not perfect.

**Solution: keep a buffer and average the values.**

In the end, a consistent result required a buffer size of between 40-50. Otherwise, it would still fluctuate.

The following is what I ended up with:

```js
const MINUTE_SECONDS = 60;
const PPQ = 24;
class MidiClock {
  constructor(bpm = 120, bufferSize = 50) {
    this.buffer = Array(bufferSize).fill(bpm);
    this.stamp = new Date();
  }

  // 1. Record time stamp
  // 2. Calculate BPM
  // 3. Add BPM to buffer
  tick() {
    const _s = new Date();
    const hz = 1000/(_s-this.stamp);
    const bpm = MINUTE_SECONDS*hz/PPQ;
    this.stamp = _s;

    this.buffer.push(bpm);
    this.buffer.shift();
  }

  // Use average value of buffer to calculate BPM
  bpm() {
    return Math.floor(this.buffer.reduce((a,b) => a + b, 0)/this.buffer.length);
  }

}

export default MidiClock;
```

Usage:

```js
const t = new MidiClock();

// When a clock note is received
t.tick()

// BPM
t.bpm()
```

# Conclusion

Don't forget 24! At least with midi.




