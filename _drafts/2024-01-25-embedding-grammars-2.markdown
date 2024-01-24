---
layout: post
title:  "Using GF grammars from Haskell: advanced options"
date:   2024-01-25
categories: gf
tags: gf
---

This is a sequel to the 2019 post on [using GF grammar from an external program](../../../2019/12/12/embedding-grammars.html). In that post, I covered the basics:
- how to compile your grammar into a PGF (Portable Grammar Format)
- how to access the PGF file from a Haskell or Python program
- how to parse a sentence into a tree, and transform that tree into a different tree.

In this post, I will only concentrate on Haskell. If anyone wants a post on more advanced things in Python, let me know and maybe I'll write one in 5 years.


## Lexical functions under single constructor per type

First the simplest thing I didn't explain in the first post.
`--haskell=lexical --lexical=A,N,V,V2,…,Prep,Pron,PN`

## The two Haskell libraries: PGF vs. PGF2

`PGF` is a native Haskell library, and `PGF2` is Haskell bindings to a C library. This section summarizes the main differences between them.

### PGF
**The library called PGF** is:
- Natively written in Haskell—also called "the Haskell runtime"
- [Found on Hackage](https://hackage.haskell.org/package/gf-3.11/docs/PGF.html)  as part of the `gf-3.11` package
- Installed globally on your computer if you get your GF executable via `cabal install gf`. (If you use Stack[^1], you probably don't even want global Haskell packages.)

**Pros**:
- It only depends on Haskell
- It is better documented than the PGF2 library
- It is used in more GF projects, so there are more examples of it around
- Some features are [only implemented in PGF](https://github.com/GrammaticalFramework/gf-core/issues/131), not in PGF2

**Cons**:
- For larger grammars, it is slow
- It doesn't have probabilities
- It can't handle various orthography features, like `"Hello!"`—you need to pre- and postprocess your text into `" hello ! "`
- For Haskell projects using Stack, if you want to use it with any reasonably modern lts (e.g. one that works with Haskell Language Server!), you can't use the latest release gf-3.11, because that clashes with some other dependencies. So to get it working in your project, you need to specify a commit from the GF github and also specific commits of the other projects, [like this](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/4c238cde570bc79d2b05745195314073e5ae245a/stack.yaml#L9-L13).

### PGF2

**The library called PGF2** is:
- Haskell bindings via FFI to a library natively written in C
- [Found on Hackage](https://hackage.haskell.org/package/pgf2-1.3.0/docs/PGF2.html2) as a standalone library, BUT **it needs the C libraries to be installed separately**

**Pros**:
- It is faster than PGF—only becomes noticeable with larger grammars
- It has probabilities, so you can e.g. easily access the n best trees for a given string, _without having to first generate all parse trees_
- It handles [various orthography features](https://aclanthology.org/W15-3305/) like capitalization and spacing, so if you use the `BIND`/`SOFT_BIND` and `CAPIT` tokens in your grammar, you can parse `"Hello!"` and not have to preprocess it into `" hello ! "`
- The Haskell part doesn't depend on the more complex GF package, so including it in your Stack project is straightforward: no need to specify commit hashes for various libraries.

**Cons**:
- You need to install the C library separately
- It is not as well documented as the native PGF library
- It lacks some features that PGF has
- It doesn't have as many examples around


## Installation of C runtime

To use the PGF2 library, you need to install the C runtime by compiling it from source. (The Python bindings manage this with a Python wheel, so in Python there's no need to manually compile any C libraries.)

### Manually install from source
The steps to do this are as follows:

```bash
$ git clone https://github.com/GrammaticalFramework/gf-core.git
$ cd gf-core/src/runtime/c
$ cat INSTALL
# … follow the instructions in that file for your system!
```
Here are the instructions if you want to look at them. [github.com/GrammaticalFramework/gf-core/blob/master/src/runtime/c/INSTALL]
(https://github.com/GrammaticalFramework/gf-core/blob/master/src/runtime/c/INSTALL)

### Docker

I have prepared a [Dockerfile](https://gist.github.com/inariksit/b8545883ff3cb8b6688973c872494707) which you can use to try out the example in my gf-embedded-grammars-tutorial. You can download the file anywhere on your computer, and follow the instructions below:

```bash
$ docker build -t pgf2 .
```

This will take a long time. Once it's done, you should see output like following.

```
 => [10/10] RUN stack build --extra-include-dirs=/usr/local/lib --extra-lib-dirs=/usr/local/lib
 => exporting to image
 => => exporting layers
 => => writing image sha256:81712ef079ace0d888e94a56d64517c65156db625f10bbc873d9432eacab87f9
 => => naming to docker.io/library/pgf2
```

Now you can run the demo in the Docker image:

```bash
$ docker run -it pgf2 bash
root@8472aab978a6:/app/gf-embedded-grammars-tutorial/advanced-pgf2# stack run
```

## PGF2 with Haskell version of abstract syntax

To get the Haskell version of the abstract syntax to conform to PGF2, you need to use the flag `--haskell=pgf2`.

You can read about the differences in [inariksit/gf-embedded-grammars-tutorial/tree/master/advanced-pgf2](https://github.com/inariksit/gf-embedded-grammars-tutorial/tree/master/advanced-pgf2#readme). This is also the example that is used in the Dockerfile above. I won't repeat the information here, so just read the README and inspect the code, which you can run in Docker or without.




## GADTs

The flag `--haskell=gadt` creates a Haskell abstract syntax

### composOp



### composOpMonoid

## Read more

* Bringert and Ranta (2008). [A pattern for almost compositional functions](https://core.ac.uk/download/pdf/70575784.pdf)

* Blog post: [Defeating return type polymorphism](https://philipphagenlocher.de/post/defeating-return-type-polymorphism/) When you work with the GADT abstract syntax and you want return a tree that is transformed in the way you want, but

## Footnotes

[^1]: If you use `stack install` to install GF from source, honestly I'm not really sure what happens—I'm just used to having to specify dependencies per project that I don't even expect to have any global libraries on my computer with stack. I will update this once I know the answer, nobody else among the GF devs uses stack and I have other priorities right now.