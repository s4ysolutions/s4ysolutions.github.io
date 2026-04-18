---
layout: post
title: "Type Strength vs Type Anemia: a Missing Dimension in Static Typing"
date: 2026-04-18 10:00:00 +0000
tags: [ Type Systems, Java, Kotlin, Scala, Haskell ]
style: fill
color: secondary
comments: false
---

## A missing dimension in typing

Besides statically typed, dynamically typed, or untyped languages, there is another dimension.
A language may be statically typed, yet still not enforce types as a real compile-time constraint.

The easiest way to understand the idea is this sample.

Assume a service integrates with a third-party system that stores user data identified by email.
The external system must not receive raw email addresses, since they are considered personal data.

The intended approach is simple: use an external identifier instead of
email `externalId = hash(email)`

But both _email_ and _externalId_ are represented as `String`. Nothing prevents a developer
from accidentally passing an email to the external system instead of the hashed identifier <sup>*</sup>.


For a Haskell or Scala developer, this case is usually straightforward. It is common to define
separate types for `Email` and `ExternalId`, and then the mistake becomes impossible at
compile time. In these ecosystems, reusing raw `String` for both is often treated as a
design smell.

Experienced Kotlin developers can and should use inline/value types for this kind of separation.
It is a known tool in the language, and its use in cases like this is often expected once
the concept is understood.

In Java, the default approach is still to keep both as `String`. Wrapping them into domain types
is possible, but historically the language has not pushed strongly toward this style of modeling.
Even when developers want stronger typing, current Java versions offered no lightweight
mechanism for it, and the common alternative was full class wrappers with additional
allocation and GC overhead.

Across these examples, the same idea produces different levels of type enforcement depending
on how easily a language supports distinct domain types.

Why it is not just a curiosity but becomes relevant in the AI era is that code suggested by agents
still requires thorough review. In general, generated code cannot be trusted by default.
When primitives are used for semantically different concepts, the reviewer’s mental load increases:
meaning must be reconstructed from context rather than carried by the type system.
This also increases the chance of overlooking mistakes or accepting incorrect assumptions produced by the model.

With languages that have stronger type systems, part of this burden is shifted away from review. The compiler
enforces boundaries, and the model itself receives clearer signals about intended usage.
In the worst case, the type system adds an additional compile-time check
that catches misuse before it reaches runtime.
---
<sup>*</sup> *despite the probability being low, the impact is high enough that it should be avoided*
