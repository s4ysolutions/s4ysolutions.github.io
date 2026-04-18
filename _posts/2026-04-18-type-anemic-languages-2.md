---
layout: post
title: "Type Strength vs Type Anemia: a Missing Dimension in Static Typing"
date: 2026-04-18 10:00:00 +0000
tags: [ Type Systems, Java, Kotlin, Scala, Haskell, AI ]
style: fill
color: secondary
comments: false
---
## A Missing Dimension in Typing

Besides statically typed, dynamically typed, or untyped languages, there is another important dimension.
A language can be statically typed and still fail to enforce types as a real compile-time constraint.

The easiest way to understand this idea is with a simple example.

Imagine a service that integrates with a third-party system storing user data by email.
The external system must not receive raw email addresses because they are personal data.

The correct approach is straightforward: use a hashed external identifier instead of the raw email.

`externalId = hash(email)`

However, both `email` and `externalId` are usually represented as `String`. Nothing stops a developer
from accidentally passing the raw email to the external system instead of the hashed value.<sup>*</sup>

For a Haskell or Scala developer, this problem is usually easy to solve. It is common to create
separate types for `Email` and `ExternalId` (using `newtype` in Haskell or opaque types in Scala 3).
This makes the mistake impossible at compile time. Reusing a raw `String` for both concepts is often
seen as a design smell in these ecosystems.

Experienced Kotlin developers can (and should) use **value classes** for this kind of separation. It is a
well-known feature in the language, and using it is often expected once the concept is understood.

In Java, the traditional approach is still to use `String` for both. It is possible to wrap them in
domain-specific classes, but the language has not strongly encouraged this style historically. Even
when developers wanted stronger typing, there was no lightweight mechanism available, so people have
to use full class wrappers — which come with extra memory allocation and garbage collection overhead.

These three approaches show very different levels of type enforcement for the same problem. The
difference comes from how easily a language lets you create distinct domain types without runtime cost.

This is not just a theoretical curiosity. It becomes especially relevant in the AI era. Code suggested by AI
agents still needs careful human review — generated code cannot be trusted by default.

When primitives like `String` are used for semantically different concepts, the reviewer’s mental load increases.
Meaning must be reconstructed from context instead of being clearly expressed by the type system.
This raises the chance of missing mistakes or accepting wrong assumptions from the model.

Languages with stronger support for distinct domain types shift part of this burden away from the reviewer. The compiler enforces the boundaries, and the AI model itself gets clearer signals about the intended usage. In the best case, the type system adds a compile-time check that catches misuse before it ever reaches runtime.

---
<sup>*</sup> *Even if the probability of the mistake is low, the impact is high enough that it should be prevented.*
