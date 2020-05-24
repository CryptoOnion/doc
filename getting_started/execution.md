---
layout: default
title: 4. Running your Application
parent: Getting Started
nav_order: 4
---

<h1>4. Running your Application </h1>
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Making your Application executable

In order to make your application executable, you have to implement one or more objects that extend the `App` class. Normally you will define one App for each of your peer types.

Additionally, you will have to run `multitier start` to start an instance of one of the previously defined peer types. Depending on the selected communication backend you will also have to establish a connection between the instances. For the TCP backend, this is demonstrated below:

```scala
object Server extends App {
  multitier start new Instance[Chat.Server](
    listen[Chat.Client] { TCP(43053) })
}

object Client extends App {
  multitier start new Instance[Chat.Client](
    connect[Chat.Server] { TCP("localhost", 43053) })
}
```

Notice how the **Server** instance uses `listen[Chat.Client]` to open a port for **Client** instances while the **Clients** use `connect[Chat.Server]` to connect to the server.

## Overview of the Chat Application

If you got everything right, the complete chat application should look like this now:

```scala
package chatsimple

import loci._
import loci.transmitter.rescala._
import loci.serializer.upickle._
import loci.communicator.tcp._

import rescala.default._


@multitier object Chat {
  @peer type Server <: { type Tie <: Multiple[Client] }
  @peer type Client <: { type Tie <: Single[Server] }

  val message = on[Client] { Evt[String] }

  val publicMessage = on[Server] sbj { client: Remote[Client] =>
    message.asLocalFromAllSeq collect {
      case (remote, message) if remote != client => message
    }
  }

  def main() = on[Client] {
    publicMessage.asLocal observe println

    for (line <- scala.io.Source.stdin.getLines)
      message.fire(line)
  }
}


object Server extends App {
  multitier start new Instance[Chat.Server](
    listen[Chat.Client] { TCP(43053) })
}

object Client extends App {
  multitier start new Instance[Chat.Client](
    connect[Chat.Server] { TCP("localhost", 43053) })
}
```

## Running the Application using sbt

To run the above application just type

```bash
$ sbt run
```

sbt should detect all implemented main classes and will let you select which one you want to execute.

If you want to build more sophisticated applications or simply want to learn more about ScalaLoci, have a look at the [Concepts Section](../concepts/concepts).

---
[Back: Placing Values and Functions](placement.html){: .btn .btn-purple}
