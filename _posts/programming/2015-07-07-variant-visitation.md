---
layout: post
title: "Variant Visitation"
excerpt: "A visitation mechanism for `std::variant`."
modified: 2016-05-11
categories: programming
tags: [c++, variant, visitation]
share: true
---

## Introduction

`variant` reached a design consensus at the fall ISO C++ committee meeting in
Kona, HI, USA. While the design is still not final, I've been experimenting to
deliver a [reference implementation](https://github.com/mpark/variant).

In this post, I'll be presenting my approach and implementation of the
visitation mechanism. There are interesting parts in the implementation of
`variant` itself, but a lot of those aspects are already covered in
[Eggs.Variant - Part I] and [Eggs.Variant - Part II] by
[Agustín "K-ballo" Bergé].

## Variant

Let us get started with a minimal `variant` interface. This will give us a frame
of reference throughout the post. We only need the following:

  * `v.index()`: Returns the __index__ of the currently stored value.
  * `get<I>(v)`: Returns the currently stored value if `v.index() == I` else
                 throws `std::bad_variant_access`.

## Visitation

Here is the signature of `visit`:

```c++
template <typename F, typename... Vs>
decltype(auto) visit(F &&f, Vs &&... vs);
```

It invokes `f` with the currently stored value of each of the `vs...`.

Consider the following function:

```c++
// `F` and `Vs...` is in context from outer scope.
template <std::size_t... Is>
constexpr decltype(auto) dispatch(F f, Vs... vs) {
  static_assert(sizeof...(Is) == sizeof...(Vs));
  std::invoke(static_cast<F>(f), get<Is>(static_cast<Vs>(vs))...);
}
```

Since `get<I>(v)` returns the currently stored value of `v` for us,
we ultimately want to invoke `dispatch<Is...>(f, vs...)` where `Is...` are
the compile-time values of `vs.index()...`.

How do we do that?

## `N`-dimensional Matrix of Function Pointers

The approach is to build an `N`-dimensional matrix of function pointers
(representative of the `N`-ary cartesian product), then use `vs.index()...` to
index into the matrix at runtime where the appropriate `&dispatch<Is...>` will
be waiting for us.

For example, given `variant<A, B>` we have the following function matrix:

|      `A`       |      `B`       |
|:--------------:|:--------------:|
| `&dispatch<0>` | `&dispatch<1>` |


given `variant<A, B>` and `variant<X, Y, Z>`, we have:

|         |        `X`        |        `Y`        |        `Z`        |
|:-------:|:-----------------:|:-----------------:|:-----------------:|
| __`A`__ | `&dispatch<0, 0>` | `&dispatch<0, 1>` | `&dispatch<0, 2>` |
| __`B`__ | `&dispatch<1, 0>` | `&dispatch<1, 1>` | `&dispatch<2, 2>` |

## Implementation

### `make_fmatrix`

The base case should look familiar. Given `F`, `Vs...` and `Is...`, it returns
the function pointer `&dispatch<Is...>` from above.

```c++
// Base case for `make_fmatrix_impl`.
template <typename F, typename... Vs, std::size_t... Is>
constexpr auto make_fmatrix_impl(std::index_sequence<Is...>) {
  struct dispatcher {
    static constexpr decltype(auto) dispatch(F f, Vs... vs) {
      return std::invoke(static_cast<F>(f), get<Is>(static_cast<Vs>(vs))...);
    }
  };
  return &dispatcher::dispatch;
}
```

The recursive case implements the `N`-ary cartesian product resulting in the
`N`-dimensional matrix. We build this recursively, keeping the first
`std::index_sequence` as our accumulator. For each given list, we make a
recursive call with each of its elements appended onto the accumulator, and the
rest of the lists.

```c++
// Recursive case for `make_fmatrix_impl`.
template <typename F,
          typename... Vs,
          std::size_t... Is,
          std::size_t... Js,
          typename... Ls>
constexpr auto make_fmatrix_impl(
    std::index_sequence<Is...>, std::index_sequence<Js...>, Ls... ls) {
  return std::experimental::make_array(
      make_fmatrix_impl<F, Vs...>(std::index_sequence<Is..., Js>{}, ls...)...);
}
```

$$
\definecolor{x}{RGB}{250,128,114}
\definecolor{y}{RGB}{34,139,34}
\definecolor{z}{RGB}{65,105,225}
$$

Let's go through an example with three lists $$ [\color{x}0\color{black}, \color{x}1\color{black}] $$, $$ [\color{y}0\color{black}, \color{y}1\color{black}, \color{y}2\color{black}] $$, $$ [\color{z}0\color{black}, \color{z}1\color{black}] $$:

$$ \operatorname{make\_fmatrix\_impl}([], [\color{x}0\color{black}, \color{x}1\color{black}], [\color{y}0\color{black}, \color{y}1\color{black}, \color{y}2\color{black}], [\color{z}0\color{black}, \color{z}1\color{black}]) $$

$$
\Rightarrow
\begin{bmatrix}
  \operatorname{make\_fmatrix\_impl}([\color{x}0\color{black}], [\color{y}0\color{black}, \color{y}1\color{black}, \color{y}2\color{black}], [\color{z}0\color{black}, \color{z}1\color{black}]) \\
  \operatorname{make\_fmatrix\_impl}([\color{x}1\color{black}], [\color{y}0\color{black}, \color{y}1\color{black}, \color{y}2\color{black}], [\color{z}0\color{black}, \color{z}1\color{black}]) \\
\end{bmatrix}
$$

$$
\Rightarrow
\begin{bmatrix}
  \begin{bmatrix}
    \operatorname{make\_fmatrix\_impl}([\color{x}0\color{black}, \color{y}0\color{black}], [\color{z}0\color{black}, \color{z}1\color{black}]) \\
    \operatorname{make\_fmatrix\_impl}([\color{x}0\color{black}, \color{y}1\color{black}], [\color{z}0\color{black}, \color{z}1\color{black}]) \\
    \operatorname{make\_fmatrix\_impl}([\color{x}0\color{black}, \color{y}2\color{black}], [\color{z}0\color{black}, \color{z}1\color{black}]) \\
  \end{bmatrix}
  \begin{bmatrix}
    \operatorname{make\_fmatrix\_impl}([\color{x}1\color{black}, \color{y}0\color{black}], [\color{z}0\color{black}, \color{z}1\color{black}]) \\
    \operatorname{make\_fmatrix\_impl}([\color{x}1\color{black}, \color{y}1\color{black}], [\color{z}0\color{black}, \color{z}1\color{black}]) \\
    \operatorname{make\_fmatrix\_impl}([\color{x}1\color{black}, \color{y}2\color{black}], [\color{z}0\color{black}, \color{z}1\color{black}]) \\
  \end{bmatrix}
\end{bmatrix}
$$

$$
\Rightarrow
\begin{bmatrix}
  \begin{bmatrix}
    \begin{bmatrix}
      \operatorname{make\_fmatrix\_impl}([\color{x}0\color{black}, \color{y}0\color{black}, \color{z}0\color{black}]) \\
      \operatorname{make\_fmatrix\_impl}([\color{x}0\color{black}, \color{y}0\color{black}, \color{z}1\color{black}]) \\
    \end{bmatrix} \\
    \begin{bmatrix}
      \operatorname{make\_fmatrix\_impl}([\color{x}0\color{black}, \color{y}1\color{black}, \color{z}0\color{black}]) \\
      \operatorname{make\_fmatrix\_impl}([\color{x}0\color{black}, \color{y}1\color{black}, \color{z}1\color{black}]) \\
    \end{bmatrix} \\
    \begin{bmatrix}
      \operatorname{make\_fmatrix\_impl}([\color{x}0\color{black}, \color{y}2\color{black}, \color{z}0\color{black}]) \\
      \operatorname{make\_fmatrix\_impl}([\color{x}0\color{black}, \color{y}2\color{black}, \color{z}1\color{black}]) \\
    \end{bmatrix} \\
  \end{bmatrix}
  \begin{bmatrix}
    \begin{bmatrix}
      \operatorname{make\_fmatrix\_impl}([\color{x}1\color{black}, \color{y}0\color{black}, \color{z}0\color{black}]) \\
      \operatorname{make\_fmatrix\_impl}([\color{x}1\color{black}, \color{y}0\color{black}, \color{z}1\color{black}]) \\
    \end{bmatrix} \\
    \begin{bmatrix}
      \operatorname{make\_fmatrix\_impl}([\color{x}1\color{black}, \color{y}1\color{black}, \color{z}0\color{black}]) \\
      \operatorname{make\_fmatrix\_impl}([\color{x}1\color{black}, \color{y}1\color{black}, \color{z}1\color{black}]) \\
    \end{bmatrix} \\
    \begin{bmatrix}
      \operatorname{make\_fmatrix\_impl}([\color{x}1\color{black}, \color{y}2\color{black}, \color{z}0\color{black}]) \\
      \operatorname{make\_fmatrix\_impl}([\color{x}1\color{black}, \color{y}2\color{black}, \color{z}1\color{black}]) \\
    \end{bmatrix} \\
  \end{bmatrix}
\end{bmatrix}
$$

$$
\Rightarrow
\begin{bmatrix}
  \begin{bmatrix}
    \begin{bmatrix}
      \operatorname{\&dispatch}\mathopen{<}\color{x}0\color{black}, \color{y}0\color{black}, \color{z}0\color{black}\mathclose{>} \\
      \operatorname{\&dispatch}\mathopen{<}\color{x}0\color{black}, \color{y}0\color{black}, \color{z}1\color{black}\mathclose{>} \\
    \end{bmatrix} \\
    \begin{bmatrix}
      \operatorname{\&dispatch}\mathopen{<}\color{x}0\color{black}, \color{y}1\color{black}, \color{z}0\color{black}\mathclose{>} \\
      \operatorname{\&dispatch}\mathopen{<}\color{x}0\color{black}, \color{y}1\color{black}, \color{z}1\color{black}\mathclose{>} \\
    \end{bmatrix} \\
    \begin{bmatrix}
      \operatorname{\&dispatch}\mathopen{<}\color{x}0\color{black}, \color{y}2\color{black}, \color{z}0\color{black}\mathclose{>} \\
      \operatorname{\&dispatch}\mathopen{<}\color{x}0\color{black}, \color{y}2\color{black}, \color{z}1\color{black}\mathclose{>} \\
    \end{bmatrix} \\
  \end{bmatrix}
  \begin{bmatrix}
    \begin{bmatrix}
      \operatorname{\&dispatch}\mathopen{<}\color{x}1\color{black}, \color{y}0\color{black}, \color{z}0\color{black}\mathclose{>} \\
      \operatorname{\&dispatch}\mathopen{<}\color{x}1\color{black}, \color{y}0\color{black}, \color{z}1\color{black}\mathclose{>} \\
    \end{bmatrix} \\
    \begin{bmatrix}
      \operatorname{\&dispatch}\mathopen{<}\color{x}1\color{black}, \color{y}1\color{black}, \color{z}0\color{black}\mathclose{>} \\
      \operatorname{\&dispatch}\mathopen{<}\color{x}1\color{black}, \color{y}1\color{black}, \color{z}1\color{black}\mathclose{>} \\
    \end{bmatrix} \\
    \begin{bmatrix}
      \operatorname{\&dispatch}\mathopen{<}\color{x}1\color{black}, \color{y}2\color{black}, \color{z}0\color{black}\mathclose{>} \\
      \operatorname{\&dispatch}\mathopen{<}\color{x}1\color{black}, \color{y}2\color{black}, \color{z}1\color{black}\mathclose{>} \\
    \end{bmatrix} \\
  \end{bmatrix}
\end{bmatrix}
$$

`make_fmatrix` bootstraps the above with the empty list `[]` as the initial
accumulator, and index sequences `[0, tuple_size_v<V_i>)` for each of the `V_i`
in `Vs...`.

```c++
template <typename F, typename... Vs>
constexpr auto make_fmatrix() {
  return make_fmatrix_impl<F, Vs...>(
      std::index_sequence<>{},
      std::make_index_sequence<std::tuple_size<std::decay_t<Vs>>::value>{}...);
}
```

Now we can construct a `static constexpr` instance of `fmatrix` within `visit`.

```c++
template <typename F, typename... Vs>
decltype(auto) visit(F &&f, Vs &&... vs) {
  static constexpr auto fmatrix = make_fmatrix<F &&, Vs &&...>();
  /* ... */
}
```

We're now left with the task of accessing the `fmatrix` with `vs.index()...`.
Since we can't say something like: `fmatrix([vs.index()]...)` to mean
`fmatrix[i_0][i_1]...[i_N]`, we need to introduce a helper function.

### `at` for `N`-dimensional arrays

`at(array, i_0, i_1, ..., i_N)` returns `array[i_0][i_1]...[i_N]`.

```c++
// Base case for `at_impl`.
template <typename T>
constexpr T &&at_impl(T &&elem) { return std::forward<T>(elem); }

// Recursive case for `at_impl`.
template <typename T, typename... Is>
constexpr auto &&at_impl(T &&elems, std::size_t i, Is... is) {
  return at_impl(std::forward<T>(elems)[i], is...);
}

template <typename T, typename... Is>
constexpr auto &&at(T &&elems, Is... is) {
  // Commented out since `std::rank` is not defined for `std::array`.
  // static_assert(std::rank<std::decay_t<T>>::value == sizeof...(Is));
  return at_impl(std::forward<T>(elems), is...);
}
```

Alright, almost there! We can now use `at` to retrieve the `&dispatch<Is...>`
corresponding to `vs.index()...` in `fmatrix` and invoke it like this:

```c++
template <typename F, typename... Vs>
decltype(auto) visit(F &&f, Vs &&... vs) {
  static constexpr auto fmatrix = make_fmatrix<F &&, Vs &&...>();
  return at(fmatrix, vs.index()...)(std::forward<F>(f), std::forward<Vs>(vs)...);
}
```

That's it!

We've managed to decouple the implementation of `variant` and the visitation
mechanism, generalized to any `N`. This means that we can implement parts of
`variant` that required single visitation using `visit`. For example, copy
construction could look something along the lines of:

```c++
template <typename... Ts>
class variant {
public:
  /* ... */

  variant(const variant &that) : variant{/* valueless_by_exception */} {
    if (!that.valueless_by_exception()) {
      visit([this](auto &&value) {
              this->construct(std::forward<decltype(value)>(value));
            },
            that);
    }
  }

  /* ... */
};
```

__NOTE__: It's actually a little more complicated than this because duplicate
types in a `variant` have distinct states. We therefore need to initialize the
alternative in `this` at the same index as the one stored in `that`.

The full reference implementation of `std::variant` can be
found [here](https://github.com/mpark/variant).

Thanks to [Agustín "K-ballo" Bergé] for reviewing this post!

[P0088R0]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0088r0.pdf
[Agustín "K-ballo" Bergé]: https://github.com/K-ballo
[Eggs.Variant]: http://eggs-cpp.github.io/variant/
[Eggs.Variant - Part I]: http://talesofcpp.fusionfenix.com/post-17/eggs.variant---part-i
[Eggs.Variant - Part II]: http://talesofcpp.fusionfenix.com/post-20/eggs.variant---part-ii-the-constexpr-experience
