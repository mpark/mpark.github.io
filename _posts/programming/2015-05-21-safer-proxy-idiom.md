---
layout: post
title: "Safer Proxy Idiom"
excerpt: "How to make proxy objects harder to use incorrectly."
modified: 2016-05-11
categories: programming
tags: [c++, proxy]
---

## Introduction

Today I want to tackle a very specific problem around proxy objects such as
`std::vector<bool>::reference`. The general pattern here is that the object
being returned could be holding references to a temporary object and therefore
needs to be consumed before the next sequence point. This is roughly the wording
employed to describe the behavior of `std::forward_as_tuple` as well.

> If you need a more credible description of this problem, you can refer to
> Item 6 of Scott Meyers' Effective Modern C++ titled: "Use the explicitly typed
> initializer idiom when `auto` deduces undesired types."

## The Problem

The main issue with proxy objects such as `std::vector<bool>::reference` and
`std::forward_as_tuple` is that often they are designed to only live for a
single statement. Suppose I have a proxy object `Join` which is returned from a
string `join` function. In this case expectation is that the user immediately
consumes the proxy object. They can do so by streaming it out, or actually
construct a string out of it.

```c++
class Join {
  public:

  // Implicitly convert to a `std::string`.
  operator std::string() && {
    std::ostringstream strm;
    strm << *this;
    return std::move(strm).str();
  }

  explicit Join(const std::string &delimiter,
                const std::string &lhs,
                const std::string &rhs)
      : delimiter_(delimiter), lhs_(lhs), rhs_(rhs) {}

  private:

  // References to the parts.
  const std::string &delimiter_;
  const std::string &lhs_;
  const std::string &rhs_;

  // Directly stream the arguments out to the stream.
  friend std::ostream &operator<<(std::ostream &strm, const Join &join) {
    return strm << join.lhs_ << join.delimiter_ << join.rhs_;
  }
};

Join join(const std::string &delimiter,
          const std::string &lhs,
          const std::string &rhs) {
  return Join(delimiter, lhs, rhs);
}
```

It can be used like so:

```c++
// Construct a string and use it.
std::string str = join(", ", "a", "b");
std::cout << str << std::endl;

// Streaming it out directly avoids the temporary string unlike the above.
std::cout << join(", ", "hello", "world") << std::endl;
```

This is great, unless we use it incorrectly of course. One subtle pitfall here
is if we were to replace `std::string` with `auto`. For example:

```c++
// Construct a string? and use it...
auto str = join(", ", "a", "b");

// Not safe! `str` could be holding references to temporary objects!
std::cout << str << std::endl;
```

Here we've arrived at undefined behavior land. The reason is because `str` is
a proxy object which could be holding references to temporary objects
(which, in this case it is). The temporary objects that `str` is holding
references of have gone out of scope by the time we get to using it.

## How Can We Help?

Our goal is to __make the interface harder to use incorrectly__ by disallowing
statements of the following form:

```c++
auto str = join(", ", "a", "b");
```

## Implementation

We need to disallow the public use of any of the constructors of `Join`, but
allow `join` to access them since it returns `Join` by value. Well, `friend`
sounds like the right candidate for the job. We make all the constructors
`private` and then make `join` a `friend`.

```c++
class Join {
  public:

  // Implicitly convert to a `std::string`.
  operator std::string() && { /* unchanged */ }

  private:

  explicit Join(const std::string &delimiter,
                const std::string &lhs,
                const std::string &rhs)
      : delimiter_(delimiter), lhs_(lhs), rhs_(rhs) {}

  Join(const Join &) = default;
  Join(Join &&) = default;

  // References to the parts.
  /* unchanged */

  // Directly stream the arguments out to the stream.
  friend std::ostream &operator<<(std::ostream &strm, const Join &join) {
    /* unchanged */
  }

  friend Join join(const std::string &delimiter,
                   const std::string &lhs,
                   const std::string &rhs);
};

Join join(const std::string &delimiter,
          const std::string &lhs,
          const std::string &rhs) {
  return Join(delimiter, lhs, rhs);
}
```

Now the attempts to "save" an instance of `Join` will be disallowed.

```c++
// error: calling a private constructor of class 'Join'
auto str = join(", ", "a", "b");
```

That's it! This approach works for cases where the expected result from a
function is a value. In our case, it's probably helpful since we're
essentially expecting a `std::string` as the result of `join`, `Join` works
seamlessly with that expectation, with a compile-time error when you try to
capture it with `auto`.

## Limitations

As of now, I don't know of a way to completely prevent bad behavior.
But in a language where we can bind references to a temporary object,
there's not much we can do: `const auto &ref = join(", ", "a", "b");`

You may look at the above code and say, "but the lifetime of the temporary is
extended to the lifetime of the reference!" and yes, it absolutely is. However,
it does not extend the lifetime of the temporary we want to extend.
In our case, we simply extend the lifetime of the temporary proxy object. Not
the temporary objects that `Join` holds! If `join` were to return a
`std::string`, this approach would work. Binding a reference to a temporary
object however is bad practice in general, so this limitation doesn't worry me
too much.
