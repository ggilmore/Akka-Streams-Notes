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
A Flow that has both ends "attached" to a Source and Sink respectively, and is ready to be run().