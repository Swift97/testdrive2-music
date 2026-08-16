# SONGS.BIN / MUSIC.BIN Format Notes

`SONGS.BIN` contains the music tracks used by Test Drive 2: The Duel. The same file is called `MUSIC.BIN` in Grand Prix Circuit.

At its core, the songs are similarly structured to tracker music; Short musical patterns arranged in a sequencer roll, making up a complete track.

The file format can be divided into the following 3 groups:
* __Track definitions__; Offsets pointing to the addresses of the first sequencer entry in the tracks' sequencer roll.
* __Sequencing/arrangement__; Sequencer entries containing a 2 byte offset pointing to the address of the pattern data to be played, and a single byte of transpose value.
* __Piano Roll__; The pattern data itself, made up of note/duration byte pairs.

---

## Group 1 - Track pointers
The track pointer section is made up of __groups of 4 bytes__. At its beginning/end, a value of `00 00 00 00` serves as a marker. Each of the 4-byte groups in-between serve as pointers to the start of the sequencer roll for each track that's used by the game.

As this behavior is shared between TD2 and GPC, they _might_ be markers for the start and end of track data.

### List of TD2/GPC tracks:

| Game | Track | Hex Offset of 1st seq (LE) | When does it play? |
|-----:|:-----:|:--------------------------:|:-------------------|
| TD2  |   #1  |        `14 00 00 00`       | Intro / Menu / Intermission |
| TD2  |   #2  |        `68 00 00 00`       | Ending: Handcuffed |
| TD2  |   #3  |        `77 00 00 00`       | Ending: No cuffs   |
| GPC  |   #1  |        `1C 00 00 00`       | Menu               |
| GPC  |   #2  |        `05 01 00 00`       | Intro / Ending     |
 
>_Raw data from the beginning of TD2's SONGS.BIN:_
>`00 00 00 00 / 14 00 00 00 / 68 00 00 00 / 77 00 00 00 / 00 00 00 00`
>From this example, we can see that the group begins and ends with `00 00 00 00`.
>Between the markers, the title song's first sequence is defined as address `0x14`, the 1st part of the ending song as `0x68`, and the 2nd part as `0x77`.

---

## Group 2 - Sequencer roll
The sequencer data is made up of __groups of 3 bytes__.

- The first and second bytes serve as the Little Endian uint16 address of the pattern to be played.
- The third byte is a transpose value that's applied to whole pattern.

__IMPORTANT:__ During reverse-engineering experiments, I've found that a transpose value of 0 may break playback in-game. Further experimentation necessary.

>_The following example is taken from address `0x14` onwards (title song):_
>`86 00 05 / AC 00 05 / CE 00 05 / CE 00 05`
>Looking at the first group, we can see that it plays the pattern starting at address `0x0086` with a +5 semitone transposition.

---

## Group 3 - Pattern data
The pattern data is comprised of __groups of 2 bytes__, storing note ID and length respectively.

Various control commands are also used in this area.

__IMPORTANT:__ The first pattern __MUST__ start with the Tempo command __`0xFE`__ which will be detailed in the "__Control Commands__" list below.

### Musical notes
* __Note ID:__ The lowest ID 1 corresponds to D#2. Each successive note is one semitone higher. <br>
>Further experimentation is necessary to find the maximum value the game can handle, but somewhere around `value 0x4F`, playback might start to break. Try not to exceed this pitch, especially if you're transposing your pattern in the sequencer roll or using a custom `VOICES.BIN` definition that has a wild arpeggiator on it.

* __Length:__ The note length is treated like a multiplier for the song tempo value, 1 being the smallest length unit you can use.
>_(Please see __`0xFE xx - Tempo command`__ in the command list below for further details about how they work together.)_

---

# Pattern Roll --- Control Command list

__`0xFB 00` - End of Track marker__

This marker causes the music to end without looping.

---

__`0xFC 00` - End of Pattern marker__

This marker must be called at the end of each pattern. Causes the game to jump to the next sequence in the sequencer list.

---

__`0xFD xx` - Voice selector marker__

This marker selects an instrument from `VOICES.BIN`, where xx is the instrument index starting from 0.
>_Please refer to VOICES.md for more information about its format._

---

__`0xFE xx` - Tempo command__ <br>
While this document refers to it as "tempo", it's more of a timer divider that affects the duration of each note in the track. __A higher value results in longer notes overall, thus a slower tempo.__

The value roughly corresponds to the on-time of a note of length=1, and is __APPROXIMATELY__ given in millisecs/10, give or take.

For example, a value of 16 is around 160ms on-time for Note Length 01, while a value of 7 results in a duration of about 70ms.

>__NOTE:__ All of this is based on my general time measurements of audio clips recorded from Dosbox. The game's sound methods were not exactly accurate, there was ample spread, and the actual tempo might deviate from calculations.

---

__`0xFF 00` - Loop Music marker.__

Called at the very end of the track. Causes the game to loop back to the first sequence in the arranger.

>_The following example is taken from address 0x86 onwards (First pattern of title song):_
>`FE 07 / FD 01 / 17 02 / 0B 02 / 1A 02 / 0B 02 / 19 02 / 0B 02 / 15 02 / 17 02 / 0B 02 / 12 02 / 1A 02 / 0B 02 / 19 02 / 0B 02 / 15 02 / 17 02 / FC 00`

In the above example, you can observe the _Tempo_ marker `0xFE` as well as the _Voice selector_ `0xFD01` being used at the beginning of the pattern, while the _End of Pattern_ marker `0xFC00`__ is the last command.

Everything in-between these commands is a note/length byte pair.
