---
layout: default
title: 1. Installation
parent: Getting Started
nav_order: 1
---

<h1>1. Installation</h1>

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Setting up your project using sbt

The following instructions will use [sbt](https://www.scala-sbt.org) as the build tool to manage your project and its dependencies.

After installing sbt (see their [documentation](https://www.scala-sbt.org/1.x/docs/index.html) for more information on that) run the following shell commands to create a new sbt project. 

```bash
$ mkdir my-chat-app
$ cd my-chat-app
$ touch build.sbt
```

## Getting Scala Loci

Now there are a few steps needed to install the Scala Loci language for use in your Scala project. (Those are copied from the Scala Loci [Github repository](https://github.com/scala-loci/scala-loci) so you might want to check there too just in case anything changes.)

1. Enable the [Macro Paradise Plugin](http://docs.scala-lang.org/overviews/macros/paradise.html) (for macro annotations) in your `build.sbt`

   ```scala
   addCompilerPlugin("org.scalamacros" % "paradise" % "2.1.1" cross CrossVersion.patch)
   ```

2. Add the resolver for the ScalaLoci dependencies to your `build.sbt`

   ```scala
   resolvers += Resolver.bintrayRepo("stg-tud", "maven")
   ```

3. Add the ScalaLoci dependencies that you need for your system to your `build.sbt`

   1. ScalaLoci language (always required)

      ```scala
      libraryDependencies += "de.tuda.stg" %% "scala-loci-lang" % "0.3.0"
      ```

   2. Transmitter for the types of values to be accessed remotely
      (built-in Scala types and standard collections are directly supported without additional dependencies)

      * [REScala](http://www.rescala-lang.com/) reactive events and signals

        ```scala
        libraryDependencies += "de.tuda.stg" %% "scala-loci-lang-transmitter-rescala" % "0.3.0"
        ```

   3. Network communicators to connect the different components of the distributed system

      * TCP [*JVM only*]
  
        ```scala
        libraryDependencies += "de.tuda.stg" %% "scala-loci-communicator-tcp" % "0.3.0"
        ```

      * WebSocket (using [Akka HTTP](http://doc.akka.io/docs/akka-http/current/) on the JVM) [*server: JVM only, client: JVM and JS web browser APIs*]

        ```scala
        libraryDependencies += "de.tuda.stg" %% "scala-loci-communicator-ws-akka" % "0.3.0"
        ```

      * WebSocket ([Play](http://www.playframework.com) integration) [*server: JVM only, client: JVM and JS web browser APIs*]

        ```scala
        libraryDependencies += "de.tuda.stg" %% "scala-loci-communicator-ws-akka-play" % "0.3.0"
        ```

      * WebRTC [*JS web browser APIs only*]

        ```scala
        libraryDependencies += "de.tuda.stg" %% "scala-loci-communicator-webrtc" % "0.3.0"
        ```

   4. Serializer for network communication

      * [µPickle](http://www.lihaoyi.com/upickle/) serialization

        ```scala
        libraryDependencies += "de.tuda.stg" %% "scala-loci-serializer-upickle" % "0.3.0"
        ```

      * [Circe](http://circe.github.io/circe/) serialization

        ```scala
        libraryDependencies += "de.tuda.stg" %% "scala-loci-serializer-circe" % "0.3.0"
        ```