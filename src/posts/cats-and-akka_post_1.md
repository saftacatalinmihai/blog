---
layout: post
title: Pure Functional Stream processing in Scala
excerpt: Cats and Akka – Part 1
date: 2021-02-06
updatedDate: 2021-02-06
tags:
  - post
  - scala
  - cats
  - akka
---

In [Scala](https://typelevel.org/cats/) you can write [pure functional](https://en.wikipedia.org/wiki/Purely_functional_programming) code, similar to Haskell or other [pure functional languages](https://en.wikipedia.org/wiki/List_of_programming_languages_by_type#Pure), but you’re not obligated to. Wikipedia categories Scala as an impure Functional language.

FP purists view this as a weakness of Scala, but others view the option of “cheating” pureness as an acceptable choice sometimes. Even if you can do everything purely, it’s sometimes a lot easier to think about the problem in a different paradigm.

Pure FP is great for writing correct functions you can easily reason about in isolation and compose well with other pure functions. We can easily unit test them since pure functions only depend on their input arguments and always produce the same output for the same arguments – they are [referentially transparent](https://en.wikipedia.org/wiki/Referential_transparency).

This allows the programmer and the compiler to reason about program behavior as a rewrite system. This can help in proving correctness, simplifying an algorithm, help change code without breaking it, or optimizing code through memoization, common sub-expression elimination, lazy evaluation, or parallelization.

* * *

There are, however, other approaches to thinking about compossibility of programs.

One such approach is to think of software components as black boxes running a process.
They have a variable number of input and output ports which have Types associated to them.
Messages pass asynchronously from component to component after linking their corresponding ports together (if the types match).
We specify the connections outside the components themselves.
This is the thinking behind flow based programming.
(This is also how microservices work at a larger scale)

My view is that these two ways of thinking about composable programs are not mutually exclusive and they can work together in synergy. I will try to make this case by the end of this post.

* * *

### Pure Functional Programming in Scala

Using Cats, you can use Type Classes: Functor, Applicative, Monad etc… to model your programs based on these highly general computational abstractions.
There are other ecosystems for pure FP in Scala. I chose Cats because I’m most familiar with it.

Adding Cats-effect, you can also model IO in a pure functional way. The idea is to write the entire program, including all the effects like: calling external services, writing to file, pushing messages to queues, as a single composed expression that returns an IO data structure representing the action of running all these effects, without actually running them. You only execute them at the “end of the world” in the “main” method.

This is a simple example of a pure functional program using cats-effect.

```scala
import cats.effect.{ IO, Sync }
import cats.implicits._
import scala.io.StdIn

object App {

  def getUserName[F[_]: Sync]: F[String] =
    for {
      _    <- Sync[F].delay(println("What's your name?"))
      name <- Sync[F].delay(StdIn.readLine())
    } yield name

  def greatUser[F[_]: Sync](name: String): F[Unit] =
    Sync[F].delay(println(s"Hello $name!"))

  def program[F[_]: Sync]: F[String] = for {
    name <- getUserName
    _    <- greatUser(name)
  } yield s"Greeted $name"

  def main(args: Array[String]): Unit = {
    program[IO].unsafeRunSync()
  }
}
```

Real programs will, of course, be much more complex, but it all boils down to a single IO value that combines all the effects of running the program which we execute in the “main” method.

Runar has a great talk where he compares using pure FP and IO as working with unexploded TNT. That is much easier to work with as opposed to working with exploded TNT (by actually executing effects in each function).

### Stream processing in Scala

Akka Streams implements the Reactive Streams protocol that’s now standardised in the JVM ecosystem.

Streams have added benefits over simple functions by implementing flow control mechanisms which include back-pressure.

You can think of streams as managed functions, similar to how the Operating System manages threads.

A stream component can decide when to ask for more input messages to pass to its processing function, how many parallel calls to the function to allow, and whether to slow down processing because there is no demand from downstream functions.

Akka is using the abstractions of Source, Flow, Sink for modeling streams.

<p align="center">
    <img alt="source-flow-sink" title="Source via Flow to Sink" src="/11rblog/SourceFlowSink-4.png">
</p>

<!-- ![source-flow-sink](/11rblog/SourceFlowSink-4.png "Source via Flow to Sink") -->