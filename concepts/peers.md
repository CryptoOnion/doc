---
layout: default
title: Peers and Ties
parent: Concepts
nav_order: 1
---

<h1>Peers and Ties</h1>

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Peer Definition

Using the `Peer` type, a programmer can specify the different peers needed for the program.
The relation between them is specified using the `Tie` type.
An example for this is a typical client-server relationship (where one server is connected to multiple clients), which can be defined like this:
```scala
@peer type Peer
@peer type Server <: Peer {
  type Tie <: Multiple[Client]
}
@peer type Client <: Peer {
  type Tie <: Single[Server]    
}
```

## Multiple Connections per Peer

Additional connections can be specified by adding them with the `with` keyword:
```scala
type Tie <: Multiple[Client] with Single[Registry]
```

## Optional Connections
If a tie might not be needed during the complete runtime of a program (e.g. if an introducer is used),
it can be specified using an `Optional` tie:
```scala
type Tie <: Multiple[Client] with Optional[Registry]
```
In this case the connection to the Registry might not be available, which has to be handled by the programmer.