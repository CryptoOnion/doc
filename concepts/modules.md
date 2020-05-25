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
	@peer type Storage <: { type Tie <: Multiple[Processor] }

	def store(id: Long, data: Data): Unit on Processor
	def load(id: Long): Future[Data] on Processor
}
```

This interface can then be implemented using a concrete storage system by using `extend`.

```scala
@multitier trait FileBackup extends Backup {
	def store(id: Long, data: Data): Unit on Processor = placed {
		// code to store data to a file
	}

	def load(id: Long): Unit on Processor = placed {
		// code to load data from a file
	}
}
```

Multitier modules can then be combined. Note how the chat system remains agnostic to the implementation details of the backup service.

```scala
@multitier trait Chat {
	@peer type Client <: { type Tie <: Single[Server] }
	@peer type Server <: { type Tie <: Multiple[Client] }

	val backup: Backup
}
```

Finally a multitier module can be instantiated by declaring a n `object` that extends its `trait`.

```scala
@multitier object editor extends Chat {
	@multitier object backup extends FileBackup
}
```

## Abstract Peer Types

