---
layout: post
title:  "Using GF grammars from an external program"
date:   2019-12-12
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
- [Installation of GF](#installation-of-gf)
- [Installation of the libraries](#installation-of-the-libraries)
  * [Python](#installation-in-python)
  * [Haskell](#installation-in-haskell)
- [Embedding grammars](#embedding-grammars)
  * [Python](#python)
  * [Haskell](#haskell)
- [Links](#links)

# GF ecosystem

The relevant bits for embedding GF grammars to other programming languages are explained in the following.

<!-- For a fuller picture, including community and such, here's a [presentation](http://www.grammaticalframework.org/~aarne/GF-ecosystem.pdf) that Aarne gave in the [GF summer school 2018](http://school.grammaticalframework.org/2018/).
If you know what GF and PGF are but haven't installed them, skip to [Installation](#installation). -->

## GF: programming language & executable

A GF grammar consists of an abstract syntax and a number of concrete syntaxes. They live in files that end in `.gf`. The executable is called `gf`, and you can use it in two ways:

### 1) Run your grammar in a GF shell

Assuming that you have a file called `LangEng.gf` in the same directory, you can run the following command.

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

A GF file is compiled into a Portable Grammar Format, shortened PGF. If we want to use a GF grammar from another program, most often we need to compile it into PGF first. (Sometimes we can skip the PGF level: see [tutorial](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc159) for compiling the grammar directly to JavaScript.)

PGF is also the name of a [Haskell library](https://hackage.haskell.org/package/gf-3.10/docs/PGF.html), which contains functions for reading and manipulating PGF files.

How about if you don't want to use Haskell? Not a problem, there's [another library called PGF](http://www.grammaticalframework.org/doc/runtime-api.html), written in C. The recommended usage of the C library is through bindings to another programming language. Currently there are 4 options: Python, Java, C# and Haskell again (in which case it is called `PGF2`, to distinguish from the native Haskell library called `PGF`). In this post, we will use Python.

# Installation of GF

If you **haven't installed GF yet**, you most likely want to do it now.
The options are: a) download a binary, b) install from Hackage, and c) compile from source.

## I want to use Python

If you have **Mac or Ubuntu**, the easiest way is to [download the binary](http://www.grammaticalframework.org/download/index.html). Python bindings are included in the binary.

If you don't have Mac or Ubuntu, you can install GF in any way you like---see instructions on the [download page](http://www.grammaticalframework.org/download/index.html)---but it won't include the Python bindings, so you will need to set them up separately.

## I want to use Haskell

For **any system**, the easiest way is to install GF and the libraries from Hackage: type `cabal install gf`.

If you don't (want to) have a system-wide GHC, you have two options:

1. You want to **only work with ready-made PGFs** and never compile GF files yourself. Skip all the way to [Embedding grammars](#embedding-grammars), and run my tutorial using Stack---it downloads the PGF library for you! *The instructions include compiling a GF file into PGF and Haskell file, but I have cheated and put them under version control just so that you can run this tutorial.*
1. You want to **compile GF files into PGF**. Unfortunately GF isn't in Stackage, but you can install GF the executable from source using Stack. Clone the [gf-core repository](https://github.com/GrammaticalFramework/gf-core) and type `stack install`.


# Installation of the libraries

Now follows installation instructions for the `PGF` library in Python and Haskell. **To follow this tutorial, it is enough to choose only one.**

* [Python](#installation-in-python) (further in this post)
* [Haskell](#installation-in-haskell) (further in this post)

## Installation in Python

### 0) Check if it's already installed

<!--Depending on how you installed GF, you might already have the Python bindings. -->
If you downloaded the Mac or Ubuntu binary, then you should have the Python bindings already. (Of course, if you have Mac or Ubuntu but you installed GF in another way, you can still download the binary just for the sake of the libraries and ignore everything else that comes with it!)

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

*Note that for Mac, the library is installed only for the system Python, so if you have installed another Python from e.g. Homebrew and you'd prefer to use that for your GF+Python programming, then you need to go further to step 1.*

*I have no experience on installing the libraries in Windows, so I recommend that you just try to follow these instructions---modify them if needed or use your favourite unix-like compatibility layer. If it doesn't work, we would appreciate if you open an issue at [GF's GitHub](https://github.com/GrammaticalFramework/gf-core/issues) describing your problem.*

______
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

### 3) Test that it works

Now open a Python shell (with the same Python that you used to build+install in the previous step) and type `import pgf`---if it works, now you can skip to [Embedding grammars](#embedding-grammars).

How about if you have followed the steps until here, and it doesn't work? Please open an issue at [GF's GitHub](https://github.com/GrammaticalFramework/gf-core/issues) describing your setup, what steps you took and the output.

## Installation in Haskell

If you want to use Haskell, the first question is which library to use, `PGF` or `PGF2`? Remember, `PGF` is a native Haskell library, and `PGF2` is Haskell bindings to a C library.
<!--For most purposes, they are equally good, and on the scale of small or medium-sized grammars, there is no significant difference in speed. -->
For this post, I chose `PGF` for three reasons:

1. It's installed by default if you get GF from Hackage or compile it from source.
1. The API is better documented than `PGF2`.
1. It works smoothly with the abstract syntax compiled into Haskell.

<!--When would you like to use PGF2 then? If you have a large grammar, PGF2 is faster. Once you're familiar with `PGF`, switching to `PGF2` is just a minor change---a couple of functions have a slightly different type signature and that's it.-->


### 0) Check if it's already installed

If you installed GF from Hackage (typing `cabal install gf`) or compiled it from source, then you should have the PGF library.

Open your `ghci` and type `import PGF`. If it succeeds, you have successfully installed the `PGF` library, and you can skip all the way to [Embedding grammars](#embedding-grammars).

If you have installed GF by other means, or you don't want to have a system-wide GHC, read further.

### 1a) Use Stack

In my repository [gf-embedded-grammars-tutorial](https://github.com/inariksit/gf-embedded-grammars-tutorial), you'll find a Stack file, which downloads all relevant libraries for you in an isolated location.
Clone the repository and skip to [Embedding grammars](#embedding-grammars), where one of your first tasks is to run `stack build`.

### 1b) Non-stack options

Let's see. Are you **sure** you don't want to use Stack?

▶ [I dunno, I've never used it and I'm overwhelmed with all this new information, but I could give it a try](#stack-newbie)

▶ [Yes, I know what Stack is and don't want to use it](#seriously-no-stack-please)


<img src="/images/flowchart.jpg" alt="You didn't install GF from Hackage nor compile from source; you want to use Haskell, and
 don't want to use Stack." />
<em><small>You have ended up in the current branch of this choose-your-adventure, if you followed one of the red routes in this flowchart.</small></em>

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

When you install a program with Stack, it will not affect your previous Haskell ecosystem in any way. The downside is that it will download another version of GHC and libraries, which takes more space, but this is a trade-off for guaranteeing reproducible builds. If you use Stack just once for this project, you can still keep using Cabal only for all other projects in the past and future. So unless disk space is absolutely critical, I recommend this option.

First, [install Stack](https://docs.haskellstack.org/en/stable/README/#how-to-install). This is a simple process involving running one command on your terminal. After that, the rest of the process involves one extra `stack build` and then typing `stack run <program>` instead of `runghc <program>`. If you want to run a ghci with the libraries that are installed locally, you need to write `stack ghci` instead of `ghci`. That's pretty much the concrete noticeable differences that affect your daily life. If you want to learn more, you can read the documentation at [docs.haskellstack.org](https://docs.haskellstack.org/en/stable/README/).

If you decided to give Stack a try, you can skip to [Embedding grammars](#embedding-grammars). Otherwise, read on.

#### Seriously, no Stack please

If you haven't installed GF: GOTO [install GF](#installation) and choose either from Hackage or compile from source.

If your current GF is the downloaded binary, you could do **one of** the following:

* Stop using the binary and install a fresh GF from Hackage or by [compiling from source](https://github.com/GrammaticalFramework/gf-core); **OR**
* Clone the [gf-core repository](https://github.com/GrammaticalFramework/gf-core), comment out all executables from the Cabal file, and `cabal install` only the libraries; **OR**
* Keep using the binary (e.g. if you use the Python bindings!), but create another GF installation from Hackage: `cabal install gf`. This reinstalls the GF executable (hopefully to different place where your binary is) but doesn't include the RGL, so you likely don't need to change your `$GF_LIB_PATH` or any other environment variables you might have.
<!-- If you choose this option, you could as well just stop using the downloaded binary and use the newly cabal-installed GF, then it doesn't matter if Cabal rewrites some path. -->

If something weird happens from having multiple GF installations, or anything else goes wrong, you can open an issue at [GF's GitHub](https://github.com/GrammaticalFramework/gf-core/issues).

# Embedding grammars

From this point on, I assume that you have managed to install the PGF library for Python or Haskell. Again, you can choose to follow the instructions for [Python](#python) or [Haskell](#haskell) further in this post.

## Python

### Preliminaries

* Clone my repository [gf-embedded-grammars-tutorial](https://github.com/inariksit/gf-embedded-grammars-tutorial).

* In the main directory (i.e. called `gf-embedded-grammars-tutorial`), run
 > `gf -make resource/MiniLangEng.gf`

  This creates the PGF file `MiniLang.pgf`.


### Static tutorial

The repository contains a Jupyter notebook named `ReflTransfer.ipynb`. It's meant to be opened with Jupyter, but if you don't have the possibility to install Jupyter on the machine you're reading this, you can still view the notebook on GitHub, where it just looks like a standard non-interactive tutorial. Here's the link: [ReflTransfer.ipynb on GitHub](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/master/ReflTransfer.ipynb).

### Interactive tutorial

If you have a chance to use Jupyter on your own computer, I recommend it: you can modify the code and add new features. If you haven't used Jupyter notebooks before, here's a [tutorial and installation instructions](https://compsci697l.github.io/notes/jupyter-tutorial/).

Once you have installed Jupyter, go to the main directory of my repository (i.e. the one called `embedded-grammars-tutorial`) and run the command `jupyter notebook`.

```
$ jupyter notebook
```
This will open your browser with the following view. Click the file ReflTransfer.ipynb.

<img src="/images/jupyter-notebook-1.png" alt="Picture of Jupyter Notebook server, showing the file ReflTransfer.ipynb and others." />

Now you can use the notebook as an interactive tutorial. You can modify anything in the cells or write new cells and run them.

The rest of this post will be about Haskell, so unless you want to learn how to embed grammars bilingually, you're done now! Here's the last jump in this post, to [links](#links).

## Haskell

### Preliminaries

The first steps are:

* Clone my repository [gf-embedded-grammars-tutorial](https://github.com/inariksit/gf-embedded-grammars-tutorial)

* In the main directory (i.e. called `gf-embedded-grammars-tutorial`), run

  > `gf -make -f haskell resource/MiniLangEng.gf`

  This creates the PGF file `MiniLang.pgf` and the Haskell file `MiniLang.hs`.

* **If you use Stack:** run `stack build`, still in the main directory.

If you are not using Stack, you can ignore both the Stack and the Cabal files in the repository, just `runghc ReflTransfer.hs` will be enough later on.

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

<!--To see a bit more complex program, you can check out the [Simple translator](https://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc147) from the GF tutorial.-->

<!--
### Simple translator

To get started, here's a simple translation application (a bare-bones version of the application presented in the GF [tutorial](https://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc147)). It's also included in my repository with the name `Translator.hs`.

```haskell
import PGF

main :: IO ()
main = do
  gr <- readPGF "MiniLang.pgf" --Open the PGF file
  putStrLn "Write your sentence here to translate it."
  s <- getLine --Get sentence from user when program runs
  putStrLn (translate gr s)
  putStrLn "Thanks for using the great GF translator!"

translate :: PGF -> String -> String
translate gr s = case parseAllLang gr (startCat gr) s of
  (lg,t:_):_
    -> unlines [ linearize gr l t
               | l <- languages gr
               , l /= lg ]
  _ -> "NO PARSE"
```

In the main function, we first read the PGF file `MiniLang.pgf`. (What do you need to do to make it read from command line?) Then we read input from the user, and apply the Haskell function `translate` to that input.
The function `translate` tries to parse the string in all the concrete languages in the grammar. When it finds a language in which the string can be parsed, it then parses the sentence in that language, and linearises the resulting tree in all other languages in the grammar.


-->

### Syntactic transfer

Before we go further into the technologies, let us have a concrete goal to keep it interesting! We want to do **semantics-preserving syntactic transfer**.

I added a function called `ReflV2` into the good old miniresource abstract syntax. The enhanced miniresource is found in the tutorial repository, [MiniGrammar](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/master/resource/MiniGrammar.gf#L42).



<div class="language-haskell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">UseV</span>      <span class="c1">:</span> <span class="c1">V</span>   <span class="c1">-&gt;</span> <span class="c1">VP</span> <span class="c1">;</span>             <span class="c1">-- sleep</span>
<span class="c1">ComplV2</span>   <span class="c1">:</span> <span class="c1">V2</span>  <span class="c1">-&gt;</span> <span class="c1">NP</span> <span class="c1">-&gt;</span> <span class="c1">VP</span> <span class="c1">;</span>       <span class="c1">-- love it</span>
<span class="kt">ReflV2</span>    <span class="o">:</span> <span class="kt">V2</span> <span class="o">-&gt;</span> <span class="kt">VP</span> <span class="o">;</span>              <span class="o">-- use itself</span>
<span class="c1">UseAP</span>     <span class="c1">:</span> <span class="c1">AP</span>  <span class="c1">-&gt;</span> <span class="c1">VP</span> <span class="c1">;</span>             <span class="c1">-- be small</span>
<span class="c1">AdvVP</span>     <span class="c1">:</span> <span class="c1">VP</span> <span class="c1">-&gt;</span> <span class="c1">Adv</span> <span class="c1">-&gt;</span> <span class="c1">VP</span> <span class="c1">;</span>       <span class="c1">-- sleep here</span>
</code></pre></div></div>

<!-- ```haskell
-- Verb
    UseV      : V   -> VP ;             -- sleep
    ComplV2   : V2  -> NP -> VP ;       -- love it
    ReflV2    : V2 -> VP ;              -- use itself
    UseAP     : AP  -> VP ;             -- be small
    AdvVP     : VP -> Adv -> VP ;       -- sleep here
``` -->

And the implementation is in [MiniGrammarEng](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/master/resource/MiniGrammarEng.gf#L56-L65).

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

Now what do we want to do: *transform all sentences with the same subject and object into reflexive*, otherwise leave sentence untouched. Some examples:

* I see me -> I see *myself*
* water drinks water -> water drinks *itself*
* John sees him -> (no change)
* John sleeps -> (no change)

A program that does this modification is our goal. So far we have involved just the PGF library, for parsing and linearising. But the current goal involves more complex manipulation of the trees, and here we are going to introduce another way of interacting with the GF trees.

### GF abstract syntax in Haskell

Remember the flag `-f haskell` when we compiled the GF grammar? It produced a file called `MiniLang.hs`, and now we are going to use that.

#### Why

So first of all, why do we do this? Our overall goal is to manipulate trees, and this is much simpler using pure Haskell datatypes, than using the PGF functions. I'm not even going to bother show how to do it in pure PGF expressions---check out the [Python tutorial](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/master/ReflTransfer.ipynb) if you want to see awkward and type-unsafe programming.

Our goal is to go from PGF expressions to the GF abstract syntax in Haskell, do our transformations operating on the Haskell datatypes, and then go back to the PGF expressions.

#### What & How

Here's (a sample of) how the Haskell module looks like:

```haskell
data GCl = GPredVP GNP GVP

data GNP =
     GDetCN GDet GCN
   | GMassNP GCN
   | GUsePN GPN
   | GUsePron GPron

data GVP =
      GAdvVP GVP GAdv
    | GComplV2 GV2 GNP
    | GReflV2 GV2
    | GUseAP GAP
    | GUseV GV
```

And so on. If you're familiar with the miniresource, you should recognise all these constructors---it's a Haskell translation of the abstract syntax of MiniLang! Where in GF you had `fun PredVP : NP -> VP -> Cl`, in Haskell you have `data GCl = GPredVP GNP GVP`.

In addition, we have a way to relate these Haskell datatypes to the PGF of the same grammar that produced it. Here's a type class `Gf`:

```haskell
class Gf a where
  gf :: a -> Expr
  fg :: Expr -> a
```

The type `Expr` comes from the PGF library. In place of `a`, we will put the Haskell data types just defined, such as `GAdv` or `GCl`.

By making a datatype into an instance of the typeclass `Gf`, we need to provide a translation to and from the PGF datatype `Expr`. (I will skip the details here; you can see them in the generated `MiniLang.hs` file if you are interested.) Thanks to the functions `gf` and `fg`, we can now have a workflow as follows:

1. Parse a sentence into an `Expr`, using the PGF library.
1. Turn the PGF expression into a Haskell expression, using `fg : Expr -> a` for some suitable `a` (e.g. `GCl`).
1. Transform the Haskell expression into a new Haskell expression, using some function that you wrote yourself.
1. Turn the new Haskell expression back into a PGF expression, using `gf : a -> Expr`.
1. Linearise the transformed PGF expression, using the PGF library.

<!--
Let's start from translation *to*:

```haskell
instance Gf GA where
  gf Gbad_A = mkApp (mkCId "bad_A") []
```

The functions `mkApp` and `mkCId` come from the PGF library. `mkCId` constructs an identifier called `bad_A`, and `mkApp` applies  it to an empty list of arguments---because `bad_A` is just a lexical 0-place function, it doesn't have arguments. The full list of Gf instance for GA is as follows:

```haskell
instance Gf GA where
  -- gf :: a -> Expr
  gf Gbad_A = mkApp (mkCId "bad_A") []
  gf Gbig_A = mkApp (mkCId "big_A") []
  ...
  gf Gyoung_A = mkApp (mkCId "young_A") []
```

The translation *from* involves also pattern matching, but from the tree's side.

```haskell
instance Gf GA where
  -- fg :: Expr -> a
  fg t =
    case unApp t of
      Just (i,[]) | i == mkCId "bad_A" -> Gbad_A
      Just (i,[]) | i == mkCId "big_A" -> Gbig_A
      ...
      Just (i,[]) | i == mkCId "young_A" -> Gyoung_A
```

If we translate a data type that has more complex constructors, then  we just call  `fg` and `gf` recursively. See for example `GCl`:

```haskell
instance Gf GCl where
  gf (GPredVP x1 x2) = mkApp (mkCId "PredVP") [gf x1, gf x2]
```

-->


### The program

<!-- In case you forgot in the sea of datatypes, let us recap the goal again!  -->
Now let's get back to the goal! We want to transform sentences with the same subject and object into reflexive, for example *I like me* -> *I like myself*. The first sentence is parsed as follows in the miniresource:

<img src="/images/i-see-me.png" alt="Tree for I see me." height="400" />

I have highlighted the two arguments that are identical, and will trigger the change into reflexive.

The identical argument in question is `UsePron i_Pron`: there are two instances of the tree. Their most recent common ancestor is `PredVP`, which constructs a `Cl`. So we need to design a function that does the following:

1. Pattern match a Cl:
  * Does it contain a ComplV2?
  * Is the ComplV2's NP argument same as PredVP's NP argument?
1. If yes, change the ComplV2 into ReflV2.
1. Return the new tree:

<img src="/images/i-see-myself.png" alt="Tree for I see myself." margin-left="100" height="300" />

At this point, just go and see the actual Haskell program! The full code is found in [ReflTransfer.hs](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/master/ReflTransfer.hs), and the relevant parts are pasted below.

```haskell
transfer :: Tree -> Tree
transfer = gf . toReflexive . fg

-- Wrapper for the more interesting trasfer functions.
-- Need this because Utt is the start category;
-- the strings we input are parsed as Utt by default.
toReflexive :: GUtt -> GUtt
toReflexive (GUttNP x) = GUttNP x -- NPs can't be made reflexive
toReflexive (GUttS s) = GUttS (toReflexiveS s)

-- Another layer of wrapper
toReflexiveS :: GS -> GS
toReflexiveS s = case s of
  GCoordS conj s1 s2 -> GCoordS conj (toReflexiveS s1) (toReflexiveS s2)
  GUsePresCl pol cl -> GUsePresCl pol (toReflexiveCl cl)

-- The relevant transfer function is Cl -> Cl
toReflexiveCl :: GCl -> GCl
toReflexiveCl cl@(GPredVP subj vp) = -- PredVP is the only constructor for Cl in the mini resource
  case vp of
    GComplV2 v2 obj
      -> if show subj == show obj -- GNP has no Eq instance, need to compare string
          then GPredVP subj (GReflV2 v2)
          else cl
    _ -> cl -- Any other way to form VP: keep it unchanged
```

### Run the program

<!--You have certainly compiled your `MiniLangEng.gf` into `MiniLang.pgf` and `MiniLang.hs`. (If those files have rusted away in the time it took to read from the beginning to here, go back to the [preliminaries](#haskell) to see how to regenerate them!)-->

Now you can run the program, alternatively by `runghc ReflTransfer.hs`, or `stack run ReflTransfer`. This is what it should look like:

```
EITHER
  $ runghc ReflTransfer.hs
OR
  $ stack run ReflTransfer
Write your sentence here, I will transform it into reflexive, if it has the same subject and object.
Write quit to exit.
I see me
I see myself
a car
a car
John sleeps and the water drinks the water
John sleeps and the water drinks itself
quit
bye
```

If you are unable to repeat these steps, please let me know! This time it's not a GF core issue, just an issue about my tutorial, so create an issue in [gf-embedded-grammars-tutorial](https://github.com/inariksit/gf-embedded-grammars-tutorial/issues) repository or [email me](mailto:inari.listenmaa@gmail.com).

# Links

* Do you like GF and Python? Wish you could write GF in a more familiar environment? Here's a Jupyter kernel for writing GF grammars and running GF shell in a Jupyter notebook: [github.com/kwarc/gf_kernel](https://github.com/kwarc/gf_kernel)

* This post has concentrated on PGF the library. But maybe you're curious about PGF the file format and the compilation? You can get the gist of it in [my blog](https://inariksit.github.io/gf/2018/06/13/pmcfg.html), and a thorough description in [Krasimir's PhD thesis](http://www.cse.chalmers.se/~krasimir/phd-thesis.pdf).

* At some point, I'm planning to write a follow-up post comparing the two Haskell options: `PGF` (native Haskell library) vs. `PGF2` (Haskell bindings to the C library). When such a post exists, I'll link it here.
