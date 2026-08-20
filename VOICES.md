
# VOICES.BIN Format Notes

The `VOICES.BIN` file is used by **Grand Prix Circuit** (and supported by **Test Drive II: The Duel**, albeit seemingly not used by default) to define simple individual "voices" with different characteristics.

Each voice occupies 32 bytes in the file. The music definition files [`SONGS.BIN`](SONGS.md) (TD2) and [`MUSIC.BIN`](SONGS.md) (GPC) select a voice in a pattern using the `0xFD xx` command, where `xx` is the voice index starting from 0.

While the record is 32 bytes long, the game only copies **28 bytes** into memory. __The last 4 bytes are unused__ and might be padding bytes.

Based on the results of my experiments, modders seeking to create custom voices __may use these 4 bytes for naming purposes__ with no adverse effects to the game.

__NOTE:__ As with [`SONGS.BIN`](SONGS.md), __Little Endian__ ordering is used.

__NOTE:__ The timings given in this file (i.e. Arpeggiator and LFO step dividers, sizes, etc.) are __independent__ of the music file's `0xFE` tempo command!

## Data layout

| Offset | Size | Type                | Description                      |
|:------:|:----:|:--------------------|:---------------------------------|
| 0x00   |    1 | unsigned byte       | Note gating                      |
| 0x01   |    1 | -                   | Unused __(default value: 0)__    |
| 0x02   |    2 | uint16              | Arpeggiator initial delay        |
| 0x04   |    2 | uint16              | Arpeggiator step divider         |
| 0x06   |    2 | uint16              | Arpeggiator index mask           |
| 0x08   |    8 | byte[8]             | Arpeggiator note-offset table    |
| 0x10   |    2 | uint16              | LFO initial delay                |
| 0x12   |    2 | uint16              | LFO step divider                 |
| 0x14   |    1 | unsigned byte       | LFO waveform / initial direction |
| 0x15   |    1 | -                   | Unused __(default value: 0)__    |
| 0x16   |    2 | uint16              | LFO excursion step size          |
| 0x18   |    2 | short               | Lower pitch excursion limit      |
| 0x1A   |    2 | short               | Upper pitch excursion limit      |

---

# Brief explanation of available modulation types

## The arpeggiator

The arpeggiator is the first modulation mechanism in a voice definition. It's like a mini-sequencer that works with note offset values and uses a timing scheme independent from the track's tempo.

A note will play normally until an _initial delay_ passes, after which the sound engine will cycle through an array of _note offsets_ at a speed set through the _step divider_.

## The pitch LFO

The pitch LFO is the second modulation mechanism that you can find in `VOICES.BIN`. As its name suggests, it gives every note a customizable vibrato-type effect.

Similarly to the Arpeggiator, it also has an _initial delay_, and a _step divider_ to set its speed. On top of that, it also has values for _step size_ and _upper/lower excursion limits_ (which is how this document refers to _modulation amplitude_).

---

# Detailed description of each address/data field

### `0x00` --- Note gating

Unsigned byte that controls how early a note stops relative to its nominal duration.
Lower non-zero values produce stronger, more premature staccato-like shortening.

A value of `0x00` is specially treated as no shortening (full note).

| Value | Duration | Percentage    |
|:-----:|:---------|:--------------|
| 0x00  | Full     | 100%          |
| 0x01  | 1/2      | 50%           |
| 0x02  | 3/4      | 75%           |
| 0x03  | 7/8      | 87.5%         |
| 0x04  | 15/16    | 93.75%        |
| 0x05  | 31/32    | 96.875%       |
| 0x06  | 63/64    | 98.4375%      |
| 0x07  | 127/128  | 99.21875%     |

---

### `0x01` --- Unused byte

It is recommended that you set this byte to zero.

---

### `0x02, 0x03` --- Arpeggiator initial delay

A 16-bit integer that sets the delay before the arpeggiator becomes active.

A value of `0x00` will cause the arp to start immediately.

---

### `0x04, 0x05` --- Arpeggiator step divider

A 16-bit integer that sets the delay between successive arpeggiator steps.

Higher values will slow down the arp, while lower ones make it cycle faster.

---

### `0x06, 0x07` --- Arpeggiator index mask

A 16-bit integer bitmask applied to the arpeggiator index.

While the game loads both bytes, only the following values are valid/intended to avoid erroneous operation.

| Mask | Arp slots used |
|-----:|:---------------|
| 0000 | 1 note (No arp)|
| 0100 | 2 notes        |
| 0300 | 4 notes        |
| 0700 | 8 notes        |

__Arbitrary amounts of note slots__ (such as 3, 5, 6, or 7 notes) __are not possible.__

__WARNING:__ Values larger than `0x0700` allow the index to read beyond the eight-byte note table and into unrelated memory. Therefore, if you're making custom voices, only use the values in the table above.

---

### `0x08 to 0x0F: 8 byte range` --- Arpeggiator note-offset table

Eight unsigned note-offset bytes, where values count as semitones to be added to the base note.

Example (hex): `00 04 07 0C 00 00 00 00`

with a mask value of `0x0300` _(4 notes)_ written into the `Arpeggiator index mask`, this gives us the following:

```
base
+4 semitones
+7 semitones
+12 semitones
repeat
```

---

### `0x10, 0x11` --- LFO initial delay

A 16-bit integer value that serves as the countdown before the LFO begins moving.

A value of `0x00` will cause the LFO to immediately start playing without delay.

---

### `0x12, 0x13` --- LFO step divider

A 16-bit integer that sets the delay between LFO excursion steps.

Just like the Arpeggiator step divider, higher notes slow down the LFO while lower ones speed it up.

The effective LFO rate depends on this field and the excursion step at `address 0x16` in tandem.

---

### `0x14` --- LFO waveform / initial direction

An unsigned byte value that sets the LFO waveform and the initial direction at once.

Valid value ranges are as follows:

* `0x00` - Triangle waveform. Initial direction: Falling.
* `0x01 - 0x7F` - This range yields a sawtooth waveform, initially falling.
* `0x80 - 0xFF` - This range yields a sawtooth waveform, initially rising.

The game does not consider the "magnitude" of the value at all. As long as the value falls within one of these ranges, that mode will be active.

---

### `0x15` --- Unused

It is recommended that you set this byte to zero.

---

### `0x16, 0x17` --- LFO excursion step size

A 16-bit integer value that the LFO increments on every update.

Unlike _step dividers_, __a higher value makes the excursion traverse its range more quickly.__

To produce no movement, a value of `0x0000` may be used.

__WARNING:__ The maximum observed value seems to be `0x0020`. Anything past this value has consistently produced crashes, so please __do not exceed this limit.__

---

### `0x18, 0x19` --- Lower pitch excursion limit

Signed 16-bit integer value. __Further experimentation required.__

`0x19` must be `0xFF`, otherwise the game crashes.

Example: `D8 FF = -40`

Negative values raise the resulting pitch as they reduce the Programmable Interval Timer divisor.

---

### `0x1A, 0x1B` --- Upper pitch excursion limit

Signed 16-bit integer value. __Further experimentation required.__

Example: `28 00 = +40`

Positive values lower the resulting pitch as they increase the Programmable Interval Timer divisor.

---

### `0x1C to 0x1F: 4 byte range` --- Padding

These four bytes are not copied to memory when `0xFD xx` loads a voice. The padding bytes likely only exist to simplify how the game jumps to the voice offset in memory, by simply bit-shifting the voice index value to the left by 5 when accessing voice definitions. This gives offsets in multiples of 32.

As the game does not load these bytes to memory, they may be freely used to name your voices to make it easier to keep track of them. I've extensively tested this possible use case and found no errors when I gave my voices all sorts of silly names.

---

# Example Voice _(ID #0 from GPC's `VOICES.BIN`)_

### _Raw hexadecimal:_ 

`03 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` <br>
`00 00 01 00 00 00 14 00 D8 FF 28 00 00 00 00 00`

### _Parsed voice definition:_

```
0x00     03       gate = 7/8 duration (87.5%)
0x01     00       unused

--- Arpeggiator setup (All zeroes, Disabled)
0x02     00 00    arp initial delay = 0
0x04     00 00    arp step div = 0
0x06     00 00    arp mask = 0
0x08-0F  00...00  arp note offsets = 0

---LFO setup
0x10     00 00    no LFO onset delay
0x12     01 00    LFO step div = 1
0x14     00       triangle waveform, initially falling
0x15     00       unused
0x16     14 00    excursion step = 20
0x18     D8 FF    lower limit = -40
0x1A     28 00    upper limit = +40

0x1C     00 00 00 00    padding / ignored
```

---
---




---

# Author's remarks

### A specific observation about Test Drive 2's VOICES.BIN

Despite TD2's [`SONGS.BIN`](SONGS.md) making several calls to Voice #1 via `0xFD 01`, the stock `VOICES.BIN` that comes with the game only contains 2 bytes: `0D 0A`. It is unclear if Test Drive 2 had its voice definitions compiled into the .exe, but replacing its `VOICES.BIN` with the 288 byte version from Grand Prix Circuit confirms that the game WILL load voices from the external file if it contains valid definitions.

### Current state and usefulness of this document

While I consider this documentation mostly complete, a few things remain a mystery at this time, such as the exact duration of one arpeggiator/LFO timer tick, or how many voice definitions can the game REALLY handle (theoretically 0-255).

I may or may not get to the bottom of such things, but I think it's safe to say that this document is already detailed and organized enough so that it may be used to understand the format, create custom voices, or even an editor/compiler for custom voices.

Do with this information what you wish. Make cool tunes, sick effects, all sorts of custom content that'll help keep these old gems of gaming history alive.
