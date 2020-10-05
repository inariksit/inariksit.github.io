---
layout: post
title:  "Real World GF"
date:   2020-10-30
categories: gf
tags: "gf programming"
---

I was happy when bars started selling bad vegan pizza. Not because my favourite food is dried out soy chunks on top of indeterminate mush, but because that means that vegan food is becoming a normal thing.
Contrast this with a conference dinner that was literally raw zucchini in fancy shapes and topped with seeds and stuff. The artistic value of the zucchini may be higher, but I'm also interested in being nourished, and knowing that I can get pizza anywhere, or at most move to the next door, instead of walking to the other side of the town to find the only vegan pizzeria.

So imagine my delight when I've started seeing bad GF design in the wild. This tells me that GF is moving from the zucchini stage to the bad pizza stage. Actually this metaphor doesn't make any sense, but I will make the example grammar to be about zucchinis and pizzas, so that you think it looks coherent.


## Toy NLG application

![nlg_pizza_order](/images/pizza-order-interface.png "An interface for ordering pizza")

Suppose you have an application as shown above. You can choose the dish, and then choose specifics of the dish: a pizza has toppings, a lasagna has fillings and so on. After you have chosen your dish, an order confirmation is generated as a natural language sentence: "Your pizza has [topping1] and [topping2]".

The natural language sentence is generated with a GF grammar. This grammar contains the vocabulary, such as "pizza", "lasagna" and "zucchini", as well as the sentence structure for "[food] contains [ingredient]".
All the logic for clicking and displaying things to the user is written in some general-purpose programming language. This includes also rules to construct a GF tree that matches the user's selection.

![nlg_pizza_order](/images/pizza-order-interface2.png "GF tree for the order confirmation")

<!-- As we know from [an earlier post](https://inariksit.github.io/gf/2019/12/12/embedding-grammars.html), such an application can be written in any language that has support for the PGF library. -->

## Abstract syntax

Suppose that the application is slightly more advanced than this. Maybe you can even describe the foods' properties.

```
abstract Foods = {

  cat
    Comment ; Item ; Kind ; Quality ;

  fun
}



It's important to keep your abstract syntax flexible.
