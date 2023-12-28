---
layout: post
title:  "Derivational morphology in GF"
date:   2023-12-27
categories: gf
tags: "gf"
---

This happens to me regularly:
- I introduce GF to a new person
- They start playing around with the RGL
- They linearize `write_V2` and expect to find the string *book* among the forms in the inflection table, but it is not there
- They ask me why, and I finally wrote this blog post so I can link to it and explain!

This example comes up all the time with Arabic (كَتَبَ *kataba* 'write' / كِتَاب *kitāb* 'book'). But there are other expectations with other languages: maybe they expect *kill* from `die_V`, or *dormisco* 'fall asleep' from `dormio_V` 'sleep' (Latin).

In English, these particular examples don't make sense because the words look different, but consider *book* and *bookish*—clearly the latter comes from the former by adding an affix. Morphologically the process is just like making *killed* from *kill*—add an affix and you get a different string with a different but related meaning. So why is *bookish* not in `book_N` but *killed* is in `kill_V2`?

The short answer is as follows:

> A single lexical entry only contains *inflectional* morphology, such as the singular and plural forms for nouns, or past and present forms for verbs. In contrast, the process to make "bookish" from "book" is not inflection of the noun, but *derivation* from a noun to an adjective, and those words belong to separate lexical entries.

In most RGL languages with rich morphology, adding derivation carelessly would make lexical entries large and slow down the grammar. But maybe that will be fixed once Krasimir is done with the ✨ majestic runtime ✨, and then the question becomes philosophical again. So the long answer is the rest of this blog post.

- [Inflection vs. derivation](#inflection-vs-derivation)
- [Operational definition in terms of GF RGL](#operational-definition-in-terms-of-gf-rgl)
  - [English](#english)
  - [Latin](#latin)
  - [Hungarian](#hungarian)
  - [Derivations to same part of speech](#derivations-to-same-part-of-speech)
  - [Derivations to different part of speech](#derivations-to-different-part-of-speech)
- [Derivations as opers in resource modules](#derivations-as-opers-in-resource-modules)
  - [Paradigms module for a hypothetical language](#paradigms-module-for-a-hypothetical-language)
  - [Example for language learning application](#example-for-language-learning-application)
- [Footnotes](#footnotes)


## Inflection vs. derivation

There are lots of explanations about inflection and derivation in introductory texts on linguistics, you can read about it e.g. [here](https://www.languagetools.info/grammarpedia/derivinfl.htm).

TL;DR is that derivation creates new words, whereas inflection creates forms of the same word.

The question "what is a word" is one of those where the more you know about linguistics, the harder it becomes to answer. But I argue that we can, in fact, give a precise definition in the narrow context of the GF RGL. My argument is the following:

> A single lexical entry needs **all forms** that make relevant distinctions in **all different contexts** where it can end up in the grammar.

In the next section, I explain in detail what that means.

## Operational definition in terms of GF RGL

Consider the following inflection table for an English noun.

```haskell
$ gf alltenses/LexiconEng.gfo
> l -table child_N
s Sg Nom : child      s Pl Nom : children
s Sg Gen : child's    s Pl Gen : children's
```

### English

These forms are in the inflection table of `child_N`, because there are constructors in the RGL that access all those forms. Here's an example of such trees that use all 4 forms:

```haskell
$ gf alltenses/AllEng.gfo
> l PredVP (DetCN (DetQuant DefArt NumSg) (UseN child_N)) (UseV walk_V)
the child walks           -- uses Sg Nom

> l PredVP (DetCN (DetQuant DefArt NumPl) (UseN child_N)) (UseV walk_V)
the children walk         -- uses Pl Nom

> l PredVP (DetCN (DetQuant (GenNP (DetCN (DetQuant DefArt NumSg) (UseN child_N))) NumSg) (UseN dog_N)) (UseV walk_V)
the child's dog walks     -- uses Sg Gen

> l PredVP (DetCN (DetQuant (GenNP (DetCN (DetQuant DefArt NumPl) (UseN child_N))) NumSg) (UseN dog_N)) (UseV walk_V)
the children's dog walks  -- uses Pl Gen
```

This is the complete list of morphological forms that are needed to make all trees that use `child_N` grammatical. There are many other constructions in the RGL, but all of them—*in the English resource grammar*—use one of those 4 forms.

<!--

```haskell
> l PredVP (DetCN (DetQuant DefArt NumSg) (PossNP (UseN dog_N) (DetCN (DetQuant DefArt NumSg) (UseN child_N)))) (UseV walk_V)
the dog of the child walks
```
-->

### Latin

In contrast, let's look at a language with a larger inflection table. Here's the same lexical entry for Latin.

```haskell
$ gf alltenses/LexiconLat.gfo
> l -table child_N
s Sg Nom : puer       s Pl Nom : pueri
s Sg Acc : puerum     s Pl Acc : pueros
s Sg Gen : pueri      s Pl Gen : puerorum
s Sg Dat : puero      s Pl Dat : pueris
s Sg Abl : puero      s Pl Abl : pueris
s Sg Voc : puer       s Pl Voc : pueri
```

In order to produce grammatical Latin, the RGL constructions require more distinct forms in the Latin grammar than in the English one. For example, consider the subject and the object cases in transitive sentences:

```haskell
$ gf alltenses/LangEng.gfo alltenses/LangLat.gfo
> p "I saw the child" | l -lang=Lat
puerum videbam

> p "the child saw me" | l -lang=Lat
puer me videbat
```

### Hungarian

In yet another RGL language, the inflection table contains forms that are traditionally classified as clitics, not inflection per se.
This is the case for Hungarian possessive suffixes:

```haskell
$ gf alltenses/LangEng.gfo alltenses/LangLat.gfo alltenses/LangHun.gfo
> p "my child" | l -bind -lang=Lat,Hun
puer meus
én gyerekem

> p "your child" | l -bind
puer tuus    -- singular you
te gyereked

puer vester  -- plural you
ti gyereketök
```

Even though clitics are not traditionally considered as inflection, it makes sense for those forms to be accessible from the inflection table.[^1]
That's because functions in GF may not inspect nor manipulate the tokens in its arguments' inflection tables, but only rearrange them. (For a longer explanation on restrictions at runtime, see [the gotchas post](../../08/28/gf-gotchas.html#unsupported-token-gluing).) So in the RGL, a lexical entry must contain every form that all syntactic functions may need later on. Sometimes it's just inflection, sometimes it's also clitics.

With this definition, we can answer the question "what should go in an inflection table" fully in terms of the GF RGL.
If there is a function in the RGL that requires a string commonly considered as derivation, then that string should be in the inflection table.

Let's consider the two cases of derivation: into same part of speech (*tidy*→*untidy*) and into a different part of speech (*thought*→*thoughtful*).

### Derivations to same part of speech

Derivations that keep the part of speech are often semantic in nature. In English, a common example is the prefix *un-*, which creates an opposite.

On the other hand, comparative and superlative are also semantic in nature, and those forms are in the same inflection table.

```haskell
> l PositA warm_A
warm

> l UseComparA warm_A
warmer

> l AdjOrd (OrdSuperl warm_A)
warmest
```

So if there was an RGL function called `OppositeA : A -> AP`, then we would need the derived form *unX* for all adjectives.

```haskell
-- NB. Not an actual RGL function
> l OppositeA warm_A
unwarm
```

But the common core of the RGL doesn't have such a function, so the typical RGL language doesn't include such forms in its adjective tables. Maybe such a construction is less common across languages, or makes sense for a subset of words. In general, the RGL doesn't contain anything too semantic in its common core, and that's a feature, not a bug.

However, some grammarian might want to add such forms in their grammar in the *concrete syntax* as internal params, and then add a function such as `OppositeA : A -> AP` in the language-specific Extra module. (More on RGL extensions [here](../../../2021/02/15/rgl-api-core-extensions.html).)
I think that is all fine, just consider the following:
1. If large inflection tables make the grammar slow, the grammar should prioritize the ones that are necessary to make all core RGL functions produce grammatical output.
2. If the grammar is planned to be used for morphological analysis, would you prefer *untidy* to be considered as a form of *tidy*, or an entry of its own? What about cases where the meanings have diverged more widely? For example, English *unclean* has significantly different (e.g. religious) connotations from *dirty*.
3. If the grammar is meant to generate language, consider whether this derivation is truly productive for all words of the given part of speech. For example, outside of Orwell's 1984, we don't normally see \*ungood.

### Derivations to different part of speech

This is actually already happening in many RGL languages, to some extent. For instance, the Arabic resource grammar does contain a verbal noun (مَصْدَر *masdar*[^2]), but not a full inflection of it. It wasn't there in the original implementation, but I added that as a single field in 2018, because a particular application grammar needed it.

There are already several functions in the RGL (mostly in the Extend module) that change the part of speech of a phrasal category. These all come from VP:

```haskell
PastPartAP  : VPSlash -> AP ;  -- lost (opportunity) ; (opportunity) lost in space
PresPartAP  : VP -> AP ;       -- (the man) looking at Mary
GerundCN    : VP -> CN ;       -- publishing of the document
```

In English, they all use forms that are traditionally considered part of the verb conjugation. *Lost* is a conjugated verb form, that is used for **predication** in the context "I *have lost* my keys", and **modification** in the context "my *lost* keys". The same holds for the gerund form: "the dog *is running*" (predication) and "the *running* dog" (modification). So let's recap:

1. In the GF RGL, the *syntactic-semantic function* of predication is implemented by the *category* `VP`, and the *syntactic-semantic function* of modification is implemented by the *category* `AP`.
2. In English, the verb form used for modification happens to coincide with a verb form already used for predication.

Thanks to these two facts, we can implement `VP* -> AP` functions for English without adding any new forms in verbs' inflection tables.

But another RGL language might use a completely different grammatical strategy for these syntactic-semantic functions. Then, if one wants to use such a language with the same API as English, that language needs to include in its inflection tables forms that are traditionally considered derivation, not inflection. If that doesn't blow up the grammar due to large inflection tables, I believe this is a good solution.
If it does blow up the grammar, then I would suggest not implementing the same API that works for English, but rather finding another design and implementing it in a language-specific Extra module.

## Derivations as opers in resource modules

I hope that the previous section has cleared up some definitions, and what matters in a GF grammar. It's not about following tradition blindly—nobody is aesthetically offended by mixing inflection and derivation, it's mostly about limiting the size of the inflection tables, and keeping the core RGL generic for as many languages as possible.

In contrast, there are no limitations or considerations in what kind of `oper`s to write in resource modules.
The Paradigms modules are 100% language dependent, and they may include anything that is convenient for a particular language.
So this would be a great design for some language that has all these features.

### Paradigms module for a hypothetical language

```haskell
-- basic verb constructor(s)
mkV : (_,…,_ : Str) -> V ;

-- V -> V derivations
habitualV : V -> V ;   -- to wash habitually
causativeV : V -> V ;  -- to make someone wash
reflexiveV : V -> V ;  -- to wash oneself

-- V -> N derivations
doerN : V -> N ;   -- to farm → farmer
actionN : V -> N ; -- to farm → farming

-- verbing weirds language
verbifyNounV : N -> V ;
verbifyAdjV : A -> V ;
```

These constructors would just be convenience for the grammarian, and not show up at any abstract and language-independent representation.

On the other hand, if one wanted to use derivations for an application such as language learning, one could, in fact, port these target language opers to the source language. If the source language doesn't have corresponding morphological constructions, then their linearizations could be just glosses.

### Example for language learning application

So let's do this for Finnish and English: you're an English speaker who wants to learn about derivations in Finnish. The first option is to *not use* the Finnish-specific derivation opers for English, but just have everything at a lexical level, as follows:

```haskell
-- Fin                         --  Eng
see_V  = mkV "nähdä" ;        | mkV "see" ;
eat_V  = mkV "syödä" ;        | mkV "eat" ;
feed_V = causativeV eat_V ;   | mkV "feed" ;
show_V = causativeV see_V ;   | mkV "show" ;
```

Or we could emphasize the aspect how these words are related to each other, and create a constructor `causativeV` for English as follows:

```haskell
-- same type signature for Fin and Eng
oper causativeV : V -> V ;

-- For English
oper causativeV v = make_V ** {
  s = \\x => make_V.s ! x ++ v.s ! VVF VInf
  } ; -- "make eat", "make see"

-- For Finnish
oper causativeV v =
{- Actually derive the causative verb from argument
    "syödä" → "syöttää"
    "nähdä" → "nãyttää"
can be applied recursively: näyttää → näytättää → näytätyttää → …
warning: stops making sense semantically at some point -}
```

Now we could actually use the Finnish-specific opers for English:

```haskell
-- Fin                      --  Eng
see_V  = mkV "nähdä" ;      | mkV "see" ;
eat_V  = mkV "syödä" ;      | mkV "eat" ;
feed_V = causativeV eat_V ; | causativeV eat_V ;
show_V = causativeV see_V ; | causativeV see_V ;
```

And now the linearizations for `feed_V` and `show_V` for English become "make eat" and "make see".

These `oper`s construct *new lexical items*, so no changes are required in lincats. They just might be useful for some application.

<!-- ### Derivations in an application grammar


Many application grammars already use a custom lincat to achieve this. For example, `lincat Action = {verb : V2 ; noun : N2}` makes it possible to create both "apply (the function)" and "application (of the function)". -->

## Footnotes


[^1]: The actual internal details of the Hungarian RG are not relevant here, but the 12 possessed forms (2x number of noun, 2x number of possessor, 3x person of possessor) are formed out of 4 stems in the inflection table. The 4 forms are chosen so that the possessive pronouns need to only store the simplest possible forms for the suffixes, and not have to deal with vowel harmony. Maybe another design could've cut the number of stems to an even smaller number, but such optimizations come at the cost of making the code harder to understand. I think 4 instead of 12 is a good compromise. <br/>If you are interested in the inflection table, you can open `alltenses/LangHun.gfo` in the GF shell and linearize `child_N` to inspect it yourself.


[^2]: Meaning 'verbal noun', which is itself a verbal noun of صَدَرَ [*sadara*](https://en.wiktionary.org/wiki/%D8%B5%D8%AF%D8%B1#Arabic).