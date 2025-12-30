---
layout: post
title: "Expression Problem in Java using Tagless Final. Part 1"
date: 2025-12-29 10:00:00 +0000
tags: [ Java, Functional Programming, Tagless Final, Expression Problem ]
style: fill
color: secondary
comments: false
---
## Theory: The expression problem

The expression problem asks: how can we add new data types (kinds) and new operations
on those types without modifying existing code, while maintaining static type safety?

## Practice: The real world problem of the extensible filesystem objects

Let's have filesystem objects:

Kinds (variants, types, java classes):

- Text
- SQL
- Script (added later)

Operations:

- change
- verify
- execute (only SQL + Script, added later)

This is exactly the expression problem:

- add new variants
- add new operations
- with static (compile time) guarantees

### Possible solutions

#### 1: Classic OOP with inheritance and interfaces

initially:

```java
sealed interface FsObject permits TextObj, SqlObj {
    void change(String newContent);

    boolean verify();
}

final class TextObj implements FsObject {
    String text;

    public void change(String newContent) {
        this.text = newContent;
    }

    public boolean verify() {
        return !text.isEmpty();
    }
}

final class SqlObj implements FsObject {
    String query;

    public void change(String newContent) {
        this.query = newContent;
    }

    public boolean verify() {
        return query.endsWith(";");
    }
}
```

later,
add `ScriptObj`, it is easy:

```java
final class ScriptObj implements FsObject { ...
}
```

but add execute operation is hard: one needs to modify the FsObject interface and all implementations:

```java
interface FsObject {
    void change(String newContent);

    boolean verify();

    void execute(); // <-- forced
}
```

and even worse `TextObj` cannot execute, so either throw exception:

```java
final class TextObj implements FsObject {
    ...

    public void execute() {
        throw new UnsupportedOperationException("TextObj cannot be executed");
    }
}
```

or introduce a new(marker) interface:

```java
interface Executable extends FsObject {
    void execute();
}
```

and use it with runtime type check and cast:

```java
void runIfExecutable(FsObject obj) {
    if (obj instanceof Executable execObj) {
        execObj.execute();
    } else {
        throw new IllegalArgumentException("Object is not executable");
    }
}
```

*Summary:* easy to add new kinds, hard to add new operations, partial operations - awkward and
not compile time-safe.

#### 2: Functional approach with Algebraic Data Types (ADTs) and pattern matching

Initially:

ADTs:

```java
sealed interface FsObject permits TextObj, SqlObj {
}

record TextObj(String text) implements FsObject {
}

record SqlObj(String query) implements FsObject {
}
```

operations as functions:

```java
static FsObject change(FsObject o, String newContent) {
    return switch (o) {
        case TextObj t -> new TextObj(newContent);
        case SqlObj s -> new SqlObj(newContent);
    };
}

static boolean verify(FsObject o) {
    return switch (o) {
        case TextObj t -> !t.text().isEmpty();
        case SqlObj s -> s.query().endsWith(";");
    };
}
```

Later, add new operation is as easy as

```java
static void execute(FsObject o) {
    switch (o) {
        case SqlObj s -> runSql(s.query());
        default -> throw new UnsupportedOperationException();
    }
}
```

But adding `ScriptObj`:

```java
record ScriptObj(String script) implements FsObject {
}
```

causes compile-time errors in all operations because the `switch` is no longer exhaustive,
which is good, but all operations need to be modified to handle the new kind.

*Summary:* easy to add new operations, hard to add new kinds, partial operations - still awkward and not
compile time-safe.

#### 3: Capabilities interface, real solution to the expression problem

Define the capabilities as interfaces parametrized by the kind `F`:

```java
interface Changeable<F> {
    F change(F obj, String newContent);
}

interface Verifiable<F> {
    boolean verify(F obj);
}
```

Having this interfaces, we already can use them in the "programs":

```java
static <F> F updateAndVerify(
        F obj,
        String newContent,
        Changeable<F> changer,
        Verifiable<F> verifier
) {
    F updatedObj = changer.change(obj, newContent);
    boolean isValid = verifier.verify(updatedObj);
    if (!isValid) {
        throw new IllegalArgumentException("Invalid object after update");
    }
    return updatedObj;
}
```

Adding the new operation is as easy as defining a new interface:

```java
interface Executable<F> {
    void execute(F obj);
}
```

And it can be used in the programs that need it:

```java
static <F> void runIfExecutable(
        F obj,
        Executable<F> executor
) {
    executor.execute(obj);
}
```

From this point the programs stay the same for virtually any kind `F`, and we can add new kinds
without modifying existing code.

To run the programs, we need to provide implementations of the capability interfaces for each kind.

For `TextObj` and `SqlObj`:

```java
class TextObjChanger implements Changeable<TextObj> {
    public TextObj change(TextObj obj, String newContent) {
        return new TextObj(newContent);
    }
}

class TextObjVerifier implements Verifiable<TextObj> {
    public boolean verify(TextObj obj) {
        return !obj.text().isEmpty();
    }
}

class SqlObjChanger implements Changeable<SqlObj> {
    public SqlObj change(SqlObj obj, String newContent) {
        return new SqlObj(newContent);
    }
}

class SqlObjVerifier implements Verifiable<SqlObj> {
    public boolean verify(SqlObj obj) {
        return obj.query().endsWith(";");
    }
}
```

And use them:

```java
// instance of some FsObject
TextObj textObj = new TextObj("Hello, World!");
// Capability implementations (interpreters)
Changeable<TextObj> textChanger = new TextObjChanger();
Verifiable<TextObj> textVerifier = new TextObjVerifier();
// Using the program
TextObj updatedTextObj = updateAndVerify(textObj, "New Content", textChanger, textVerifier);

// use the same program for SqlObj
SqlObj sqlObj = new SqlObj("SELECT * FROM users;");
Changeable<SqlObj> sqlChanger = new SqlObjChanger();
Verifiable<SqlObj> sqlVerifier = new SqlObjVerifier();
SqlObj updatedSqlObj = updateAndVerify(sqlObj, "SELECT * FROM orders;", sqlChanger, sqlVerifier);
```

Extending the system with the new kind `ScriptObj` is just adding the
capability implementations(interpreters):

```java
class ScriptObjChanger implements Changeable<ScriptObj> {
    public ScriptObj change(ScriptObj obj, String newContent) {
        return new ScriptObj(newContent);
    }
}

class ScriptObjVerifier implements Verifiable<ScriptObj> {
    public boolean verify(ScriptObj obj) {
        return obj.script().startsWith("#!");
    }
}
```

and extending the existing kinds SqlObj and ScriptObj with the Executable capability:

```java
class SqlObjExecutor implements Executable<SqlObj> {
    public void execute(SqlObj obj) {
        runSql(obj.query());
    }
}

class ScriptObjExecutor implements Executable<ScriptObj> {
    public void execute(ScriptObj obj) {
        runScript(obj.script());
    }
}
```

And the benefit is that the existing programs do not need to be modified, and their use is
still type-safe at compile time: `updateAndVerify` works for any kind `F` that has
`Changeable<F>` and `Verifiable<F>` implementations only and `runIfExecutable` works for any kind `F`
that has `Executable<F>` implementation only.

*Summary:* easy to add new kinds, easy to add new operations, partial operations - clean and compile time-safe.

## Theory: Tagless Final

The latter approach is known as Tagless Final in the functional programming community, and it leverages
parametric polymorphism (generics in Java) to achieve extensibility in both dimensions: kinds and operations
without virtual methods and runtime type casts.

The name "Tagless Final" comes from the historical reasons and has almost no meaning for modern programming languages
with the native generics support and with AST is not used heavily.

In the pre-generics era (and in the nowadays in TypeScript/JavaScript), the data structures had to carry "tags"
(discriminators) to identify their kinds at runtime:

```typescript
type FsObject =
    | { type: "TextObj"; text: string }
    | { type: "SqlObj"; query: string };
```

Generics eliminate the need for such tags, so the data structures are "tagless".

The “Final” part of the name comes from the fact that the operations are defined as functions that
evaluate directly to final results, rather than to intermediate representations
(such as ASTs, as used in some JSON frameworks that preserve an intermediate structure after parsing
and before producing the target value). Using intermediate representations would require an additional
interpretation step, which this approach avoids (while still allowing them).



[Part 2](expression-problem-tagless_final-java-2)
