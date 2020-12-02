---
layout: post
title:  "Subtyping in GF: practical examples"
date:   2018-05-25
categories: gf
tags: gf
---




## What is subtyping

`S <: T`

We say:

- ‘S is a subtype of T’
- ‘T is a supertype of S’

Terms of type `S` are more *informative*, for example:

`{s : Str ; b : Bool} <: {s : Str}`

We can use `S` anywhere we could use `T`. In other words, if we want
to do something with an `s : Str`, we don't mind that the extra field
`b : Bool`, we can just ignore it.

See also the GF reference manual
[entry on subtyping](http://www.grammaticalframework.org/doc/gf-refman.html#toc41).
The most important thing for GF syntax is the *record extension* `**`, as follows:


```haskell
oper
  -- Record extension for types
  A : Type = {s : Str} ;
  B : Type = A ** {s2 : Str} ; -- {s,s2 : Str}

  -- …and for terms
  a : A = {s = "a"} ;
  b : B = a ** {s2 = "b"} ;    -- {s = "a" ; s2 = "b}
```

## Examples


### Reuse `oper`s

A common use case is to reuse a single `oper` for several `lin`s, even
though they are of different type (but share a supertype!)
Consider VP, VPSlash and ClSlash: VP has a verb and 0-n complements,
depending on the verb subcategory (intransitive, transitive, sentence
complement, …). VPSlash may also have any number of existing
complements but crucially, it is missing a complement. ClSlash is a
clause missing a complement: just like VPSlash, but it also has a
subject.

Here are types for VP, VPSlash and ClSlash in an unnamed RGL
language<sup>[1](#ok-its-basque)</sup>:


```haskell
oper
  VPhrase : Type = {- details don't matter -} ;
  VPhraseSlash : Type =
    VPhrase ** {post : Postposition ;
                missing : MissingArg} ;
  ClauseSlash : Type =
    VPhraseSlash ** {subj : NounPhrase} ;

lincat
  VP = VPhrase ;
  VPSlash = VPhraseSlash ;
  ClSlash = ClauseSlash ;
```

And here is the oper `insertAdv`, which inserts an adverb into a `VPhrase`.

```haskell
oper
  insertAdv : VPhrase -> {s : Str} -> VPhrase = \vp,a ->
    vp ** {adv = vp.adv ++ a.s} ;
```

We can reuse the same oper `insertAdv` for all subtypes of `VPhrase`:

```haskell
lin
  -- : VP -> Adv -> VP ;            -- sleep here
  AdvVP = insertAdv ;

  -- : VPSlash -> Adv -> VPSlash ;  -- use (it) here
  AdvVPSlash vps adv = vps ** insertAdv vps adv ;

  -- : ClSlash -> Adv -> ClSlash ;  -- (whom) he sees today
  AdvSlash cls adv = cls ** insertAdv cls adv ;
```

`AdvVP` is straightforward: the lincats of `VP` and `Adv` match
exactly `VPhrase` and `{s : Str}`.
But AdvVPSlash and AdvSlash manipulate subtypes of `VPhrase`--how
does that work?

`insertAdv` *takes* a `VPhrase`, so it can also take
any of its subtypes, such as `VPSlash` and `ClSlash`. Thus calling
`insertAdv cls adv` is perfectly fine.

`insertAdv` also *returns* a `VPhrase`, but `AdvVPSlash` and
  `AdvSlash` expect more informative types. Thus the following would
  be wrong:

<div class="language-haskell highlighter-rouge"><div
class="highlight">
<pre class="highlight"><code><span class="err">ΑdvSlash cls adv = insertAdv cls adv ;</span></code></pre></div></div>


Why? Because `insertAdv adv cls` returns a `VPhrase`, but `AdvSlash`
needs to return a `ClSlash`. The `subj`, `post` and `missing` fields were
already present in the argument `cls`, and inserting an adverb doesn't
change that, so we add `cls ** ` before calling `insertAdv`: this
*extends* the original `cls` by the changes made by `insertAdv`. We
could be more verbose and write the same thing like this:

```haskell
AdvSlash cls adv =
  insertAdv cls adv ** {subj = cls.subj ;
                        post = cls.post ;
                        missing = cls.missing} ;
```

### Interlude

Here's an exercise: what subtype relations of `Det ⁇ Idet` and `NP ⁇
IP` make the following code work?

```haskell
lin
  -- : Det -> CN -> NP          -- the songs
  DetCN det cn = {- actual implementation here -} ;

  -- : IDet -> CN -> IP ;       -- which five songs
  IdetCN = DetCN ;
```

(Obviously, `Det = Idet` and `NP = IP`, but what if they have to be different?)

We [return to this question](#interlude-2), and meanwhile I explain that by the way,
we can also reuse `lin`s, not just opers!

### Reuse `lin`s


```haskell
oper
  insertComp : Comp -> VPhrase -> VPhrase = \c,vp ->
    vp ** {comp = {- details don't matter -}} ;

lin
  -- : VV  -> VP -> VP ;
  ComplVV vv vp =
   let vcomp : Comp = mkComp vp ;
    in insertComp vcomp (useV vv) ;

  -- : VV  -> VPSlash -> VPSlash ;
  SlashVV vv vps = vps ** ComplVV vv vps ;
```

Look, we just reused `ComplVV` for implementing `SlashVV`! As before,
`ComplVV` returns a *less* informative type than what `VPSlash` needs,
so we need to prefix the result with `vps **` to keep all the necessary
fields of the `vps`.
But since `ComplVV` is a `lin` and not an `oper`, it adds some hidden
`VP`-specific junk to the record (namely, a *lock field*). However,
VP-with-a-lock-field is simply a subtype of VP-without-a-lock-field,
so `SlashVV` doesn't care! It can just happily use the relevant
fields from the result of `ComplVV vv vps`.

Now, let's talk about lock fields in more detail.

## Lock fields

You may have seen GF compiler output the following messages:

* Warning: ignoring lock fields in resolving `<some expression>`
* Error: X not in lincat of Y: try wrapping it with `lin Y`

And if you've looked around in the resource grammars, you may have seen this kind of pattern in several places:

```haskell
oper
  Noun : Type = {s : NForm => Str ; g : Gender} ;

lincat
  N  = Noun ;
  N2 = Noun ** {c2 : Preposition} ;
  N3 = Noun ** {c2,c3 : Preposition} ;
```

So here we define `N2` and `N3` as subtypes of `N`, except that we do
it in an awkward way, extending an *oper* `Noun` instead of the lincat
`N`.

When you write a lincat for some cat, you make it into some concrete
record type. You can use the same concrete record for many different
lincats, e.g. `lincat Adv, Conj = {s : Str} ;`.
To you these are identical, but the GF compiler inserts a hidden field,
so actually `Adv` is `{s : Str ; lock_Adv : {}}` and Conj is `{s : Str ; lock_Conj : {}}`.

That's why we don't write `lincat N2 = N ** {c2 : Prep}`: it would
make `N2` into the following type:

```haskell
{s : NForm => Str ; g : Gender ; lock_N : {} ;
 c2 : Prep ; lock_N2 : {}} ;
```

This is also the reason why you might have seen `lin X` in the
paradigms, like in the following:

```haskell
oper
  mkNoun : Str -> Noun = {- details don't matter -} ;

  mkN  : Str -> N = \s -> lin N (mkNoun s) ;

  mkN2 : Str -> Prep -> N2 =
   \s,prep -> lin N2 (mkNoun s ** {c2 = prep}) ;

  mkN3 : Str -> Prep -> N3 =
   \s,p,r -> lin N3 (mkNoun s ** {c2 = p ; c3 = r}) ;
```

The oper `mkNoun` creates a record with fields `{s : NForm => Str ; g : Gender}`.
Since `N`, `N2` and `N3` all share those two fields, it makes sense to
reuse the common parts. Then we make the resulting `Noun ** {c2 :
Prep}` into an actual `N2` by wrapping it in `lin N2`.

```haskell
        mkNoun s ** {c2 = prep}  -- : Noun ** {c2:Prep}
lin N2 (mkNoun s ** {c2 = prep}) -- : N2
```
<!--
TODO: include this gotcha.

Consider an oper like this:

```haskell
oper compoundN : Str -> N -> N ;
```
This is the expected use of such an oper.

```haskell
lin foobar_1_N = compoundN "foo" bar_N ;
```

But if we have more lincats that are the same as N's lincat, we can abuse the `compoundN` oper and apply it to them.

```
lincat N, CN = ResXxx.Noun ;
lin foobar_2_N = compoundN "foo" (mkCN bar_N) ;
```

Why is that? Because some parts of GF actually ignore the lock fields. But this is actually sensitive to the types being otherwise 100% same. If we do this, then `foobar_2_N` stops working.

```
lincat
  N = ResXxx.Noun ;
  CN = ResXxx.Noun ** {someField : Str} ;
lin foobar_2_N = compoundN "foo" (mkCN bar_N) ; -- this doesn't work
```

As long as N is the supertype of CN, it can still work if we just annotate the CN explicitly: "treat this as if it was a N".

```
lincat
  N = ResXxx.Noun ;
  CN = ResXxx.Noun ** {someField : Str} ;
lin foobar_2_N = compoundN "foo" <mkCN bar_N : N> ; -- this should work (TODO test)
```

-->

## Interlude 2

Now we get back to the [earlier question](#interlude): which subtype relations hold between `Det ⁇ Idet` and `NP ⁇ IP` (other than =)?

```haskell
lin
  -- : Det -> CN -> NP          -- the songs
  DetCN det cn = {- actual implementation here -} ;

  -- : IDet -> CN -> IP ;       -- which five songs
  IdetCN = DetCN ;
```

* We can *give* an `Idet` as an argument to `DetCN`, which expects a `Det`. Thus `Idet` has to have *more* fields than `Det`.
* We can *return* an `NP` as the result of `IdetCN`, which expects to return an `IP`. Thus `NP` has to have more fields than `IP`.

Here we have the answer: `IDet <: Det` and `NP <: IP`.

## Further reading

How about subtypes on abstract syntax? Here's a [presentation by Hans Leiß](http://school.grammaticalframework.org/2015/slides/hans-gf-subtyping.pdf).


---

<a name="ok-its-basque">1</a>: I believe in the pedagogical principle of "don't tell
people what you're teaching them is complex, so they won't be scared of it".
