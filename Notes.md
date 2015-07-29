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

Defining and running streams
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
materialized value of the sink. In our example the FoldSink materializes a value
of type `Future` which will represent the result of the folding process over the
stream. In general, a stream can expose multiple materialized values, but it is
quite common to be interested in only the value of the Source or the Sink in
the stream. For this reason there is a convenience method called `runWith()`
available for Sink, Source or Flow requiring, respectively, a supplied Source
(in order to run a Sink), a Sink (in order to run a Source) or both a Source
and a Sink (in order to run a Flow, since it has neither attached yet).
