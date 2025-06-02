---
layout: post
title:  "Programming in GF: tips and gotchas"
date:   2018-08-28
categories: gf
tags: gf
---

![gf_rubberduck](/images/gf-rubber-duck.png "Your favourite companion for writing GF")
Latest update: 2023-03-26

This post contains real-life examples when I or others have been
confused in the past. It might be updated whenever someone is confused
again.

- [Unsupported token gluing](#unsupported-token-gluing)
- [Restricted inheritance](#restricted-inheritance)
- [linref](#linref)
- [What's the problem with variants in resource grammar?](#whats-the-problem-with-variants-in-resource-grammar)
- [A bit about types](#a-bit-about-types)
- [Some overlooked GF shell commands](#some-overlooked-gf-shell-commands)
  - [ai](#ai)
  - [ma](#ma)
  - [ca](#ca)
  - [tt](#tt)
  - [The coolest flags of `l`](#the-coolest-flags-of-l)
- [Generating MissingXxx](#generating-missingxxx)
- [Metavariables, or those question marks that appear when parsing](#metavariables-or-those-question-marks-that-appear-when-parsing)
- [Placeholders/empty linearisations](#placeholdersempty-linearisations)
  - [Linearise an empty string](#linearise-an-empty-string-to-mark-a-nonexistent-option)
  - [Linearise some other form that exists](#linearise-some-other-form-that-exists)
  - [Linearise a theoretical form](#linearise-theoretical-forms-and-let-the-application-grammarian-to-deal-with-them)
  - [Raise an exception](#raise-an-exception)
  - [variants {}](#empty-variants)
- [Wrong claim that some function is unimplemented](#wrong-claim-that-some-function-is-unimplemented)
- [Mysterious empty strings even though lin exists](#mysterious-empty-strings-even-though-lin-exists)
- [Too good linearisations for some RGL functions](#too-good-linearisations-for-some-rgl-functions)
- [Re-export RGL opers in application grammar](#re-export-rgl-opers-in-application-grammar)
- [Naming conventions](#naming-conventions)
- [Cute way to count syllables](#cute-way-to-count-syllables)

## Unsupported token gluing

In my experience from teaching GF, the most common source of confusion is what you're allowed to do with strings. Why is it sometimes okay to use `+`, and other times you get an error about "unsupported token gluing"?

The short answer is [in the GF tutorial](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc67). I've written this longer answer, because I've found that it's often needed.

### Restricted string operations


* Gluing strings with `+`

  ```haskell
  oper
    addBar : Str -> Str ;
    addBar x = x + "bar" ;

   -- addBar "foo" returns "foobar"
  ```

* Pattern matching strings
<!-- As expected, `isPlural "car"` returns `False`, and `isPlural "cars"` returns `True`. -->

  ```haskell
  oper
    isPlural : Str -> Bool ;
    isPlural str = case str of {
      _ + "s" => True ;
      _       => False } ;
  ```

You can use these operations when constructing lexicon, but not in a function that takes arguments.

### Tokens need to be known at compile-time

To quote the [GF tutorial for Python programmers](https://daherb.github.io/GF-for-Python-programmers/Tutorial.html#compile-time-tokens-vs-run-time-strings):

<!-- > This is as good a time as any to point out the difference between the operators `+` and `++`. This is a common source of problems. GF scrupulously observes the difference between compile-time string operations and run-time string operations. But this distinction is more obvious to GF than it is to you, and can be the source of mysterious errors. -->
<!-- > -->
> GF requires that every token – every separate word – be known at compile-time. Rearranging known tokens in new ways, no problem: GF can generate an infinite variety of different combinations of words.
>
> But they have to be words known to GF at compile-time. GF is not improv: as Shakespeare might have said, if anybody’s going to make up new words around here, it’ll be the playwright, not the actor. You can `+` tokens together but only at compile-time. If you try to do it at run-time, you will get weird errors, like `unsupported token gluing` or, worse, `Internal error in GeneratePMCFG`.

So how do you know whether a line of code is executing at compile-time or at run-time?

### Run-time string operations

Look at the functions' type signatures. If the `fun` takes an argument, that's a run-time argument.

```haskell
fun
  Pred : Item -> Quality -> Comment ; -- 2 args: Item and Quality
  Mod  : Quality -> Kind -> Kind ;    -- 2 args: Quality and Kind
  Very : Quality -> Quality ;         -- 1 arg: Quality
```

When you write the grammar, you tell a function like `Pred` *how* to do its thing. When you run the grammar in a GF shell and tell it to linearise `Pred (This Pizza) Italian`, then it will *actually do* its thing, with the actual arguments `This Pizza` and `Italian`. Hence, run-time arguments.


<!-- All of these functions take arguments. `Pred` and `Mod` take two arguments, `Very` takes one. -->
In the linearisation of these functions, you are working on tokens that come from the arguments. You may concatenate them with `++`, introduce new tokens (like "very" in the example below), even duplicate or remove some of the arguments. But you may not use `+` or pattern match.

This is correct.

```haskell
lin
  Very qual = {s = "very" ++ qual.s} ;
```

This is wrong.

<div class="language-haskell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">lin</span>
  <span class="kt">Very</span> <span class="n">qual</span> <span class="o">=</span> <span class="p">{</span><span class="n">s</span> <span class="o">=</span> <span class="s">"very"</span> <span class="err">+</span> <span class="n">qual</span><span class="o">.</span><span class="n">s</span><span class="p">}</span> <span class="p">;</span>
</code></pre></div></div>

This is also wrong. If you want to prevent multiple "very", it's better to manipulate the GF trees [outside the GF grammar](../../../2019/12/12/embedding-grammars.html).

<div class="language-haskell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">lin</span>
  <span class="kt">Very</span> <span class="n">qual</span> <span class="o">=</span> <span class="p">{</span>
    <span class="n">s</span> <span class="o">=</span> <span class="err">case qual.s of {</span> <span class="c1">-- don't add a second very</span>
         <span class="err">"very" + x =&gt; qual.s ;</span>
         <span class="err">_ =&gt; "very" + qual.s }</span>
    <span class="p">}</span> <span class="p">;</span>
</code></pre></div></div>


<!-- ```haskell -->
<!-- lin -->
<!--   Very qual = { -->
<!--     s = case qual.s of { -- don't add a second very -->
<!--          "very" + x => qual.s ; -->
<!--          _          => "very" + qual.s} -->
<!--     } ; -->
<!-- ``` -->


### Compile-time string operations

In contrast, these functions take no arguments.

```haskell
fun
  Pizza : Kind ;
  Vegan : Quality ;
```

How do you know? There's no arrow in the type. The types are just `Kind` and `Quality`, as opposed to something like `Kind -> Quality -> Kind`.

This means that in the linearisation of `Pizza`, you can use as many single `+`s and pattern matches as you want.
Doesn't matter if they're directly in the `lin`, or via `oper`s that are called in the `lin`.

```haskell
lin
  Pizza = mkN "pizza" ; -- mkN calls addS, which
                        -- uses plus and pattern match
oper
  mkN : Str -> Str -> {s : Number => Str} ;
  mkN str = {s = table {Sg => str ; Pl => addS str}} ;

  addS : Str -> Str ;
  addS str = case str of {
    bab + "y" => bab + "ies" ; -- y -> ie, e.g. babies
    _         => str + "s" ; -- regular, e.g. dogs
    } ;
```

The oper `mkN` takes the string `"pizza"` as an argument, but that's no problem. The string `"pizza"` is not a _run-time argument_ to the fun `Pizza`.

That line, `lin Pizza = mkN "pizza"`, really means this:

```haskell
lin
  Pizza = {s = table {Sg => "pizza" ; Pl => "pizzas"}} ;
```

So you can ignore all the opers, the only meaningful question is whether the fun has arguments. `Pizza` has none---we're only constructing what will become a run-time argument for other GF funs.

<!-- ### Pattern matching `param`s is always allowed -->

<!-- All these restrictions only apply to strings. You may pattern match `param`s in any context, no restrictions. -->


### How to debug

If you get the error for unsupported token gluing, even though you're certain that you don't have any lins with `+`, check two things.

1. Are you sure you are not using a chain of opers calling opers calling opers, where some oper sneakily uses `+` or pattern match?
2. Do you have nonexhaustive pattern in some *legal* pattern match? There is a [known bug in the GF compiler](https://github.com/GrammaticalFramework/gf-core/issues/52) that blames unsupported token gluing, when the actual failure is due to nonexhaustive patterns.

____

## Restricted inheritance

Restricted inheritance is explained with the gentle example of
`abstract Foodmarket = Food, Fruit [Peach], Mushroom - [Agaric]` in
the
[tutorial](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc102).

However, when you have a large grammar (like the resource grammar),
you need to exclude abstract syntax functions or categories in *all* modules that
extend the original module. For instance,
[Common.gf](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/abstract/Common.gf)
is used in
[Cat](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/abstract/Cat.gf#L21),
[Tense](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/abstract/Tense.gf#L11)
and [Text](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/abstract/Text.gf#L7).

Sometimes you need to write your own definition of some category
defined in `Common`, like
[here in CatEng](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/english/CatEng.gf#L1):

```haskell
concrete CatEng of Cat = CommonX - [Pol,SC,CAdv] **
  open ResEng, Prelude in { ... }
```

Then in addition, you need to exclude `Pol`, `SC` and `CAdv` in
[GrammarEng](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/english/GrammarEng.gf#L14-L17):

```haskell
concrete GrammarEng of Grammar =
  NounEng,
  ...,
  PhraseEng,
  TextX - [Pol,SC,CAdv],
  StructuralEng,
  IdiomEng,
  TenseX - [Pol,SC,CAdv]
** open ResEng, Prelude in { ... }
```

If you don't do this, and only exclude the categories in `CatEng`, you
get the following error of all the categories:

```
GrammarEng.gf:
   cannot unify the information
       lincat CAdv = {s : Str; p : Str} ;
       lindef CAdv = \str_0 -> {s = str_0; p = str_0} ;
       linref CAdv = \str_0 -> str_0.s ;
   in module CommonX with
       lincat CAdv = {s : ParamX.Polarity => Str; p : Str} ;
       lindef CAdv = \str_0 -> {s = table ParamX.Polarity {
                                      _ => str_0
                                    };
                                p = str_0} ;
       linref CAdv = \str_0 -> str_0.s ! ParamX.Pos ;
   in module CatEng
```

### Restricted inheritance and the C runtime

An anonymous GF developer had a problem with the Czech concrete syntax of an application grammar. The grammar worked fine in the ordinary GF shell, but it crashed when using the C runtime.

> I am experiencing something strange with the Czech grammar. When compiling to PGF and loading with the C runtime, I get:
> ```
> Abstract category A is missing
> pgf_read_cnccat (pgf/reader.c:1067): assertion failed
>   cnccat->abscat != NULL
> ```
>
> This doesn't occur when loading with the Haskell runtime.
> I'm not doing anything special, just  `gf -make AppCze.gf && gf -cshell App.pgf`

The problem turned out to be about the (non)restricted inheritance in the Czech resource grammar. Here's the first (content) line of [Numeral.gf](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/abstract/Numeral.gf#L20):

```haskell
abstract Numeral = Cat [Numeral,Digits] ** {
```

And here's the corresponding line in [NumeralCze.gf](https://github.com/GrammaticalFramework/gf-rgl/blob/0d6948f59b25b9f0d67244172ba85570ffd6405d/src/czech/NumeralCze.gf#L1-L3) at the time before it had been fixed:

```haskell
concrete NumeralCze of Numeral =

  CatCze ** {
```

NumeralCze was inheriting all of CatCze and not just `[Numeral,Digits]`. After restricting the inheritance, the problem was solved. The current [NumeralCze.gf](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/czech/NumeralCze.gf#L1-L3) looks like this, and the application works correctly.

```haskell
concrete NumeralCze of Numeral =

  CatCze [Numeral,Digits] ** {
```
___

## linref

Does this ever happen to you?

```
Lang> gr -cat=VP | l -treebank
Lang: UseComp (CompAP (UseComparA probable_AS))
LangDut: zijn
LangEng: be more probable
```

Why is there only *zijn* 'to be' in the Dutch and the whole VP in
English? Well, that's because the Dutch grammar has no `linref` for
`VP`.

`linref` is defined in the
[reference manual](http://www.grammaticalframework.org/doc/gf-refman.html#linearization-reference-definitions-linref)
as the "reference linearization of category C, i.e. the function
applicable to an object of the linearization type of C to make it into
a string".

So if your `VP` is a record with a bunch of fields,
and you don't have a linref, then the GF shell shows only the thing
in the `s` field. If you have a linref, then the function is applied
to the record, and GF shell shows a string that makes sense, like *be
more probable*.

If you want to play around with linrefs, I have an [example grammar](https://gist.github.com/inariksit/83cec5f55ced6bece35a249b9ad73c9e) where you can comment out, uncomment and modify the linref and see what happens.

Here are some
[English linrefs](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/english/CatEng.gf#L129-L149)
 in the RGL, if you want to see other than toy examples.

___

## What's the problem with variants in resource grammar?

In the worst case, it can lead to [this](https://github.com/GrammaticalFramework/GF/pull/28).

It's a long thread, so here's the essential.

![cautionary-tale](/images/variants.png "A cautionary tale")

Here's an [alternative hack](https://github.com/GrammaticalFramework/GF/issues/32),
if you need the behaviour of `variants` but it's too slow. However, consider
also whether you really want variants--e.g. in the thread in question,
it is much better to separate European and Brazilian Portuguese into
distinct modules.

### `variants` vs. `|`

You may have come across the use of `variants` while reading GF
code. Except for the behaviour of [empty variants](#empty-variants),
`t1 | ... | tn` and `variants {t1 ; ... ; tn}` are equivalent.

### Nested variants

It is clear that `{s = "ink well" | "ink-well"}` is the same as `{s =
"ink well"} | {s = "ink-well}`, but which is better? Both will compile
to the same thing [at the PGF level](../../06/13/pmcfg.html#variants), but the former is clearer and less
repetitive. Just be careful with examples which are not actually
equivalent, such as `{s = "Auto" ; g = Neutr} | {s = "Wagen" ; g =
Masc}` and `{s = "Auto" | "Wagen" ; g = Neutr | Masc}`.

___

## A bit about types

**Update 2018-12-28**: I thought this part deserved its own post, go and [read it](/gf/2018/12/28/dependent-types.html).

___

## Some overlooked GF shell commands

First tip: type `help` in the GF shell and read the output! You might
find cool features.

Second tip: read on for a random list of things I think are useful and
suspect are less known.

### ai

The sophisticated version of opening a terminal in the
`gf-rgl/src/abstract` and grepping for a RGL function to figure out
what it's doing.

`ai`, short for `abstract_info`, shows info about expressions, functions and
categories. Load a grammar, and type e.g. `ai VPSlash`: you get all
functions that produce a  `VPSlash`. Start typing `ai In` and press
tab, you see all functions that start with `In`. Works for whole
expressions as well: `ai UseN dog_N` tells you that it's of type `CN`
and has the probability 3.3967391304347825e-4.

Here's a concrete use case: you're implementing a resource grammar but
nothing comes out when you do `gt` or `gr`. Probably you're missing
some function along the way, but which one? There's so many of them!
You can start by typing  `ai Phr`, which is the start category; this
gives you `fun PhrUtt : PConj -> Utt -> Voc -> Phr`, so you can go and
search for `PConj`s, `Utt`s and `Voc`s next. You can use `pg -missing`
and grep for the functions in a category to see if you're missing
linearisations. Here's a concrete example:

```
Lang> ai Voc
cat Voc ;

fun NoVoc : Voc ;
fun VocNP : NP -> Voc ;
fun please_Voc : Voc ;
```

You can of course check your files in your favourite text editor or
IDE and see if any of those is implemented, or you can also do the
following in the GF shell:

```
Lang> pg -missing | ? grep -o Voc
```

Then, if you are missing any of `NoVoc`, `please_Voc` or `VocNP`, it
will show in the output.

### ma

Wouldn't it be handy if you just typed a word in the GF shell and
could find out which abstract syntax function(s) it comes from, and
which fields? `ma`, short for `morpho_analyse`, does this as follows:

```
Lang> ma -lang=Dut "het"
het
ProgrVP : a2
PossPron : sp Sg Neutr
DefArt : s True Sg Neutr
DefArt : s False Sg Neutr
...
```

The `ma` command is useful also if you're dealing with GF text that is first generated in GF, but postprocessed with another program after that generation. If you need to debug or reverse engineer such output, then the `p` command won't work, because it contains strings outside the grammar. But you can still `ma` a text like that: it just ignores the words that it doesn't recognise. Example:

```
Lang> p "책과 고양이가 좋습니다"
The parser failed at token 1: "\52293\44284"

Lang> ma "책과 고양이가 좋습니다"
책과

고양이가
cat_N : s Subject

좋습니다
good_A : s (VF Formal Pos)
```

You still don't know what the whole sentence is about, but at least you know its subject is a cat 😺 and something is good.

<a name="ma-for-ambiguity"></a>Another unexpected use case for `ma` is if your grammar is extremely ambiguous. Say you have a few empty strings in strategic places, or are using a large lexicon with tons of synonyms--you'll easily get hundreds or even thousands of parses for a 3-word sentence. Example:

```
Lang> p "작은 고양이가 좋습니다" | ? wc
     359   11232   86068

Lang> ma "작은 고양이가 좋습니다"
작은
small_A : s (VAttr Pos)
short_A : s (VAttr Pos)

고양이가
cat_N : s Subject

좋습니다
good_A : s (VF Formal Pos)
```

This is much more readable than 359 trees. The subject is a small or short cat, and the predicate is that the cat is good. Just by seeing the morphological parameters from the inflection tables, we can infer that `small` is attributive and `good` is predicative.

### ca

If your grammar includes bind tokens (`&+`), the standard GF shell can't parse it out of the box. You can either switch to the C shell (install GF with C runtime support and then type `gf -cshell`), or preprocess your input with `clitic_analyse`, `ca` for short. For this to work, you need a list of clitics in the GF grammar.

The Korean RG uses clitics for Conj (e.g. 고, 하고, 며, 이며), Prep (e.g. 와, 과) and Predet (e.g. 마다), and a few other things. If the grammar uses a the BIND token a lot, it's honestly pretty terrible to try to gather them all in order use `ca`, but let's just show this for the sake of example. Let's go back to the sentence we couldn't parse in the [`ma`](#ma) section: "책과 고양이가 좋습니다".

```
Lang> ca -clitics=와,과,고,하고,며,이며 "책과 고양이가 좋습니다"
책 &+ 과 고양이가 좋습니다
```

`ca` gives a hint why we couldn't parse it: the first word is in fact two tokens, `책 &+ 과`. The output of the `ca` command can be directly piped into other commands.

```
Lang> ca -clitics=와,과,고,하고,며,이며 "책과 고양이가 좋습니다" | p | ? wc
    1799   62920  496268
```

As you can see, `p` is more than happy to parse that stuff, it can find 1799 trees for those 3 words! What an improvement to `The parser failed at token 1: "\52293\44284"`. Let's try `ma`:

```
Lang> ca -clitics=와,과,고,하고,며,이며 "책과 고양이가 좋습니다" | ma
책
book_N : s Bare

&+

과
with_Prep : s Consonant
married_A2 : p2 s Consonant

고양이가
cat_N : s Subject

좋습니다
good_A : s (VF Formal Pos)
```

So it's either a cat with a book or cat married to a book that is good. The mystery is solved.


### tt

`tt`, short for `to_trie`, outputs the results of a parse in a trie
format, so you can see at a glance where the differences lie. Use it
in place of `l`, like in the example below:

```
MiniLang> p "you are a cat" | tt
* UttS
    * UseCl
        * TSim
          PPos
          PredVP
            * UsePron
                1 youPl_Pron
                2 youSg_Pron
              UseNP
                * DetCN
                    * a_Det
                      UseN
                        * cat_N
```

### The coolest flags of `l`

Isn't it boring when you generate poetry using `gr` and you get a
really good sentence, but don't see the AST? Then you need to copy the
sentence and parse it all over again, so much hassle.


```
Lang> gr | l
en wie alsjeblieft
and who please
ja kuka ole hyvä
```

Fear not, with the `-treebank` flag you get it all in one step:

```
Lang> gr | l -treebank
Lang: PhrUtt (PConjConj and_Conj) (UttIP whoSg_IP) please_Voc
LangDut: en wie alsjeblieft
LangEng: and who please
LangFin: ja kuka ole hyvä
```

Also wouldn't it be nice to see all forms of a tree? `-table` gives
you that:

```
Lang> gr -cat=NP | l -treebank -table -lang=Eng,Dut
Lang: UsePron she_Pron

LangEng: s (NCase Nom) : she
LangEng: s (NCase Gen) : her
LangEng: s NPAcc : her
LangEng: s NPNomPoss : hers

LangDut: s NPNom : zij
LangDut: s NPAcc : haar
```

How about more internal structure? `-bracket` is just what you need:

```
Lang> gr -cat=Cl | l -bracket -treebank
Lang: PredVP (languageNP norwegian_Language) tired_VP
LangDut: (Cl:3 (NP:1 (Language:0 Noors)) (VP:2 is) (VP:2 moe))
LangEng: (Cl:3 (NP:1 (Language:0 Norwegian)) (VP:2 is) (VP:2 tired))
LangFin: (Cl:3 (NP:1 (Language:0 norjaa)) (VP:2 väsyttää))
```

Find out the rest by typing `help l` in the GF shell.

___

## Generating MissingXxx

When you have a resource grammar that is not quite complete, but you still want to have the API functions for it, you generate a `MissingXxx.gf` (where `Xxx` is the extension of your language). The file itself looks like [this](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/basque/MissingEus.gf#L3). **You need to add `** open MissingXxx in {}` in front of every file in the `api` directory, like [this](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/api/ConstructorsLat.gf#L3-L4)**. If it still doesn't work, try to include the directory in the header, like this: `--# -path=.:alltenses:prelude:../Xxx`. But I've seen it work without the latter so I'm not quite sure, maybe it depends on some environment variables or the position of the stars.

Here's how to generate one, mostly for myself so I don't need to type these again next time. :-P

```haskell
-- (in GF shell)
Lang> pg -missing | ? tr ' ' '\n' > /tmp/missingLins.txt
```

You can also prepare the file in the language directory:

```
echo "resource MissingXxx = open GrammarXxx, Prelude in {" > MissingXxx.gf
echo "-- temporary definitions to enable the compilation of RGL API" >> MissingXxx.gf
```

Then go to the directory `abstract`, and do this:

```
for l in `cat /tmp/missingLins.txt` ; do egrep -o "\<$l\> +:.*;" *.gf | tr -s ' ' | egrep -v "(Definition|Document|Inflection|Language|Mark|Month|Monthday|Tag|Timeunit|Hour|Weekday|Year|Month)" | sed 's/gf:/ ;/g' | cut -f 2 -d ';' | sed -E 's/(.*) : (.*)/oper \1 : \2 = notYet "\1" ;/' ; done >> ../XXX/MissingXXX.gf
```

And remember to add the ending brace to the `MissingXxx` file.

It won't work straight out of the box, because it only greps the abstract files, and some old definitions are in the file commented out. Just fix those as you try to build and get an error.

If you also haven't implemented `SymbolXxx`, then you need to find missing lins from there as well. Open SymbolXxx in GF shell, do `pg -missing` and complete the same procedure.  And on the first line you need to add `resource MissingXxx = open GrammarXxx,` **SymbolXxx**`, Prelude in …`.

___

## Metavariables, or those question marks that appear when parsing

Has something similar happened to you before? You parse a completely
unambiguous sentence, and get mysterious question marks in the tree.

![metavariable ?6](/images/metavariable.png "Parsing a string and getting ?1 and ?2 in the tree")

Follow-up question: have you seen this before?

```haskell
             -- Cl = {s : Polarity => Tense => Str}
UseCl tense pol cl = {
  s = tense.s ++ pol.s ++ cl.s ! pol.p ! tense.t
  } ;

               -- p : Polarity
PPos  = {s = [] ; p = Pos} ;
PNeg  = {s = [] ; p = Neg} ;

               -- t : Tense
TSim  = {s = [] ; t = Sim} ;
TAnt  = {s = [] ; t = Ant} ;

```

In other words, all the tenses and polarities have an empty `s` field,
but `UseCl` still adds it to the linearisation.

The latter is done exactly to prevent the first situation from
happening.  The rule is that every argument needs to contribute with a
*string*, otherwise it isn't recognised when parsing. This happens
even if there is no ambiguity: the parameters `Pos`, `Neg`, `Sim` and
`Ant` all select a different form from the table in `Cl`.

The solution is simply to include some string field in every lincat,
even if its only purpose is to select a form from an inflection table
in another category.

Sometimes you run into these situations even when there is a string:
for instance, say that you implement obligatory prodrop in the following way.

```haskell
PredVP np vp =
  let subj = case np.isPron of {
      	       True  => [] ;
     	       False => np.s ! Nom
      } ;
      pred = vp.s ! np.a ;
  in {s = subj ++ pred} ;
```

Then when you parse "the cat sleeps", the `cat_N` has contributed with
a string that is selected in `np.s ! Nom`, but in case of "I sleep", `subj` becomes just the empty
string.  The solution is to add some dummy string field to your `NP`
type, and add it in case the `NP` is a pronoun, as shown below.

```haskell
-- NP = {s : Case => Str ; empty : Str}
let subj = case np.isPron of {
      True  => np.empty ;
      False => np.s ! Nom
   } ;
```
<!--
So why do they even appear in the first place? Quoting http://www.cse.chalmers.se/alumni/bringert/publ/pgf/pgf.pdf
> Metavariables are an extension useful for several purposes. In particular, they are usedfor encoding suppressed arguments when parsing grammars with erasing linearization rules (Section 4.5). In dependent type checking, they are used for unifying type dependencies,and in interactive syntax editors (Khegai et al. 2003), they are used for marking unfinished parts of syntax trees.
-->

___

## Placeholders/empty linearisations

Some words have incomplete paradigms: a form just doesn't exist. One
example of this is
[plurale tantum](https://en.wikipedia.org/wiki/Plurale_tantum) nouns:
you can't say *a scissor*, only *scissors*. In GF, you have a few
different alternatives: linearising an empty string, another form, a
nonexistent form, or raising an exception. I go through the options in
the following.

#### Linearise an empty string to mark a nonexistent option

In the case of scissors, you would create the following inflection table:

```haskell
Sg => "" ;
Pl => "scissors" ;
```

First of all, linearising empty strings results in weird
results:

```
Lang> l PredVP (UsePron i_Pron) (AdvVP (UseV run_V) (PrepNP with_Prep (MassNP (UseN scissors_N))))
I run with
```

Furthermore, it creates ambiguity, which can be
dangerous. An example (real cases are often much more subtle and hard
to debug): suppose that `today_Adv` is linearised as an empty
string. Then this happens:

```
Lang> l PredVP (UsePron i_Pron) (AdvVP (UseV run_V) today_Adv)
I run

Lang> l PredVP (UsePron i_Pron) (UseV run_V)
I run
```

The sentence *I run* is now ambiguous, you could get both trees when
parsing it. For these reasons, most often you should consider other options.


#### Linearise some other form that exists


```haskell
Sg => "scissors" ;
Pl => "scissors" ;
```

Now we get *I run with scissors* when applying `MassNP` to
`scissors_N`, just like you get *I run with beer* for the `MassNP` of
`beer_N`. It's pretty much what we'd like `MassNP` to do, so this works
much better than the previous approach! The downside is that the
grammar will now overgenerate sentences with wrong agreement.

```
Lang> l PredVP (DetCN (DetQuant DefArt NumPl) (UseN scissor_N)) (UseComp (CompAdv here_Adv))
the scissors are here

Lang> l PredVP (DetCN (DetQuant DefArt NumSg) (UseN scissor_N)) (UseComp (CompAdv here_Adv))
the scissors is here
```

Also, problems with ambiguity may occur, which can give weird results
for translation: you parse "I run with scissors" in English, get two
trees, accidentally choose the singular, and linearise in some other
language "I run with scissor". The core point is parsing text
that makes sense and getting trees that don't--we return to it [later in
this post](#too-good-linearisations-for-some-rgl-functions).)

If you think any of the two points will be a problem in your use case,
proceed with caution. Otherwise, I think this approach would work
pretty well for many applications.

#### Linearise theoretical forms and let the application grammarian to deal with them


```haskell
Sg => "scissor" ;
Pl => "scissors" ;
```

Just put *scissor* in the table, and if someone needs it in an
application, let them take care to always choose the plural.

Okay, *scissor* is not such a controversial thing to put in an
inflection table. After all, the form does exist e.g. in
[compound words](http://images4.fanpop.com/image/photos/16700000/Edward-Scissorhands-edward-scissorhands-16774394-488-720.jpg). But
in my personal opinion, this is a feasible option even for more questionable forms.

I'm mostly thinking about RGL, which is already full
of somewhat questionable language. Consider the following trees:

```haskell
PredVP (UsePron i_Pron) (ComplSlash (SlashV2a like_V2) (UsePron i_Pron))
-- 'I like me'

PredVP (UsePron i_Pron) (ReflVP (SlashV2a like_V2))
-- 'I like myself'
```

*I like me* sounds less idiomatic in English than *I like myself*, but
the RGL's task is not to judge what is idiomatic, just to provide a
wide variety of syntactic structures, and let an application
grammarian to choose the most idiomatic structures for a given
language and situation.

The first sentence is created by a construction, which creates all
combinations of `<Pron> <Verb> <Pron>`. Sure, we could write a special
case for those instances where the two pronouns are the same, and
replace the second pronoun by the appropriate reflexive pronoun. But
the reflexive construction already exists in the RGL, as a completely
separate entity. I personally think both can coexist in the RGL: *I like me*
has distinct uses from *I like myself*, for instance as a [hypothetical](https://twitter.com/GretchenAMcC/status/1134941280277073920) "I wouldn't like me if I was you".

Now let's consider Basque, where transitive verbs agree with both
subject and object. Except when the subject and object are both the
same 1st or 2nd person pronoun. So *he/she/they like(s) him/her/them* is
all fine, but *I/we like me/us* or *you like you*--the verb form
needed to convey this information just doesn't exist. In such a case,
the reflexive construction (where the object agreement is always 3rd
person singular) is the only way to go.

Despite this, I decided to linearise nonexisting forms: the
morpheme for e.g. 1st person as a subject exists, so does the morpheme
for 1st person as an object, just put the morphemes together and call
it a theoretical form<sup>[1](#def)</sup>.<a name="fn-1"></a>

This approach is definitely not for all use cases. For instance, don't
use it in applications where you provide suggestions for the next
word, unless your users are into speculative morphotactics.

#### Raise an exception

```haskell
Sg => nonExist ; --if reached, nothing is linearised
Pl => "scissors" ;
```

As defined in
[Orthography Engineering in Grammatical Framework](http://www.aclweb.org/anthology/W15-3305)
(Angelov, 2015):

> In generation mode, `nonExist` behaves like an exception.  Any
> attempt to linearise a sentence which contains `nonExist` will fail
> completely and no output will be generated.  Ideally the grammarian
> should avoid exceptional situations and write grammars that always
> produce sensible output.  At least this should be the goal for
> expressions generated from the intended start category, i.e.  for
> sentences.  Avoiding the exceptions is usually possible by using
> parameters to control the possible choices in the grammar.  `nonExist`
> is still useful as an explicit marker for impossible combinations in
> nested categories, i.e.  categories that are not used as start
> categories.  If hiding all exceptions in the grammar is not possible
> then at least by using `nonExist` the user gets a chance to handle the
> situation by rephrasing the request.

#### Empty variants

If you want to skip the linearisation of a whole abstract syntax
function, instead of just a single form, you can use empty variants:

```haskell
lin beer_N = variants {} ;
```
Using `beer_N` in a sentence works like this:

```
Lang> l PredVP (UsePron i_Pron) (ComplSlash (SlashV2a like_V2) (MassNP (UseN beer_N)))
I like [beer_N]
```

If we had defined `beer_N` as a table `\\_ => nonExist`, then
linearising the sentence would have produced no output whatsoever. Now
at least we get *I like* part properly linearised.

___

## Wrong claim that some function is unimplemented

You're implementing a concrete syntax for some grammar, and try to linearise a test tree. In the example below, we are linearising the tree for "second cat" in English and Malay:

```haskell
Lang> l AdjCN (AdjOrd (OrdNumeral (num (pot2as3 (pot1as2 (pot0as1 (pot0 n2))))))) (UseN cat_N)
second cat
kucing [AdjOrd]
```

Based on the output, the subtree `UseN cat_N` is implemented in Malay, but the function `AdjOrd` is not. But you are sure that you have implemented `AdjOrd`, so what is the problem?

In this case, the unimplemented function is not `AdjOrd`, but one (or more) of the functions in its subtree(s), so e.g. `OrdNumeral`, `num`, or any of the `pot…` funs. This is confusing, because the shell output puts the blame on a function that actually exists.

To find out which function is the actually missing one, do the following:

1. Import the grammar with the flag `-retain`
2. Use the command `cc` (compute_concrete) on the same tree

```
Lang> i -retain LangMay.gf
40 msec
> cc AdjCN (AdjOrd (OrdNumeral (num (pot2as3 (pot1as2 (pot0as1 (pot0 n2))))))) (UseN cat_N)
constant not found: OrdNumeral
```

So now you know which one is missing, go and implement `OrdNumeral`, and then you can try again linearising the original tree for "second cat".

___

## Mysterious empty strings even though lin exists

You've implemented a linearisation for some fun, but when testing some tree in the GF shell, you only get an empty string. *(NB. This is a made up example, if you try this in the actual RGL, it will work correctly.)* But suppose, for the sake of example, that you try the following:


```haskell
$ gf AllEng.gf
AllEngAbs> l ApposNP nothing_NP something_NP


```

The result is just an empty string.

Then you grep for the functions in the source code:

```haskell
StructuralEng.gf
109:  something_NP = regNP "something" singular ;

StructuralEng.gf
148:  nothing_NP = regNP "nothing" singular ;

ExtendEng.gf
435:    ApposNP np1 np2 = {s = \\c => np1.s ! c ++ comma ++ np2.s ! c; a = np1.a} ;
```

As we can see, all three functions are defined in the files. But the issue here is that `ExtendEng` is based on [ExtendFunctor](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/common/ExtendFunctor.gf), and the default linearisation for `ApposNP` (and most other functions) is `variants {}`.

So in your own implementation of `ExtendLang`, you will need to do two things for each function:

1. Write the actual linearisation; and
2. Override the default linearisation from ExtendFunctor, as in [here](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/english/ExtendEng.gf#L4-L7). The actual English works because `ApposNP` is in the exclusion list.

This holds for any concrete syntax that is based on a functor. If you override a functor implementation, you will need to add it specifically to the exlucison list. Although `ExtendFunctor` is rather uncommon in that most linearisations are `variants {}`, which is usually not the case in functors for application grammars.


___

## Too good linearisations for some RGL functions

For instance, `DetNP every_Det` returns "everything" for English.
One can argue that it's better for as many RGL trees as possible to return something that makes sense;
in such a case, "everything is blue" or "I see everything" is preferrable to "every is blue" and "I see every".
On the other hand, such (over)engineered solution creates ambiguity: now "everything" is a linearisation of both `everything_NP` and `DetNP every_Det`. Furthermore, it forces other languages to do something that makes equally much sense, because now you will get more nonsensical trees while parsing text that makes sense.

___

## Re-export RGL opers in application grammar

Here's what you can do in any grammar in the GF shell: load any grammar, parse and linearise arbitrary trees in the grammar.

```haskell
Foods> l Pred (These Pizza) Delicious
estas pizzas son deliciosas

Foods> p "este vino es caro"
Pred (This Wine) Expensive
```

Here's another standard thing we can do in the GF shell: import the Paradigms module with `retain`, and create any inflection table.

```haskell
$ gf
> i -retain alltenses/ParadigmsSpa.gfo
> cc -table mkN "tortilla"
s . ParamX.Sg => tortilla
s . ParamX.Pl => tortillas
g . CommonRomance.Fem
```

But what about combining the two? What if I want to create sentences about delicious tortillas, but can't be bothered to extend the abstract syntax of Foods? Wouldn't it be nice if I could just do this?

<div class="language-haskell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="err">Foods> l Pred (That (mkN "tortilla")) Delicious
esa tortilla es deliciosa
</span></code></pre></div></div>


Unfortunately this won't work exactly like that. But you can get very close. Let's look at the [concrete syntax](https://github.com/inariksit/gf-contrib/blob/re-export/foods/FoodsSpa.gf#L1-L28):

```haskell
lincat
  Comment = Utt ;
  Item = NP ;
  Kind = CN ;
  Quality = AP ;
lin
  Pred item quality = mkUtt (mkCl item quality) ;
  This kind = mkNP this_QuantSg kind ;
  Mod quality kind = mkCN quality kind ;
  Very quality = mkAP very_AdA quality ;
  Wine = mkCN (mkN "vino") ;
  Pizza = mkCN (mkN "pizza") ;
  Delicious = mkAP (mkA "delicioso") ;
```

Now we know which RGL categories to use, so we can be more precise. Kinds are CNs, so we need `mkN` and `mkCN` to write make a `Kind` out of the string "tortilla". These opers, `mkN` and `mkCN`, are in scope when writing the grammar, because the grammar *opens* ParadigmsSpa and SyntaxSpa. But these opers are only *used* in the Foods grammar, they aren't retained when we import the Foods grammar in the GF shell using `i -retain`.

So how to make those `mkN` and `mkCN` usable? We can __re-export__ them. Add the following lines in the concrete syntax:

```haskell
oper
   mkN : Str -> N = ParadigmsSpa.mkN ;
   mkCN : N -> CN = SyntaxSpa.mkCN ;
```

Now you can do the following in the GF shell:

```haskell
$ gf
> i -retain FoodsSpa.gf
> cc -one Pred (These (mkCN (mkN "tortilla"))) Delicious
estas tortillas son deliciosas
```

You can repeat the process for other parts of speech. Want to say that the tortilla is vegan? Just check what's the lincat for Quality and re-export the appropriate opers from ParadigmsSpa and SyntaxSpa. In fact, you don't even need to call the functions the same. You could as well do this:

```haskell
oper
  kind : Str -> CN = \s -> mkCN (mkN s) ;
  qual : Str -> AP = \s -> mkAP (mkA s) ;
```

To be used in the GF shell like this:

```haskell
> cc -one Pred (These (kind "tortilla")) (qual "vegano")
estas tortillas son veganas
```

The full grammar, including my additions, is [here](https://github.com/inariksit/gf-contrib/blob/re-export/foods/FoodsSpa.gf#L29).

___

## Naming conventions

There is no official style guide for GF, but if you look at the existing grammars, there are many conventions. I also have my own set of practices that I use from grammar to grammar, but which are not necessarily used by others. I share here both kinds of naming schemes.

### Names of categories

The biggest distinction is application grammar categories vs. RGL categories.

#### RGL categories
The general principle is that RGL cats are purely syntactic, and are named after the part of speech, e.g. `A`, `N`, `V` and their phrasal forms `AP`, `NP`, `VP`. Numbers after the category mean how many arguments they take:
* `V` means *intransitive verb*: only subject, no object, e.g. `sleep_V`
* `V2` means *transitive verb*: subject and object, e.g. `break_V2`
* `V3` means *ditransitive verb*: subject, object and indirect object, e.g. `give_V3`.

Nouns and adjectives follow the same number scheme, so we have adjectives like `married_A2` "*married to X*" and nouns like `distance_N3` "*distance from Y to Z*".

For RGL verbs, we have even more subcategories, such as `VV`, `VA` and `VS`. The names are hopefully compositional enough: `VV` takes a verbal complement (*want [to sleep]*), `VA` an adjectival complement (*become [tired]*) and `VS` takes a whole sentence (*say [that she sleeps]*). These can also be combined with the numbers, so for example `V2A` takes both a direct object and an adjectival complement: *paint [the house] [red]*.

#### Application grammar categories

Often the categories of an application grammar are more fine-grained: not just part-of-speech based like `NP` or `V2`, but more semantic distinctions like `Weekday`, `Currency`, `Language`, `Action`, `State`.

Some application grammars are built from scratch, and all of their categories are like above. Other application grammars [extend the RGL](https://arxiv.org/abs/1406.4057), and thus contain both the RGL categories and more semantically oriented categories. There are not that many conventions, it just depends on your application what is significant.

### Names of fields

#### The `s` field

Most GF records, no matter if in a resource or an application grammar, have a field called `s`, which contains the main element. For instance, the lincat of CN could look like this in many languages:

```haskell
lincat CN = {
  s : Number => Str ; -- house / houses
  postmod : Str ;     -- on a hill
  } ;
```

The `s` field is significant to the GF shell: unless we have specified a [linref](#linref), the GF shell will only parse and linearise contents in the `s` field and ignore the rest. Similarly, human GF programmers are used to finding more central contents in the `s` and more peripheral contents in other fields.


#### Common field names in the RGL internals

The following names are mostly interesting for RGL implementors, or those who live dangerously and use the RGL not via the API, but touching the raw parameters.

#### `sp`

20 RGL languages use a field called `sp` (source: I went to gf-rgl/src and did `grep  "\<sp :" */*.gf | cut -f 1 -d '/' | sort -u`).

The `sp` field appears in the category `Det` and other categories that may become Det (e.g. `Quant`, `Num`, `Pron`). It is meant to be the field that contains a *standalone* version of the Det. A simplified example for English:

```haskell
l -table PossPron i_Pron
s  : my
sp : mine
```

The `s` field is then used in the primary function of Det, which is being a determiner. The `sp` field is used when the determiner is promoted into a NP. This is from the actual English resource grammar:

```
Lang> p -cat=NP "my dog"
DetCN (DetQuant (PossPron i_Pron) NumSg) (UseN dog_N)

Lang> p -cat=NP "mine"
DetNP (DetQuant (PossPron i_Pron) NumSg)
```


#### `c2` and `c3`

The field `c2` is used in 39 RGL languages, and `c3` in 34 (source: the same bash oneliner I used for `sp`, but grep for the string `c2` or `c3` instead of `sp`).

These fields appear in categories that take complements, so almost all verb subcategories (`V2`, `VS`, `V2A`…), but also `N2`, `A2` and their ditransitive versions `N3`, `A3`. They belong to the categories that are still waiting for their complement, so `V2`'s `c2` field is inherited to the `VPSlash`, but not to `VP`. A simplified example below:

```haskell
cat
  V2 ; VP ; NP ;
fun
  ComplV2 : V2 -> NP -> VP ;
  believe_V2 : V2 ;
  see_V2 : V2 ;

lincat
  NP, VP = {s : Str} ;
  V2 = {
    s,         -- main verb ('believe', 'see')
    c2 : Str   -- optional preposition ('in')
    } ;

lin
  ComplV2 vp np = {
    s = vp.s ++ vp.c2 ++ np.s ;
    } ;

  believe_V2 = {
    s = "believe" ;
    c2 = "in"
    } ;

  see_V2 = {
    s = "see" ;
    c2 = []
    } ;
```

This works for other concepts than just prepositions, and more complex types than just strings. What kinds of types are used for those fields? You can inspect yourself with this oneliner, running it in the directory gf-rgl/src:

```
grep -o  "\<c2 : [^ ]*\>" */*.gf | cut -f 3 -d ':' | sort -u
```

Here are the cleaned up results, merging e.g. `ResNep.Compl` together with `Compl`:

 * Adposition
 * Agr
 * Agreement
 * Case
 * Clitics
 * Compl
 * Complement
 * ComplementCase
 * NForm
 * Prep
 * PrepCombination
 * Preposition
 * Str
 * {s : Str ; hasPrep : Bool}

The names don't tell much—all of those, except for `Str` and `Bool`, are internal types defined in each concrete RGL language. (For all we know, maybe earlier in the grammar it says `Compl : Type = Str`.) I just wanted to show proof that `c2` fields can contain much more than just strings. If you are interested in their relative frequencies, you can run this command in gf-rgl/src:
```
grep -o  "\<c2 : [^ ]*\>" */*.gf | cut -f 3 -d ':' | sort | uniq -c | sort -nr
```

### My naming scheme for lincats and opers

This is idiosyncratic to me, but I share it because
* I contribute to quite a few RGL languages, so you may some day debug code I wrote, and
* I find this practice helpful, and maybe you could benefit from it too in your own grammars.

So, I have a grammar—RGL or application—with some categories, let's call them `Foo` and `Bar`. Then for their lincats, I follow this convention:

```haskell
cat
  Foo ; Bar ;

lincat
  Foo = LinFoo ;
  Bar = LinBar ;

linref
  Foo = linFoo ;
  Bar = linBar ;

oper
  LinFoo : Type = {- … -} ;
  linFoo : LinFoo -> Str = \foo -> {- … -} ;

  LinBar : Type = {- … -} ;
  linBar : LinBar -> Str = \bar -> {- … -} ;

  doStuff : Str -> LinFoo -> Str = \someStr,foo ->
    someStr ++ linFoo foo ;
```

1. For a cat `C`, I create a type called `LinC` as an `oper`, and use it for the lincat of `C`.
1. For a type `LinC`, I create an oper `linC : LinC -> Str`, and use it for the linref of `C`.
1. The oper `linC` is also useful in many other places where you need to make a `C` into a string, e.g. in `doStuff`.

This is useful, because often many cats share a lincat, and you need to do a `LinC` and `linC` only once per distinct lincat. It also makes the code safer: every time you need to produce a string out a `C`, you just call `linC` instead of manually adding all the fields together. Then when you update the lincat, you only need to update in one place, that is `linC`.

<!--
If you wonder why I do this, a few reasons:
* I generally avoid using lincats as arguments for opers, because [lock fields](https://inariksit.github.io/gf/2018/05/25/subtyping-gf.html#lock-fields) sometimes cause mysterious issues. So in big grammars, I don't usually define lincats raw (`lincat Foo = {definition here}`), but first create an oper, which I then use for the lincat (`lincat Foo = LinFoo`).
* Often many cats have the same lincat, so this is a way to avoid repetition.
* Often I need to turn some category into a string in a function. and so very often, some field is missing. Even if everything works when you write the grammar, at a later stage someone may modify a lincat, but doesn't update all the places where that lincat is used. With `linC : LinC -> Str`, it only needs to be updated in one place.
-->
___

## Cute way to count syllables

A syllable consists of an _onset_, _nucleus_ and _coda_. The following image from [Wikipedia](https://en.wikipedia.org/wiki/File:Syllable_illustrations_3and4.svg) illustrates the concept:

![[p][ou][t] and [p][o][nd]](https://upload.wikimedia.org/wikipedia/commons/thumb/b/b0/Syllable_illustrations_3and4.svg/640px-Syllable_illustrations_3and4.svg.png)

Many languages allow only vowels in the nucleus. If this is the case, then we can easily count the syllables of a word by counting the isolated blobs of vowels. For instance:
- a, ba, ab, bab = 1
- aba, baba, abab, babab, aaba, baaba, abaa, babaa, … = 2
- ababa, bababa, ababab, … = 3

Enumerating all options gets tedious soon. In a normal programming language, you could write a function like this:

```haskell
isVowel = (`elem` "aeiou")

countSyllables :: String -> Int
countSyllables = go False
  where
    go _ [] = 0
    go wasVowel (c:cs)
      | wasVowel  = go (isVowel c) cs
      | isVowel c = go True cs + 1
      | otherwise = go False cs
```

But in GF, we can't have nice things, something reversibility something not Turing-complete. But if I really don't want to list every possible syllable structure, what can I do in GF?

I'm glad you asked:

```haskell
-- Lithuanian vowels for the sake of example—substitute with your own!
v : pattern Str = #("a"|"ā"|"e"|"ē"|"i"|"ī"|"o"|"u"|"ū") ;

countSyllables : Str -> Ints 10 = \word  ->
  case word of {
      #v + s => Predef.plus (sc s True) 1 ;
      ?  + s => sc s False ;
      _      => 0
  } ;

SylCnt : Type = Str -> Bool -> Ints 10 ;
sc : SylCnt = mkSC {-finite # of mkSC-}  (mkSC scBase)
  where {
    mkSC : SylCnt -> SylCnt = \sc,word,afterVowel ->
        case <word,afterVowel> of {
          <#v + s, False> => Predef.plus (sc s True) 1 ;
          <#v + s, True>  => sc s True ;
          <?  + s, _>     => sc s False ;
          _               => 0 } ;
    scBase : SylCnt = \_,_ -> 0
  } ;
```

This will check as many characters of a word as you type `mkSC` before `scBase`. For instance, this works for words up to 40 characters:

```haskell
mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC (mkSC scBase)))))))))))))))))))))))))))))))))))))))
```

This looks horrible but I honestly think it's nicer than typing out every single combination of  vowels and consonants (ababa, bababa, ababab, abbaba, aababa, …).


## Footnotes


1.<a name="def"> </a>Actually, I wonder if the
[def](http://www.grammaticalframework.org/doc/gf-refman.html#toc19)
feature would work here--just define all trees of form

```haskell
PredVP (UsePron p) (ComplSlash (SlashV2a v) (UsePron p))
```

as the corresponding tree using `ReflVP`:

```haskell
PredVP (UsePron p) (ReflVP (SlashV2a v))
```

In practice, it won't work in the RGL, because relevant funs are defined as `fun`, and they'd need to be defined as `data`.

However, if you want to make such a thing work for your own application, here's a [pull request](https://github.com/GrammaticalFramework/gf-rgl/pull/8) that you can use in your own copy of the RGL. <a href="#fn-1">↩</a>
