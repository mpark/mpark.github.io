---
layout: post
title: "Alternative std::decay Implementation"
excerpt: "A simpler definition and implementation of std::decay."
modified: 2016-05-11
categories: programming
tags: [c++, std-decay]
share: true
---

> UPDATE: My concern discussed here has been answered on [Stackoverflow].

## What's `std::decay`?

`decay` is a _metafunction_ added to the `<type_traits>` header in C++11. A _metafunction_ is a function (in the mathematical sense) that takes types as arguments rather than values. A simpler metafunction to describe is `add_const` which takes any arbitrary type and adds `const` to it. For example:

```c++
static_assert(std::is_same_v<std::add_const_t<int>, const int>);
static_assert(std::is_same_v<std::add_const_t<int *>, int * const>);
```

`decay` applies type transformations as if it were being passed as a function argument by value. For example:

```c++
static_assert(std::is_same_v<std::decay_t<void ()>, void (*)()>);
static_assert(std::is_same_v<std::decay_t<int [3]>, int *>);
static_assert(std::is_same_v<std::decay_t<const int>, int>);
```

## Current Implementation

Here's a possible implementation as given in by
[cppreference.com](http://en.cppreference.com/w/cpp/types/decay).
This is also roughly the implementation used in both `libstdc++` and `libc++`.

```c++
template< class T >
struct decay {
  typedef typename std::remove_reference<T>::type U;
  typedef typename std::conditional<
    std::is_array<U>::value,
    typename std::remove_extent<U>::type*,
    typename std::conditional<
      std::is_function<U>::value,
      typename std::add_pointer<U>::type,
      typename std::remove_cv<U>::type
    >::type
  >::type type;
};
```

## Alternative Implementation

The idea here is to leverage the compiler which already defines the type
transformations necessary for pass-by-value semantics, rather than manually
writing them out for `decay`.

Here it is:

```c++
template <typename T>
struct decay {
  template <typename U> static U impl(U);
  using type = decltype(impl(std::declval<T>()));
};
```

The function `impl` takes an argument by value, so any argument of type `T` that
gets passed to it will go through the type transformations necessary for
pass-by-value sematics. Notice that this is exactly what we want for `decay`!

I pass an rvalue reference of `T` to `impl`, and the deduced type `U` is the
result we want for `decay`.

## So What?

Not only is the implementation simpler this way, but it also follows the
_DRY (Don't Repeat Yourself)_ principle. Suppose the definition of pass-by-value
transformations were to change in the standard. In the current state, at least 4
modifications are required.

1. Rules for template type deduction
2. Definition of `decay`
3. Implementation of template type deduction in the compiler
4. Implementation of `decay` in the standard library

If we were to piggy-back on the rules of template type deduction to define
`decay` instead, (1) and (3) still needs to be done but we get (2) and (4)
for free.

I also think it allows for stronger claims about what `decay` does. Here's a
note about `decay` from [N3797](https://isocpp.org/files/papers/N3797.pdf):

> [ Note: This behavior is similar to the lvalue-to-rvalue (4.1),
> array-to-pointer (4.2), and function-to-pointer (4.3) conversions applied when
> an lvalue expression is used as an rvalue, but also strips cv-qualiﬁers from
> class types in order to more closely model by-value argument passing.
> — end note ]

What does "__more closely__ model by-value argument passing" mean here? It
sounds to me like it __doesn't exactly__ model by-value argument passing. I
think leveraging the existing rules of template type deduction would allow us to
claim that it __exactly__ models by-value argument passing.

Am I off base here? Let me know what you think!

[Stackoverflow]: http://stackoverflow.com/questions/30419426/what-are-the-differences-between-stddecay-and-pass-by-value
