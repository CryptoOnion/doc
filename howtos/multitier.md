---
layout: default
title: Multitier Setup
parent: Getting Started
nav_order: 1
---
# Multitier Setup
{: .no_toc }


## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## multitier environments
Currently, all code has to live inside a scala object, with a `@multitier` annotation if it wants to make use of ScalaLoci's features:
```scala
@multitier
object Chat {
    // server and client code lives in here
}
```
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
```
