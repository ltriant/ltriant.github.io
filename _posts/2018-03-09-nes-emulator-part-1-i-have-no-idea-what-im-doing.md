---
layout: post
title:  "NES Emulator, Part 1: I have no idea what I'm doing"
date:   2018-03-09 00:00:00 +1100
description: "I decided to jump into the deep end of emulator development by writing a NES emulator"
---
For a very long time I’ve wanted to have a go at writing an emulator, and for one reason or another I never did it, but a few weeks ago I decided to pull the trigger and start writing a NES emulator. And I’ve chosen to write it in Rust, as I have a goal this year to achieve a moderate-to-high level of competency in the language.

This is the first of a few blog posts to briefly document my journey as someone who has never written an emulator before.

I have very limited technical knowledge in this space, so maybe this will be useful to someone in the future.

## The Beginning

The [NesDev wiki](https://wiki.nesdev.com/w/index.php/Nesdev) is full of useful information, and is easily the most useful resource I’ve found so far.

On the wiki there are a ton of links and pages describing the basic architecture of the NES; the CPU is a 6502 processor, and there’s a PPU for rendering the graphics, and an APU for handling the audio. As far as I can tell (and I’ll correct this in future posts if I need to), the CPU simply writes to certain memory addresses that the PPU and APU then read from to do stuff. And in a single emulator cycle, based on the clock rates of each component, X number of CPU cycles are executed, Y number of PPU cycles are executed, and Z number of APU cycles are executed.

The first place I decided to dive in, after reading various threads on the [EmuDev subreddit](https://www.reddit.com/r/EmuDev/), was with the CPU implementation. I have zero experience with the 6502 beyond reading about the early days of Apple and Steve Wozniak’s stories from the Homebrew Computer Club, but it’s a thoroughly documented processor and the NesDev wiki has plenty of resources for it. It’s a pretty basic CPU to understand; there are three standard registers, a few status flags, a program counter, and a stack pointer. Nothing new if you’ve ever written any assembly, or had to debug stuff in gdb before.

Initially, I started from the bottom up, modelling the memory map and writing the CPU’s interactions with the memory. However, because of the different things that the memory maps to (controllers, joysticks, the game ROM data, etc.), I realised that I’d have to write a mountain of code before I’d even know if the design I was rolling with would even work, something that can be fairly unforgiving to redo in a Rust project because of how strict the compiler is. So I changed track a little by going top down instead, and started writing a very simple and incomplete parser for the iNES file format, which is the format that most ROMs are available in. There’s [a wiki page](http://wiki.nesdev.com/w/index.php/INES) for that too.

I then grabbed the nestest ROM from [the emulator test page](https://wiki.nesdev.com/w/index.php/Emulator_tests), and starting implementing new instructions and addressing modes every time I hit something my emulator didn’t know how to do.

The disassembly output that my emulator prints isn’t exactly the same as what the nestest log output is, and I’m not sure how worried I should be about that yet. Most posts that I find on the NesDev forums suggest that being *mostly* correct is good enough at the start, and to just use it as a guide. But it makes me feel all kinds of uncomfortable.

At this point in time, I’m still implementing instructions for the CPU.

![It’s alive! (sort of)](/assets/nestest.png)
	
*It’s alive! (sort of)*

## Addressing Modes

Addressing modes (which dictate how an opcode/instruction gets its operands/arguments) confused the hell out of me at first, but I understand it a lot better after following through [the addressing modes page on the NES Hacker wiki](http://www.thealmightyguru.com/Games/Hacking/Wiki/index.php/Addressing_Modes).

## Learn from Others

The last thing that has been super useful is reading the source of other emulators, and any blog posts people may have written.

[Michael Fogleman’s article about writing a NES emulator](https://medium.com/@fogleman/i-made-an-nes-emulator-here-s-what-i-learned-about-the-original-nintendo-2e078c9b28fe) was great source of lower-level information to start with, and [the code](https://github.com/fogleman/nes) is super easy to follow, even if you’re not overly familiar with Go.

[This Rust emulator](https://github.com/xSke/nes-emu) has also been useful, when I’m trying to figure out how best to model certain things in Rust.
