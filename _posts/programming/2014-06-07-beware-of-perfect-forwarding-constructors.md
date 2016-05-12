---
layout: post
title: "Beware of Perfect Forwarding Constructors"
excerpt: "The pitfall of eagerness of perfect forwarding."
modified: 2016-05-07
categories: programming
tags: [c++, perfect-forwarding, constructor, accurate-characterization]
share: true
---

## The Problem

Suppose we want to write a wrapper class in modern C++. Of course we want to
take advantage of move-semantics along with perfect forwarding. So let's provide
a move constructor as well as a templated constructor which perfectly forwards
an argument to initialize our wrapped value. Something like:

```c++
template <typename T>
class wrapper {
  public:

  wrapper() = default;

  wrapper(const wrapper &/* that */) /* : value(that.value) */ {
    std::cout << "wrapper(const wrapper &)" << std::endl;
  }

  wrapper(wrapper &&/* that */) /* : value(std::move(that.value)) */ {
    std::cout << "wrapper(wrapper &&)" << std::endl;
  }

  template <typename U>
  wrapper(U &&/* u */) /* : value(std::forward<U>(u)) */ {
    std::cout << "wrapper(U &&)" << std::endl;
  }

  private:

  // T value;
};
```

> __NOTE:__ The __ctor-init-lists__ are commented out because we want to analyze
> which constructors get invoked on various types, and it doesn't compile in
> some of the cases we want to explore.

Let's try passing `wrapper` by `const &`, `&`, `const &&`, and `&&` with the
intention to copy or move the object.

```c++
// const and non-const objs.
const wrapper<int> const_obj;
wrapper<int> non_const_obj;
// invoke constructors.
wrapper<int> const_lval(const_obj);
wrapper<int> non_const_lval(non_const_obj);
wrapper<int> const_rval(std::move(const_obj));
wrapper<int> non_const_rval(std::move(non_const_obj));
```

This outputs:

```c++
wrapper(const wrapper &)
wrapper(U &&)
wrapper(U &&)
wrapper(wrapper &&)
```

Huh? So passing `wrapper` by `&` and `const &&` didn't seem to invoke the
correct constructors. Without the templated constructor, the output would be:

```c++
wrapper(const wrapper &)
wrapper(const wrapper &)
wrapper(const wrapper &)
wrapper(wrapper &&)
```

What's going on here?

## Explanation

It's important to note that the parameter of the templated constructor is a
[Universal Reference]. Without getting into too much detail, a universal
reference binds to __anything__ and preserves the qualifiers of the argument,
that is, whether the argument is `const`, `volatile`, lvalue or rvalue.
Let's take a look at the second case, where we pass an instance of `wrapper`
by `&`. Since universal reference preserves all the qualifiers, the templated
constructor gets instantiated to `wrapper(wrapper &)`.

> __NOTE:__ If you don't understand how perfect forwarding works,
> [Universal Reference] by Scott Meyers illustrates how it all works.
> It boils down to a special rule for template type deduction in conjunction
> with reference collapsing, but that's for another post.

Now overload resolution kicks in and selects between the two viable functions:

```c++
wrapper(const wrapper &); // copy constructor (user-defined / compiler-defined).
wrapper(wrapper &);       // instantiated templated constructor.
```

What we need to recognize is that `wrapper &` is an exact match for
`non_const_obj` whereas the copy constructor requires an implicit conversion
from `wrapper &` to `const wrapper &`. Since zero conversions is better than
one, `wrapper(wrapper &)` wins overload resolution and thus the templated
constructor gets invoked.

The same line of reasoning goes for the third case, where the template parameter
gets instantiated to `wrapper(const wrapper &&)` and becomes an exact match for
`std::move(const_obj)`. For attempted move on const objects, we rely on the
implicit conversion from `const wrapper &&` to `const wrapper &` in order to
fallback to a copy.

The bottom line of the problem is that templates can be instantiate to a better
match than non-templates in some cases.

## Solutions

#### Pass by Value

The preferred solution here is to forget about perfect forwarding and just go
with the trivial _pass-by-value-then-move_ pattern. As long as `T` is movable,
we incur at most an extra move which is negligible in most cases anyway.

```c++
/* Pass-by-value constructor. */
wrapper(T value) : value(std::move(value)) {
  std::cout << "wrapper(T)" << std::endl;
}
```

> __NOTE:__ If `T` happens to be copyable but not movable, we incur an extra
> __copy__ rather than an extra move. I was concerned with this situation but I
> feel better after a
> [discussion](https://gist.github.com/mpark/2955020279470d2da354) with
> Sean Parent.

It's important to note that alternative solutions should match this constructor
in its behavior. I mean in regards to binding, rather than performance behavior.

#### Better Match

Another solution is to explicitly define the `wrapper(wrapper &)` constructor
and delegate to the `const` version. But then while we're at it we should also
add another one for `wrapper(const wrapper &&)`... mm... not liking this already.

How does it compare to the model solution?

Suppose `T` is unrelated to `wrapper`, and that `derived_wrapper` is a class
that publicly inherits from `wrapper`. If we attempt to copy-/move-construct
off of a `derived_wrapper`, under the pass-by-value solution it would bind to
the copy or move constructor. With this solution however, we would end up
binding to the perfect forwarding constructor again.

The situation I described above is admittedly obscure. The point I'm trying to
make here is that the templated constructor binds to __anything__, and
attempting to hard-code the cases won't work in general. A better approach is to
constrain the templated constructor to be less greedy.

#### SFINAE

The last solution I'll consider is to [SFINAE] out the templated constructor
from the overload set. If you've read the standards document and have seen:
_"This constructor shall not participate in overload resolution."_, this is what
it's talking about. We enable the templated constructor only if `U`
is convertible to `T`.

```c++
/* Constrained perfect forwarding constructor. */
template <
    typename U,
    std::enable_if_t<std::is_convertible<U, T>::value> * = nullptr>
wrapper(U &&u) : value(std::forward<U>(u)) {
  std::cout << "wrapper(U &&)" << std::endl;
}
```

Compared to the model solution?

This behaves equivalently to the pass-by-value pattern. For the pass-by-value
pattern we have `T` as the parameter, which implies that we can call it with
arguments that implicitly convertible to `T`. That semantic behavior is what
we have captured here with the constraint condition.

> __EDIT:__ The solution is actually __not__ equivalent to the pass-by-value
> pattern. Scott Meyers kindly pointed this out to me:
> "There are what I call '[perfect forwarding failure cases](
> https://groups.google.com/forum/#!topic/comp.std.c++/wu4r3CWG-VE)' that
> will succeed with pass-by-value but fail with perfect forwarding."

## Not New to C++11

It may seem like this is a new problem introduced in C++11, but it's actually
not a new problem. It's just that it is much easier to encounter it because
perfect-forwarding arguments is so tempting, whereas the C++98 way is perhaps
less tempting.

In C++98 and also in C++11, the following template would also beat the copy
constructor if a non-const lvalue is passed.

```c++
template <typename U>
wrapper(U &) {
  std::cout << "wrapper(U &)" << std::endl;
}
```

My point is that templates being instantiated to be better matching than
non-templates is not a new problem. It's not specific to perfect forwarding,
nor is it specific to copy constructors. Therefore, it shouldn't be remembered
as if it's a special case.

## Summary

* Avoid overloading templates and non-templates as the template could
  instantiateto be a better match (e.g. perfect-forwarding constructor) than the
  non-template which might rely on implicit conversions to handle all of its
  cases (e.g. copy constructor).
* Prefer to use the _pass-by-value-then-move_ pattern if possible. It's much
  simpler, more readable, and only incurs at most an extra move as long as the
  type is movable.
* If _pass-by-value-then-move_ is not an option, (e.g. variadic templates,
  efficiency) then consider the constraints that the pass-by-value version would
  impose and satisfy those requirements with template metaprogramming techniques
  such as _tag dispatch_, _SFINAE_ and _explicit/partial specializations_.

## Credits

The pitfalls of perfect-forwarding constructors have been discussed by others
already.

* [Some pitfalls with forwarding constructors] by R. Martinho Fernandes
* [Copying Constructors in C++11] by Scott Meyers
* [Universal References and the Copy Constructor] by Eric Niebler
* [Too perfect forwarding] by Andrzej Krzemieński

[Copying Constructors in C++11]: http://scottmeyers.blogspot.ca/2012/10/copying-constructors-in-c11.html
[SFINAE]: http://en.cppreference.com/w/cpp/language/sfinae
[Some pitfalls with forwarding constructors]: http://flamingdangerzone.com/cxx11/2012/06/05/is_related.html
[Too perfect forwarding]: http://akrzemi1.wordpress.com/2013/10/10/too-perfect-forwarding/
[Universal Reference]: http://channel9.msdn.com/Shows/Going+Deep/Cpp-and-Beyond-2012-Scott-Meyers-Universal-References-in-Cpp11
[Universal References and the Copy Constructor]: http://ericniebler.com/2013/08/07/universal-references-and-the-copy-constructo/
