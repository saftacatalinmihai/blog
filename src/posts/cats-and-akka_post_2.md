---
layout: post
title: Pure Functional Stream processing in Scala [2]
excerpt: Cats and Akka – Part 2
date: 2021-02-14
updatedDate: 2021-02-14
tags:
  - post
  - scala
  - cats
  - akka
---

In the last [post](/post/pure-functional-stream-processing-in-scala-1/), we saw how to combine pure functions running in IO and Akka streams using `.mapAsync` and `.unsafeToFuture`

```scala
val source: Source[String, NotUsed] = ???
val sink: Sink[Int, NotUsed] = ???
def pureFunction[F[_]](s: String): F[Int] = ???

source
  .mapAsync(parallelism = 8)(m => pureFunction[IO](m).unsafeToFuture())
  .runWith(sink)
```

In order to make it easier to work with IO in Akka Streams, we can write some helpers to add a method on Streams that automatically run the IO inside the flow. This will simplify the interaction between pure code and stream code.

Readability will improve, **but** before we get there, it’s very important to recognise that we should view these 2 ways of writing code<sup id="a1">[[1]](#f1)</sup> as having [orthogonal](https://en.wikipedia.org/wiki/Orthogonality_\(programming\) ) purposes.

### Streams as plumbing

<div align="center">
    <img alt="plumbing" title="Plumbing" src="/plumbing.jpeg">
</div>

The philosophy is identical to how [Unix pipes](https://en.wikipedia.org/wiki/Pipeline_(Unix))  work.\
You write small pieces of code using simple programs like: `ps, grep, find, sed, awk, xargs, kill` etc… and join those smaller programs into a bigger one by piping data between them. The output of one program goes into the input of the next.

There is a clear separation of duties between programs that do something<sup id="a2">[[2]](#f2)</sup> and pipes that just pass data along between programs.

Here’s an example of a pipeline that finds all java processed and stops them:

```scala
ps aux | grep java | grep -v grep | awk '{ print $2 }' | xargs kill -9
```

The similarity between this and Akka flows is almost one to one<sup id="a3">[[3]](#f3)</sup>.

One issue in Scala is that we write both the **program** and the **pipeline / stream** in the same language, which can blurry the separation of concerns…\
For this reason, some people don’t see why we should separate these two ways of writing the full program.
My view is that we should try to separate them logically to get to the simple way of composing programs like using Unix Pipelines.

My suggestion here is to have separate modules for the domain, and Classes with pure functions working on the domain that run certain actions.\
After constructing those Classes (which can be individually unit-tested), write a Class taking the other domain and functionality Classes as constructor arguments.

This Class should just combine all the individual pure functions by wrapping them in Akka flows, but nothing more. No extra business logic other than the order of operations…

### Simplify Cats and Akka interaction

Assuming we understand the separation of concerns between pure functions and streams, we still want to make their combination easy. Maybe as easy as the Unix pipe.

The first step is to simplify using `.mapAsync` and IO.

What we need is another method on Akka streams that automatically runs the IO<sup id="a4">[[4]](#f4)</sup> inside the Flow by transforming it into a Future.

We can enable this generically by wrapping a source or flow in extension classes with the extra method: `.unsafeMapAsync`

```scala
implicit class SourceExtensions[A, Mat](val source: Source[A, Mat]) extends AnyVal {
  def unsafeMapAsync[F[_]: Effect, B](parallelism: Int)(f: A => F[B]): source.Repr[B] =
    source.mapAsync(parallelism)(b => Effect[F].toIO(f(b)).unsafeToFuture())
}

implicit class FlowExtensions[A, B, Mat](val flow: Flow[A, B, Mat]) extends AnyVal {
  def unsafeMapAsync[F[_]: Effect, C](parallelism: Int)(f: B => F[C]): flow.Repr[C] =
    flow.mapAsync(parallelism)(b => Effect[F].toIO(f(b)).unsafeToFuture())
}
```

The method name shows we are running the effect which is unsafe<sup id="a5">[[5]](#f5)</sup> – similar to the method name `.unsafeToFuture`
This is the split between pure FP and Akka streams. We are now in streaming territory where we run the effects in each flow step.

* * *

Now we can rewrite the code from the previous [post](/post/pure-functional-stream-processing-in-scala-1/#mapAsync-unsafeToFuture).

<h6 id="refactor-1"></h6>

```scala
Source(List("1|123", "2|123", "3|789"))
  .unsafeMapAsync(parallelism = 8)(m => parseMessage[IO](m))
  .unsafeMapAsync(parallelism = 8)(m => getUser[IO](m.userId))
  .unsafeMapAsync(parallelism = 8)(u => checkPermission[IO](u).map(p => (u, p)))
  .unsafeMapAsync(parallelism = 8) { case (u, p) => sendNewsletterIfAllowed[IO](u, p) }
  .runWith(Sink.seq)
```

Notice the lack of calls to `.unsafeToFuture()`\
Much cleaner.

* * *

Another refactor we can do is to extract individual flows in values – same as we could do with functions, but in this case they are Flow types.

```scala
val parseFlow = Flow[String].unsafeMapAsync(8)(parseMessage[IO])
val getUserFlow = Flow[Message].unsafeMapAsync(8)(m => getUser[IO](m.userId))
val checkPermissionFlow = Flow[User].unsafeMapAsync(8)(u => checkPermission[IO](u).map( (u, _) ))
val sendNewsletterFlow = Flow[(User, Boolean)].unsafeMapAsync(8) {
  case (u, p) => sendNewsletterIfAllowed[IO](u, p)
}
```

And then use the `.via` method to combine them in a bigger flow

```scala
val flow = parseFlow
  .via(getUserFlow)
  .via(checkPermissionFlow)
  .via(sendNewsletterFlow)
```

Looks much closer to the unix pipeline example, so I’m happy 😃.\
We construct the full stream using .via to connect the Source and Sink as well

```scala
Source(List("1|123", "2|123", "3|789"))
  .via(flow)
  .runWith(Sink.seq)
```

* * *

We can go a step further and use the [GraphDSL](https://doc.akka.io/docs/akka/current/stream/stream-graphs.html#constructing-graphs)

```scala
val flow = Flow.fromGraph(GraphDSL.create() { implicit builder =>
  import GraphDSL.Implicits._

  val Parse = builder.add(parseFlow)
  val GetUser = builder.add(getUserFlow)
  val CheckPermission = builder.add(checkPermissionFlow)
  val SendNewsletter = builder.add(sendNewsletterFlow)

  Parse ~> GetUser ~> CheckPermission ~> SendNewsletter

  FlowShape(Parse.in, SendNewsletter.out)
})
```

Here we add the flows as nodes in the Akka Graph, then join them together in a single flow using Akka’s GraphDSL and the `~>` operator.\
The result is a graph with the [Shape](https://doc.akka.io/docs/akka/current/stream/stream-graphs.html#predefined-shapes) of a Flow – one input, out output. We can create an actual Flow from the graph with the constructor: `Flow.fromGraph`

Notice how similar the expression:\
`Parse ~> GetUser ~> CheckPermission ~> SendNewsletter`\
Is to a unix pipeline. Just replace ~> with | and it’s the same.

It’s also very similar to something we might draw to represent this flow.

<div align="center">
    <img alt="flow-representation" title="Flow Representation" src="/Flow-1.png">
</div>

I think this is one of the significant advantages of using Akka streams and the GraphDSL – **you can look at the actual code to see the computational graph instead of looking at representations of it**.\
It’s not drawn by someone else, which can also become outdated quickly. The code does not lie, a representation can.

The major difference between the GraphDSL and using `.unsafeMapAsync` directly is that we separate individual graph nodes from their connection to other nodes – similar to the idea in [Flow based programming](https://en.wikipedia.org/wiki/Flow-based_programming). This allows for higher flexibility.\
If there are multiple implementations for a component with the same interface, we can easily just swap the component to the new one, but the graph structure remains the same.

The GraphDSL allows us to create more complex computational graphs than this simple one.\
For this one, it is of course easier to use the simpler version from the [first example](#refactor-1).

### Complex graphs

So far we haven’t looked at how to treat errors in the stream…\
If we do nothing, for any exception thrown in the stream, Akka will simply silently ignore them… There are methods to deal with that the [Akka way](https://doc.akka.io/docs/akka/current/stream/stream-error.html).

However, let’s look at what we can implement to make sure the stream never stops processing but we still handle errors. We can do that because we’re in control of the way we execute the effects in the stream.

We can design a different stream component that will catch exceptions in the **IO** and push it on a separate error stream – similar to the STDERR of Unix.

<div align="center">
    <img alt="component-err" title="Component Error Output" src="/ComponentErr-1.png">
</div>

To achieve this we can add another extension method on Flow and Source which will run the effect but return a component with a specific output for the Error, if there is any error thrown in the effect.

```scala
implicit class FlowExtensions[A, B](val flow: Flow[A, B, NotUsed]) extends AnyVal {

  // ...
  def safeMapAsync[F[_]: Effect, C](parallelism: Int)(
    f: B => F[C]
  ): Graph[FanOutShape2[A, C, Throwable], NotUsed] = {
    split[A, Throwable, C](
      flow.mapAsync(parallelism)(b => Effect[F].toIO(f(b)).attempt.unsafeToFuture())
    )
  }
}
```













</br>

#### References

<b id="f1">[1]</b> Pure code and Streams code [↩](#a1)

<b id="f2">[2]</b> Programs that do some work and have side effects [↩](#a2)

<b id="f3">[3]</b> The difference is that Unix pipes have 2 output streams: STDOUT and STDERR.\
In Akka flows, you only have one output as a stream – which is for successful processing.\
It throws errors out of bound – you don’t have an error stream [↩](#a3)

<b id="f4">[4]</b> We can be more generic by using the [Effect](https://typelevel.org/cats-effect/typeclasses/effect.html) type constraint instead of IO directly [↩](#a4)

<b id="f5">[5]</b> Because it executes side effects [↩](#a5)