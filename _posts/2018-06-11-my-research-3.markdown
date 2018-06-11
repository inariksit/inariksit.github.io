---
layout: post
title:  "My research, part III"
date:   2018-06-11
categories: research
tags: research
---


* [Part I](https://inariksit.github.io/research/2018/06/06/my-research-1.html)
* [Part II](https://inariksit.github.io/research/2018/06/07/my-research-2.html)

So now you know the two different formalisms I've worked with. Here
I'm rambling a bit about various connections between them, or really
anything that pops up in my mind.


# Automatic

The GF grammarian (or an informant) had to come up with phenomena to
test, and trees that capture those phenomena. Now both steps are fully
automated: the grammarian only needs to input a function or category,
and the system outputs a minimal and representative set of
sentences. Humans are still needed to read and evaluate the example
sentences, as well as write the grammars in the first place.
I am an active user of this system, and so happy of its
existence that I don't really need a degree to be happier. There are
even external users who are not me, which after 30 years of existing
and 10+ years of programming feels pretty amazing. 

On the CG side, I have also automated a task, but I'm not sure if it's
a task people were doing widely. Sure, CG grammarians evaluate their
grammars---I've seen in practice people applying their rules to a
corpus, and reading through output and judging if it's fine. But the
kind of systematic, "do some of these rules prevent others from
applying" seems to not have been a common activity. Automatic
reordering has been a thing, but with a black-box machine learning
rather than formal methods such as SAT-solvers.

What I'm saying is that my CG tools haven't had any adoption. In a
sense this is not surprising---I was never a CG writer myself, and the
starting point was honestly just "SAT is cool, let's find a
problem". The SAT-encoding itself works as a parallel CG engine,
meaning that it would complain even well-ordered sequences of rules,
if they "contradict" each other. Our first version was not of much
practical use, but I could make smug remarks like quoting Fred
Karlsson (1995, page 11):

> More generally, the formalism should be such that individual constraints are
> unordered and maximally independent of each other. These properties facilitate
> incremental adding of new constraints to the grammar, especially to a large one.

I don't know if anyone had bothered to actually try to execute a full
existing CG grammar in parallel after this was
written. But I made that remark already in my licentiate thesis, so
let's move on.

So, the initial parallel SAT-encoding was not much use neither for
writing new CG rules nor executing or testing old CG rules. Thus we
made a second one, with added state. This turned out to be more
useful: the conflicts it reports are *actually conflicts* in an
actually execution order. I'm still hopeful that people may have liked
it if I had made it easier for them---the tool is written in Haskell
(which made the only potential user who tried to install it to joke
about the [relevant xkcd](https://xkcd.com/1312/)), and it's not
integrated to any existing workflow.

# Systematic

I don't know about you but "automatic systematic" just makes me think
of [this](https://www.youtube.com/watch?v=MNyG-xu-7SQ).

Anyway, let's start from CG now because it's easier to reason about.  A
CG grammar can be "good" or "bad" according to purely internal
criteria. Good means just that the rules are internally consistent,
and bad the opposite of that. The point of symbolic evaluation is to
explore all possibilities: if there was a way in which some rule would
apply (e.g. make another rule not apply and thus preserving a reading;
or create a solution in which a particular reading was already false),
the SAT-solver would have found it. You know, that's just how
SAT-solvers work. So we can be confident that if the system reports no
conflicts, there are no conflicts---"what if we just run a few more
tests" doesn't make sense to worry about.

It is a separate question (not my research question!) whether the CG
grammar also does a good job at disambiguating natural language. Such
questions are better left for corpus-based methods.

On the GF side, we actually are trying to answer the harder question:
does this grammar do an adequate job at producing English/French/…?
For that, we need a set of test sentences that covers all phenomena in
the grammar.  In the CG world, we had a natural restriction: there is
only a few hundreds or thousands of different tags in the
morphological lexicon, and there are restrictions (in a lucky case
also well-documented and not like 3 different Word documents that are
deprecated in a different way) how they mix together into readings. So the
language of the tags and readings is finite---sounds reasonable that we can
do the job thoroughly.

But natural language is infinite. How does that work with a finite set
of test cases?

First of all, we are testing a PMCFG representation of natural
language. This representation is also infinite: for example, the
grammar can parse and produce a sentence with [arbitrarily many
relative clauses](https://inariksit.github.io/gf/2018/06/10/completeness.html)
(*I know that you think that she disagrees that … this example is
silly*), only limited by physical constraints such as memory or having
a source of power after all humans die. But importantly, the grammar
still has a finite number of *functions*. Let's see, can we test one
function with finite test cases?

Take any GF function `f : A -> B -> C`. All categories `A`, `B` and
`C` may contain fields with strings, tables and parameters.  Recall
that in the PMCFG, first all categories `A`, `B` and `C` are compiled
into several concrete categories, one for each combination of
parameters. Then, the function `f` is compiled into several concrete
functions, one for each combination of concrete categories. This is
still finite, and just makes our job easier! Now we can ask: can we
test one *concrete* function with finite test cases?

Let's take a bit more concrete example: `Mod : Quality -> Kind ->
Kind` from Foods grammar, Spanish concrete syntax. It compiles into 4
concrete functions:

```haskell
Mod_pre_fem   : Quality_pre  -> Kind_fem  -> Kind_fem
Mod_post_fem  : Quality_post -> Kind_fem  -> Kind_fem
Mod_pre_masc  : Quality_pre  -> Kind_masc -> Kind_masc
Mod_post_masc : Quality_post -> Kind_masc -> Kind_masc
```

`Mod_pre_fem` takes a feminine `Kind` and a premodifier `Quality`. As
you can guess by the name (just kidding, in the PMCFG it's like
`Kind_3493` and `Quality_6270` so you can't guess anything), it
chooses the feminine forms (singular and plural) of the `Quality`
argument, and places them before the singular and plural forms of
`Kind` argument. If this function fails, we can totally scold it by
saying "you had one job".

So what inputs does this function need? Duh, one premodifier `Quality`
and one feminine `Kind`. At this level, all we need to do is to choose
*maximally representative* premodifier and feminine arguments. In
other words, we prefer words with maximally unique<sup><a name="footnote"
href="#max-unique">1</a></sup> inflection tables. If we find such
arguments, then `Mod_pre_fem` should also construct an inflection
table with unique strings. Or if it doesn't, at least we know that the
repetition is due to `Mod_pre_fem` and not its arguments.

Okay, now we agree that the *arguments* to the functions are finite.
Now it's just contexts left. In general, a test case needs as many
contexts as it has strings---certainly a finite amount.
In order to be representative, we need to make sure that the contexts
don't introduce the same words that appear in the test cases: if the
test case is *likes the dog* and the context happens to include *the
dog* as a subject, we cannot be certain which *dog* comes from the
test case and which one from the context.

Summary of what I just said:

"How can we test a whole big infinite grammar?"  
Easy: there's a finite amount of functions, just test each function separately.

"How can we be sure to use one function in all the different ways?"  
No problem: split it into several functions that can all be used in only
one way, then test each of them.

Summary for those who like their summaries with bullet points:

* Finite categories in GF grammar
  * Finite concrete categories in PMCFG
* Finite functions in GF grammar
  * Finite concrete functions in PMCFG
  * Finite arguments to the concrete functions → finite test cases
    * (Combinations of) arguments with unique strings
  * Finite contexts for the test cases
    * Unique words in the contexts
  


# Failure modes

Now let's look at how we can still mess things up.

A given CG grammar is internally consistent *given a
certain tagset, rules to combine tags into readings, and ambiguity
classes*. All of this information comes from a morphological
lexicon, and we all know that morphological lexica are generated at
airports at 04:00 after the conference dinner, because that was the
cheapest flight and you didn't have to pay for one more night of
accommodation, and the free drinks would keep you up and cheerful
until your flight departs.

So, CG analysis is as reliable as is the morphological lexicon where
the tags and the readings come from. How about GF test cases?

I just tried to convince you how the system produces a test case for
each combination of parameters. But the failure mode here is, simply,
*not enough parameters*. Consider the `Mod` function again, and say
that we don't have the parameter for pre- and postmodifier at all,
and `Mod` only comes in two variants, one for feminine and one for
masculine. Suppose that all adjectives are, incorrectly,
postmodifiers. Then we would notice the error only if the single `Quality`
argument for `Mod_fem` and `Mod_masc` happened to be a premodifier
one.

If the missing parameter handles a common phenomenon, it is more
likely that we catch the error even with missing parameters. This was
the case with, for example, Dutch preposition contraction (which I've
explained in Chapter 4 of the thesis). Originally, none of the
prepositions contracted, but this is the most common behaviour---only a
handful of prepositions *don't* contract. Thus we got some test cases
which showed the error, and went on to fix the grammar. After the
fixes, the system created separate test cases for contracting and
non-contracting prepositions.

# Moral of the story

Test your grammars! I don't know. I'll stop this now anyway,
it's pretty long and I have more to blog! Next up, Koen's Fancy
Fixpoints and/or other interesting statistics we can gather from GF
grammars:

* Which fields are always the same (i.e. is there a parameter
that doesn't make a difference?)
* Are some fields always empty?
* Are some fields or categories unreachable from another category?
* Are some subtrees erased by some function?

Stay tuned!

****

<a name="max-unique" href="#footnote">1</a>: *I, me, my* is a better
example than either *you, you, your* or *she, her, her* alone, because
*I* only has unique strings, and *you* and *she* repeat some strings.
But together *you* and *she* cover everything that *I* does alone.
