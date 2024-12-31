---
layout: post
title: "Laminar Cheat Sheet"
date: 2024-12-30 09:48:55 +0200
tags: [ Scala, Laminar, ScalaJS ]
style: fill
color: secondary
comments: false
---

## Dependencies

```scala
"com.raquo" %%% "laminar" % "last.version"
```

maven repo: [https://mvnrepository.com/artifact/com.raquo/laminar](https://mvnrepository.com/artifact/com.raquo/laminar)

## Main

```scala
@main
def Main(): Unit =
  renderOnDomContentLoaded(
    dom.document.getElementById("app"),
    Root()
  )
```

or

```scala
@main
def Main(): Unit =
  windowEvents(_.onLoad).foreach { _ =>
    renderOnDomContentLoaded(
      dom.document.getElementById("app"),
      Root()
    )
  }
```

## Tags & Elements

`div` is a factory for `HtmlTag` and `div(...)` is an equivalent to `HtmlTag.apply(...)`

## Modifiers

`typ := "checkbox"` is an equivalent to `typ("checkbox")` and
`input(typ := "checkbox", defaultChecked := true)` equals to `input(typ("checkbox"), defaultChecked(true))`

Strings are not modifiers themselves, but are implicitly converted to `TextNode`

If you want Laminar to render your custom type `A` in a similar manner, define an
implicit instance of `RenderableText[A]` for it

`Seq[Modifier[A]]` is also implicitly converted to a `Modifier[A]` that applies all the modifiers in the `Seq`:
see `Implicits.seqToModifier`

`Implicits.optionToModifier` performs a similar conversion for `Option[Modifier[A]]`

There's also `emptyMod`

### Empty Nodes

```scala
val node: Node = if (foo) element else emptyNode
```

### Manual Application

To add modifiers to an already existing element, call `amend` on it:

```scala
val el = div("Hello")
// ...
el.amend(
  color := "red",
  span("a new child!")
)
```

or `amendThis`:

```scala
TextInput().amendThis { thisNode =>
  onInput.mapTo(thisNode.ref.value) --> nameVar
}
```

### inContext

in modern Laminar it's actually not needed

### Other Modifier Helpers

`nodeSeq` helps with type inference: `List("text", span("element"))`, whereas
`nodeSeq("text", span("element"))` returns the same exact list, but it's typed.

### Reusing Elements

**Do not**. An element can only have one parent element at a time

## Reactive Data

Please do read [Airstream documentation](https://github.com/raquo/Airstream)

### Attributes and Properties

Dynamic attributes are the simplest use case

```scala
val prettyColorStream: EventStream[String] = ???
/// ...
div(color <-- prettyColorStream, "Hello")
```

`<--` is defined on class `StyleProp`. It creates a `Binder` modifier that subscribes to the given stream.
This subscription is automatically enabled when the div element is added into the DOM,
and automatically disabled when the div is removed from the DOM.

For the modifier to do its job, you need to pass `color <-- prettyColorStream` to the element on which you
want to update the color property

### Event Streams and Signals

- **EventStream** is a lazy Observable _without_ a concept of "current value"
- **Signal** is a _also_ lazy Observable but _with_ a concept of "current value"

### Individual Children

There is a use case for streams of elements. If you need a different type of element displayed, you
will need to re-create the Laminar element

```scala
def MaybeBlogUrl(maybeUrlSignal: Signal[Option[String]]): Signal[HtmlElement] = {
  maybeUrlSignal.map {
    case Some(url) => a(href := url, "a blog")
    case None => i("no blog")
  }
}
// ...
val app: HtmlElement = div(
  "Hello, I have ",
  child <-- MaybeBlogUrl(maybeBlogUrlSignal), //maybeBlogUrlSignal is defined somewhere else
  ", isn't it great?"
)
```

#### Efficiency

A new element is created, even if we're moving from `Some(url1)` to `Some(url2)`, we might want to optimize for that

`split`/`splitOption` operators designed specifically to avoid re-rendering of elements in this kind of situation

```scala
def renderBlogLink(urlSignal: Signal[String]): HtmlElement = {
  a(href <-- urlSignal, "a blog")
}

def MaybeBlogUrl(maybeUrlSignal: Signal[Option[String]]): Signal[HtmlElement] = {
  lazy val noBlog = i("no blog")

  val maybeLinkSignal: Signal[Option[HtmlElement]] =
    maybeUrlSignal.splitOption((_, urlSignal) => renderBlogLink(urlSignal)) // splitOption is a hero

  maybeLinkSignal.map(_.getOrElse(noBlog))
}
```

It will be explained in detail later, but in short `renderBlogLink` is called only when
`maybeUrlSignal` switches from `None` to `Some(url)`.

Also note that we only create one `noBlog` element, and reuse it.

#### Optional children

If you do not want to render anything when the observable emits None, you can use `child.maybe`

```scala
val maybeBlogLink: Signal[Option[HtmlElement]] = ???
div(
  child.maybe <-- maybeBlogLink
)
```

#### Text and text-like values

To render dynamic text, do not do this

```scala
val textStream: EventStream[String] = ???
div(child <-- textStream.map(str => span(str))) // No! Bad!
```

This is needlessly creating a new span element instead of using the text receiver:

```scala
val textStream: EventStream[String] = ???
div(text <-- textStream)
div(child.text <-- textStream) // Same thing, original name from before v17
```

### Lists of Children

```scala
// Can be any Seq-like collection
val childrenSignal: Signal[List[Node]] = ???
div("Hello, ", children <-- childrenSignal)
```

Laminar only compares elements by reference equality, we do not diff any attributes

it would be very inefficient with naive code like

```scala
val userElementsStream: EventStream[List[Div]] =
  usersStream.map(users => users.map(renderUser))

div(
  children <-- userElementsStream
)
```

because such an implementation would
create N new `div` elements every time modelsSignal emits

_User/userStream/renderUser are defined somewhere else like this:_

```scala
case class User(id: String, name: String)

val usersStream: EventStream[List[User]] = ???

def renderUser(user: User): HtmlElement = {
  div(
    p("user id: ", user.id),
    p("name: ", user.name)
  )
}
```

Airstream's `split` operator was designed specifically for this use case

` users.map(renderUser)` should be replaced with `users.split(_.id)(newRenderUser)` that
creates a special kind of stream:

```scala
val userElementsSignal: EventStream[List[HtmlElement]] =
  usersStream.split(_.id)(newRenderUser)
```

where `newRenderUser` is defined like this:

```scala
def newRenderUser(userId: String, initialUser: User, userSignal: Signal[User]): HtmlElement = {
  div(
    p("user id: " + userId),
    p("name: ", text <-- userSignal.map(_.name))
  )
}
```

The rendering code stays the same:
```scala
div(
  children <-- userElementsSignal
)
```

More details of how `split` works:

 - when the `id` is first appeared in upstream, `newRenderUser` is called and creates a new `div`.
 - this `div` is then remembered
 - when the `id` is emitted again, `newRenderUser` is called again, but this time it updates the existing `div`
 - when the `id` is no longer present in the upstream, the `div` is removed from the DOM

For any given userId, we will only create and render a single element, and any subsequent updates to this user will be channeled into its individual userSignal

#### Splitting Without a Key

with splitByIndex you can use the index of the item in the list as its key.

#### Splitting Options and Other Types

`splitOption` is simply using the `Option`'s `.isDefined` as a key, creating a new element when switching from
`None` to `Some(model)`

#### Splitting Individual Items

`splitOne` works on observables of `Foo` itself

```scala
case class Editor(text: Boolean, isMultiLine: Boolean)

val inputSignal: Signal[Editor] = ???

def renderEditor(
                  isMultiLine: Boolean,
                  initialEditor: Editor,
                  editorSignal: Signal[Editor]
                ): HtmlElement = {
  val tag = if (isMultiLine) textArea else input
  tag(value <-- editorSignal.map(_.text))
}

val outputSignal: Signal[HtmlElement] =
  inputSignal.split(key = _.isMultiLine)(project = renderEditor)
```

#### Performant Children Rendering – children.command

```scala
val commandBus = new EventBus[ChildrenCommand]
val commandStream = commandBus.events
div("Hello, ", children.command <-- commandStream)
//...
commandBus.writer.onNext(ChildrenCommand.AppendChild(span("world")))  
// or
onClick.map { _ =>
  //...
  CollectionCommand.Append(span(s"Child # $counter"))
} --> commandBus
```

**You can mix both `child <-- ...` and `children <-- ...` and `children.command <-- ...`**

#### Rendering Mutable Collections

https://laminar.dev/documentation#rendering-mutable-collections

#### Inserters

#### Binding Observables

### Conditional Rendering
```scala
when(bool)
//...
whenNot(bool)
//...
if (bool) b(" world") else emptyNode
//...
if (bool)
  b("world") :: Nil
else
  div("foo") :: div("bar") :: Nil
//...
a(
  option.map(str => href := str), // conditionally apply a modifier 
  option.map(str => span(str))
)
//...
option.map(str => span(str)).getOrElse("– nothing –")
```

```scala
child <-- maybeElementSignal.map(_.getOrElse(span("no data")))
```

### Rendering Loading State
```scala
val veryImportant: Modifier[HtmlElement] = cls := "vip"
div(
  veryImportant,
  //...
)
```

`cls` modifier instead of setting the list of CSS classes appends `newClass` to the list of CSS classes already
present on the element

`rel` and `role` in Laminar are exactly identical to `cls`

## Event System: Emitters, Processors, Buses

### Registering a DOM Event Listener

### EventBus

```scala
val clickBus = new EventBus[dom.MouseEvent]

val element: Div = div(
  onClick --> clickBus.writer,
  //...
)
```

## SVG
```scala
  svg(
    height := "800",
    width := "500",
    polyline(
      points := "20,20 40,25 60,40",
      className := "someSvgClassName",
      L.onClick --> ???,
      L.children <-- ???
    )
  )
```
### Rendering External SVGs

## Network Requests
In Laminar, you can use Airstream's FetchStream:

```scala
div(
  // Make fetch request when this div element is mounted:
  FetchStream.get(url) --> { responseText => doSomething },
  // Make fetch request on every click:
  onClick.flatMap(_ => FetchStream.get(url)) --> { responseText => doSomething },
  // Same, but also get the click event:
  onClick.flatMap(ev => FetchStream.get(url).map((ev, _))) --> {
    case (ev, responseText) => doSomething
  }
)
```

### Codecs and Type-Safe Requests
you can use MessagePack or any other encoding instead of JSON). Typically, you would use a JSON library like jsoniter-scala or uPickle to derive codecs for Scala case classes 

### GraphQL

You can use Caliban as a type-safe GraphQL client – they have a Laminar integration that uses Laminext's WebSockets implementation mentioned above.

## URL Routing

[Waypoint](https://github.com/raquo/Waypoint) – Laminar URL router for Laminar.

[frountroute](https://github.com/tulz-app/frontroute) – Alternative router for Laminar with API inspired by Akka HTTP

