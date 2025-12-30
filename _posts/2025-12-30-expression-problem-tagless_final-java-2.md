---
layout: post
title: "Expression Problem in Java using Tagless Final. Part 2"
date: 2025-12-30 10:00:00 +0000
tags: [ Java, Functional Programming, Tagless Final, Expression Problem ]
style: fill
color: secondary
comments: false
---
## Practice: constructing FsObject kinds inside the programs

In the previous part, we instantiated the FsObject kinds with the `new` keyword directly and then
pass to the programs as a parameter, but what if we want to create them inside the programs?

The answer is to have an interface that defines the constructors for each kind:

```java
interface FsObjectAlgebra<F> {
    F createTextObj(String text);

    F createSqlObj(String query);
}
```

And then the programs can use this algebra to create the objects:

```java
static <F> F createAndVerifyTextObj(
        String text,
        FsObjectAlgebra<F> algebra,
        Verifiable<F> verifier
) {
    F obj = algebra.createTextObj(text);
    boolean isValid = verifier.verify(obj);
    if (!isValid) {
        throw new IllegalArgumentException("Invalid object");
    }
    return obj;
}
``` 

And now if we have the new kind `ScriptObj`, we just need to extend the algebra:

```java
interface FsObjectAlgebra<F> {
    F createTextObj(String text);
    F createSqlObj(String query);
    F createScriptObj(String script); // new kind
}
```

Despite the algebra being extended, the existing programs do not need to be modified,
and the compiler starts to complain if the existing implementation(s) of the interface
lacks the handling of new kind. This is the place where the static (compile time) guarantees
come into play and make sure we never face the dreadful runtime errors due to missing cases
in the pattern matching `switch` statements for example.

## Theory: Algebras and interpreters

From the high-school math perspective (extremely zoomed out), a set of operations defined on a certain set
forms an algebraic structure called an algebra. Treating Java classes as set of `.class` types and
looking back at the examples above, we can see that the interface methods define operations
that transform values from some Java classes to other Java classes (or the same class),
thus forming an algebra.

This is why these constructor and capabilities interfaces are called algebras in functional
programming jargon.

In the examples above I defined the interfaces with a very narrowed purpose, but in real-world
applications they are often grouped together forming the single interface that defines the whole
algebra for the domain or a "domain language".

When a program uses algebra interface(s), it calls methods on a concrete implementation, which
interprets the program logic in a specific way. This is why concrete implementations of algebra
interfaces are called interpreters in the same functional programming jargon.


[Part 1](expression-problem-tagless_final-java)
