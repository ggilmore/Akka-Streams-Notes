```scala
package sample.stream

import java.io.File

import akka.actor.ActorSystem
import akka.stream.ActorMaterializer
import akka.stream.scaladsl._
import akka.util.ByteString

import scala.concurrent.forkjoin.ThreadLocalRandom
import scala.util.{ Failure, Success }

object WritePrimes {

  def main(args: Array[String]): Unit = {
    implicit val system = ActorSystem("Sys")
    import system.dispatcher
    implicit val materializer = ActorMaterializer()

    // generate random numbers
    val maxRandomNumberSize = 1000000
    val primeSource: Source[Int, Unit] =
      Source(() => Iterator.continually(ThreadLocalRandom.current().nextInt(maxRandomNumberSize))).
        // filter prime numbers
        filter(rnd => isPrime(rnd)).
        // and neighbor +2 is also prime
        filter(prime => isPrime(prime + 2))

    // write to file sink
    import akka.stream.io.Implicits._
    //synchronousFile is a sink that accepts ByteStrings and will eventually complete with a Future[Long] (which is the size of the file in bytes)
    val fileSink = Sink.synchronousFile(new File("target/primes.txt"))
    val slowSink = Flow[Int]
      // act as if processing is really slow
      .map(i => { Thread.sleep(1000); ByteString(i.toString) })
      .toMat(fileSink)((_, bytesWritten) => bytesWritten)

    /**
    * .toMat docs:
    * Connect this [[Flow]] to a [[Sink]], concatenating the processing steps of both.
    * {{{
    *     +----------------------------+
    *     | Resulting Sink             |
    *     |                            |
    *     |  +------+        +------+  |
    *     |  |      |        |      |  |
    * In ~~> | flow | ~Out~> | sink |  |
    *     |  |      |        |      |  |
    *     |  +------+        +------+  |
    *     +----------------------------+
    * }}}
    * The `combine` function is used to compose the materialized values of this flow and that
    * Sink into the materialized value of the resulting Sink.
    */
    
    //in this case (_, bytesWritten) => bytesWritten is equivalent to Keep.right
    //overall, connect the "throttler" to the file writer, return the size of the file when done

    // console output sink
    val consoleSink = Sink.foreach[Int](println)

    // send primes to both slow file sink and console sink using graph API
    val materialized = FlowGraph.closed(slowSink, consoleSink)((slow, _) => slow) { implicit builder =>
      (slow, console) =>
        import FlowGraph.Implicits._
        val broadcast = builder.add(Broadcast[Int](2)) // the splitter - like a Unix tee
        primeSource ~> broadcast ~> slow // connect primes to splitter, and one side to file
        broadcast ~> console // connect other side of splitter to console
    }.run()

    // ensure the output file is closed and the system shutdown upon completion
    materialized.onComplete {
      case Success(_) =>
        system.shutdown()
      case Failure(e) =>
        println(s"Failure: ${e.getMessage}")
        system.shutdown()
    }

  }

  def isPrime(n: Int): Boolean = {
    if (n <= 1) false
    else if (n == 2) true
    else !(2 to (n - 1)).exists(x => n % x == 0)
  }
}

```
