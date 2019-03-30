---
layout: post
title:  "NES Emulator, Part 2: I sort of know what I’m doing"
date:   2018-06-29 00:00:00 +1100
description: "Finishing the CPU implementation, diving into the PPU, and reflecting on choosing Rust"
---
In my first post about my journey to the center of the NES, I was at the point where I was still working on the CPU; implementing new addressing modes and instructions as I made my way through the nestest ROM. Well, I finally finished the CPU, including a handful of the illegal opcodes. The last of the illegal opcodes just need some placeholders, because, as I understand it, very few games use them.

I also pushed the current state of [my code to GitHub](https://github.com/ltriant/nes).

## The Golden Log

Previously, I mentioned an issue about checking my emulator’s output against the nestest golden log output:
> The disassembly output that my emulator prints isn’t exactly the same as what the nestest log output is, and I’m not sure how worried I should be about that yet. Most posts that I find on the NesDev forums suggest that being *mostly* correct is good enough at the start, and to just use it as a guide. But it makes me feel all kinds of uncomfortable.

After fixing up a ton of bugs, my output perfectly matches the nestest golden log.

So if I had a piece of advice for budding NES emulator developers: use it as a guide, but it can and should eventually end up being spot-on accurate.

## Tool Making

In almost any project, I’m a big advocate of building specialised tools to make a job easier.

Initially, I had one immediate need: finding a way to compare my emulator’s disassembly output to the nestest log. I didn’t want to fumble around trying to get my output to match perfectly, so I wrote a quick little Perl script - which I named [nes-diff](https://github.com/ltriant/nes/blob/master/tools/nes-diff) - to find the first line that doesn’t match, output the diff, and then exit, because if one thing doesn’t match, any further output is likely to be different aswell.

    $ tools/nes-diff out.txt doc/nestest.log
    230 < C936 ADC A:01 X:00 Y:00 P:6D SP:FB CYC:295
    231 - C938 BMI A:6B X:00 Y:00 P:2C SP:FB CYC:301
    231 + C938 BMI A:6B X:00 Y:00 P:2D SP:FB CYC:301

This output tells me:

* The ADC instruction was executed

* Subsequently, prior to the execution of a BMI instruction, the processor flag was set to the wrong value; the nestest log says the flag should be 0x2C, and my emulator is outputting 0x2D

Funnily enough, one of the most common source of bugs that I had related to status flags. So I wrote another, very primitive tool, [nes-flags](https://github.com/ltriant/nes/blob/master/tools/nes-flags), to show which flags are set for a particular value. I would then know which specific flag to focus my debugging efforts on.

    $ tools/nes-flags 2C 2D
          sv__dizc
    0x2C: 00101100
    0x2D: 00101101

This tells me that the carry flag wasn’t set correctly.

So now I have the two most important pieces of information that I need to fix my CPU bug:

1. Which instruction has the bug (ADC)

1. Roughly where the bug is (the carry flag)

The next tool I’m intending to build is a debugger, which I imagine will be more useful in developing the PPU.

## Introducing Bugs On Purpose

There is a bug in the CPU that every NES emulator must recreate, and if you’re following through the nestest log, you’ll eventually run into it.

If the parameter for an instruction needs to be retrieved from a 16-bit address (which is determined by the addressing mode of the opcode), and the first byte of this 16-bit address is, for example, retrieved from memory at the address 0x10FF, then the second byte should, logically, come from 0x1100, because we’re reading 2 bytes from 0x10FF. But the bug is that the second byte actually comes from 0x1000 because the addition doesn’t carry over into the top byte of the memory address.

[Michael Fogleman](undefined)’s [article](https://medium.com/@fogleman/i-made-an-nes-emulator-here-s-what-i-learned-about-the-original-nintendo-2e078c9b28fe) about his adventures in NES emulation touches on this bug. When I ran into this problem in my CPU, I was fortunate enough that I had this little nugget of information in the back of my mind, so it didn’t take long to discover and then introduce into my emulator.

## Onto the PPU

The first source of documentation was - as it was for the CPU - the NesDev wiki: [PPU programmer reference](https://wiki.nesdev.com/w/index.php/PPU_programmer_reference). I have a printout of this that sits on my desk at the moment.

My first thoughts as I was reading through the docs weren’t pretty. As someone who has pretty much never done any graphical programming (and I’m pretty sure adding a curses UI to an app doesn’t count), the PPU is full of ideas that are completely foreign to me.

At the moment I’m still loosely implementing the PPU registers, and am edging closer and closer to displaying something cool to the screen.

Undoubtedly, the best articles I’ve come across so far to explain how NES graphics work are:

* [NES Graphics — Part 1](http://www.dustmop.io/blog/2015/04/28/nes-graphics-part-1/)

* [NES Graphics — Part 2](http://www.dustmop.io/blog/2015/06/08/nes-graphics-part-2/)

* [NES Graphics — Part 3](http://www.dustmop.io/blog/2015/12/18/nes-graphics-part-3/)

The author, dustmop, does a great job of explaining NES graphics, and provides many helpful images in each article. I only wish I’d found these sooner.

## Rust Suitability

So far, Rust has proven to be an excellent choice for emulator development as a novice in this space.

The module system provides a sensible way to separate the CPU code from the PPU code from everything else, and it seems to make sense with how my brain wants module systems to work.

But the biggest win has been the static type checks and the number of bugs that the compiler errors and warnings have prevented.

Many instructions set the carry flag, and I was doing this by checking if a calculated value was greater than or equal to zero. However, because the type was *u8*, it was guaranteed to always be true, and thus the carry flag would always be set. But luckily, the Rust compiler is super helpful and hints to you that you’re probably doing something wrong:

    warning: comparison is useless due to type limits
       --> src/cpu.rs:510:18
        |
    510 |         self.c = n >= 0;
        |                  ^^^^^^
        |
        = note: #[warn(unused_comparisons)] on by default

Negative offsets used in relative addressing (e.g. a BNE that wants to branch backwards, rather than forwards) presented another casting issue. I ended up with this little nugget, which comes from the function that handles relative addressing and returns the absolute address:

```rust
((cpu.pc as i16) + (offset as i8 as i16)) as u16
```

But this kind of ugly cast is the only one that I’ve got. And it handles what could otherwise have been a tricky situation to debug.

In order to make a somewhat fair comparison, I’m planning to implement a CHIP-8 emulator, but this time I want to use C to compare the experience of using each language for emulator development. And also because I like C.

I’ll post another update when my emulator can, at the very least, render Donkey Kong. Wish me luck!
