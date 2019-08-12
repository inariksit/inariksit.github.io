---
layout: post
title:  "Using GF grammars from an external program"
date:   2019-09-01
categories: gf
tags: gf
---

This post will show how to use GF grammars from an external program, and how to manipulate GF trees from that program.
The topic is introduced in [Lesson 7](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc143) of the tutorial, and I will cover parts that I find missing in the tutorial:

* Installation, other practicalities of the GF ecosystem
* Linguistically motivated example of tree manipulation
* Examples in both Haskell and Python (the examples are the same, so only knowing one is enough)

Not all things are *missing* from the tutorial per se, but they are explained in different places. In contrast, I aim to make this post as self-contained as possible. If you have already installed GF and the PGF library in the language of your choice, you can go directly to [Embedding grammars](#embedding-grammars).

It is also possible to embed GF grammars into C#, JavaScript/TypeScript and Java, but I will not cover them in this tutorial. This is enough of a complex choose-your-adventure already.  

# GF ecosystem

The relevant bits for embedding GF grammars to other programming languages are explained in the following.

<!-- For a fuller picture, including community and such, here's a [presentation](http://www.grammaticalframework.org/~aarne/GF-ecosystem.pdf) that Aarne gave in the [GF summer school 2018](http://school.grammaticalframework.org/2018/).
If you know what GF and PGF are but haven't installed them, skip to [Installation](#installation). -->

## GF: programming language & executable

A GF grammar consists of an abstract syntax and a number of concrete syntaxes. They live in files that end in `.gf`. The executable is called `gf`, and you can use it in two ways:

### 1) Run your grammar in a GF shell

```
$ gf LangEng.gf
Lang> p -cat=Cl "I am an apple"
PredVP (UsePron i_Pron) (UseComp (CompCN (UseN apple_N)))
Lang> help
<lots of helpful output>
```

### 2) Compile your grammar into one of the various formats

```
$ gf -make LangEng.gf
linking ... OK
Writing Lang.pgf...
```

If you don't specify anything else than `-make`, then you will get a `.pgf` file. (More on PGF in the next section.) You can get other formats using the flag `-f`:

```
$ gf -make -f haskell LangEng.gf
linking ... OK
Writing Lang.pgf...
Writing Lang.hs...
```

Lang.hs is a Haskell version of the abstract syntax. We'll get back to it in the section about transforming GF trees.

You can find other arguments to `-f` if you run `gf -h`.

## PGF: file format & library

A GF file is compiled into a PGF[^1], short for Portable Grammar Format. If we want to use a GF grammar from another program, most often we need to compile it into PGF first. (Sometimes we don't need PGF: see [tutorial](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc159) for compiling the grammar directly to JavaScript.)

PGF is also the name of a [Haskell library](https://hackage.haskell.org/package/gf-3.10/docs/PGF.html), which contains functions for reading and manipulating PGF files.

How about if you don't want to use Haskell? Not a problem, there's [another library called PGF](http://www.grammaticalframework.org/doc/runtime-api.html), written in C. The recommended usage of the C library is through bindings to another programming language. Currently there are 4 options: Python, Java, C# and Haskell again (in which case it is called `PGF2`, to distinguish from the native Haskell library called `PGF`). In this post, we will use Python.

# Installation

_______

If you **haven't installed GF yet**, do it now. [The options](http://www.grammaticalframework.org/download/index.html) are: a) download a binary, b) install from Hackage, and c) compile from source.

* If you want to use Python and you have Mac or Ubuntu[^2], the easiest way is to [download the binary](http://www.grammaticalframework.org/download/index.html).
* If you want to use Haskell and have any system at all, the easiest way is to install from Hackage: `cabal install gf`.
  * If you don't (want to) have a system-wide GHC, you can install GF the executable from source using Stack: there is a [stack file](https://github.com/GrammaticalFramework/gf-core/blob/master/stack.yaml) in the [gf-core repository](https://github.com/GrammaticalFramework/gf-core).

______

Now follows installation instructions for the `PGF` library in Python and Haskell. **To follow this tutorial, it is enough to choose only one.**

* [Python](#python) (further in this post)
* [Haskell](#haskell) (further in this post)

## Python

### 0) Check if it's already installed

Depending on how you installed GF, you might already have the Python bindings. If you downloaded the Mac or Ubuntu binary, then you should have it.
You can test it by typing `import pgf` in a Python shell:


<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>python
<span class="nv">&lt;information about your python&gt;</span>
Type <span class="s2">"help"</span>, <span class="s2">"copyright"</span>, <span class="s2">"credits"</span> or <span class="s2">"license"</span> for more information.
<span class="o">&gt;&gt;&gt;</span> import pgf
</code></pre></div></div>

<!-- ```bash
$ python
<information about your python>
Type "help", "copyright", "credits" or "license" for more information.
>>> import pgf
``` -->

If the import succeeeds, you have the library, and you can skip all the way to [Embedding grammars](#embedding-grammars).

______

Note that for Mac, the library is installed only for the system Python, so if you have installed another Python from e.g. Homebrew and you'd prefer to use that for your GF+Python programming, then you need to go further to step 1.

I have no experience on installing the libraries in Windows, so I recommend that you just try to follow these instructions---modify them if needed or use your favourite unix-like compatibility layer. If it doesn't work, we would appreciate if you open an issue at [GF's GitHub](https://github.com/GrammaticalFramework/gf-core/issues) describing your problem.
<!-- If it does work, please let me know ([inari.listenmaa@gmail.com](mailto:inari.listenmaa@gmail.com)) so that I can update this post! -->


### 1) Install the C runtime

If your GF comes from source, Hackage or Windows binary, then you need to install the C runtime and the Python bindings separately. (Of course, if you have Mac or Ubuntu, you can also download the binary just for the sake of the libraries and ignore everything else that comes with it.)


You need to download the source at [gf-core](https://github.com/GrammaticalFramework/gf-core) and go to the directory `gf-core/src/runtime/c`, where you find [installation instructions](https://github.com/GrammaticalFramework/gf-core/blob/master/src/runtime/c/INSTALL).


### 2) Install the Python bindings

After installing the C runtime at `gf-core/src/runtime/c`, go to `gf-core/src/runtime/python` and run the following commands:

```bash
$ python setup.py build
$ sudo python setup.py install
```

If you have several versions of Python on your computer, make sure that you use the right one when installing. If desired, substitute `python` in the above commands for `python3` or the path to your custom Python binary.

Now open Python shell (with the same Python that you used to build+install in the previous step) and type `import pgf`---if it works, now you can skip to [Embedding grammars](#embedding-grammars).

## Haskell

If you want to use Haskell, the first question is which library to use, `PGF` or `PGF2`? Remember, `PGF` is a native Haskell library, and `PGF2` is Haskell bindings to a C library. For most purposes, they are equally good, and on the scale of small or medium-sized grammars, there is no significant difference in speed. For this post, I chose `PGF` for two reasons: 1) it's installed by default if you get GF from Hackage or compile it from source and 2) the API is better documented than `PGF2`.

Once you're familiar with `PGF`, switching to `PGF2` is just a minor change---a couple of functions have a slightly different type signature and that's it.


### 0) Check if it's already installed

If you installed GF from Hackage (typing `cabal install gf`) or compiled it from source, then you should have the PGF library.

Open your `ghci` and type `import PGF`. If it succeeds, you have successfully installed the `PGF` library, and you can skip all the way to [Embedding grammars](#embedding-grammars).

If you have installed GF by other means, or you don't want to have a system-wide GHC, read further.

<!--You can either a) use Stack, which creates an isolated environment where the right libraries are installed, or b) install the library globally with your normal GHC.-->

### 1a) Use Stack

In my repository [embedded-grammars-tutorial](TODO), you'll find a [Stack file](TODO).
Clone the repository and do `stack install`, then skip to [Embedding grammars](#embedding-grammars).

### 1b) Non-stack options

You have ended up in the current branch of this choose-your-adventure, if you followed one of the bold routes in this flowchart:

* didn't install GF from Hackage nor compiled from source
* want to use Haskell, and
* don't want to use Stack.

(TODO: draw this in a flowchart)

Let's see. Are you **sure** you don't want to use Stack?

▶ [I dunno, I've never used it and I'm overwhelmed with all this new information, but I could give it a try](#stack-newbie)

▶ [Yes, I know what Stack is and don't want to use it](#seriously-no-stack-please)

#### Stack newbie

Let me quote [docs.haskellstack.org](https://docs.haskellstack.org/en/stable/README/):

> Stack is a cross-platform program for developing Haskell projects. It is aimed at Haskellers both new and experienced.
>
> It features:
>
> * Installing GHC automatically, in an isolated location.
> * Installing packages needed for your project.
> * Building your project.
> * …

When you install a program with Stack, it will not affect your previous Haskell ecosystem in any way. The downside is that it will download another version of GHC and libraries, which takes more space, but this is a trade-off for guaranteeing reproducibile builds. If you use Stack just once for this project, you can still keep using Cabal only for all other projects in the past and future. So unless disk space is absolutely critical, I recommend this option.

First, [install stack](https://docs.haskellstack.org/en/stable/README/#how-to-install) and then `stack install` in the repository I linked. Then you can skip to [Embedding grammars](#embedding-grammars).

<!-- So just read the docs at [docs.haskellstack.org](https://docs.haskellstack.org/en/stable/README/). -->


#### Seriously, no Stack please

If you haven't installed GF: GOTO [install GF](#installation) and choose either from Hackage or compile from source.

If your current GF is the downloaded binary, you could do one of the following:

1. Clone the [gf-core repository](https://github.com/GrammaticalFramework/gf-core), comment out all executables from the Cabal file, and `cabal install` only the libraries.
1. Create another GF installation from Hackage: `cabal install gf`. This reinstalls the GF executable (hopefully to different place where your binary is) but doesn't include the RGL, so you likely don't need to change your `$GF_LIB_PATH` or any other environment variables you might have.
<!-- If you choose this option, you could as well just stop using the downloaded binary and use the newly cabal-installed GF, then it doesn't matter if Cabal rewrites some path. -->

If something weird happens from having multiple GF installations, you can open an issue at [GF's GitHub](https://github.com/GrammaticalFramework/gf-core/issues).

# Embedding grammars

From this point on, I assume that you have managed to install the PGF library for Python or Haskell.

## Reading PGF files

## Manipulating trees


* Translation from GF grammar into Haskell
* `app` and `unApp`

https://github.com/inariksit/gf-contrib/tree/ReflTransfer/mini/newmini


## Links

* Python notebook for writing GF grammars and running GF shell

Pitch about https://github.com/kwarc/gf_kernel

## Author's notes

I tried to write this tutorial over the course of 1000 years (TODO put a correct amount of time when this is finished) and I got so stressed out from the choose-your-own-adventure part that I just couldn't finish the actual part earlier.

<!--Next time I'll blog about something blogger-friendly, like how I spend a significant amount of my paid time imagining different combinations of prepositions, object pronouns, reflexive and impersonal markers in Somali and googling them to see if they exist, and then pasting the bits of text to Google translate and trying to guess if that word is actually what I imagine it to be or if a Somali cat was walking on the keyboard.-->

## Footnotes

[^1]: Are you curious about the compilation and the low-level format? You can get the gist of it in [my blog](https://inariksit.github.io/gf/2018/06/13/pmcfg.html), and a thorough description in [Krasimir's PhD thesis](http://www.cse.chalmers.se/~krasimir/phd-thesis.pdf).
