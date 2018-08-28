---
layout: post
title:  "Programming in GF: random tips and gotchas"
date:   2018-08-28
categories: gf
tags: gf
---

![duck](/images/gf-rubber-duck.png "Your favourite companion for writing GF")
Latest update: 2018-08-28

This post contains real-life examples when I or my friends have been
confused in the past. It might be updated whenever someone is confused
again.

- [Restricted inheritance](#restricted-inheritance)
- [What's the problem with variants in resource grammar?](#whats-the-problem-with-variants-in-resource-grammar)
- [`Type` and `PType`](#type-and-ptype)
- [Some overlooked GF shell commands](#some-overlooked-gf-shell-commands)
  - [ai](#ai)
  - [ma](#ma)
  - [The coolest flags of `l`](#the-coolest-flags-of-l)
  - [linref](#linref)
- [Placeholders/empty linearisations](#placeholdersempty-linearisations)
  - [Linearise an empty string](#linearise-an-empty-string-to-mark-a-nonexistent-option)
  - [Linearise some other form that exists](#linearise-some-other-form-that-exists)
  - [Linearise a theoretical form](#linearise-theoretical-forms-and-let-the-application-grammarian-to-deal-with-them)
  - [Raise an exception](#raise-an-exception)
  - [variants {}](#empty-variants)


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


## What's the problem with variants in resource grammar?

In the worst case, it can lead to [this](https://github.com/GrammaticalFramework/GF/pull/28).

It's a long thread, so here's the essential.

![cautionary-tale](/images/variants.png "A cautionary tale")

Here's an [alternative hack](https://github.com/GrammaticalFramework/GF/issues/32), if you need the behaviour of `variants` but it's too slow. However, consider also whether you really want variants--e.g. in the thread in question, it is much better to separate European and Brazilian Portuguese into distinct modules.



## `Type` and `PType`

GF was my first functional language, and certainly my first
dependently typed language<sup>[1](#dep-types)</sup>.<a name="fn-1">
</a> Back in 2010 I knew so little about types or programming that it
didn't even occur to me to think that this new thing called "dependent
types" is something particularly fancy. I heard the cool kids namedrop
lambda calculus, and I asked a friend whether we were supposed to
learn lambda calculus at school and I just missed all of it. (For
anyone else who is wondering: no, we weren't.)  And now I'm posing with
a massive lambda in my blog which is so far exclusively about
programming.

Okay, that's enough about my ~~empowering origin story~~
daily life. Let’s go to something more concrete. 

### Types are of type `Type`

You may have seen things like this before:

```haskell
oper
  Verb : Type = {- s : Agr => String ; c : Case -} ;

  copula : Verb = {- … -} ;
```
  
Look at that! It's so human-understandable. It's telling you that
`Verb` is a type, and `copula` is a `Verb`. Or in a more verbose way:
`Verb` is of type `Type`, and `copula` is of type `Verb`. There is no
reason why types cannot be *of type* something!

`PType` is the same but for `param`s. Like this:

```haskell
param
  Case = Nom | Acc | ... ;
```

The values `Nom`, `Acc` etc. are of type `Case`, and `Case` is of type `PType`.

### Type variables

Now that we're granting basic human rights to types, look at this
piece of code:

```haskell
AdjCN adj cn = {
  s = if_then_else Str adj.isPre 
        (adj.s ++ cn.s)
        (cn.s ++ adj.s)
  } ;
```

Looks kinda awkward, to be honest. You might want to mentally sugar it
into the following:

```haskell
AdjCN adj cn = {
  s = if adj.isPre
       then adj.s ++ cn.s
       else cn.s ++ adj.s
  } ;
```

Now it should look more readable. What happens is that in GF there is
no `if … then … else` keyword, just a function `if_then_else` that
takes, in our example, the following 4 arguments:

* The type of what it returns (`Str`)
* The Boolean that it checks (`adj.isPre`, i.e. whether the adjective
  is premodifier)
* What happens if the adjective *is* premodifier (put `adj.s` before
`cn.s`, return that string)
* What happens if the adjective *is not* premodifier (put `adj.s` after
`cn.s`, return that string).

Why do we need the first argument? I don't know--my understanding is
at the level that the cool expressive power that comes from dependent
types comes with a tradeoff: it is hard (or impossible?) to guess
things implicitly. Like in this case, we gave `if_then_else` two
strings as arguments, but we still needed to give the *type* `Str` as
the first argument.


This is how it is defined in the
[GF prelude](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/prelude/Prelude.gf#L64-L69):

```haskell
oper
  if_then_else : (A : Type) -> Bool -> A -> A -> A = \_,c,d,e -> 
    case c of {
      True => d ;
      False => e
} ;
```

### Dependent types

The "basic human rights for types" thing, as well as "annotate the
hell out of everything because the type checker seems like a clueless
idiot" thing is just a prerequisite for doing cool stuff with dependent types.

Despite the subheading, I am not going to go further into dependent
types in this post. You don't need them to write resource grammars, or
most application grammars; in contrast, the stuff with `Type` and type
variables you do see in the RGL all the time.

If you want to learn more about dependent types in GF, [here's the
section in the tutorial](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc110).

If you want to learn about dependent types in more general-purpose
programming languages, maybe pick up some
[Agda](http://wiki.portal.chalmers.se/agda/pmwiki.php) or [Idris](https://www.idris-lang.org/).

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

and it also appears in `CleftAdv, CleftNP, ImpersCl, weekdayPunctualAdv,
monthYearAdv` and `monthAdv`. 


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

### linref

Does this ever happen to you?

```
Lang> gr -cat=VP | l -treebank
Lang: UseComp (CompAP (UseComparA probable_AS))
LangDut: zijn
LangEng: be more probable
```

Why is there only *zijn* 'to be' in the Dutch and the whole VP in
English? Well, that's because the Dutch grammar has no `linref` for
`VP`.

`linref` is defined in the
[reference manual](http://www.grammaticalframework.org/doc/gf-refman.html#toc24_r)
as the "reference linearization of category C, i.e. the function
applicable to an object of the linearization type of C to make it into
a string". So if your `VP` is a record with a bunch of fields,
and you don't have a linref, then the GF shell shows only the thing
in the `s` field. If you have a linref, then the function is applied
to the record, and GF shell shows a string that makes sense, like *be
more probable*.

Here's some
[English linrefs](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/english/CatEng.gf#L129-L149)
if you want to have a look.



## Placeholders/empty linearisations

Some words have incomplete paradigms: a form just doesn't exist. One
example of this is
[plurale tantum](https://en.wikipedia.org/wiki/Plurale_tantum) nouns:
you cannot say *a scissor*, only *scissors*. In GF, you have a few
different alternatives: linearising an empty string, another form, a
nonexistent form, or raising an exception. I go through the options in
the following.

#### Linearise an empty string to mark a nonexistent option

In the case of scissors, you would create the following inflection table:

```haskell
Sg => "" ; 
Pl => "scissors" ;
```

This means that when you linearise `PredVP (UsePron i_Pron) (AdvVP
(UseV run_V) (PrepNP with_Prep (MassNP (UseN scissors_N))))`, you get
the perfectly grammatical *I run with*.

In general, linearising empty strings means creating ambiguity, which
can be dangerous. An example (real cases are often much more subtle
and hard to debug): suppose that `today_Adv` is linearised as an empty
string. Then this happens:

```
Lang> l PredVP (UsePron i_Pron) (AdvVP (UseV run_V) today_Adv)
I run

Lang> l PredVP (UsePron i_Pron) (UseV run_V) 
I run
```

The sentence *I run* is now ambiguous, you could get both trees when
parsing it.


#### Linearise some other form that exists


```haskell
Sg => "scissors" ; 
Pl => "scissors" ;
```

This works too, but it will overgenerate sentences with wrong
agreement.

```
Lang> l PredVP (DetCN (DetQuant DefArt NumPl) (UseN scissor_N)) (UseComp (CompAdv here_Adv))
the scissors are here

Lang> l PredVP (DetCN (DetQuant DefArt NumSg) (UseN scissor_N)) (UseComp (CompAdv here_Adv))
the scissors is here
```

If you think this will be a problem, proceed with caution. Also,
problems with ambiguity may occur, but it's not as dangerous as
linearising empty strings.

#### Linearise theoretical forms and let the application grammarian to deal with them


```haskell
Sg => "scissor" ; 
Pl => "scissors" ;
```

Just put *scissor* in the table, and if someone needs it in an
application, let them take care to never actually use the singular.

I kind of like this option, but I understand it's not for
every situation. Here's a more hardcore example: Basque verb agreement when
both subject and object is the same person doesn't exist, instead you
need to use reflexive, which is a whole different construction with
different verb agreement and words and stuff. There is already an
abstract syntax function for reflexive, so why would I need to put it
in just any old verb? I decided to linearise nonexisting forms: the
morpheme for e.g. 1st person as a subject exists, so does the morpheme
for 1st person as an object, just put the morphemes together and call
it a theoretical form<sup>[2](#def)</sup>.<a name="fn-2"></a>



Another point in favour of "theoretical forms" is that even in
English, *I like me* is a bit questionable, but still the RG
constructs that form, as a separate entity from the more natural *I
like myself*, which has a different AST. In Basque, it's just more
radical, because the verb form needed to construct *I like me* doesn't
exist.

(Side note: my nick *inariksit* is a combination of Finnish morphemes
in an illegal way, so maybe I just like breaking the
~~law~~ morphotactics.)


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



## Footnotes


1.<a name="dep-types"> </a>The only, really. I can't say I know Agda
apart from a couple of key combinations in its Emacs mode--knowledge
which I use exclusively to troll Agda programmers.<a href="#fn-1">↩</a>

2.<a name="def"> </a>Actually, I wonder if the
[def](http://www.grammaticalframework.org/doc/gf-refman.html#toc19)
feature would work here--just define the following tree

```haskell
PredVP (UsePron i_Pron) (ComplSlash (SlashV2a like_V2) (UsePron i_Pron))
-- 'I like me'
``` 
as this tree:

```haskell
PredVP (UsePron i_Pron) (ReflVP (SlashV2a like_V2))
-- 'I like myself'
```
<a href="#fn-2">↩</a>
