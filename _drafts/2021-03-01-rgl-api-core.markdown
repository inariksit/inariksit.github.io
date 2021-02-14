---
layout: post
title:  "GF Resource Grammar Library: the core and the API"
date:   2021-02-01
categories: gf
tags: gf
---


The [Resource Grammar Library](http://www.grammaticalframework.org/lib/doc/synopsis/index.html) (RGL) is the standard library of GF. In this post, we explore the API and the internals of the RGL.

To appreciate this post, you should understand what this line means.

```haskell
mkCl (mkNP the_Det cat_N) sleep_V
```

After reading this post, you will (hopefully) also understand where this tree came from,

```haskell
PredVP
  (DetCN (DetQuant DefArt NumSg) (UseN cat_N))
  (UseV sleep_V)
```

and how it is related to the first line.

If you don't understand the first line (`mkCl …`), I recommend that you read the GF tutorial all the way to [Lesson 4](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc69), until you reach the section on [functors](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc90).

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Using the RGL (API) in an application grammar](#using-the-rgl-api-in-an-application-grammar)
- [Insides of the RGL](#insides-of-the-rgl)
- [History lesson](#history-lesson)
    - [Before RGL](#before-rgl)
    - [Introducing RGL: an `abstract` with `concrete`s](#introducing-rgl-an-abstract-with-concretes)
    - [Before RGL API](#before-rgl-api)
    - [RGL API, finally](#rgl-api-finally)
- [Footnotes](#footnotes)

<!-- markdown-toc end -->


## Using the RGL (API) in an application grammar

This is how we learn to use the RGL: open the API modules `SyntaxXxx` or `ParadigmsXxx` in our application grammar. A minimal example:

```haskell
abstract Example = {
  cat
    Phrase ;
  fun
    example1_Phrase : Phrase ;
    example2_Phrase : Phrase ;
}
```

<a name="first-concrete"></a>
Let's do an English concrete for this grammar.

```haskell
concrete ExampleEng of Example = open SyntaxEng,
                                      LexiconEng,
                                      ParadigmsEng in {
  lincat
    Phrase = Utt ; -- category Utt comes from SyntaxEng
  lin
    example1_Phrase =
      mkUtt            -- mkUtt, mkNP and
        (mkNP this_Det  -- this_Det come from SyntaxEng
              dog_N    -- dog_N comes from LexiconEng
        ) ;

   example2_Phrase =
      mkUtt
        (mkNP many_Det
              (mkN     -- mkN comes from ParadigmsEng
                "aardvark")
        ) ;

}
```

I trust that you have seen, and probably used, something like this before.
<!-- All of the `mkXxx` things are `oper`s, and you can use them in defining your `lin`s.  -->
And you should know how to make this example multilingual: just look up the base form of the word "aardvark" [in a dictionary](https://en.wiktionary.org/wiki/oritteropo#Italian), and change the string `Eng` to `Ita`, and you have an Italian concrete syntax.
<!-- If the oper takes a string as an argument (like `mkN` does), it comes from a Paradigms module, and otherwise (like `mkUtt` or `this_Det`), it comes most likely from a Syntax module. -->
<!-- I expect you to know all of this already. But I don't expect you to know the rest of this post. -->

```haskell
concrete ExampleIta of Example = open SyntaxIta,
                                      LexiconIta,
                                      ParadigmsIta in {
  lincat
    Phrase = Utt ;
  lin
    example1_Phrase = mkUtt (mkNP this_Det dog_N) ;
    example2_Phrase = mkUtt (mkNP many_Det (mkN "oritteropo")) ;
}
```

But where do all these `mkXxx` opers come from? How come they behave the same for all languages?

## Insides of the RGL

Depending on where you learned GF, you might have seen something like the following.

<img src="/images/open-RGL-in-GF-shell.png"
     alt="Screenshot of a GF shell, where LangEng.gfo is open, and we see an abstract syntax tree for 'the cat sleeps': PredVP (DetCN (DetQuant DefArt NumSg) (UseN cat_N)) (UseV sleep_V)."/>

The screenshot is showing a GF shell, where I've opened a GF file called `LangEng.gfo`. Then I parsed the sentence *"the cat sleeps"*, and got the following abstract syntax tree.

```haskell
PredVP
  (DetCN (DetQuant DefArt NumSg) (UseN cat_N))
  (UseV sleep_V)
```

This is glimpse to the abstract syntax of the RGL. All of these words, like `PredVP` and `DefArt`, are `fun`s in modules such as [Sentence.gf](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/abstract/Sentence.gf) or [Noun.gf](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/abstract/Noun.gf#L81-L86), and `lin`s in their respective [SentenceEng.gf](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/english/SentenceEng.gf), [NounRus.gf](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/russian/NounRus.gf) or any other of the 40+ languages in the RGL.

No need to understand the linked code, I just wanted to show you how the RLG is just another GF grammar.




## History lesson

Disclaimer: I only came into GF in 2010, and by then the RGL API was stable. Think of the historical details as a dramatic re-enactment.

### Before RGL

Once upon a time, there was no RGL. All application grammars looked like the [Foods grammar](https://github.com/GrammaticalFramework/gf-contrib/blob/master/foods/FoodsEng.gf) that isn't using the RGL, or the examples from [lesson 3](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc46) of the GF tutorial.

<!-- Soon, it got very tiring to have to define noun inflection, verb conjugation, word order, preposition contraction and whatnot for every new application grammar. The domains of, say, mathematics and healthcare may be very different in vocabulary and style, so it makes sense to write different application grammars for them. But regardless of the domain, Italian adjectives need to agree in number and gender with their head.  -->

Defining all the low-level details again for each new application grammar would get tiring. English nouns inflect just the same, no matter if the domain is mathematics of health care. So the obvious first idea is to save those `param`s and `oper`s [from  FoodsEng](https://github.com/GrammaticalFramework/gf-contrib/blob/master/foods/FoodsEng.gf#L29-L42), and reuse them for other English concrete syntaxes of different application grammars.

```haskell
param
  Number = Sg | Pl ;
oper
  det : Number -> Str ->
    {s : Number => Str} -> {s : Str ; n : Number} =
      \n,det,noun -> {s = det ++ noun.s ! n ; n = n} ;
  noun : Str -> Str -> {s : Number => Str} =
    \man,men -> {s = table {Sg => man ; Pl => men}} ;
  regNoun : Str -> {s : Number => Str} =
    \car -> noun car (car + "s") ;
  adj : Str -> {s : Str} =
    \cold -> {s = cold} ;
  copula : Number => Str =
    table {Sg => "is" ; Pl => "are"} ;
```

So we save this into a resource module called `ProtoRGLEng.gf`. It's a good start, but there are limitations.

Nothing requires the whole set of `oper`s to be internally consistent. The type `{s : Number => Str}` is used in `noun` and `det`, but one of them could easily change, and the  `ProtoRGLEng` module will still compile. Only when you try to compile an application grammar that uses those opers, it will stop working.

In addition, nothing requires the opers in `ProtoRGLEng` and, say, `ProtoRGLIta` to be consistent with each other. Some things are necessarily different in such opers, like inflection tables for adjective---1 form in English, 4 forms in Italian. But other choices are arbitrary, like how to implement determiners (like *this*, *many*, *the*). The code below illustrates two alternatives.

```haskell
oper
  -- Common ground: Nouns and NPs
  Noun : Type ;
  dog_N : Noun ;

  NP : Type ; -- NPs are formed out of Nouns

-- Option 1: Determiner as a category
  Det : Type ;
  this_Det, many_Det : Det ;
  addDet : Det -> Noun -> NP ;

-- Option 2: Determiners as functions: Noun -> NP
  this, many : Noun -> NP ;
```

Decisions like that don't really matter much in the grand scheme of things, but it's annoying to try to remember which languages choose `addDet this_Det dog_N` and which choose `this dog_N`. And while we're talking about annoying, is it really necessary to keep thinking about the details of Italian and English adjective inflection while doing syntax?

Now is there a way in GF to enforce a common, abstract idea and hide the implementation details in different languages? 🤔

Sounds like we're better off defining this resource with *abstract* and *concrete* syntaxes.

### Introducing RGL: an `abstract` with `concrete`s

Now, let's take these `oper`s from the previous section, and redesign them as `cat`s and `fun`s. We want to cover nouns, determiners and noun phrases. I'll present an almost-real subset of the RGL abstract syntax in the following. Since we already have [mini and micro versions](https://github.com/GrammaticalFramework/comp-syntax-2020/tree/master/lab2/grammar/abstract), I'll call this the NanoRGL.

First, we have our types for determiners, nouns and noun phrases. These are proper `cat`s, not opers.

```haskell
cat
  Det ; -- Determiner
  N ;   -- Noun
  NP ;  -- Noun phrase
```

Our lexicon and the single syntactic function are all `fun`s. With these, our goal is to say "this dog" and "many dogs".

```haskell
fun
  -- Lexicon
  this_Det : Det ;
  many_Det : Det ;
  dog_N : N ;

  -- Syntactic functions
  DetN : Det -> N -> NP ;
```

Now that we have made this into an abstract syntax, we can do much more fun things with it! Like open it in a GF shell and generate all combinations.

```
NanoRGL> gt -cat=NP
DetN many_Det dog_N
DetN this_Det dog_N
```

#### Concrete syntax

I omit the concrete syntaxes in the post, but English is available on [GitHub](https://github.com/inariksit/rgl-tutorial/tree/main/lesson1) (other languages are homework). As for any GF abstract and concrete, you don't need to know anything about the implementation details to linearise something.

```
NanoRGL> l DetN many_Det dog_N
many dogs
molti cani
```


<!-- Now in a *concrete syntax* like NanoRGLEng, we can and should define internal `oper`s that make sense for English and satisfy the aesthetic preferences of the grammarian. -->
<!-- I'd demonstrate with a really silly way of defining `DetN`, but if my readers read this blog like I read other programming blogs, they will skip the explanations and just [copy and paste snippets in monospace font](https://gkoberger.github.io/stacksort/) into a file until it compiles. So I won't do it, but you are welcome to try it on your own. -->

Internal implementation of `NanoRGLEng` and `NanoRGLIta` may be different, but that doesn't matter, because the common abstract syntax guarantees homogeneity. (Unless the grammarian is evil and decides that `DetN` should always ignore its arguments and just return a constant string.)

#### Benefits of abstract NanoRGL/concrete NanoRGLEng

<!-- So now we have transformed the `oper`s from Foods into an abstract and two concretes called NanoRGL.  -->

The two benefits, compared to language-specific opers, are

* Internal consistency. If we change the lincat of `N`, we must also change the implementations of `dog_N` and `DetN`, otherwise the grammar won't even compile.
* Hides language-specific implementation details.

Now we have an abstract and two concrete modules. How do we use them from an application grammar?

<!-- like `noun` and `regNoun`: -->

<!-- ```haskell -->
<!-- oper -->
<!--   noun : Str -> Str -> {s : Number => Str} = -->
<!--     \man,men -> {s = table {Sg => man ; Pl => men}} ; -->

<!--   regNoun : Str -> {s : Number => Str} = -->
<!--     \car -> noun car (car + "s") ; -->
<!-- ``` -->

<!-- So that the linearisation for `dog_N` can be just `dog_N = regNoun "dog"`. Likewise, the Italian concrete may have its own idiosyncracies. -->


### Before RGL API

I suspect that this step is the most non-obvious one. All the time until now, we've seen how to use `oper`s in definitions of `lin`s. But here's the news:

1. **You can define `lin`s in terms of other `lin`s too, not just `oper`s.**
1. **Hence, you can use RGL without the API.**

Remember the minimal application grammar from the beginning of the post?
First I implemented the concrete using the RGL API. Now I'll implement another concrete, first using our NanoRGL, and then using the actual RGL abstract syntax, not via API.
<!-- As long as you have -->
<!-- `GF_LIB_PATH` set up on your computer and NanoRGL in the same directory with Example, you should be able to run all 3 concrete syntaxes. -->

As a reminder, the abstract syntax of the application.

```haskell
abstract Example = {
  cat
    Phrase ;
  fun
    example1_Phrase : Phrase ;
    example2_Phrase : Phrase ;
}
```

The first concrete uses the NanoRGL. If you have Example and NanoRGL in the same directory, you can run this grammar perfectly fine.

```haskell
concrete ExampleEngNano of Example = open NanoRGLEng in {
  lincat
    Phrase = NP ;       -- defined in NanoRGL as cat
  lin
    example1_Phrase =
      DetN               -- defined in NanoRGL as fun
        this_Det         -- defined in NanoRGL as fun
        dog_N ;          -- defined in NanoRGL as fun

    example2_Phrase =
      DetN
        many_Det         -- defined in NanoRGL as fun
        (regNoun         -- defined in NanoRGLEng as oper
          "aardvark") ;
}
```

The second concrete uses the actual RGL, and recreates the [first concrete](#first-concrete) that uses the API. Now instead of a single `SyntaxEng`, I need to open multiple modules, such as `StructuralEng`, `NounEng` and `PhraseEng`, because all these funs and lins are in different modules. In contrast, all of the `mkUtt`, `mkNP` etc. are in a single module, `SyntaxEng`.



```haskell
concrete
  ExampleEngRGLNoAPI of Example = open CatEng,
                                       NounEng,
                                       PhraseEng,
                                       LexiconEng,
                                       StructuralEng,
                                       ParadigmsEng in {
lincat
  Phrase = Utt ; -- from CatEng
lin
  example1_Phrase =
    UttNP        -- from PhraseEng
      (DetCN     -- homework: where do the rest come from?
         (DetQuant this_Quant NumSg)
         (UseN dog_N)
      ) ;
}
```

This was a step forward, but still a bit annoying to write. So many things to import! So many long names that make no sense to non-linguists, and confuse actual linguists, because the `fun` and `cat` names are a random mix and match from different linguistic traditions.

Around 2006 (TODO check year), there were enough people writing GF that the RGL situation was getting chaotic. The `fun`s and `lin`s were changing all the time. So the wise people at Chalmers decided that maybe, after all, we need more `oper`s. But let's be smart this time.


### RGL API, finally

We learned that it's possible to define `lin`s in terms of other `lin`s. I hope you've had time to digest that, because here's some more news:

* __You can define `oper`s in terms of `lin`s, too.__

Here's a trivial example. You can put this resource module `NanoAPIEng` in the same directory with the abstract and concrete NanoRGL(Eng). In `NanoAPIEng`, we have an oper called `mkNP`, which is calling the lin `DetN` from `NanoRGLEng`.

```haskell
resource NanoAPIEng = open NanoRGLEng in {
  oper
    mkNP : Det -> N -> NP = DetN ;
}
```

What's so great about this? Well, a lot of things!

* We can *overload* opers. This means that we can have several opers called `mkNP`, with different type signatures.
* We can add shorthands to the opers for the convenience of application grammarians, while keeping the abstract/concrete as complicated as it needs to be, to accommodate the more complicated linguistic constructions.

<!-- (For instance, `CN`, which is between `N` and `NP`.) -->

#### Overloading

Suppose that we want to allow two kinds of `NP`: with a determiner and without. We'll call both `mkNP`, and the distinguishing feature is the type signature.

```haskell
resource NanoAPIEng = open NanoRGLEng in {

oper
  mkNP = overload {
    mkNP : Det -> N -> NP = DetN ; -- same as before
    mkNP : N -> NP = \n ->
      let emptyDet : Det = lin Det {s = [] ; num = Sg}
       in DetN emptyDet n
   } ;
}
```

<small>For an explanation of `lin Det`, see [my post on subtyping](../../../2018/05/25/subtyping-gf.html#lock-fields). Not essential for the rest of this post.</small>

Let's test this in the GF shell.
<!-- With `i -retain`, we keep all opers in scope, even those from `NanoRGLEng`. -->

```
$ gf
…
> i -retain NanoAPIEng.gf
> cc mkNP (regNoun "dog")
{s = "dog"; lock_NP = <>}

> cc mkNP this_Det (regNoun "dog")
{s = "this" ++ "dog"; lock_NP = <>}
```

#### Shorthand for more complex internal structure

In our minimal NanoRGL, we had the type signature `Det -> N -> NP` for making noun phrases. This is a useful API,
which can stay unchanged in the resource module, even if we add more layers of indirection in the abstract/concrete modules.

Now let's change the internals of NanoRGL! Introducing a new category `CN`, which is between `N` and `NP`.[^1] The internal abstract syntax is now this:

```haskell
cat
  Det ; N ; CN ; NP ;

fun
  UseN : N -> CN ;
  DetCN : Det -> CN -> NP ;

  -- Used to be DetN : Det -> N -> NP
```

Actually, I can't be bothered to make a new version of NanoRGL. I'll just make a custom API for the full RGL! The snippet I just pasted is literally a fragment of the full RGL.

If you have your `$GF_LIB_PATH` set properly, you can copy and paste this file anywhere on your computer and run it in GF shell.

```haskell
resource FullAPIEng = open
  NounEng, -- this module comes from the actual RGL

  -- To get some lexicon to play with in GF shell
  StructuralEng,
  LexiconEng
  in {

  oper
    mkNP : Det -> N -> NP = \det,n ->
      DetCN det (UseN n) ;
}
```

Here's my new custom API. To define `mkNP`, I only need to open the module `NounEng`, but I also open `StructuralEng` and `LexiconEng` to get some words to play with in the GF shell.

Here's what I can do in the GF shell.

```
$ gf
…
> i -retain FullAPIEng.gf
> cc mkNP many_Det cat_N
{s = table
       ResEng.NPCase
       ["many" ++ "cats"; "many" ++ "cats'"; "many" ++ "cats";
        "many" ++ "cats"];
 a = ResEng.AgP3Pl ResEng.Neutr; lock_NP = <>}
```

And naturally, I can use this `FullAPIEng` resource module in any application grammar if I wanted to. For actual grammars, better stick to `SyntaxEng` though---it has many more overloaded instances of `mkNP`.

<a href="http://www.grammaticalframework.org/lib/doc/synopsis/index.html#NP"><img src="/images/mkNP-api-fragment.png" alt="Screenshot of RGL API, with many overloaded instances of mkNP" /></a>

### Recap

* The opers in `SyntaxEng`, `SyntaxRus` and any other `SyntaxXxx` come from funs and lins in the RGL internal modules, such as `NounEng`, `PhraseRus`, `AdverbIta`.

* There are benefits to defining all of these `Noun`, `Phrase`, `Adverb` etc. modules as abstract syntaxes with concretes: internal consistency and hiding language-specific details.

* There are also benefits to define an API of opers on top of the low-level core: overloading and simplifying shortcuts.

I have simplified this view in one way: the actual `SyntaxXxx` modules are not technically `resource` modules, but there's an `interface` module `Syntax` with `instance`s for different concrete syntaxes. I have deliberately avoided talking about those parts of the GF module structure, and I will continue doing so. Understanding those details is not crucial for this topic.

## Funs and lins in RGL outside the API

So, the RGL API was made in the 2000s, but the RGL has continued to develop since then. There are a number of modules and even funs in the old modules, that are not accessible from the API.

## Read more

I have made a tutorial [on GitHub](https://github.com/inariksit/rgl-tutorial), which covers much of the same topics that this post does. I'm keeping the two as separate but overlapping resources, mostly because I want to explore different ways of explaining this topic to new GF users.

<!-- Opening Syntax brings most of Lang into scope, with standard API overloaded names. So if you opened LangEng and SyntaxEng in an application, then you'd have e.g. `mkNP` and `DetCN` both in scope. -->

<!--  GF grammarian's journey where they find out that the RGL is made of ~~people~~abstract syntax. -->

## Footnotes


[^1]: Slightly offtopic for the purposes of this post, but if you're curious about why do we need `CN` in the full RGL, read on.<br/><br/>In addition to determiners, nouns and noun phrases, we have a category `CN`, *common noun*. It's there to allow modifiers that can be piled on. Adjectives are an example of such modifiers: you can say "big old red house" in standard English. Determiners are not: you can't say "*the a my house".<br/><br/>Since `N` is strictly a lexical category, we need an intermediate category `CN` that can be _modified_ by adding e.g. adjectives (_big_ house), adverbs (house _on the hill_) or relative clauses (house _that was sold_). It's not a `NP` yet, because it doesn't have a determiner, but I hope you agree that "big house" is not just a noun anymore. Hence `CN`.<br/><br/>So adjectives and other such modifiers were the justification for adding an intermediate category `CN`. But we still want to allow also noun phrases without adjectives: "this dog", not just "this small dog". Ideally, we want a single rule to turn "dog" and "small dog" into a `NP`. That is possible, if we allow the smallest `CN` to be just a noun. That's why we have the seemingly useless function `UseN : N -> CN`.
