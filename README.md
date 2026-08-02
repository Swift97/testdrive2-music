# Reverse-engineering _SONGS.BIN_<br>from the MS-DOS version of Test Drive 2

_Below you will find the results of my research since the beginning of this reverse-engineering project in 2017._
_This is a hobby project that I came back to after 9 years of hiatus, after I found some old screenshots and text files._ <br>
_Given this, possible updates to this page might be slow to come. I am publishing this only to , and just in case someone else also gets the idea to play around with this old game's files._

__NOTE: _Grand Prix Circuit_, another Accolade game, seems to share a similar data layout in its MUSIC.BIN at first glance.__ <br>
__You may consider this documentation as starting grounds if you want to work on that game instead, but it hasn't been tested.__

## About the file and its format
_SONGS.BIN_ contains the music tracks used by TD2: The Duel. At the core, the songs are similarly structured to tracker music, with patterns referenced by sequencers.<br>
The game has 3 tracks in total, each of them being built up from patterns, which are then arranged in a sequencer fashion.

Thus, the format can be divided into the following 3 groups:
* Pointers to the first sequence in the sequencer
* Sequencer roll containing pointers to the pattern data
* The pattern data itself.

## Group 1 - Track pointers
The file starts with five groups of 4 bytes. In these groups, the first address is a pointer to the start of the sequencer roll.
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
Various control commands are also stored in this area.

__NOTE:__ The first pattern __MUST__ contain the control commands __0xFE__ and __0xFD__. See "Control Commands" list for details.

* Note ID: <br>
An ID of 0 inserts silence. _UNTESTED: You __might__ be able to create pauses between notes with this_ <br>
An ID of 1 corresponds to D#2. Each successive note is one semitone higher. <br>
* Length: <br>
A multiplier for the song tempo value _(See control command 0xFE)_, 1 being the smallest value you can use. <br>
Larger value = longer note.

### Control commands _(incomplete list)_

__0xFB00 - Undocumented, untested.__ <br>
Only appears once at the end of SONGS.BIN, I have no idea what it does.

__0xFC00 - End of Pattern marker__ <br>
This must be the last pair of each pattern. Causes the game to jump to the next sequence in the sequencer list.

__0xFD01 - Undocumented__ <br>
I'm uncertain about its true purpose. Default TD2 songs call this command at the start. <br>
Without this, the notes have some kind of weird vibrato on them. I've also had the playback glitching out during my experiments when I forgot to set it, but YMMV.

__0xFExx - Tempo command__ <br>
This sets the tempo by modifying the on-time of each note in the track. This value corresponds to the on-time of a note of length=1, given in millisecs/10. <br>
For example, a value of 0x10 (16) gives 160ms on-time for Note Length 01, while a value 0x07 (7) results in a duration of 70ms. <br>
Thus it can be deduced that the value of 0xFE times 10 is the duration of smallest note length unit in milliseconds

__0xFF00 - End of Track marker.__ <br>
Called at the very end of the whole song. Causes the game to loop the song.

_The following example is taken from address 0x86 onwards (First pattern of title song):_ <br>
FE 07 / FD 01 / 17 02 / 0B 02 / 1A 02 / 0B 02 / 19 02 / 0B 02 / 15 02 / 17 02 / 0B 02 / 12 02 / 1A 02 / 0B 02 / 19 02 / 0B 02 / 15 02 / 17 02 / FC 00

In the above example, you can observe the _Tempo_ command __(0xFE)__ as well as the _undocumented_ __0xFD01__ being used at the beginning of the pattern, with the _End of Pattern_ marker __(0xFC00)__ being the last command. <br>
Everything in-between is a note+length pair.
