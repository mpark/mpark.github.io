---
layout: post
title: "constexpr function parameters"
modified:
categories: programming
excerpt: "Can we work around the limitations of non-type template parameters?"
tags: [c++, constexpr, parameters, metaprogramming]
date: 2017-05-26
share: true
draft: true
---

__NOTE__: Welp, this post is about 3 years late. But... it still seems to come
up from time to time and the technique (trick?) doesn't seem to be widely known,
so I figure I should write it down.

## Introduction

Since C++11, functions can be marked `constexpr`. What does this really mean?
Well, it specifically does __not__ mean that it __will__ be evaluated at
compile-time. It actually means that it is __possible__ for it to be evaluated
at compile-time.

Why is this important? Well, it means that the function needs to be written in
such a way that the parameters can be compile-time __or__ runtime values.

Consider the following contrived example:

```c++
constexpr void f(std::size_t n) {
  static_assert(n == 42, "");  // not allowed.
}
```

This function is not allowed because `n`  __could__ be runtime value, in which
case it would violate the requirement that `static_assert` expects a constant
expression.

[constexpr]: http://en.cppreference.com/w/cpp/language/constexpr

## Non-Type Template Parameter

The example above is pretty simple to fix though, since we can just make the
function parameter be a non-type template parameter.

```c++
template <std::size_t N>
constexpr void f() {
  static_assert(N == 42, "");
}
```

Simple enough. But what if it's a slightly more complex type, like
`std::tuple<int, int>`? Maybe something like this...?

```c++
template <std::tuple<int, int> X>  // not allowed.
constexpr void f() {
  static_assert(std::get<0>(X) == 101, "");
  static_assert(std::get<1>(X) == 202, "");
}
```

Well, this doesn't work because we're not allowed to pass an arbitrary literal
type as a non-type template parameter.

> [N3413][N3413] proposes to allow arbitrary literal types for non-type template
> paramters.

Can we get around this limitation?

[N3413]: http://open-std.org/JTC1/SC22/WG21/docs/papers/2012/n3413.html

## Value-Encoded Type

The [`std::integral_constant`][integral_constant] class template takes a type
and a value of that type. It essentially encodes the value as part of the type.

[integral_constant]: http://en.cppreference.com/w/cpp/types/integral_constant

This isn't directly helpful for us though, since the value still needs to meet
the requirements of a non-type template parameter. However, the idea of shoving
a value inside the type remains useful. We simply need a way to encode the value
inside the type without the value being __part__ of the type.

Okay, I admit that may be a bit confusing. Here is the code:

```c++
struct  {
  static constexpr auto value() { return std::make_tuple(101, 202); }
};
```

Here, we've created a type `S` that encodes the value without the value actually
being part of the type. The `tuple(101, 202)` above is not _really_ part of the
type `S`, but it __is__ encoded in it.

We can then pass it into a function and use it like this:

```c++
template <typename X>
constexpr void f(X) {
  static_assert(std::get<0>(X::value()) == 101, "");
  static_assert(std::get<1>(X::value()) == 202, "");
}

int main() {
  struct S {
    static constexpr auto value() { return std::make_tuple(101, 202); }
  };
  f(S{});
}
```

## Macros

I don't like macros either. But at least this is a pretty simple one.

```c++
#define CONSTANT(...) \
  struct { static constexpr auto value() { return __VA_ARGS__; } }
```

The above example can then be written like this:

```c++
template <typename X>
constexpr void f(X) {
  static_assert(std::get<0>(X::value()) == 101, "");
  static_assert(std::get<1>(X::value()) == 202, "");
}

int main() {
  using S = CONSTANT(std::make_tuple(101, 202));
  f(S{});
}
```

We could also have a version that returns an instance of such a type so
that we don't have to have a preceding using-declaration every time.

```c++
#define CONSTANT_VALUE(...) \
  [] { using R = CONSTANT(__VA_ARGS__); return R{}; }()
```

With this we could pass it directly to the function:

```c++
template <typename X>
constexpr void f(X) {
  static_assert(std::get<0>(X::value()) == 101, "");
  static_assert(std::get<1>(X::value()) == 202, "");
}

int main() {
  f(CONSTANT_VALUE(std::make_tuple(101, 202)));
}
```

## Compile-Time String

Compile-time strings in the context of static reflection and metaprogramming are
a point of discussion in the C++ committee. Meanwhile, this macro can be used to
capture and pass around compile-time strings, or even a
[`std::string_view`][string_view].

[string_view]: http://en.cppreference.com/w/cpp/string/basic_string_view

```c++
#include <string_view>

template <typename StrView>
constexpr void f(StrView) {
  static_assert(StrView::value() == "hello", "");
}

int main() {
  using namespace std::literals;
  f(CONSTANT_VALUE("hello"sv));
}
```

Initially, I was concerned about the fact that the types will be different
even if the values are the same. For example:

```c++
auto x = CONSTANT_VALUE(42);
auto y = CONSTANT_VALUE(42);

static_assert(std::is_same<decltype(x), decltype(y)>::value, "");  // fail!
```

But... this is kind of like how lambdas work. Specifically, lambdas that happen
to be lexically equivalent don't have the same type.

```c++
auto x = [] {};
auto y = [] {};

static_assert(std::is_same<decltype(x), decltype(y)>::value, "");  // fail!
```

It seems that "having too many types" may not be an issue.

## Final Remarks

So far I've only used this technique for __[mpark/format]__ which is an
experimental string format library where the format string is parsed and
checked at compile-time.

I'd love to hear about your use cases if you find this useful!

[mpark/format]: https://github.com/mpark/format
