---
layout: post
title:  "Working with Clojure: Learning Functional Design"
date:   2019-08-13 00:00:00
description: "Dynamic typing, functional design, transitioning from Perl/Go, and other ramblings."
---
I really wanted to write this much earlier on, but I found that my perspective was changing fairly rapidly as I learned more from week to week, so I decided to wait a few more months to let my knowledge and opinions settle down a bit.

It's been nearly 6 months now, so I figure it's time.

In my last post, I only focused on how I had finally setup my development environment. This time I want to dive into some of the things I've had to learn (or relearn) with respect to writing, designing, and maintaining systems in Clojure, especially with how it relates to my experiences working with Perl and Go for many years.

## Dynamic Typing

Working with dynamically typed languages isn't really a new thing for me after ten years of Perl, but the lack of types really drove me nuts in the beginning; functions were being passed deep maps, and then the structure of the maps was being changed, and everything just kind of silently stopped working. Which functions needed to be changed? All of them? Some of them?

When in this situation, it's no different than working with static typing in that there are several components that all need to be changed, only the compiler can't help.

I've occasionally found myself screaming for some god damned static types. However, more and more, I have days where the dynamism flows like silk.

In Perl I used to battle this with modules like [Moose](https://metacpan.org/pod/Moose)/[Mouse](https://metacpan.org/pod/Mouse)/[Moo](https://metacpan.org/pod/Moo)/[Type::Tiny](https://metacpan.org/pod/Type::Tiny) or tons of manual checks to validate deep structures. These practices were codified in our development team's mantra, so we were able to work around common issues with dynamic typing to produce robust systems.

Personally, robustness is something I want for the systems I write and contribute to, and dynamic typing is often in conflict with that unless certain practices are in place; it's just not something you get by default.

Yes, dynamic typing allows us to change things quickly, but when the system being changed is non-trivial and contains many layers and many modules written by many developers who have long since moved on and who all had different philosophies about software design, I end up spending more time verifying the structure of parameters than I feel like I should.

So how to proceed with this issue? [Spec](https://clojure.org/guides/spec) all the things? Maybe in time I'll discover that's the way to go, but as it is in most cases, I think the answer lays somewhere in the middle.

In codebases that heavily use spec, I add more specs to verify the structures of maps and the types of the parameters and return values. And in other codebases, I sprinkle [pre-condition checks](http://blog.fogus.me/2009/12/21/clojures-pre-and-post/) around the place.

When we do use spec, we don't add specs to every function ever. The way we think of it is "spec on the boundaries, dynamic in the middle". So anything that comes from an external source (e.g. a JSON payload from a web service response, or a JSON payload from a client request, or a message from a Kafka topic), spec is used to validate those structures. And when pulling data out of (or putting data into) a schemaless database, spec is used to validate those structures.

Beyond the checking of types, these mechanisms also serve as documentation for future developers, especially for developers who are new to Clojure (like I am), and new to the codebase aswell.

And lastly, tests are a must for verifying systems built on dynamic typing, but integration tests and system tests matter much more than unit tests do; just because several functions work as perfectly as they were individually designed to, doesn't mean they work together.

The importance of integration and systems tests certainly isn't unique to systems built on dynamic languages and are things I've felt were important for years now, and since starting this job, they've been my best friend while working through new systems.

## Java Interop

Initially I wasn't sure how much I was going to need to interoperate with the Java world, and I didn't think it was going to be much at all, but - so far in my experience - it's been plentiful and unavoidable, for various reasons:

1. There's no Clojure library for the thing you want, but there are plenty of Java options.
2. There's a Clojure library for the thing you want, but it's a wrapper around a Java library, and it's missing functionality you need.
3. There are Clojure and Java libraries for the thing you want, but the best, most mature, and most production-hardened version is the Java library.

In its most basic form, and taken from the [official documentation](https://clojure.org/reference/java_interop), when there's a piece of Java code like this:

```java
Pokemon helioptile = new Pokemon();
heliolisk.setName("Helioptile");
heliolisk.setPrimaryType("Electric");
heliolisk.setSecondaryType("Normal");
```

The `doto` macro allows for a simple translation:

```clojure
(doto (Pokemon.)
  (.setName "Helioptile")
  (.setPrimaryType "Electric")
  (.setSecondaryType "Normal"))
```

But there are also many Java libraries that implement the Builder pattern. So where the Java code may look like this:

```java
Pokemon helioptile = new PokemonBuilder()
    .withName("Helioptile")
    .withPrimaryType("Electric")
    .withSecondaryType("Normal")
    .build();
```

The `->` threading macro gets used a lot.

```clojure
(-> (PokemonBuilder.)
    (.withName "Helioptile")
    (.withPrimaryType "Electric")
    (.withSecondaryType "Normal")
    .build)
```

Or perhaps there are an indeterminate number of things to build into a single object, common with attribute or property objects built from user options that are stored in a database. In that case, `reduce` is the hero of the day, e.g.

Perhaps something like this:

```clojure
(let [opts {"name" "Luke"
            "age"  33
            "too_old_to_play_pokemon" false}]
  (reduce
   (fn [acc [k v]]
     (.withProperty acc k v))
   (Properties.)
   opts))
```

I'm embarassed to say this, but the idea of using `reduce` to take a collection of things and injecting them into a single object is something that seems so obvious now and certainly belongs in a Functional Programming 101 class, but is something that never really clicked, given 99% of the examples for using `reduce` (or `fold`, depending on the language) are for summing a list of numbers.

In the past, I would've just done this with a loop and called `.setProperty` a bunch of times, perhaps like this:

```clojure
(let [opts {"name" "Luke"
            "age"  33
            "too_old_to_play_pokemon" false}
      props (Properties.)]
  (doseq [[k v] opts]
    (.setProperty props k v))
  props)
```

This works perfectly fine, and when there's no equivalent set of `withXXX` methods offered by a Java library, it's a necessity, but when given the choice, I'm drinking the kool-aid and sprinkling as much functional sugar on my code as possible at the moment.

## Living with nil

I've spent enough time in my life dealing with null values to have experienced the very wide variety of pathologies they can be responsible for.

Clojure and other Lisps favors the idea of [nil punning](https://lispcast.com/nil-punning/), and much of Clojure's standard library is built on handling `nil` as an expected first-class value.

This took me a little getting used in the beginning, and is still something I double-check anyway.

Is it perfect? In my opinion, no. It still requires discipline and is something that needs to be kept in the back of one's mind, but it's a novel approach to handling null values that still gets across what the code is trying to do, without having to litter the code with tons of null checks.

The end result has a certain kind of beauty to it.

The `some->` threading macro is `nil`'s best friend. When there are a bunch of functions whose return values flow into the next function, but any one of the functions may return `nil`, the code below may be written by a novice:

```clojure
(when-let [thing (f1 params)]
  (when-let [thing (f2 thing)]
    (when-let [thing (f3 thing)]
       thing)))
```

This is reminiscent of lots of code I've written in Perl:

```perl
my $thing = f1($params);
return unless $thing;

$thing = f2($thing);
return unless $thing;

return f3($thing);
```

But the Clojure code looks off, and this is usually a good sign that there's a better way to do it. And there is. It's more idiomatically written as:

```clojure
(some-> params f1 f2 f3)
```

The `some->` macro works the same as the `->` macro, except that if any of the functions in the chain return `nil`, the rest of the functions are short-circuited and the entire thing returns `nil`. This way the intermediate functions don't necessarily have to handle a `nil` parameter in some way (although they probably should, for the sake of being robust).

This isn't perfectly equivalent to the first solution, since `when-let` tests for [truthiness](http://blog.jayfields.com/2011/02/clojure-truthy-and-falsey.html) (also familiar from the Perl world) and `some->` tests specifically for `nil`, but it's close enough.

## Peeling Back the Layers

The standard library is pretty rich, and there are tons of functions that I'm still yet to touch. For me, I've explored the standard library in "layers", and the more layers that are uncovered with time, the more power that I can wield.

As most languages have these functions, I started out with `map` and `filter` and quickly adopted the `->` and `->>` threading macros along with some iterators like `doseq`, and `loop` and `recur`. These alone will got me pretty far for iterating over collections and composing functions.

The next "layer" included `cond->`, `cond->>`, `some->`, `some->>`, `reduce`, `comp`, and `partial`, and now even more options for composing functions are opened up.

And it was a similar story for working with data structures; it was easy to start with things like `cons`, `conj`, `assoc`, `dissoc`, `merge`, `first`, `rest`, and then slowly grow into `merge-with`, `assoc-in`, `update-in`, and `get-in`.

The "next-layer" functions that are opened up can all be implemented with the "previous-layer" functions and some more verbose code around them, so it wasn't necessary to understand them immediately from day one.

As time goes on, more layers are uncovered.

As with learning any new language and its idioms, reading lots of existing code, getting my code reviewed by more experienced Clojure developers, and also reading reviews of other developers' code has been the key to getting better.

## Transitioning from Perl and Go

When moving from one ecosystem to another, everything is new. Obviously the language and syntax are new, but so are the available libraries, the de facto standard libraries, the testing frameworks, the deployment options, dependency management, the culture and philosophies, the organisations and individuals to look up to, the best books, blogs, and articles to read, and so many other things.

The upside is that because it's a whole new world, there's _so_ much material out there to soak up, and learning new stuff is fun. Occasionally I'll come across a blogger who writes about Clojure or functional programming or other Lisps, and I'll disappear down the rabbit hole for several hours.

I'm still going back and reading through the [Clojure for the Brave and True](https://www.braveclojure.com/) book/website, and I have [The Joy of Clojure](https://www.manning.com/books/the-joy-of-clojure-second-edition) sitting at home.

Probably the hardest part of the transition into Clojure has been at the software design level; I came from a place where I wrote far fewer functions that were larger and with deeper functionality, whereas Clojure favors writing many smaller functions, and that's certainly the norm within my team. I still think in bigger functions, but slowly I'm getting used to breaking them down; code reviews and pair programming help a lot here.

Another consideration is picking how much of a language to use at any point in time. Go is very opinionated in the way it wants you to work with the language and will force structs and interfaces on you whether you like or not, however Perl is a very forgiving language and can be written in many different styles, which is something Clojure sort of seems to encourage aswell.

For example, in the codebases I've been working on, records and protocols are a rare find, however multimethods are common. In other circles, I've also heard of people not fully embracing the functional paradigm and simply using Clojure as a "better Java". And I've read articles about using records and protocols heavily when coming from a Java background.

The last part of the transition that was difficult early on was in testing. Initially, when systems tests were lacking for a piece of code I was working on, I wrote my external system tests in Perl ([Test::Mojo](https://metacpan.org/pod/Test::Mojo) is fantastic for testing web services), just so that I could get some tests working super quickly, and then later on translate them into the appropriate Clojure tests.

---

Despite any issues I've run into, I'm actually super happy to be in the Clojure world at the moment. I've always had a fondness for the Lisp family of languages, and it's refreshing to work with one professionally.
