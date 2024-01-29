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

- [Lexical functions under single constructor per type](#lexical-functions-under-single-constructor-per-type)
- [The two Haskell libraries: PGF vs. PGF2](#the-two-haskell-libraries-pgf-vs-pgf2)
- [Installation of C runtime](#installation-of-c-runtime)
- [PGF2 with Haskell version of abstract syntax](#pgf2-with-haskell-version-of-abstract-syntax)
- [GADTs](#gadts)
   * [Transform a subtree within a larger tree: composOp](#transform-a-subtree-within-a-larger-tree-composop)
   * [Extract subtree(s) from a larger tree: composOpMonoid](#extract-subtrees-from-a-larger-tree-composopmonoid)
- [Read more](#read-more)
- [Footnotes](#footnotes)

## Lexical functions under single constructor per type

First the simplest thing I didn't explain in the first post. The following flags are added to the  `gf` command:

```
--haskell=lexical --lexical=[GF categories]
```

A full command could look like this:

```bash
gf -make \
   -f haskell \
   --haskell=lexical \
   --lexical=A,N \
   YourGrammar.gf
```

Suppose that the original grammar has the following abstract syntax functions:

```haskell
fun
  small_A : A ;
  big_A : A ;
  cat_N : N ;
  dog_N : N ;
  -- … and more lexicon of types A and N
  ```

### Without `lexical` flags
The resulting Haskell module, compiled **without** the `lexical` flags, would look as follows. Every lexical function becomes a constructor of the data type:


```haskell
data GA =
   Gsmall_A
 | Gbig_A
 | …
  deriving Show

data GN =
   Gcat_N
 | Gdog_N
 | …
  deriving Show
```


### With `lexical` flags

Now compare with the Haskell module compiled **with** the `lexical` flags:

```haskell
data GA =
   LexA String
  deriving Show

data GN =
   LexN String
  deriving Show
```

Now for each GF category `C` specified in the `--lexical=[C]` here is a single constructor `LexC` that takes a String as its argument.

If you tried to give a category that has no 0-place functions as an argument to `--lexical=`, it would just do nothing. And if a category has both 0-place and ≥1-place functions, then only the 0-place functions are replaced by the `LexC` constructor—see for instance `Adv` in the RGL:

```haskell
data GAdv =
   GPrepNP GPrep GNP -- in the house
 | LexAdv String     -- tomorrow
  deriving Show
```

If you are working on the full RGL, the list of categories with 0-place functions can be quite large. I usually grep my whole computer for `--haskell=lexical --lexical=` and copy whatever list I find. Here's one I just found:

```
N,N2,N3,A,A2,A3,V,V2,V3,VA,V2V,VV,VS,V2A,V2S,V2Q,Adv,AdV,AdA,AdN,ACard,CAdv,Conj,Interj,PN,Prep,Pron,Quant,Det,Card,Predet,Subj
```

## The two Haskell libraries: PGF vs. PGF2

`PGF` is a native Haskell library, and `PGF2` is Haskell bindings to a C library. This section summarizes the main differences between them.

### PGF
**The library called PGF** is:
- Natively written in Haskell—also called "the Haskell runtime"
- [Found on Hackage](https://hackage.haskell.org/package/gf-3.11/docs/PGF.html)  as part of the `gf-3.11` package
- Installed globally on your computer if you get your GF executable via `cabal install gf`.
  - If you use `stack install` to install GF from source, you will get the executable `gf` globally on your computer, but not the library. So with stack, you still need to specify the `gf` package in every project in both package.yaml and stack.yaml (because gf is not in Stackage).

**Pros**:
- Only depends on Haskell
- Better documented than the PGF2 library
- Used in more GF projects, so there are more examples of it around
- Some features are [only implemented in PGF](https://github.com/GrammaticalFramework/gf-core/issues/131), not in PGF2

**Cons**:
- Slow for larger grammars
- Doesn't have probabilities
- Can't handle various orthography features: expect to pre- and postprocess `"Hello!"` into `" hello ! "`
- For Haskell projects using Stack, if you want to use it with any reasonably modern lts (e.g. one that works with Haskell Language Server!), you can't use the latest release gf-3.11, because that clashes with some other dependencies. So to get it working in your project, you need to specify a commit from the GF github and also specific commits of the other libraries, [like this](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/4c238cde570bc79d2b05745195314073e5ae245a/stack.yaml#L9-L13).
<!-- - Krasimir keeps saying it's "deprecated" but I really don't understand what that means—people are using it all the time. -->

### PGF2

**The library called PGF2** is:
- Haskell bindings via FFI to a library natively written in C
- [Found on Hackage](https://hackage.haskell.org/package/pgf2-1.3.0/docs/PGF2.html2) as a standalone library, __but it needs the C libraries to be installed separately__

**Pros**:
- Faster for large grammars
- Has probabilities, so you can e.g. easily generate the n best trees for a given string and stop there
  - Contrast with the pure Haskell version, where you need to first generate all the trees and then write your own ranking function to get the n best trees. This in combination with pure-Haskell-PGF being slower, the PGF2 library is a significant improvement with large and ambiguous grammars.
- Handles [various orthography features](https://aclanthology.org/W15-3305/) like capitalization and spacing, so if you use the `BIND`/`SOFT_BIND` and `CAPIT` tokens in your grammar, you can parse `"Hello!"` and not have to preprocess it into `" hello ! "`

**Cons**:
- You need to install the C library separately—see next section for instructions
- Not as well documented as the native PGF library
- Lacks some features that PGF has
- Less used over time, so doesn't have as many examples around


## Installation of C runtime

The PGF2 library in Hackage doesn't come with the C library, so you need to install the C library yourself. (The Python bindings manage this with a Python wheel, so in Python there's no need to manually compile any C libraries.)

The C library is installed in the location `/usr/local/lib`, and that path may need to be specified in a few different places. I have put the following in the [stack.yaml file](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/7b7c1b98b55b24fac8030c21e5fe82a66816b2e5/advanced-pgf2/stack.yaml#L9C1-L19C45) in my repository:

```yaml
extra-lib-dirs:
- /usr/local/lib
ghc-options:
  "$locals": -optl=-Wl,-rpath,/usr/local/lib
```

Depending on your system, you might need 0-2 of these options in your stack.yaml.
<!-- I found that last line in [Roman Cheplyaka's blog](https://ro-che.info/articles/2020-04-07-haskell-local-c-library), when I was looking for alternatives to setting a `LD_LIB_PATH`. If you try this and it doesn't work on your system, please let me know and I can update this tutorial. -->

### Manually install from source
The steps to do this are as follows:

```bash
$ git clone https://github.com/GrammaticalFramework/gf-core.git
$ cd gf-core/src/runtime/c
$ cat INSTALL
# … follow the instructions in that file for your system!
```
Here are the instructions if you want to look at them: [github.com/GrammaticalFramework/gf-core/blob/master/src/runtime/c/INSTALL](https://github.com/GrammaticalFramework/gf-core/blob/master/src/runtime/c/INSTALL)

### Alternative: use Docker

I have prepared a [Dockerfile](https://gist.github.com/inariksit/b8545883ff3cb8b6688973c872494707) which you can use to try out the example in my [tutorial on GitHub](https://github.com/inariksit/gf-embedded-grammars-tutorial/). You can download the file anywhere on your computer, and follow the instructions below:

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

```haskell
$ docker run -it pgf2 bash
root@8472aab978a6:/app/gf-embedded-grammars-tutorial/advanced-pgf2# stack run
```

## PGF2 with Haskell version of abstract syntax

To get the Haskell version of the abstract syntax to conform to PGF2, you need to use the flag `--haskell=pgf2`. A full example would look like this:

```bash
gf -make \
   -f haskell \
   --haskell=pgf2 \ # As many --haskell=… flags as you want
   --haskell=lexical --lexical=A,N,V,… \
   YourGrammar.gf
```

You can read about the differences in [inariksit/gf-embedded-grammars-tutorial/tree/master/advanced-pgf2](https://github.com/inariksit/gf-embedded-grammars-tutorial/tree/master/advanced-pgf2#readme). This is also the example that is used in the Dockerfile above. I won't repeat the information here, so just read the README and inspect the code, which you can run in Docker or without.


## GADTs

The next flag that can be added to the usual command is `--haskell=gadt`. An example of a full command looks like this:

```bash
gf -make \
   -f haskell \
   --haskell=gadt \
   --haskell=lexical --lexical=A,Adv,N,V,… \
   YourGrammar.gf
```

Adding the flag `--haskell=gadt` creates a Haskell module where the full GF abstract syntax is represented under a single Haskell data type—a [GADT](https://en.wikibooks.org/wiki/Haskell/GADT). [Earlier in this post](#without-lexical-flags), we saw how the GF types `A` and `N` were converted into two different Haskell types. Now everything is part of the same type:

```haskell
data Tree :: * -> * where
  LexA    :: String -> Tree GA_
  GPositA :: GA -> Tree GAP_
  GPrepNP :: GPrep -> GNP -> Tree GAdv_
  LexAdv  :: String -> Tree GAdv_
  …
  GUseV   :: GV -> Tree GVP_
  GString :: String -> Tree GString_
  GInt    :: Int -> Tree GInt_
  GFloat  :: Double -> Tree GFloat_
```

The constructors of that data type can take any number of arguments, corresponding to the original GF function. As before, if the GF category  `C` is specified as an argument to the `--lexical=` flag, then its constructor in the GADT is called `LexC` and it takes a string. Otherwise, the constructors take other GF categories as arguments.[^1]

This design gives some major benefits in doing tree transformations. If you read [the previous post](../../../2019/12/12/embedding-grammars.html#the-program), you see these layers of wrappers before we get to modify the actual subtree that we want to modify;

```haskell
-- first layer of wrapper
toReflexive :: GUtt -> GUtt
toReflexive (GUttS s) = GUttS (toReflexiveS s)
-- … other cases leave the Utt intact

-- second layer of wrapper
toReflexiveS :: GS -> GS
toReflexiveS (GUsePresCl pol cl) = GUsePresCl pol (toReflexiveCl cl)
-- …

-- The relevant transfer function is Cl -> Cl
toReflexiveCl :: GCl -> GCl
-- here happens the actual transformation
```

The further inside the start category we want to modify, the more awkward wrapper functions we need to write. Now contrast the previous function to the [GADT version](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/master/advanced-gadts/ReflTransferWithGADTs.hs#L14-L24):

```haskell
-- Transform a subtree, keep rest of the tree intact
toReflexive :: forall a . Tree a -> Tree a
toReflexive tree = case tree of
  -- If argument tree matches, do the transformation
  GPredVP subj (GComplV2 v2 obj) ->
    if isSame subj obj
      then GPredVP subj (GReflV2 v2)
      else tree

  -- If argument tree doesn't match, apply toReflexive to all subtrees
  _ -> composOp toReflexive tree
```

Here we only needed to write a single function, with type signature `Tree a -> Tree a`, and only write code for the actual case we want to transform. Then that same function is applied to all the subtrees with `composOp`.

### Transform a subtree within a larger tree: composOp

The function `composOp` is included in the Haskell module generated with `--haskell=gadt`. You can inspect the code [here](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/d805849610f0e5543d83226d3968ad56d48f8552/advanced-gadts/MiniLang.hs#L373-L395), but if that doesn't say much to you, don't worry. There are simple patterns that you can adopt in your GF grammar without understanding the internals.

`composOp` is used for patterns where you want to modify an existing tree. So the pattern goes like this:

```haskell
transformTree :: forall a . Tree a -> Tree a
transformTree tree = case tree of
  -- If argument tree matches, transform it
  SimpleSubtree -> AnotherSimpleSubtree
  ComplexSubtree a1 a2 -> ComplexSubtree a1 ConstantArg2
  Foo foo -> Bar Baz (computeResultFrom foo)
  …

  -- Otherwise, try to apply transformTree to all subtrees
  _ -> composOp transformTree tree
```

And this works exactly because both the larger tree and its subtrees are *of the same type*: `Tree a`.

Note that you do need to transform the tree into the same type: if `SimpleSubtree` and `AnotherSimpleSubtree` are of different types, you will get an error. But `SimpleSubtree`, `ComplexSubtree … …` and `Foo …` can all be different types.

### Extract subtree(s) from a larger tree: composOpMonoid

Another useful pattern is to extract something from a tree. This is the [example from my tutorial](https://github.com/inariksit/gf-embedded-grammars-tutorial/blob/d805849610f0e5543d83226d3968ad56d48f8552/advanced-gadts/ReflTransferWithGADTs.hs#L31-L43):

```haskell
-- If argument is a lexical function, return the String in a list
-- If argument doesn't match, apply getLex to all subtrees
getLex :: forall a . Tree a -> [String]
getLex tree = case tree of
  LexA    s -> [s]
  LexDet  s -> [s]
  LexN    s -> [s]
  LexPN   s -> [s]
  LexPrep s -> [s]
  LexPron s -> [s]
  LexV    s -> [s]
  LexV2   s -> [s]
  x -> composOpMonoid getLex x
```

Let's take an example tree
```haskell
-- the small cat sees a big dog
PredVP
    ( DetCN the_Det
        (  AdjCN (  PositA small_A  ) (  UseN cat_N  )  )
    )
    ( ComplV2 see_V2
        ( DetCN a_Det
            (  AdjCN (  PositA big_A  ) (  UseN dog_N  )  )
        )
    )
```

Applying `getLex` on that tree gives us a list of all the lexical functions as Strings:

```haskell
[the_Det,  small_A, cat_N, see_V2, a_Det, big_A, dog_N]
```

The return type doesn't have to be a list, it can be any monoid. Haskell just has to know how to `<>` together the two values, since the function is applied to the whole tree, and there are potentially multiple subtrees that match the extraction condition.

## Read more

* Bringert and Ranta (2008). [A pattern for almost compositional functions](https://core.ac.uk/download/pdf/70575784.pdf) This was the original paper that inspired the GADT and `compos*` design. There is also  `composOpFold` and more, but I have never found use for them in my day-to-day GF tree transformation needs.


* Blog post by some random person on the internet: [Defeating return type polymorphism](https://philipphagenlocher.de/post/defeating-return-type-polymorphism/) When you work with the GADT abstract syntax and you would like to have `composOpMonoid` return a potentially heterogeneous list, you can take inspiration from this post and make your own newtype wrapper.

## Footnotes

[^1]: The types like `GAdv` and `GAdv_` are just dummy types generated automatically earlier in the Haskell module. You may use them as convenience in your own functions, but under the hood, everything is just a single datatype `Tree a`. For instance, the type signature `GV -> Tree GVP_` just desugars into `Tree GV_ -> Tree GVP_`.
