---
layout: post
title:  "RGL: the core and the API
date:   2021-03-01
categories: gf
tags: gf
---


The [Resource Grammar Library](http://www.grammaticalframework.org/lib/doc/synopsis/index.html) (RGL) is the standard library of GF. To appreciate this section, you should be familiar with [Lesson 4 in the GF tutorial](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc69), everything before the section on [functors](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc90).


## Using the RGL in an application grammar
This is how we learn to use the RGL: open the API modules `SyntaxXxx` or `ParadigmsXxx` in our application grammar. A minimal example:

```haskell
abstract Example = {
  cat
    Comment ;
  fun
    example_Comment : Comment ;
}
```

Let's do an English concrete for this grammar.

```haskell
concrete ExampleEng of Example = open SyntaxEng,
                                      ParadigmsEng in {
  lincat
    Comment = Utt ; -- category Utt comes from SyntaxEng
  lin
    example_Comment =
      mkUtt
        (mkNP this_Det -- mkUtt, mkNP and this_Det come from SyntaxEng
          (mkN "dog")  -- mkN comes from ParadigmsEng
        ) ;
}
```

I trust that you have seen, and probably used, something like this before. All of the `mkXxx` things are `oper`s, and you can use them in defining your `lin`s.


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

No need to understand the linked code, I just wanted to show where to find the RGL source.

## History lesson

### Before RGL

Once upon a time, there was no RGL. All grammars looked like the [Foods grammar](https://github.com/GrammaticalFramework/gf-contrib/blob/master/foods/FoodsEng.gf) that isn't using the RGL, or the examples from [lesson 3](http://www.grammaticalframework.org/doc/tutorial/gf-tutorial.html#toc46) of the GF tutorial.

<!-- Soon, it got very tiring to have to define noun inflection, verb conjugation, word order, preposition contraction and whatnot for every new application grammar. The domains of, say, mathematics and healthcare may be very different in vocabulary and style, so it makes sense to write different application grammars for them. But regardless of the domain, Italian adjectives need to agree in number and gender with their head.  -->

Defining all those opers again for each new application grammar would get tiring. English nouns inflect just the same, no matter if the domain is mathematics of health care. So the obvious first idea is to save those `param`s and `oper`s [from line 29 on](https://github.com/GrammaticalFramework/gf-contrib/blob/master/foods/FoodsEng.gf#L29-L42) and reuse them for other English grammars.

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

### Introducing RGL

Now, let's take these `oper`s from the previous section, and redesign them as `fun`s and `lin`s. We want to cover nouns, determiners and noun phrases. I'll present an almost-real subset of the RGL abstract syntax in the following.

First, we have our types for determiners, nouns and noun phrases.

```haskell
cat
  Det ; -- Determiner
  N ;   -- Noun
  NP ;  -- Noun phrase
```

Next I introduce our lexicon. With these, our goal is to say "this dog" and "many dogs".
(No adjectives yet, because I want to keep the example small.)

```haskell
fun
  -- Lexicon
  this_Det : Det ;
  many_Det : Det ;
  dog_N : N ;
```

Finally, here's a syntactic function that combines the determiner and the noun.


```haskell
fun
  -- Syntactic functions
  DetN : Det -> N -> NP ;
```

Now that we have made this into an abstract syntax, we can do much more fun things with it! Like open it in a GF shell and generate all combinations.

```
RGLLite> gt -cat=NP
DetN many_Det dog_N
DetN this_Det dog_N
```

#### Concrete syntaxes of the RGL

I omit the concrete syntaxes in the post, but they are available at [linkTODO](TODO). As for any GF abstract and concrete, you don't need to know anything about the implementation details to linearise something.

```
RGLLite> l DetN many_Det dog_N
many dogs
molti cani
```

Now in the *concrete syntax* of this proto-RGL, we can and should write all these previous opers, like `noun` and `regNoun`:

```haskell
noun : Str -> Str -> {s : Number => Str} =
  \man,men -> {s = table {Sg => man ; Pl => men}} ;
regNoun : Str -> {s : Number => Str} =
   \car -> noun car (car + "s") ;
```

So that the linearisation for `dog_N` can be just `dog_N = noun "dog"`.

<!-- In addition to determiners, nouns and noun phrases, we have a category `CN`, *common noun*. It's there to allow modifiers that can be piled on. Adjectives are an example of such modifiers: you can say "big old red house" in standard English. Determiners are not: you can't say "*this a my house". -->

<!-- Since `N` is strictly a lexical category, we need an intermediate category `CN` that can be *modified* by adding e.g. adjectives (*big* house), adverbs (house *on the hill*) or relative clauses (house *that was sold*). It's not a `NP` yet, because it doesn't have a determiner, but I hope you agree that "big house" is not just a noun anymore. Hence `CN`. -->



<!-- Yes, adjectives and other such modifiers were the justification for adding an intermediate category `CN`. But we still want to allow also noun phrases without adjectives: "this dog", not just "this small dog". Ideally, we want a single rule to turn "dog" and "small dog" into a `NP`. That is possible, if we allow the smallest `CN` to be just a noun. -->

### Before RGL API

I suspect that this step is the most non-obvious one. All the time until now, we've seen how to use `oper`s in definitions of `lin`s. But here's the news:

1. **You can define `lin`s in terms of other `lin`s too, not just `oper`s**
1. **Hence, you can use RGL without the API.**

Remember the minimal application grammar [from the beginning of the post](TODO)? First I implemented the concrete using the RGL API. Now I'll implement another concrete, first using our RGLLite, and then using the actual RGL abstract syntax, not via API. As long as you have `GF_LIB_PATH` set up on your computer, you should be able to run all 3.

As a refresher, the abstract syntax of the application.

```haskell
abstract Example = {
  cat
    Comment ;
  fun
    example_Comment : Comment ;
}
```

The first concrete uses the RGLLite. If you have it in the same directory, you can run this.

```haskell
concrete ExampleEngLite of Example = open RGLLiteEng in {
  lincat
    Comment = NP ;
  lin
    example_Comment =
      DetN               -- defined in RGLLite as fun/lin
        this_Det         -- defined in RGLLite as fun/lin
        (noun "dog")     -- defined in RGLLiteEng as oper

}
```

The second concrete uses the actual RGL, and recreates the [first concrete](TODO) that uses the API. Now instead of a single `SyntaxEng`, I need to open `StructuralEng`, `NounEng` and `PhraseEng`, because all these funs and lins are in different modules. In contrast, all of the `mkUtt`, `mkNP` etc. are in a single module, `SyntaxEng`.


```haskell
concrete ExampleEng of Example = open SyntaxEng,
                                      ParadigmsEng in {
  lincat
    Comment = Utt ; -- category Utt comes from SyntaxEng
  lin
    example_Comment =
      mkUtt         -- opers mkUtt, mkNP, a_Det
        (mkNP a_Det -- come from SyntaxEng
          (mkN "example") -- mkN comes from ParadigmsEng
        ) ;
}
```
<!-- Opening Syntax brings most of Lang into scope, with standard API overloaded names. So if you opened LangEng and SyntaxEng in an application, then you'd have e.g. `mkNP` and `DetCN` both in scope. -->

<!--  GF grammarian's journey where they find out that the RGL is made of ~~people~~abstract syntax. -->

<!-- I've written posts about -->
