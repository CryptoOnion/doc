---
layout: default
title: Module Mixin
parent: Concepts
nav_order: 5
---

# Module Mixin

{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Introduction

This guide shows some approaches how to combine multiple modules together, which are dependent on each other. Please
refer to the modules guide for an introduction into modules. 

Multitier objects that follow unidirectional access can access the peer defintions of their child modules. In the 
following we present soe approaches how to deal with cyclic references.

## Approaches

### Encapsulated

The first approach is about making independent modules that expose signals and events for their respective
in and output. Peer definitions then have to be independent from each other and can only be accessed
from the same multitier object.

```scala
CommandModule {
  // leave undefined
  val clientOutput: Event[String] on Client
}

SensoryModule {
  val readOutput: Event[String] on Controller = [...]
}

Main {
  val injectedOutput = on[Client] { Evt[String] }

  val sensors extends SensoryModule
  val commands extends CommandModule

 def main(): Unit on Client = {
   combined.readOutput.asLocal observer { injectedOutput fire }
 }
}
```

### Self types

The next approach uses the `Cake` pattern for `Scala`. This allows to swap modules like shown
[here](https://stackoverflow.com/a/5172697).

```scala
CommandModule {
  val clientInput: Signal[String] on Client = [...]
}

ControlModule {
  // dependency on command
  self: CommandModule =>

  val doSomethingAction: Event[String] on Controller = clientInput[...]
}

Main {
  @multitier object combined extends CommandModule with ControlModule
}
```
