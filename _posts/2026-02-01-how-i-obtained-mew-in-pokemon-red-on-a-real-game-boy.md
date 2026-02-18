---
path: "/posts/5"
date: "2026-02-01"
title: "How I Obtained Mew in Pokémon Red on a Real Game Boy"
icon: "fa-dragon"
redirect_from:
  - /posts/5/
layout: post
---

<img src="/assets/5_0.png" style="width: 40%; height:240px; object-fit: cover;" alt="">

If you grew up with **Pokémon Red/Blue/Yellow**, you probably remember *Mew*. Not the one you caught (because you didn’t) but the one you *heard about*. The one your friend's cousin totally had. The one that was supposedly hiding [under a truck](https://gaming-urban-legends.fandom.com/wiki/Mew_Under_the_Truck) if you used Strength at just the right moment. For most players, Mew was unobtainable unless you attended a [distribution event](https://bulbapedia.bulbagarden.net/wiki/List_of_European_language_event_Pokémon_distributions_in_Generation_I) or used a GameShark (which kinda feels like cheating). More recently, methods exist of manipulating the memory of the game using [in-game glitches](https://bulbapedia.bulbagarden.net/wiki/Mew_glitch) to cause Mew to appear (with some limitations, see end of the article)! Having [previously poked around the trade logic of these Pokemon games](/2015/05/26/arbitrary-code-execution-in-pokemon-red/), I wanted to see if I could "trade" Pokemon to myself and create a Mew from thin air.

# What Pokémon Trading Actually Looks Like

Pokémon Red/Blue/Yellow’s trade system is quite simple and has been well documented by others.  When two Game Boys connect to trade Pokémon over a link cable, the first system to initiate the interaction becomes the primary Gamebnoy, while the other acts as the secondary. Both sides repeatedly transmit menu state information until the `Trade` option is selected, where then both players begin to exchange party information. Each Gameboy sends its data to the other in three consecutive blobs: a 10-byte random seed (used to synchronize random events and animations), a 418-byte party data structure, and a patch list used to work around limitations of the Game Boy’s serial protocol (Certain byte values like `0xFE` have special meanings on the link cable and cannot be transmitted directly; whenever those values appear inside the party data, they are omitted during transmission and later reinserted using a list of offsets). Each data block is preceded by an arbitrary number of `0xFD` preamble bytes to mark boundaries.

## Party Data Layout
The party data itself is sent as a fixed-layout block of memory (credit to [Bulbapedia](http://bulbapedia.bulbagarden.net/wiki/Pok%C3%A9mon_data_structure_in_Generation_I)):

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

<p></p>
Crucially, the trade system does not attempt to validate where this data comes from. It only cares that it arrives in the correct format and passes basic consistency checks. If the received bytes are structurally valid, the game accepts them without question. Editing these values can potentially allow us to "trade" a Mew!

# The Missing Link

<img src="/assets/5_1.jpg" style="width: 100%; height:240px; object-fit: cover;" alt="PC connected to Gameboy with cable">

Pokémon trades were designed to happen over the Game Boy Link Cable — a simple, peer-to-peer serial interface intended to connect two handhelds sitting next to each other. In my [previous post](/2015/05/26/arbitrary-code-execution-in-pokemon-red/), I had used an Arduino to mimic this serial communication and hardcoded what to send on the microcontroller itself. I wanted a bit more flexibility and wondered if I could send data from my computer instead, to make this more extensible and easier to iterate with. To do that, I needed a device that physically connects to the Gameboy's Link Port, understands the Game Boy’s serial timing, and exposes the exchange to a modern computer in a controllable way. So I designed one:

<a href="https://www.tindie.com/products/vaguilar/gblink-usb-to-game-boy-link-port/" target="_blank"><img src="/assets/5_2.jpeg" style="width: 100%; height:240px; object-fit: cover;" alt="GBLink PCB"></a>

The result is a small PCB that bridges the Game Boy Link Port to a standard UART interface, which can then be connected to a PC over USB. A low-power ATtiny412 microcontroller handles real-time link protocol timing. The Game Boy sees a perfectly valid link partner, while the computer sees a simple serial device. From the Game Boy’s perspective, it’s just trading with another handheld. From the PC’s perspective, it’s just talking bytes over a serial port. I called it the GBLink.

⭐️ I made a small batch of these GBLink devices that you can buy on my <a href="https://www.tindie.com/products/vaguilar/gblink-usb-to-game-boy-link-port/" target="_blank">Tindie page</a>! 


# Trading a Mew That Never Existed

Armed with a newly built GBLink, I wrote a program on the host (PC) side that simply echoes back whatever it receives from the real Gameboy. This allows one to get pretty far in the trading process. It's enough to get a dump of all the blobs that are exchanged between Gameboys for trading and allowed me to start examining them:

```
...
 60 60 00 00 fd fd fd fd  fd fd fd fd e6 6b f2 7b  # random seed
 05 91 1f af 41 d5 fd fd  fd fd fd fd fd fd fd 81  # party starts
 8b 94 84 50 86 80 91 98  50 89 01 b1 ff 00 00 00 
 00 00 b1 00 18 00 00 15  15 2d 21 27 00 00 d6 d3  # pokemon 1
 00 01 1f 00 c3 00 f0 00  cc 01 1d 00 c3 18 b9 23
 1e 00 00 07 00 18 00 0c  00 0f 00 0c 00 0d 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 81 8b 94 84 50 86  # OT
 80 91 98 50 89 00 00 00  00 00 00 00 00 00 00 00 
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00
 00 00 00 00 00 00 00 00  00 00 00 00 92 90 94 88  # nicknames
 91 93 8b 84 50 50 50 00  00 00 00 00 00 00 00 00
...
```
Full trade dump available [here](/assets/cable_club_bytes_received.txt).

At first glance, it might seem like obtaining Mew is as simple as swapping the Pokémon <a href="https://bulbapedia.bulbagarden.net/wiki/List_of_Pokémon_by_Kanto_Pokédex_number" target="_blank">species ID</a> of a replayed trade exhange, and that's only somewhat true. But we also need to update a few other fields like a the Pokemon's `type` and their list of moves to line up with what a <a href="https://pokemondb.net/pokedex/mew/moves/1" target="_blank">"real" Mew would have</a>. Also, to make the Mew compatible with other games like Poké Transporter, we must modify the original trainer names and IDs to <a href="https://www.reddit.com/r/pokemon/comments/5q4meg/how_to_trick_pokebank_into_thinking_your_gen_1/" target="_blank">certain values</a>, otherwise this Mew would not be tradable. I also wanted the trade to be somewhat robust and extensible for others to use.

<a href="/assets/5_4.svg"><img src="/assets/5_4.svg" style="height: 480px;" alt="State machine diagram of happy path of Pokemon Trade"></a>

After mapping the blobs that are exchanged, as well as deciphering some menu options that are used to transition before and after trading, I was able to start putting together a state machine for Pokemon trades. I wrote a Python program for the host side that opens a UART connection for the GBLink, waits for the other Pokémon game's link handshake, and enters the state machine to handle the exchanging of data. The cartridge responds to the output and progresses its own trading state machine, until we finally inject a custom party data blob with a handcrafted Mew from the host onto the Gameboy:

<img src="/assets/5_5.jpeg" style="width: 100%; height: 360px; object-fit: cover;" alt="PC connected to Gameboy with cable">

And then finally running the trade for real. I offer a level 3 Rattata in exchange for the GBLink's mythical Pokemon. The animations play. The trade completes. Nothing crashes. And suddenly, there’s a Mew in my party. Just a normal trade with a partner that doesn’t exist.


<img src="/assets/5_6.jpeg" style="width: 100%; height: 360px; object-fit: cover;" alt="PC connected to Gameboy with cable">

## Closing Thoughts

Back when these games were first released, Mew felt mythical. Today, with a little hardware and a lot of curiosity, it turns out Mew was never hiding under a truck but in in plain sight, waiting for the right bytes to be sent over a link cable.

Code available on <a href="https://github.com/vaguilar/gblink" target="_blank">Github</a>.