---
layout: post
title:  "This Weird Trick Makes Your Grammar Use 50GB Less RAM"
date:   2020-11-30
categories: programming
tags: "gf haskell"
---

If you've spent any time with GF, you probably have experienced, or at least heard, that GF can be slow. I have written [almost 3000 words](https://inariksit.github.io/gf/2018/09/22/grammar-blowup.html) of tips how to reduce your grammar size. On my spare time, I'm torturing my friends with monologues about parameters and concrete categories.

![Screenshot showing a series of messages about optimising a GF grammar.](/images/gf-monologue.png "That's just a third of the actual monologue.")

So you could say that the subject is close to my heart. But do you know what had never occurred to me?

**Run a profiler on GF.**

The first challenge is to even consider the possibility. I have been aware of [profiling](http://dev.stephendiehl.com/hask/#rts-profiling) for years, and even profiled some of my own code. But thinking of doing it for other people's code feels like arrogance. _Do I think I'm a better Haskell programmer than those who wrote GF? If there was a low-hanging fruit, wouldn't they have caught it already?_ But how my Haskell skills compare to Aarne et al. is an irrelevant question---I wouldn't be able to implement GF, but I can run a profiler. It's a different task.

<!-- (This is a [just-so](https://en.wikipedia.org/wiki/Just-so_story) explanation.  -->
(This explanation is a later narrative. It's not that I considered profiling GF, but decided against for the fear of being arrogant. The thought just didn't occur to me.)

So who actually had that thought? It was [Anka](https://github.com/anka-213), who hasn't written a line of GF, just heard complaints about GF's slowness.
<!-- enough times to start thinking _I wonder where the problem is_. -->
<!-- The rest of us just accepted that this is the best we can do. -->


## Profiling GF

 <!--. My grammar compiles in 30 seconds and uses a couple of GB RAM. Then I add one function, which -->
<!-- Picture me on a Saturday evening writing GF, cursing and muttering things like "it's only 648 concrete functions, wtf" and "36… 44… 50… 54GB?!?!?" Luckily I live with [a sane person](https://github.com/anka-213), whose reaction to _small number_ + _takes 50 GB memory_ was "have you run a profiler"? -->

<!-- Obviously, I hadn't. So a bit of googling, recompiling GF with a bunch of flags, and we see this graph. These are the Haskell functions that are called when GF is compiling my grammar. -->

The second challenge is to learn how to use a profiler. To minimise that particular hurdle for future me and any potential readers, there's a step-by-step guide at the [end of this post](#step-by-step-guide). Here I just present the results.

The following picture shows the situation on the 28th of November 2020.
These are the Haskell functions that were called when GF was compiling some grammar.

![Graph of profiling GF. The function value2term takes the most resources.](/images/gf-profiling-before.png)

It's clear that `value2term` dominates the graph. Here's how [the code](https://github.com/GrammaticalFramework/gf-core/blob/37c63a0c22ccc73e60222335263c702873b6af2c/src/compiler/GF/Compile/Compute/ConcreteNew.hs#L499-L531) looked like when the graph was produced.

```haskell
value2term :: GLocation -> [Ident] -> Value -> Either Int Term
value2term loc xs v0 =
  case v0 of
    VApp pre vs    -> liftM (foldl App (Q (cPredef,predefName pre))) (mapM v2t vs)
    VCApp f vs     -> liftM  (foldl App (QC f))   (mapM v2t vs)
    VGen j vs      -> liftM2 (foldl App) (var j)  (mapM v2t vs)
    VMeta j env vs -> liftM  (foldl App (Meta j)) (mapM v2t vs)
    VProd bt v x f -> liftM2 (Prod bt x) (v2t v) (v2t' x f)
    VAbs  bt   x f -> liftM  (Abs  bt x)         (v2t' x f)
    ... -- Skipped to the end for brevity
    VError err     -> return (Error err)
    _              -> bug ("value2term "++show loc++" : "++show v0)

```

The third challenge is to interpret the profiling results. I don't know how much I would've gotten out of it alone, but Anka noticed immediately that the code was full of `mapM`, which apparently is known to cause space leaks. (Now I know too!)

After looking around in the code, it was clear that the `Either Int` part was unnecessary. `value2term` was called in 5 places, and in each of them, the potential `Left` value was treated almost identically: raise an error with the message "variable #n is out of scope". So it wouldn't change anything to just throw that error directly in `value2term`, and change the return type just into `Term`. With this change, none of the `liftM` and `mapM` was needed.

TODO: better explanation on the reasons. `Either Int` was strict, after removing it, the conversion from value to term is lazy. As a rule of thumb, it's useful to be lazy, if you're converting between big data types, and you don't necessarily need to read the whole tree. When the function returned an `Either Int Term`, all functions that were using the result, had to wait for the whole computation to finish, to know whether it returned a `Right term` or a `Left int`. With the pure function, any caller can start lazily consuming the result whenever, and how much or little of it as they need.

The results were absolutely dramatic. The grammar that was using >50GB memory, was now using 42MB. Compiling the French resource grammar never went over 700 MB. Much more of the time was actually spent compiling instead of garbage collection.

This was the situation before. Green is activity, orange is garbage collection.

![gfResourcesBefore](/images/gf-graph-before.png "Graph showing the activity during GF compilation, before fixing the memory leak. Bursts of activity between long stretches of garbage collection.")

And here is the situation after fixing the memory leak.

![gfResourcesAfter](/images/gf-graph-after.png "Graph showing the activity during GF compilation, after fixing the memory leak. Short periods of GC in between active work.")

Now if you want to know how to produce such graphs yourself, keep reading! If not, that's the end of the post. GF is now faster. It's not a miracle cure---if your PGF has over 100 000 concrete categories, it may still take minutes to compile (or fail to compile), depending on your computer. But if it succeeds, it will use less memory while doing so!

## Step-by-step guide

TODO: write a coherent story

When the program takes too long time to run, you can always press Ctrl-C to abort early. All the statistics are still collected.

You can also look at the heap profile and the event log while the program is running. But the summary (-s) and the cpu profile (-P) is only available when the program has terminated.


```
+RTS -hc -P -s -l

-hc creates the heap profile gf.hp
-P creates the cpu profile gf.prof
-s shows a summary of usage when the program terminates
-l creates the event log gf.eventlog that can be opened with threadscope

hp2ps -c gf.hp; ps2pdf gf.ps; open gf.pdf

--

$ threadscope gf.eventlog
shows a graph of gc time and cpu usage over time

--
What is ... in
(27653)>>=.\.\/>>=.\/>>=/goB...
?

grep 27653 gf.prof
--


(different bug, adapt to this post)

grep ' no\. ' -A1 gf.prof ;grep 29956 gf.prof
COST CENTRE                                                  MODULE                         SRC                                                                                          no.       entries  %time %alloc   %time %alloc  ticks     bytes

                      inferLType                             GF.Compile.TypeCheck.RConcrete src/compiler/GF/Compile/TypeCheck/RConcrete.hs:(79,1)-(322,56)                               29956    1780925    4.0   11.0    34.5   79.6   1053 953211656
```
