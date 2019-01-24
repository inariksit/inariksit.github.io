---
layout: post
title:  "Grammar blowup: some causes and solutions"
date:   2018-09-22
categories: gf
tags: gf
---

* [I. The problem](#i-the-problem)
* [II. Reducing the amount of concrete categories](#ii-reducing-the-amount-of-concrete-categories)
* [III. Reducing the size of concrete categories](#iii-reducing-the-size-of-concrete-categories)
* [IV. Trading size for amount: a case study in Basque](#iv-trading-size-for-amount)
* [V. Quick fix for testing an oversized grammar](#v-quick-fix-for-testing-an-oversized-grammar)



## I. The problem

Some GF grammars take a long time to compile and parse (and to
generate test cases when using
[gftest](https://github.com/GrammaticalFramework/gftest#readme)). The
reasons for this can be many and complicated, but we can identify two concrete reasons:

* The *amount* of [concrete
categories](/gf/2018/06/13/pmcfg.html#concrete-categories) and
[concrete functions](/gf/2018/06/13/pmcfg.html#concrete-functions) that the
grammar compiles into. For instance, Bantu languages have up to 20 noun classes<sup>[citation needed]</sup>: these all compile into different concrete categories.
(If you like staring at PGF files, look for `range` in [here](/gf/2018/06/13/pmcfg.html#concrete-categories).)

* The *size* of concrete categories. For instance, Finnish nouns have 14 cases and 2 numbers. There's just one concrete category, but they have 28 strings in them.
(Again for those comfortable with PGF, look for `labels` in [here](/gf/2018/06/13/pmcfg.html#concrete-categories).)

Both of these things can make a grammar blow up, and reducing either
can make it decent again. Especially the amount of concrete categories
can be very sneaky; you wouldn't expect that adding just a tiny
Boolean to some insignificant category would cause a blowup, but it
just might. Did you ever wonder why French is so much slower than
Spanish, even though they use the same functor? It's because French
participles agree in gender and
number<sup>[1](#belgium-reform)</sup><a name="fn-1">,</a> and Spanish
don't---a difference of two 2-valued parameters cause a major
difference in compile time.


Here's another anecdote: back in 2011 or so, I tried to add a 2-valued
vowel harmony parameter to a few categories in Finnish, and the
grammar just stopped compiling. Later, Aarne reduced the *size* of
nouns and verbs, by replacing the inflection tables of nouns and verbs
with just the minimum number of different stems (the method known as
[glu `&+` ing morph `&+` eme `&+`
s](#iii-reducing-the-size-of-concrete-categories)). That sped up
the grammar a lot, with vowel harmony and all!

So, a problem that comes from adding one parameter can be solved by
shaving off from somewhere else, either by getting rid of some other
parameter, or making the inflection tables smaller (in the changed
category, or elsewhere). You can get some ideas of the numbers when
you compile your GF grammar with the setting
`-v=3`<sup>[2](#verbosity)</sup>.<a name="fn-2"></a> The bad news is
that a lot of categories are deeply interconnected, so you really
might have no clue where to start. The good news is that if you just
start from where it seems
[*feasible*](https://en.wikipedia.org/wiki/Streetlight_effect), it's
often actually helpful.


Before we get started, here's a disclaimer. These hacks can make the
grammar less readable and harder to debug. If you don't have a
performance problem, and you suspect you won't have one in the future,
you don't need to think about these all that much.

## II. Reducing the amount of concrete categories

<!-- Sometimes adding a parameter into just one category can create an
explosion in the other categories, and slow down the grammar
considerably. In such a case, it is a good idea to rethink the
parameters in your grammar. Maybe the latest addition is really
necessary, but getting rid of some other parameter balances it out and
brings the number of categories to the previous (or even smaller)
level.-->

In order to reduce the amount of concrete categories, you need to
reduce the amount of *inherent* parameters. Here's a couple of cases,
with inspiration from real life, but simplified for the sake of
example.


### Word order

If you're thinking of adding a parameter about word order (pre or
post), often you can do without a parameter, and just create two
fields in your record type instead.

Think of any major Romance language: some adjectives, like *good* come
before the noun (*buena* pizza 'good pizza'), but most adjectives come
after (pizza *vegana* 'vegan pizza').

This is how you do it with a parameter:

```haskell
lincat
  Adj = { s : Str ; isPre : Bool } ;
  CN  = { s : Str } ;

lin
  AdjCN adj cn = {
    s = case adj.isPre of {
          True  => adj.s ++ cn.s ;
          False => cn.s ++ adj.s }
    } ;

  Good  = { s = "buena"  ; isPre = True } ;
  Vegan = { s = "vegana" ; isPre = False } ;
```

And this is how you do it without a parameter.

```haskell
lincat
  Adj = { sPre : Str ; sPost : Str } ;
  CN  = { s : Str } ;

lin
  AdjCN adj cn = { s = adj.sPre ++ cn.s ++ adj.sPost } ;

  Good  = { sPre = "buena" ; sPost = "" } ;
  Vegan = { sPre = ""      ; sPost = "vegana" } ;

linref
  Adj = \adj -> adj.sPre ++ adj.sPost ;
```

The latter solution looks uglier, and needs a
[linref](/gf/2018/08/28/gf-gotchas.html#linref) to make any sense in
the GF shell. (Alternatively, you could have a field called `s`, which
is just a copy of the non-empty member of `sPre`/`sPost`.)

This also opens up a possibility for new kinds of bugs: you might end
up with an adjective that has strings in both `sPre` and `sPost`, like
"*buena* pizza *buena*". But this cuts down the need for parameter,
i.e. does its job.

### Has `<some function>` been applied?

Consider a language, where only the last word in the noun phrase gets
the case marking, and the rest are in some unmarked base form (which
we can language-agnostically call absolutive). Both nouns and
adjectives can inflect in case, and the case inflection causes some
serious stem changes, so you cannot just be lazy and pile up the words
in the `CN -> CN` functions, and finally glue the case suffix in
whatever `CN -> NP` function you are using.

Here's one way to do it, with a parameter `hasAdj` in the `CN`, and
`DetCN` checking if its argument `CN` has an adjective. If it has,
then `DetCN` chooses the absolutive form of the noun, and only puts
the rest of the case endings at the end of the adjective.

```haskell
lincat
  CN = { s,adj : Case => Str ; hasAdj : Bool } ;
  NP = { s = Case => Str } ;

lin
  -- : Adj -> CN -> CN
  AdjCN adj cn = { -- Move previous adjectives into s.
   s   = \\c => cn.s ! c ++ cn.adj ! c ;
   adj = \\c => adj.s ! c ;
   hasAdj = True
  } ;

  -- : Det -> CN -> NP
  DetCN det cn = {
   s = case cn.hasAdj of {
        True  => \\c => det.s ++ cn.s ! Abs ++ cn.adj ! c ;
        False => \\c => det.s ++ cn.s ! c }
  } ;
```

This has the positive property that at any phase of `CN`, all forms
are intact. The `s` field contains a table with all the potential
forms of the noun, and the parameter `hasAdj` tells whether to
actually use all that potential. The parameter is necessary to get the
desired behaviour, if we want to retain all forms of the noun (and any
other adjectives that were added before the latest application of
`AdjCN`).

However, we can also do it without a parameter: when applying `AdjCN`,
force everything else but the newest adjective into absolutive. Then
you can choose any case in `DetCN` and you get the correct behaviour:
only the newest adjective gets the other case endings.

```haskell
lincat
  CN = { s,adj : Case => Str } ;
  NP = { s : Case => Str } ;

lin
  AdjCN adj cn = {
    s   = \\c => cn.s ! Abs ;
    adj = \\c => cn.adj ! Abs ++ adj.s ! c ;
  } ;

  DetCN det cn = {
    s = \\c => det.s ++ cn.s ! c ++ cn.adj ! c
  } ;
```

This means that when you select e.g. `cn.s ! Gen` of a `CN` that has
undergone `AdjCN`, you actually get absolutive, not genitive. It's up
to you how you feel about such a decision.

## III. Reducing the size of concrete categories


This time, we don't touch the inherent parameters, but the
*variable* ones, that is, the LHS of a table. If our
Noun type is `{s : Case => Str}`, we want to make the parameter `Case`
smaller.


### Inflection tables vs. glu &+ ing morph &+ eme &+ s


A traditional GF grammar would store an inflection table in the
entries for noun and verb, and the function for building sentences
would pick the right forms for the subject, verb and object. See an
example below (not a real RGL function):

```haskell
lincat
  S  = Str ;
  NP = { s : Case => Str ; agr : Agr } ;
  V2 = { s : Agr => Str ; compl : Case } ;


lin
  -- : NP -> V2 -> NP -> S ;
  MakeSentence subj verb obj = subj.s ! Nom
                            ++ verb.s ! subj.agr
                            ++ obj.s ! verb.compl ;
```

Let's take a linguist version of assuming a perfectly spherical cow in
a vacuum. Suppose that your language had 100 % regular concatenative
morphology, then you could write a GF grammar like this:

```haskell
lincat
  S  = Str ;
  NP = { s : Str ; agr : Str } ;
  V2 = { s : Str ; compl : Str } ;

lin
  -- : NP -> V2 -> NP -> S ;
  MakeSentence subj verb obj = subj.s -- base form = Nom
                            ++ glue verb.s subj.agr
                            ++ glue obj.s verb.compl ;

oper
  -- Concatenates two strings at runtime without spaces
  glue x y = x ++ BIND ++ y ;
```

In this ideal world, the category for `NP` would just contain the
morpheme that needs to be added to the verb to make it agree with the
subject `NP`, and the category for `V2` would contain the case that
needs to be added to the object `NP`.  Now (eh, since 2015 or so) that
GF has the magical `BIND` token, which concatenates two strings *at
runtime* without spaces, it is possible to do things this way.

Often real world morphemes aren't all that ideal though. (My job
wouldn't exist if they were!)  You could still get away with a similar
solution: instead of a full inflection table of, say, 50 forms, maybe
you only need 5-10 different *stems* where you attach only, say, 1-5
different allomorphs of whatever morphemes you now want to
attach. Here's an [example in the Basque
RG](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/basque/ResEus.gf#L27-L30)
that demonstrates both:

```haskell
let stem =
  itsaso ++ table {
      {- other cases … -} ;
      LocStem => table {
        FinalCons => BIND ++ "e" ; --mutil+e+tik
        FinalR    => BIND ++ "re" ; --txakur+re+tik
        FinalA    => BIND ++ "a" ; --neska+tik
        _         => [] } --itsaso+tik
   } ;

```

So, the case `LocStem` replaces locative cases, because they all
use the same stem, and the parameter `Phono` with values `FinalR,
FinalCons, FinalA, FinalVow` tells which allomorph to choose. Then the
[`locPrep`
oper](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/basque/ParadigmsEus.gf#L144-L145)
chooses the form with parameter `LocStem` from its `NP` argument and
glues its string argument into the `NP`.


<h2 id="iv-trading-size-for-amount">IV. Trading size for amount: a case study in Basque</h2>

Here's a real-life case study, where first I had few concrete
categories but they were huge (verb inflection tables). I replaced
those with many concrete categories that were small, and it improved
the performance a lot.

If you're reading this blog, chances are high that
you know a thing or two about Basque verbs, but in case no, here's a
recap:
* Agrees with subject, direct object (if it has one) and indirect object (if it has one), in
all tenses and moods.
* Most verbs are inflected with an auxiliary; so think
"*intransitive-do* sleeping", "*transitive-do* eating",
"*ditransitive-do* talking" and so on. The "✱-*do*" part inflects in
hundreds of forms (or thousands, if we get liberal with clitics and
such), but the "✱ing" part only inflects in 3 forms.
* A handful of verbs have a full inflection; in the Basque linguistics
  jargon, these are called synthetic verbs.

What are the implications for GF grammars? First of all, the *size* of
the verb categories can get really big, if all verbs have all the
forms in an inflection table.  But since most of the verbs share the inflecting part with
other verbs in the same valency, putting it all in inflection tables
sounds like a waste.

So in all of the verb categories in the Basque RG, there are no [finite](https://en.wikipedia.org/wiki/Finite_verb) verb forms:
<!--([link](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/basque/ResEus.gf#L296-L299)):-->

```haskell
Verb : Type = {
  prc : Tense => Str ; -- 3 forms
  nstem : Str ;
  aux : AuxType -- <20 different values
  } ;
```

The participle in the `prc` field has only 3 forms: past, progressive
and future. In the lexical categories, we only have the
[AuxType](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/basque/ParamEus.gf#L9-L20)
param, which has around 10 different values so far: the three
auxiliaries, and the handful of synthetic (i.e. fully inflecting)
verbs.  The parameter just tells us which auxiliary or synthetic verb
to choose for the verb. Furthermore, this `Verb` is the lincat of *every
single subcategory* of verbs in the RG!

Here's an example--the words `Etorri`, `Ibili` and `Jakin` are values
of the parameter `AuxType`:

```haskell
lin
  -- Ordinary verb, using the transitive auxiliary
  buy_V2 = mkV2 "erosi" ;

  -- Some synthetic verbs
  come_V = syntVerb1 "etorri" Etorri ; -- intrans.
  walk_V = syntVerb1 "ibili" Ibili ;   -- intrans.

  know_V2 = syntVerb2 "jakin" Jakin ;  -- trans.
```

As the `VP` keeps forming, we keep track of the direct and indirect
object in corresponding fields--still no finite verb forms, just the
parameter that was there since the beginning.  Finally, we introduce
the actual verb inflection in `PredVP`. It's only at that point we
know all the arguments (subject, direct object and indirect object),
so we can choose the agreement at once<sup>[3](#basque-details)</sup>.<a name="fn-3"></a>

I might write more about this, if I ever write a paper about the
Basque RG. But now let's recap what we did for verbs:

* First design: no inherent parameters, 200+-form inflection tables. Compiled in minutes--hours.
* Second design: ~10 inherent parameters, 3-form inflection tables. Compiles in <30 seconds.

I wonder if this scales up when I add the remaining synthetic
verbs. There should be around 20 parameters for a complete grammar,
rather than 10, but I haven't gotten around adding them; we don't need
more in the small resource lexicon. It is possible that this approach
doesn't scale up, but at least is has scaled up better than the
original version with inflection tables.

Of course, not all languages support this kind of thing--but maybe you
can find other redundancies to exploit in yours. The take-home message
here was that "you don't need to put verb inflection forms in the
*verb* category, but somewhere higher up". Maybe write a RG or two,
and by your third, you'll get creative! ^_^

## V. Quick fix for testing an oversized grammar

The part that takes time is the conversion to PMCFG--that's what is
needed for parsing.  However, it is also possible to use a GF grammar
such that it only linearises.  Unfortunately, as of December 2018,
this is possible only in the GF shell, i.e. it is not supported in any
external application.

But if you want to test a slow grammar just for linearising, you can do as follows.

![nopmcfg](/images/no-pmcfg.png "Importing a grammar without compiling it to PMCFG")

In words:

* Import your grammar with flags [`-retain`](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc42) and `-no-pmcfg`.
  * `-no-pmcfg`, as the name suggests, imports the grammar without creating a PMCFG.
  * `-retain` imports the grammar but leaves all `oper` definitions available, so you can also test opers and not just lins, if you want to.
* To test a linearisation, use the command `cc <syntax tree>`. In the screenshot, I have used the option `-list` to `cc`; you can see all options when you type `help cc` in a gf shell.


```
> help cc
cc, compute_concrete
computes concrete syntax term using a source grammar

syntax:
  cc (-all | -table | -unqual)? TERM

Compute TERM by concrete syntax definitions. Uses the topmost
module (the last one imported) to resolve constant names.
N.B.1 You need the flag -retain when importing the grammar, if you want
the definitions to be retained after compilation.
N.B.2 The resulting term is not a tree in the sense of abstract syntax
and hence not a valid input to a Tree-expecting command.
This command must be a line of its own, and thus cannot be a part
of a pipe.

options:
 -all	pick all strings (forms and variants) from records and tables
 -list	all strings, comma-separated on one line
 -one	pick the first strings, if there is any, from records and tables
 -table	show all strings labelled by parameters
 -unqual	hide qualifying module names
 -trace	trace computations
```

## Read more

[Krasimir's thesis](http://www.cse.chalmers.se/~krasimir/phd-thesis.pdf) Section 2.8.4 "Hints for Efficient Grammars" is a good read. It is found on page 67 (page number), which is page 79 in the PDF.

## Footnotes

1.<a name="belgium-reform"> </a>Let's hope that the French language goes through this [grammar reform](https://www.liberation.fr/debats/2018/09/02/les-crepes-que-j-ai-mange-un-nouvel-accord-pour-le-participe-passe_1676135), then we have some hope of GF grammars running faster! <a href="#fn-1">↩</a>

2.<a name="verbosity"> </a>Compiling with `-v=3`, you get this kind of output:

```
  generating PMCFG
+ AdVVP 1280 (1280,1)
+ AdVVPSlash 69120 (69120,1)
+ AdvVP 1280 (1280,1)
+ AdvVPSlash 69120 (69120,1)
+ CompAP 8 (2,2)
+ CompAdv 1 (1,1)
+ CompCN 8 (2,2)
+ CompNP 32 (1,1)
+ ComplSlash 2211840 (25600,21)
```

If you're developing a resource grammar, you can try playing around with the parameters and see if you get a difference in the numbers. <a href="#fn-2">↩</a>


3.<a name="basque-details"> </a>I know this is quite abstract, so [here's some
code](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/basque/ResEus.gf#L529-L544). The
actual verbs live in the weird-sounding identifiers like
[AditzTrinkoak](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/basque/AditzTrinkoak.gf#L255-L295).`ukanZaio`,
and `chooseAuxPol` picks the right forms (the right forms also depend
on if the sentence is negative or positive, and that comes one step
later, from `UseCl`). <a href="#fn-3">↩</a>
