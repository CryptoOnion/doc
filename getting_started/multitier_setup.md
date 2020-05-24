---
layout: default
title: 2. Multitier Setup
parent: Getting Started
nav_order: 1
---
<h1> 2. Multitier & Peer Setup</h1>

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

ScalaLoci is a distributed multitier language. This means that you can write distributed applications as if they were one single application. When compiling your program, it will be 'sliced' into multiple parts and the compiler will create an executable for each **_peer_** (the machines that are going to run parts of your program). All the network communication logic will be generated automatically by ScalaLoci.

This part of our tutorial will cover the peer setup of your application where you define the different types of nodes in your distributed application and their relationship (the topology of your network).

## Imports
In the same directory as your build.sbt, create the folder src/main/scala with the file `Chat.scala`. This will be our chat application.

Add the following imports to `Chat.scala` make ScalaLoci available in your program:

```scala
import loci._ // the loci language
import loci.serializer.upickle._ // the selected serializer
import loci.communicator.tcp._ // the selected communication backend
```

In our example we will use upickle as our serializer and the tcp communication backend. Of course you can make different choices for your real world application. For more information on this see the the ScalaLoci [Github repository](https://github.com/scala-loci/scala-loci).

This example will additionally make use of the [REScala](https://github.com/rescala-lang/REScala) library for reactive and event driven programming in Scala. This allows us to model new chat messages as events.

```scala
import rescala.default._
```

## The multitier environment

In a ScalaLoci application, all code has to live in the multitier environment which you specify with the `@multitier` annotation.

```scala
@multitier object Chat {
    // server and client code lives in here
}
```

## Peer Setup

Let's continue with the definition of our peers. This can be done using the `@peer` annotation. 

Our example will use multiple **_Client_** peers and one **_Server_** peer which will manage the connections. Each client will only have a connection to the server while the server will be connected to all clients.

```scala
@multitier object Chat {
  @peer type Server <: { type Tie <: Multiple[Client] }
  @peer type Client <: { type Tie <: Single[Server] }
}
```

Notice how the keywords `Multiple` and `Single` specify the type and quantity of the ties between the peers.

---
[Back: Installation](installation.html){: .btn .btn-purple .mr-90 }
[Next: Placing Values and Functions](placement.html){: .btn .btn-green}
<!-- TODO: managing connections at runtime -->

<!-- 
## multitier setup
To execute a multitier application all peers have to be initialized inside an `App`:
```scala
object Client extends App {
    multitier setup new Chat.Client {
        def connect = {
             connect[Chat.Server] { TCP("server-address", 12345)
            }
            .and(listen[Chat.Client] {TCP(12346)})
        }
    }
}
```
The above code initializes a Client which connects to a server with the address "server-address" and the port 12345. It also listens
for other Clients on port 12346. If it is only necessary to connect or listen, a shorter syntax can be used:
```scala
object Server extends App {
    multitier setup Chat.Server {
        def connect = listen[Chat.Client] {
            TCP(12345)
        }
    }
}
```


## Disconnecting from a remote
If it is necessary to disconnect from a peer it can be done by calling the `disconnect` method on the `remote` instance:
```scala
remote[Client].disconnect()
``` -->
