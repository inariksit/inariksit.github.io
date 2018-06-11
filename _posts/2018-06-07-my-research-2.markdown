---
layout: post
title:  "My research, part II"
date:   2018-06-07
categories: research
tags: research cg
---


[Part I](https://inariksit.github.io/research/2018/06/06/my-research-1.html)

In a parallel universe, there was another grammar formalism called
[Constraint Grammar](http://visl.sdu.dk/constraint_grammar.html)
(CG). While I hardly wrote any such grammars, I was dating someone who
wrote them, and at the same time, I was supervised by someone who had
an obsession to SAT-solvers. So the two worlds met through me, and
thus was a new research question born.

## Writing CG

If you are reading this page, you supposedly know English. Thus you
would have no trouble determining that the word *wish* is a noun in
the context "a wish", and a verb in the context "I wish my thesis was
finished already".

Constraint Grammar is the formalisation of this knowledge, in the following format:

```haskell
SELECT noun IF (0 noun/verb) (-1 article) ;
```

These rules work on a text that is previously analysed
morphologically: that is, some other piece of software has gone
through the words one by one, and given them all possible tags they
can get. Initially, all instances of *wish* get both tags, `noun` and
`verb`. After applying all the rules, hopefully each *wish* is left
with only one tag. If we're lucky, it is even the correct one.

As an important distinction from other grammar formalisms you might
know, CG grammars are inherently *heuristic*, not absolute. They are
born from the notion that all grammars leak, and that we are not
supposed to define all and only valid sentences in the language. We
just want something that works! Real language that humans write is not
perfect--even if you write "✱the boy drink beer", we would still want
to analyse it. The word *drink* is still a verb, just conjugated
incorrectly.

Writing CG rules<sup><a name="footnote1" href="#footnote">1</a></sup>
has a more practical and low-level feel to it than writing GF
grammars. I'll be happy to be proven wrong, but I feel that actual
knowledge of the language is more crucial for CG. Every rule should
reflect some actual sentence that appears in an actual text, and is
not given a proper analysis before you wrote the rule. A lot of CG
rules include a comment of an example sentence where it works, or
alternatively, where it doesn't work.  That's right--CG grammarians
know that their rules fail for some sentences, and *include them
anyway*. What's the deal with that?

The deal (at least in the current mainstream CG implementation VISL
CG-3) is *ordering*. The rules are applied to the sentences in the
order they appear in the file. More reliable and efficient rules are
placed in the beginning, and more dubious rules at the end. Thus
we'd hope that when the dubious rule gets its turn to act, the falsely
triggering sentences are already disambiguated by an appropriate rule,
and are thus safe from the dubious rule.

Does that sound complicated?  Previous experiments show a difference
of a few percent points in precision and recall just by changing the
order (Bick, Lager, Nivre) and/or execution strategy (myself but with
really shitty grammars so take it with a grain of salt). I'd be pretty
annoyed if I wrote CG regularly. I just want to formalise my knowledge
about language, not keep track of dependencies of thousands of rules.
I only wish someone had done something about this!

## Testing CG

So we have a bunch of rules, some of which may conflict in one order,
but be fine in another order. Some rules may also be inherently
contradictory no matter the order (think "remove all nouns after
article" vs. "select all nouns after article"). In addition, some
rules can be internally contradictory: "select noun only if it *isn't*
a noun".

We could apply all the rules in sequence to a corpus, and see if a
rule never triggers. But it may be just that the corpus is incomplete,
and doesn't contain just the sentence needed to trigger that
particular rule. Even if we found such a rule that never triggers, we
wouldn't know which rule(s) are those that conflict with it.
To solve these problems, we use *symbolic evaluation*.

We construct an initial sentence of some width, e.g. 10 words. We call
this a *symbolic sentence*. Initially, all symbolic words in the symbolic
sentence contain *all possible analyses*--hundreds or thousands,
depending on how granular the morphological analyser is and how
complex the language is. (In my experience, that's the order of
importance). We start applying the rules one by one to the symbolic
sentence, but instead of rules directly removing or selecting
analyses, they form logical formulas. Take the rule we know from before:

```haskell
SELECT noun IF (0 noun/verb) (-1 article) ;
```

Now in order for that rule to apply in a particular symbolic word, it
has to have 3 things:

* At least one previous word (so we cannot apply it to the first word);
* Previous word has to have an `article` reading;
* The word itself has to have both `noun` and `verb` tags.

We form these requirements for every single word in the symbolic
sentence. It is very much expected that not every rule will be able to
apply to every single word---say that we apply next a rule that
requires `verb` as a context, and thus there has to be at least one
word in the sentence where `verb` reading is intact. It would be easy
to just find a solution where e.g. previous word has no `article`
reading, or the word itself has no `noun` reading to start with, so by
the rule, we are not obligated to remove `verb`.

But what if two rules are really conflicting (or just in wrong order)?
Then it becomes impossible to form a sequence where both rules may
apply! Thanks to this analysis, I know exactly if two (or more) rules
are in conflict, and I can try to change their order or remove one
altogether.

That sounds a bit underwhelmingly simple. More detail is available in
[my licentiate thesis](https://listenmaa.fi/lic.pdf)---but it will be
also in a more compact form in my PhD thesis, and with more fancy
results. So unless you're like my supervisor and have a fetish for
SAT, why don't you rather just chill over the summer and read my shiny
PhD thesis as soon as I've written it.

## Doing weird things with CG

I said earlier that CG was not conceived as defining "all and only the
grammatical sentences in a language", but just to do the job and
assign some tags even for a grammatically incorrect text. But the
mechanisms that we developed in order to test CG have interesting side
effects, that let us model CG as a *generative* formalism.

If we were sculpting a statue, generative formalism would start from
an empty place and start splashing some clay around. In contrast, a
reductionist formalism would start from a heavy stone block and carve
away everything that is not part of the statue. In the end, both have
formed an identical statue. An empty generative grammar would output
just thin air, and empty reductionist grammar would output an intact
stone block.

We can see symbolic sentences as these stone blocks. To give a classic
example, say we have the alphabet `{a,b}` and the language
`aⁿbⁿ`. Then we can construct a CG grammar, apply it to an even-length
symbolic sentence consisting of `a`s and `b`s, and have it output a
string where the first half is `a`s, followed by a second half of
`b`s. This language is context-free, and we have only found a method
to write such a grammar manually, but we have a method that can
translate any regular language into a corresponding CG automatically,
and apply it to a symbolic sentence.

<!-- Bick and Didriksen describe CG as "a declarative whole of contextual
possibilities and impossibilities for a language or genre". -->

If you want to read more, here's Wen's [blog post](https://wenkokke.github.io/2016/constraint-grammar-can-count/).
There will also be a section in my thesis on this.

Part III coming up sometime.

---

<a name="footnote" href="#footnote1">1</a>: I once wrote 19 rules to prove a point. I
wanted to get to 20, but the 19 already worked better (on a specific
corpus) than 300 (probably hand-tailored for another corpus) and I was
bored so it became 19. But believe me, I have *read* a whole lot of CG
rules!

