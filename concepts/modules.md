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

This interface can then be implemented using a concrete storage system by using `extends`.

```scala
@multitier trait FileBackup extends Backup {
	def store(id: Long, data: Data): Unit on Processor = placed {
		// code that makes a remote call to insert() to send data to Storage
	}

	def load(id: Long): Future[Data] on Processor = placed {
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

## Dependence between Modules

This section will highlight concepts on how to deal with dependencies between modules. The previous sections
show examples for unidirectional dependencies. So an `Editor` referencing the storage module or
a `MonitoredMasterWorker` using the implementation of its subsystems meanwhile the subsystems being independent of 
each other. This means there is only a one-way dependency. Designing a system, often one problem occurs: cyclic 
dependencies. This means that an multitier module `A` has a reference to multitier module `B` and vice versa.

Considering the mentioned peer types specialization strategies, the following sections present one approach based on
each strategy for dealing with them. Besides, specially targeting cyclic dependencies, other projects could include 
these ideas too in order to integrate a clear structure. 

### Encapsulated

The first approach is about making independent modules that expose signals and events for their respective
input and output. Note that in this case peer definitions need to be independent of each other.

Considering module referencing with the editor example with an added logging module. This logging module is responsible
for logging changes made to the document and aggregating them on a separate peer. 

First, create a new multitier module. There are two inputs (i.e. `injectedDoc` and `saveEvent`) and one output 
(i.e. `logMessages`). 

```scala
@multitier trait Logging {

  @peer type ChangeSource <: { type Tie <: Single[Aggregator] }
  @peer type Aggregator <: { type Tie <: Single[ChangeSource] }

  val injectedDoc: Signal[String] on ChangeSource
  val saveEvent: Event[Unit] on ChangeSource

  [...]

  val logMessages: Signal[Seq[String]] on Aggregator = ...
}
```  

The inputs are retrieved from the parent `Editor` module. The aggregator is responsible for reacting on the changes
made on the client (e.g. mapping them into a different format) and collecting them. It exposes the messages it holds. 
The implementation how the messages are displayed is left out. In this scenario the Editor is responsible for rerouting
it. Therefore, we have a cyclic reference, because the `Logging` takes two inputs from the Editor while the `Editor`
reads the `logMessages` on `Logging`.  

In the `Editor` the peer types needs to be adjusted and the logging module have to be added. The two variables 
`document` and `uiLog` were also added representing the current document and log messages that are taken from the
`Logging` module and are displayed on the UI.

```scala
@multitier trait Editor {
  @peer type Client <: backup.Processor with logging.ChangeSource {
    type Tie <: Single[Server] with Single[backup.Storage] with Single[logging.Aggregator]
  }
  @peer type Server <: backup.Storage with logging.Aggregator {
    type Tie <: Single[Client] with Single[backup.Processor] with Single[logging.ChangeSource]
  }

  // the two modules
  val backup: Backup
  val logging: Logging

  val document: Signal[String] on Client = placed { Var.empty[String] }
  val uiLog: Signal[Seq[String]] on Client = logging.logMessages.asLocal
}
```

The two inputs in the `Logging` module are still undefined. The companion object `editor` needs to be adjusted. This
requires to create two extra variables, here prefixed with `bridge*`. Those variables  have the type `Evt` and `Var`
in order to fire or set them externally, meanwhile the internal usage `Logging` only sees `Event` and `Signal` and
cannot override them. Lastly the new variables needs to be connected. Therefore a main method is introduced that
adds an observer to both original sources and forwards the changes to our bridge variables.

```scala
@multitier object editor extends Editor {
  @multitier object backup extends FileBackup

  // added:
  @multitier object logging extends Logging {
      // The new events needs to be created inside this module, because:
      // (1) variables placed on peers defined by the module needs to placed inside them
      // (2) if events are placed on the parent, asLocal would be required, but it's not usable at this stage
      val bridgeSaveEvent: Evt[Unit] on ChangeSource = placed { Evt[Unit]() }
      val saveEvent: Event[Unit] on ChangeSource = bridgeSaveEvent

      val bridgeDoc: Var[String] on ChangeSource = placed { Var.empty[String] }
      val injectedDoc: Signal[String] on ChangeSource = bridgeDoc
  }

  // connect the new variables
  def main(): Unit on Client = {
    saveButton observe { logging.bridgeSaveEvent fire _ }
    document.changed.observe { logging.bridgeDoc set _ }
  }
}
```

### Self types

The next approach uses the [`Cake`](https://web.archive.org/web/20161002031809/http://www.cakesolutions.net/teamblogs/2011/12/19/cake-pattern-in-depth) 
pattern for `Scala`. It implements dependency injection ([DI](https://en.wikipedia.org/wiki/Dependency_injection)) 
concepts in `Scala` using self-types. The self-types represent the dependency service (e.g. trait) the current
scope requires.

This approach uses Mixin. Therefore, the following example is based of the previous example for peer type mixin. 
Considering a scenario with an added command module. Commands are used to shutdown the and also query the monitoring.
Therefore, we have a similar cyclic problem like before. 

Separating the command module into three components, allows to create unidirectional data flow so that no cyclic
references exist. First create ControlCommand, this will expose the users (here: issuer) intend to shutdown the 
service (here: Actor). This module represents actions that should be performed on actors.

```scala
@multitier trait ControlCommand {
  @peer type Actor <: { type Tie <: Multiple[ControlIssuer] }
  @peer type ControlIssuer <: { type Tie <: Single[Actor] }

  val shutdown: Signal[Boolean] on ControlIssuer = Var(false)
}
```

Next create query module. This will query the Oracle for the current data. In this case the Oracle is a sub-type
of a Monitor peer. However, the data and the peer definition is located inside the Monitoring module:

```scala
@multitier trait Monitoring {
  [...]
  val connected: Signal[Int] on Monitor = placed { Signal.dynamic {
    remote[Monitored].connected().size
  }}
[...]
}
```

Therefore, it's required to add a self-type which represents a dependency on this module. Then the variable `connected` 
could be retrieved in `commandOutput` inside the query module.

```scala 
@multitier trait QueryCommand {
  self: Monitoring =>

  @peer type Oracle <: Monitor { type Tie <: Multiple[QuerySource] with Multiple[Monitored] }
  @peer type QuerySource <: { type Tie <: Single[Oracle] with Single[Monitor] }

  val commandOutput: Signal[String] on QuerySource = connected.asLocal.map("Connected: " + _)
}
```

For convenience, it's possible to group the implementation now in a Command module. It combines both query and control
components. Note that the value of `Monitoring` dependency hasn't specified yet. So we have to specificy the dependency
here too. Furthermore, the tie specification has to be pulled up too. 

```scala
@multitier trait Command extends QueryCommand with ControlCommand {
  self: Monitoring =>

  @peer type Receiver <: Oracle with Monitor with Actor {
    type Tie <: Multiple[QuerySource] with Multiple[Monitored] with Multiple[ControlIssuer]
  }

  @peer type Sender <: QuerySource with ControlIssuer {
    type Tie <: Single[Receiver] with Single[Oracle] with Single[Monitor] with Single[Actor]
  }
}
```

In the end, the combined module `MonitoredMasterWorker` that includes all modules has to add the above command module
and including a new peer type `Client`. In this scenario, the Master peer handles the commands and therefore adds
Receiver for its super type and pulls the respective tie specifications up. 

The `Monitoring` dependency that is required in `Command` is satisfied in this stage, because the combined module
includes the submodule using `extends [...] Monitoring`. 

```scala
@multitier trait MonitoredMasterWorker[T] extends Command with MultipleMasterWorker[T] with Monitoring {
  @peer type Master <: Monitor with Receiver {
    type Tie <: Multiple[Worker] with Multiple[Monitored] with Multiple[Sender]
  }

  @peer type Worker <: Monitored { type Tie <: Single[Master] with Single[Monitor] }
  @peer type Client <: Sender {
    type Tie <: Single[Receiver] with Single[Oracle] with Single[Actor] with Single[Monitor]
  }
}
```

