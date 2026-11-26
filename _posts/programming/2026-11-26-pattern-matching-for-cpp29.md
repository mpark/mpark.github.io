---
layout: post
title: "Pattern Matching for C++29"
excerpt: "Update from Búzios, a tour of P2688R6, and next steps"
date: 2026-11-26
categories: programming
tags: [c++, c++29, pattern-matching, wg21, proposals]
draft: true
---

**TL;DR**: EWG discussed [P2688R6] in Búzios, and took the following poll
([EWG Paper Tracker](https://github.com/cplusplus/papers/issues/1353#issuecomment-7019509541)):

> EWG encourages more work in direction of P2688R6
> 
> | SF | F  | N | A | SA |
> |:--:|:--:|:-:|:-:|:--:|
> | 33 | 11 | 2 | 0 |  0 |
>
> Result: **Consensus**

My plan is to bring wording to the Istanbul meeting in March 2027. The feedback
on this latest design has been overwhelmingly positive. As C++26 wraps up (🤞),
we should move now for pattern matching to make it in C++29.

# Introduction

[P2688R5]: https://wg21.link/p2688r5
[P2688R6]: https://wg21.link/p2688r6

Last week, I attended the C++ committee meeting held in Búzios, Brazil.
This was my first in-person meeting in about 2.5 years, the last one
having been the St. Louis meeting in June 2024. After that, I attended
meetings remotely to be present with my wife during her pregnancy.

Last time I presented pattern matching in EWG (remotely) was at the Hagenberg
meeting in Feb 2025, where [P2688R5] was being considered for C++26
([EWG Paper Tracker](https://github.com/cplusplus/papers/issues/1353#issuecomment-2659441311)).
The "forward to CWG for inclusion in C++26" poll resulted in `20/11/6/13/5`,
and did not gain consensus.

While I was somewhat dejected with the result, I also agreed with the sentiment.

A few months later, in May 2025, my son was born 🎉. Needless to say, I decided
to give pattern matching a bit of a rest, and be fully present with my family.

Almost 1.5 years later, I submitted [P2688R6] in Sept 2026. It still offers
**pattern composition**, as I consider it to be an absolute non-negotiable for
pattern matching. However, it challenged many aspects of R5. For example, R6
uses existing declaration syntax to introduce new names instead of `let`,
replaces infix `match` with a prefix context-sensitive `match`
(e.g. `match (x) { ... }`), and includes exhaustiveness checking.

I was absolutely stoked to present this in Búzios. Silly me, I thought we were
going to be discussing C++29 features at this meeting! Instead, most of the week
ended up being dedicated to resolving the disapproval of the DIS. There's been
more than enough discussion about that in published papers, [`r/cpp`](https://reddit.com/r/cpp),
etc, so I won't be discussing any of that in this post.

Since most of the post-meeting discourse will likely be dominated by Contracts
discussions as well, I'll give an update of the latest pattern matching proposal
in this post.

# A Tour of [P2688R6]

[2.1 Using Pattern Matching]: https://wg21.link/p2688r6#using-pattern-matching
[2.2 Customizating an Alternative Type]: https://wg21.link/p2688r6#customizing-an-alternative-type

We'll focus on user-side examples here, from [2.1 Using Pattern Matching].
For a library-side example, please refer to
[2.2 Customizating an Alternative Type].

The implementation of the proposal is available on [Compiler Explorer](https://godbolt.org)
under `x86-64 clang (pattern matching - P2688)`. The examples from this section
can be found at <https://godbolt.org/z/P68Y4ePe9>, so you can try stuff out as we go along.

[The Rust Programming Language]: https://doc.rust-lang.org/book/ch06-02-match.html

The following coin example is adapted from [The Rust Programming Language] book,
Chapter 6.2:

```cpp
enum class Coin { Penny, Nickel, Dime, Quarter };

int value_in_cents(Coin coin) {
  return match (coin) {
    case Coin::Penny => 1;
    //   ^^^^^^^^^^^  value pattern
    case Coin::Nickel => 5;
    case Coin::Dime => 10;
    case Coin::Quarter => 25;
  };
}
```

This is a `match` expression, so the expression in the selected handler provides
its value. A *value pattern* such as `Coin::Penny` matches if the subject has
the same value. Exhaustiveness checking is required in general. For an `enum`,
every enumerator value is required to be covered. Therefore, omitting
the `Coin::Quarter` case is ill-formed:

```text
error: match expression is not exhaustive; example of a missing case: Quarter
```

Next, let's consider a dice roll example from the same chapter:

```cpp
int dice_roll();

void take_turn() {
  match (dice_roll()) {
    case 3 => add_fancy_hat();
    case 7 => remove_fancy_hat();
    case int roll => move_player(roll);
    //   ^^^^^^^^  declaration pattern
  }
}
```

This is a `match` statement, so the statement in the selected handler is
executed. `dice_roll()` is evaluated exactly once. The *declaration pattern*
(`int roll`) matches every other result and initializes `roll` from it.

> *Note*: `3` and `7` are just expressions, so if we had a constant named
> `seven` with value `7`, `case seven` would have the same effect as `case 7`.

If every result other than `3` or `7` should simply be re-rolled, its value can
be ignored with a *wildcard pattern* (`_`):

```cpp
match (dice_roll()) {
  case 3 => add_fancy_hat();
  case 7 => remove_fancy_hat();
  case _ => reroll();
  //   ^  wildcard pattern
}
```

Patterns can also be combined with `or`. An *or-pattern* tries its alternatives
against the same subject from left to right and matches when one succeeds:

```cpp
match (dice_roll()) {
  case 1 or 2 => lose_turn();
  //   ^^^^^^  or-pattern
  case 3 => add_fancy_hat();
  case 7 => remove_fancy_hat();
  case int roll => move_player(roll);
}
```

Suppose now that we roll two dice and return them as a `std::pair`:

```cpp
std::pair<int, int> roll_two_dice();

match (roll_two_dice()) {
  case [1, 1] => snake_eyes();
  //   ^^^^^^  decomposition pattern
  case [6, 6] => boxcars();
  case [int r1, int r2] if (r1 == r2) => {
    play_doubles(r1);
  }
  case [int r1, int r2] => {
    move_player(r1 + r2);
  }
}
```

`[P1, P2]` is a *decomposition pattern*. It recursively matches the two
elements.

> *Note 1*: These can be mixed of course, like `case [6, int r2]` or `case [six, int r2]`.

> *Note 2*: In this example, we could also use structured bindings and
> write `case auto [r1, r2]` instead of `case [int r1, int r2]`.

After a pattern matches, if there is an `if` guard present,
the handler is taken if the guard evaluates to `true`. Otherwise,
the next match case is considered.

Alternatively, a `match` can match two separately produced values directly:

```cpp
match (dice_roll(), dice_roll()) {
  case [1, 1] => snake_eyes();
  case [6, 6] => boxcars();
  case [int r1, int r2] if (r1 == r2) => {
    play_doubles(r1);
  }
  case [int r1, int r2] => {
    move_player(r1 + r2);
  }
}
```

Multiple subjects form an unnamed type and are matched together.

Let's shift gears and look at alternative types. Suppose our die may roll off
the table, and its result therefore is a `std::optional`:

```cpp
std::optional<int> try_dice_roll();

match (try_dice_roll()) {
  case { int roll } => move_player(roll);
  //   ^^^^^^^^^^^^  alternative pattern
  case {} => roll_again();
}
```

`{ P }` projects the `int` inside the `optional`, if present, and recursively
applies `P` to it. In this example, `P` is the declaration pattern `int roll`.
`{}` matches the `optional`'s empty state.

The pattern inside the braces can itself be a decomposition pattern. For an
`optional<pair<int, int>>`, we can write:

```cpp
void take_optional_pair_turn(std::optional<std::pair<int, int>> dice) {
  match (dice) {
    case { [1, 1] } => snake_eyes();
    case { [int r1, int r2] } => move_player(r1 + r2);
    case {} => roll_again();
  }
}
```

This recursive composition is an important part of the design. The braces
select a contained alternative, the brackets decompose it, and the nested
patterns test or declare its elements.

A `std::variant` can have several different alternatives:

```cpp
using SearchResult = std::variant<File, Directory>;

void open(const SearchResult& result) {
  match (result) {
    case { const File& file } => open_file(file);
    case { const Directory& directory } => open_directory(directory);
  }
}
```

Each declaration pattern matches an alternative whose payload has
the corresponding type and introduces a name for that payload in the handler.

A `std::expected` result can be matched similarly to a `variant`:

```cpp
std::expected<int, std::string> parse_port(std::string_view text);

match (parse_port(text)) {
  case { int port } => connect(port);
  case { const std::string& message } => report(message);
}
```

Note that the value and error types of `expected` can be the same.
`std::expected<std::string, std::string>` in particular comes up often in
practice. In that case, *named selector*s can be used to distinguish the
alternatives:

```cpp
std::expected<std::string, std::string> read_file(std::string_view path);

auto result = read_file(path);

match (result) {
  case { .value: const auto& contents } => process(contents);
  //     ^^^^^^  named selector
  case { .error: const auto& message } => report(message);
}
```

A declaration pattern can use `dynamic_cast` semantics when applied to a
polymorphic class subject.

```cpp
struct Shape { virtual ~Shape() = default; };

struct Circle : Shape { double radius() const; };

struct Rectangle : Shape {
  double width() const;
  double height() const;
};

double compute_area(const Shape& shape) {
  return match (shape) {
    case const Circle& c => std::numbers::pi * c.radius() * c.radius();
    case const Rectangle& r => r.width() * r.height();
    case _ => throw std::invalid_argument("unknown shape");
  };
}
```

The wildcard covers other classes that may derive from `Shape`.

Patterns can also be used in boolean tests and control-flow conditions:

```cpp
SearchResult result = search(path);

// boolean-yielding single-pattern test expression
bool is_file = match(result, case { const File& });

// pattern condition
if (case { const Directory& directory } = result) {
  open_directory(directory);
}
```

`const File&` is a declaration pattern with an omitted name. It tests whether
the corresponding declaration could be initialized, but does not perform that
initialization.

A single-pattern test produces `bool` whether the pattern matched or not.
A *pattern condition* additionally makes names introduced by its pattern
available in subsequent conditions and the successful branch.

# Directional Consensus in EWG

[P2688R6] was discussed on Thurs afternoon in EWG. The presented slides: [P2688R6.slides.html].

[P2688R6.slides.html]: https://mpark.github.io/wg21-papers/staging/match/P2688R6.slides.html

EWG took the following poll
([EWG Paper Tracker](https://github.com/cplusplus/papers/issues/1353#issuecomment-7019509541)):

> EWG encourages more work in direction of [P2688R6]
> 
> | SF | F  | N | A | SA |
> |:--:|:--:|:-:|:-:|:--:|
> | 33 | 11 | 2 | 0 |  0 |
>
> Result: **Consensus**

This is of course, overwhelming support. It is also perfectly aligned with
overwhelming feedback I've received so far which mainly consist of
"this feels like C++" and "this is a significant improvement".

Folks who were in favor of R5 are even more in favor of R6, and several who were
against R5 have told me that they are now in favor of R6. Personally, this is
the most confident, optimistic, and satisfied I've been with the design.

# What's Next

[P1000R8]: https://wg21.link/p1000r8

I plan to bring wording for P2688 to the Istanbul meeting, as well as the
library companion paper for LEWG. The ambitious goal is to have the proposal
forwarded to the CWG for C++29. Realistically, it'll need a few more meetings
to get to that point.

The following is an adapted timeline from [P1000R8]:

<table>
  <thead>
    <tr><th>Meeting</th><th>Milestone</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>2026.2 - Brno</td>
      <td>First meeting of C++29</td>
    </tr>
    <tr>
      <td>2026.3 - Búzios</td>
      <td rowspan="4">Try to front-load “big” language features including ones with broad library impact<br>(incl. try to merge TSes here)</td>
    </tr>
    <tr class="timeline-current"><td><strong>We are here</strong></td></tr>
    <tr><td>2027.1 - Istanbul</td></tr>
    <tr><td>2027.2 - Sofia</td></tr>
    <tr>
      <td>2027.3 - Budapest (tentative)</td>
      <td>EWG: Last meeting for new C++29 language proposals we haven’t seen before.<br>EWG → LEWG: Last meeting to approve C++29 features needing library response.<br>LEWG: Focus on progressing papers on how to react to new language features.</td>
    </tr>
    <tr>
      <td>2028.1 - Kona</td>
      <td>* → CWG, LWG: Last meeting to send proposals to wording review<br>(incl. TS merges).<br><strong>C++29 design is feature-complete.</strong></td>
    </tr>
  </tbody>
</table>

So we really don't have **that** many more meetings left. If we're serious about
shipping pattern matching in C++29, we've gotta get moving.

Artisan programmers deserve better facilities, and our AI companions would
benefit from the guardrails provided by exhaustiveness checking.
