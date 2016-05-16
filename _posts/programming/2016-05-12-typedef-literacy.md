---
layout: post
title: "Typedef Literacy"
modified:
categories: programming
excerpt: "Do you know how to read/write `typedef` properly?"
tags: [c++, typedef, type-alias]
date: 2016-05-12
share: true
draft: true
---

__NOTE__: If you're writing >= C++11, reach for [type alias] rather than
typedef declaration. The syntax matches what people naturally expect, and
it also supports templated type aliases (i.e. alias template).

## Introduction

The typedef declaration provides a way to declare an alias for an existing
type. For example, we can provide an alias for `int` called `integer` like so:

```c++
typedef int integer;
```

I imagine most people have seen such declarations, and it is fairly simple to
read.  In fact it's so simple that we may conclude that the syntax for `typedef`
is:

```c++
typedef <from> <to>;
```

This syntax works for simple cases, but at some point we encounter a typedef
declaration that doesn't quite meet our expectation:

```c++
typedef int (*function_pointer)(float, double);
```

This declares an alias for `int (*)(float, double)` (pointer to function) called
`function_pointer`. This clearly isn't consistent with our earlier conclusion,
since if it were, we would have seen the following instead:

```c++
typedef int (*)(float, double) function_pointer;
```

In an effort to keep the simple cases simple, people understandably start to
treat this as a special case. Until they encounter another weird one.

```c++
typedef int array[3];
```

Fine. Yet another special case. But what about...

```c++
typedef int integer, (*function_pointer)(int, double), array[3];
```

Huh...? Is this even legal? --- Yes. Yes it is. (I'm not saying do it...)

How do we read these damn things?

## Correct Syntax

The core of the problem lies in the seemingly innocent and simple examples that
lead us to naively assume the simple syntax. Here's the correct syntax for the
typedef declaration according to [cppreference]:

```c++
typedef type_declaration;
```

The `type_declaration` after the `typedef` keyword is a simple declaration
with some restrictions (e.g. cannot be declared `static`). In a variable
declaration, the associated names are instances of the corresponding types.
When the `typedef` keyword precedes the declaration, the associated names are
aliases of the corresponding types.

For example, `array` in `int array[3];` is an instance of `int [3]`. When the
`typedef` keyword appears before the declaration (i.e. `typedef int array[3];`),
`array` becomes an alias for the type `int [3]`.

Thus, the task of reading a typedef declaration can be delegated to a reading
a variable declaration then reinterpreting the names!

## Inaccurate Sources

Unfortunately, the source of the misunderstanding goes beyond us naively
assuming the simple syntax.

- [Wikipedia article] enumerates a bunch of scenarios and makess it seem as
  if there are special cases, for function pointers for example.

> Function pointers are somewhat different than all other types because the
> syntax does not follow the pattern `typedef <old type name> <new alias>;`.
> Instead, the new alias for the type appears in the middle between the return
> type (on the left) and the argument types (on the right). ...

* [Cplusplus tutorial] flat out incorrectly says:

> In C++, there are two syntaxes for creating such type aliases: The first,
> inherited from the C language, uses the `typedef` keyword:
>
> `typedef existing_type new_type_name ;`
>
> where `existing_type` is any type, either fundamental or compound, and
> `new_type_name` is an identifier with the new name given to the type.

## Conclusion

Hopefully I've helped in your ability to reason about `typedef` declarations.
As I mentioned at the beginning of the post, reach for [type alias] instead if
you're writing modern C++. What's interesting about `typedef` to me is that even
though it successfully achieves the "make simple things simple" principle, the
rule that one would naturally deduce from the simple case doesn't prepare you
for the complex cases at all. I've also shown that the underlying rule is
actually quite simple and consistent. It's an interesting result from a
coherent design that managed to keep the simple cases simple.

## Standardese

The formal syntax can be found in §7 Declaration:

- §7 ¶1:

$$
\begin{align}
\textit{simp}&\textit{le-declaration:} \\
             &\textit{decl-specifier-seq} \space \textit{init-declarator-list}_{opt}\textit{;} \\
             &\textit{attribute-specifier-seq} \space \textit{decl-specifier-seq}\space\textit{init-declarator-list;} \\
\end{align}
$$

- §7 ¶9: If the _decl-specifier-seq_ contains the __typedef__ specifier, the
  declaration is called a _typedef declaration_ and the name of each
  _init-declarator_ is declared to be a _typedef-name_, synonymous with its
  associated type (7.1.3).  ...

- §7.1.3 ¶1: Declarations containing the _decl-specifier_ __typedef__ declare
  identifiers that can be used later for naming fundamental (3.9.1) or compound
  (3.9.2) types. ...

[type alias]: http://en.cppreference.com/w/cpp/language/type_alias
[cppreference]: http://en.cppreference.com/w/cpp/language/typedef
[Wikipedia article]: https://en.wikipedia.org/wiki/Typedef
[Cplusplus tutorial]: http://www.cplusplus.com/doc/tutorial/other_data_types/
