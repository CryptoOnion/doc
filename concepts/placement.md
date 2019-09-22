---
layout: default
title: Placements
parent: Concepts
nav_order: 2
---
## Placements
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
The type in the placed annotations defines, where the function is placed. In the case above, the function
would be available on all `Registry`-peers and can be remote called on different peers. 
If the function should return Value, additional type hints are necessary:
```scala
def peerfunc2(): Int on Registry = placed {
    // code runs on registry and returns an Int
}
```

Notice that while the first example uses an `on[*]` block to tell Loci about the placement, the second example just uses `placed` which indicates a block of code that is going to be placed on a peer. The more specific `on[*]` can be omitted here because the type signature of the function already determines its placement.

## Variables
The syntax for variables is quite similar to the one for functions:
```scala
val chats: Array[Strings] on Registry = placed { new Array[Strings](10)}
```

## localOn
Variables and functions defined with `on` are visible and accessible by all peers. If only local access on the peer is needed,
a `Local[*]` type can be used.
```scala
val local_chats: Local[Array[Strings]] on Registry = placed { new Array[Strings](5)}
```
<div class="code-example" markdown="1">

Warning
{: .label .label-red }
Functions with `localOn` have to explicitly return (have an return statement), even if they do not have any return values.
</div>