---
layout: post
title:  "Using GF grammars from an external program"
date:   2019-08-01
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

- [GF ecosystem](#gf-ecosystem)
- [Installation](#installation)
- [Embedding grammars](#embedding-grammars)
  * [Python](#python)
  * [Haskell](#haskell)
    + [Reading PGF files in Haskell](#reading-pgf-files-in-haskell)
    + [Manipulating trees](#manipulating-trees)
    + [Example of syntactic transfer](#example-of-syntactic-transfer)
    + [Transfer](#transfer)
- [Links](#links)
- [Footnotes](#footnotes)

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

A GF file is compiled into a Portable Grammar Format[^1], shortened PGF. If we want to use a GF grammar from another program, most often we need to compile it into PGF first. (Sometimes we can skip the PGF level: see [tutorial](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc159) for compiling the grammar directly to JavaScript.)

PGF is also the name of a [Haskell library](https://hackage.haskell.org/package/gf-3.10/docs/PGF.html), which contains functions for reading and manipulating PGF files.

How about if you don't want to use Haskell? Not a problem, there's [another library called PGF](http://www.grammaticalframework.org/doc/runtime-api.html), written in C. The recommended usage of the C library is through bindings to another programming language. Currently there are 4 options: Python, Java, C# and Haskell again (in which case it is called `PGF2`, to distinguish from the native Haskell library called `PGF`). In this post, we will use Python.

# Installation

_______

If you **haven't installed GF yet**, do it now. [The options](http://www.grammaticalframework.org/download/index.html) are: a) download a binary, b) install from Hackage, and c) compile from source.

* If you want to use Python and you have Mac or Ubuntu, the easiest way is to [download the binary](http://www.grammaticalframework.org/download/index.html).
* If you want to use Haskell and have any system at all, the easiest way is to install from Hackage: `cabal install gf`.
  * If you don't (want to) have a system-wide GHC, you can install GF the executable from source using Stack: there is a [stack file](https://github.com/GrammaticalFramework/gf-core/blob/master/stack.yaml) in the [gf-core repository](https://github.com/GrammaticalFramework/gf-core).

______

Now follows installation instructions for the `PGF` library in Python and Haskell. **To follow this tutorial, it is enough to choose only one.**

* [Python](#installation-in-python) (further in this post)
* [Haskell](#installation-in-haskell) (further in this post)

## Installation in Python

### 0) Check if it's already installed

Depending on how you installed GF, you might already have the Python bindings. If you downloaded the Mac or Ubuntu binary, then you should have them. (Of course, if you have Mac or Ubuntu but you installed GF in another way, you can still download the binary just for the sake of the libraries and ignore everything else that comes with it!)

To test if you have the Python bindings, open a Python shell and type `import pgf`:

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

If you can't or don't want to download the binary for Mac or Ubuntu (e.g. not having Mac or Ubuntu are pretty solid reasons!), then you need to install the C runtime and the Python bindings separately.

You need to download the source code at [gf-core](https://github.com/GrammaticalFramework/gf-core). Then, go to the directory `gf-core/src/runtime/c`, where you find [installation instructions](https://github.com/GrammaticalFramework/gf-core/blob/master/src/runtime/c/INSTALL). Follow them to install the C runtime.

### 2) Install the Python bindings

After installing the C runtime at `gf-core/src/runtime/c`, go to `gf-core/src/runtime/python` and run the following commands:

```bash
$ python setup.py build
$ sudo python setup.py install
```

If you have several versions of Python on your computer, make sure that you use the right one when installing. If desired, substitute `python` in the above commands for `python3` or the path to your custom Python binary.

Now open a Python shell (with the same Python that you used to build+install in the previous step) and type `import pgf`---if it works, now you can skip to [Embedding grammars](#embedding-grammars).

## Installation in Haskell

If you want to use Haskell, the first question is which library to use, `PGF` or `PGF2`? Remember, `PGF` is a native Haskell library, and `PGF2` is Haskell bindings to a C library. For most purposes, they are equally good, and on the scale of small or medium-sized grammars, there is no significant difference in speed. For this post, I chose `PGF` for two reasons: 1) it's installed by default if you get GF from Hackage or compile it from source and 2) the API is better documented than `PGF2`.

Once you're familiar with `PGF`, switching to `PGF2` is just a minor change---a couple of functions have a slightly different type signature and that's it.


### 0) Check if it's already installed

If you installed GF from Hackage (typing `cabal install gf`) or compiled it from source, then you should have the PGF library.

Open your `ghci` and type `import PGF`. If it succeeds, you have successfully installed the `PGF` library, and you can skip all the way to [Embedding grammars](#embedding-grammars).

If you have installed GF by other means, or you don't want to have a system-wide GHC, read further.

### 1a) Use Stack

In my repository [embedded-grammars-tutorial](https://github.com/inariksit/gf-embedded-grammars-tutorial), you'll find a [Stack file](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/master/stack.yaml), which downloads all relevant libraries for you in an isolated location.
Clone the repository and skip to [Embedding grammars](#embedding-grammars), where one of your first tasks is to run `stack build`.

### 1b) Non-stack options

You have ended up in the current branch of this choose-your-adventure, if you followed one of the red routes in this flowchart:

<img src="/images/flowchart.jpg" alt="You didn't install GF from Hackage nor compiled from source; you want to use Haskell, and
 don't want to use Stack." />

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

First, [install stack](https://docs.haskellstack.org/en/stable/README/#how-to-install). This is a simple process involving running one command on your terminal. After that, the rest of the process involves one extra `stack build` and then typing `stack run <program>` instead of `runghc <program>`. If you want to run a ghci with the libraries that are installed locally, you need to write `stack ghci` instead of `ghci`

 and then you can skip to [Embedding grammars](#embedding-grammars),

<!-- So just read the docs at [docs.haskellstack.org](https://docs.haskellstack.org/en/stable/README/). -->

#### Seriously, no Stack please

If you haven't installed GF: GOTO [install GF](#installation) and choose either from Hackage or compile from source.

If your current GF is the downloaded binary, you could do one of the following:

* Stop using the binary and install a fresh GF from Hackage or by [compiling from source](https://github.com/GrammaticalFramework/gf-core); **OR**
* Clone the [gf-core repository](https://github.com/GrammaticalFramework/gf-core), comment out all executables from the Cabal file, and `cabal install` only the libraries; **OR**
* Keep using the binary (e.g. if you use the Python bindings!), but create another GF installation from Hackage: `cabal install gf`. This reinstalls the GF executable (hopefully to different place where your binary is) but doesn't include the RGL, so you likely don't need to change your `$GF_LIB_PATH` or any other environment variables you might have.
<!-- If you choose this option, you could as well just stop using the downloaded binary and use the newly cabal-installed GF, then it doesn't matter if Cabal rewrites some path. -->

If something weird happens from having multiple GF installations, you can open an issue at [GF's GitHub](https://github.com/GrammaticalFramework/gf-core/issues).

# Embedding grammars

From this point on, I assume that you have managed to install the PGF library for Python or Haskell. Again, you can choose to follow the instructions for [Python](#python) or [Haskell](#haskell) further in this post.

## Python

### Preliminaries

1. Clone my repository [embedded-grammars-tutorial](https://github.com/inariksit/gf-embedded-grammars-tutorial)
1. In the main directory (i.e. called `embedded-grammars-tutorial`), run `gf -make -f haskell MiniLangEng.gf`. This creates the PGF file `MiniLang.pgf`.


### Static tutorial

The repository contains a Jupyter notebook named `ReflTransfer.ipynb`. It's meant to be opened with Jupyter, but if you don't have the possibility to install Jupyter on the machine you're reading this, you can still view the notebook on GitHub, where it just looks like a standard non-interactive tutorial. Here's the link: [ReflTransfer.ipynb on GitHub](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/master/ReflTransfer.ipynb).

### Interactive tutorial

If you do have a chance to use Jupyter on your own computer, I recommend it: you can modify the code and add new features. If you haven't used Jupyter notebooks before, here's a [tutorial and installation instructions](https://compsci697l.github.io/notes/jupyter-tutorial/).

Once you have installed Jupyter, go to the main directory of my repository (i.e. the one called `embedded-grammars-tutorial`) and run the command `jupyter notebook`.

```
$ jupyter notebook
```
This will open your browser with the following view. Click the file ReflTransfer.ipynb.

<img src="/images/jupyter-notebook-1.png" alt="Picture of Jupyter Notebook server, showing the file ReflTransfer.ipynb and others." />

Now you can use the notebook as an interactive tutorial. You can modify anything in the cells or write new cells and run them.

The rest of this post will be about Haskell, so unless you want to learn how to embed grammars bilingually, you're done now!

## Haskell

### Preliminaries

The first steps are:

1. Clone my repository [embedded-grammars-tutorial](https://github.com/inariksit/gf-embedded-grammars-tutorial)
1. In the main directory (i.e. called `embedded-grammars-tutorial`), run `gf -make -f haskell MiniLangEng.gf`. This creates the PGF file `MiniLang.pgf` and the Haskell file `MiniLang.hs`.
1. **If you use Stack:** run `stack build`, still in the main directory.

If you are not using Stack, you can ignore both the Stack and the Cabal files in the repository, just `runghc Main.hs` will be enough later on.

### PGF API

The PGF library is documented at [Hackage](https://hackage.haskell.org/package/gf-3.10/docs/PGF.html). The standard GF tutorial lists some of [the most important functions](https://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc146), if you want to see fewer things at once. I will explain the functions I use in my code, but once you're familiar with the small examples from the GF tutorial and this tutorial, do browse the full API at Hackage!

### Reading PGF files

Open a Haskell shell (e.g. `ghci` or `stack ghci`) and import the PGF library. Do this in the main directory of my repository, same where you compiled `MiniLangEng.gf` into PGF and Haskell files.

```
$ stack ghci
…
Ok, two modules loaded.
> import PGF
```

Now you can open `MiniLang.pgf` in the shell as follows.

```
PGF> gr <- readPGF "MiniLang.pgf"
PGF> :t gr
gr :: PGF
PGF> languages gr
[MiniLangEng]
PGF> categories gr
[A,AP,Adv,CN,Cl,Conj, … ,VP]

```

In order to parse or linearise, you need a concrete language as well. Here's one way to do it:

```
PGF> let eng = head $ languages gr
PGF> parse gr eng (startCat gr) "I sleep"
[EApp (EFun UttS) (EApp (EApp (EFun UsePresCl) (EFun PPos)) (EApp (EApp (EFun PredVP) (EApp (EFun UsePron) (EFun i_Pron))) (EApp (EFun UseV) (EFun sleep_V))))]
```

If you want to see trees that look like from GF, you need to use `showExpr` from the PGF library, like this:

```
PGF> let trees = parse gr eng (startCat gr) "I sleep"
PGF> map (showExpr []) trees
["UttS (UsePresCl PPos (PredVP (UsePron i_Pron) (UseV sleep_V)))"]
```

The original GF tutorial shows a simple [translation application](https://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc147). The example program in my repository is bigger and does more things, so if you prefer to study a simpler piece of code, that program is a good choice.

You're welcome to read the rest of the original GF tutorial's [chapter 7](https://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc143), if you want. It's not a prerequisite for the rest of my tutorial, but it might just be useful to read about the same concept written by different people and from different perspectives. You can also read it after my tutorial.

### Manipulating trees

Now, I expect you have read the first example in the GF tutorial, about the simple translator. More specifically, I expect that you understand that the Haskell program reads the user's input, gives that input to the GF grammar, and finally prints the output from the GF grammar back to the user.

So far we have involved just the PGF library, for parsing and linearising. The next example involves more linguistic manipulation of trees, and here we are going to introduce another way of interacting with the GF trees.

#### GF abstract syntax in Haskell

Remember the flag `-f haskell` when we compiled the GF grammar? It produced a file called `MiniLang.hs`, and now we are going to use that.




### Example of syntactic transfer

This is a minimal example of semantic-preserving syntactic transfer.

I added a function called `ReflV2` into the old miniresource [abstract syntax](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/master/MiniGrammar.gf#L42):

```haskell
ComplV2   : V2 -> NP -> VP ;  -- love it  ---s
ReflV2    : V2 -> VP ;        -- see itself
```

And the [implementation](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/master/MiniGrammarEng.gf#L56-L65) is in MiniGrammarEng.

```haskell
ReflV2 v2 = {
  verb = verb2gverb v2 ;
  compl = table {
    Agr Sg Per1 => "myself" ;
    Agr Sg Per2 => "yourself" ;
    Agr Sg Per3 => "itself" ; -- simplification, no human referent
    Agr Pl Per1 => "ourselves" ;
    Agr Pl Per2 => "yourselves" ;
    Agr Pl Per3 => "themselves" }
} ;
```

## Transfer

We want to transform sentences with the same subject and object into reflexive. For example:
```
> I like me
I like myself
> John sees John
John sees himself
```

The best way is to export the GF grammar into another format and access it from another program.
In this example, we use Haskell.

The code is found in [Main.hs](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/master/Main.hs).
In order to make it work, you need to generate the files `MiniLang.pgf` and `MiniLang.hs`. Run the following command to generate both:

```
gf -make --output-format=haskell MiniLangEng.gf
```

After generating the files, you can run the program, alternatively by `runghc Main.hs`, or `stack run ReflTransfer`, if you don't have a system-wide GHC.

```
EITHER
  $ runghc Main.hs
OR
  $ stack run ReflTransfer
Write your sentence here, I will transform it into reflexive, if it has the same subject and object.
Write quit to exit.
> I see me
I see myself
> a car
a car
> John sleeps and the water drinks the water
John sleeps and the water drinks itself
> quit
bye
```

## Links

* Python notebook for writing GF grammars and running GF shell

Pitch about https://github.com/kwarc/gf_kernel



## Footnotes

[^1]: Are you curious about the compilation and the low-level format? You can get the gist of it in [my blog](https://inariksit.github.io/gf/2018/06/13/pmcfg.html), and a thorough description in [Krasimir's PhD thesis](http://www.cse.chalmers.se/~krasimir/phd-thesis.pdf).
