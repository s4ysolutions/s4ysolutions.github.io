---
layout: post
title: "Kotlin Iterable vs Sequence"
date: 2024-10-21 09:48:55 +0200
tags: [Kotlin]
style: fill
color: secondary
comments: false
---

Kotlin, known for its expressive syntax and pragmatic approach, has an intriguing detail in its
collections framework—both Iterable and Sequence exist as interfaces for iterating over elements.
The presence of these two often raises questions, as Iterable is widely used, while Sequence
tends to remain in its shadow, rarely mentioned.

![Iterable vs Sequence](/assets/images/kotlin/iterable-vs-sequence.svg)

Let’s explore the difference between Iterable and Sequence with a quick guessing game. Consider
the following two functions. They look almost identical, but their behavior is fundamentally
different:

```kotlin
fun iterable(): Iterable<Int> {
    val list = listOf(1, 2, 3, 4, 5)
    return list
        .map { println("Mapping $it"); it * 2 }
        .filter { println("Filtering $it"); it > 5 }
}

fun sequence(): Sequence<Int> {
    val sequence = sequenceOf(1, 2, 3, 4, 5)
    return sequence
        .map { println("Mapping $it"); it * 2 }
        .filter { println("Filtering $it"); it > 5 }
}
```
Both functions do the same thing, right? They take a list of numbers, double each one, and then
filter out those that are not greater than 5. But wait, here’s the twist: when you run these
functions, they behave very differently.

### A Surprising Observation
```kotlin
fun main() {
    println("Using Iterable:")
    iterable()
    println("Using Sequence:")
    sequence()
}
```

When running this code, the output might surprise anyone who assumes Iterable and Sequence behave the same way:

```
Using Iterable:
Mapping 1
Mapping 2
Mapping 3
Mapping 4
Mapping 5
Filtering 2
Filtering 4
Filtering 6
Filtering 8
Filtering 10

Using Sequence:
```

**Wait, what just happened?** The `iterable()` function produces a series of outputs as it maps
and filters the elements. But `sequence()`? Not a single line is printed!

### Why Does This Happen?

-	**Iterable behavior**: As soon as the `iterable()` function is called, the map and filter operations are
immediately applied to every element in the list, even if no one actually iterates over the result.
-	**Sequence behavior** The `sequence()` function doesn’t perform any operations until the Sequence
is actually consumed. Because there is no terminal operation like `toList()` or `forEach()` in this version,
the map and filter blocks remain completely unexecuted.

### Adding `toList()` to the Sequence

```kotlin
fun main() {
    println("Using Iterable:")
    iterable()

    println("\nUsing Sequence:")
    sequence().toList()
}
```

### The New Output

```
Using Iterable:
Mapping 1
Mapping 2
Mapping 3
Mapping 4
Mapping 5
Filtering 2
Filtering 4
Filtering 6
Filtering 8
Filtering 10

Using Sequence:
Mapping 1
Filtering 2
Mapping 2
Filtering 4
Mapping 3
Filtering 6
Mapping 4
Filtering 8
Mapping 5
Filtering 10
```

With the addition of `toList()`, the _Sequence_ is now evaluated, and its map and filter operations are
triggered. However, the way the output appears reveals a key difference between _Iterable_ and _Sequence_:

-	**Iterable eagerly evaluates**: All elements are processed through map first, followed by filter, even
though no result is directly used in main. This means that all mappings are executed before any filtering.
-	**Sequence lazily evaluates**: Each element is processed one at a time. First, an element is mapped,
then it’s immediately filtered before moving on to the next element. This interleaving of operations is
what produces the different order of Mapping and Filtering print statements.

### Playground

To see this difference in action, these examples can be tested directly using this link to the
[Kotlin Playground](https://play.kotlinlang.org/#eyJ2ZXJzaW9uIjoiMi4wLjIxIiwicGxhdGZvcm0iOiJqYXZhIiwiYXJncyI6IiIsIm5vbmVNYXJrZXJzIjp0cnVlLCJ0aGVtZSI6ImlkZWEiLCJjb2RlIjoiZnVuIGl0ZXJhYmxlKCk6IEl0ZXJhYmxlPEludD4ge1xuICAgIHZhbCBsaXN0ID0gbGlzdE9mKDEsIDIsIDMsIDQsIDUpXG4gICAgcmV0dXJuIGxpc3RcbiAgICAgICAgLm1hcCB7IHByaW50bG4oXCJNYXBwaW5nICRpdFwiKTsgaXQgKiAyIH1cbiAgICAgICAgLmZpbHRlciB7IHByaW50bG4oXCJGaWx0ZXJpbmcgJGl0XCIpOyBpdCA+IDUgfVxufVxuXG5mdW4gc2VxdWVuY2UoKTogU2VxdWVuY2U8SW50PiB7XG4gICAgdmFsIHNlcXVlbmNlID0gc2VxdWVuY2VPZigxLCAyLCAzLCA0LCA1KVxuICAgIHJldHVybiBzZXF1ZW5jZVxuICAgICAgICAubWFwIHsgcHJpbnRsbihcIk1hcHBpbmcgJGl0XCIpOyBpdCAqIDIgfVxuICAgICAgICAuZmlsdGVyIHsgcHJpbnRsbihcIkZpbHRlcmluZyAkaXRcIik7IGl0ID4gNSB9XG59XG5cbmZ1biBtYWluKCkge1xuICAgIGl0ZXJhYmxlKClcbiAgICBzZXF1ZW5jZSgpLnRvTGlzdCgpXG59In0=). 