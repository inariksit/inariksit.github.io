---
layout: post
title:  "Generalising agreement"
date:   2020-01-01
categories: gf
tags: "gf linguistics"
---

Lately I've been writing blog posts I anticipate to be useful for someone else. Also lately my blogging speed is like two posts a year, and I hate every word I write. So I'm trying a new strategy: a 100% theoretical post where all I do is nitpick about definitions that have no practical impact on anything! (I think this means I'm ready to return to academia, after being a developer for 1.5 years.)

## Agreement in linguistics

So, today I'm talking about verbal agreement: one or more of the verb's arguments are marked in the verb form. If you're not familiar with other languages than Standard Average European, you've probably only seen subject agreement:

 &nbsp;          |1st person singular | 3rd person singular
-----------------|----------------|---------------------------
**Intransitive** |I sleep-**∅**       | the cat sleep**s**
**Transitive**   |I drink-**∅** water | the cat drink**s** water
-----------------|----------------|---------------------------

Both verbs, *sleep* and *drink*, only show marking for the subject/agent[^1]: null morpheme for *I*, and -s for *the cat*. *Water*, which is the object (or "patient") of *drink*, doesn't contribute to the conjugation. Contrast this with Hungarian [[Wikipedia](https://en.wikipedia.org/wiki/Hungarian_verbs#Definite_and_indefinite_conjugations)]:

<img src="/images/hungarian-agreement.png" alt="Screenshot of Wikipedia. The text reads: In Hungarian, verbs not only show agreement with their subjects but also carry information on the definiteness of their direct objects. This results in two types of conjugations: definite (used if there is a definite object) and indefinite (if there is no definite object" />

In fact, multiple agreement is pretty common in languages around the world. The map below is taken from [World Atlas of Language Structures](https://wals.info/feature/102A#1/17/149): red dot means "like English", yellow dot means (roughly) "like Hungarian".

<img src="/images/person-marking-wals.png" />

<!-- > Verbal person marking of just the A [agent] is most common in Eurasia. It is characteristic of most of the Indo-European languages of Europe, the Uralic languages of Russia, Finland, Estonia and Hungary, the Turkic languages, and the Dravidian languages of India. It is also found in eastern Africa, especially among the Afro-Asiatic languages, in the north of South America and in New Guinea. By contrast, it is extremely rare in North America and there is no instance of it among the languages in the sample in Australia. Verbal person marking of just the P [patient] is an infrequent phenomenon everywhere. -->

## Agreement in GF

*For simplicity, I'm only showing positive present indicative forms. I will address tense, aspect, mood etc. in later sections.*

### No person marking

The simplest person marking is no person marking. Let's start easy and do Chinese:

```haskell
lincat
  V,V2 = Str ;

lin
  sleep_V  = "sleep_in_chinese" ;

  drink_V2 = "drink_in_chinese" ;
```

### 1-dimensional person marking

Next possibility, the verb can mark one of its core arguments, agent or patient. Let's use English as an example of agent marking:

```haskell
param
  Agr = Sg3 | Other ;

lincat
  V,V2 = {s : Agr => Str} ;

lin
  sleep_V  = {s = table { Sg3   => "sleeps" ;
                          Other => "sleep" }} ;

  drink_V2 = {s = table { Sg3   => "drinks" ;
                          Other => "drink" }} ;
```

So far we have only created an inflection table---nothing in the code says that we have *agent* marking, just that we have one-dimensional table. But when we add lincats and linearisations for VP and Cl, it will become apparent which argument the verb marks. The following GF code models English (and other red dots on the WALS map):

#### English

```haskell
lincat
  VP = {s : Agr => Str} ;
  NP = {s : Str ; a : Agr} ;
  Cl = {s : Str} ;

lin
  -- UseV : V -> VP
  UseV v = v ;

  -- ComplV2 : V2 -> NP -> VP
  ComplV2 v2 np = {s = \\agr => v2 ! agr ++ np.s} ;

  -- PredVP : NP -> VP -> Cl
  PredVP np vp = {s = np.s ++ vp.s ! np.a} ;
```

Now, let's implement the missing categories and functions for a language that marks only patient, not agent.  According to WALS, this is quite rare but not unheard of; 24 languages of the sample of 378 do that.
To keep it simple, let's assume that there is a language just like English but patient marking.

|Transitive         |Sg1 subject | Sg3 subject  |
|:-----------------:|:----------:|:--------------------------:|
|**Sg1 object**   |I drink-**∅** me | the cat drink-**∅** me |
|**Sg3 object**   |I drink-**s** water | the cat drink-**s** water |
|-----------------|----------------|---------------------------|

<span title="If you want to shout E R G A T I V I T Y, be patient.">Now, the WALS data doesn't tell me what do these 24 languages mark in an intransitive verb, so I decide that in Like-English-but-patient-marking, intransitives have no person marking. (There are other options---more on those later!)</span>


| Intransitive    |Sg1 subject | Sg3 subject |
|:---------------:|:----------|:--------------------------|
| &nbsp;          | I sleep-**∅** | the cat sleep-**∅** |
|-----------------|----------------|---------------------------|

<!-- Let's ignore for now the concrete values of `Agr` and the strings in `drink_V` and `sleep_V`---only look at the categories and the syntactic functions.  -->


#### Like-English-but-patient-marking

```haskell
lincat
  V, VP = {s : Str} ;
  V2 = {s : Agr => Str} ;
  NP = {s : Str ; a : Agr} ;
  Cl = {s : Str} ;

lin
  -- UseV : V -> VP
  UseV v = v ;

  -- ComplV2 : V2 -> NP -> VP
  ComplV2 v2 np = {s = v2.s ! agr ++ np.s} ;

  -- PredVP : NP -> VP -> Cl
  PredVP np vp = {s = np.s ++ vp.s} ;
```

Let's look at the lexical categories, `V` and `V2`. The easy case first: `V2` is still an inflection table `Agr => Str`, only this time `Agr` marks the patient. In contrast, the lincat for `V` doesn't need a table, just a string!

This change is reflected in `VP`. It's clear when you look how to construct a VP: either from a `V`, which never inflected, or from a `V2` and a patient---just what is needed to pick out the correct form from the `V2`. In turn, `PredVP` is just trivial concatenation, because the subject doesn't contribute anything to the verb inflection.

<!--
In English, `VP` needs to retain its potential for agreement, and only `PredVP` can pick out a string, because that's when the subject is added. In Like-English-but-…, we get rid of the incertainty already at `ComplV2`, and adding subject is just trivial concatenation. -->

<!-- Compare `ComplV2` and `PredVP` in English and Like-English-but-patient-marking:

```haskell
-- English
ComplV2 v2 np = {s = \\agr => v2 ! agr ++ np.s} ;

-- Like-English-but-…
ComplV2 v2 np = {s = v2.s ! agr ++ np.s} ;
```

```haskell
-- English
PredVP np vp = {s = np.s ++ vp.s ! np.a} ;

-- Like-English-but-…
PredVP np vp = {s = np.s ++ vp.s} ;
```

Pretty cool right? -->

Have we exhausted the possible 1-dimensional inflection? Well of course not! Let's do something slightly less trivial, and introduce ergativity.

#### Ergative English



### 2-dimensional marking


Next, let's do Hungarian (within the sample shown previously, that is, ignoring the 2nd person object suffix):

```haskell
param
  Agent = Sg1 | Sg2 | Sg3 | Pl1 | Pl2 | Pl3 ;
  Object = DefObj | IndefObj ;

oper
  V  =           Agent => Str ;
  V2 = Object => Agent => Str ;

  sleep_V = table { Sg1 => "..." ;
                    ...
                    Pl3 => "..." } ;

  drink_V2 =
    table { DefObj   => table { Sg1 => "..." ;
                                ...
                                Pl3 => "..." } ;
            IndefObj => table { Sg1 => "..." ;
                                ...
                                Pl3 => "..." }
    } ;
```

And why stop at direct object? If you've read my blog before, you knew this was coming---Basque ditransitive auxiliary agrees with agent, patient and indirect object.

Before we get to the GF code, I need to introduce a new concept, namely, *ergativity*.



The patient agreement is restricted to singular vs. plural (in reality, not my simplification!), and I'm ignoring allocutive forms (don't google it now, I will introduce it when it's time). Within these restrictions, we can model Basque verbs as follows:

```haskell
param
  Agr = Sg1 | Sg2 | Sg3 | Pl1 | Pl2 | Pl3 ;
  ObjAgr = SgObj | PlObj ;

oper
  IntransVerb : Type =                  Agr => Str ;
  TransVerb   : Type =           Agr => Agr => Str ;
  DitransVerb : Type = ObjAgr => Agr => Agr => Str ;

  sleep : IntransVerb =
    table { Sg1 => "..." ;
            ...
            Pl3 => "..." } ;

  drink : TransVerb =
    table { Def   => table { Sg1 => "..." ;
                             ...
                             Pl3 => "..." } ;
            Indef => table { Sg1 => "..." ;
                             ...
                             Pl3 => "..." }
    } ;
```

## Read more

* The whole WALS article about person marking: [wals.info/chapter/102](https://wals.info/chapter/102)

## Footnotes

[^1]:
###  Subject vs. Agent
Terminology note: In a lot of linguistic literature, the term "<font color="green">subject</font>" refers to the only argument of an intransitive verb (such as *sleep*, *walk* etc.), whereas transitive verbs have an "<font color="b lue">agent</font>" and a "<font color="blue">patient</font>". Like this:

<ul>
<li><span title="SUBJECT"><font color="green">the cat</font></span>:S sleeps</li>
<li><span title="AGENT"><font color="purple">the cat</font></span>:A eats <span title="PATIENT (or OBJECT)"><font color="blue">the mouse</font></span>:P</li>
</ul>

How about the word "object"? Doesn't matter all that much: *agent* vs. *subject* is an important distinction regarding argument marking, but "object" and "patient" can be used interchangeably for the non-agent argument of a transitive verb.

[^2]: If you want to shout "ergativity" at me, please be patient---I'm trying to be pedagogical by introducing only one new concept at a time.
