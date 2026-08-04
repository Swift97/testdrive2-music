# Reverse-engineering _SONGS.BIN_<br>from the MS-DOS version of Test Drive 2

_Below you will find the results of my research since the beginning of this reverse-engineering project in 2017._
_This is a hobby project that I came back to after 9 years of hiatus, after I found some old screenshots and text files._ <br>
_Given this, possible updates to this page might be slow to come. I am publishing this only to archive this knowledge, just in case someone else also gets the idea to customize this game's music._ <br>
_Tools used: Dosbox, Hex Workshop, Audacity_

__NOTE: _Grand Prix Circuit_, another Accolade game, seems to use this data layout in its MUSIC.BIN.__ <br>
__Copying the MUSIC.BIN to the TD2 folder and renaming it to SONGS.BIN will cause tracks to play just fine.__
__Therefore, I consider this documentation to be effective for GPC as well, however no further experimenting was done.__

__Please also note that when I talk about bytes being the first/second/last/etc. in relation to others, I'm talking as if we were looking at the BIN file in a Hex Editor, being read from left to right, from start to the end of the file.__

## About the file and its format
_SONGS.BIN_ contains the music tracks used by TD2: The Duel. At its core, the songs are similarly structured to tracker music; Short musical patterns arranged in a sequencer roll, making up a complete track. <br>
_(For your early information: the game contains 3 tracks in total. See chapter __"Group 1 - Track pointers"__ for their order of definition in the file, and when they're used.)_

The file format can be divided into the following 3 groups:
* __Track definitions__; Pointers to the address of the first sequencer entry in the track's sequencer roll.
* __Sequencing/arrangement__; Sequencer entries containing a 2 byte pointer to the address of the pattern data to be played, and a single byte of transpose value.
* __Piano Roll__; The pattern data itself, made up of note/duration byte pairs.

## Group 1 - Track pointers
The file starts with five groups of 4 bytes. In each of these groups, the first byte is a pointer to the start of the sequencer roll.
- The first and fifth groups are full zeroes. They _might_ be markers for the start and end of track data.
- The second group points to the title music of the game that plays in the intro/main menu/gas station intermissions.
- The third group plays during the ending cutscene, when the player is handcuffed.
- The fourth group also plays during the ending, but until the player is handcuffed.
 
_Excerpt from the beginning of SONGS.BIN:_ <br>
00 00 00 00 / 14 00 00 00 / 68 00 00 00 / 77 00 00 00 / 00 00 00 00

From this, we can see that the title song's first sequence is located at address 0x14, the 1st part of the ending song at 0x68, and the 2nd part at 0x77.

## Group 2 - Sequencer roll
The sequencer data is made up of groups of 3 bytes.<br>
- The first and second bytes serve as the low and high address _(respectively)_ of the pattern to be played.<br>
- The third byte is a transpose value that's applied to whole pattern.

__NOTE:__ During reverse-engineering experiments, I've found that a transpose value of 0 may break playback in-game. <br>

_The following example is taken from address 0x14 onwards (title song):_ <br>
86 00 05 / AC 00 05 / CE 00 05 / CE 00 05

Looking at the first group, we can see that it plays the pattern starting at address 0x0086, with a +5 semitone transposition.

## Group 3 - Pattern data
The pattern data is comprised of groups of 2 bytes, storing note ID and length respectively.<br>
Various control commands are also used in this area.

__NOTE:__ The first pattern __MUST__ start with the Tempo command __0xFE__ which will be detailed in the "__Control Commands__" list below.

### Musical notes
* Note ID: <br>
The lowest ID 1 corresponds to D#2. Each successive note is one semitone higher. <br>
Further experimentation is necessary to find the maximum value the game can handle, but somewhere around 79, playback might start to break. Try not to exceed this pitch, especially if you're transposing your pattern in the sequencer roll.

* Length: <br>
The note length is treated like a multiplier for the song tempo value, 1 being the smallest length unit you can use. <br>
_(Please see __0xFExx - Tempo command__ in the command list below for further details about how they work together.)_

### Control commands _(incomplete list)_

__0xFB00 - FURTHER RESEARCH NEEDED__ <br>
__Might be some kind of Stop or EOF marker...__ <br>
Only appears once at the end of SONGS.BIN, but doesn't seem to have any direct references to it from any of the original tracks. <br>
This command causes the music to go dead silent with no looping. Possibly an End-of-File marker?

__0xFC00 - End of Pattern marker__ <br>
Call this at the end of each pattern. Causes the game to jump to the next sequence in the sequencer list.

__0xFDxx - FURTHER RESEARCH NEEDED__ <br>
__Possibly an inverted Vibrato command?__ <br>
I'm uncertain about its true purpose. If you don't put this at the beginning of your pattern, the notes will have a really fast and shaky vibrato on them. <br>
The default TD2 songs call 0xFD with a value of 1, but testing any other value did not result in any deviation from its behavior. <br>
The playback would also sometimes get glitchy in Dosbox during my experiments without this command, but YMMV.

__0xFExx - Tempo command__ <br>
Contrary to conventional understanding of musical tempo, the 0xFE command operates inversely by defining the duration of each note in the track. A higher value results in longer notes overall, thus a slower tempo. <br>
The value roughly corresponds to the on-time of a note of length=1, and is approximately given in millisecs/10. <br>
For example, a value of 16 gives ~160ms on-time for Note Length 01, while a value of 7 results in a duration of ~70ms. <br>
To summarize: it can be deduced that the value of 0xFE times 10 is the duration of smallest note length unit in milliseconds. <br>
Do note however; this is based on my general time measurements of audio clips recorded from Dosbox. The game's sound methods were not exactly accurate, and the actual tempo might deviate from calculations.

__0xFF00 - Loop Music marker.__ <br>
Called at the very end of the whole song. Causes the game to loop back to the first sequence. <br>
In the original SONGS.BIN, there's a "pattern" that only contains this marker, and the songs have their last sequences simply pointing to its address. Possibly done to save a little on file size...?

_The following example is taken from address 0x86 onwards (First pattern of title song):_ <br>
FE 07 / FD 01 / 17 02 / 0B 02 / 1A 02 / 0B 02 / 19 02 / 0B 02 / 15 02 / 17 02 / 0B 02 / 12 02 / 1A 02 / 0B 02 / 19 02 / 0B 02 / 15 02 / 17 02 / FC 00

In the above example, you can observe the _Tempo_ command __(0xFE)__ as well as the _elusive_ __0xFD01__ being used at the beginning of the pattern, with the _End of Pattern_ marker __(0xFC00)__ being the last command. <br>
Everything in-between these commands is a note/length byte pair.
