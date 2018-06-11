---
layout: post
title:  "My research, part III"
date:   2018-06-11
categories: research
tags: research
---


[Part I](https://inariksit.github.io/research/2018/06/06/my-research-1.html)  
[Part II](https://inariksit.github.io/research/2018/06/07/my-research-2.html)

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
grammars--I've seen in practice people applying their rules to a
corpus, and reading through output and judging if it's fine. But the
kind of systematic, "do some of these rules prevent others from
applying" seems to not have been a common activity. Automatic
reordering has been a thing, but with a black-box machine learning
rather than formal methods such as SAT-solvers.

What I'm saying is that the CG tools haven't had any adoption. In a
sense this is not surprising--I was never a CG writer myself, and the
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

So, the intial parallel SAT-encoding was not much use neither for
writing new CG rules nor executing or testing old CG rules. Thus we
made a second one, with added state. This turned out to be more
useful--the conflicts it reports are *actually conflicts* in an
actually execution order. I'm still hopeful that people may have liked
it if I had made it easier for them--the tool is written in Haskell
(which made the only potential user who tried to install it to joke
about the [relevant xkcd](https://xkcd.com/1312/)), and it's not
integrated to any existing workflow.

# Systematic

I don't know about you but "automatic systematic" just makes me think of [this](https://www.youtube.com/watch?v=MNyG-xu-7SQ).

Anyway, let's start from CG now because it's easier to argue about.  A
CG grammar is a complete system of its own--a grammar can be "good" or "bad"
according to purely internal criteria. Good means just that the
rules are internally consistent, and bad the opposite of that. The
point of symbolic evaluation is to explore all possibilities--if there
was a way in which some rule would apply (e.g. make another rule not
apply and thus preserving a reading; or create a solution in which a
particular reading was already false), the SAT-solver would have found
it. So we can be confident that if the system reports no conflicts, there
are no conflicts--just "running a few more tests" is not a meaningful concept here.

It is a separate question (not my research question!) whether the CG
grammar also does a good job at disambiguating natural language. Such
questions are better left for corpus-based methods.

On the GF side, we actually are trying to answer the harder question:
does this grammar do an adequate job at producing English/French/…?
For that, we need a set of test sentences that covers all phenomena in the grammar.
In the CG world, we had a natural restriction: there is only a few hundreds or
thousands of different tags in the morphological lexicon, and there are restrictions
(in a lucky case also well-documented and not like 3 different Word documents that
are differently deprecated) how they mix together into readings. So the language of
the tags is finite--that is pretty reassuring that we can do the job thoroughly.

But natural language is infinite. How can we test it with finite set of examples?


minor note about completeness [here](https://inariksit.github.io/gf/2018/06/10/completeness.html)
