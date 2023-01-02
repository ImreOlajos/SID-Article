# SID-Article v0.2

**_MOS Technology SID soundchip internals and applications on the Commodore 64_**

## What is the SID Chip?

The Sound Interface Device (or SID) soundchip contributed heavily to the success
of the Commodore 64 personal computer (usually abbreviated as _C64_). It's
unique among the soundchips of the microcomputer era with its outstanding
capabilities and sound quality. Designed in 1982 by a team led by Bob Yannes, it
featured synthesizer techniques yet unseen with other computer brands: an analog
filter, mixing abilities combined with highly flexible digital control of 3
separate analog oscillators with pitch, timbre, and volume control and with
cutoff curves.

The SID chip in essence is a digitally controlled analog synthesizer. Although a
lot can be controlled just through the digital registers of the SID chip itself,
even more is possible when a computer program is updating those registers at a
rapid rate. In modern terms you can think of these programs - on the Commodore
64 usually written in 6510 assembly language - as sophisticated sequencers that
can also utilize LFOs (e.g. for vibrato effect) and custom macro capabilities
(e.g. for wavetables).

Soon after the Commodore 64 was released the SID became a beloved playground of
great musicians making music for the burgeoning game industry, with people like
Rob Hubbard, Ben Daglish, Martin Galway, Tim Follin, just to name a few of the
early pioneers. Thanks to its capabilities the SID chip can still achieve juicy
sounds today in good hands, be it any music genre or trend, like Drum'n'Bass or
Dubstep wave. Every now and then new techniques utilizing the SID are revealed
by creative demoscene enthusiasts who create new pieces of music for games,
demos, or just for fun.

## What is This Document About?

This document covers mostly the internals of how the SID works behind the
scenes, but that doesn't mean this document is for geeks only. This document
strives to be an easy-to-read explanation, gradually increasing its complexity.
Even in the harder parts it tries to explain the concepts in the simplest
possible way, accompanied by pictures.

SID musicians who want to get a deeper understandig of the _Hard Restart_ and
other mysterious workings of the SID will probably find parts of this document
just as beneficial as music-player routine developers or implementers of
software/hardware SID-emulation engines. Whatever you use in your life, you can
utilize it better if you know its internals rather than treating it as a mere
black-box. It's up to you whether you feel curious enough to compose SID music
by instinct or whether you try to build upon a solid knowledge of the internals
of the SID chip.

There are many resourceful materials about the SID on the Internet already, but
it can be very tiresome to dig them all up from different places. There is an
overall technical document about the graphic chip of the C64 called VIC-article.
Surprisingly there hasn't been a similar document for the SID - until now. Some
of the sources used here was an interview Hermit made with Boba Yannes, the
source-code comments of Dag Lem's ReSID, the ReSID-FP engine, the Kevtris
reverse-engineering site, a document about DC levels by Levente Hársfalvi, and
Hermit's own findings that will be explained later. So hopefully this document
collects sufficient information for you in this article in one place to get the
big picture.

## Revisions and Location of the SID Chip

There are two major versions of the SID chip called by their part numbers: the
original one is called __6581__ and the newer, revised SID is called __8580__.
The 6581 can be found in the top-middle region of the old C64 mainboards and it
requires a 12V power-supply for its analog circuitry besides the 5V digital
supply. The 8580 is found slightly to the right at the bottom of new C64 boards
and it requires 9V and 5V power rails.

__TODO__: Diagram needed here

There are two external capacitors to support the filter circuits integrated into
SID. These capacitors are quite different in value for the old and new models,
therefore the SID versions are not readily interchangeable without any modding.
6581 models also prefer a 1 kOhm resistor towards ground on their output for
the simplified output-stage driver circuit. At least their 28-pin DIP package
form-factor is the same so they fit into each others' sockets without any
hassles.

Both models are nearly equivalent in their digital portions, but the different
silicon process they are based on (6581:NMOS, 8580:HMOS-II) and the different
designs result in clear differences how their analog circuits sound like,
especially in mixing, filter curves, and with combined waveforms. More on these
later. There are big differences even between 6581 revisions themselves[^1]. (A
6582 model was also released but its internals are the same as that of the 8580,
only the labels differ.)

[^1]: __TODO_: As per Lagerfeldt's findings, the differences
in 6581s are due to the filter components, not due to the chip revisions
themselves. See [Mythbusting the 6581 revisions](https://ultimatesid.dk/).

## Pinout of the SID Chip

| Pin name | Pin # (left) | Pin # (right) | Pin name  |
| --------:| -----------  | -------------:| --------- |
|    CAP1A | 1            |            28 | Vdd       |
|    CAP1B | 2            |            27 | AUDIO OUT |
|    CAP2A | 3            |            26 | EXT IN    |
|    CAP2B | 4            |            25 | Vcc       |
|     !RES | 5            |            24 | POT X     |
|    !PHI2 | 6            |            23 | POT Y     |
|     R/!W | 7            |            22 | D7        |
|      !CS | 8            |            21 | D6        |
|       A0 | 9            |            20 | D5        |
|       A1 | 10           |            19 | D4        |
|       A2 | 11           |            18 | D3        |
|       A3 | 12           |            17 | D2        |
|       A4 | 13           |            16 | D1        |
|      GND | 14           |            15 | D0        |

| Pin name      | Description    |
| ------------- | -------------- |
| CAP1A, CAP1B  | Filter capacitor 1 (6581: 470 pF, 8580: 20 nF |
| CAP2A, CAP2B  | Filter capacitor 2 (6581: 470 pF, 8580: 20 nF |
| !RES          | Reset input - if low for at least 10 phi2 cycles, all internal registers reset |
| PHI2          | Input for system oscillator, eceives data only when high |
| R/!W          | High = read allowed, Low = write allowed |
| !CS           | Chip Select - active low input, bus data needs to be valid when active |
| A0..A4        | Address inputs to select one of the 32 internal registers |
| GND           | Ground |
| Vdd           | Second voltage (6581: +12 VDC, 8580: +9 VDC) |
| AUDIO OUT     | Audio outout, 6 VDC 3Vp-p at max volume |
| EXT IN        | External audio input, mixes with SID output and can be filtered. |
|               | (8580 needs a ca. 330 kOhm to GND on this pin to fix old digi sounds.) |
| Vcc           | Main voltage, +5 VDC |
| POT X         | Input for potentiometer (paddle) X-axis |
| POT Y         | Input for potentiometer (paddle) Y-axis |
| D0..D7        | Data bus bits 0..7 |


## How Does the C64 Control the SID? (Registers)

The method to control the sound-parameters of a SID in a Commodore 64 is called
_Memory-mapped I/O_. This means that the Commodore 64 sees the SID at addresses
`$D400..$D420` (hexadecimal) and it can write to the internal registers
('control- bytes') of the SID just like to any other portion of the memory.
Whenever you write to these addresses you essentially modify the flip-flops
inside the SID, which in turn set parameters like pitch, envelope, filter, etc.
in real-time. Simple, isn't it? (With modern hardware mods more SID chips can be
added to the C64 and in that case their base-addresses differ from `$D400`.
There's no set specification, yet, for what address they should reside at.)

__TODO__: Really? I thought the SID file format specifies these addresses?...

Most registers are write-only and you can't read them back, but there is also a
little feedback from SID towards the C64 in the form of read-only registers, not
to mention bit-fading which makes tricks like Hein's `ROR D400,X` possible.
Let's examine the registers for the 3 channels one-by-one:

### Pitch

| __Voice__ | __Low byte__ | __High byte__ |
| --------- |:------------:|:-------------:|
| Voice 1   | `$D400`      | `$D401`       |
| Voice 2   | `$D407`      | `$D408`       |
| Voice 3   | `$D40E`      | `$D40F`       |

Pitch low-, and high-byte. These bytes together control the pitch of an
oscillator. 16 bits give us quite enough resolution to make perfect pitches in
the region of 15Hz to 3848Hz (PAL) with equal steps of cca 0.06Hz. The human ear
and brain perceives pitches in a non-linear fashion, that is we hear less
difference between the equal frequency steps in higher regions than in the lower
ones. Every upper octave has twice the frequency of its lower counterpart. So
for scales of musical notes we need to have a lookup table (or _frequency
table_) to map the notes into frequency values. In the most widely used equally
tempered chromatic (Western) scale successive notes have the frequency ratio of
12th root of 2.

### Pulse Width (Duty-cycle) of the Square Waveform

| __Voice__ | __Low byte__ | __High byte__ (bits 0..3 only) |
| --------- |:------------:|:------------------------------:|
| Voice 1   | `$D402`      | `$D403`                        |
| Voice 2   | `$D409`      | `$D40A`                        |
| Voice 3   | `$D410`      | `$D411`                        |

Pulse width (duty-cycle) of the square waveform. This is a really important
feature of the SID chip because variations of the pulse (square) wave have very
different spectral characteristics. This is really useful for smooth transitions
between timbres (called _sweeps_) that make it sound more lively than just a
monotonic beep. Most of the time this is used to create lead instruments. Fast-
sweeping the pulse width enriches the spectrum of the sound and adds a kind of
chorus effect.

Luckily, the shape of some combined waveforms can also be altered by the duty-
cycle setting, giving even more timbres to choose from. The upper byte has only
the lower nybble (lower 4 bits) wired in so there are 4096 possibilities of
pulsewidth to choose from, between 0 and 100% duty-cycle from the thinnest to
the fattest sound. One of the strengths of the SID is that it essentially
operates at 1MHz 'sampling' frequency and the thinnest sounds are clear. This is
not always the case with emulated SID sounds of only 44kHz or so.

### Waveform and Envelope Control

| __Voice__ | __Byte__     |
| --------- |:------------:|
| Voice 1   | `$D404`      |
| Voice 2   | `$D40B`      |
| Voice 3   | `$D412`      |

The bits in this register control different things separately. The upper nybble
controls which of the 4 available waveforms are turned on. They can also be
turned on at the same time due to the properties of the underlying silicon
technology. This results in the the so-called 'combined waveforms' which Bob
Yannes is officially against of, which is understandable because these connect
some outputs together. Nevertheless, these have been used in many masterworks
from the beginning. The new 8580 has more defined and louder combined waveforms.
How this is done deserves a separate topic in this article, so stay tuned.

#### Waveform Control Bits

| __Bit__       | __Control__    |
| ------------- |:--------------:|
| Bit 0 (`$01`) | GATE           |
| Bit 1 (`$02`) | SYNC           |
| Bit 2 (`$04`) | RING           |
| Bit 3 (`$08`) | TEST           |
| Bit 4 (`$10`) | Triangle       |
| Bit 5 (`$20`) | Sawtooth       |
| Bit 6 (`$40`) | Pulse (Square) |
| Bit 7 (`$80`) | Noise          |

If none of the waveforms are selected then the floating of the last wave-output
value can be observed for a while, then it decays. On a real C64 the duration of
the decay is temperature (uptime) dependent.

The oscillator in the SID can be reset at any time and stopped by the turning on
bit 3 (value: 8) 'TEST'-bit. It was probably implemented in the SID for factory
testing but it comes handy as a tool in chipmusic. Whether it generates a high
or low steady output depends on the selected waveform. (Contrary to popular
belief, the test-bit doesn't have any effect on the envelope-generator - it
affects the oscillator only.)

If you want really special, jawdropping tones, there are two more weapons to
utilize, one is bit 2 (value:4) 'RING'-modulation, the other is bit 1 (value:2)
channel-'SYNC'.

Ring-modulation affects only the waveforms that contain triangle, but not
sawtooth, and in a nutshell it mirrors/folds the wave's upper half when the
neighboring channel's oscillator is in the 2nd half of its period. This creates
a richer spectrum and very interesting effects, including formant-like sounds,
all without filters.

__TODO__: Needs a diagram to explain

Channel-synchronization, on the other hand, resets the oscillator whenever a
neighboring oscillator enters the 2nd half of its period. This also creates
fascinating waves that resemble the human voice (where formants are synced to
vocal cords).

The controlling channels in both of these scenarios are always the lower
channels. For example, channel 2 is controlled by channel 1. Mostly these two
functions are mastered with experimentation, as it's hard to get a grasp how it
really works and to estimate the results. As with the waveforms, ringmod and
sync can be combined together.

Last, but not least, there is the 'GATE'-bit which does more than one would
think at first sight. It controls the volume-envelope of the generated waveform:
it starts and stops the notes, so to speak.

### Attack/Decay/Sustain/Release (ADSR) Envelope Generator Settings

| __Voice__ | __Attack, Decay__ | __Sustain, Release__ |
| --------- |:-----------------:|:--------------------:|
| Voice 1   | `$D405`           | `$D406`              |
| Voice 2   | `$D40C`           | `$D40D`              |
| Voice 3   | `$D413`           | `$D414`              |

* _Attack_: Bits 4..7
* _Decay_: Bits 0..3
* _Sustain_: Bits 4..7
* _Release_: Bits 0..3

When the GATE-bit is turned on, the 'ADSR' envelope-generator starts an 'attack'
phase, thus it starts a sound. If it's kept active, the volume rises at the rate
of the corresponding ADSR setting until it reaches the maximum level. Then it
falls to the 'sustain' level at the rate of 'decay' setting. Turning off the
gate-bit starts the 'release' phase, which means the note's volume falls towards
zero at the rate of the 'release' setting. Though this is the basic operation of
the GATE-bit, it can be turned on/off during any phase of the ADSR envelope. As
a rule of thumb when it turns on it always starts an attack phase, and initiates
release when it's turned off, though the envelope isn't reset to 0 if it's in
the middle region.

But, unfortunately, like with other things with SID, life is not so simple. As
you will see later in the more thorough explanations, ADSR sometimes does not do
what it's told to. You'll get weaker or even missed notes with certain ADSR
values and GATE-triggering schemes. No, it's not a 'humanize' function
intentionally built into the SID. I think the reason is the resourcefullness
that was a must for people making VLSI chip design in the early 80s. Maybe some
rushed work contributed to it, too, so ADSR rate-counters are never reset in the
SID. It would be logical to reset them when a note gets triggered by the GATE-
bit, but that's not the case. What makes it worse is the fact that the counters
can overlook their rate-settings. We'll explain it later, for now it's
sufficient to know that luckily people came up with a solution a long time ago:
the '_Hard restart_'. The optimal way to reset the ADSR before triggering notes
is still a subject of discussions at CSDb forums. Different music players
implement it in slightly different ways. There are also some lesser known
wraparound issues in the envelope-generator that will be explained in upcoming
parts of this document.

Attack happens on a linear scale. Here's a list of Attack times on PAL C64
machines:

2ms, 8ms, 16ms, 24ms, 38ms, 56ms, 68ms, 80ms,
100ms, 250ms, 500ms, 800ms, 1s, 3s, 5s, 8s

Decay and Release have longer (3 times that of Attack) non-linear curves.

### Filter Cutoff Frequency

|                         | __Low byte (bits 0..2 only)__ | __High byte__ |
| ----------------------- |:-----------------------------:|:-------------:|
| Filter cutoff frequency | `$D415`                       | `$D416`       |


One of the SID's strengths is its analog filter. The process of creating raw
waves with rich spectral content and then filtering out some of the components
is called _substractive sound synthesis_. The single filter is shared between
all the channels but it can be applied to them separately on demand. Once the
filter is set on a channel the cutoff-frequency can be controlled at an 11-bit
resolution. (Low-byte has only the lower 3 bits implemented, the other bits have
no effect.)

On the 6581 variant the curve of the cutoff-control is nonlinear, with cca 200Hz
below a 'threshold' and often the basses sound more muffled compared to the 8580
which has a nearly perfect linear control-curve. (But again our ears hear
differences at low frequencies better than at the high ones so we perceive it as
nonlinear too.)

On the flipside, the 6581 cutoff frequency can go up to the top of the hearable
range, while the 8580 can go down near 0Hz but tops out at ~13kHz. The 6581 has
an interesting distortion at low frequencies which will be explained later.
8580's new filter-design lacks this 'feature' and there's only distortion when
high resonances boost the signal.

### Filter Routing and Resonance

|                              | __Byte__ |
| ---------------------------- |:--------:|
| Filter routing and resonance | `$D417`  |


| __Bit__       | __Control__      |
| ---------     |:----------------:|
| Bit 0 (`$01`) | Voice 1          |
| Bit 1 (`$02`) | Voice 2          |
| Bit 2 (`$04`) | Voice 3          |
| Bit 3 (`$08`) | External input   |
| Bit 4..7      | Filter resonance |

The high-nybble here controls the resonance of the filter, the hump at the
cutoff frequency. The filter sounds more prominent with this setting than a neutral
curve with no emphasis. Just like the cutoff-control, this behaves differently
for the different SID-models: the 6581 doesn't change much up to a point while
the 8580's resonance-control is continuous, although it's non-linear.

Setting high resonance can lead to distortions as the magnified signal's level
approaches the limits presented by the 9V/12V power.

The low nybble has 3 bits dedicated to turn the filter on/off for the separate
channels: bit 2 (value 4):channel 3, bit 1 (2):channel 2, bit 0 (1):channel 1.
The number of selected filtered channels has a small effect on the cutoff and
resonance, but it's not very noticable.

Bit 3 (value: 8) is the switch for the external audio input which can be fed to
the SID and mixed into the output together with the internal channels. Some
people use this bit to decrease the noise coming into the SID from outside by
filtering it out. Originally it might have been added so that the SID could be
used as a wah-effect pedal.

### Main Volume and Filter Band Selector

|                              | __Byte__ |
| ---------------------------- |:--------:|
| Main volume and filter band  | `$D418`  |


| __Bit__       | __Control__      |
| ---------     |:----------------:|
| Bit 0..3      | Main volume      |
| Bit 4 (`$10`) | Low pass         |
| Bit 5 (`$20`) | Band pass        |
| Bit 6 (`$40`) | High pass        |
| Bit 7 (`$80`) | Mute voice 3     |

The low nybble of this register controls the main volume of the SID. There is a
little bit of leakage though, so even when you set it to 0 it passes through a
little amount of sound. However, the more important fact about this nybble is
that it causes a little shift in the output signal. The bigger the volume the
more the offset is. Since the 1980s this artifact was utilized to play digital
samples (or 'digis'). This effect is much less noticable in the refined 8580
circuitry, so this is a classical problem with new C64 machines: digitized
speech is barely audible on them. But this somewhat compensates for the harsh
clicks of the 6581 that appear when the master volume or filter- parameters are
changed.

The high nybble of this register has 3 bits that control what kind of filter to
use: bit 6 (value `$40`): high-pass, bit 5 (`$20`): band-pass, bit 4 (`$10`):
low- pass. These modes can be combined together to form e.g. a notch-filter or a
low- pass filter with brighter sound.

Bit 7 (value: `$80`) has a special function, it can prevent channel 3 from going
to the mixer, though it's still passed to the filter, so this has no effect on a
filtered 3rd channel. The idea was to use channel 3 as an LFO (low-frequency
oscillator) to control parameters without the need of the CPU to do that task.
But in practice we don't want to lose a precious channel when the CPU can create
any control-waveform easily. So let's leave this bit at zero, please.

### Paddle Values (Read-only)

| Paddle                       | __Byte__ |
| ---------------------------- |:--------:|
| Paddle X value (POTX)        | `$D419`  |
| Paddle Y value (POTY)        | `$D41A`  |

The SID chip also took on the responsibility for reading the analog resistance
values on the C64's inputs. This was used mostly for paddles to control games
but a mouse can be connected to these inputs as well, or any potentiometer with
around 500 kOhm maximal resistance to utilize the full range of 0..255 values.
Voltage can't be applied to these inputs to digitize sound, etc. It works by
charging and discharging a capacitor through the connected resistance and it
determines the resistance periodically by how much time it took to charge the
capacitor. Unfortunately this measurement has a jittering even with steady
input. Software-based filtering can help to smooth this out. (A 'moving average'
filter works just fine.)

### Oscillator and Envelope of Voice 3 (Read-only)

|                              | __Byte__ |
| ---------------------------- |:--------:|
| Oscillator voice 3 (OSC3)    | `$D41B`  |
| Envelope voice 3 (ENV3)      | `$D41C`  |


These are 8-bit readable registers that represent the waveform selector and
envelope generator outputs of the 3rd channel. In combination with the channel 3
disabling mentioned before these can be used as an LFO in rare cases. But for us
a more useful feature of these registers is to determine which model of SID is
present in the machine by checking for waveform and envelope differences. For
example, Hermit used these registers many times to display an oscilloscope for
the 3rd channel or to control graphic effects by the music. Use your imagination
what else these could be used for.

## How Does the SID Produce Sound? (SID-internals)

### Phase-accumulators (Oscillators, Pitch)

First of all, let's start with the three oscillators. Without oscillation a
sound could never be heard through the air, as you might know. In the SID this
is done by the 'phase-accumulators'. A phase-accumulator is basically a 24-bit
counter which can be incremented not only by a single step each clock, but also
by any number of steps between 0 and 65535. The 16-bit value which we add
at each clock pulse directly determines the frequency of the oscillation. How?
The phase accumulator wraps around when it reaches its maximal value, and starts
over to count up again. This represents a sawtooth-like waveform. We build upon
this basic concept in the next stages of the sound-generation chain.

The master clock-frequency and thus, also the SID's clock is running at 985248Hz
in the PAL version of the C64. If the frequency value is 0 in the frequency
registers, we don't add to the phase accumulator, and the oscillation is
stopped. Adding 1 gives the lowest hearable frequency we can produce. With the
24-bit phase-accumulator it takes 2 to the 24th power of clock pulses to fully
count up, so at the C64 clock frequency this happens 17 times per second. So,
17Hz is the lowest sound we can make. Adding 65535, the maximal value needs 256
clock steps to reach the top, so the highest the pitch can be is 3849 Hz.

__TODO__: This would prob need a diagram.

Beside setting the pitch we have some more control over the phase-accumulators,
they can be zeroed (reset) by:
- setting the TEST-bit (mentioned above) to 1 on the corresponding channel, or
- when the SYNC-bit is 1 on a channel, the phase-accumulator on that channel is
  zeroed at the moment the other (source) channel's MSB (bit 23) rises to 1.
  (Again, Sync source-to-destination channel-pairs are:  1->2 , 2->3 , 3->1 )

### Waveform-generators (Unfiltered Waveforms/Timbres)

As mentioned before we have 4 basic waveforms to choose from on each cannel.
They are created in different ways in 12-bit resolution.

Sawtooth is the simplest one, it's simply the upper 12 bits of the phase-
accumulator.

Pulse/square-waveform is derived by comparing the pulsewidth/duty-cycle
registers (value 0..4095) to the current top 12 bits of the phase-accumulator,
and connecting all output-bits to 1 (Vcc) when it's greater, and to 0 (GND)
when it's smaller.

__TODO__: This would need a diagram, too.

Triangle waveform is made from the phase-accumulator (sawtooth) by XOR-ing all
of its 11 upper bits with its MSB (bit 23). This causes the 2nd half of the
sawtooth-wave to fold back giving us the triangle waveform. But as this has a
halved amplitude, the 12-bit wave-output must be generated from the left-shifted
form of this to ensure that the output has the same amplitude as that of a
sawtooth wave.

__TODO__: This would need a diagram, too.

The ring-modulation for the triangle wave is achieved by enhancing the above-mentioned
MSB XOR-ing with an extra XOR if the RING-bit is set. This extra XOR with the
MSB happens whenever the source (modulation) channel's phase-accumulator MSB
is 1. In other words, the triangle is inverted/flipped by the other channel
(again, ring source-to-destination channel-pairs are: 1->2 , 2->3 , 3->1 ).

Noise waveform has its own 'counter' in the form of a pseudo-random sequence
generator. It's realized by a 23-bit LFSR (Linear Feedback Shift-Register),
which is a shift-register that when clocked, simply shifts its 0/1 contents to
the 'left'. What makes it an LFSR is the feedback mechanism that generates the
signal to be fed back to its rightmost bit (LSB). There are so-called 'taps' on
carefully selected places, bit 22 and 17 of the LFSR, that are XOR-ed and that
value is fed back to the LSB.

__TODO__: This would need a diagram, too.

This generates a very long sequence of pseudo-random values before it repeats.
The LFSR is clocked by the rising edge of bit 20 of the phase-accumulator so the
noise spectrum - the 'sense of pitch' - can be controlled. With the TEST-bit
enabled the feedback can be forced to 1 in the LFSR so it can be filled with 1s
over a cca 8000 cycles' period, reaching the value of `$7FFFFF` which is
probably the initial value of it at startup.

The 23-bit LFSR value still has some linearity/predictability between the
adjacent bits so we take the noise-output from a so-called 'scrambler' instead.
In case of the SID the scrambling is simply done by using 8 different bits of
the LFSR to constitute to the wave-output (bit 20,18,14,11,9,5,2,0) which only
has 8 bit resolution, but it's sufficient for noise. The 4 low-bits are not used
for noise.

### Waveform Routing

Now the we generate the 4 basic waveforms (still digital, 12-bit wide) we are
ready for ready for routing. Inside the SID there are pass-transistors (FETs) on
all 12 bits of the waveform outputs acting as series-switches to select which of
the 4 waveform-ouptputs we want to route to the bit-drivers of a channel's
output.

Ideally only one of them would be allowed to be turned on at a time, but there's
no multiplexing logic to ensure that. This brings us further possibilities. Bob
Yannes himself discouraged the usage of combining the waveforms by turning on
multiple outputs, probably because connecting active output-drivers together is
never a good idea in electronics - they will fight against each other when one
wants to drive a low signal while the other wants to drive a high signal.
However, in chips made with NMOS and HMOS chip-fabrication technologies the
driving strengths of the outputs are less for high logic signals than for low
ones due to the 'upper' MOSFETs used as 'active'/'dynamic' resistors. (CMOS can
drive both logic values equally strong, so that would be more prone to high
currents and failures if the SID was ever recreated with this more up-to-date
manufacturing technology.)

In practice there have been no reports that SIDs broke due to the usage of
combined waveforms but who can tell? These chips can become hot enough with
just standard usage...

### Combined Waveforms (A More Thorough Explanation)

During the development of jsSID Hermit did research on how the complex combined
waveforms are generated. (He could generate them with functions, see the jsSID
source code for more details and for ASCII schematics.)

If you look at them closely you probably notice that they look like fractals,
small portions of them resembling their overall shape. It's logical to deduce
from this that there is some recursive bit-wise manipulation responsible for
that.

Checking on the great reverse-engineering results of decapped SIDs at the
Kevtris webpage (__TODO__: Link needed) revealed the circuit of the above-
mentioned waveform-selector logic. There are simple amplifiers for all of the
selected (routed) waveform-bits before the DAC (Digital-to-Analog) stage. The
waveform-combining happens *before* these bit-amplifiers and the DAC, so the
combining is not made on the analog outputs, but bit-by-bit in the previous
digital stage.

To understand how the combined waveforms are generated, the analog behaviours
of this digital circuit-region needs to be discussed. The 3 crucial analog
contributors are the above mentioned weak driving of high-bits, the resistance
of the chip-fabric and the treshold-level of the amplifiers before the DACs.

For example, take the simplest case: when you connect sawtooth and pulse
waveforms together, you connect all their bits together through a weaker
connection, because that is what a square waveform does, as explained above: it
connects all 12 bits either to GND or to power-rail.

If the square/pulse output is 0, driving it to GND is so strong that no matter
if the sawtooth-bit wants to drive 1, it will be below the treshold of the bit-
amplifier so the DAC gets 0 to output. But when the pulse-output is 1 the bits
are driven by it to high only 'weakly'. In that case a 0 sawtooth-bit can bring
the combined bit-value low enough to ensure 0 at the output of the bit-
amplifier and the DAC. This sounds like an AND operation between the pulse and
sawtooth waveform-bits, and we would get a sawtooth if pulsewidth is 100%. But
it's not that simple: as the pulse-waveform connects all bits together through a
given resistance (depending on the chip-technology) it's possible for
neighboring bits of the sawtooth to affect each other. The closer a bit to the
other bit is, the more it pulls it down towards 0 or up towards 1, eventually
agreeing on a level that is above or below the bit-output's treshold. This is
the recursive process responsible for the fractal-like look. (In code Hermit
made this by creating two nested 'for' loops where each bit has a value affected
by all the others. With proper parameters the waveforms were very close to the
original.)

It's worth noting that if you look at a pulse+sawtooth with 100% duty-cycle, the
combined waveform samples don't go above the corresponding sawtooth values, only
below. This means that the FETs driving high are really much weaker than the
ones driving low. Especially on the old 6581 SID where most combined waveforms
are weak (contain many 0s) and have MSB suppressed (as a result their amplitudes
are halved but their frequencies are doubled, except with the pulse+triangle
combination).

If you add triangle to the 'mix' it gets even more complex: it connects adjacent
bits (for the left-shifting mentioned at waveform-generators) and provides even
more connections to 0 level, so combined waveforms containing a triangle have
more low or zero values when you look at them.

Noise can be combined with other waveforms but the discussed zeroing effect is
able to clear the bits in the LFSR gradually and when the LFSR is filled with 0s
it 'locks up' and can only be restarted by test-bits. (Interestingly, in a VICE
emulator it's possible to set a pulsewidth very close to 100% and combine that
pulse with a noise (`$C1` waveform) without locking it up. Never tried this on
real SID, by the way.)

### Envelope Generator (ADSR a.k.a. Channel Volume)

The ADSR envelope generator affects the analog part of the SID around the DAC.
Each channel has one, and its output is an 8-bit value (0..255) controlling the
VCA (Voltage-Controlled Amplifier) that determines the volume of the
corresponding channel.

Based on the ADSR parameters given in the register and the GATE bit in the
waveform control register, three internal counters (per channel) are operated:

- The 8-bit 'Envelope-counter' which is fed to the DAC controlling the
  channel-VCA.

- The 15-bit 'Rate-counter' is a prescaler to set the steepness (speed) of the
  envelope-counter in Attack/Decay/Release phases. Tthese are the prescale
  values (periods) for the Attack/Decay/Release values of 0..F:

  | Nybble value | Periods |
  | ------------:| ------- |
  |            0 | 9       |
  |            1 | 32      |
  |            2 | 63      |
  |            3 | 95      |
  |            4 | 149     |
  |            5 | 220     |
  |            6 | 267     |
  |            7 | 313     |
  |            8 | 392     |
  |            9 | 977     |
  |            A | 1954    |
  |            B | 3126    |
  |            C | 3907    |
  |            D | 11720   |
  |            E | 19532   |
  |            F | 31251   |

  (Note that a value of 0 still has a period, and therefore there's nothing like
  'zero-time' Attack/Decay. That's why sounds with AD=00 have clicky starts.)

- The 'Exponent-counter' is a further prescaler for the envelope-counter in
  Decay and Release phases to ensure more ear-friendly nonlinear sound decays.
  There's a space-efficient exponential table in the SID with prescale-values
  paired to envelope-counter value ranges/stages:

  | Envelope counter value range | Prescale value |
  | ----------------------------:| -------------- |
  |                      255..95 | 1x             |
  |                       93..55 | 2x             |
  |                       54..27 | 4x             |
  |                       26..15 | 8x             |
  |                        14..7 | 16x            |
  |                         6..1 | 30x            |
  |                            0 | 1x             |


  255..95:1x, 93..55:2x, 54..27:4x, 26..15:8x, 14..7:16x, 6..1:30x, 0:1x

  (At the start of a fast 1x prescaling/division, it gradually grows to a 30x
  slow decay when the envelope-counter falls below 6.)

Several state-bits determine the current ADSR state/phase:
- Attack-phase
- Decay+Sustain phase
- Hold-at-zero state

A transition of GATE-bit to 1 turns on Attack-phase and prepares for the next
Decay+Sustain-phase, and disables any Hold-at-zero state.

Attack-phase lasts until the envelope-counter counts up to `$FF` then it's
turned off and the Decay+Sustain-phase dominates in which the envelope-counter
through the exponent-counter prescaling counts down until it reaches the
sustain-value. (Which is expanded to 0..$FF by doubling the 0..F sustain-value
to the high-nybble.)

A transition of GATE-bit to 0 turns off any Attack or Decay+Sustain-phase and
counts down through the exp-prescaler until it reaches 0 (Release-phase) and
enters 'Hold-at-zero' state and only leaves this state when GATE goes to 1.

#### ADSR delay-bug

For hardware-efficiency in the SID most counters are made from LFSRs that need
less parts and it doesn't matter if they don't count linearly, the comparison
values are simply selected according to their predetermined (pseudo-random)
sequence.

But that turned out to be a problem with rate-counters that determine the speed
of the envelope-counter. The rate-counters are only reset when they count up to
their current comparison value which is based on a prescale-table looked up by
the 0..F Attack/Decay/Release value, depending on the current ADSR phase.

As there's a single rate-counter for all the 3 timed ADSR-phases and starting a
new phase doesn't reset it, it's possible for the rate-counter to miss a
prescale-value (rate-period) that it already went through: when for example a
faster (lower-period) Attack follows a slower (greater-period) Release. In that
case a match is not found until the rate-counter goes up through its full
sequence and starts over by wrapping around. That can take as long as 32.8ms
(counting at 1MHz, cycletime is 1 microsecond, 32768 * 1 microsecond = 32.8ms).
The next note can delay that long, and it's quite audible. This is the so-called
ADSR delay-bug.

The ADSR delay-bug appears statistically rarely when the difference between the
rate-periods of adjacent phases is small (doesn't decrease much), but is more
frequent when the next rate-period is much smaller than the previous. (It's
less-known/less-noticed but it is logical: this delay-bug can happen after a
transition from an Sustain to a Release phase, too.)

We'll see later how to overcome (or how to enforce) this delay-bug situation.
But for now these are all the important details about how the ADSR basically
works.

### Filter (Shaped Timbre, Substractive Synthesis)

After the channel-DACs there are 2 routes for the now analog waveforms to take:
either the direct lines to the main output mixer or through the filter circuitry.
In the register-section I described the exact addresses and bits of the filter-
controls which determine this route. The channels going into the filter are
summed through resistors.

__TODO__: Needs a block diagram

This circuit in the SID is a '2-integrator loop bi-quadratic' filter and as its
name suggests, it contains two integrators in a loop through a 3rd member, a
simple amplifier for resonance/emphasis. The integrators are basically inverting
operational amplifiers with capacitive feedback. These capacitors are both
outside the SID-chip on the motherboard and are of fixed values. What makes it
possible to change the filter cutoff-frequency is that the VCR (Voltage-
Controlled Resistance) is in series to the inputs of these integrators, serving
as the variable resistors (both controlled in tandem) in the RC
filter/integrator.

The 8580 and the old 6581 SIDs differ very much in their implementation of the
VCR: the 6581 uses 2 single FETs as VCRs with some attempts for linearity by
having negative feedback resistances to their gates and operating in their near-
zero signal-region. There are resistor-ladder DACs that convert the filter-
frequency set in the registers to an analog signal, which also control the gates
of the VCR MOSFETs. But this simple control of a series (common-source) FET has
the disadvantage of generating feedback from the signal-path: the FET is a
transconductance device that transforms voltage between its Gate and Source
terminals. But the source is not at GND so the Gate-Source voltage depends not
only on the cutoff control signal but also slightly on the integrator's small
input-signal. That causes a kind of distortion unique to the 6581 old SID,
because thanks to the 'resistance-modulation' of the VCR the cutoff-control
signal gets a bit of the audio-signal, so the audio signal essentially alters
its own filter-cutoff frequency, and even during a single wave, the waveform
becomes less rounded. The unique fat sound produced by this is actually
preferred by many people.

The control-curve of the 6581 SID is also nonlinear, because there are about
1.5GOhm 'shunt' resistors between the Drain and Source terminals of the VCR-
FETs. So when their resistances go above this value, less and less change is
seen in the cutoff-frequency. On average this 1.5GOhm resistance ensures a
minimum cutoff-frequency of about 200Hz. There is big spread among 6581 SIDs, so
these resistance-values and the cutoff-frequencies vary wildly from chip-to-
chip. (These are the so-called 'dark' and 'light' SID cutoff-curves...)

The 8580 SID's redesign affected its filter a lot: as seen in the die photos at
kevtris.org ( __TODO__: Link needed ), the single-FET VCRs were replaced by a
different method. Essentially, the filter-cutoff control-voltage DACs seem to be
integrated with the VCRs into a digitally controlled resistor-ladder VCR.
Anyhow, this results in a very precise (I'd say laboratory quality) and linear
filter-cutoff control, and filter-distortion seems to be gone, too.

(Back at the time Robert Moog could only make sophisticated analog VCFs from
bipolar transistors connected as differential amplifiers in a ladder layout,
having capacitors in the rungs, which was the so-called 'Moog-filter'. MOSFETs
in SID made this easier.)

### Mixing and Output (Main Volume)

The output stage is simply mixing the non-filtered audio route and the filter's
output and applies the main volume on them through a VCA, then the SID's sound
is ready for amplification which goes to the outside world in ways described
earlier. (Some people say the simple transistor-based output-amplifier in the
C64 might add some characteristic nonlinearity to its sound... maybe.)

---
## Usage of SID in Practice, Tips, Tricks and Secrets
---

Knowing the internals of SID is not enough for squeezing good music out of it.
We still need an interface from the SID to the Human, namely: the composer.

The composer needs a tool to compose and execute his/her ideas. There are many
so-called trackers around and most of them add a lot of extras by software to
the SID to bring it closer to ready-made synthesizers: frequency-tables,
vibratos, LFOs, etc.

### Hard-Restart (Preventing the Delay-bug)

One of the most important features of such tools is to eliminate the ADSR delay-
bug so the composer can rely on sound-starts. The workaround to the bug is
called '_Hard-Restart_'. (Hermit has another article about it in the FlexSID's user
manual. (__TODO__: Link needed) )

The method 'resets' the ADSR rate-counters, so when a new sound (Attack) happens
we know the rate-counter is not just anywhere, but it's counting inside the
first 9 steps rapidly. To achieve this we simply need to set both AD and SR
registers to 00 and the GATE to 0 for at least 2 PAL-frames (40ms) before a new
sound is about to start. This also ensures that the Release-phase arrives at a 0
level.

At least this is the method that always works, no matter what ADSR settings the
previous and the next instruments have. Different programs use different
approaches, many times only the SR is zeroed, which is OK too. In programs where
the Hard-Restart-ADSR can be set to any value, people tend to use in-between
values in the pursuit of softer sound-starts, but most methods seem casually
suitable for a given piece.

There's a 'new kind of hard-restart' mentioned at CodeBase64 (by Shrydar, and
Lft is involved here too), they call it 'Bottle', and it's a totally different
cycle-exact code approach. It is able to reset the rate-counter in the timeframe
of about 10 rasterlines (less than 1ms) instead of a 20ms frame, by utilizing
the delaybug-free safe transition from a slow attack to a fast decay. There's
another ADSR-bug in the SID: the envelope-counter can also wrap around when it
is at value `$FF` and an Attack is triggered. This is used to bring the envelope
back to `$00` fast with this 'Bottle' approach.

Only time will tell how soon this restart-method gets implemented in players...

### To Delay-bug Or Not to Delay-bug? (Sexy-start)

What differs in most editors and their players/drivers is how the new sound
starts after the Hard-Restart. Different recipes work differently. To avoid a
new delay-bug after Hard-Restart the best write-order - as per Hermit - is to
set AD first before turning on GATE so it won't affect the rate-counter (thanks
to it being in release-phase after Hard-Restart), then set the GATE-bit to 1 to
change to Attack phase, and then immediately set the SR-register which will only
have an effect when the next GATE-bit turnoff happens (note ends).

But most player-routines do quite the opposite instead (causing the delay-bug)
to achieve the so-called 'sexy-start' of sounds: they write AD and SR before
turning the GATE-bit on and if the Release-value is big enough (above 3..4), a
new sound with a small Attack value of 0..1 is started, during which time the
rate-counter had a chance to advance and cause a miss of the Attack compare-
value. The more one waits between AD+SR and GATE writing the more probable this
situation becomes. The result is having a delay in the start of sound of about
32.8ms, so the 1st frame of the sound is silent (usually waveform `$09` is set
here to set TEST-bit), but the 2nd frame's last 7..8 milliseconds are audible.
And that is what musicians need, a very short waveform (the first row of a
waveform- sequencer table in the instrument-editor) to make the start of a sound
percussive without turning to multi-speed tunes. Usually high-pitched white-
noise is placed here. Our ears are much more sensitive to changes and a special
start of the sound fulfills that scenario.

It's not important to use Hard-Restart for fairly stable 'sexy-start' notes, but
the release of a previous note should be much bigger than the attack of the new
note, and it's advised to decay to 0 level before the new note starts. It's
easier with drums that are decayed fast from within the waveform-tables. Many
composers in the past didn't use Hard-Restart at all, but knew these rules and
selected ADSR values of adjacent sounds carefully.

To allow more diverse instruments but still retain a much bigger release before
an attack we can set the SR register to `$0F` artificially for 1 frame before
the sound-start. This is a simpler kind of 'hard-restart' and this doesn't zero
the rate-counter, but it does the opposite: it allows enough time (20ms) for it
to count into a region which will almost surely be above the compare-value of
the next Attack and cause a delay-bug, and thanks to that, a shortened 1st frame
of the next sound, aka 'sexy-start'.

### Digital samples on the SID

SID was not designed to play back digital samples, so digis were not the
strongest side of the SID in the past. The volume-register setting trick causing
an offset is described in the section about SID registers above, but that could
only produce 4-bit resolution sound and it affected the main volume of the SID.
So if normal SID-music was played alongside it, it sounded distorted/modulated.
(Not to mention the big difference between the volume of the digital samples on
the two SID variants.) Some emulations (e.g PlaySID on the Amiga) could separate
the AC-component of the main volume and send it to a separate digi-channel, and
the DC-component could still be used as main volume, and was even freed from
audible pops when the volume was changed slowly for fade-in.)

The next 'easiest' method to play digital samples on a C64 is Mahoney's method:
He measured the offsets caused by all the bits (lowpass/highpass/volume/etc.) in
the SID with different settings of 100% duty-cycle pulses and ordered them in a
256-byte table for both the 8580 and 6581 SIDs. There are about 30..50 different
quantization-levels that can be reached with the proper settings and tables
which is better than the 16 levels of simple digis. The downside is that no
normal SID-channels can be used beside the digi, but it's still as simple as
writing `$D418` at a given sampling frequency.

The other methods that can achieve 8-bit digi-resolution on a SID need precisely
timed (cycle-exact) code. Some methods (e.g. those used in the _Wonderland_
demo-series) used the pulse width control of the SID to create an up to ~15kHz
PWM signal by resetting the phase-accumulator at this rate. Because of the
audible carrier frequency this was not so appealing for music (maybe a lowpass-
filter is a solution for the carrier noise), but a step forward, nevertheless.

The ultimate solution at the moment is _SounDemon_'s digi-routine which utilizes
the floating signal on a channel when no waveform is selected (waveform `$01`).
The task of this method is similar to the PWM-method: to periodically reset the
phase-accumulator at the given sample rate with the TEST-bit, and set the
oscillator frequency proportionally to the desired sample level. The ADSR must
be kept at sustain-level, so the waveform is kept at value `$01` most of the
time. Then after a given amount of time (that should fit in the sample-period)
the waveform is set to `$11` (triangle) for a short moment to update the
floating value at the waveform-selector output. The next round comes so fast
that the floating value doesn't decay significantly. This is like a sample-and-
hold circuitry. The upward-slope of the triangle waveform instead of a sawtooth
ensures higher range (resolution) in less time.

The only disadvantage of the SounDemon digi is the strict timing and the high
CPU resource it needs, but it sounds good, and the other SID-channels can still
be used along with the digi just fine.


## Sound Design & Composition Tips

Here I collect some of my findings about good SID sounds. In the past I coded a
tool called 'SIDhack' to separate the SID-channels and debug them in realtime to
determine what different SID tunes do to sound so good. That tool came handy and
I made some tunes containing ripoffs from other SIDs as case studies, like the
tune 'sidhack' in the SID-Wizard package. The CSDb forum can also be a good
source for sound-design tips & tricks where people share their experiences with
each other.

In general, our brain, our neurons are more sensitive to changes rathe than
steady signals. This might be an evolutionary solution to long-term stimuli and
to focus better on new events. We have 'differentators' built-in. For example,
the sense of smells and colours and even touch degrades over a short time: we
get used to smells, we see the opposite of a color when it's removed, placing
our hand on a raw surface feels it but the feeling fades. Our eyes make micro-
movements just to keep the nerve-signals frequently updated, stopping them would
lead to slow loss of vision.

The ears work similarly and our whole appreciation of music depends on it: we
like changing sounds better than the steady ones, therefore a pulsewidth-
modulation instead of a fixed duty cycle can make a big difference in a lead
instrument. This is the same for the filter: a filter sweep on a bass sound or
performing some keyboard- tracking (opening the filter as the pitch increases)
is more desirable than a muffled, steady cutoff frequency.

So one ingredient of good SID sound is pulse/filter-sweep. Even better if the
program supports turning off the filter/pulsewidth-program reset when the new
sound starts, which provides even more variations.

Speaking about variations, rhythmic variations, syncopation and, of course,
melodic variations, and even variations at a higher level (in the structure,
arrangement) are desired in music, but that's a whole other topic in its own
right, so I'll stay with the sound design for now.

Sawtooth and triangle waveforms are invariable compared to pulse, but sometimes
they providea a better character for an instrument. I especially like when
arpeggios are made with triangle, they provide a feeling of ambience.

But I guess pulse/square is the most used because its spectral content can vary
much. The thin pulses are similar to sawtooth waves, the 50% pulses are glassy
and Nintendo-like, but for a bit more harmonic content I usually like to set the
pulse width somewhere near 50% instead. Slow pulse-sweep is the key for
beautiful lead sounds, but a very fast pulse-sweep is the key for some 'chorus'
effect. (Probably the reason for the 'chorus'/'room' effect is two-fold: the
change of harmonic content might cause a bit of percieved detuning and my other
assumption is that when a pulse width is changed it doesn't happen in sync with
the phase-accumulator and as a result there are many partial pulses creating
different frequencies temporarily.)

And last but not least, the pitch: our ears are very sensitive to even small
changes in sound, but to me it seems we're most sensitive to pitch (or
frequency) in music. A very minimal detuning can cause unpleasant sounds in
melodies. But detuning can be our friend, too, to create good instruments. Sure,
detuning a channel compared to an other can achieve a choir effect like with the
accordion, doing it with vibrato can make the music even more lively. Vibratos
are ment to be used with a little bit of delay even on live instruments because
our ear needs a stable note-start before letting the rest of it to vibrate.
Vibratos are good tools to emphasize notes, just like dynamics. Some chorus
effect comes very handy for basses, too, to make them appear stronger under the
lowpass-filter.

For high-pass filters I usually don't prefer to use big resonances, but they're
nearly essential on the SID where the resonance is not too strong compared to
analog syths and VSTs. To make a bass sound somewhat richer I usually set
both low-pass and bandbass filters. For special sounds like claps or speech,
the bandbass filter alone seems very good and should be used more frequently
in SID-tunes.

Sometimes it's good to enhance a lead instrument by dedicating the sole filter
to it. Basses in a mix can stay unfiltered and still sound good because they
have the deepness, but their harmonic contents still keep the richness of the
music. Sometimes with jazzy and soft tunes the triangle waveform is good to
act as a doublebass-like sound or even in techno tunes, and the filter can be
used to make the leads sound modern, sound more like an expensive synth.

### Sync/ringmod?

These have always been mysterious despite knowing how they work. Most of the
time some good sounds could be made by tweaking. In general, sync-effect seems
to be more deterministic, but ringmod gives frequencies that are hard to follow.
Using both is even more interesting and uncontrollable, but playing around can
lead to good results...

Echo-simulations on a single channel in a melody are quite possible by inserting
softer notes between the normal notes. I don't know what others think, but I
usually feel the notes that best fit there are repeated notes and not
necessarily some notes appearing earlier on that channel or notes in the melody.

Good bass drums can be made without the filter in the waveform-table, sometimes
even with triangles. But the strongest bass drums contain ~50% square waves in
a sudden frequency-drop at the first 1..2 frames, then only several notes
of drops in the last frames.

Good snare sounds can be made by only 1 but maximum 2 frames of ~50% square,
then the decaying white-noise sould continue asap. I like snares in funky
music which end abruptly but it's not obvious how to make them on the SID.
Usually release values of 5..7 are fine for snare. I like the snares of Shogoon
which are made on 2 channels and you can really hear the oomph and the
snare noise at the same time. But that's not always possible.

Arpeggios are tricky beasts. They can sound ugly when the pitch changes every
frame, I usually let pitches last at least 2..3 frames. To reduce the abrupt
pitch-changes even further arpeggios can return to the base note. Chord-
inversions can also make arpeggios even more listenable. If done well, complex
harmonies with dissonant intervals sometimes are more listenable as arpeggios
than when they're played together. After all an arpeggio is a fast melody, so
dissonances disappear fast, maybe that's why.

Dynamics in music are usually desired, there's more variety in a hihat or kick
or snare when there are strong and weak hits in good places. But interestingly
for some oldschool C64 tunes the fast repetitions without any dynamics sound
better, maybe they emphasize that this is a different style of music and not
something played by a human who automatically adds dynamics by the laws of
physics.

All in all, good ears and being open-minded for new possibilities are probably
the best leads for creating interesting SID music. And the importance of
composing should never be underestimated, a good composition with simple
instruments is often more joy to listen than music with good sounds but
without an idea or a story.

# Authors

## Original Document

v0.1 by Hermit (Mihály Horváth), 2022

## Other Contributors

- LaLa (Imre Olajos) - proofreading, reformatting

