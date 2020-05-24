---
layout: default
title: 3. Placing Values and Functions
parent: Getting Started
nav_order: 3
---
<h1> 3. Placing Values and Functions </h1>
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Placing Values

When defining **values** in our `@multitier` environment, we have to define where those values are going to be stored. We do so using the `on` keyword.

To define a "message" event which can be fired by the clients in order to send a new message we write the following inside our `Chat` object:

```scala
val message = on[Client] { Evt[String] }
```

`on[Client]` specifies that the value is going to be stored on the clients. The actual value of the `val` will be defined inside the curly braces.

The `Evt` here is a **REscala** keyword used for specifying events. `Evt[String]` says that new messages will be represented by events of type String.

## Accessing Remote Values

To access a value that lives on **another peer**, we have to call `.asLocal` on these values.
If multiple peers hold a copy of the values we have to use `.asLocalFromAll` which returns a map from connections to values or `.asLocalFromAllSeq` which directly returns a sequence of the values.

In our chat example we use this to define an event on the server which combines all message events from the clients:

```scala
val publicMessage = on[Server] sbj { client: Remote[Client] =>
    message.asLocalFromAllSeq collect {
      case (remote, message) if remote != client => message
    }
}
```

Notice the `sbj` keyword which let's us adapt the value to each peer that is accessing it. In this example we use this to not display their own messages to clients that are accessing the public messages.

## Placing Functions

Functions can be placed on peers in the same way as values. This is demonstrated in the application logic for our client peers:

```scala
def main() = on[Client] {
    // observe the publicMessage event and print messages as they arrive
    publicMessage.asLocal observe println

    // let the user send new messages via stdin
    for (line <- scala.io.Source.stdin.getLines) {
      message.fire(line)
    }
    }
}
```

Notice how we access the `publicMessage` event which is stored on another peer (the server) using `asLocal`.

We can also call functions of other peers using `asLocal`, however, this will not be covered in this tutorial. More information can be found in the [Concepts section](../concepts/remote_calls).

---
[Back: Multitier Setup](multitier_setup.html){: .btn .btn-purple}
[Next: Running your Application](execution.html){: .btn .btn-green}

