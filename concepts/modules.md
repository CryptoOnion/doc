---
layout: default
title: Modules
parent: Concepts
nav_order: 5
---

<h1>Modules</h1>

ScalaLoci comes with a module system that supports encapsulating each (cross-peer) functionality and defining it over abstract peer types. This enables scalability of distributed system made with ScalaLoci, as each of its subsystems can be modularized into its own compilation unit.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Multitier Modules

Multitier modules are defined by a trait with the `@multitier` annotation. Such modules can define peers, their architectural relationship, as well as values that are placed on them.

They can be defined as only abstract members to act as a module interface, or as concrete implementations to later be instantiated. Consider for example the following code, in which a backup service module interface is defined.

```scala
@multitier trait Backup {
	@peer type Processor <: { type Tie <: Single[Storage] }
	@peer type Storage <: { type Tie <: Single[Processor] }

	def store(id: Long, data: Data): Unit on Processor
	def load(id: Long): Future[Data] on Processor
}
```

This example defines a Backup service as having a processor that has a single tie to a storage, as well as a storage that has a single tie to a processor. Furthermore it defines the `store` and `load` methods as having the placed types `.. on Processor`, placing them on the `Processor` peer. Note that the `load` function results in a `Future` to account for processing time and networking delays.

This interface can then be implemented using a concrete storage system by using `extend`.

```scala
@multitier trait FileBackup extends Backup {
	def store(id: Long, data: Data): Unit on Processor = placed {
		// code that makes a remote call to insert() to send data to Storage
	}

	def load(id: Long): Unit on Processor = placed {
		// code that makes a remote call to query() to load data from Storage
	}

	private def write(id: Long, data: Data): Unit on Storage = placed {
		// code that saves data into a file
	}

	private def read(id: Long): Future[Data] on Storage = placed {
		// code that loads data from a file
	}
}
```

In this case the abstract method declarations from before are now implemented. In this case, the code would realize the `store` and `load` methods as calling the `write` or `read` method, which write and load data from a file, respectively. A different implementation might implement these methods in regards to a different storage medium, for example a database.

Remember that because the `write` and `read` methods are defined on the `Storage` peer, `store` and `load`, which are placed on the `Processor` peer, will need to make use of the `remote call` functionality, as described in the section *Remote Access*.

Multitier modules can then be combined. Note how the editor remains agnostic to the implementation details of the backup service.

```scala
@multitier trait Editor {
	@peer type Client <: { type Tie <: Single[Server] }
	@peer type Server <: { type Tie <: Single[Client] }

	val backup: Backup
}
```

Finally a multitier module can be instantiated by declaring an `object` that extends its `trait`.

```scala
@multitier object editor extends Editor {
	@multitier object backup extends FileBackup
}
```

## Abstract Peer Types

To understand peer types it first needs to be said that a peer does *not* refer to a physical place in a system, but a *logical* one instead. Specifically, a peer type distinguishes a values place only at the type level, meaning the placement type `T on P` represents a run time value of only type `T`. A value of type `P` is never constructed at run time. 

Consider the example in the previous section: Both the editor and its backup service define abstract peer types. When running this distributed system, one might want to define the `Client` peer to adopt the `Processor` role.

The next two sections describe two compositions mechanisms to achieve such an effect. The first mechanism uses a module reference, continuing the previous example in which the `Editor` system *includes* a `Backup` system. The other mechanism, module mixing, fully combines two different subsystems into a single module. While both achieve a very similar effect, which one to use will depend on the relationship and functionality of the subsystems and is left up to the programmer.

### Peer type Specialization with Module References

When encapsulating a subsystem within a multitier modules, a peer type can be specialized by narrowing its upper type bound, thereby augmenting it with roles defined by another peer. This is similar to subtyping on classes and enables polymorphic usage of peers.

To define an upper type bound, `<:` is used, like in the following code, which continues the example of the previous section.

```scala
@multitier trait Editor {
    @peer type Client <: backup.Processor { type Tie <: Single[Server] with Single[backup.Storage] }
    @peer type Server <: backup.Storage { type Tie <: Single[Client] with Single[backup.Processor] }
    
    val backup: Backup
}
```

Here the `Client` peer is specified to be a (subtype of the) `backup.Processor` peer, while the `Server` is specified as a (subtype of the) `backup.Storage` peer. The result of this is that all the functionality of the processor peer is also placed on the client peer. This means that all values and values of the processor are available on the client.

Note how the `Client` is defined to have a tie to not only a single `Server` but also a single `Storage`. Even though a Server is defined to be a subtype of Storage, these kinds of ties still need to be properly defined.

### Peer type Specialization with Multitier Mixing

Mixing multitier modules allows a single module to contain the implementations of different subsystems. 

The following example aims to demonstrate module mixing by combining a task distribution system together with a monitoring system.

Firstly, consider this implementation of a master-worker model, in which a master offloads tasks to worker nodes:

```scala
@multitier trait MultipleMasterWorker[T] {
    @peer type Master <: { type Tie <: Multiple[Worker] }
    @peer type Worker <: { type Tie <: Single[Master] }
    
    def run(task: Task[T]): Future[T] on Master = placed {
        (remote(selectWorker()) call execute(task)).asLocal
    }
    private def execute(task: Task[T]): T on Worker = placed { task.process() }
    ...
}
```

A master is defined to have a multiple tie to workers and a worker is defined to have a single tie to a master. The `run` method is placed on `Master` and its offloads a given task by making a remote call to a workers `execute` method. It bases its selection of which worker to select on the `selectWorker` method. The implementation of this method is omitted for simplicities sake.

Secondly, consider this example case of a Monitoring subsystem, which reacts to possible failures of other distributed systems by using heartbeat messages. The `monitoredTimedOut` method is to be called whenever a heartbeat has not been received from a monitored peer instance for some time. The implementation of this mechanism is left out for simplicity.

```scala
@multitier trait Monitoring {
    @peer type Monitor <: { type Tie <: Multiple[Monitored] }
    @peer type Monitored <: { type Tie <: Single[Monitor] }
    
    def monitoredTimedOut(monitored: Remote[Monitored]): Unit on Monitor
    ...
}
```

With module mixing, this monitoring system can easily be added to another subsystem, such as the master-worker model. This is done by extending both modules that are to be mixed and then combining their architectures by specializing the peers of one module to the peers of the other module by using `<:`, like so:

```scala
@multitier trait MonitoredMasterWorker[T] extends MultipleMasterWorker[T] with Monitoring {
    @peer type Master <: Monitor { type Tie <: Multiple[Worker] with Multiple[Monitored] }
    @peer type Worker <: Monitored { type Tie <: Single[Master] with Single[Monitor] }
}
```

In this case, the `Master` and `Worker` peers are overridden to be (subtypes of) `Monitor` and `Monitored` peers. Note how `Master` still needs to specify a tie both to `Worker` and `Monitored`, even though `Worker` is now specified as a subtype of `Monitored`.