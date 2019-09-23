---
layout: default
title: Connection Management
parent: Concepts
nav_order: 4
---
# Connection Management
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Static Connections
Peer connections that are known at compile time can be established when creating the `Instance` in the `App` object:

```scala
object Client extends App {
    multitier start new Instance[Chat.Client](
             connect[Chat.Server] { TCP("server-address", 12345) }
            .and(listen[Chat.Client] { TCP(12346) })
    )
}
```
The above code initializes a Client which connects to a Server with the address "server-address" and the port 12345. It also listens
for other Clients on port 12346. If it is only necessary to connect or listen, a shorter syntax can be used:
```scala
object Server extends App {
    multitier start new Instance[Chat.Server](
        listen[Chat.Client] { TCP(12345) }
    )
}
```


## Disconnecting from a Remote
If it is necessary to disconnect from a peer it can be done by calling the `disconnect` method on the `remote` instance:
```scala
remote[Chat.Client].disconnect()
```

## Dynamic Connections
Similarly, we can establish a connection to another peer at runtime by calling `connect` on the remote instance:

```scala
remote[Chat.Client] connect TCP("localhost", 12345)
```