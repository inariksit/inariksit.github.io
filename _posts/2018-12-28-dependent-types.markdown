---
layout: post
title:  "Dependent types in GF"
date:   2018-12-28
categories: gf
---
*The first half of this post was originally included in [GF gotchas](/gf/2018/08/28/gf-gotchas.html).*
*If you've read the original post, feel free to jump directly to the [second half](#dependent-types)!*

You may have heard that GF is dependently typed, and wondered what
that means. (Or maybe this is the first time, and now you sure are
wondering what that means!) Perhaps you have looked at
[Coordination.gf](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/prelude/Coordination.gf)
and gotten confused about the weird functions that take *types* as
arguments.

## Types are of type `Type`

You may have seen things like this before:

```haskell
oper
  Verb : Type = {- s : Agr => String ; c : Case -} ;

  copula : Verb = {- … -} ;
```

This piece of code is telling you that `Verb` is a type, and `copula`
is a `Verb`. Or in a more verbose way: `Verb` is of type `Type`, and
`copula` is of type `Verb`. There is no reason why types cannot be *of
type* something!

`PType` is the same but for `param`s. Like this:

```haskell
param
  Case = Nom | Acc | ... ;
```

The values `Nom`, `Acc` etc. are of type `Case`, and `Case` is of type
`PType`. Check out
[here](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/prelude/Coordination.gf#L38-L39)
and
[here](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/english/ConjunctionEng.gf#L39)
to see some `PType` in action.

## Types can be arguments to functions

Now that we're granting basic human rights to types, look at this
piece of code:

```haskell
AdjCN adj cn = {
  s = if_then_else Str adj.isPre
        (adj.s ++ cn.s)
        (cn.s ++ adj.s)
  } ;
```

If the `if_then_else` bit looks strange, we can sugar it
into the following:

```haskell
if adj.isPre; then adj.s ++ cn.s; else cn.s ++ adj.s
```

So what is happening? In GF, there is no `if … then … else` keyword, just a function
`if_then_else` that takes 4 arguments, of which the first one is a
type. This is how it's defined in the
[GF prelude](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/prelude/Prelude.gf#L64-L69):

```haskell
oper
 if_then_else : (A : Type) -> Bool -> A -> A -> A = \_,c,d,e ->
   case c of {
     True => d ;
     False => e
} ;
```

In our `AdjCN` example, we give the following 4 arguments:

* The type of what it returns (`Str`)
* The Boolean that it checks (`adj.isPre`, i.e. whether the adjective
  is premodifier)
* What happens if the adjective *is* premodifier (put `adj.s` before
`cn.s`, return that string)
* What happens if the adjective *is not* premodifier (put `adj.s` after
`cn.s`, return that string).

Why do we need the first argument? I don't know the details, but as
far as I've understood, the cool expressive power that comes from
dependent types comes with a tradeoff: the compiler can't just *guess*
things anymore. (In type theorist lingo, dependent types don't mix
well with polymorphism<!--type inference ? -->.) Like in this case,
we gave `if_then_else` two strings as arguments, but we still needed
to give the *type* `Str` as the first argument.


## Dependent types

The "types can be *of type*" thing, as well as "annotate the
hell out of everything because the type checker seems like a clueless
idiot" thing is just a prerequisite for doing cool stuff with dependent types.

<!-- Despite the subheading, I am not going to go further into dependent
types in this post. In order to write resource grammars (or most
application grammars), you would do perfectly fine without ever having
heard the concept *dependent type*. I just wanted to explain two things
that appear in GF grammars quite often, and which probably seem weird
if you have background in other programming languages. -->

Because this post is about GF, my example of "cool stuff" will be about "make grammars produce sentences that make sense"[^1]. If you find this example too contrived
but are still curious about dependent types, go and read [something](https://blog.jle.im/entry/practical-dependent-types-in-haskell-1.html) [else](https://www.quora.com/Why-do-dependent-types-matter)!



### The problem

My supervisor tells me that research topics should emerge from a desire to solve a problem, not from a desire to use a particular technology. (Or at least one should give the former reason in the paper.) So let's start with a problem that we will [solve with dependent types](https://www.xkcd.com/2021/).

As you know, the GF resource grammar is purely syntactic and overgenerates a lot.
Application grammars often use more elaborate types in the abstract syntax, such that "Monet painted flowers" is a valid sentence but "flowers painted Monet" isn't.
We can write such a grammar easily with just any old boring types[^2], like this:

```haskell
cat
  Person ;
  Object ;
  Sentence ;
fun
  Paint : Person -> Object -> Sentence ;
  Monet : Person ;
  Flowers : Object ;
```

The distinctions such as separating humans from objects are broad enough to have their own categories. But sometimes we need even finer distinctions, which may only apply to certain constructions.

For example, in the domain of museum object descriptions, the people appearing in the texts are likely to be either artists or subjects. As a concrete motivation for fine-tuning the semantic distinctions in a grammar, we examined the corpus provided by Göteborgs Stadsmuseum, and found out that the descriptions *artwork of* (depicting) and *artwork by* (authored) are often expressed with the same construction, using the Swedish preposition *av* for both meanings. In other words, *ett porträtt av Mona Lisa* gets two parses: one where Mona Lisa is the the subject and one where she is the painter.

### Restrictive abstract syntax, without dependent types

As a solution to ambiguity of the roles, we could split the abstract syntax category `Person` into two categories, `Painter` and `Subject`, and define the functions `authoredBy` and `depicting` so that they accept only one as their argument. This way, even if the constructions are identical, the type of the argument disambiguates the expression.

```haskell
cat
  Painting ;
  Painter ;
  Subject ;
fun
  portrait   : Painting ;
  authoredBy : Painting -> Painter -> Painting ;
  depicting  : Painting -> Subject -> Painting ;
  Mona_Lisa  : Subject ;
  Leonardo   : Painter ;
```

However, separating painters and subjects will lead into problems. There are many portraits depicting painters, but the grammar would not allow a painter to be in any other role than an author--*a portrait of Leonardo da Vinci* would be considered a syntax error and not be parsed. Furthermore, if there are functions common to all people, such as being born or dying, each of them would have to be defined separately for painters and subjects. There are some actions that can only be done to one of them, but since the majority of the actions are the same, it feels redundant to put them in different categories.

### Restrictive abstract syntax with dependent types

As a solution to too rigid grammar, we can define *dependent types*. In addition to the simple type `Person`, we add dependent type constructors `Painter` and `Subject`, which take some `Person` as argument, producing types such as `Painter Leonardo` and `Subject Mona_Lisa`. They provide a semantic layer on top of the syntactic level, more flexible and easier to modify.

```haskell
cat
  Painting ;
  Person ;
  Subject Person ;
  Painter Person ;
fun
  Mona_Lisa  : Person ;
  Leonardo   : Person ;
  portrait   : Painting ;
  authoredBy : Painting -> (p : Person) -> Painter p -> Painting ;
  depicting  : Painting -> (p : Person) -> Subject p -> Painting ;
  -- proof objects
  Mona_Lisa_Subject : Subject Mona_Lisa ;
  Leonardo_Painter  : Painter Leonardo ;
```

The declarations of portrait, Mona Lisa and Leonardo da Vinci haven't changed.
But now, let us look at the functions `authoredBy` and `depicting`. Like before, they take a `Painting` and a `Person` as their arguments and construct a new value of type `Painting`. The notation `(p : Person)` is a way to bind the argument to a variable, so that we can refer to the same argument later (just like the first `(A : Type)` in [if_then_else](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/prelude/Prelude.gf#L64-L69)).

The third argument of  `authoredBy` and `depicting` is of the dependent type `Painter p` or `Subject p`, in which the variable `p` must be the `Person` given as the second argument.
In our concrete case, the only options are `Subject Mona_Lisa` and `Painter Leonardo`.

We define these terms, called `Mona_Lisa_Subject` (of type `Subject Mona_Lisa`) and `Leonardo_Painter` (of type `Painter Leonardo`) at the end of the grammar. They are called *proof objects*, and their purpose is to control which combinations of persons and roles are valid. Technically they are (argumentless) functions just like any other under the `fun` title, but they are not linearised in the concrete syntax. They only exist to guide the GF compiler that something is, or isn't, a valid tree.

It is possible to define many proof objects for one person. Say we want to add
*self-portrait* into the lexicon, and require that an author of a self-portrait
can only be a person who is both a subject and a painter. To do this, we would
add to the abstract syntax a function that constructs the self-portrait, and
accompanying proof objects for each painter who has done a self-portrait.
Below is an example for Leonardo da Vinci.

```haskell
fun
  selfportrait : Painting -> (p : Person) ->
                 Painter p -> Subject p -> Painting ;
  Leonardo_Subject : Subject Leonardo ;
```

Let's recap a bit: the function `authoredBy` can be applied to any person `p`,
for which we have defined a proof object of type `Painter p`. Likewise for
`depicting`, with a proof object of type `Subject p`.

Adding dependent types does not rule out anything previously done. The basic type
of all people is still `Person`. In addition, Leonardo da Vinci belongs to the
categories `Painter Person` and `Subject Person`, and Mona Lisa to `Subject Person`.
If some language treats the owner of a portrait in a way that results in ambiguity,
it is easy to add a new type, `Collector Person`, and write a proof object for
each person who is a collector.

### Ontologies in GF

So far our concern has been to handle the ambiguity of natural language, in this case the
same preposition being used for author and subject. But how about the bigger goal, creating
sentences that make sense?

Do you see the problem? With our fancy grammar, we could still claim that *any*
painting is painted, depicts or is a self-portrait of Leonardo.
The function `selfportrait` only checks whether the potential author has ever
been a subject, nothing about the *painting* itself.

To ensure that the painting really is a self-portrait, we need to fine-tune our approach a bit. Dependent types can contain several terms: for all requirements of `depicting`, we need a type constructor that takes both `Person` and `Painting`, and produces a proof object of type "yes, this $painting depicts this $person". For `authoredBy` it's the same business, except the proof object says "yes, this $painting was authored by this $person".

And what kind of proof object(s) does `selfportrait` need? Think a moment and then look at the code below.

```haskell
cat
  Person ;
  Painting ;
  Authored Person Painting ;
  Depicting Person Painting ;
fun
  Portrait_of_a_man : Painting ;
  Portrait_Mona_Lisa : Painting
  Leonardo : Person ;
  Mona_Lisa : Person ;
  depicting : (pe : Person) -> (pa: Painting) ->
    Depicting pe pa -> Painting ;
  authoredBy : (pe : Person) -> (pa: Painting) ->
    Authored pe pa -> Painting ;
  selfportrait : (pe : Person) -> (pa: Painting) ->
    Authored pe pa -> Depicting pe pa -> Painting ;
  Authored_Leonardo_ML : Authored Leonardo Portrait_Mona_Lisa ;
  Authored_Leonardo_PAM : Authored Leonardo Portrait_of_a_man ;
  Depicting_Leonardo_PAM : Depicting Leonardo Portrait_of_a_man ;
  Depicting_MonaLisa_PML : Depicting Mona_Lisa Portrait_Mona_Lisa ;
```

That's right, we need both `Authored pe pa` and `Depicting pe pa`, otherwise it's not a self-portrait.

And now we can generate sentences that are a) not ambiguous, and b) describe the reality!
To be honest, at this point the grammar is not so nice to read nor write anymore, but there's always tradeoffs in life.

In fact, the [SUMO ontology](https://en.wikipedia.org/wiki/Suggested_Upper_Merged_Ontology)
has been translated into GF [(Enache, 2010)](https://publications.lib.chalmers.se/records/fulltext/116606.pdf). Instead of fine semantic distinctions, dependent types can be used to represent the whole inheritance hierarchy of an ontology.
I'm not going to delve into this particular work any further, but here's the contents--if anything in the contents looks interesting, go and read the [thesis](https://publications.lib.chalmers.se/records/fulltext/116606.pdf) and check out the [code](https://github.com/GrammaticalFramework/gf-contrib/tree/master/SUMO)!

![Reasoning and Language Generation in the SUMO
Ontology](/images/ramona-thesis.png "contents of Reasoning and Language Generation in the SUMO
Ontology (Enache, 2010)")

## Read more

If you want a different introduction into dependent types in GF (with a similar focus on sentences that make sense),
[here's the section in the tutorial](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc110).

If you want to learn about dependent types in more general-purpose
programming languages, maybe pick up some
[Agda](http://wiki.portal.chalmers.se/agda/pmwiki.php) or [Idris](https://www.idris-lang.org/). If you're already into Haskell, [this tutorial](https://www.schoolofhaskell.com/user/konn/prove-your-haskell-for-great-safety/dependent-types-in-haskell) might be more readable (it's from 2014, I don't know if the concrete details are still state of the art in Haskell). I heard recently about [Ç](https://github.com/cedille), which is apparently implemented in Agda--that's the only reason I can call Agda a "general-purpose programming language" with a straight face.


## Footnotes

[^1]: The second half is adapted from my master's thesis which I wrote in 2012. I originally started this blog to get started with PhD thesis writing, so recycling text from two theses back completes the circle.

[^2]: Maybe for those coming from [other](https://en.wikipedia.org/wiki/Python_(programming_language)) [programming](https://en.wikipedia.org/wiki/Common_Lisp) [languages](https://en.wikipedia.org/wiki/Lambda_calculus), even the standard GF types are whoa.
