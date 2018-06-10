---
layout: post
title:  "Answer to Reviewer #2 on completeness of gftest"
date:   2018-06-10
categories: gf
tags: research gf
---

Reviewer #2 from [NLCS](http://www.indiana.edu/~iulg/nlcs.html)
raised the following point about long-distance dependencies. There was no space to address that in the camera-ready, so I'm blogging about it instead. I'll add some condensed version in the actual thesis, this is just a brain dump.

Here's the point:

> I do not think that this agreement can be tested by considering finitely many examples. 
> Only a reasoning by induction may prove that the grammar works properly. 
> So,  I think enumerating the possible cases is sometimes impossible.

Consider long distance agreement in e.g. French. It's possible to put
arbitrarily many nested clauses between a verb and its object, e.g.
"Which books do you think that Mary believes that John knows that
Betty has read". The participle *read* is masculine and plural,
i.e. agrees with *which books*. (A curious reader may try [online
resources](https://translate.google.com/#en/fr/Which%20books%20Betty%20has%20read%3F%0AWhich%20books%20do%20you%20think%20that%20Mary%20believes%20that%20John%20knows%20that%20Betty%20has%20read%3F)
to see how well data-driven methods handle long-distance agreement.)

As a grammarian, I could show you the code that handles such cases.
Let's just handwave it here with a weird pseudo-categorial-grammar
syntax that confuses both GF and catgram people:

*Betty has read* is an `S\NP`. The `S` part means that it's a sentence:
 we've got a subject and a verb, and the `\NP` part means that we're
 just missing a noun phrase as an object. *Which books* is an `NP`,
 which is just what we need to complete the `S\NP`.

Before we actually combine these two phrases, we can enjoy the
infinite recursive properties of natural language. Recall that *Betty
has read* has the type `S\NP`--and so has *you think that Mary
believes that John knows that Betty has read*. There is a chain of
functions that transform the original *Betty has read* into this
complex phrase, still preserving its type.

Let's generalise a bit: take any function `ComplX : X\Y -> Y -> X`,
which completes an `X\Y` and a `Y` into an `X`. Assume that both
`X\Y` and `Y` are vectors of strings, with some additional
parameters. Some parameters in `Y` specify which string(s) to choose
from `X\Y`, and other parameters in `X\Y` specify which string(s) to
choose from `Y`. The function `ComplX` puts together these strings
into a final `X`, which may also be another vector of strings with
parameters of its own.

Now, it doesn't matter which particular strings there are in the
arguments of `ComplX`, as long as we test a variety of
parameters. Instead of adding complexity by nested clauses, we need to
test with arguments that change some parameters: *we* instead of *Betty*
to inflect *have* differently; *Which house* (singular feminine) to
inflect *read* differently.

You may not be convinced yet. Who knows if the transformation
function, while adding *John*, *Mary* and whatnot, actually messed up
and picked an inflection for *has read* without consulting *Which books*?
Wouldn't you want to have a couple of nested examples thrown in to assure you
that it works?

First of all, if the transformation function is supposed to keep the
type of `S\NP` intact, it cannot choose an inflection for *has read*
without changing the type. The original *Betty has read* was a
vector with 4 strings:

* Betty has read`.sg.fem`
* Betty has read`.sg.masc`
* Betty has read`.pl.fem`
* Betty has read`.pl.masc`

And only a `ComplS` function may choose one of these strings. In contrast,
a transformation function (call it `Nest`) can only add its arguments to
the `S\NP`, so that it's still a vector of 4 strings:

* you think that Mary believes that John knows that Betty has read`.sg.fem`
* …
* you think that … Betty has read`.pl.masc`

Alright, `Nest` could definitely be type-correct and still buggy--for example, it returns a vector of 4 strings, but every string would have the singular feminine version of *read*. Then no matter what object we get in `ComplS`, the verb *read* would always be singular feminine.

It's true that we wouldn't catch this bug if we're only testing `ComplS`. If `ComplS` works correctly, i.e. it chooses always the right inflection form of *read*, then the system will generate 4 different objects, for the combinations of `{sg,pl} x {fem,masc}`. The objects are minimal: prioritised by the number of unique forms (`[I,me,my,mine] > [you,you,your,yours]`), and secondarily by the size of the tree (`you > you and your dog`). We see that an `S\NP` formed without `Nest` is smaller, so as long a `ComplS` can find four maximally unique examples of an `S\NP` formed by other functions, it never chooses an example formed by `Nest`. This makes sense, because the tests are currently based on a single function, and here we are testing `ComplS`.

But we would totally catch the bug when testing `Nest`. The test cases need to be put in contexts of start category, and now `ComplS` with various objects forms part of the context. So there you would get the *which books*`.pl.masc` *you think Betty has read*`.sg.fem`.

To recap:
* `ComplS : S\NP -> NP -> S` is fine to test with minimal `NP`s to know whether it works.
* If there is a bug in `Nest : S\NP -> S\NP`, we find it when testing `Nest` itself, because the context generation will put it in a context where `ComplS` is called.
* The per-function design of test case generation may be redundant (sometimes a single set of test cases covers exhaustively several functions!) but it will certainly be a finite amount.

