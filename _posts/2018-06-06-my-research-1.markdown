---
layout: post
title:  "My research, part I"
date:   2018-06-06 10:49:00
categories: research
tags: research gf
---


During the past 5 years, my research has centered around grammars:
writing, testing and doing weird things with them. This is an attempt
at build some kind of narrative. There will be 3 parts, and the
purpose of this is just to have words written down in a non-pressuring
format which I can then copypaste into `thesis.tex` and make it nicer.


## Writing (GF) grammars

Already before starting my PhD, I was involved in the Grammatical
Framework (GF) community, writing and improving grammars in their
Resource Grammar Library (RGL). While the grammar writing itself was
never the main focus of my PhD, I've kept doing it on the side
throughout the 5 years.

In 2015, I started writing a Basque grammar completely from
scratch--initially it was a joint project with Francis Tyers, but
after the first months I took over. Basque is an isolate language,
without anything similar in the RGL, so there wasn't much to build on.
There was also the additional challenge of me not speaking
Basque--below A1 level in any practical things, with some grasp of
nominal and verbal morphology. I could have some chance of detecting
if the morphology was wrong sometimes, but with syntax I had to rely
on a grammar book and native speakers.

So how to strategically employ native speakers? You implement a
function and show them sentences. This was my workflow in summer 2016,
when I had 2 weeks to sit down with a native speaker.

******

Inari: "Today I'm doing relative clauses, can you help me a bit!"

I can piece together simple sentences, like "the boy drinks beer", but
I start out with no idea whatsoever how to form relative clauses.  So
I just give the boy and the beer sentence, and ask how to transform
that into a relative clause.

Inari: "How do you make this into a relative clause? *The boy who
drinks beer*?"

Informant: "Beer-absolutive drink-relative boy-absolutive."

Inari: "Okay, and how about *The beer that the boy drinks*?"

Informant: "Boy-ergative drink-relative beer-absolutive."

I write first version of the function, and generate 10 sentences,
which all use relative clauses.  I'm not expecting the function to be
perfect, but to prod the informant to come up with more examples: it's
easier to correct an error than to dig the perfect description of
Basque relative clauses from your head.

Informant: "What is this sentence supposed to be, *our your dog drink beer*?"

Inari: "It's supposed to be *our beer, which your dog drinks*."

Informant: "The word order should be: *[your dog drink] our beer*."

Inari: "Aha right, I see! And how about *these [your dog drink] beers*
(these beers, which your dog drinks)?"

Informant: "That's fine."

Inari: "And *these five [your dog drink] beers* (these five beers,
which your dog drinks)?"

Informant: "That's wrong too; should be *these [your dog drink] five beers*;
the number *five* should go at the end like *our*. But if you
have *These five yellow beers*, then it's right."

After some discussion, I add a distinction between "heavy" and "light"
modifiers: adjectives are light, relative clauses are heavy. In
addition, I split determiners into two classes: numbers and possessives
go after heavy modifiers and before light modifiers, the rest go always
before all modifiers.

I generate 10 more random sentences which use relative.

Informant: "The relative clauses are correct now. But verb X takes
partitive; and if you have partitive, you must put the verb in singular."

Inari: "Okay, thanks for the info! Quick fix, I'll force singular
agreement with partitive object."

I generate a new set of examples, which all use partitive.

Informant: "Actually, personal pronouns don't have partitive; and
also, now I remember that negation changes the word order, can you
generate some negative sentences?"

*******

Oh but that sounds pretty smooth, so what's the problem here? It's
exactly the "generate 10 sentences" part: we're talking about nothing
more than the `gr` command in the GF shell. First I just start off
creating any kind of NP which uses `RelCN`, and then I get sentences
like *the blue small boy bigger than you whom they won't have
loved*. To combat that, I just type in an example sentence: *the beer
which the dog drinks*, copy the whole tree, and generate random
sentences with the template `? beer which ? dog drinks`. This is how
it looks in the actual tree:

```haskell
DetCN (DetQuant DefArt NumSg) (RelCN (UseN beer_N) (UseRCl (TTAnt
TPres ASimul) PPos (RelSlash IdRP (SlashVP (DetCN (DetQuant DefArt
NumSg) (UseN boy_N)) (SlashV2a drink_V2)))))
```

And I would manually change into question marks the parts where I want
variation:

```haskell
DetCN ? (RelCN (UseN beer_N) (UseRCl (TTAnt TPres ASimul) PPos (RelSlash
IdRP (SlashVP (DetCN ? (UseN dog_N)) (SlashV2a drink_V2)))))
```

This is better than 100 % random *the blue small boy bigger than you
whom they won't have loved* type of sentences, but both approaches
miss everything that is formed with a different constructor: for
instance `MassNP (RelCN (UseN beer_N) (UseRCl ...)`, which uses
`MassNP` instead of `DetCN`. Even in the trees that I do consider, I
need to know what exactly to replace.

Also, my informant happened to be a computational linguist, so it was
easy for her to suggest that I generate negative sentences, when I was
testing partitive. An ordinary mortal may have easily missed such a
detail, and then the test wouldn't have been as exhaustive.

So how have things improved in 2 years? 

## Testing (GF) grammars

We set out to develop an automatic and systematic method for testing
grammars. Earlier we relied on the informant to remember and explain
all important phenomena to test, and the grammarian to form all
relevant trees that show those phenomena. Now, we have a system that
takes a GF grammar, a concrete language and a list of functions and/or
categories. Then the system generates a representative and minimal set
of example sentences that showcases all important variation in the
given functions and categories.

Example 1: We test a single word, e.g. `she_Pron` in English. The test
program generates contexts that pick out all the different inflection
forms of the word: she, her and hers, and put them into context. For
instance, putting `she_Pron` in the category of `Cl` returns *she is*,
*there is her right* and *there is hers*. Putting `she_Pron` in the
category of `Adv` returns *more youngly than she*, *without her*, and
*without some year of hers*.

Example 2: We test a function with arguments, say `PrepNP : Prep -> NP
-> Adv` in English and Dutch. Now instead of just having a single word
with already existing fields, we need to generate good combinations of
`PrepNP`’s arguments, that is, `Prep`s and `NP`s. The program creates
1 test case for English, e.g. `PrepNP without_Prep (UsePron she_Pron)`
*without her*, and then puts it in a context, *there is something
without her*. Dutch has more complex phenomena going on, with
prepositions contracting in certain places, so the program needs to
generate 4 test cases to cover all different combinations.

(A note here: if the grammar itself has no semantic coherence, the
sentences will be nonsensical as well. Testing an application grammar
results in much nicer sentences.)

So, we give the English test case to an English speaker, and the 4
Dutch test cases to a Dutch speaker. If the tester finds one or all of
the sentences grammatically incorrect, we can narrow the bug down to
either `PrepNP` or some of its arguments. For instance, imagine we have
a bug in English, and the output is *✱there is something without she*:
either `she_Pron` doesn’t contain the accusative form her, or `PrepNP`
doesn’t choose it, but we don’t know which one yet. We can run some
more tests in the program, e.g. test `she_Pron` to see all of its forms
in appropriate contexts. When we have an idea where the bug is
located, we fix the bug(s) and then generate the test cases again with
the new grammar.

If all the test cases are grammatically correct, we can be fairly
confident that `PrepNP` works correctly. Furthermore, we can be
confident that we have seen all relevant distinctions—it would be
redundant to give the 4 Dutch trees linearised into English to the
English speaker, and not enough to give just the one English test case
in Dutch to the Dutch speaker.

To understand the details of how this works, you should read my
thesis!

To use the tool in practice, see
[instructions](https://github.com/GrammaticalFramework/GF/blob/master/src/tools/gftest/README.md).

[Part II of this series](https://inariksit.github.io/research/2018/06/07/my-research-2.html).