---
layout: default
title: FAQ
nav_order: 4
---
<h1>FAQ</h1>
{: .no_toc }

Here you can find a few common questions regarding the framework.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## My peers mysteriously stop working without any error message

This might be caused by an uncaught exception. If an exceptions occurs and is not properly handled, a node can crash without any warnings. To find the cause of the exception, you can try observing the reactives:

```scala
myEvent observe println
mySignal observe println
```

Usually, the observe notices the exception and prints its message and stacktrace to the console.

Does your program throw an exception when you `fire` an `Evt`? Remember to check the depending reactives for problems!

## Forward references to reactives

When aggregating reactives (e.g. asLocal, asLocalFromAll, ...) it is ok to reference reactives that are defined later in the source code. Otherwise, if you want a reactive A to depend on a reactive B, A has to come after B in the source code.

## Is it possible to pass a peer values through a constructor?

Yes, sort of. You need to give your peer trait a value member, which you can then set when the peer is created:

```scala
@multitier
object Example {
    trait Node extends Peer { type Tie <: Multiple[Node]
        val givenId: Int}

    val id = placed[Node] { Var[Int](peer.givenId) }

    ...
}

object ExampleNode extends App {
    multitier setup new Example.Node {
        def connect = ...

        val computedId = ...

        override val givenId: Int = computedId
    }
}
```

## What does this error message mean: `value asLocalFromAll is not a member of loci.on[...]`

Sometimes the compiler gives incomprehensible error messages inside placed expressions. You can help the compiler by replacing `placed[...] { ... }` with `placed[...] { implicit! => ... }`. After doing this, you will often get more comprehensive error messages.

## What does this error message mean: `... is not marshallable`

When using the uPickle serializer, not all values of all types can be serialized "out of the box". For example, if you want to transmit a case class, you need to define a serializer in its companion object:

```scala
import upickle.default.{ReadWriter => RW, macroRW}
...
case class Example(...)
object Example {
    implicit val rw: RW[Example] = macroRW
}
```

Make sure that this definition is placed outside of any multitier environments.

## How should I use signals in collect/filter?

Currently, you cannot use `Signal.apply` when constructing an Event with `collect` or `filter`, which makes it difficult to establish a reactive dependency. However, you can use `Signal.apply` in `map`, so in the following code

```scala
val sig = ...

val someEvent = ...

val filteredEvent = placed[...] {
    someEvent filter {
        case v => v == sig()
    }
}
```

`filteredEvent` can be rewritten as

```scala
val filteredEvent = placed[...] {
    someEvent map { (_, sig()) } collect {
        case (v, s) if v == s => v
    }
}
```

or

```scala
val filteredEvent = placed[...] {
    someEvent map { (_, sig()) } filter {
        case (v, s) => v == s
    } map { _.1 }
}
```

The trick is to map `someEvent` into a pair with the current value of `sig`. Afterwards, the signal value can be used by deconstructing the pair in a `case` expression. Since usually, the signal value should no longer be present in the derived event (`filteredEvent`), you have to either use another map after the filter expression or use `collect` instead of `filter`. 

## I get an exception when folding an event, but the body of the fold doesn't even get executed

You can try replacing `event.fold(init) { ... }` with `Events.foldOne(event, init) { ... }`

## Where are my exceptions?

Per default, exceptions are not printed in Loci. So if your application just stops working, be sure to handle all possible exceptions yourself. For example, `.asLocal_?` might throw an exception if the Await times out. Therefore, it can help to wrap it in a `Try {}`.