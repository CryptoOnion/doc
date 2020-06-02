---
layout: default
title: Transmittables
parent: Concepts
nav_order: 3
---
# Transmittables

{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Primitives

Scala-Loci implements serialization support for a set of common objects. This includes primitives like
`String`, `Int` and often used collections like `Map`, `Option` and `Array`.

## Custom transmittables

However, if you require custom transmittable objects, you need to use custom serializer. In the following example,
we will use `upickle`[https://github.com/lihaoyi/upickle] for this. 

Here we want to transport this case class between peers. For simple objects we could use `IdenticallyTransmittable`,
this will serialize it as-is.
```scala
// import serializer lib
import upickle.default._

// the object we want to transport
final case class Failure(reason: String)

package object command {
  // mark it as transmittable without any custom logic
  implicit val resultTransmittable: IdenticallyTransmittable[Failure] = IdenticallyTransmittable()
  // create a upickle readwriter that should be used - here we can use the default one
  implicit val rwResult: ReadWriter[CommandResult] = macroRW
}
```

### Companion Objects

As an alternative to the above approach we could also use companion objects.  

```scala
object Failure {
  final implicit def transmittableResult: IdenticallyTransmittable[Failure] = IdenticallyTransmittable()
  implicit val rwResult: ReadWriter[Failure] = macroRW
}
```

## Common Issues

### NPE

If you use `implicit`s and inheritance for the transmittable objects, the order of the definitions is relevant for 
Scala. `ReadWriter`s should be defined from child to parent, or you will encounter a runtime exception. 
