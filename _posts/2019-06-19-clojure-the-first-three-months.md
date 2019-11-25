---
layout: post
title:  "Working with Clojure: The First Three Months"
date:   2019-06-19 00:00:00
description: "I started a new job as a Clojure programmer, so I had to figure out a comfortable way to work with Clojure"
---
For the past ten years, I've been writing code professionally mostly in Perl, but also in Python, PHP, and bits of Java and C.

A few months ago, I started a new job working on distributed systems in Clojure. Having only used Clojure for quick prototypes prior to starting the job, the jump to writing Clojure full time for real, non-trivial projects has been interesting.

## Vim

I started using Vim about 15 years ago at university and I've been using it ever since, and although I knew that the IntelliJ story for Clojure was super popular and productive, and that almost all of the other developers in my team were using IntelliJ (plus one diehard Emacs fan), I still wanted to use Vim with the caveat that if things got too difficult, I would concede defeat and move to IntelliJ.

Over the years, regardless of the language I'm using, I've settled on three main plugins for better navigation and overview:

1. [nerdtree](https://github.com/scrooloose/nerdtree) for project overview
2. [fzf](https://github.com/junegunn/fzf) for fast navigation when I know specifically which file I want to jump to
3. [vim-airline](https://github.com/vim-airline/vim-airline) for a much better status bar, a better tab bar, and [vim-fugitive](https://github.com/tpope/vim-fugitive) integration

These things give me everything I need to navigate a large project; fzf was a big game changer for me. I tried to deprecate my use of nerdtree entirely in place of fzf, but I still find myself using nerdtree in certain circumstances, so I'm happy to keep it for now.

There are a few other plugins that I've been using, specifically for working with Clojure:

1. [vim-fireplace](https://github.com/tpope/vim-fireplace) is the king of the plugins for Clojure, offering REPL integration, documentation lookups, the ability to jump to a specific function (even if it's in another file, deep down in a dependency somewhere), and more.
2. [vim-cider](https://github.com/clojure-vim/vim-cider) offers a few extra functions, including [cljfmt](https://github.com/weavejester/cljfmt) integration and namespace/import cleanup. There were a couple of bugs in this plugin that required a couple of upstream PRs to fix.
3. [vim-surround](https://github.com/tpope/vim-surround) for adding/removing/changing brackets around something. Although not a Clojure-specific plugin, it has obvious benefits for manipulating parentheses and square brackets.
4. [paredit.vim](https://github.com/vim-scripts/paredit.vim) for automatically adding closing parentheses to keep things in balance. This took a bit of time to get used to, and it occasionally causes problems, especially when resolving merge conflicts in vim, but after some time it got out of my way and has been a worthwhile plugin to keep around.

## Color Scheme

Perhaps it's just a symptom of getting older, but after years of using dark backgrounds and dark colorschemes, my eyes finally started to feel tired and strained. So, after reading a handful of articles on the subject, [this article about a brutalist emacs theme](https://asylum.madhouse-project.org/blog/2018/09/06/the-brutalist-path/) being the initial trigger, I decided to try my hand at creating a light colorscheme that uses very obvious highlighting for certain things (like opposing brackets, current line number, etc), bold and underline for note-worthy things (language keywords and types), and color for a few cases (static strings).

I ended up with this (work in progress) colorscheme:

![Slightly easier on the eyes](/assets/vim-colorscheme-clojure.png){:.post-image}

I'm debating whether or not to change the orange to something else, but of the alternative colors that I've tried, I still like the orange.

I specifically targetted Clojure with the colorscheme, but it doesn't look too bad with other languages either.

![I like the type highlighting](/assets/vim-colorscheme-c.png){:.post-image}

---

Even after ten years with Perl, I was still refining my working environment bit by bit, so this is by no means complete, but it's been a great place to start.
