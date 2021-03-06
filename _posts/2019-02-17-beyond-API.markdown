---
layout: post
title:  "Low-level hacks in application grammars: better practices"
date:   2019-02-17
categories: gf
tags: gf
---

This post addresses the common dilemma in a grammarian's life: you're *supposed* to stick to the RGL API, but sometime's it's not enough. Reasons include, but are not limited to

* The resource grammar itself has an error, and you're not comfortable with either the language you're working with or the RGL structure itself to fix the error at the source. So you need to bypass it in your application.
* Your abstract syntax isn't quite as abstract as you'd wished, and for some reason[^1], you can't change the abstract syntax. For example, you have `undress_V : V` in the lexicon, but the best way to translate it in your language is "take off my/your/… clothes". There's no API function to force an *inflecting* non-verby component into a `V`, because that stuff just screams "I am a VP level phenomenon".

Of course, the *best* practice is that you fix or extend the RG, or change the abstract syntax of your application grammar. Also remember to check [Extend](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/abstract/Extend.gf#L1) to see if the function you want isn't there already. But in practice, we grammarians are writing dubious hacks anyway, so might as well do it *better*.

- [Example](#example)
- [How to make the questionable less questionable](#how-to-make-the-questionable-less-questionable)
  * [Record extension](#record-extension)
  * [Use opers that are indispensable in the RG](#use-opers-that-are-indispensable-in-the-rg)
- [Discussion](#discussion)

## Example

So you have an application grammar with the following function:

```haskell
fun WhereIsThing : Thing -> Question ;
```

You've written a linearisation for your application grammar in a language you don't know (we'll just call it L from now on), using the API function `mkQCl`:

```haskell
-- mkQCl : IP -> NP -> QCl    -- who is the man

lincat 
  Thing = NP ;
  Question = QCl ;

lin 
  WhereIsThing np = mkQCl where_IP np ;
```

You show the application-specific sentence, say, *where is the money* to your informant, and they tell you that it's wrong: *money* should be in accusative.

Unfortunately, your second-in-command shoots the informant in frustration, and you don't have access to any other L speakers in reasonable distance. What do you do?

### Ideal: fix the RG

The ideal solution is, of course, to fix all the relevant functions in QuestionL.gf, and if so needed, update the categories of `IP` and `NP`.

But you don't speak L, and your informant didn't leave enough evidence: does the accusative happen with all interrogative pronouns? Do the properties of the NP or the rest of the clause make a difference? After all, you have only witnessed that with the IP *where*, a single NP which is uncountable and definite takes an accusative case, when the sentence is active, positive and in the present tense.

Given these circumstances, it's perfectly understandable that you don't feel comfortable touching QuestionL. Just remember to [open an issue](https://github.com/GrammaticalFramework/gf-rgl/issues) where you describe the problem, so that someone can fix it later.

### Questionable: touch the raw parameters

So you've created an issue, nobody has volunteered to fix it, and you need to urgently interrogate more L speakers.

You've peeked into the resource grammar of L, and you know that `NP` has an `s`
field which is a table of `Case => Str`. So you go for this quick and dirty solution:

```haskell
WhereIsThing np = mkQCl where_IP npInAcc
  where {
    npInAcc = np ** {s = \\_ => np.s ! Acc}
  } ;
```

You're extending the original *np* argument, so it's not the most terrible solution imaginable (elaborated later in this post), but you're still leaving your code vulnerable to any potential changes in the `s` field. If that happens, your grammar stops working and you sure won't find your money.

### OK: find another RGL API function

So, let's look at `WhereIsThing` again.

```haskell
-- Try 1
WhereIsThing np = mkQCl where_IP np ;

-- Try 2
WhereIsThing np = mkQCl where_IP npInAcc
  where { npInAcc = ... } ;
```

 You have been using the `IP -> NP -> QCl` instance of `mkQCl`, but could you use some other API function instead? Look at the options in the [RGL synopsis](http://www.grammaticalframework.org/lib/doc/synopsis/index.html#QCl):

![mkQCl](/images/mkQCl.png "some overload instances of mkQCl")

Maybe `IP -> Adv -> QCl` will do? Then look at [how to construct an Adv](http://www.grammaticalframework.org/lib/doc/synopsis/index.html#Adv), and you find `mkAdv : Prep -> NP -> Adv`.

Now the only thing you need is a `Prep` that only forces out the accusative case but doesn't add a string. If there is not one already, maybe you could add such a instance for `mkPrep` to the `ParadigmsL` module. That's not a huge commitment like changing the entire `QuestionL`, and you get to enjoy the security that your code is safe for future internal changes in the RG for L.

## How to make the questionable less questionable

Despite our best efforts, it's not always possible to find a creative solution that uses only the API .
So here are two techniques I use all the time. Just a couple of disclaimers before we get started:

* If you want to just make your own lexical item and you don't find a constructor in `ParadigmsL`, you can often find it in a module called `MakeStructuralL`, e.g. this one for [English](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/english/MakeStructuralEng.gf)). The techniques I describe are meant for dealing with phrasal categories.
* I expect you to know about record extension in GF. No need to go further
than [this blog](https://inariksit.github.io/gf/2018/05/25/subtyping-gf.html),
if you need to brush up on that.

So, here are two techniques I use all the time.

### Record extension

Say you want to construct your own `NP` from a single `Str` argument. Let’s see first what is the lincat of `NP` in [CatEng](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/english/CatEng.gf#L57):

```haskell
lincat
  NP = {s : NPCase => Str ; a : Agr} ;
param
  Agr = AgP1 Number | AgP2 Number | AgP3Sg Gender | AgP3Pl Gender ;
  NPCase = NCase Case | NPAcc | NPNomPoss ;
  Gender = Neutr | Masc | Fem ;
```

It's unsafe to write an oper like this manually in an application grammar:

<div class="language-haskell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">unsafeNP</span> <span class="o">:</span> <span class="kt">Str</span> <span class="o">-&gt;</span> <span class="kt">NP</span> <span class="o">=</span> <span class="nf">\</span><span class="n">str</span> <span class="o">-&gt;</span> <span class="n">lin</span> <span class="kt">NP</span>
  <span class="err">s = \\_ => str ;</span>
  <span class="err">a = AgP3Sg Neutr</span>
<span class="p">  } ;</span>
</code></pre></div></div>

But we can make it just tiny bit less dangerous by a simple trick: extend some known stable `NP`. You can find a bunch of them [in the synopsis](http://www.grammaticalframework.org/lib/doc/synopsis/index.html#NP), they work for every language!

So let's say that we want our NP to be third person singular. Then we can extend `it_NP` from the API, and only change manually the `s` field.

```haskell
lessUnsafeNP : Str -> NP = \str ->
  it_NP ** {s = \\_ => str} ;
```

Even though `it_NP` is available for all the RGL languages, the part after `**` is still specific to English. For any other language L you want to do this for, you need to check `CatL` to see what is the lincat of `NP` in L. If `NP` in L has an `s` field with a table `Foo => Bar => Baz => Str`, then you need to write this instead:

```haskell
lessUnsafeNP : Str -> NP = \str ->
  it_NP ** {s = \\_,_,_ => str} ;
```

Record extension guards you against changes in the rest of the category you are hacking, but you're still vulnerable for changes in the field you modify manually. For instance, someone notices that most combinations of `Foo` and `Bar` don't happen, and merges them into just one parameter that spells out the combinations explicitly, then `\\_,_,_ => str` is wrong and should be changed for `\\_,_ => str`.

If you want to combine `s` from one `NP` and agreement from another, then you can do it perfectly safely. Here's a concrete use case: you want to parse and linearise `<number> is <NP>`, e.g. "ten is the best grade". If you use `mkNP (mkDet num)`, you will get "ten are the best grade". So you need to force singular agreement into the `NP` constructed from the number.

As long as all the languages in your grammar have a field called `s` in `NP`, this is perfectly safe and works for all languages:

```haskell
lin Num_is_NP_Cl num np =
  let numSg_NP = it_NP ** {s = (mkNP (mkDet num)).s} ;
   in mkCl numSg_NP np ;
```

Note that it would be much less safe the following way:

```haskell
lin Num_is_NP_Cl num np =
  let numSg_NP = mkNP (mkDet num) ** {agr = it_NP.agr} ;
   in mkCl numSg_NP np ;
```

It's unsafe, because unlike `s` field, we have no guarantee that all languages have a field called `agr`.

### Use opers that are indispensable in the RG

1. Get intimate with the resource grammar for L
1. Find or add opers that create and manipulate the relevant category `C` in `ResL`
1. Make RGL functions with goal category `C` use these opers, such that if the category `C` is changed, the opers `makeC` or `modifyC` are also updated.
1. Open `(R=ResL)` in your application and use `R.makeC` and `R.modifyC` in your application grammar when necessary.

For easy categories like `NP`, you're likely to find such functions already. For English, you have a `mkNP` in [ResEng](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/english/ResEng.gf#L174-L182) and [MakeStructuralEng](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/english/MakeStructuralEng.gf#L8-L9). Important for the 3rd point, these functions are [used](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/english/ResEng.gf#L168-L172) in the RG too.

Technically, the 3rd point is not 100 % crucial: even if `ResEng.mkNP` wasn't used anywhere, changing the lincat of `NP` without changing `ResEng.mkNP` would still result in a compilation error. But there is some danger though: maybe the grammarian who changes the English `NP` notices that this function is not used anywhere, and decides to remove it altogether, and then your application grammar which relies on `ResEng.mkNP` breaks.

Let's just look at a bunch of concrete examples.

#### `defaultC : C ;`

This is from the Arabic RG: a default NP with an empty string for `s` and the most unmarked agreement.

```haskell
oper
  emptyNP : NP = {
    s = \\_ => [] ;
    a = {pgn = Per3 Masc Sg ; isPron = False} ;
    isHeavy = False
  } ;
```

This is used in [tons](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/arabic/NounAra.gf#L25) [of](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/arabic/NounAra.gf#L43-L46) [places](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/arabic/NounAra.gf#L179-L183),
where the modifications in the `s` and `a` fields require some more complex computation.

Note that this is a practical design pattern in general, even if you're not planning on using anything outside the API in application grammars.
When you change a lincat in the RG, then you need to change only 1 oper instead of changing 10 lins.

When I discussed this with Aarne, he suggested to use not an empty string for the default C, but rather something that is obviously meant to be replaced. So here's an alternative implementation:

```haskell
oper
  defaultNP : NP = {
    s = \\_ => "HELPIAMTRAPPEDINAGRAMMAR" ;
    a = {pgn = Per3 Masc Sg ; isPron = False} ;
    isHeavy = False
  } ;
```

Then you could find out if something is wrong by doing `ma "HELPIAMTRAPPEDINAGRAMMAR"` in the GF shell, and if you get a hit, you'll see which function is not overriding the default linearisation.

#### `makeC : Str -> C`

We continue with the Arabic RG. There are also derived opers in `ResAra` which use `emptyNP`, such as `indeclNP` and `agrNP`:

```haskell
indeclNP : Str -> NP = \s -> emptyNP ** {s = \\c => s} ;
agrNP : Agr -> NP = \agr -> emptyNP ** {a = agr} ;
```

These are, in turn, used elsewhere, e.g. in [IdiomAra](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/arabic/IdiomAra.gf#L31-L32):

```haskell
-- : NP -> Cl ;        -- there is a house
ExistNP np =
  predVP (indeclNP "هُنَاكَ") (UseComp (CompNP np)) ;

-- : NP -> Adv -> Cl ;    -- there is a house in Paris
ExistNPAdv np adv =
  predVP (indeclNP "هُنَاكَ") (AdvVP (UseComp (CompNP np)) adv) ;
```

Using `indeclNP` in your application is safer than using `emptyNP ** {foo = …}`, because with `indeclNP` you make zero assumptions on what is inside an `NP`. (Well, except that there is a string in some form.) It's hard to come up with a reason why a future grammarian would want to eliminate these functions from the RG, because they also make the resource grammarian's life easier, by having to change things in fewer places.

#### `modifyC : Feature -> C -> C ;`

What if you don't want to make a new `C` from scratch but modify an existing one? Here's my favourite [way of](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/arabic/ResAra.gf#L170-L173) [handling](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/arabic/ResAra.gf#L715-L717) [it](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/arabic/ResAra.gf#L454-L464).

```haskell
-- hack, but better to have it here than define
-- ad hoc in every application grammar /IL
forceCase : Case -> NP -> NP = \c,np -> np ** {
  s = \\_ => np.s ! c
} ;

-- Impersonal verbs have only 1 form (per3 masc sg);
-- ideally, use an impersonal syntactic construction,
-- less ideally, hardcode the forms to verb. /IL
forcePerson : PerGenNum -> Verb -> Verb = \pgn,verb ->
  let gn = pgn2gn pgn in verb ** {
  s = \\vf => case vf of {
                VPerf   v _ => verb.s ! VPerf v pgn ;
                VImpf m v _ => verb.s ! VImpf m v pgn ;
                VImp _g _n  => verb.s ! VImp gn.g gn.n ;
                _           => verb.s ! vf }
  } ;
```

In contrast, none of these is used elsewhere in the Arabic RG. I'm just relying on my comments to convey to future Arabic resource grammarians why they should be kept.

## Discussion

### Isn't this difficult?

If you noticed, the first step was "Get intimate with the resource grammar for L". This might seem like a daunting task, but you still don't need to learn L the language to look at its resource grammar. You might be only interested in modifying one category in your application, and then it's enough to find (or create!) `defaultC`, `makeC` and `modifyC` for that one category. Shocking news really, in order to use low-level hacks and bypass the API, you need to *know* the RG of the language beyond the API.

If you do explore the RG and introduce these functions, don't be shy to [replace repetitive code](https://github.com/GrammaticalFramework/gf-rgl/commit/2eefb1b74b1d0d707c1e8b299b899b75fc18ebda#diff-a6365eec7d770ccbeb3014a994bc0137) you might find in that resource grammar itself. For instance, here I have added [mkOrd](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/japanese/ResJpn.gf#L151-L159) into `ResJpn` and use it in [three](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/japanese/NounJpn.gf#L159) [functions](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/japanese/SymbolJpn.gf#L36) elsewhere. I didn't do anything else to learn Japanese, I just copied and pasted the code that was previously repeated in 2 places and called it `mkOrd`. In that case, my motivation was to fill SymbolJpn and that was just a byproduct, because I didn't want to copy and paste the code for the third time. But if I'd want to use something like that in my application, having an oper `mkOrd` is way better than writing `pred = \\st,t,p => symb.s ++ "番目" ++ mkCopula.s ! st ! t ! p` by hand.

How to make sure you don't change the behaviour of a language you don't know? You can use [gftest's `-o` feature](https://github.com/GrammaticalFramework/gftest/blob/master/README.md#compare-against-an-old-version-of-the-grammar--o) to make sure you didn't break things. It's saved me from serious errors so many times, I might just write a blog post to list the most atrocious stupidities I was about to commit, if `gftest` hadn't caught them for me.

### Isn't this still questionable?

Hot take: `defaultC` and `makeC` style opers are great and make everyone's life better, resource and application grammarians alike. `forceFeature` can be a sign that someone somewhere could've designed things better, but it's way better than having to stand your grammar outputting things you don't want it to.

Out of the examples, `forcePerson` is pretty fine and there are legit use cases. Say that you need to express a concept like *$AGR faints* as *it blacks out on $AGR*. All you need is to have a possibility for non-nominative subject case in `V`, and a possibility to force all inflection forms into 3rd person singular.
If you expect vanilla RGL to translate "I like GF" as "me gusta GF" (too bad [it doesn't](https://github.com/GrammaticalFramework/gf-rgl/blob/master/src/spanish/README.md#known-issues), because *gustar* actually agrees with the object and the inflection tables are just one-dimensional), then using `forcePerson` is not very controversial. We could totally do it for intransitive verbs in the Spanish RG already.

Why is deviating subject case and incomplete paradigm okay, but *remove $AGR's clothes* bad? Well, consider again that we are incorporating a) an *inflecting noun* and b) a *inflecting pronoun* into a `V`. As it happens in Finnish, and certainly similar things happen in other languages, the noun gets a different case in active sentences vs. passive sentences, and in negative sentences vs. positive sentences. As it also happens, some of these things are expressed periphrastically and thus built at the `Cl` level, so we can't ensure the right case for *clothes* just by inserting it in the verb's inflection table. So hacking `VP`/`Cl` level stuff into `V` will never be supported in the common RGL API, because that must work for all languages.

If `V` and `Cl` have identical inflection tables in your language, then it's safe to add anything you like, and it won't introduce ungrammatical sentences. If you know you're working on languages with very little morphology, honestly, do whatever you want. If you find that you're using such hacks all the time and it's become a standard practice, you could even add them to `ParadigmsL`[^2], because Paradigms are totally language-specific; then you need to import fewer modules. I support anything that makes grammarians' lives easier *and* doesn't produce horribly wrong sentences. Just don't expect that all languages can copy that design, and be prepared to change lincats if you ever add more languages to your application.

## Footnotes

[^1]: I'm not saying there aren't legit reasons to not change the abstract syntax, like you work for an evil tyrant who will throw you into a fiery pit if you dare to suggest that their abstract syntax is less than perfect. Who am I to moralise others from my privileged position where my boss is literally Aarne Ranta. I wish you luck in job hunt in this imperfect world, and I hope we one day eradicate all suffering and injustice.

[^2]: Possibly [not visible in the synopsis](https://groups.google.com/forum/#!searchin/gf-dev/overloads%7Csort:date/gf-dev/JL_5H92BdIo/DP6rhKmQFgAJ) if you feel self-conscious among those resource grammarians who happened to be born with a native language that has a 2-digit number of cases.
