---
layout: post
title: "Improvement vs Consistency"
excerpt: "Is it better to partially improve at the cost of inconsistency?"
modified: 2016-05-11
categories: programming
tags: [c++, lambda, mutable]
---

The _lambda expression_ is one of my favorite C++11 features next to
_variadic templates_. I won't talk about the lambda expressions in particular
in this post, but I want to talk about the `mutable` keyword in the context of
lambda expressions.

## `const` by Default

C++'s lambdas are technically closures, since it has the ability to capture
(save) the surrounding context. Similar to how we need to decide whether to pass
function arguments by value or by reference, we need to decide whether we want
capture by value or by reference. If we capture by value, the saved context is
now part of the lambda's state and we need to decide whether the `operator()` is
allowed to modify it or not. Let's see an example:

```c++
int n = 42;
auto good = [n]() mutable { n = 45; };  // ok, mutable lambda.
auto bad = [n]() { n = 45; };  // error: cannot assign to a variable captured by
                               //        copy in a non-mutable lambda
```

As you can see, the default for lambdas is to be _non-mutable_, or `const`.
That is, a lambda needs the `mutable` keyword to be able to modify its internal
state.

## This is New...

Throughout C++, the language consistently chooses mutable by default. For
example, variable declarations (local, member, etc) and member function
declarations need `const` tacked onto them in order to make them _non-mutable_.
The fact that lambdas have the opposite default behavior is new to the language.

## Partial Improvement

The improvement here is that `const` by default is actually the better choice.
I suspect that this is a lesson learned from functional languages such as ML and
OCaml which provide immutable values by default. As a result, we get
_referential transparency_ more often which helps programmers and compilers to
reason about a program. Andrej Bauer also talks about this topic in his blog
post, [On programming language design], in which he says:

> We should design the language so that the default case is an immutable value.
> If the programmer wants a mutable value, he should say so explicitly.

We also don't want to be restrained by the decisions made in the past, of course
we want to keep on improving... but we still need backwards-compatibility.

## But... Consistency...

I have a big thing for consistency. I think keeping consistency in a language
makes it so that there's less to remember and therefore easier to learn and use.
This is no different in programming languages than human languages such as...
English. English has a __ridiculous__ number of special cases to the point where
one has to learn the language almost on a phrase-by-phrase basis. Learning a
pattern and applying it consistently will certainly lead you to sound like a
foreigner in no time.

I think the language becomes __unnecessarily__ complicated and harder to teach
when we introduce special cases like these.

## So which is better?

I've argued both sides against myself and there doesn't seem to be a definitive
answer. I want to be able to make improvements going forward, but I'm tired of
saying and also hearing "They're the same, pretty much. They're almost always
the same. Um, one case where it's different..."

In this specific case, I think I would've preferred to keep consistency.
The committee members voted otherwise however, so who am I to disagree.

[On programming language design]: http://math.andrej.com/2009/04/11/on-programming-language-design/
