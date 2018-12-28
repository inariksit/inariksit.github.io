---
layout: post
title:  "Programming in GF: random tips and gotchas"
date:   2018-08-28
categories: gf
tags: gf
---

![duck](/images/gf-rubber-duck.png "Your favourite companion for writing GF")
Latest update: 2018-12-28

This post contains real-life examples when I or my friends have been
confused in the past. It might be updated whenever someone is confused
again.

- [Restricted inheritance](#restricted-inheritance)
- [linref](#linref)
- [What's the problem with variants in resource grammar?](#whats-the-problem-with-variants-in-resource-grammar)
- [A bit about types](#a-bit-about-types)
- [Some overlooked GF shell commands](#some-overlooked-gf-shell-commands)
  - [ai](#ai)
  - [ma](#ma)
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
- [Too good linearisations for some RGL functions](#too-good-linearisations-for-some-rgl-functions)


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

## linref

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


## What's the problem with variants in resource grammar?

In the worst case, it can lead to [this](https://github.com/GrammaticalFramework/GF/pull/28).

It's a long thread, so here's the essential.

![cautionary-tale](/images/variants.png "A cautionary tale")

Here's an [alternative hack](https://github.com/GrammaticalFramework/GF/issues/32),
if you need the behaviour of `variants` but it's too slow. However, consider
also whether you really want variants--e.g. in the thread in question,
it is much better to separate European and Brazilian Portuguese into
distinct modules.

### Variants vs. `|`

You may have come across the use of `variants` while reading GF
code. Except for the behaviour of [empty variants](#empty-variants),
`t1 | ... | tn` and `variants {t1 ; ... ; tn}` are equivalent.

### Nested variants

It is clear that `{s = "ink well" | "ink-well"}` is the same as `{s =
"ink well"} | {s = "ink-well}`, but which is better? Both will compile
to the same thing at the PGF level, but the former is clearer and less
repetitive. Just be careful with examples which are not actually
equivalent, such as `{s = "Auto" ; g = Neutr} | {s = "Wagen" ; g =
Masc}` and `{s = "Auto" | "Wagen" ; g = Neutr | Masc}`.


## A bit about types

**Update 2018-12-28**: I thought this part deserved its own post, go and [read it](/gf/2018/12/28/dependent-types.html).

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
for l in `cat /tmp/missingLins.txt` ; do egrep -o "\<$l\> +:.*;" *.gf | tr -s ' ' | egrep -v "(Definition|Document|Inflection|Language|Mark|Month|Monthday|Tag|Timeunit|Weekday|Year|Month)" | sed 's/gf:/ ;/g' | cut -f 2 -d ';' | sed -E 's/(.*) : (.*)/oper \1 : \2 = notYet "\1" ;/' ; done >> ../XXX/MissingXXX.gf
```

And remember to add the ending brace to the `MissingXxx` file.

It won't work straight out of the box, because it only greps the abstract files, and some old definitions are in the file commented out. Just fix those as you try to build and get an error.

If you also haven't implemented `SymbolXxx`, then you need to find missing lins from there as well. Open SymbolXxx in GF shell, do `pg -missing` and complete the same procedure.  And on the first line you need to add `resource MissingXxx = open GrammarXxx,` **SymbolXxx**`, Prelude in …`.

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
in my personal opinion<sup>[1](#opinion)</sup>,<a name="fn-1"></a>
this is a feasible option even for more questionable forms.

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
separate entity. I personally think both can coexist in the RGL. Maybe
*I like me* sounds weird in English, but in some language it could
well sound natural. Or it could apply to an extraordinary situation,
involving clones, time travel etc., in a way that *I like myself*
doesn't quite capture.

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
it a theoretical form<sup>[2](#def)</sup>.<a name="fn-2"></a>

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

## Too good linearisations for some RGL functions

For instance, `DetNP every_Det` returns "everything" for English.
One can argue that it's better for as many RGL trees as possible to return something that makes sense;
in such a case, "everything is blue" or "I see everything" is preferrable to "every is blue" and "I see every".
On the other hand, such (over)engineered solution creates ambiguity: now "everything" is a linearisation of both `everything_NP` and `DetNP every_Det`. Furthermore, it forces other languages to do something that makes equally much sense, because now you will get more nonsensical trees while parsing text that makes sense.


## Footnotes



1.<a name="opinion"> </a>My nick *inariksit* is a combination of Finnish morphemes
in an illegal way, so maybe I just like breaking the
~~law~~ morphotactics. <a href="#fn-1">↩</a>

2.<a name="def"> </a>Actually, I wonder if the
[def](http://www.grammaticalframework.org/doc/gf-refman.html#toc19)
feature would work here--just define all trees of form

```haskell
PredVP (UsePron p) (ComplSlash (SlashV2a v) (UsePron p))
```

as the corresponding tree using `ReflVP`:

```haskell
PredVP (UsePron p) (ReflVP (SlashV2a v))
```
<a href="#fn-2">↩</a>
