---
layout: post
title: "Learning 3D Graphics with OpenGL and Nim"
description: "I wanted to learn OpenGL and 3D graphics programming, so I spent some of 2022 doing that."
date: 2022-11-08 00:00:00
---

Way back in January — while on holiday — I decided that this year I wanted to "learn OpenGL", whatever that meant.

Why? Because I play a lot of video games that do awesome stuff with 3D graphics at high frame rates, and I wanted to know a little more about how that happens. And — based on an extensive survey of one developer — when developers have zero experience in 3D graphics, OpenGL tends to be the piece of technology that they can name.

I didn't know what I wanted to do, and I had no prior 3D graphics experience. I had done very minimal 2D graphics stuff for emulators and very simple visualisations, but those things required no more than drawing rectangles to surfaces at pretty low resolutions, and I was using higher-level frameworks like SDL.

So, on an uncomfortably humid night, while I was pushing my 12 month old daughter around in her pram in a caravan park in an attempt to get her to sleep, I came across [Finding Your Home in Game Graphics Programming](https://alextardif.com/LearningGraphics.html) by Alex Tardif. I took a piece of advice from the article and — a few days after arriving home — I started with [Learn OpenGL](https://learnopengl.com).

Initially, I was writing code in C, but — as I'd been playing with it in the preceding months — I pivoted to Nim pretty quickly as I found myself wanting to move a little faster with really simple non-OpenGL tasks like slurping the contents of a file. It's just a lot less fuss to write this:

```nim
let vertexShader = "main.vs".readFile
```

As opposed to the super verbose way you'd do it in C. I wanted to keep my code pretty simple, and expressive, but also be fast enough to perform all the math required, and Nim strikes a really nice balance here.

Despite there being an [official OpenGL package](https://github.com/nim-lang/opengl), I decided on the [nimgl](https://github.com/nimgl/nimgl) library, which has bindings to the OpenGL _and_ GLFW APIs — which I was going to use immediately — but it also has bindings for Vulkan and Dear ImGui, which I'm curious about to play with later on.

Of the languages that I'm most familiar with, most of the libraries that I briefly checked out didn't have a nice 1:1 mapping to the OpenGL API, usually offering a slightly higher level of abstraction instead, or only supporting an older version of the API. In the future those higher level abstractions will be fine, but I wanted to stay as close to the raw OpenGL API to easily follow the Learn OpenGL lessons first.

Other library wrappers that I used throughout the lessons — [stb\_image](https://github.com/define-private-public/stb_image-Nim), [glm](https://github.com/stavenko/nim-glm), and [assimp](https://github.com/beef331/nimassimp) — are relatively close to the original APIs that they wrap, although that was less important.

As an added bonus, Nim's [uniform function-call syntax](https://en.wikipedia.org/wiki/Uniform_Function_Call_Syntax) made the glm code a little more elegant.

```nim
var model = mat4f(1.0)
  .rotate(radians(45.0), vec3f(0.1, 1.0, 0.2))
  .scale(vec3f(0.25, 0.5, 0.75))
```

With that, I jumped straight in and pretty quickly I had a rotating cube with a functioning camera that I could move around!

![A rotating cube](/assets/3d-rotating-cube.gif){:.post-image}

After getting over the initial hurdles, and gaining a better understanding of the graphics rendering pipeline, shaders, linear algebra, and many of the other basic primitives in this space, most of the other lessons end up being relatively standalone. It becomes more of a choose-your-own-adventure, which I really like.

The most fun part in all of this was getting a point where I could load other peoples' 3D models from the internet. Like this model of a character from a popular series of video games.

![Spyro](/assets/3d-spyro.png){:.post-image}

The situation with the assimp package — a library for importing assets that were created in a 3D modelling application like [Blender](https://www.blender.org) — was a little less elegant than everything else up to this point. [The only assimp package on the Nim package directory](https://nimble.directory/pkg/assimp) points to a repository that hasn't been updated in many years, and targets an old version of Nim. There are several forks, all of which have been updated in slightly different ways, _none_ of which are published on the Nim package directory. There are open PRs on the original repo, but they've been open for 2-3 years, and haven't been addressed. After digging through what's been updated in each fork, I settled on [one of them](https://github.com/beef331/nimassimp), and even ended up [fixing a bug](https://github.com/beef331/nimassimp/pull/1) as the official assimp data structures had been modified slightly since the fork had last been updated.

Yay, open source!

In Nim, declaring a dependency directly from a Github repo is super easy:

```nim
requires "https://github.com/beef331/nimassimp.git >= 0.1.4"
```

It's a little unfortunate, but not the end of the world, especially for a hobby project.

What now? I reeeeally like the idea of a project to simulate certain natural phenomenon, like rain, fog, fire, or smoke. That's something I'd like to start working on next year. Another idea is to create my own rendering library as an abstraction over OpenGL, similar to libraries like [bgfx](https://github.com/bkaradzic/bgfx) and [sokol](https://github.com/floooh/sokol). Maybe I'll do that too? But the point of all this for now was to learn how 3D graphics works, and I did, so mission accomplished!

---

Code is [on Github](https://github.com/ltriant/render3d), with absolutely zero guarantee as to the state of it at any given time.
