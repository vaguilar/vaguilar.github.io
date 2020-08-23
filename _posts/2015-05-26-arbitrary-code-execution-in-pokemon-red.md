---
path: "/posts/1"
date: "2015-05-26"
title: "Arbitrary Code Execution in Pokemon Red"
icon: "fa-gamepad"
redirect_from:
  - /posts/1/
layout: post
---

  <iframe
    class="center yt-video"
    width="560"
    height="315"
    src="https://www.youtube.com/embed/m3e_SyhE3xc"
    frameborder="0"
    allowfullscreen
  ></iframe>

The Gameboy Color is a nifty piece of hardware. After reading about several
projects related to the original Pokemon games, I decided to give it a go
myself. I dusted off my old Gameboy Color and ordered a copy of Pokemon Red
off eBay and set off to do _something_.

While I waited, I began to sift through the awesome [Pokemon R/B/Y disassembly](https://github.com/iimarckus/pokered/) project, and found the section dedicated to the Cable Club. The Cable Club is the room inside Pokemon Centers where you can trade Pokemon with another person using a link cable. This part intruiged me since data is sent over the link cable and curiosity lead me to search for some way to manipulate the data other than [spoofing a trade](http://www.adanscotney.com/2014/01/spoofing-pokemon-trades-with-stellaris.html) . Using the [BGB emulator](http://bgb.bircd.org/) and its documentation on its link cable protocol, I slapped together a [Python script](https://github.com/vaguilar/pokemon-red-cable-club-hack/tree/master/py/) to interact with the Pokemon ROM.

# The Exchange

![](/assets/1_1.png)

The initial communication between Gameboys is outlined [here](http://www.adanscotney.com/2014/01/spoofing-pokemon-trades-with-stellaris.html) pretty well. It's not very interesting or relevant to the exploit, but basically the first Gameboy to talk to the Cable Club lady is the master (the emulator or the actual Gameboy later, referred to as Player 1 from here on out) while the other is the slave (Player 2, the script/Arduino). The menu options are then transmitted back and forth repeatedly as players choose what to do. Once the trade option is selected you are taken to a room to begin sending the other player your Pokemon data. It is sent in three parts:

```
Random seed value: 10 bytes
Pokemon party structure: 415 bytes
Patch list: 196 bytes
```

The random seed value is used so random events have the same outcome on both Gameboys. The pokemon party data section includes the other player's name, party (species of Pokemon, nicknames, HP, attacks etc.), as well as the original trainer's names. And finally, since the byte `0xFE` has a special meaning when received as serial data (it is the designated "no data" byte, along with `0xFD` being the preamble byte), the patch list is simply the offset of bytes in the party that should have the byte `0xFE`. When these are transmitted, the sections of data are preceded by an arbitray number of `0xFD` bytes.

The party structure is organized like so: (credit to [Bulbapedia](http://bulbapedia.bulbagarden.net/wiki/Pok%C3%A9mon_data_structure_in_Generation_I))

| Name                   | Length (bytes) |                                Comment                                 |
| :--------------------- | :------------: | :--------------------------------------------------------------------: |
| Player Name            |       11       |                         Terminated with `0x50`                         |
| Size of Pokemon Party  |       1        |                         Normally max size of 6                         |
| Pokemon Species IDs    |       7        | List terminated with `0xFF`, padded with `0x00` if less than 6 Pokemon |
| Pokemon #1             |       44       |                          First Pokemon's data                          |
| ...                    |       44       |                                  ...                                   |
| Pokemon #6             |       44       |                          Last Pokemon's data                           |
| Nicknames              |       66       |                 11 bytes each, terminated with `0x50`                  |
| Original Trainer Names |       66       |                 11 bytes each, terminated with `0x50`                  |
| ???                    |       3        |                                Unknown                                 |

When this party data is sent over the link cable, the Player 2's name is
stored separately in memory at address `0xD887` and the rest of the data at
`0xD89C`:

```
0xD887 | 80 - First character of the Player 2's name
...
0xD89C | 06 - Number of Pokemon in Player 2's party
0xD98D | 01 - Species ID of 1st Pokemon
0xD98E | 02 - Species ID of 2nd Pokemon
0xD98F | 03 - Species ID of 3rd Pokemon
0xD990 | 04 - Species ID of 4th Pokemon
0xD991 | 05 - Species ID of 5th Pokemon
0xD992 | 06 - Species ID of 6th Pokemon
0xD993 | FF - List terminator
0xD994 | 01 - Start of 1st Pokemon data
...
```

# The Buffer Overflow

![](/assets/1_2.png)

When you finally enter the trading room, each of the player's original Pokemon
names are printed to the screen (not their nicknames for some reason). The way
older video game consoles displayed some graphics is by using tiles. The video
buffer is simply an array of tile IDs that are to be displayed on screen. This
applies to the text on screen as well. Each letter is its own tile, and as a
result the function to write strings to the screen simply writes bytes to the
corresponding area of video buffer starting at `0xC4A0` (1 byte per tile, 20x16
in total):

```asm6502
...
	hlCoord 2, 1	; $C4A0
	ld de, wPartySpecies ; $D89D
	call TradeCenter_PrintPartyListNames
...

TradeCenter_PrintPartyListNames:
	ld c, $0
.loop
	ld a, [de]
	cp $ff
	ret z
	ld [wd11e], a
	push bc
	push hl
	push de
	push hl
	ld a, c
	ld [$ff95], a
	call GetMonName
	pop hl
	call PlaceString
	pop de
	inc de
	pop hl
	ld bc, 20
	add hl, bc
	pop bc
	inc c
	jr .loop
```

Or in C, roughly:

```c
// TradeCenter_PrintPartyListNames(0xD89D, 0xC4A0);

void TradeCenter_PrintPartyListNames(char* start_ids, char* dest) {
	char* current_id = start_ids;
	while (*current_id != 0xFF) {
		char* name_str = GetMonName(*current_id); // Returns pointer to a string
		PlaceString(name_str, dest); // Essentially strcpy, but 0x50 terminated
		dest += 20; // Move to next line in buffer
	}
}
```

The TradeCenter_PrintPartyListNames function is passed the address of Player
2's first Pokemon ID and begins to write all of the names to the video buffer.
But the way the function is written, a Pokemon's name can be written outside
the intended area of memory if the byte `0xFF` is not encountered within 7
bytes. Past that, we can write outside of the video buffer all together and
even further down memory to where the stack is located (`0xD000-0xDFFF`). What
good is writing Pokemon names to the stack though?

```wasm
GetMonName:: ; 2f9e (0:2f9e)
	push hl
	ld a,[H_LOADEDROMBANK]
	push af
	ld a,BANK(MonsterNames) ; 07
	ld [H_LOADEDROMBANK],a
	ld [$2000],a
	ld a,[wd11e]
	dec a
	ld hl,MonsterNames ; 421E
	ld c,10
	ld b,0
	call AddNTimes
	ld de,wcd6d
	push de
	ld bc,10
	call CopyData
	ld hl,wcd77
	ld [hl], "@"
	pop de
	pop af
	ld [H_LOADEDROMBANK],a
	ld [$2000],a
	pop hl
	ret
```

C pseudo-code:

```c
char* GetMonName(char id) {

	char* buffer, current, i;

	// Switch to the bank that stores the Pokemon names
	// along with other data...
	switchBank(7);

	// Names start at 421E and contain 10 bytes each
	current = 0x421E + (id * 10);
	buffer  = 0xCD77; // buffer

	// CopyData, essentially memcpy
	for (i = 0; i < 10; i++) {
		buffer[i] = current[i];
	}

	buffer[i] = 0x50; // Make sure string is terminated

	return buffer;
}
```

The GetMonName function looks up the Pokemon's name in ROM bank 7 (one of the
extra sections of memory of the Gameboy) and copies it into a buffer. There
are only 151 pokemon in the game and it makes sense that there should only be
that many names too. Space is quite limited in Gameboy games so there is some
unrelated code/data directly after the table of names:

<script src="http://gist-it.appspot.com/github/iimarckus/pokered/blob/c2efe700ac1c5cca88bac710b98388a99665741e/main.asm?slice=4939:4950"></script>

This data can be copied to the buffer just like a Pokemon name if we pass a
value greater than 151 for a species ID. We can now wreck havoc on the stack
and write whatever 10 byte sequence we find in the table to certain parts of
the stack. We'll now try to overwrite something interesting.

Setting a breakpoint inside the PlaceString function, we see that the return
address stored in the stack is at address `0xDFDD`. When the PlaceString
function finishes and executes its RET instruction, it will jump to this
memory address and continue executing from there. Could we possibly write to
this address and have the Gameboy execute other code elsewhere?

# I choose you, @(\*#&!

```c
// TradeCenter_PrintPartyListNames(0xD89D, 0xC4A0);

void TradeCenter_PrintPartyListNames(char* start_ids, char* dest) {
	char* current_id = start_ids;
	while (*current_id != 0xFF) {
		char* name_str = GetMonName(*current_id); // Returns pointer to a string
		PlaceString(name_str, dest); // Essentially strcpy, but 0x50 terminated
		dest += 20; // Move to next line in buffer
	}
}
```

Taking a look at this code again, we can write up to 10 bytes every 20 bytes
starting at `0xC4A0`. Doing some math `(20n + 0xC4A0)`, we see that we can indeed
start writing at address `0xDFD6` which is 7 bytes before the return address (n
= 346 which is less than the total party size of 415). Perfect! Now, what are
the chances that an address to some place useful is located in the unrelated
data past the Pokemon names, in little endian format no less?

```
...
0xcc   4A 23 23 78 22 79 22 C9 17 AD
0xcd   5D 22 50 CD 3C 3C 21 26 D1 CB
0xce   EE 21 96 D7 CB 86 21 **A3 D7** CB
0xcf   8E 21 34 4A FA 39 D6 C3 97 3D
0xd0   38 4A 73 4A 06 2B CD 93 34 C0
...

(left column is the species ID and the corresponding "name")
```

The address `0xD7A3` is at the right location to be injected into the return
value. Player 2's name just so happens to be located at `0xD887`, about 230
bytes after the potential return value. By sheer luck, the portion of memory
between just happens to be filled with mostly 0's (the nop opcode!) and makes
a nice nop sled for us! What are the odds?

There is still one issue. To get this far down the stack we spray Pokemon
names all over the place (since we have to print 346 of them to get this far)
and potentially over important parts of the game. Let's do something about
that.

```
...
0xe1   49 3C C3 D7 24 17 BF 55 23 50
0xe2   17 43 56 23 0B 50 17 52 56 23
0xe3   **50** 17 7C 56 23 50 17 9F 56 23
0xe4   **50** 17 20 57 23 50 05 06 05 C0
0xe5   41 82 50 0E 4B 00 0A 54 FA 4B
0xe6   D7 CB 77 C4 76 50 3E 01 EA 0C
...

(left column is the species ID and the corresponding "name")
```

There are two "names" that begin with `0x50`, the string terminating byte.
Calling PlaceName on these means that nothing will be written to memory for
that species ID. At this point we have enough to write a bootstrapping program
to execute arbitary code.

# The Payload

By sending Player 2's name as assembled Z80 code, we can jump to any part of
memory (and fix the stack if we'd like to return to the game). And as a
consequence of our crafted party that lacks any `0xFE` bytes, the patch list
will be mostly empty, save for two `0xFF` terminating bytes at the beginning. We
can then stash our shell code here. We include a jump instruction to the patch
list in Player 2's name, `C3 D6 C5` (JP \$C5D6) and our party looks
like this:

```
# Fix stack to previous state, jump to 0xc5d6, the patch list
name         | f8 00 36 fd 01 3e 58 c5 c3 d6 c5
party size   | 06                   # not actualy used, can be any value
species IDs  | 15 15 15 15 15 15    # arbitrary species IDs, in this case Mew's
pokemon data | e3 (346x) ... ce ff 00 00 00 ...
# crafted party structure, terminated with 0xff byte and followed by 0's
```

For the shellcode, there are a couple of assemblers for the
[Gameboy's Z80 like processor](http://marc.rawer.de/Gameboy/Docs/GBCPUman.pdf)
, but I found that only [WLA DX](https://github.com/vhelin/wla-dx) allows you to easily
assemble code for the WRAM section of memory (`0xC000-0xCFFF`). I have put
together a simple animated
[hello world](https://github.com/vaguilar/pokemon-red-cable-club-hack/tree/master/asm/hello/hello.asm)
proof of concept with a simple build file to put it all together. There are
only 194 bytes available in the patch list for our shell code but its enough
to do some interesting things. One thing to remember is that we can't send any
`0xFE` bytes in the patch list or party, so any instructions using that byte
can't be sent (the compare immediate value instruction is one of them).

![](/assets/1_3.png)

# Now for Real

So far we've been using an emulator for this hack, but wouldn't it be much
more satisfying to see this happen on actual hardware?

<img class="center" src="/assets/1_4.jpg" width="400" />

The Gameboy uses a primitive serial protocol for the link cable that runs at
5V. The four
[main connections](http://www.hardwarebook.info/Game_Boy_Link) are
serial in, serial out, clock and ground. The master Gameboy initiates the
serial transfer and a byte is simultaneously pushed out as one is read in.
This can be implemented easily with a microcontroller and has been done so
[already for the Arduino](https://github.com/EstebanFuentealba/Arduino-Spoofing-Gameboy-Pokemon-Trades). Building off that project, it was easily modified to send the patch list as
well. All that was required was cut up a Link Cable that I got off eBay for
\$2, no resistors or anything much else. I recommend you break the plastic off
one of the connectors and then pry the metal to expose the easily connectable
prongs.

# Conclusion

I didn't have as much time as I'd like to explore possibilites of taking this
further, but having done nothing like this before this project was quite an
adventure. It can be extended to send over even larger programs as a payload,
as there are portions of memory that can be overwritten (without side effects,
like Player 2's party data) by sending a bootstrap program that requests more
data over serial. It can be used as a "GameShark" that allows you to modify
parts of the game (more money, add arbitary Pokemon to your party, etc). You
could dump your save file and keep a backup on your computer. Or even dump the
whole ROM!

Even though this project has been written for Pokemon Red, I'm sure it could
run on Pokemon Blue, maybe even Yellow. If you do anything interesting, I'd
like to hear about it.