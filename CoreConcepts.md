# Akka Streams Notes

"Akka Streams... Translated to everyday terms it is possible to express a chain
(or as we see later, graphs) of processing entities, each executing
independently (and possibly concurrently) from the others while only buffering a
 limited number of elements at any given time."

## Core Concepts
- Stream:
active process that involves moving and transforming data
- Element:
processing unit of a stream, all operations transform and transfer elements from
upstream to downstream
- Back Pressure:
A means of flow-control, a way for consumers of data to notify a producer about
their current availability, effectively slowing down the upstream producer to
match their consumption speeds. In the context of Akka Streams back-pressure is
always understood as non-blocking and asynchronous.
- Non-Blocking:
Means that a certain operation does not hinder the progress of the calling
thread, even if it takes long time to finish the requested operation.

- Processing Stage
The common name for all building blocks that build up a Flow or FlowGraph.
Examples of a processing stage would be operations like `map()`, `filter()`,
stages added by `transform()` like `PushStage`, `PushPullStage`, `StatefulStage`
and graph junctions like `Merge` or `Broadcast`.

*Note: Processing Stages are immutable*
```scala
val source = Source(1 to 10)
source.map(_ => 0) // has no effect on source, since it's immutable
source.runWith(Sink.fold(0)(_ + _)) // 55

val zeroes = source.map(_ => 0) // returns new Source[Int], with `map()` appended
zeroes.runWith(Sink.fold(0)(_ + _)) // 0
```
___

### Defining and running streams
Linear processing pipelines can be expressed in Akka Streams using the following core abstractions:

- Source:
A processing stage with exactly one output, emitting data elements whenever
downstream processing stages are ready to receive them.
- Sink:
A processing stage with exactly one input, requesting and accepting data
elements possibly slowing down the upstream producer of elements

- Flow:
A processing stage which has exactly one input and output, which connects its
up- and downstreams by transforming the data elements flowing through it.

- RunnableGraph:
A Flow that has both ends "attached" to a Source and Sink respectively, and is
ready to be `run()`.

It is important to remember that even after constructing the `RunnableGraph`
by connecting all the source, sink and different processing stages, no data will
 flow through it until it is materialized. **Materialization**
is the process of allocating all resources needed to run the computation
 described by a Flow (in Akka Streams this will often involve starting up
Actors).

```scala
val source = Source(1 to 10)
val sink = Sink.fold[Int, Int](0)(_ + _)

// connect the Source to the Sink, obtaining a RunnableGraph
val runnable: RunnableGraph[Future[Int]] = source.toMat(sink)(Keep.right)

// materialize the flow and get the value of the FoldSink
val sum: Future[Int] = runnable.run()
```

After running (materializing) the `RunnableGraph[T]` we get back the
materialized value of type` T`. Every stream processing stage can produce a
materialized value, and it is the responsibility of the user to combine them to
a new type. In the above example we used `toMat` to indicate that we want to
transform the materialized value of the source and sink, and we used the
convenience function `Keep.right` to say that we are only interested in the
materialized value of the sink. In our example the `FoldSink` materializes a
value of type `Future` which will represent the result of the folding
 process over the stream. In general, a stream can expose multiple
 materialized values, but it is quite common to be interested in only the
 value of the `Source` or the `Sink` in
the stream. For this reason there is a convenience method called `runWith()`
available for `Sink`, `Source` or `Flow` requiring, respectively, a supplied `Source`
(in order to run a `Sink`), a `Sink` (in order to run a Source) or both a `Source`
and a `Sink` (in order to run a `Flow`, since it has neither attached yet).

```scala
val source = Source(1 to 10)
val sink = Sink.fold[Int, Int](0)(_ + _)

// materialize the flow, getting the Sinks materialized value
val sum: Future[Int] = source.runWith(sink)
```
The `runWith` method both materializes the stream and returns the
materialized value of the given sink or source.

Since a stream can be materialized multiple times, the materialized value
will also be calculated anew for each such materialization, usually leading to
 different values being returned each time. In the example below we create two
 running materialized instance of the stream that we described in the `runnable`
 variable, and both materializations give us a different `Future` from the map
 even though we used the same `sink` to refer to the future:

 ```scala
 // connect the Source to the Sink, obtaining a RunnableGraph
val sink = Sink.fold[Int, Int](0)(_ + _)
val runnable: RunnableGraph[Future[Int]] =
  Source(1 to 10).toMat(sink)(Keep.right)

// get the materialized value of the FoldSink
val sum1: Future[Int] = runnable.run()
val sum2: Future[Int] = runnable.run()

// sum1 and sum2 are different Futures!
```

### Defining sources, sinks and flows

The objects `Source` and `Sink` define various ways to create sources and
sinks of elements. The following examples show some of the most useful
constructs (refer to the API documentation for more details):

```scala
// Create a source from an Iterable
Source(List(1, 2, 3))

// Create a source from a Future
Source(Future.successful("Hello Streams!"))

// Create a source from a single element
Source.single("only one element")

// an empty source
Source.empty

// Sink that folds over the stream and returns a Future
// of the final result as its materialized value
Sink.fold[Int, Int](0)(_ + _)

// Sink that returns a Future as its materialized value,
// containing the first element of the stream
Sink.head

// A Sink that consumes a stream without doing anything with the elements
Sink.ignore

// A Sink that executes a side-effecting call for every element of the stream
Sink.foreach[String](println(_))
```
There are various ways to wire up different parts of a stream,
the following examples show some of the available options:

```scala
// Explicitly creating and wiring up a Source, Sink and Flow
Source(1 to 6).via(Flow[Int].map(_ * 2)).to(Sink.foreach(println(_)))

// Starting from a Source
val source = Source(1 to 6).map(_ * 2)
source.to(Sink.foreach(println(_)))

// Starting from a Sink
val sink: Sink[Int, Unit] = Flow[Int].map(_ * 2).to(Sink.foreach(println(_)))
Source(1 to 6).to(sink)
```

### Stream Materialization

Since every processing stage in Akka Streams can provide a materialized
value after being materialized, it is necessary to somehow express how these
 values should be composed to a final value when we plug these stages together.
 For this, many combinator methods have variants that take an additional
 argument, a function, that will be used to combine the resulting values.
 Some examples of using these combiners are illustrated in the example below.

 ```scala
 // An empty source that can be shut down explicitly from the outside
val source: Source[Int, Promise[Unit]] = Source.lazyEmpty[Int]

// A flow that internally throttles elements to 1/second, and returns a Cancellable
// which can be used to shut down the stream
val flow: Flow[Int, Int, Cancellable] = throttler

// A sink that returns the first element of a stream in the returned Future
val sink: Sink[Int, Future[Int]] = Sink.head[Int]

// By default, the materialized value of the leftmost stage is preserved
val r1: RunnableGraph[Promise[Unit]] = source.via(flow).to(sink)

// Simple selection of materialized values by using Keep.right
val r2: RunnableGraph[Cancellable] = source.viaMat(flow)(Keep.right).to(sink)
val r3: RunnableGraph[Future[Int]] = source.via(flow).toMat(sink)(Keep.right)

// Using runWith will always give the materialized values of the stages added
// by runWith() itself
val r4: Future[Int] = source.via(flow).runWith(sink)
val r5: Promise[Unit] = flow.to(sink).runWith(source)
val r6: (Promise[Unit], Future[Int]) = flow.runWith(source, sink)

// Using more complex combinations
val r7: RunnableGraph[(Promise[Unit], Cancellable)] =
  source.viaMat(flow)(Keep.both).to(sink)

val r8: RunnableGraph[(Promise[Unit], Future[Int])] =
  source.via(flow).toMat(sink)(Keep.both)

val r9: RunnableGraph[((Promise[Unit], Cancellable), Future[Int])] =
  source.viaMat(flow)(Keep.both).toMat(sink)(Keep.both)

val r10: RunnableGraph[(Cancellable, Future[Int])] =
  source.viaMat(flow)(Keep.right).toMat(sink)(Keep.both)

// It is also possible to map over the materialized values. In r9 we had a
// doubly nested pair, but we want to flatten it out
val r11: RunnableGraph[(Promise[Unit], Cancellable, Future[Int])] =
  r9.mapMaterializedValue {
    case ((promise, cancellable), future) =>
      (promise, cancellable, future)
  }

// Now we can use pattern matching to get the resulting materialized values
val (promise, cancellable, future) = r11.run()

// Type inference works as expected
promise.success(0)
cancellable.cancel()
future.map(_ + 3)

// The result of r11 can be also achieved by using the Graph API
val r12: RunnableGraph[(Promise[Unit], Cancellable, Future[Int])] =
  FlowGraph.closed(source, flow, sink)((_, _, _)) { implicit builder =>
    (src, f, dst) =>
      import FlowGraph.Implicits._
      src ~> f ~> dst
  }
```
