---
layout: post
title: "`using` killed the `FUN` on GCC 5"
modified:
categories: programming
excerpt: "An anecdote about surprising implementation challenges of `FUN`."
tags: [c++, variant, using-declaration, FUN, gcc]
date: 2017-08-08
share: true
---

> An anecdote from benchmarking the compile time of [mpark/variant],
> an implementation of `std::variant` for C++11/14/17.

## Introduction

`std::variant` has a converting constructor that looks like this:

```c++
template <typename... Ts>
class variant {
  template <typename T>
  constexpr variant(T &&);
};
```

For this constructor, we need to determine which of the `Ts...` will be
initialized with `T`. For `std::variant`, the behavior is defined as follows:

[`template <class T> constexpr variant(T&& t) noexcept(see below);`](http://eel.is/c++draft/variant#ctor-12)

> Let `Tj` be a type that is determined as follows: build an imaginary
> function `FUN(Ti)` for each alternative type `Ti`. The overload `FUN(Tj)`
> selected by overload resolution for the expression
> `FUN(std::forward<T>(t))` defines the alternative `Tj` which is the type
> of the contained value after construction.

For example, given `variant<int, std::string> v("hello");`, we want to create
an overload set `FUN` with `int` and `std::string` then manually perform
overload resolution with an argument of type `const char (&)[6]`.

The following is the initial approach I took for `FUN`:

```c++
template <typename Head, typename... Tail>
struct FUN;

// Base case.
template <typename Head>
struct FUN<Head> {
  Head operator()(Head) const;
};

// Recursive case.
template <typename Head, typename... Tail>
struct FUN : FUN<Tail...> {
  using FUN<Tail...>::operator();
  Head operator()(Head) const;
};
```

Then the winner of overload resolution can be determined using `FUN` like this:

```c++
using best_overload = std::result_of_t<FUN<int, std::string>(const char (&)[6])>;
```

The compile-time benchmark of `variant`'s converting constructor using this
implementation looked like this on GCC 5:

<figure>
  <img src="/images/bench.png" alt="Compile-time Benchmark">
</figure>

A [minimal example on Wandbox] shows (via `-ftime-report`) that the cost of
using this particular implementation of `FUN` at `N = 350` is __~20s__!

What's so expensive...?

[mpark/variant]: https://github.com/mpark/variant
[minimal example on Wandbox]: https://wandbox.org/permlink/rcmDjQCaF311MvkB

## Determining the Source

The first thing I found is that [commenting out the `using` declaration] from
the above example takes the compile time down to __~0.15s__. So the recursive
inheritance seems fine, and the `using` declaration could be the issue,
but by not pulling in the parent `operator()` we're also no longer performing
overload resolution. Could it be the cost of overload resolution?

Well, the [non-recursive, hand-rolled version of `FUN`] which does perform
overload resolution takes __~0.1s__. Okay, it must be the `using`
declaration.

Now what...?

[commenting out the `using` declaration]: https://wandbox.org/permlink/ex7gpsGblVzC6Ahb
[non-recursive, hand-rolled version of `FUN`]: https://wandbox.org/permlink/H0GzXu1mP7mF0zvt

## Surrogate Call Function

When considering the overload set for a call to object of a class type,
it's not just the `operator()`s that get involved in the action.

For example, an implicit conversion operator to `R (*)(P1, ..., PN)` is also
considered. They participate in overload resolution as _surrogate call function_
with the unique name _call-function_ and has the form:

```c++
using F = R (*)(P1, ..., PN);

R call-function(F f, P1 a1, ..., PN aN) { return f(a1, ..., aN); }
```

This is an example taken basically verbatim from [[over.call.object]]:

```c++
int f1(int);
int f2(float);

typedef int (*fp1)(int);
typedef int (*fp2)(float);

struct A {
  operator fp1() { return f1; }
  operator fp2() { return f2; }
} a;

int i = a(1);  // calls f1 via pointer returned from conversion function
```

How does this help us?

[over.call.object]: http://eel.is/c++draft/over.call.object#4

## What's in a Name?

Why did we need the `using` declaration in the initial approach, anyway?
It's due to the fact that the __name__ of the overloaded function in
that case was `operator()`, and `FUN<T, Ts...>::operator()` __shadowed__
`FUN<Ts...>::operator()`. So we needed to pull the
`FUN<Ts...>::operator()` name explicitly into `FUN<T, Ts...>`'s scope.

With surrogate call functions, the "pulling in" happens implicitly:

> Similarly, surrogate call functions are added to the set of candidate
> functions for each non-explicit conversion function declared in a base
> class of `T` provided the function is not hidden within `T` by another
> intervening declaration.

As such, the "name" is essentially the type of the pointer to function.

Now, we can implement `FUN` like this:

```c++
template <typename Head, typename... Tail>
struct FUN;

// Base case.
template <typename Head>
struct FUN<Head> {
  using F = Head (*)(Head);
  operator F() const;
};

// Recursive case.
template <typename Head, typename... Tail>
struct FUN : FUN<Tail...> {
  using F = Head (*)(Head);
  operator F() const;
};
```

With [this implementation](https://wandbox.org/permlink/KQm3wbkCufSvOmoJ),
at `N = 350` we get __~0.6s__! MUCH better than __~20s__.

## Final Touch with Multiple Inheritance

Since the surrogate call functions are implicitly pulled in from the base
classes, we don't even need to keep the recursive inheritance anymore.

We can flatten, simplify, and improve the implementation with multiple
inheritance:

```c++
template <typename T>
struct FUN_leaf {
  using F = T (*)(T);
  operator F() const;
};

template <typename... Ts>
struct FUN : FUN_leaf<Ts>... {};
```

Now, with [this implementation](https://wandbox.org/permlink/wJk5gmghjNMl0tBq),
at `N = 350` we get about __~0.2s__! Not as drastic, but still a ~3x improvement.

## Final Remarks

This issue was discovered while I was setting up [MPark.Variant/Benchmark] to
compare the many `variant` implementations out there.

My hope is that this website will help to improve the community-wide
implementation quality of `variant` by comparing existing implementations.

Take a look at the latest [GCC 5 results] to see the improvements that
have been made to [mpark/variant]!

Thanks to [Agustín "K-ballo" Bergé] for introducing / explaining
surrogate call functions to me.

Thanks to [Louis Dionne] and [Bruno Dutra] for developing the [metabench]
framework and setting a precedent with [metaben.ch]!

[MPark.Variant/Benchmark]: https://mpark.github.io/variant
[GCC 5 results]: https://mpark.github.io/variant/compile_time/gcc-5/ctor.fwd/index.html
[Agustín "K-ballo" Bergé]: https://github.com/K-ballo
[metabench]: https://github.com/ldionne/metabench
[metaben.ch]: http://metaben.ch
[Louis Dionne]: https://github.com/ldionne
[Bruno Dutra]: https://github.com/brunocodutra
