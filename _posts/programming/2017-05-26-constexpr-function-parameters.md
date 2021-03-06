---
layout: post
title: "`constexpr` function parameters"
modified:
categories: programming
excerpt: "Can we work around the limitations of non-type template parameters?"
tags: [c++, constexpr, parameters, metaprogramming]
date: 2017-05-26
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

This function is not allowed because `n`  __could__ be a runtime value, in which
case it would violate the requirement that `static_assert` must be given
a constant expression.

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

This doesn't work because we're not allowed to pass an arbitrary literal type as
a non-type template parameter.

> [N3413][N3413] proposes to allow arbitrary literal types for non-type template
> parameters.

Why does this restriction on non-type template parameters exist in the first
place?

For one, C++ has a fundamental notion of "same type". Given values `x` and `y`,
we have to be able to answer whether `foo<x>` and `foo<y>` are the same type.
Currently, the non-type template parameters are restricted such that this
question can be answered using the built-in value equality comparison.
Once user-defined literal types are involved, we have to be able to find a
unique `operator==` across translation units to perform equality comparisons,
and also mangle the same types to the same name. This potentially introduces
a lot of complexity into the linker.

> __NOTE__: This doesn't cover the entirety of the issue since "same type"
> doesn't cover function templates. For example, given the function template `f`
> above, `f<std::make_tuple(101, 202)>` and `f<std::make_tuple(23, 24)>` both
> have the type `void ()`. Thanks [tcanens] for the [reddit comment]!

[tcanens]: https://www.reddit.com/user/tcanens
[reddit comment]: https://www.reddit.com/r/cpp/comments/6en2ok/constexpr_function_parameters/dibmnty/

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
struct S {
  static constexpr auto value() { return std::make_tuple(101, 202); }
};
```

Here, we've created a type `S` that encodes the value
`std::make_tuple(101, 202)` without the value actually being part of the
type. The `tuple(101, 202)` is not _really_ part of the type `S`, but
we can extract the value from `S` (via `S::value()`). The compiler therefore
must somehow associate the value with the type, which I consider to be a
valid encoding.

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

## Macros (< C++17)

I don't like macros either. But at least this is a pretty simple one.

```c++
#define CONSTANT(...) \
  union { static constexpr auto value() { return __VA_ARGS__; } }
```

> __NOTE__: Shout-out to [@lichray] for pointing out that anonymous `struct` is
> not part of the C++ standard in this [tweet]!

[@lichray]: https://twitter.com/lichray
[tweet]: https://twitter.com/lichray/status/870376890611056640

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

Note that this macro introduces a new type for every value. We're essentially
"solving" the problem of performing equality comparison of literal types by
punting it entirely and saying "they're never equal". For example,

```c++
auto x = CONSTANT_VALUE(42);
auto y = CONSTANT_VALUE(42);

static_assert(std::is_same<decltype(x), decltype(y)>::value, "");  // fail!
```

Initially, I was concerned about the potential of introducing too many types.
But this is actually similar to how lambdas work. Specifically, lambdas
that happen to be lexically equivalent don't have the same type.

```c++
auto x = [] {};
auto y = [] {};

static_assert(std::is_same<decltype(x), decltype(y)>::value, "");  // fail!
```

It seems that "having too many types" may not be an issue.

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

This technique is also used in __[Boost.Hana]__ to implement
[`BOOST_HANA_STRING`][hana-string] which came out of a conversation
with [Louis Dionne] during our first meeting back in CppCon 2014 😊

## C++17

Since we have `constexpr` lambdas in C++17, we can ditch the macros from above
and pass around the lambdas directly. For example,

```c++
template <typename X>
constexpr void f(X x) {
  static_assert(std::get<0>(x()) == 101, "");
  static_assert(std::get<1>(x()) == 202, "");
}

int main() {
  f([] { return std::make_tuple(101, 202); });
}
```

## Final Remarks

I've used this technique for __[mpark/format]__ which is an experimental string
format library where the format string is parsed and checked at compile-time.

I'd love to hear about your use cases if you find this useful!

Thanks to [Louis Dionne] for reviewing this post!

[mpark/format]: https://github.com/mpark/format
[Boost.Hana]: http://boostorg.github.io/hana
[hana-string]: https://github.com/boostorg/hana/blob/cae3fb3fb3f3f1d88a89fe88c3f15e6261a4fd3e/include/boost/hana/string.hpp#L101-L107
[Louis Dionne]: https://github.com/ldionne
