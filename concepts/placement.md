---
layout: default
title: Placements
parent: Concepts
nav_order: 2
---
<h1>Placements</h1>
A placement in ScalaLoci defines the location where a function or a variable is located.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}


## Functions
For a function without return values, this can be defined like this:
```scala
def peerfunc() = on[Registry] {
    // code runs on registry
}
```
In the case above, the function
would be available on all `Registry`-peers and can be remote called on different peers. 
If the function should return Value, additional type annotations are necessary:
```scala
def peerfunc2(): Int on Registry = placed {
    // code runs on registry and returns an Int
}
```

Notice that while the first example uses an `on[*]` block to tell Loci about the placement, the second example just uses `placed` which indicates a block of code that is going to be placed on a peer. The more specific `on[*]` can be omitted here because the type signature of the function already determines its placement.

## Variables
The syntax for variables is quite similar to the one for functions:

```scala
val chats = on[Registry] { new Array[String](10)}
```

```scala
val chats: Array[String] on Registry = placed { new Array[String](10)}
```

## Subjective Values
By default, every peer accessing a shared value will receive the same data. In some cases, it is beneficial to return a 
different value based on the accessing peer instance. In this case subjective values can be used.

The `sbj` modifier binds an identifier holding a reference to the peer instance that accesses the subjective value. 
This identifier can be used to operate (e.g. mapping) on a per remote peer basis. The values are created and saved
on a per remote basis. Peers can then request the value (e.g. using `asLocal`) in order to transmit the value from
the peer, that holds the specific value, to the request issuer. 

For example, in the following code snippet, a `Client` accessing `publicMessage` will receive all `message` values of 
other `Client` instances, but not his own. Internally there are multiple `publicMessage` values, one for each connected
peer. Code inside the `sbj` runs locally (here: on the Server) and updates the internal values one by one.

```scala
val message = on[Client] { Evt[String] }

val publicMessage = on[Server] sbj { client: Remote[Client] =>
  message.asLocalFromAllSeq collect {
    case (remote, message) if remote != client => message
  }
}
```

## Local-Only Placement
Variables and functions defined with `on` are visible and accessible by all peers. If only local access on the peer is needed, a `Local[*]` type can be used.
```scala
val local_chats: Local[Array[String]] on Registry = placed { new Array[String](5)}
```
The `local` keyword can also be used instead, in which case Scala will infer the type automatically. It behaves similarly to `placed` expressions.
```scala
val local_chats = on[Registry] local { new Array[String](5)}
```
<div class="code-example" markdown="1">

Warning
{: .label .label-red }
Functions with `Local` have to explicitly return (have an return statement), even if they do not have any return values.
</div>
