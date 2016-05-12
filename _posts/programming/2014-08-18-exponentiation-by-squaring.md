---
layout: post
title: "Exponentiation By Squaring"
excerpt: "Exploring a faster exponentiation algorithm."
modified: 2016-05-11
categories: programming
tags: [algorithm, exponentiation]
share: true
---

In this post, I want to talk about a fast exponentiation algorithm that
I learned recently.

Our goal is to write a function `double exp(double x, int n);` which returns
$$ x^n $$.

## Naive Implementation

```c++
double exp(double x, int n) {
  if (n < 0) return 1 / exp(x, -n);
  double result = 1;
  while (n--) result *= x;
  return result;
}
```

We have an $$ O(n) $$ algorithm here. Let's improve it to be $$ O(\lg n) $$.
The algorithm is called [Exponentiation by Squaring].

## Recursive Approach

In this approach we derive the recursive relation and the implementation is
quite straight forward.

### Key Property

$$
x^n =
  \begin{cases}
    \frac{1}{x^{-n}}     & \text{if $ n $ < 0}\\
    1                    & \text{if $ n $ = 0}\\
    x(x^2)^\frac{n-1}{2} & \text{if $ n $ is odd}\\
    (x^2)^\frac{n}{2}    & \text{if $ n $ is even}
  \end{cases}
$$

### Implementation

```c++
double exp(double x, int n) {
  if (n < 0) return 1 / exp(x, -n);
  if (n == 0) return 1;
  return (n % 2 == 1 ? x : 1) * exp(x * x, n / 2);
}
```

Since we divide the problem in half at each step, we get our $$ O(\lg n) $$
algorithm.

## Iterative Approach

In this approach, we iterate through the bits of $$ n $$ and accumulate the
result to $$ x^n $$.

### Key Property

Consider the binary representation of $$ n = b_k \ldots b_2 b_1 b_0 $$ where
each $$ b_i $$ are binary digits.

$$
n = b_0 \times 2^0 + b_1 \times 2^1 + b_2 \times 2^2 + \ldots + b_k \times 2^k
$$

Plugging this into $$ x^n $$ we get,

$$
\begin{align*}
x^n &= x^{b_0 2^0 + b_1 2^1 + b_2 2^2 + \ldots + b_k 2^k}\\
    &= x^{b_0 2^0} \times x^{b_1 2^1} \times
       x^{b_2 2^2} \times \ldots \times x^{b_k 2^k}\\
    &= (x^{2^0})^{b_0} \times (x^{2^1})^{b_1} \times
       (x^{2^2})^{b_2} \times \ldots \times (x^{2^k})^{b_k}\\
    &= (x)^{b_0} \times (x^2)^{b_1} \times
       (x^{2^2})^{b_2} \times \ldots \times (x^{2^k})^{b_k}
\end{align*}
$$

We notice 2 things here.

1. A term contributes to the result only if $$ b_i $$ is 1,
   since if $$ b_i $$ is 0, then the term becomes 1.
2. As we iterate through $$ i $$ starting from 0, $$ (x^{2^i}) $$ starts at
   $$ x $$ and gets squared every iteration.

### Implementation

```c++
double exp(double x, int n) {
  if (n < 0) return 1 / exp(x, -n);
  double result = 1;
  while (n > 0) {
    if (n % 2 == 1) result *= x;
    x *= x;
    n /= 2;
  }  // while
  return result;
}
```

1. `n % 2` represents our bit $$ b_i $$. At each iteration, we only contribute
   the term to the result only if `n % 2` is 1.
2. The variable `x` represents the $$ (x^{2^i}) $$. It starts at $$ x $$ and
   gets squared on every iteration.

## Some Numbers

These are performance numbers for computing $$ x^{35} $$ for
$$ x = [0 \ldots 100,000,000) $$

|------------|------------|
| Approach   | Time (ms)  |
|:----------:|:----------:|
| Naive      | 1733       |
| Recursive  | 1670       |
| Iterative  | 267        |
| `std::pow` | 7099       |
|------------|------------|

Clearly the iterative $$ O(\lg n) $$ algorithm is the most efficient and it
seems that even though the recursive algorithm is also logarithmic, it's not
actually that much better than the naive version at $$ n = 35 $$.

[Exponentiation by Squaring]: http://en.wikipedia.org/wiki/Exponentiation_by_squaring
