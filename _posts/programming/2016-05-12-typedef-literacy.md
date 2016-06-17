---
layout: post
title: "Typedef Literacy"
modified:
categories: programming
excerpt: "Do you know how to read/write `typedef` properly?"
tags: [c++, typedef, type-alias]
date: 2016-05-12
share: true
---

__NOTE__: If you're writing ≥ C++11, reach for [type alias] rather than
typedef declaration. The syntax matches what people naturally expect, and
it also supports templated type aliases (i.e. [alias template][type alias]).

## Introduction

The typedef declaration provides a way to create an alias for an existing
type. For example, we can provide an alias for `int` called `integer` like so:

```c++
typedef int integer;
```

I imagine most people have seen such declarations, and they are fairly simple to
read. In fact it's so simple that we may incorrectly conclude that the syntax
for `typedef` is:

```c++
typedef <existing_type> <new_type_name>;
```

This syntax works for simple cases, but at some point we encounter a typedef
declaration that doesn't quite meet our expectations:

```c++
typedef int (*function_pointer)(double);
```

This declares an alias for `int (*)(double)` (pointer to function) called
`function_pointer`. This is clearly inconsistent with our earlier conclusion,
since if it were, we would have seen the following instead:

```c++
typedef int (*)(double) function_pointer;
```

In an effort to keep the simple cases simple, many people understandably start
to treat this as a special case. Until they encounter another weird one:

```c++
typedef int array[3];
```

Fine! Yet another special case. But what about...

```c++
typedef int integer, (*function_pointer)(double), array[3];
```

Huh...? Is this even legal? --- Yes. Yes it is. ---
I'm __NOT__ suggesting that you do this!

Seriously, how do we read these damn things?

## Correct Reading

The misconception and mystery behind the reading of the typedef declaration lies
in the seemingly innocent, but misleading simple examples. Here is the syntax
for the typedef declaration according to [cppreference]:

```c++
typedef type_declaration;
```

The _type_declaration_ after the `typedef` keyword is a simple declaration
with some restrictions (e.g. cannot be declared `static`).

> Refer to the [Standardese](#standardese) section if you're interested in
> some of the fine details.

Consider the following variable declarations:

```c++
int i;
int (*f)(double);
int a[3];
```

In variable declarations, the introduced names are __instances__ of
the corresponding types.

```c++
int i;             // `i` is an instance of `int`.
int (*f)(double);  // `f` is an instance of `int (*)(double)`.
int a[3];          // `a` is an instance of `int [3]`.
```

However, when the `typedef` keyword precedes the declaration, the introduced
names are __aliases__ of the corresponding types.

```c++
typedef int i;             // `i` is an alias of `int`.
typedef int (*f)(double);  // `f` is an alias of `int (*)(double)`.
typedef int a[3];          // `a` is an alias of `int [3]`.
```

Thus, the task of reading a typedef declaration can be delegated to a reading
of a variable declaration then reinterpreting the names to be __aliases__ of the
corresponding types as opposed to __instances__ of them!

## Inaccurate Sources

Unfortunately, the source of the misunderstanding goes beyond us naively
assuming the simple syntax.

- [Wikipedia article] enumerates a bunch of scenarios. It makes it seem as
  if there is a special case for function pointers, for example.

> Function pointers are somewhat different than all other types because the
> syntax does not follow the pattern `typedef <old type name> <new alias>;`.
> Instead, the new alias for the type appears in the middle between the return
> type (on the left) and the argument types (on the right). [...]

* [Cplusplus tutorial] flat out incorrectly says:

> In C++, there are two syntaxes for creating such type aliases: The first,
> inherited from the C language, uses the `typedef` keyword:
>
> `typedef existing_type new_type_name ;`
>
> where `existing_type` is any type, either fundamental or compound, and
> `new_type_name` is an identifier with the new name given to the type.

## Conclusion

Hopefully I've helped you understand how typedef declarations actually work.
As I mentioned at the beginning of the post, reach for [type alias] instead if
you're writing modern C++.

As we've seen in the earlier examples, the design of typedef declaration
successfully achieves the "make simple things simple" principle. However, the
generalized rule that one would naturally deduce from the simple cases do not
prepare you for the complex cases at all.

I've also shown that the underlying rule is actually quite simple and
consistent. The level of confusion that arise from what seems like such a
coherent and simple design is quite intriguing.

## Standardese

The formal definition of the syntax can be found in __§7 Declaration__:

- __§7 ¶1__:

$$
\begin{align*}
\textit{simp}&\textit{le-declaration:} \\
             &\textit{decl-specifier-seq} \space \textit{init-declarator-list}_{opt}\textit{;} \\
             &\textit{attribute-specifier-seq} \space \textit{decl-specifier-seq}\space\textit{init-declarator-list;} \\
\end{align*}
$$

- __§7 ¶9__: If the _decl-specifier-seq_ contains the __typedef__ specifier,
  the declaration is called a _typedef declaration_ and the name of each
  _init-declarator_ is declared to be a _typedef-name_, synonymous with its
  associated type (7.1.3). [...]

- __§7.1 ¶1__:

$$
\begin{align*}
\textit{decl}&\textit{-specifier:} \\
             &\textit{storage-class-specifier} \\
             &\textit{defining-type-specifier} \\
             &\textit{function-specifier} \\
             &\textbf{friend} \\
             &\textbf{typedef} \\
             &\textbf{constexpr} \\

\textit{decl}&\textit{-specifier-seq:} \\
             &\textit{decl-specifier} \space \textit{attribute-specifier-seq}_{opt} \\
             &\textit{decl-specifier} \space \textit{decl-specifier-seq} \\
\end{align*}
$$

- __§7.1.3 ¶1__: Declarations containing the _decl-specifier_ __typedef__
  declare identifiers that can be used later for naming fundamental (3.9.1) or
  compound (3.9.2) types. The __typedef__ specifier shall not be combined in a
  _decl-specifier-seq_ with any other kind of specifier except a
  _defining-type-specifier_, [...] If a __typedef__ specifier appears in a
  declaration without a _declarator_, the program is ill-formed.

  $$
  \begin{align}
  \textit{type}&\textit{def-name:} \\
               &\textit{identifier} \\
  \end{align}
  $$

  A name declared with the __typedef__ specifier becomes a _typedef-name_.
  Within the scope of its declaration, a _typedef-name_ is syntactically
  equivalent to a keyword and names the type associated with the identifier in
  the way described in Clause 8. A _typedef-name_ is thus a synonym for another
  type. A _typedef-name_ does not introduce a new type the way a class
  declaration (9.1) or enum declaration does. [...]

[type alias]: http://en.cppreference.com/w/cpp/language/type_alias
[cppreference]: http://en.cppreference.com/w/cpp/language/typedef
[Wikipedia article]: https://en.wikipedia.org/wiki/Typedef
[Cplusplus tutorial]: http://www.cplusplus.com/doc/tutorial/other_data_types/
