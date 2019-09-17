---
layout: post
title:  "NES Emulator, Part 3: It's alive!"
date:   2019-09-04 00:00:00
description: "The final leg of my journey to emulator city"
---

Before I get too dramatic about how long it's taken to get to this point, progress has been very on and off - mostly off, due to my wife and I buying a house, making changes to the house, starting a new job, oh and having a baby. But in the midst of all of that, diving into how graphics work in the NES - and how various emulators have implemented it - has been a fun adventure.

There's not a lot of material out there for brand new emulator developers to learn how to implement the PPU, so I'm going to provide some of the key pieces of information that _I_ needed to finally get to this point.

![Super Mario Bros](/assets/super-mario-bros.gif){:.post-image}

## PPU Cycle Timing

A confusing point that I got to early on was the warm-up state of the PPU; there are a few places in the various bits of documentation that say that the PPU has to "warm-up". I had no idea what that meant, and it was hard to figure out by searching around, and even more so after looking at other emulators' code and seeing no references to this.

It seemed like my emulator had to do something, or wait for something, in order for the PPU to be considered "warmed up". But that's not the case; it's the NES games that are waiting for the PPU to "warm up", and _that_ warming up is something that the PPU will naturally do by triggering the vertical blanking (vblank) status twice.

How does the vblank status get triggered twice?

For each cycle that the PPU processes in its step function, there is some internal state that needs to be managed. To understand what that state is, there are a few details that emulator developers need to know.

The NES displays a 256x240 pixel image to the player.

Which pixel the emulator should render is determined per PPU cycle. In simple terms: from left to right and top to bottom, the first PPU cycle renders the first pixel on the first line of the frame (the top-left corner of the screen), the second cycle is the second pixel of the first line of the frame, etc, until all pixels have been rendered.

Each of these lines of pixels are referred to as scanlines.

In reality, the PPU actually processes 341x262 "points" of data, but only 256x240 are displayable to the user. When we reach the 241st scanline, the game enters the [vertical blanking interval](https://en.wikipedia.org/wiki/Vertical_blanking_interval), and the emulator needs to flag that the vblank state has been reached by reflecting this in [the PPUSTATUS register](http://wiki.nesdev.com/w/index.php/PPU_registers#Status_.28.242002.29_.3C_read).

Once the emulator reaches the 262nd scanline, the emulator clears the flag that indicates we're in the vblank state (again, this is in the PPUSTATUS register), and then the emulator will wrap back around to the 1st scanline again to start the whole process over.

In this 241-262 scanline section, the PPU doesn't mess around with memory at all, so the game is free to mess around with memory that may affect the image on screen, e.g. preparing data for the next frame, like changing color palettes, loading new sprites, etc.

Two cycles of this are required before a NES game considers the PPU "warmed up".

Similarly, the 257-341st points on each scanline have their own relevance.

These details (and much more) are part of the cycle timing accuracy that emulator developers need to solve for their emulators.

The NesDev wiki has a page on [PPU rendering](http://wiki.nesdev.com/w/index.php/PPU_rendering), and the section on line-by-line timing is of great use. Once I started looking into those details, I referenced [this timing diagram](http://wiki.nesdev.com/w/index.php/File:Ntsc_timing.png) _a lot_.

The timing diagram is where all the details of fetching the necessary data to render pixels to the screen are. From this diagram comes the next steps for the PPU: fetching nametable and attribute data at the correct time, and rendering pixels.

## Rendering

dustmop's articles (parts [1](http://www.dustmop.io/blog/2015/04/28/nes-graphics-part-1/), [2](http://www.dustmop.io/blog/2015/06/08/nes-graphics-part-2/), and [3](http://www.dustmop.io/blog/2015/12/18/nes-graphics-part-3/)) give a fantastic overview of how graphics work on the NES.

[This video from The 8-bit Guy](https://www.youtube.com/watch?v=Tfh0ytz8S0k) and [this video from Retro Games Mechanics](https://www.youtube.com/watch?v=wfrNnwJrujw) also give some extra details that might be useful.

These references do a great job of explaining the components used in the PPU (nametables, attributes, palettes, OAM, etc.) and understanding what these things are is essential to debugging the PPU.

For example, I had a perfectly rendered tile appearing in the wrong X position on the screen. Because it's rendered perfectly fine, and the Y position is correct, I know the first few bytes of OAM are correct (because they dictate the Y position, and the palette to use), but the last byte of OAM is probably wrong (which dictates the X position), and, in this case, OAM is written by a DMA operation. Not understanding the layout of OAM and how/when OAM is populated would've made this a much harder problem to solve.

In addition to the various pages on the NesDev wiki, I also had the code from a few other emulators open to help understand how rendering worked. For me, this made the process much faster than it otherwise would have taken.

## Donkey Kong

My first major goal was to render Donkey Kong. Along with Balloon Fight, these are reported to be the easiest games to emulate, so they seemed like a great place to start.

After the first cut of the rendering code, I had this on my screen:

![First Screen of Donkey Kong with glitches](/assets/donkey-kong-1.png){:height="500px" .post-image}

It looked like it was kinda on the right path, and seeing this filled me with a ton of motivation and excitement, even if it was wrong.

After fixing a timing issue (where I was double-incrementing the PPUADDR value on a PPUDATA write), I implemented sprite rendering, and finally I reached the single most exciting moment that I had been chasing since deciding to take on this project:

![Donkey Kong](/assets/donkey-kong.gif){:.post-image}

The moment of child-like glee was worth the pain it took to get there :)

## Reflecting on Emulator Development

One of the original reasons I wanted to write an emulator was because it was something I had been exposed to through various IRC communities that I've been apart of since I was a teenager in the late 90s and early 2000s, and it was also of interest to me because of my own life-long love for retro gaming dating as far back as getting the original GameBoy for Christmas (which I then complained about, because the box that it came in was smaller than the box my brother's Christmas present came in).

It's also an enlightening and fun experience to travel back in time and appreciate the history and architecture of old systems.

The 6502 CPU was a lot of fun to implement, to the point that I would recommend writing a 6502 emulator on its own as a fun project, if a full gaming emulator seemed too big of a project for the amount of time someone may have available. It's a project that could be knocked out in a fairly short period of time, and has [a fun history](https://en.wikipedia.org/wiki/MOS_Technology_6502#Computers_and_games) in retro gaming and computing.

---

After my emulator was successfully reproducing the output from the nestest ROM with perfect accuracy, I thought that my CPU implementation was pretty much done. But after I started running [other emulator tests](https://wiki.nesdev.com/w/index.php/Emulator_tests), I discovered that I still had all kinds of issues, especially related to cycle timing in trickier cases, such as crossing page boundaries, branch statements taking extra cycles, vblank/NMI timing, and a particularly nasty double overflow/underflow bug in my SBC implementation that caused Super Mario Bros to freeze when loading the second nametable.

In the process of fixing these various issues, I learned the most about how the PPU actually works, which made debugging subsequent issues easier and easier. Whereas, when I was starting to write code to render pixels to the screen, I was blindly following what the docs said, without really understanding why.

These debugging stories were incredibly rewarding, as they are in all aspects of software development, and I've kept a small journal of them.

---

Writing my own emulator always seemed like it was a space reserved for the most hardcore of systems programmers, and that felt like a circle I wasn't qualified enough to join.

It's not hard to see how people can feel that way, but it's a ridiculous thought, and in 2019 there are so many resources available for getting into emulator development; [the EmuDev subreddit](https://www.reddit.com/r/emudev), the Freenode ##emudev IRC channel, [the EmuDev Discord server](https://www.reddit.com/r/EmuDev/comments/8xp8h0/remudev_now_has_a_discord/), blogs, other peoples' emulators, and more.

Laptops and PCs are fast enough that emulators don't have to be written in C and assembly anymore either; a quick scan of GitHub shows NES emulators written in JavaScript, Go, Ruby, Rust, Nim, C#, Java, Python, Clojure, and Lua.

We really are standing on giants shoulders in this respect. I can't imagine what this experience would've been like 20 years ago.

Something that is super valuable for a novice emulator developer would be a simple guide with an opinionated path to follow, like [yizhang82's overview](https://yizhang82.dev/nes-emu-overview). This even contains information on things that aren't mandatory for a basic emulator too (like scrolling). This can be difficult to figure out on your own, but is nice to know upfront when understanding how much time you may or may not want to put into your emulator.

![Ice Climber](/assets/ice-climber.gif){:.post-image}

## Reflecting on Rust

The biggest obstacle in using Rust in the beginning was deciding on the system design. And a lot of that was dictated by how much (or how little) I knew of the Rust language.

I started out using lifetimes, because that seemed to make sense; I had a Console object, which contained a CPU object, a PPU object, and a Memory object, and I needed each of these objects within the Console to talk to each other. This model mapped what it looked like in my mind, because it was more closely mapped to what the physical architecture looks like.

But that caused all sorts of heartache with the compiler, so I threw away the design with lifetimes and ended up with a design that was much more simple in terms of Rust feature usage; there is a Console object, which contains a CPU object, which contains a Memory object, which contains a PPU object. These object relationships didn't make sense to my mental model, but it worked without fighting the lifetime errors.

Traits were another consideration.

When I first looked at other emulators that were written in Rust, the authors had decided to heavily sprinkle traits throughout their projects. This also made understanding the source code of those emulator projects much harder.

In the end, the hardest thing about using Rust was deciding how much - or how little - of the language to use. In the end I opted to use the fewest language features possible to keep the codebase simple to understand for others, and also for myself when I come back to it in time.

Despite these issues, there were plenty of positive experiences, especially with the compiler warnings and errors when doing bit-wise arithmetic mixing u8, i8, u16, i16, and u32s. I can't imagine the world of hurt this could have caused had I been doing all this arithmetic in C, triggering untold amounts of undefined behaviour.

![Balloon Fight](/assets/balloon-fight.gif){:.post-image}

## The End?

Initially, my goals were to play Donkey Kong and Super Mario Bros without any errors, which I achieved. And I've gone a little further than that, adding support for SxROM (mapper 1) games like The Legend of Zelda and Ninja Gaiden (or Shadow Warriors for me, being from Australia). So next up, I'd like to add the ability to save game state.

After that, I'm happy to put this project to the side for a while to focus on other projects, some of which, funnily enough, involve writing more emulators.

If I get inspired, I'll look at implementing the APU so that I can hear some funky 8-bit music, but that's low on the priority list at the moment.

For now, I'm happy to reap the benefits of my time and just play games on my own emulator for a while.

Full code is available at: [https://github.com/ltriant/nes](https://github.com/ltriant/nes)
