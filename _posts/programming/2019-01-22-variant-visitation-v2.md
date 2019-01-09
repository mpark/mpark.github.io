---
layout: post
title: "Variant Visitation V2"
excerpt: "`switch`-based implementation for `visit`"
modified: 2019-01-22
categories: programming
tags: [c++, variant, visitation]
---

## Introduction

In 2015, I wrote an article titled [Variant Visitation] which described
an implementation strategy for `std::visit`. The approach involved a matrix
of function pointers, and many have raised concerns regarding poor code-gen
caused by optimization limitations of function pointers on some compilers.

Here are a few that I remember:
  - [Open GCC issue] from Vittorio Romeo
  - [Tweet] from Jason Turner [@lefticus](https://twitter.com/lefticus)
  - [Blog post] from Björn Fahller [@bjorn_fahller](https://twitter.com/bjorn_fahller)
  - [CppCon 2018 Talk] from Nir Friedman [quicknir]
  - Several Reddit threads:
    - [Disappearing C++17 std::variant act][disppear]
    - std::variant code bloat? Looks like it's std::visit fault ([Part 1][codebloat1]) ([Part 2][codebloat2])
    - [When performance guarantees hurts performance: std::visit][guarantees]
    - [std::visit unable to generate optimal assembly][optimal]
    - [std::visit overhead][overhead].

Anyway, it's been on my list of things to do for a while. The motivation
for action didn't come however until a few months ago when Nir Friedman
([quicknir]) filed an [issue] on [mpark/variant]
and generously offered to help with the effort!

This post describes the `switch`-based approach implemented in [mpark/variant],
and its benchmark results.

[Variant Visitation]: /programming/2015/07/07/variant-visitation/
[Open GCC issue]: https://gcc.gnu.org/bugzilla/show_bug.cgi?id=78113
[Tweet]: https://twitter.com/lefticus/status/1018265568393064448
[Blog post]: https://playfulprogramming.blogspot.com/2018/12/when-performance-guarantees-hurts.html
[disppear]: https://www.reddit.com/r/cpp/comments/597x8e/disappearing_c17_stdvariant_act/
[codebloat1]: https://www.reddit.com/r/cpp/comments/9khij8/stdvariant_code_bloat_looks_like_its_stdvisit/
[codebloat2]: https://www.reddit.com/r/cpp/comments/9mxvw5/stdvariant_code_bloat_looks_like_its_stdvisit/
[guarantees]: https://www.reddit.com/r/cpp/comments/a8xkl3/when_performance_guarantees_hurts_performance/
[optimal]: https://www.reddit.com/r/cpp/comments/a8bmpn/stdvisit_unable_to_generate_optimal_assembly/
[overhead]: https://www.reddit.com/r/cpp/comments/7pya5s/stdvisit_overhead/
[issue]: https://github.com/mpark/variant/issues/51
[mpark/variant]: https://github.com/mpark/variant
[CppCon 2018 Talk]: https://youtu.be/8nyq8SNUTSc?t=2806

## Variant

Let us get started with a minimal `variant` interface. This will give us a frame
of reference throughout the post. We only need the following:

  * `v.index()`: Returns the __index__ of the currently stored value.
  * `std::get<I>(v)`: Returns the currently stored value if `v.index() == I` else
                    throws `std::bad_variant_access`.

## Visitation

Here is the signature of `visit`:

```c++
template <typename F, typename... Vs>
constexpr decltype(auto) visit(F &&f, Vs &&... vs);
```

It performs `f(std::get<Is>(vs)...)` where `Is...` are
the compile-time values of `vs.index()...`.

## Implementation

For single visitation in [Variant Visitation], we generate a 1-dimensional
array of function pointers where the function pointer at `I` performs
`f(std::get<I>(v))`. We can then invoke the function pointer at `v.index()`.

In this implementation, we want to `switch` on `v.index()` instead and invoke
`f(std::get<I>(v))` directly rather than through a function pointer.
This should facilitate compilers that have trouble inlining function pointers.

Here's a limited but [simple example](https://godbolt.org/z/idtjpc) on GCC 8.2:

#### Single Visitation: Exactly 4 Alternatives

Let's start with the following snippet:

```cpp
template <typename F, typename V>
constexpr decltype(auto) visit(F &&f, V &&v) {
  switch (v.index()) {
    case 0: {
      return std::invoke(std::forward<F>(f), std::get<0>(std::forward<V>(v)));
    }
    case 1: {
      return std::invoke(std::forward<F>(f), std::get<1>(std::forward<V>(v)));
    }
    case 2: {
      return std::invoke(std::forward<F>(f), std::get<2>(std::forward<V>(v)));
    }
    case 3: {
      return std::invoke(std::forward<F>(f), std::get<3>(std::forward<V>(v)));
    }
  }
}
```

As discussed, we `switch` on `v.index()` and perform `f(std::get<I>(v))`.
This however only works for `variant`s with __exactly__ 4 alternatives 😅.

#### Single Visitation: Up to 4 Alternatives

To do this, we need to detect and disable the invalid branches.
We'll introduce a `struct dispatcher` templated on `bool IsValid`, where
the `false` specialization contains the invalid code path declared
`__builtin_unreachable();` so that compilers know they can safely optimize
it out, and the `true` specialization performs the valid code paths.

```cpp
template <bool IsValid, typename R>
struct dispatcher;

template <typename R>
struct dispatcher<false, R> {
  template <std::size_t I, typename F, typename V>
  static R case_(F &&, V &&) { __builtin_unreachable(); }
};

template <typename R>
struct dispatcher<true, R> {
  template <std::size_t I, typename F, typename V>
  static R case_(F &&f, V &&v) {
    using Expected = R;
    using Actual = decltype(
        std::invoke(std::forward<F>(f), std::get<I>(std::forward<V>(v))));
    static_assert(
        std::is_same_v<Expected, Actual>,
        "`mpark::visit` requires the visitor to have a single return type");
    return std::invoke(std::forward<F>(f), std::get<I>(std::forward<V>(v)));
  }
};
```

In `visit`, we call this conditional `case_` function by checking whether the
case index is within `std::variant_size_v`. If the case is valid, we invoke `f`
with `std::get<I>(v)` as before. Otherwise it's unreachable.

```diff
  template <typename F, typename V>
  constexpr decltype(auto) visit(F &&f, V &&v) {
+   using R = decltype(
+       std::invoke(std::forward<F>(f), std::get<0>(std::forward<V>(v))));
+   constexpr std::size_t size = std::variant_size_v<std::remove_cvref_t<V>>;
    switch (v.index()) {
      case 0: {
-       return std::invoke(std::forward<F>(f), std::get<0>(std::forward<V>(v)));
+       return dispatcher<0 < size, R>::template case_<0>(
+           std::forward<F>(f), std::forward<V>(v));
      }
      case 1: {
-       return std::invoke(std::forward<F>(f), std::get<1>(std::forward<V>(v)));
+       return dispatcher<1 < size, R>::template case_<1>(
+           std::forward<F>(f), std::forward<V>(v));
      }
      case 2: {
-       return std::invoke(std::forward<F>(f), std::get<2>(std::forward<V>(v)));
+       return dispatcher<2 < size, R>::template case_<2>(
+           std::forward<F>(f), std::forward<V>(v));
      }
      case 3: {
-       return std::invoke(std::forward<F>(f), std::get<3>(std::forward<V>(v)));
+       return dispatcher<3 < size, R>::template case_<3>(
+           std::forward<F>(f), std::forward<V>(v));
      }
    }
  }
```


At this point, we can visit `variant`s with __upto__ 4 alternatives.
Next let's enable visiting `variant`s with arbitrary number of alternatives.

#### Single Visitation: Arbitrary # of Alternatives

In the above example, we only handled cases 0 through 3. Since there is no such
thing as a variadic `switch` statement, we need to instead use recursion to
handle the next range of values 4 through 7, and continue on until we've covered
all of the valid cases.

We'll do this by modifying the `switch` statement to handle the same range of
values but adjusted by a base number. That is, given a base `B`, each
recursive call will handle `B+0`, `B+1`, `B+2`, `B+3`. The initial value for `B`
will be `0`, and each recursive call will increment `B` by `4`. We can continue
while `B` is within a value range, i.e., `B` is within `std::variant_size_v`.

That means we have another pair of valid/invalid code paths to consider.
Let's call it `switch_`, to complement the `case_` we already have:

```diff
  template <bool IsValid, typename R>
  struct dispatcher;

  template <typename R>
  struct dispatcher<false, R> {
    template <std::size_t I, typename F, typename V>
    static constexpr R case_(F &&, V &&) { __builtin_unreachable(); }

+   template <std::size_t B, typename F, typename V>
+   static constexpr R switch_(F &&, V &&) { __builtin_unreachable(); }
  };

  template <typename R>
  struct dispatcher<true, R> {
    template <std::size_t I, typename F, typename V>
    static constexpr R case_(F &&f, V &&v) {
      using Expected = R;
      using Actual = decltype(
          std::invoke(std::forward<F>(f), std::get<I>(std::forward<V>(v))));
      static_assert(
          std::is_same_v<Expected, Actual>,
          "`mpark::visit` requires the visitor to have a single return type");
      return std::invoke(std::forward<F>(f), std::get<I>(std::forward<V>(v)));
    }

+   template <std::size_t B, typename F, typename V>
+   static constexpr R switch_(F &&f, V &&v) {
-     using R = decltype(
-         std::invoke(std::forward<F>(f), std::get<0>(std::forward<V>(v))));
      constexpr std::size_t size = std::variant_size_v<std::remove_cvref_t<V>>;
      switch (v.index()) {
        case B+0: {
-         return dispatcher<0 < size, R>::template case_<0>(
+         return dispatcher<B+0 < size, R>::template case_<B+0>(
              std::forward<F>(f), std::forward<V>(v));
        }
        case B+1: {
-         return dispatcher<1 < size, R>::template case_<1>(
+         return dispatcher<B+1 < size, R>::template case_<B+1>(
              std::forward<F>(f), std::forward<V>(v));
        }
        case B+2: {
-         return dispatcher<2 < size, R>::template case_<2>(
+         return dispatcher<B+2 < size, R>::template case_<B+2>(
              std::forward<F>(f), std::forward<V>(v));
        }
        case B+3: {
-         return dispatcher<3 < size, R>::template case_<3>(
+         return dispatcher<B+3 < size, R>::template case_<B+3>(
              std::forward<F>(f), std::forward<V>(v));
        }
+       default: {
+         return dispatcher<B+4 < size, R>::template switch_<B+4>(
+             std::forward<F>(f), std::forward<V>(v));
+       }
      }
+   }
  };
```

With this mechanism in place, `visit` simply bootstraps the recursion.

```diff
  template <typename F, typename V>
  constexpr decltype(auto) visit(F &&f, V &&v) {
    using R = decltype(
        std::invoke(std::forward<F>(f), std::get<0>(std::forward<V>(v))));
+   return dispatcher<true, R>::template switch_<0>(std::forward<F>(v),
+                                                   std::forward<V>(v));
  }
```

Great! We've got single visitation supporting arbitrary number of alternatives.
While I showed each recursive level handling 4 cases, the actual implementation
handles 32 cases at a time.

#### Multi-visitation

Even though multi-visitation is currently implemented with a generalized
version of this technique, I'll leave the details for another post since
there's more work to be done. Specifically, I want to investigate some
compile-time improvements and add some benchmarks for runtime performance.

## Benchmark

The following benchmarking code is what the measurements are based on,
and is inspired by the one that Björn Fahller presented in his blog post:
[When performance guarantees hurts performance - std::visit][Blog post].

The benchmark creates a `vector` of approximately 5000 `variant`s with `N`
alternatives, and sums up the compile-time indices that are assigned to
each variant. The idea being that if the visitor is inlined for
the `switch` and out-of-line for jump table, `switch` should perform better.

```cpp
#include <benchmark/benchmark.h>

#include <cstddef>
#include <type_traits>
#include <vector>

#include <mpark/variant.hpp>

template <std::size_t I>
using size_constant = std::integral_constant<std::size_t, I>;

template <std::size_t... Is>
auto make_variants(lib::index_sequence<Is...>) {
  std::vector<mpark::variant<size_constant<Is>...>> vs;
  for (std::size_t i = 0; i < 5000 / sizeof...(Is); ++i) {
    int dummy[] = {(vs.push_back(size_constant<Is>{}), 0)...};
    (void)dummy;
  }
  return vs;
}

struct visitor {
  template <std::size_t I>
  constexpr std::size_t operator()(size_constant<I>) const {
    return I;
  }
};

template <std::size_t N>
static void visit1(benchmark::State &state) {
  auto vs = make_variants(lib::make_index_sequence<N>{});
  for (auto _ : state) {
    int sum = 0;
    for (const auto &v : vs) {
      sum += mpark::visit(visitor{}, v);
    }
    benchmark::DoNotOptimize(sum);
  }
}

BENCHMARK_TEMPLATE(visit1, 1);
BENCHMARK_TEMPLATE(visit1, 2);
BENCHMARK_TEMPLATE(visit1, 3);
BENCHMARK_TEMPLATE(visit1, 5);
// ...
BENCHMARK_TEMPLATE(visit1, 200);
```

This benchmark was only tested with GCC and Clang. GCC 4.8/4.9/5/6/7/8
and Clang 3.8/3.9/4.0/5.0/6.0/7 under each of C++11/14/17, as supported.

The following is the [result](https://travis-ci.org/mpark/variant/jobs/476107152) for __Clang 7__ under __C++17__:

#### Jump Table
```
---------------------------------------------------
Benchmark            Time           CPU Iterations
---------------------------------------------------
visit1<1>         2847 ns       2847 ns     244887
visit1<2>        11459 ns      11455 ns      61324
visit1<3>        10127 ns      10125 ns      68804
visit1<5>        10607 ns      10604 ns      66159
visit1<8>        11363 ns      11360 ns      61668
visit1<15>       12432 ns      12431 ns      56413
visit1<32>       10590 ns      10590 ns      64951
visit1<65>       10595 ns      10593 ns      66403
visit1<128>      21990 ns      21985 ns      31979
visit1<200>      27180 ns      27174 ns      25770
```

#### Recursive `switch`
```
---------------------------------------------------
Benchmark            Time           CPU Iterations
---------------------------------------------------
visit1<1>         2846 ns       2845 ns     246770
visit1<2>         3405 ns       3404 ns     205871
visit1<3>         2934 ns       2933 ns     239269
visit1<5>         3815 ns       3814 ns     183281
visit1<8>         2935 ns       2935 ns     238622
visit1<15>        2950 ns       2950 ns     238193
visit1<32>        2927 ns       2927 ns     239187
visit1<65>        2954 ns       2954 ns     233865
visit1<128>       9570 ns       9570 ns      73112
visit1<200>       9635 ns       9635 ns      73103
```

The following is the [result](https://travis-ci.org/mpark/variant/jobs/476107152) for __GCC 8__ under __C++17__:

#### Jump Table
```
---------------------------------------------------
Benchmark            Time           CPU Iterations
---------------------------------------------------
visit1<1>         2811 ns       2811 ns     248491
visit1<2>         9494 ns       9493 ns      73727
visit1<3>        11461 ns      11460 ns      60527
visit1<5>        11487 ns      11486 ns      60689
visit1<8>        11571 ns      11571 ns      61453
visit1<15>       11592 ns      11592 ns      60979
visit1<32>       12553 ns      12553 ns      56333
visit1<65>       10630 ns      10629 ns      65991
visit1<128>      17035 ns      17034 ns      40331
visit1<200>      18484 ns      18484 ns      37944
```

#### Recursive `switch`
```
---------------------------------------------------
Benchmark            Time           CPU Iterations
---------------------------------------------------
visit1<1>         2828 ns       2828 ns     247810
visit1<2>         4203 ns       4202 ns     167167
visit1<3>         2904 ns       2904 ns     241905
visit1<5>         2858 ns       2858 ns     245063
visit1<8>         3830 ns       3830 ns     182081
visit1<15>        2854 ns       2854 ns     245068
visit1<32>        3826 ns       3826 ns     181831
visit1<65>        6920 ns       6920 ns     102445
visit1<128>      12084 ns      12084 ns      57905
visit1<200>      14330 ns      14330 ns      48923
```

The full matrix of the benchmark results is captured in
[Travis CI](https://travis-ci.org/mpark/variant/builds/476107115).

## Conclusion

Clang/GCC both benefit from roughly a 3x improvement in performance from
the `switch`-based implementation strategy. Please reach out with your ideas
for further improvements to the library! Thanks to Nir Friedman ([quicknir])
for contributions / discussions over the past few months in making this happen.

[quicknir]: https://github.com/quicknir

Thanks to [Agustín "K-ballo" Bergé](https://github.com/K-ballo) and
[Barry Revzin](https://twitter.com/BarryRevzin) for reviewing this post!
